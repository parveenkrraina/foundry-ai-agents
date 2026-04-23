# Develop AI Agents on Azure (AI-3026)

Solution files for the **AI-3026: Develop AI Agents on Azure** course. Each lab folder contains a complete Python solution demonstrating a specific aspect of building AI agents with Azure AI Foundry and the Azure AI Projects SDK.

## Prerequisites

- An Azure subscription with an Azure AI Foundry project
- Python 3.11 or later
- A `.env` file in each lab's `Python/` folder with the following variables (where required):

```env
PROJECT_ENDPOINT=<your Azure AI Foundry project endpoint>
MODEL_DEPLOYMENT_NAME=<your deployed model name, e.g. gpt-4o>
AGENT_NAME=<your agent name>          # lab 04 only
```

---

## Lab Structure

### Lab 02 — Agent with Custom Tools (`02-agent-custom-tools/`)

**Scenario:** *Contoso Observatories* – an agent that helps users plan astronomical observation sessions.

The agent is built using `azure-ai-projects` and exposes three custom `FunctionTool` definitions backed by Python functions:

| Tool | Description |
|------|-------------|
| `next_visible_event` | Returns the next visible astronomical event for a given continent |
| `calculate_observation_cost` | Calculates telescope booking cost based on tier, hours, and priority |
| `generate_observation_report` | Generates and saves a session report to a `.txt` file |

**Key files:**

| File | Purpose |
|------|---------|
| `agent.py` | Creates the agent, handles the tool-call loop, and prints the final response |
| `functions.py` | Implements the three tool functions; reads data from `data/` |
| `data/events.txt` | Pipe-delimited astronomical events with dates and visible locations |
| `data/telescope_rates.txt` | Hourly rates per telescope tier (standard / advanced / premium) |
| `data/priority_multipliers.txt` | Cost multipliers per priority level (low / normal / high) |

**Install dependencies:**

```bash
cd 02-agent-custom-tools/Python
pip install -r requirements.txt
python agent.py
```

---

### Lab 03 — MCP Integration (`03-mcp-integration/`)

**Scenario:** Two complementary examples showing how to integrate Model Context Protocol (MCP) tools with Azure AI agents.

#### 3a — Remote MCP via `MCPTool` (`agent.py`)

Connects to the public Microsoft Learn API MCP server (`https://learn.microsoft.com/api/mcp`) and asks the agent an Azure CLI question. Demonstrates the `MCPTool` class and the MCP approval-request/response flow.

#### 3b — Local MCP Server via stdio (`server.py` + `client.py`)

- **`server.py`**: A local MCP server (built with `fastmcp`) that exposes two tools: `get_inventory_levels` and `get_weekly_sales` for a personal care products inventory.
- **`client.py`**: An async MCP client that connects to the local server via stdio, dynamically wraps its tools as `FunctionTool` objects, creates an *inventory-agent* in Azure AI Foundry, and runs a conversation loop that processes function calls and returns recommendations (restock vs. clearance).

**Key files:**

| File | Purpose |
|------|---------|
| `agent.py` | Remote MCP tool example using `MCPTool` |
| `server.py` | Local MCP server exposing inventory and sales tools |
| `client.py` | MCP client that wraps local MCP tools for use with a Foundry agent |

**Install dependencies:**

```bash
cd 03-mcp-integration/Python
pip install -r requirements.txt

# Run the remote MCP example
python agent.py

# Run the local MCP client/server example
python client.py
```

---

### Lab 04 — Integrate Agent with Foundry Knowledge (`04-integrate-agent-with-foundry-iq/`)

**Scenario:** *Contoso Outdoors* – a multi-turn chat client that connects to a pre-deployed Azure AI Foundry agent grounded in a product knowledge base.

The `agent_client.py` script:
1. Connects to an existing Foundry agent by name (set via `AGENT_NAME`)
2. Creates a persistent conversation
3. Sends user messages and streams back responses
4. Displays citations from the knowledge base when present

The `data/` folder contains the Contoso Outdoors product catalogue (Markdown and PDF) used to populate the agent's knowledge index:

- `contoso-backpacks-guide.md` / `.pdf`
- `contoso-camping-accessories.md` / `.pdf`
- `contoso-tents-catalog.md` / `.pdf`

**Install dependencies:**

```bash
cd 04-integrate-agent-with-foundry-iq/Python
pip install -r requirements.txt
python agent_client.py
```

---

### Lab 05 — Agent Orchestration (`05-agent-orchestration/`)

**Scenario:** Customer feedback triage pipeline using a sequential multi-agent workflow.

`agents.py` uses the `agent-framework` SDK's `SequentialBuilder` to chain three specialised agents:

| Agent | Role |
|-------|------|
| `summarizer` | Condenses customer feedback into one sentence |
| `classifier` | Labels feedback as *Positive*, *Negative*, or *Feature request* |
| `action` | Suggests the appropriate next action based on the summary and classification |

The agents run in sequence using `workflow.run(..., stream=True)`, and the final outputs from all three agents are printed to the console.

**Install dependencies:**

```bash
cd 05-agent-orchestration/Python
pip install -r requirements.txt
python agents.py
```

---

### Lab 07 — Agent Framework (`07-agent-framework/`)

**Scenario:** *Contoso Expenses* – an agent that reads an expenses data file and submits a formatted expense claim by email.

`agent-framework.py` uses the `agent-framework` SDK's `Agent` class with `AzureOpenAIResponsesClient`. It defines a custom `@tool` function (`submit_claim`) that simulates sending an email to `expenses@contoso.com` with itemised expenses.

The agent:
1. Reads `data.txt` (CSV-format expense entries)
2. Prompts the user for an instruction
3. Calls the `submit_claim` tool to format and "send" the email
4. Confirms the action to the user

**Key files:**

| File | Purpose |
|------|---------|
| `agent-framework.py` | Main agent script with `@tool` decorator example |
| `data.txt` | Sample expense data (date, description, amount) |

**Install dependencies:**

```bash
cd 07-agent-framework/python
pip install -r requirements.txt
python agent-framework.py
```

---

## Repository Structure

```
foundry-ai-agents/
├── 02-agent-custom-tools/Python/   # Custom FunctionTool agent (Contoso Observatories)
├── 03-mcp-integration/Python/      # MCP tool integration (remote and local)
├── 04-integrate-agent-with-foundry-iq/  # Knowledge-grounded agent (Contoso Outdoors)
├── 05-agent-orchestration/Python/  # Sequential multi-agent orchestration
└── 07-agent-framework/python/      # Agent Framework SDK with custom tools
```

## Key Technologies

| Technology | Usage |
|-----------|-------|
| [Azure AI Projects SDK](https://learn.microsoft.com/azure/ai-foundry/) | Core SDK for creating and managing agents (`azure-ai-projects`) |
| [Azure AI Foundry](https://ai.azure.com) | Hosted model deployments and agent management |
| [Model Context Protocol (MCP)](https://modelcontextprotocol.io) | Tool integration via remote and local MCP servers |
| [agent-framework](https://pypi.org/project/agent-framework/) | High-level orchestration SDK with `@tool`, `SequentialBuilder`, and `AzureAIAgentClient` |
| [Azure Identity](https://learn.microsoft.com/python/api/azure-identity/) | Authentication via `DefaultAzureCredential` / `AzureCliCredential` |
