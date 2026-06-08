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
**Condition A: No tool calls**
- Detect: Check if `assistant_message.tool_calls` is falsy (None or empty list)
- Action: Break from the loop immediately
- Return: Extract `response.choices[0].message.content` and return it
  - If content is None/empty, return fallback: "I'd like to help, but I'm having trouble generating a response. Please try rephrasing your question."

**Condition B: MAX_TOOL_ROUNDS reached**
- Detect: Track iteration count in while loop condition: `while tool_round < MAX_TOOL_ROUNDS`
- Action: Exit loop naturally when tool_round reaches MAX_TOOL_ROUNDS
- Return: Extract `response.choices[0].message.content` from the last response
  - The last response may have tool_calls that we didn't execute (due to hitting limit)
  - Still extract and return its content field
  - If content is None/empty, return fallback: "I've reached the limit of tool calls for this query. Please try a simpler question."

**Edge case handling:**
- Both empty string and None are treated as missing content → need fallback
- Response object is guaranteed to exist before extraction (loop runs at least once with MAX_TOOL_ROUNDS=5)
- Never return an empty string — always have a fallback message ready
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
The response object structure:
  response.choices[0]              → The first (only) choice
  response.choices[0].message      → The assistant message object
  response.choices[0].message.content  → The text response string

Extraction:
  final_response = response.choices[0].message.content

Fallback (for None or empty string):
  if not final_response:
      final_response = "I'd like to help, but I'm having trouble generating a response. Please try rephrasing your question."
  
  (Use a different fallback for MAX_TOOL_ROUNDS condition if desired)

Return the string directly — never return None or empty string per the output contract.
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How do I care for my pothos?"
Round 1 tool call: lookup_plant({"plant_name": "pothos"})
Result: Returns full pothos plant data including watering (1-2 weeks), light (low to bright indirect), humidity, temperature, and common issues
Final response: Agent provides specific care instructions based on the plant data

Query: "How often should I water my snake plant in winter?"
Round 1 tool call: lookup_plant({"plant_name": "snake plant"})
Result: Returns snake plant data showing watering frequency "every 2-6 weeks depending on season"
Round 2 tool call: get_seasonal_conditions({"season": "winter"})
Result: Returns winter guidance including "Water once a month or less" for the season
Final response: Agent combines both pieces of information to give specific winter watering advice

Query: "My calathea has brown edges. What should I do?"
Round 1 tool call: lookup_plant({"plant_name": "calathea"})
Result: Returns calathea data including common issues and humidity sensitivity
Final response: Agent identifies mineral buildup or humidity issues based on plant data
```

**What happens when you ask about a plant that isn't in the database?**

```
When lookup_plant returns {"found": false, "name": "banana plant", "message": "Plant not found in database..."},
the agent reads this message and generates a helpful response explaining that the plant isn't in the database.
It then offers to provide general guidance based on the user's description of the plant (color, leaf shape, size).
The agent gracefully falls back to general knowledge rather than making up plant data.
```

**One thing about the tool call API that surprised you:**

```
The LLM can decide whether to call tools at all - even with tools available,
if the LLM determines it has enough information to answer, it will respond directly
without calling tools. For example, a very general question might get answered
from the model's training data without consulting the database.
Additionally, the tool_calls field is None (not an empty list) when there are no tool calls,
so checking "if not assistant_message.tool_calls" is the right way to detect when
the loop should exit.
```
