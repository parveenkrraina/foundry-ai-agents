# Lab 03 — MCP Integration

## Overview

This lab demonstrates two distinct ways to integrate **Model Context Protocol (MCP)** tools with Azure AI Foundry agents:

1. **`agent.py`** — Connects an agent to a *remote* MCP server (the Microsoft Learn API) using the built-in `MCPTool` class. The agent automatically calls the server and the host application handles the MCP approval flow.

2. **`server.py` + `client.py`** — Builds a *local* MCP server using `fastmcp` and an async MCP client that dynamically wraps the server's tools as `FunctionTool` objects for a Foundry agent. This pattern is useful when you want to host your own tool server alongside your agent application.

Both examples share the same underlying pattern: a Foundry agent is created at runtime, tools are registered, a conversation loop runs, tool calls are dispatched and results returned, and the agent is deleted on exit.

---

## Project Structure

```
03-mcp-integration/
└── Python/
    ├── agent.py          # Remote MCP via MCPTool (Microsoft Learn API)
    ├── server.py         # Local MCP server (inventory + sales tools)
    ├── client.py         # Local MCP client that wraps server tools for a Foundry agent
    └── requirements.txt  # Python dependencies
```

---

## Prerequisites

- Python 3.11 or later
- An Azure AI Foundry project with a deployed chat model
- A `.env` file in the `Python/` folder:

```env
PROJECT_ENDPOINT=<your Azure AI Foundry project endpoint>
MODEL_DEPLOYMENT_NAME=<your model deployment name>
```

---

## Setup

```bash
cd 03-mcp-integration/Python
pip install -r requirements.txt
```

---

## Example 1 — Remote MCP Tool (`agent.py`)

### What it does

Creates a Foundry agent that is connected to the **Microsoft Learn API MCP server** at `https://learn.microsoft.com/api/mcp`. The agent uses this remote server to answer Azure CLI questions. Because MCP tool calls are treated as potentially sensitive operations, the code demonstrates the approval-request/response flow.

### Run

```bash
python agent.py
```

### File Walkthrough

#### Imports

```python
from azure.ai.projects.models import PromptAgentDefinition, MCPTool
from openai.types.responses.response_input_param import McpApprovalResponse, ResponseInputParam
```

- `MCPTool` — a first-class tool type in the Azure AI Projects SDK that points to a remote MCP server. The agent SDK handles the underlying MCP protocol.
- `McpApprovalResponse` — a typed item used to approve or deny a pending MCP tool call.

#### MCPTool Initialization

```python
mcp_tool = MCPTool(
    server_label="api-specs",
    server_url="https://learn.microsoft.com/api/mcp",
    require_approval="always",
)
```

| Parameter | Description |
|-----------|-------------|
| `server_label` | A short identifier for this server, referenced when matching approval requests |
| `server_url` | The HTTPS URL of the remote MCP server |
| `require_approval` | `"always"` means every MCP tool call must be approved by the host app before executing |

#### Agent Creation

```python
agent = project_client.agents.create_version(
    agent_name="MyAgent",
    definition=PromptAgentDefinition(
        model=model_deployment,
        instructions="You are a helpful agent that can use MCP tools...",
        tools=[mcp_tool],
    ),
)
```

A versioned agent is created in Foundry with the MCP tool attached.

#### First Request

```python
response = openai_client.responses.create(
    conversation=conversation.id,
    input="Give me the Azure CLI commands to create an Azure Container App with a managed identity.",
    extra_body={"agent": {"name": agent.name, "type": "agent_reference"}},
)
```

The user's question is sent directly as the `input` string. The model determines it needs to call the MCP server to look up the CLI syntax, so the response contains `mcp_approval_request` items instead of a text answer.

#### Approval Flow

```python
for item in response.output:
    if item.type == "mcp_approval_request":
        if item.server_label == "api-specs" and item.id:
            input_list.append(
                McpApprovalResponse(
                    type="mcp_approval_response",
                    approve=True,
                    approval_request_id=item.id,
                )
            )
```

The code automatically approves all pending approval requests from the `api-specs` server. Each `McpApprovalResponse` is keyed to a specific `approval_request_id` so the server knows exactly which call was approved. In a production app, you could prompt the user before approving.

#### Second Request (with approvals)

```python
response = openai_client.responses.create(
    input=input_list,
    previous_response_id=response.id,
    extra_body={"agent": {"name": agent.name, "type": "agent_reference"}},
)
print(f"\nAgent response: {response.output_text}")
```

The approvals are submitted as the `input` to a new response call, linked back via `previous_response_id`. The model now executes the MCP calls and returns the final answer.

#### Cleanup

```python
project_client.agents.delete_version(agent_name=agent.name, agent_version=agent.version)
```

---

## Example 2 — Local MCP Server & Client (`server.py` + `client.py`)

### What it does

Runs a self-hosted MCP server that exposes inventory and sales data as tools. An async MCP client connects to the server over stdio, discovers its tools at runtime, wraps them as `FunctionTool` definitions, and registers them with a Foundry agent. The agent then answers inventory management questions using those tools.

### Run

```bash
# The client automatically starts the server as a subprocess
python client.py
```

---

### `server.py` — Local MCP Server

#### What it does

Implements a minimal MCP server using the `fastmcp` library. It exposes two tools that return mock inventory and sales data for a personal care products business.

#### Server Initialization

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP(name="Inventory")
```

`FastMCP` is a lightweight wrapper from the `mcp` package that auto-discovers `@mcp.tool()` decorated functions and handles the MCP protocol details (JSON-RPC, tool listing, tool invocation).

#### `get_inventory_levels() -> dict`

```python
@mcp.tool()
def get_inventory_levels() -> dict:
    """Returns current inventory for all products."""
    return {
        "Moisturizer": 6, "Shampoo": 8, "Body Spray": 28, ...
    }
```

Returns a dictionary mapping product names to current stock levels. The docstring is used as the tool description that the model sees.

#### `get_weekly_sales() -> dict`

```python
@mcp.tool()
def get_weekly_sales() -> dict:
    """Returns number of units sold last week."""
    return {
        "Moisturizer": 22, "Shampoo": 18, "Body Spray": 3, ...
    }
```

Returns units sold in the past week per product. Together with `get_inventory_levels`, this gives the agent enough data to make restock or clearance recommendations.

#### Server Entry Point

```python
if __name__ == "__main__":
    mcp.run()
```

Starts the server in stdio mode by default, listening for MCP JSON-RPC messages on stdin/stdout.

---

### `client.py` — Local MCP Client

#### What it does

An async application that:
1. Spawns `server.py` as a subprocess and connects to it via stdio
2. Queries the server for its available tools
3. Wraps those tools as `FunctionTool` definitions for a Foundry agent
4. Creates the agent and runs an interactive conversation loop
5. Dispatches function calls back to the MCP server via the active session

#### Imports

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
```

- `StdioServerParameters` — defines how to launch the server subprocess (`command`, `args`)
- `stdio_client` — an async context manager that spawns the subprocess and returns a stdio transport
- `ClientSession` — the MCP session object that provides `list_tools()` and `call_tool()` methods

#### `connect_to_server(exit_stack)` — Async Function

```python
server_params = StdioServerParameters(command="python", args=["server.py"])
stdio_transport = await exit_stack.enter_async_context(stdio_client(server_params))
stdio, write = stdio_transport
session = await exit_stack.enter_async_context(ClientSession(stdio, write))
await session.initialize()
```

- `AsyncExitStack` ensures both the transport and session are properly closed when the application exits.
- `session.initialize()` performs the MCP handshake (protocol version negotiation).
- `session.list_tools()` returns the metadata for all tools registered on the server.

#### Dynamic Tool Wrapping

```python
def make_tool_func(tool_name):
    async def tool_func(**kwargs):
        result = await session.call_tool(tool_name, kwargs)
        return result
    tool_func.__name__ = tool_name
    return tool_func

functions_dict = {tool.name: make_tool_func(tool.name) for tool in tools}
```

A factory function creates one async callable per MCP tool. These callables are stored in `functions_dict` and invoked when the Foundry agent emits a `function_call` for the corresponding tool name.

The tools are also wrapped as `FunctionTool` JSON Schema definitions for the Foundry agent:

```python
for tool in tools:
    function_tool = FunctionTool(
        name=tool.name,
        description=tool.description,
        parameters={"type": "object", "properties": {}, "additionalProperties": False},
        strict=True,
    )
```

> **Note:** Since `get_inventory_levels` and `get_weekly_sales` take no parameters, the `properties` object is empty.

#### Agent Creation & Instructions

```python
agent = project_client.agents.create_version(
    agent_name="inventory-agent",
    definition=PromptAgentDefinition(
        model=model_deployment,
        instructions="""
            You are an inventory assistant. Here are some general guidelines:
            - Recommend restock if item inventory < 10 and weekly sales > 15
            - Recommend clearance if item inventory > 20 and weekly sales < 5
        """,
        tools=mcp_function_tools,
    ),
)
```

The agent's instructions encode the business rules. The model applies these rules to the data returned by the MCP tools and provides actionable recommendations.

#### Conversation Loop

The loop follows the same pattern as lab 02:
1. User types a prompt → added to the conversation
2. `responses.create(...)` returns; response may contain `function_call` items
3. For each `function_call`, `functions_dict[name](**kwargs)` is awaited (calls the MCP server)
4. Results are appended as `FunctionCallOutput` items
5. `responses.create(input=input_list, previous_response_id=...)` fetches the final answer

#### Cleanup

```python
project_client.agents.delete_version(agent_name=agent.name, agent_version=agent.version)
```

---

## How the Two Approaches Compare

| | Remote MCP (`agent.py`) | Local MCP stdio (`client.py`) |
|--|-------------------------|-------------------------------|
| **Server location** | External HTTPS URL | Subprocess on the same machine |
| **Tool class** | `MCPTool` (native Foundry) | `FunctionTool` (dynamic wrapper) |
| **Approval handling** | `McpApprovalResponse` typed items | Not required (function call pattern) |
| **Discovery** | At agent creation time | At runtime via `session.list_tools()` |
| **Best for** | SaaS MCP servers, public APIs | Local data, private tools, testing |

---

## Dependencies (`requirements.txt`)

| Package | Purpose |
|---------|---------|
| `python-dotenv` | Load `.env` file variables |
| `azure-identity` | Azure authentication |
| `azure-ai-projects==2.0.0b3` | Agents, conversations, responses |
| `openai` | Typed response models |
| `mcp` | MCP client and server library (`ClientSession`, `FastMCP`, etc.) |
| `uvicorn` | ASGI server (available if running server in HTTP mode) |
