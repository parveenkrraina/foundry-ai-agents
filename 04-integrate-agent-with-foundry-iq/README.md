# Lab 04 — Integrate Agent with Foundry Knowledge (IQ)

## Overview

This lab demonstrates how to build a **multi-turn chat client** that connects to a pre-existing Azure AI Foundry agent that is grounded in a **knowledge base** (Foundry IQ). Instead of creating the agent in code, the agent is deployed and configured in the Azure AI Foundry portal — complete with its knowledge index built from product catalogue documents. The Python client simply connects to that agent by name and conducts a conversation.

The scenario is *Contoso Outdoors*: a retail brand that sells backpacks, camping accessories, and tents. Users can ask natural-language questions about products and the agent answers using the uploaded product documentation, surfacing citations to the source documents.

---

## Project Structure

```
04-integrate-agent-with-foundry-iq/
├── data/
│   ├── contoso-backpacks-guide.md          # Backpacks product guide (Markdown)
│   ├── contoso-camping-accessories.md      # Camping accessories guide (Markdown)
│   ├── contoso-tents-catalog.md            # Tents catalogue (Markdown)
│   └── contoso-products/
│       ├── contoso-backpacks-guide.pdf     # Backpacks product guide (PDF)
│       ├── contoso-camping-accessories.pdf # Camping accessories guide (PDF)
│       └── contoso-tents-catalog.pdf       # Tents catalogue (PDF)
└── Python/
    ├── agent_client.py    # Multi-turn chat client
    └── requirements.txt   # Python dependencies
```

---

## Prerequisites

- Python 3.11 or later
- An Azure AI Foundry project with:
  - A deployed chat model (e.g., `gpt-4o`)
  - An agent already created and configured with the Contoso product files uploaded to its knowledge index
- A `.env` file in the `Python/` folder:

```env
PROJECT_ENDPOINT=<your Azure AI Foundry project endpoint>
AGENT_NAME=<the name of your pre-deployed agent>
```

> **Note:** Unlike other labs, this lab does **not** create or delete the agent in code. The agent must be set up in the Azure AI Foundry portal beforehand, with the documents from the `data/` folder uploaded to its knowledge base.

---

## Setup & Run

```bash
cd 04-integrate-agent-with-foundry-iq/Python
pip install -r requirements.txt
python agent_client.py
```

**Example questions to ask:**
- `What backpacks do you sell?`
- `What are the key features of the TrailMaster X4 tent?`
- `Do you have any camping accessories for cooking?`
- `history` — displays the full conversation history
- `quit` — ends the session

---

## Knowledge Base Documents

The `data/` folder contains six documents (three in Markdown, three in PDF) that form the agent's knowledge base. When uploaded to Azure AI Foundry, these documents are indexed and the agent can retrieve relevant passages to answer user questions.

| Document | Contents |
|----------|----------|
| `contoso-backpacks-guide` | Product descriptions, features, and specifications for Contoso backpacks |
| `contoso-camping-accessories` | Guide to camping accessories including cookware, lighting, and navigation tools |
| `contoso-tents-catalog` | Full tent catalogue with sizes, materials, and seasonal ratings |

> Both `.md` and `.pdf` versions are provided. Azure AI Foundry supports indexing either format.

---

## File Walkthrough: `agent_client.py`

### 1. Configuration & Validation

```python
load_dotenv()
project_endpoint = os.getenv("PROJECT_ENDPOINT")
agent_name = os.getenv("AGENT_NAME")

if not project_endpoint or not agent_name:
    raise ValueError("PROJECT_ENDPOINT and AGENT_NAME must be set in .env file")
```

The script fails fast with a clear error if either required environment variable is missing. This avoids confusing authentication errors later in the flow.

### 2. Authentication

```python
credential = DefaultAzureCredential(
    exclude_environment_credential=True,
    exclude_managed_identity_credential=True
)
```

`DefaultAzureCredential` is used with two providers disabled so it falls through to **Azure CLI credential** — the most convenient option for local development. In a deployed environment you would remove these exclusions to allow managed identity authentication.

### 3. Client Initialization

```python
project_client = AIProjectClient(credential=credential, endpoint=project_endpoint)
openai_client = project_client.get_openai_client()
```

- `AIProjectClient` is the top-level client for the Azure AI Foundry project.
- `get_openai_client()` returns an OpenAI-compatible client pre-configured with the project's endpoint and credential. All conversations and responses go through this client.

### 4. Connecting to the Pre-Deployed Agent

```python
agent = project_client.agents.get(agent_name=agent_name)
print(f"Connected to agent: {agent.name} (id: {agent.id})")
```

Rather than creating an agent, `agents.get(agent_name=...)` retrieves the metadata for an agent that was already deployed in the Foundry portal. The returned object provides the agent's `name` and `id`, which are needed when making responses.

### 5. Creating a Conversation

```python
conversation = openai_client.conversations.create(items=[])
print(f"Created conversation (id: {conversation.id})")
```

Each run of the script creates a fresh conversation. The `conversation.id` is a server-side session identifier that maintains message history and context across multiple turns.

### 6. `send_message_to_agent(user_message)` — Core Function

This function encapsulates the full request-response cycle for a single conversation turn.

#### Step 1: Add user message to conversation

```python
openai_client.conversations.items.create(
    conversation_id=conversation.id,
    items=[{"type": "message", "role": "user", "content": user_message}],
)
```

The user's message is appended to the server-side conversation. This maintains full context so the agent can answer follow-up questions coherently.

#### Step 2: Create a response

```python
response = openai_client.responses.create(
    conversation=conversation.id,
    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
    input=""
)
```

- `conversation=conversation.id` tells the model to use the full history of the conversation.
- `extra_body={"agent_reference": ...}` routes the request to the specific named agent in Foundry, which has the knowledge base configured.
- `input=""` is empty because the user message was already added to the conversation in the previous step.

#### Step 3: Handle MCP approval requests (if any)

```python
for item in response.output:
    if hasattr(item, 'type') and item.type == 'mcp_approval_request':
        approval_request = item
        break
```

If the agent has any MCP tools that require approval, this section intercepts those requests. The user is prompted to approve or deny the action:

```python
approval_input = input("Approve this action? (yes/no): ").strip().lower()
approval_response = {
    "type": "mcp_approval_response",
    "approval_request_id": approval_request.id,
    "approve": approval_input in ['yes', 'y']
}
```

The approval response is added to the conversation and a new `responses.create(...)` call retrieves the final answer.

#### Step 4: Display response and citations

```python
if response and response.output_text:
    print(f"{response_text}\n")
    if hasattr(response, 'citations') and response.citations:
        print("\nSources:")
        for citation in response.citations:
            print(f"  - {citation.content ...}")
```

The agent's text answer is printed. If the agent retrieved information from its knowledge base, citation objects are attached to the response. Each citation references the source document and the relevant passage, enabling users to verify answers.

#### Step 5: Client-side history tracking

```python
conversation_history.append({"role": "user", "content": user_message})
conversation_history.append({"role": "assistant", "content": response_text})
```

A local list tracks the conversation for the `history` display command. Server-side history is maintained by the conversation ID.

### 7. `display_conversation_history()`

Formats and prints the full local `conversation_history` list in a readable `USER: / ASSISTANT:` format. Triggered by typing `history` at the prompt.

### 8. `main()` — Interactive Loop

```python
while True:
    user_input = input("You: ").strip()
    if user_input.lower() == 'quit':
        break
    if user_input.lower() == 'history':
        display_conversation_history()
        continue
    send_message_to_agent(user_input)
```

A simple REPL that reads user input, handles the built-in commands (`quit`, `history`), and delegates everything else to `send_message_to_agent(...)`.

---

## How Knowledge Grounding Works

```
User question
      │
      ▼
Azure AI Foundry Agent
      │
      ├─► Searches knowledge index (Foundry IQ)
      │         │
      │         └─► Retrieves relevant passages from product docs
      │
      ▼
Model generates answer grounded in retrieved context
      │
      ▼
Response with output_text + citations
      │
      ▼
Client prints answer + source citations
```

The model does **not** rely solely on its training data. It retrieves specific product details from the indexed documents and generates answers that are grounded in — and traceable to — the source material.

---

## Dependencies (`requirements.txt`)

| Package | Version | Purpose |
|---------|---------|---------|
| `azure-ai-projects` | `2.0.0b4` | Azure AI Projects SDK (agents, conversations, responses) |
| `azure-identity` | latest | Azure authentication (`DefaultAzureCredential`) |
| `python-dotenv` | latest | Load `.env` file variables |
