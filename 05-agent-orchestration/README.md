# Lab 05 — Agent Orchestration

## Overview

This lab demonstrates **multi-agent orchestration** using the `agent-framework` SDK. Instead of a single agent handling a complex task, multiple specialised agents are chained together in a **sequential pipeline** — the output of each agent becomes the input to the next.

The scenario is a customer feedback triage system. A piece of raw customer feedback text passes through three agents in sequence:

1. **Summarizer** — distils the feedback into one concise sentence
2. **Classifier** — categorises it as *Positive*, *Negative*, or *Feature request*
3. **Action** — recommends the appropriate next step based on the summary and classification

All three agents share the same underlying model deployment (configured via environment variables) and are orchestrated using `SequentialBuilder` from the `agent-framework` library.

---

## Project Structure

```
05-agent-orchestration/
└── Python/
    ├── agents.py          # Multi-agent orchestration script
    └── requirements.txt   # Python dependencies
```

---

## Prerequisites

- Python 3.11 or later
- An Azure AI Foundry project with a deployed chat model
- Azure CLI logged in (`az login`) — the script uses `AzureCliCredential`
- A `.env` file in the `Python/` folder:

```env
PROJECT_ENDPOINT=<your Azure AI Foundry project endpoint>
MODEL_DEPLOYMENT_NAME=<your model deployment name>
```

---

## Setup & Run

```bash
cd 05-agent-orchestration/Python
pip install -r requirements.txt
python agents.py
```

The script runs non-interactively. The hardcoded feedback text is processed through the pipeline and the outputs from all three agents are printed to the console.

---

## File Walkthrough: `agents.py`

### Imports

```python
from agent_framework import Message
from agent_framework.azure import AzureAIAgentClient
from agent_framework.orchestrations import SequentialBuilder
from azure.identity import AzureCliCredential
```

- `AzureAIAgentClient` — a high-level async client from `agent-framework` that wraps the Azure AI Projects SDK. It manages connections to the Foundry project and provides the `as_agent(...)` factory method.
- `SequentialBuilder` — an orchestration builder that chains agents so that each agent's output is automatically passed as input to the next.
- `Message` — the data model returned by the framework representing an agent's response, with `role`, `author_name`, and `text` fields.
- `AzureCliCredential` — authenticates using the local Azure CLI login. No environment credential or managed identity is needed for local development.

### Agent Instructions

Each agent receives a focused, single-responsibility system prompt:

#### Summarizer

```python
summarizer_instructions = """
Summarize the customer's feedback in one short sentence. Keep it neutral and concise.
Example output:
App crashes during photo upload.
User praises dark mode feature.
"""
```

The examples in the prompt guide the model to produce consistently short, useful summaries that the downstream classifier and action agents can work with effectively.

#### Classifier

```python
classifier_instructions = """
Classify the feedback as one of the following: Positive, Negative, or Feature request.
"""
```

Deliberately minimal — the classifier receives the summary from the previous step and only needs to output one of three labels.

#### Action

```python
action_instructions = """
Based on the summary and classification, suggest the next action in one short sentence.
Example output:
Escalate as a high-priority bug for the mobile team.
Log as positive feedback to share with design and marketing.
Log as enhancement request for product backlog.
"""
```

The action agent receives the full accumulated context (original feedback + summary + classification) and suggests a concrete next step. The examples constrain the output format.

### Client Setup

```python
credential = AzureCliCredential()
async with AzureAIAgentClient(credential=credential) as chat_client:
```

`AzureAIAgentClient` is used as an async context manager. It establishes the connection to the Foundry project and cleans up resources when the `async with` block exits. The `AzureCliCredential` uses the token from `az login` — no `.env` variables are needed for credentials (only `PROJECT_ENDPOINT` and `MODEL_DEPLOYMENT_NAME` are read from `.env`).

### Creating Agents

```python
summarizer = chat_client.as_agent(
    instructions=summarizer_instructions,
    name="summarizer",
)
classifier = chat_client.as_agent(
    instructions=classifier_instructions,
    name="classifier",
)
action = chat_client.as_agent(
    instructions=action_instructions,
    name="action",
)
```

`as_agent(...)` creates lightweight agent objects that hold the instructions and name. These are not persisted in Foundry at this stage — they are configuration objects passed to the orchestration builder.

### Sample Feedback

```python
feedback = """
  I reached out to your customer support yesterday because I couldn't access my account.
  The representative responded almost immediately, was polite and professional, and fixed
  the issue within minutes. Honestly, it was one of the best support experiences I've ever had.
"""
```

A positive customer support experience. You can replace this string with any feedback text to test different outcomes.

### Building the Sequential Workflow

```python
workflow = SequentialBuilder(participants=[summarizer, classifier, action]).build()
```

`SequentialBuilder` takes an ordered list of agents as `participants`. When `build()` is called, it creates a `Workflow` object that will execute each agent in turn, passing the output of one as the input to the next.

**Execution order:**
```
Input: "Customer feedback: <feedback text>"
    │
    ▼
[summarizer]  → "The representative resolved an account access issue promptly and professionally."
    │
    ▼
[classifier]  → "Positive"
    │
    ▼
[action]      → "Log as positive feedback to share with customer support leadership."
```

### Running the Workflow

```python
outputs: list[list[Message]] = []
async for event in workflow.run(f"Customer feedback: {feedback}", stream=True):
    if event.type == "output":
        outputs.append(cast(list[Message], event.data))
```

- `workflow.run(input, stream=True)` executes the pipeline asynchronously and yields events as they complete.
- Events with `type == "output"` carry the messages produced by each agent turn.
- All output events are collected into the `outputs` list. Because of sequential chaining, `outputs[-1]` contains the final consolidated set of messages from all three agents.

### Displaying Results

```python
if outputs:
    for i, msg in enumerate(outputs[-1], start=1):
        name = msg.author_name or ("assistant" if msg.role == "assistant" else "user")
        print(f"{'-' * 60}\n{i:02d} [{name}]\n{msg.text}")
```

The last output event is printed with a numbered, labelled format:

```
------------------------------------------------------------
01 [summarizer]
The representative resolved an account access issue promptly and professionally.
------------------------------------------------------------
02 [classifier]
Positive
------------------------------------------------------------
03 [action]
Log as positive feedback to share with customer support leadership.
```

---

## Sequential Orchestration — How It Works

```
SequentialBuilder([summarizer, classifier, action])
        │
        ▼
workflow.run("Customer feedback: ...")
        │
        ▼
 Turn 1: summarizer receives original input
        │  output: one-sentence summary
        ▼
 Turn 2: classifier receives summary as input
        │  output: category label
        ▼
 Turn 3: action receives summary + category as input
        │  output: recommended next action
        ▼
 outputs[-1] = [Message(summarizer), Message(classifier), Message(action)]
```

Each agent only sees the output of the previous agent (plus the accumulated context), not the full prompt history of all previous agents. This keeps each agent focused on its single responsibility.

---

## Customising the Pipeline

To process different feedback, replace the `feedback` string with any text. To add or remove stages, modify the `participants` list in `SequentialBuilder`:

```python
# Add a fourth agent (e.g., priority scorer)
priority = chat_client.as_agent(
    instructions="Rate the urgency of this feedback from 1 (low) to 5 (critical).",
    name="priority",
)
workflow = SequentialBuilder(participants=[summarizer, classifier, action, priority]).build()
```

---

## Dependencies (`requirements.txt`)

| Package | Version | Purpose |
|---------|---------|---------|
| `python-dotenv` | latest | Load `.env` file variables |
| `azure-identity` | latest | `AzureCliCredential` for local authentication |
| `agent-framework` | `1.0.0rc3` | High-level SDK with `AzureAIAgentClient`, `SequentialBuilder`, `Message` |
| `opentelemetry-semantic-conventions-ai` | `0.4.13` | OpenTelemetry AI semantic conventions (required by `agent-framework`) |
