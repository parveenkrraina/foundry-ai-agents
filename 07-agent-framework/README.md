# Lab 07 — Agent Framework

## Overview

This lab demonstrates how to build an **AI agent with a custom tool** using the `agent-framework` SDK's `@tool` decorator pattern. The project is themed around *Contoso Expenses* — an AI assistant that reads a CSV expense data file and, at the user's instruction, formats and submits an expense claim by email.

Unlike the labs that use `azure-ai-projects` directly, this lab showcases the higher-level `agent-framework` abstractions: the `Agent` class, the `AzureOpenAIResponsesClient`, and the `@tool` decorator. The tool decorator approach is more ergonomic than defining `FunctionTool` JSON Schema objects manually — the framework introspects the function signature and `Annotated` type hints to build the schema automatically.

---

## Project Structure

```
07-agent-framework/
└── python/
    ├── agent-framework.py   # Main agent script with @tool decorator example
    ├── data.txt             # Sample expense data (CSV format)
    └── requirements.txt     # Python dependencies
```

---

## Prerequisites

- Python 3.11 or later
- An Azure AI Foundry project with a deployed chat model
- Azure CLI logged in (`az login`) — the script uses `AzureCliCredential`
- A `.env` file in the `python/` folder:

```env
PROJECT_ENDPOINT=<your Azure AI Foundry project endpoint>
MODEL_DEPLOYMENT_NAME=<your model deployment name>
```

---

## Setup & Run

```bash
cd 07-agent-framework/python
pip install -r requirements.txt
python agent-framework.py
```

The script reads `data.txt`, displays its contents, and prompts you for an instruction. Type something like:

```
Submit my expense claim
```

The agent will format the expenses and call the `submit_claim` tool, which prints the email content to the console.

---

## File Walkthrough

### `data.txt` — Expense Data

```
date,description,amount
07-Mar-2025,taxi,24.00
07-Mar-2025,dinner,65.50
07-Mar-2025,hotel,125.90
```

A simple CSV file with three columns: `date`, `description`, and `amount`. This is the raw data the agent uses to build the expense claim email. In a real application this could be read from an accounting system, spreadsheet, or database.

---

### `agent-framework.py` — Main Script

#### Imports

```python
from agent_framework import tool, Agent
from agent_framework.azure import AzureOpenAIResponsesClient
from azure.identity import AzureCliCredential
from pydantic import Field
from typing import Annotated
```

- `tool` — a decorator from `agent-framework` that transforms a regular Python function into a tool the agent can call. It reads type annotations and `Field` metadata to generate the JSON Schema description automatically.
- `Agent` — the main agent class. It is used as an async context manager and exposes a `run(messages)` method.
- `AzureOpenAIResponsesClient` — an `agent-framework` client that connects to Azure AI Foundry using the OpenAI Responses API. It accepts a credential, a deployment name, and a project endpoint.
- `AzureCliCredential` — authenticates using the locally cached token from `az login`.
- `Annotated` + `Field` — from `pydantic`; used to attach human-readable descriptions to each tool parameter. These descriptions are passed to the model so it understands what each parameter represents.

#### `main()` — Entry Point

```python
async def main():
    os.system('cls' if os.name=='nt' else 'clear')

    script_dir = Path(__file__).parent
    file_path = script_dir / 'data.txt'
    with file_path.open('r') as file:
        data = file.read() + "\n"

    user_prompt = input(f"Here is the expenses data in your file:\n\n{data}\n\nWhat would you like me to do with it?\n\n")

    await process_expenses_data(user_prompt, data)
```

- Uses `Path(__file__).parent` to locate `data.txt` relative to the script file, making the script portable regardless of the working directory.
- Displays the file contents to the user for context before asking for a prompt.

#### `process_expenses_data(prompt, expenses_data)` — Agent Execution

```python
async def process_expenses_data(prompt, expenses_data):
    credential = AzureCliCredential()
    async with (
        Agent(
            client=AzureOpenAIResponsesClient(
                credential=credential,
                deployment_name=os.getenv("MODEL_DEPLOYMENT_NAME"),
                project_endpoint=os.getenv("PROJECT_ENDPOINT"),
            ),
            instructions="""You are an AI assistant for expense claim submission.
                        At the user's request, create an expense claim and use the plug-in function
                        to send an email to expenses@contoso.com with the subject 'Expense Claim'
                        and a body that contains itemized expenses with a total.
                        Then confirm to the user that you've done so. Don't ask for any more information
                        from the user, just use the data provided to create the email.""",
            tools=[submit_claim],
        ) as agent,
    ):
```

- `AzureOpenAIResponsesClient` wraps the connection to the Foundry project's OpenAI-compatible endpoint. It handles authentication and routing to the correct model deployment.
- `Agent(...)` is created with a `client`, `instructions`, and a `tools` list. The `tools` list accepts the decorated `submit_claim` function directly — no manual JSON Schema definition needed.
- The `async with Agent(...) as agent` pattern ensures the agent's resources (any background connections or sessions) are properly cleaned up when the block exits.

#### Running the Agent

```python
prompt_messages = [f"{prompt}: {expenses_data}"]
response = await agent.run(prompt_messages)
print(f"\n# Agent:\n{response}")
```

- The user's instruction and the expense data are combined into a single message string.
- `agent.run(messages)` sends the messages, handles any tool calls transparently (the framework calls `submit_claim` automatically when the model requests it), and returns the final text response.
- Unlike the direct SDK pattern in earlier labs, the `agent-framework` handles the tool-call loop internally. You do not need to write a manual loop to dispatch `function_call` items.

#### `submit_claim` — The Tool Function

```python
@tool(approval_mode="never_require")
def submit_claim(
    to: Annotated[str, Field(description="Who to send the email to")],
    subject: Annotated[str, Field(description="The subject of the email.")],
    body: Annotated[str, Field(description="The text body of the email.")]):
        print("\nTo:", to)
        print("Subject:", subject)
        print(body, "\n")
```

This is the key pattern in this lab.

**`@tool(approval_mode="never_require")`**

The `@tool` decorator registers this function as an agent tool. `approval_mode="never_require"` means the tool is called automatically whenever the model requests it, without prompting the user for approval first (as opposed to `"always_require"` or `"auto"`).

**`Annotated[str, Field(description="...")]`**

Each parameter uses Python's `Annotated` type hint combined with a `pydantic.Field`. The `agent-framework` reads these descriptions to build the JSON Schema that is passed to the model. This replaces the verbose `FunctionTool(parameters={...})` pattern from earlier labs:

| Manual FunctionTool approach | @tool decorator approach |
|-----------------------------|--------------------------|
| Define a separate `FunctionTool` object | Just decorate the function |
| Write JSON Schema for all parameters | Use `Annotated[type, Field(description="...")]` |
| Maintain tool definition and function separately | Single source of truth |

**What the tool simulates:**

In a real application, `submit_claim` would call an email API (e.g., Microsoft Graph, SendGrid). Here it simply prints the `to`, `subject`, and `body` fields to the console to demonstrate that the agent correctly formatted an expense claim email with itemised entries and a calculated total.

**Expected console output:**

```
To: expenses@contoso.com
Subject: Expense Claim
Body:
  Expense Claim

  Date        | Description | Amount
  ------------|-------------|-------
  07-Mar-2025 | Taxi        | $24.00
  07-Mar-2025 | Dinner      | $65.50
  07-Mar-2025 | Hotel       | $125.90
  -----------------------------------
  Total                     | $215.40
```

*(Exact formatting is generated by the model based on the data and instructions.)*

---

## How the `@tool` Decorator Pattern Works

```
@tool(approval_mode="never_require")
def submit_claim(to, subject, body):
    ...
        │
        ▼
agent-framework introspects Annotated type hints
        │
        ▼
Generates JSON Schema: {type: object, properties: {to: ..., subject: ..., body: ...}}
        │
        ▼
Schema is registered with the Agent
        │
        ▼
Model receives tool description during inference
        │
        ▼
Model emits function_call → agent-framework dispatches to submit_claim()
        │
        ▼
Return value (None here) is returned to the model
        │
        ▼
Model generates confirmation message → printed to console
```

The framework handles steps 4–7 automatically inside `agent.run(...)`. You only need to write the decorated function and call `run()`.

---

## Comparison: `agent-framework` vs Direct SDK

| Aspect | Direct SDK (`azure-ai-projects`) | `agent-framework` |
|--------|----------------------------------|-------------------|
| Tool definition | `FunctionTool(name, description, parameters={...})` | `@tool` decorator on a Python function |
| Tool call dispatching | Manual loop over `response.output` | Automatic inside `agent.run()` |
| Multi-turn handling | Manual `responses.create(previous_response_id=...)` | Automatic |
| Agent orchestration | Manual | `SequentialBuilder`, `ParallelBuilder`, etc. |
| Best for | Fine-grained control | Rapid development, cleaner code |

---

## Dependencies (`requirements.txt`)

| Package | Version | Purpose |
|---------|---------|---------|
| `python-dotenv` | latest | Load `.env` file variables |
| `azure-identity` | latest | `AzureCliCredential` for local authentication |
| `agent-framework` | `1.0.0b260212` | High-level agent SDK with `@tool`, `Agent`, `AzureOpenAIResponsesClient` |
| `opentelemetry-semantic-conventions-ai` | `0.4.13` | OpenTelemetry AI semantic conventions (required by `agent-framework`) |
