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
| `history` | `list` | Gradio conversation history — list of `{"role": ..., "content": ...}` message dicts |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. The app creates its chat UI with
`type="messages"`, so Gradio history arrives as a list of API-format dicts with
`role` and `content` keys. Gradio may include extra keys (like `metadata`), so
copy only the two fields the API expects:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for msg in history:
    messages.append({"role": msg["role"], "content": msg["content"]})

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
back to the specific tool call that requested it.

⚠️ For a no-argument tool call (like `get_seasonal_conditions` with no season),
the model may send `arguments` as the JSON string `"null"` — `json.loads` turns
that into `None`, not `{}`. Normalize before dispatching:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    raw_args = tool_call.function.arguments
    tool_args = json.loads(raw_args) if raw_args else {}
    if not isinstance(tool_args, dict):
        tool_args = {}
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
[ The loop gets terminated when LLM finds no tool calls and that means it knows the answer without referring to the tool call. That's one way. 
The other way is that if you reach the maximum tool calls then the loop doesn't go through; in that case  we will return the final response of llm or  " I couldn't generate a response" ]
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
[ Usually responses.choices[0].message.content is the final response from the LLM by taking  all the messages like history, user message, tool calls and tool results. If no such response comes up, it says I could not generate the response. ]
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: [lookup_plant, {'plant_name': 'calathea'}]
Round 2 tool call: [et_seasonal_conditions, {}] (if any)
Final response: [According to the care data for your Calathea, it requires consistent moisture, but make sure not to overwater, as this can lead to root rot and other issues. Use filtered or distilled water to prevent brown edges on the leaves. The plant prefers low to medium indirect light, and high humidity (50%+), which can be achieved with a humidifier, frequent misting, or by grouping it with other plants. Keep the temperature between 60-80°F (15-27°C) and avoid cold drafts. Fertilize your Calathea monthly during the growing season with a diluted balanced fertilizer.

Given that it's currently summer, your Calathea will require more frequent watering due to the heat and dry air. Be careful not to overwater, and continue to fertilize regularly. Keep an eye out for pests like spider mites and fungus gnats, which thrive in hot and dry conditions. Make sure to provide your Calathea with bright, indirect light, and maintain high humidity to keep it happy and healthy.]
```

**What happens when you ask about a plant that isn't in the database?**

```
Query: How do I care for my string of pearls?
Tool Calls : [lookup_plant, {'plant_name': 'string of pearls'}]
Result: {"found": false, "name": "string of pearls", "message": "'string of pearls' did not match any key, display name, scienti"}
Output: I couldn't find any information on a plant called 'string of pearls' in my database. However, I believe you may be referring to Senecio rowleyanus, which is a common houseplant with the nickname 'string of pearls'. If that's correct, here are some general care tips: string of pearls prefers bright, indirect light and well-draining soil. It's also important to water sparingly, allowing the soil to dry out completely between waterings. A temperature range of 65-75°F (18-24°C) is ideal, and fertilization during the growing season (spring and summer) can promote healthy growth. Keep in mind that these are general guidelines, and the specific needs of your plant may vary. If you have any further questions or concerns, feel free to ask.

My experience: Firstly it had a call with string of pearls and looked into the database. Since it found out that there is no such plant called string of pearls, it came up with saying things like "I couldn't find any information" and also with its own knowledge it had made up an answer.
```

**One thing about the tool call API that surprised you:**

```
[  Tool calls are really fast rather than RAG, that's what I felt, because the information is flowing through the JSON format. I felt that tool calls should be implemented when it comes to when you have multiple data sources. ]
```
