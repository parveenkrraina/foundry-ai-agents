# Lab 02 — Agent with Custom Tools

## Overview

This lab demonstrates how to build an **Azure AI Foundry agent that calls custom Python functions** (tools) during a conversation. The project is themed around *Contoso Observatories* — a fictional telescope rental service that allows users to query upcoming astronomical events, calculate booking costs, and generate formal observation session reports.

The agent is created and managed using the `azure-ai-projects` SDK. Three custom `FunctionTool` definitions are registered with the agent. When the model decides a tool is needed to answer a user question, it emits a `function_call` item in its response. The host application (`agent.py`) intercepts that call, executes the matching Python function, and returns the result to the model, which then produces a final natural-language answer.

---

## Project Structure

```
02-agent-custom-tools/
└── Python/
    ├── agent.py                  # Agent creation, conversation loop, function-call dispatcher
    ├── functions.py              # The three tool functions used by the agent
    ├── requirements.txt          # Python dependencies
    └── data/
        ├── events.txt            # Astronomical events catalogue
        ├── telescope_rates.txt   # Hourly rates per telescope tier
        └── priority_multipliers.txt  # Cost multipliers per priority level
```

---

## Prerequisites

- Python 3.11 or later
- An Azure AI Foundry project with a deployed chat model (e.g., `gpt-4o`)
- A `.env` file in the `Python/` folder:

```env
PROJECT_ENDPOINT=<your Azure AI Foundry project endpoint>
MODEL_DEPLOYMENT_NAME=<your model deployment name>
```

---

## Setup & Run

```bash
cd 02-agent-custom-tools/Python
pip install -r requirements.txt
python agent.py
```

Type questions at the `USER:` prompt. Type `quit` to exit.

**Example prompts to try:**
- `What events are visible from north_america?`
- `How much would it cost to use a premium telescope for 3 hours at high priority?`
- `Generate a report for the Jupiter-Venus Conjunction in north_america using an advanced telescope for 2 hours at normal priority for observer John Smith.`

---

## File-by-File Explanation

### `agent.py`

This is the main entry point and orchestrates the entire agent interaction lifecycle.

#### Imports & Setup

```python
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FunctionTool, PromptAgentDefinition
from azure.identity import DefaultAzureCredential
from openai.types.responses.response_input_param import FunctionCallOutput, ResponseInputParam
from functions import next_visible_event, calculate_observation_cost, generate_observation_report
```

- `AIProjectClient` — authenticates with Azure AI Foundry and provides access to agents and the OpenAI client.
- `FunctionTool` — describes a tool's name, description, and JSON Schema for its parameters.
- `PromptAgentDefinition` — bundles the model deployment, system instructions, and tools together.
- `DefaultAzureCredential` — authenticates using the ambient Azure credential chain (CLI login, managed identity, etc.).
- `FunctionCallOutput` / `ResponseInputParam` — typed structures for feeding tool results back to the model.

#### Tool Definitions

Three `FunctionTool` objects are defined inline. Each one follows the JSON Schema format and uses `strict=True` (disables optional properties and requires all declared parameters):

| Tool name | Parameters | What it does |
|-----------|-----------|--------------|
| `next_visible_event` | `location` (string) | Returns the next upcoming astronomical event visible from the given continent |
| `calculate_observation_cost` | `telescope_tier`, `hours`, `priority` | Calculates the total booking cost including priority multiplier |
| `generate_observation_report` | `event_name`, `location`, `telescope_tier`, `hours`, `priority`, `observer_name` | Generates and saves a formatted `.txt` report |

#### Agent Creation

```python
agent = project_client.agents.create_version(
    agent_name="astronomy-agent",
    definition=PromptAgentDefinition(
        model=model_deployment,
        instructions="You are an astronomy observations assistant...",
        tools=[event_tool, cost_tool, report_tool],
    ),
)
```

`create_version` creates a new versioned agent in Foundry. A new version number is assigned automatically each time.

#### Conversation Loop

1. A **conversation** is created with `openai_client.conversations.create()`. This maintains message history server-side.
2. Each user message is added to the conversation via `openai_client.conversations.items.create(...)`.
3. `openai_client.responses.create(...)` is called with the conversation ID and any pending function outputs (`input_list`). The model returns a response that may contain:
   - **`function_call`** items — the model wants to call a tool
   - **Text output** — the final answer

#### Function Call Dispatcher

```python
for item in response.output:
    if item.type == "function_call":
        if item.name == "next_visible_event":
            result = next_visible_event(**json.loads(item.arguments))
        elif item.name == "calculate_observation_cost":
            result = calculate_observation_cost(**json.loads(item.arguments))
        elif item.name == "generate_observation_report":
            result = generate_observation_report(**json.loads(item.arguments))
        input_list.append(FunctionCallOutput(type="function_call_output", call_id=item.call_id, output=result))
```

After all function calls are resolved, `responses.create(...)` is called again with the populated `input_list`, and `previous_response_id` links it to the previous turn. The model uses the tool results to generate the final answer.

#### Cleanup

```python
project_client.agents.delete_version(agent_name=agent.name, agent_version=agent.version)
```

Deletes the specific agent version from Foundry when the session ends.

---

### `functions.py`

Contains the three Python functions that are called when the agent requests a tool. All functions return JSON strings so that the model receives structured data it can reason over.

#### Data Loading Helpers

```python
def _load_events(file_path: str = "data/events.txt") -> list
def _load_rates(file_path: str) -> dict
```

These are module-level helpers that read the data files once at import time and store the parsed results in module-level constants (`EVENTS`, `TELESCOPE_RATES`, `PRIORITY_MULTIPLIERS`).

`_load_events` parses the pipe-delimited `events.txt` file into a list of tuples `(name, type, sortable_date_int, date_str, locations_set)` and sorts them chronologically.

`_load_rates` parses both `telescope_rates.txt` and `priority_multipliers.txt` into dictionaries keyed by the rate/priority name.

#### `next_visible_event(location: str) -> str`

```python
today = int(datetime.now().strftime("%m%d"))
for name, event_type, date, date_str, locs in EVENTS:
    if loc in locs and date >= today:
        return json.dumps({"event": name, "type": event_type, "date": date_str, "visible_from": sorted(locs)})
```

- Converts today's date to a sortable integer (e.g., `0424` for April 24).
- Iterates through the pre-sorted event list and returns the first event visible from the given location that hasn't passed yet.
- Returns a JSON string with `event`, `type`, `date`, and `visible_from` fields.

#### `calculate_observation_cost(telescope_tier, hours, priority) -> str`

```python
base_cost = TELESCOPE_RATES[tier] * hours
multiplier = PRIORITY_MULTIPLIERS[pri]
total_cost = base_cost * multiplier
```

- Looks up the hourly rate and priority multiplier from the loaded data.
- Returns a JSON object containing `telescope_tier`, `hours`, `hourly_rate`, `priority`, `priority_multiplier`, `base_cost`, and `total_cost`.
- Returns a JSON error object for unknown tier, priority, or non-positive hours.

#### `generate_observation_report(...) -> str`

- Calls `calculate_observation_cost` and `next_visible_event` internally.
- Formats the results into a structured ASCII report with sections for date, observer, event info, telescope booking, and cost summary.
- Saves the report to a `.txt` file named `report_<event>_<timestamp>.txt` in the working directory.
- Returns a JSON object with `{"status": "Report generated", "file": "<filename>"}`.

---

### `data/events.txt`

Pipe-delimited flat file. Each row describes one astronomical event:

```
<event_name>|<event_type>|<MM-DD>|<location1>;<location2>;...
```

Example:
```
Jupiter-Venus Conjunction|conjunction|05-01|north_america;south_america;europe;asia;africa;australia
```

**Valid location values:** `north_america`, `south_america`, `europe`, `africa`, `asia`, `australia`, `antarctica`

---

### `data/telescope_rates.txt`

Pipe-delimited mapping of telescope tier to hourly rate (USD):

```
standard|50.00
advanced|120.00
premium|300.00
```

---

### `data/priority_multipliers.txt`

Pipe-delimited mapping of priority level to cost multiplier:

```
low|1.00
normal|1.25
high|1.75
urgent|2.50
```

---

## How the Tool-Call Cycle Works

```
USER INPUT
    │
    ▼
conversations.items.create()      ← add user message to conversation
    │
    ▼
responses.create(input=input_list) ← ask model; input_list may contain previous tool results
    │
    ▼
response.output contains function_call items?
    │   YES                        NO
    ▼                              ▼
Execute matching Python fn    Print response.output_text → USER
    │
    ▼
Append FunctionCallOutput to input_list
    │
    ▼
responses.create(input=input_list, previous_response_id=...)
    │
    ▼
Print response.output_text → USER
```

---

## Dependencies (`requirements.txt`)

| Package | Purpose |
|---------|---------|
| `python-dotenv` | Load `.env` file variables |
| `azure-identity` | `DefaultAzureCredential` for Azure authentication |
| `azure-ai-projects==2.0.0b3` | Azure AI Projects SDK (agents, conversations, responses) |
| `openai` | OpenAI typed response models (`FunctionCallOutput`, etc.) |
