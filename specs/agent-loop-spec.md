# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
Structure: a bounded `for _ in range(MAX_TOOL_ROUNDS):` loop, NOT `while True`.
The bound is the hard safety valve — even if the LLM keeps requesting tools every
round, the loop can iterate at most MAX_TOOL_ROUNDS times and cannot spin forever.

(a) No tool calls — the normal exit:
    After each LLM call, read `assistant_message = response.choices[0].message`.
    If `not assistant_message.tool_calls` (None or empty list), the LLM has a final
    answer. Return `assistant_message.content` — but guard it: if content is None or
    empty, return a fixed fallback string instead of "" so the UI never shows a blank
    reply. Then `break`/`return` out of the loop.

(b) MAX_TOOL_ROUNDS reached — the safety exit:
    If the loop body runs all MAX_TOOL_ROUNDS iterations and the LLM is STILL asking
    for tools, control falls past the `for` loop. At that point make ONE final LLM
    call with `tool_choice="none"` to force a plain-text answer, and return its
    content (with the same empty-guard fallback). This degrades gracefully instead
    of returning the last tool JSON or crashing.

Failure modes this design closes:
  - Loop forever  -> impossible; range() bounds the iterations.
  - Empty string  -> guarded: `content or FALLBACK` on both exit paths.
  - Exception     -> the whole body is wrapped in try/except (network error, bad
                     tool args, dispatch_tool raising); on any exception, log it and
                     return the fallback string. Also: `json.loads(arguments or "{}")`
                     is itself guarded so malformed/empty tool arguments don't raise.
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
The text lives on the assistant message: response.choices[0].message.content

`choices` is a list (index 0 is the first/only completion); `.message` is the
assistant message object; `.content` is the string to return. Note that `.content`
is None whenever `.tool_calls` is populated — that's why content is only read on the
no-tool-calls path. Even there, wrap it as `assistant_message.content or FALLBACK`
so a rare empty completion can't surface as a blank chat bubble.
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How often should I water my snake plant in winter?"
Round 1 tool call: lookup_plant({"plant_name": "snake plant"})
Round 2 tool call: get_seasonal_conditions({"season": "winter"})
Final response: Combines the plant's watering data with winter seasonal guidance —
                "water every 2–6 weeks depending on season; in winter, once a month
                or less, keep away from cold drafts." Both tool results are synthesized
                into one answer in a single turn.
```

**What happens when you ask about a plant that isn't in the database?**

```
"string of pearls" -> lookup_plant returns {"found": false, ...} with the not-found
message that lists available plants. The LLM reads that, tells the user the plant
isn't in the database, and then offers general succulent care guidance from its own
knowledge — exactly the graceful degradation the system prompt asks for. No crash,
no empty response.
```

**One thing about the tool call API that surprised you:**

```
Two things, both caught during testing:

1. For a no-argument tool call, the model doesn't always send "{}". It sometimes
   sends the JSON literal `null`, and json.loads("null") returns None, not a dict.
   Passing None into dispatch_tool's tool_args.get(...) raised AttributeError. Fix:
   coerce the parsed arguments to {} unless they're actually a dict.

2. The tool call can fail server-side. Groq occasionally returns a 400
   `tool_use_failed` when the Llama model emits malformed function-call syntax
   (e.g. `<function=lookup_plant {...}</function>`). It's intermittent and
   model-side, so the loop must wrap the API call in try/except and return a
   user-readable fallback rather than letting the exception bubble up to the UI.
```
