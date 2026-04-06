# Module 6: AI Agents

> **Scenario**: ShopAssist has grown from a simple chatbot into a sophisticated assistant that can browse inventory, process returns, check shipping, and escalate to humans. But now leadership wants it to **autonomously handle complex multi-step workflows** — researching a product complaint across multiple databases, coordinating with warehouse and finance agents, and deciding when to loop in a human. You need to understand how AI agents *think*, *plan*, *act*, and *remember* — and how to make them reliable enough for production.

---

## Screen 1: Agent Architectures — From Simple Loops to Multi-Agent Systems

### What Is an AI Agent?

An AI agent is an LLM that can **observe**, **reason**, **plan**, and **act** in a loop — using tools to interact with the real world until a goal is achieved.

```
┌──────────────────────────────────────────────────────────────────┐
│                    THE AGENT LOOP                                │
│                                                                  │
│   User Goal: "Process return for order #5521"                    │
│                                                                  │
│         ┌──────────┐                                             │
│         │ OBSERVE  │ ◀─── Environment / Tool Results             │
│         └────┬─────┘                                             │
│              │                                                   │
│              ▼                                                   │
│         ┌──────────┐                                             │
│         │  REASON  │ ◀─── LLM thinks about what to do next      │
│         └────┬─────┘                                             │
│              │                                                   │
│              ▼                                                   │
│         ┌──────────┐                                             │
│         │   PLAN   │ ◀─── Break into steps / select tool         │
│         └────┬─────┘                                             │
│              │                                                   │
│              ▼                                                   │
│         ┌──────────┐                                             │
│         │   ACT    │ ──── Call tool / Return response             │
│         └────┬─────┘                                             │
│              │                                                   │
│              ▼                                                   │
│         Done? ── No ──▶ Loop back to OBSERVE                     │
│           │                                           │
│          Yes                                                     │
│           │                                                      │
│           ▼                                                      │
│      Return Final Answer                                         │
└──────────────────────────────────────────────────────────────────┘
```

### Single-Agent Architecture (Tool Loop)

The simplest architecture: one LLM, a set of tools, a loop.

```python
from openai import OpenAI
import json

client = OpenAI()

# Define tools the agent can use
tools = [
    {
        "type": "function",
        "function": {
            "name": "lookup_order",
            "description": "Look up order details by order ID",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string", "description": "The order ID"}
                },
                "required": ["order_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "process_refund",
            "description": "Process a refund for an order",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string"},
                    "amount": {"type": "number"},
                    "reason": {"type": "string"}
                },
                "required": ["order_id", "amount", "reason"]
            }
        }
    }
]

# The single-agent tool loop
def run_agent(user_message: str) -> str:
    messages = [
        {"role": "system", "content": "You are ShopAssist. Use tools to help customers."},
        {"role": "user", "content": user_message}
    ]

    while True:
        response = client.chat.completions.create(
            model="gpt-4o", messages=messages, tools=tools
        )
        msg = response.choices[0].message

        # If no tool calls, we're done
        if not msg.tool_calls:
            return msg.content

        # Execute each tool call
        messages.append(msg)
        for tool_call in msg.tool_calls:
            result = execute_tool(tool_call.function.name,
                                  json.loads(tool_call.function.arguments))
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result)
            })
    # Loop continues until LLM returns a natural language response
```

### Multi-Agent Architectures

When a single agent gets overwhelmed (too many tools, too many domains), split into multiple specialized agents.

```
┌──────────────────────────────────────────────────────────────────────┐
│                   MULTI-AGENT PATTERNS                               │
│                                                                      │
│  1. SUPERVISOR                    2. HIERARCHICAL                    │
│     ┌──────────┐                    ┌──────────┐                     │
│     │Supervisor│                    │  Manager  │                    │
│     └──┬───┬───┘                    └──┬─────┬──┘                    │
│        │   │                           │     │                       │
│     ┌──┘   └──┐                  ┌─────┘     └─────┐                 │
│     ▼         ▼                  ▼                  ▼                 │
│  ┌──────┐ ┌──────┐          ┌──────────┐      ┌──────────┐          │
│  │Agent │ │Agent │          │Supervisor│      │Supervisor│          │
│  │  A   │ │  B   │          │  Team 1  │      │  Team 2  │          │
│  └──────┘ └──────┘          └──┬────┬──┘      └──┬────┬──┘          │
│                                │    │             │    │              │
│  LLM routes to the           ▼    ▼            ▼    ▼              │
│  right specialist         [A1] [A2]         [B1] [B2]              │
│                                                                      │
│  3. SEQUENTIAL                    4. DAG-BASED                       │
│                                                                      │
│  ┌────┐   ┌────┐   ┌────┐       ┌────┐                              │
│  │ A  │──▶│ B  │──▶│ C  │       │ A  │                              │
│  └────┘   └────┘   └────┘       └─┬──┘                              │
│                                    │                                 │
│  Pipeline: each agent           ┌──┴──┐                              │
│  passes output to next          ▼     ▼                              │
│                              ┌────┐ ┌────┐                           │
│                              │ B  │ │ C  │  ← run in parallel       │
│                              └──┬─┘ └──┬─┘                           │
│                                 └──┬───┘                             │
│                                    ▼                                 │
│                                 ┌────┐                               │
│                                 │ D  │  ← merge results             │
│                                 └────┘                               │
└──────────────────────────────────────────────────────────────────────┘
```

### When to Use Each Architecture

| Architecture | Use When | Example | Complexity |
|---|---|---|---|
| Single Agent (Tool Loop) | < 10 tools, one domain | Simple Q&A bot | Low |
| Supervisor | Multiple domains, need routing | Customer support triage | Medium |
| Hierarchical | Large orgs, nested teams | Enterprise workflow | High |
| Sequential (Pipeline) | Ordered processing stages | Research → Write → Edit | Low-Med |
| DAG-based | Parallel + dependencies | Analyze data from 3 APIs | High |

### 💡 Interview Insight

> **Q: How do you decide between single-agent and multi-agent architectures?**
>
> "I follow the **'cognitive load' heuristic**: if a single agent needs more than ~10 tools or must reason across unrelated domains, it starts making poor tool choices — just like a person juggling too many responsibilities. At that point, I split into specialists. For ShopAssist, I'd use a **supervisor pattern** — one routing agent that classifies the intent (shipping, billing, returns) and delegates to domain experts. Each specialist has only 3-5 tools and a focused system prompt. The supervisor doesn't solve problems — it just decides *who* should. I avoid hierarchical patterns unless we genuinely have nested team structures, because every layer adds latency and potential routing errors."

---

## Screen 2: Tool Use — Function Calling & MCP

### Function Calling (OpenAI / Anthropic)

Function calling lets the LLM **output structured JSON** that maps to a tool you define. The LLM never executes the tool — it just says *which* tool to call with *what* arguments.

```python
# OpenAI function calling flow:
#
# 1. You define tools with JSON Schema
# 2. LLM returns tool_calls with name + arguments
# 3. YOU execute the function
# 4. You send results back to the LLM
# 5. LLM generates final response

# Anthropic equivalent:
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[
        {
            "name": "lookup_order",
            "description": "Look up order details by order ID",
            "input_schema": {                          # ← Anthropic uses input_schema
                "type": "object",
                "properties": {
                    "order_id": {"type": "string"}
                },
                "required": ["order_id"]
            }
        }
    ],
    messages=[{"role": "user", "content": "Where is order #5521?"}]
)

# Response contains tool_use blocks instead of tool_calls
for block in response.content:
    if block.type == "tool_use":
        print(f"Tool: {block.name}, Input: {block.input}")
        # Tool: lookup_order, Input: {'order_id': '5521'}
```

### Model Context Protocol (MCP)

MCP is an **open standard** (by Anthropic) that lets agents discover and use tools from external servers — like a USB port for AI tools.

```
┌──────────────────────────────────────────────────────────────────┐
│                    MCP ARCHITECTURE                              │
│                                                                  │
│  ┌─────────────┐         ┌─────────────┐                        │
│  │  MCP Client  │◀──────▶│  MCP Server  │                        │
│  │ (Your Agent) │  stdio  │  (Tool Host) │                        │
│  │              │  or SSE │              │                        │
│  └─────────────┘         └──────┬───────┘                        │
│                                  │                                │
│                    ┌─────────────┼─────────────┐                 │
│                    │             │             │                  │
│                    ▼             ▼             ▼                  │
│              ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│              │  Tools   │ │Resources │ │ Prompts  │             │
│              │          │ │          │ │          │             │
│              │lookup_   │ │file://   │ │template  │             │
│              │order()   │ │db://     │ │library   │             │
│              │process_  │ │api://    │ │          │             │
│              │refund()  │ │          │ │          │             │
│              └──────────┘ └──────────┘ └──────────┘             │
│                                                                  │
│  Transport Options:                                              │
│  ─────────────────                                               │
│  • stdio  — subprocess, local tools (fast, simple)              │
│  • SSE    — HTTP streaming, remote servers (scalable)           │
│  • Streamable HTTP — newest, replaces SSE (2025+)               │
└──────────────────────────────────────────────────────────────────┘
```

```python
# Building an MCP Server (tool provider)
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ShopAssist Tools")

@mcp.tool()
def lookup_order(order_id: str) -> dict:
    """Look up order details by order ID.

    Args:
        order_id: The unique order identifier (e.g., 'ORD-5521')
    """
    # In production, query your database
    return {
        "order_id": order_id,
        "status": "shipped",
        "tracking": "1Z999AA10123456784",
        "estimated_delivery": "2026-04-09"
    }

@mcp.tool()
def process_refund(order_id: str, amount: float, reason: str) -> dict:
    """Process a refund for an order.

    Args:
        order_id: The order to refund
        amount: Refund amount in USD
        reason: Reason for the refund
    """
    return {"status": "approved", "refund_id": f"REF-{order_id}-001"}

# Run: python server.py
# Or use as stdio subprocess from your MCP client

# ──────────────────────────────────────────────────
# Connecting an MCP Client
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def use_mcp_tools():
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # TOOL DISCOVERY — agent learns what tools are available
            tools = await session.list_tools()
            for tool in tools.tools:
                print(f"  {tool.name}: {tool.description}")

            # TOOL EXECUTION
            result = await session.call_tool(
                "lookup_order",
                arguments={"order_id": "ORD-5521"}
            )
            print(result.content)
```

### MCP vs Direct Function Calling

| Feature | Direct Function Calling | MCP |
|---|---|---|
| Tool discovery | Hardcoded in prompt | Dynamic at runtime |
| Tool location | Same process | Any server (local/remote) |
| Schema validation | Manual | Built-in via protocol |
| Multi-model support | Provider-specific | Universal standard |
| Composability | Rewire per project | Plug & play servers |
| Ecosystem | DIY | Growing marketplace |

### 💡 Interview Insight

> **Q: What is MCP and why does it matter?**
>
> "MCP — Model Context Protocol — is an open standard that decouples **tool definition** from **agent implementation**. Think of it like USB for AI tools: any MCP client (Claude, LangChain, your custom agent) can connect to any MCP server (database tools, API tools, file tools) without custom integration. Before MCP, every agent framework had its own tool format — you'd rewrite the same 'query database' tool for LangChain, CrewAI, and OpenAI's SDK separately. With MCP, you write it once as an MCP server, and any client can discover and call it. The transport layer supports **stdio** (local subprocess, great for dev) and **SSE/Streamable HTTP** (remote, great for production). The protocol also exposes **resources** (data the agent can read) and **prompts** (reusable templates), not just tools."

---

## Screen 3: Planning & Reasoning — How Agents Think

### Task Decomposition

Complex goals must be broken into subtasks. The LLM acts as a **planner**.

```python
# Plan-and-Execute Pattern
PLANNER_PROMPT = """
You are a planning agent. Given a complex task, break it into
a numbered list of subtasks. Each subtask should be:
- Atomic (one action)
- Have clear inputs and outputs
- Executable by a tool or sub-agent

Task: {task}

Output your plan as JSON:
{{"steps": [
    {{"id": 1, "action": "...", "tool": "...", "depends_on": []}},
    {{"id": 2, "action": "...", "tool": "...", "depends_on": [1]}}
]}}
"""

# Example plan for: "Investigate why customer #8842 was double-charged"
plan = {
    "steps": [
        {"id": 1, "action": "Look up customer profile",
         "tool": "lookup_customer", "depends_on": []},
        {"id": 2, "action": "Fetch all transactions for customer",
         "tool": "get_transactions", "depends_on": [1]},
        {"id": 3, "action": "Identify duplicate charges",
         "tool": "analyze_duplicates", "depends_on": [2]},
        {"id": 4, "action": "Check payment gateway logs",
         "tool": "query_payment_logs", "depends_on": [2]},
        {"id": 5, "action": "Determine root cause and draft resolution",
         "tool": "llm_reasoning", "depends_on": [3, 4]},
        {"id": 6, "action": "Process refund if confirmed duplicate",
         "tool": "process_refund", "depends_on": [5]}
    ]
}
```

```
┌──────────────────────────────────────────────────────────────────┐
│               PLAN-AND-EXECUTE WITH REPLANNING                   │
│                                                                  │
│  ┌──────────┐     ┌───────────┐     ┌──────────┐               │
│  │  PLAN    │────▶│  EXECUTE  │────▶│ EVALUATE │               │
│  │          │     │  Step N   │     │  Result  │               │
│  └──────────┘     └───────────┘     └────┬─────┘               │
│       ▲                                   │                      │
│       │              ┌────────────────────┤                      │
│       │              │                    │                      │
│       │         Step Failed?         All Done?                   │
│       │              │                    │                      │
│       │              ▼                   Yes                     │
│       │         ┌──────────┐              │                      │
│       └─────────│ REPLAN   │              ▼                      │
│                 │ (adjust  │        ┌──────────┐                │
│                 │  steps)  │        │  FINAL   │                │
│                 └──────────┘        │  ANSWER  │                │
│                                     └──────────┘                │
│                                                                  │
│  Key: The agent doesn't blindly follow the plan.                │
│  After each step, it evaluates results and can REPLAN           │
│  if something unexpected happens.                                │
└──────────────────────────────────────────────────────────────────┘
```

### Self-Reflection & Critique

After executing a step (or an entire plan), the agent evaluates its own work:

```python
REFLECTION_PROMPT = """
You just completed this action:
  Action: {action}
  Result: {result}

Evaluate:
1. Did the action succeed? (yes/no)
2. Is the result complete and accurate?
3. Does this change the remaining plan?
4. Should we backtrack and try a different approach?

If the result is insufficient, suggest a corrective action.
"""

# Implementing reflection in the agent loop
def execute_with_reflection(plan: list, tools: dict) -> str:
    completed = []
    remaining = list(plan)

    while remaining:
        step = remaining.pop(0)

        # Execute
        result = tools[step["tool"]](step["action"])

        # Reflect
        reflection = llm_reflect(step, result)

        if reflection["success"]:
            completed.append({"step": step, "result": result})
        else:
            # BACKTRACK: replan from current state
            new_plan = llm_replan(
                original_goal=plan[0]["action"],
                completed_steps=completed,
                failed_step=step,
                failure_reason=reflection["reason"]
            )
            remaining = new_plan  # Replace remaining steps
            continue

    return synthesize_answer(completed)
```

### 💡 Interview Insight

> **Q: How do you handle agent failures mid-plan?**
>
> "I implement a **plan-and-execute with dynamic replanning** pattern. The agent creates an initial plan, but after each step, a reflection module evaluates: did the tool return useful data? Is the result consistent with expectations? If a step fails — say, the payment gateway API times out — the agent doesn't just retry blindly. It **replans**: maybe it can get the same info from the transaction database instead. I also set a **maximum step count** (typically 10-15) to prevent infinite loops, and I implement **backtracking** — if the agent goes down a dead-end path, it can revert to the last known good state and try a different approach. This is where LangGraph's checkpointing shines: you can literally rewind the graph state."

---

## Screen 4: Memory for Agents — Making Agents Remember

### The Memory Taxonomy

Agents need different types of memory for different purposes:

```
┌──────────────────────────────────────────────────────────────────┐
│                    AGENT MEMORY TYPES                             │
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │  SHORT-TERM     │    │   LONG-TERM      │                     │
│  │  (Working)      │    │   (Persistent)   │                     │
│  │                 │    │                  │                     │
│  │ • Current       │    │ • Vector store   │                     │
│  │   conversation  │    │   of past chats  │                     │
│  │ • Last N msgs   │    │ • User prefs     │                     │
│  │ • Tool results  │    │ • Learned facts  │                     │
│  │   in progress   │    │                  │                     │
│  │                 │    │ Survives across  │                     │
│  │ Cleared each    │    │ sessions         │                     │
│  │ session         │    │                  │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │  EPISODIC       │    │   SEMANTIC       │                     │
│  │  (Experiences)  │    │   (Knowledge)    │                     │
│  │                 │    │                  │                     │
│  │ • "Last time    │    │ • "Refund policy │                     │
│  │   user asked    │    │   is 30 days"    │                     │
│  │   about X,      │    │ • "Store #442    │                     │
│  │   approach Y    │    │   closes at 9pm" │                     │
│  │   worked"       │    │ • Domain facts   │                     │
│  │ • Task traces   │    │                  │                     │
│  │ • Success/fail  │    │ Retrieved via    │                     │
│  │   patterns      │    │ similarity       │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                                                  │
│  ┌─────────────────────────────────────┐                        │
│  │  PROCEDURAL (Learned Workflows)     │                        │
│  │                                     │                        │
│  │ • "To process international         │                        │
│  │   returns, ALWAYS check customs     │                        │
│  │   status before initiating refund"  │                        │
│  │ • Stored as rules or mini-prompts   │                        │
│  │ • Updated when agent discovers      │                        │
│  │   new optimal workflows             │                        │
│  └─────────────────────────────────────┘                        │
└──────────────────────────────────────────────────────────────────┘
```

### Implementing Memory in Practice

```python
from datetime import datetime
from dataclasses import dataclass, field

@dataclass
class AgentMemory:
    """Multi-tier memory system for an AI agent."""

    # SHORT-TERM: current conversation (list of messages)
    conversation_buffer: list[dict] = field(default_factory=list)
    max_buffer_size: int = 20

    # LONG-TERM: vector store for semantic search over past interactions
    vector_store: object = None  # ChromaDB / Pinecone / Qdrant

    # EPISODIC: specific experiences with outcomes
    episodes: list[dict] = field(default_factory=list)

    def add_message(self, role: str, content: str) -> None:
        """Add to short-term memory with sliding window."""
        self.conversation_buffer.append({
            "role": role, "content": content,
            "timestamp": datetime.now().isoformat()
        })
        # Sliding window — keep last N messages
        if len(self.conversation_buffer) > self.max_buffer_size:
            # Summarize older messages before discarding
            old = self.conversation_buffer[:5]
            summary = self._summarize(old)
            self.conversation_buffer = (
                [{"role": "system", "content": f"Earlier context: {summary}"}]
                + self.conversation_buffer[5:]
            )

    def store_episode(self, task: str, steps: list, outcome: str,
                      success: bool) -> None:
        """Store an episodic memory — a specific experience."""
        episode = {
            "task": task,
            "steps_taken": steps,
            "outcome": outcome,
            "success": success,
            "timestamp": datetime.now().isoformat()
        }
        self.episodes.append(episode)
        # Also embed in vector store for later retrieval
        if self.vector_store:
            self.vector_store.add(
                documents=[f"Task: {task}\nOutcome: {outcome}"],
                metadatas=[{"success": success}],
                ids=[f"episode-{len(self.episodes)}"]
            )

    def recall_similar_episodes(self, current_task: str,
                                 k: int = 3) -> list[dict]:
        """Retrieve past episodes similar to the current task."""
        if not self.vector_store:
            return []
        results = self.vector_store.query(
            query_texts=[current_task], n_results=k
        )
        return results

    def get_procedural_rules(self) -> list[str]:
        """Return learned workflow rules."""
        # These could be stored in a DB or config
        return [
            "For refunds > $100, always require manager approval",
            "For international orders, check customs status first",
            "If payment gateway times out, retry once then use backup"
        ]
```

### Memory Selection Strategy

| Memory Type | Storage | Retrieval | Lifetime | Use Case |
|---|---|---|---|---|
| Short-term | In-memory list | Direct access | Single session | Current conversation |
| Long-term | Vector DB | Semantic search | Permanent | "Have we helped this user before?" |
| Episodic | Vector DB + metadata | Similarity + filters | Permanent | "What worked last time?" |
| Semantic | Knowledge base / RAG | Similarity search | Permanent | Domain facts and policies |
| Procedural | Config / rules DB | Rule matching | Updated over time | Learned best practices |

### 💡 Interview Insight

> **Q: How do you implement memory for a production agent?**
>
> "I use a **tiered memory architecture**. Short-term memory is the conversation buffer — I keep the last 15-20 messages and **summarize** older ones to stay within the context window. Long-term memory uses a vector store (Qdrant or Chroma) — every completed interaction gets embedded and stored, so the agent can recall 'we helped this customer with a similar issue 3 weeks ago.' Episodic memory stores specific task traces with outcomes: 'last time a double-charge investigation failed because we didn't check the payment gateway logs early enough.' This lets the agent **learn from experience** — not through fine-tuning, but through retrieval. The most overlooked type is **procedural memory** — rules the agent has learned, like 'always verify the order exists before attempting a refund.' I store these as retrievable rules that get injected into the system prompt for relevant tasks."

---

## Screen 5: Multi-Agent Frameworks Compared

### The Big Four

```python
# ─── LangGraph ────────────────────────────────────────
# Graph-based, maximum control, production-ready
from langgraph.graph import StateGraph, START, END

graph = StateGraph(AgentState)
graph.add_node("researcher", research_agent)
graph.add_node("writer", writing_agent)
graph.add_node("supervisor", supervisor_agent)
graph.add_edge(START, "supervisor")
# Full control over routing, state, persistence

# ─── CrewAI ───────────────────────────────────────────
# Role-based, high-level, easy multi-agent setup
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Research Analyst",
    goal="Find accurate product data",
    backstory="Expert data researcher with 10 years experience",
    tools=[search_tool, scrape_tool]
)
crew = Crew(agents=[researcher, writer], tasks=[...])
result = crew.kickoff()

# ─── AutoGen (Microsoft) ──────────────────────────────
# Conversation-based, agents talk to each other
from autogen import AssistantAgent, UserProxyAgent

assistant = AssistantAgent("assistant", llm_config=llm_config)
user_proxy = UserProxyAgent("user", code_execution_config={"work_dir": "coding"})
user_proxy.initiate_chat(assistant, message="Analyze sales data")

# ─── OpenAI Swarm ─────────────────────────────────────
# Lightweight, handoff-based, experimental
from swarm import Swarm, Agent

triage = Agent(name="Triage", instructions="Route to the right agent",
               functions=[transfer_to_billing, transfer_to_shipping])
billing = Agent(name="Billing", instructions="Handle billing inquiries")
client = Swarm()
response = client.run(agent=triage, messages=[...])
```

### Framework Comparison Table

| Feature | LangGraph | CrewAI | AutoGen | OpenAI Swarm |
|---|---|---|---|---|
| Architecture | Graph (nodes+edges) | Role-based crews | Conversation-based | Handoff-based |
| Control level | Maximum (you define everything) | High-level (roles+goals) | Medium (conversation rules) | Minimal (handoffs) |
| State management | Built-in TypedDict | Internal | Chat history | Minimal context vars |
| Persistence | Checkpointing, time-travel | Limited | Chat logs | None |
| Human-in-the-loop | `interrupt()` built-in | Configurable | UserProxyAgent | Not built-in |
| Streaming | Full support | Limited | Limited | Basic |
| Production readiness | High (LangGraph Platform) | Medium | Medium | Experimental |
| Learning curve | Steep (graph concepts) | Easy (natural language roles) | Medium | Very easy |
| Parallel execution | `Send()` API | Sequential by default | Group chat | Sequential |
| Best for | Complex workflows, production | Quick prototypes, creative tasks | Research, code generation | Simple routing demos |

### Decision Framework

```
                    Need multi-agent?
                         │
                    ┌────┴────┐
                   No        Yes
                    │         │
              Single Agent    │
              (tool loop)     │
                         ┌────┴────────────┐
                    Need fine-grained   Quick prototype /
                    control?            creative tasks?
                         │                    │
                    ┌────┴────┐              CrewAI
                   Yes       No
                    │         │
               LangGraph   Need code
                           execution?
                              │
                         ┌────┴────┐
                        Yes       No
                         │         │
                      AutoGen   OpenAI Swarm
                               (if simple routing)
```

### 💡 Interview Insight

> **Q: Compare LangGraph, CrewAI, and AutoGen. When would you pick each?**
>
> "**LangGraph** is my choice for **production systems** — it gives me explicit control over the execution graph, built-in persistence with checkpointing, human-in-the-loop via `interrupt()`, and proper state management. The tradeoff is complexity — you're building a state machine. **CrewAI** is my choice for **rapid prototyping** and creative tasks — defining agents by role and backstory is intuitive, and you can get a multi-agent system running in 20 lines. But it's harder to customize the execution flow. **AutoGen** shines for **code generation and research** tasks — agents have a natural conversation, and the UserProxyAgent can execute code in a sandbox. **Swarm** is experimental and best for understanding handoff patterns — I wouldn't use it in production. For ShopAssist in production, I'd use LangGraph because I need deterministic routing, approval workflows, and crash recovery."

---

## Screen 6: Agent Evaluation — Measuring What Matters

### The Five Dimensions of Agent Quality

```
┌──────────────────────────────────────────────────────────────────┐
│               AGENT EVALUATION FRAMEWORK                         │
│                                                                  │
│  ┌────────────────┐  Did the agent complete the task?            │
│  │ 1. COMPLETION  │  Metric: task_success_rate                   │
│  │    RATE        │  Target: > 85% for production                │
│  └────────────────┘                                              │
│                                                                  │
│  ┌────────────────┐  Did the agent call the RIGHT tools          │
│  │ 2. TOOL-USE    │  with the RIGHT arguments?                   │
│  │    ACCURACY    │  Metric: tool_precision, tool_recall         │
│  └────────────────┘                                              │
│                                                                  │
│  ┌────────────────┐  How many steps / tokens / time              │
│  │ 3. EFFICIENCY  │  did it take?                                │
│  │                │  Metric: steps_taken, total_tokens, latency  │
│  └────────────────┘                                              │
│                                                                  │
│  ┌────────────────┐  Did the agent stay within                   │
│  │ 4. SAFETY      │  authorized boundaries?                     │
│  │                │  Metric: unauthorized_action_rate = 0        │
│  └────────────────┘                                              │
│                                                                  │
│  ┌────────────────┐  Is the final answer correct                 │
│  │ 5. ANSWER      │  and well-formatted?                        │
│  │    QUALITY     │  Metric: factual_accuracy, format_adherence │
│  └────────────────┘                                              │
└──────────────────────────────────────────────────────────────────┘
```

```python
from dataclasses import dataclass

@dataclass
class AgentEvalResult:
    """Evaluation result for a single agent run."""
    task_id: str
    completed: bool
    correct: bool
    steps_taken: int
    tools_called: list[str]
    expected_tools: list[str]
    total_tokens: int
    latency_seconds: float
    unauthorized_actions: list[str]

    @property
    def tool_precision(self) -> float:
        """Of the tools called, how many were correct?"""
        if not self.tools_called:
            return 0.0
        correct = set(self.tools_called) & set(self.expected_tools)
        return len(correct) / len(self.tools_called)

    @property
    def tool_recall(self) -> float:
        """Of the expected tools, how many were called?"""
        if not self.expected_tools:
            return 1.0
        correct = set(self.tools_called) & set(self.expected_tools)
        return len(correct) / len(self.expected_tools)

    @property
    def is_safe(self) -> bool:
        return len(self.unauthorized_actions) == 0


def evaluate_agent(agent, test_cases: list[dict]) -> dict:
    """Run agent through test cases and compute aggregate metrics."""
    results = []
    for case in test_cases:
        result = agent.run(case["input"])
        eval_result = AgentEvalResult(
            task_id=case["id"],
            completed=result.completed,
            correct=judge_correctness(result.output, case["expected"]),
            steps_taken=result.step_count,
            tools_called=result.tool_log,
            expected_tools=case["expected_tools"],
            total_tokens=result.token_usage,
            latency_seconds=result.duration,
            unauthorized_actions=check_safety(result.action_log)
        )
        results.append(eval_result)

    return {
        "completion_rate": sum(r.completed for r in results) / len(results),
        "accuracy": sum(r.correct for r in results) / len(results),
        "avg_steps": sum(r.steps_taken for r in results) / len(results),
        "avg_tool_precision": sum(r.tool_precision for r in results) / len(results),
        "safety_violations": sum(not r.is_safe for r in results),
    }
```

### Key Benchmarks

| Benchmark | What It Tests | Domain | Notable Results |
|---|---|---|---|
| SWE-bench | Fix real GitHub issues | Software engineering | Top agents ~50% verified |
| GAIA | Real-world multi-step tasks | General assistant | Humans ~92%, best AI ~75% |
| HumanEval | Code generation | Programming | GPT-4o ~90%+ |
| WebArena | Web browsing tasks | Web navigation | Best agents ~35% |
| ToolBench | API tool usage | Tool calling | Varies by API complexity |
| AgentBench | Multi-domain agent tasks | General | Frontier models lead |

### 💡 Interview Insight

> **Q: How do you evaluate an AI agent beyond just 'does it work'?**
>
> "I evaluate across five dimensions. **Task completion rate** — does the agent finish the job? This should be >85% for production. **Tool-use accuracy** — precision (did it call only the right tools?) and recall (did it call all needed tools?). An agent that calls unnecessary tools wastes tokens and may cause side effects. **Efficiency** — how many steps and tokens? A 3-step solution is better than a 10-step one for the same task. **Safety** — zero unauthorized actions is non-negotiable; I log every tool call and audit for boundary violations. **Answer quality** — is the final response correct, well-formatted, and helpful? I build a test suite of ~50 representative tasks with expected tool sequences and ground-truth answers, then run it on every agent change, treating it like a regression test suite."

---

## Screen 7: Agent Safety & Reliability — Production Guardrails

### The Safety Stack

```
┌──────────────────────────────────────────────────────────────────┐
│                 AGENT SAFETY STACK                                │
│                                                                  │
│  Layer 5: AUDIT LOGGING                                          │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ Every tool call, every decision → immutable audit log  │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                  │
│  Layer 4: HUMAN-IN-THE-LOOP                                      │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ Require human approval for high-risk actions           │      │
│  │ (refunds > $100, account deletion, data export)        │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                  │
│  Layer 3: PERMISSION SYSTEM                                      │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ Agent can only call tools it's authorized for          │      │
│  │ Role-based: read-only agent vs. read-write agent       │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                  │
│  Layer 2: RATE LIMITING                                          │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ Max 5 tool calls/minute, max 15 steps per task         │      │
│  │ Prevents runaway loops and cost explosions             │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                  │
│  Layer 1: SANDBOXING                                             │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ Code execution in Docker/E2B containers                │      │
│  │ No filesystem access, no network (unless whitelisted)  │      │
│  └────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

### Implementing Safety in Code

```python
import time
import logging
from functools import wraps
from dataclasses import dataclass, field

logger = logging.getLogger("agent_safety")

# ─── Layer 1: Rate Limiting ──────────────────────────
@dataclass
class RateLimiter:
    max_calls: int = 5
    window_seconds: int = 60
    max_steps: int = 15
    _call_times: list = field(default_factory=list)
    _step_count: int = 0

    def check_rate(self) -> bool:
        now = time.time()
        self._call_times = [t for t in self._call_times
                            if now - t < self.window_seconds]
        if len(self._call_times) >= self.max_calls:
            raise RateLimitExceeded(
                f"Max {self.max_calls} tool calls per {self.window_seconds}s"
            )
        self._call_times.append(now)
        return True

    def check_steps(self) -> bool:
        self._step_count += 1
        if self._step_count > self.max_steps:
            raise MaxStepsExceeded(
                f"Agent exceeded {self.max_steps} steps — likely stuck in a loop"
            )
        return True


# ─── Layer 2: Permission System ──────────────────────
PERMISSIONS = {
    "read_only_agent": {"lookup_order", "get_customer", "search_products"},
    "support_agent": {"lookup_order", "get_customer", "search_products",
                      "process_refund", "update_order_status"},
    "admin_agent": {"*"}  # All tools
}

def check_permission(agent_role: str, tool_name: str) -> bool:
    allowed = PERMISSIONS.get(agent_role, set())
    if "*" in allowed or tool_name in allowed:
        return True
    logger.warning(f"BLOCKED: {agent_role} tried to call {tool_name}")
    raise PermissionDenied(f"{agent_role} cannot use {tool_name}")


# ─── Layer 3: Human-in-the-Loop ──────────────────────
HIGH_RISK_ACTIONS = {
    "process_refund": lambda args: args.get("amount", 0) > 100,
    "delete_account": lambda args: True,  # Always requires approval
    "export_data": lambda args: True,
}

async def maybe_require_approval(tool_name: str, args: dict) -> bool:
    check_fn = HIGH_RISK_ACTIONS.get(tool_name)
    if check_fn and check_fn(args):
        logger.info(f"APPROVAL REQUIRED: {tool_name}({args})")
        # In LangGraph, this would be interrupt()
        # In production, send to approval queue / Slack / dashboard
        approved = await request_human_approval(
            action=tool_name,
            details=args,
            timeout_minutes=30
        )
        if not approved:
            raise ActionDenied(f"Human denied {tool_name}")
    return True


# ─── Layer 4: Audit Logging ──────────────────────────
def audit_log(func):
    """Decorator that logs every tool call with full context."""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        call_id = generate_call_id()
        logger.info(f"[{call_id}] TOOL_CALL: {func.__name__} "
                     f"args={args} kwargs={kwargs}")
        try:
            result = await func(*args, **kwargs)
            logger.info(f"[{call_id}] TOOL_RESULT: success")
            return result
        except Exception as e:
            logger.error(f"[{call_id}] TOOL_ERROR: {e}")
            raise
    return wrapper


# ─── Layer 5: Graceful Degradation ───────────────────
async def safe_tool_call(tool_name: str, args: dict,
                          agent_role: str,
                          rate_limiter: RateLimiter) -> dict:
    """Execute a tool call with all safety layers."""
    try:
        # Layer 1: Rate limiting
        rate_limiter.check_rate()
        rate_limiter.check_steps()

        # Layer 2: Permission check
        check_permission(agent_role, tool_name)

        # Layer 3: Human approval for high-risk actions
        await maybe_require_approval(tool_name, args)

        # Execute the tool
        result = await execute_tool(tool_name, args)

        return {"status": "success", "result": result}

    except RateLimitExceeded:
        return {"status": "rate_limited",
                "message": "Too many tool calls — please wait"}
    except PermissionDenied as e:
        return {"status": "denied", "message": str(e)}
    except ActionDenied:
        return {"status": "human_denied",
                "message": "A human reviewer denied this action"}
    except Exception as e:
        logger.error(f"Unexpected error in {tool_name}: {e}")
        return {"status": "error",
                "message": "Tool failed — agent should try alternative"}
```

### Sandboxing Code Execution

```python
# E2B — cloud sandboxes for agent code execution
from e2b_code_interpreter import Sandbox

async def run_code_sandboxed(code: str) -> str:
    """Execute agent-generated code in an isolated sandbox."""
    sandbox = Sandbox()          # Spin up isolated container
    try:
        execution = sandbox.run_code(code)
        if execution.error:
            return f"Error: {execution.error.name}: {execution.error.value}"
        return "\n".join(str(r) for r in execution.results)
    finally:
        sandbox.kill()           # Always clean up

# Docker alternative for on-prem
# docker run --rm --network=none --memory=512m --cpus=1 \
#   --read-only python:3.12-slim python -c "print('sandboxed')"
```

### 💡 Interview Insight

> **Q: How do you make AI agents safe for production?**
>
> "I implement defense-in-depth with five layers. **Sandboxing**: any code execution runs in Docker or E2B containers — no filesystem access, no network unless whitelisted. **Rate limiting**: max tool calls per minute and max steps per task — this prevents runaway loops that could cost thousands in API calls. **Permissions**: agents have roles — a read-only analyst agent can't call `delete_account`. **Human-in-the-loop**: high-risk actions (refunds over $100, account changes, data exports) require human approval — in LangGraph this is `interrupt()`, in production it's an approval queue with a timeout. **Audit logging**: every single tool call, argument, and result is logged immutably — we can replay exactly what the agent did and why. The key principle is **graceful degradation**: if any safety check fails, the agent doesn't crash — it gets a structured error message and adapts its plan."

---

## Screen 8: Putting It All Together — Building a Production Agent

### ShopAssist v3: Full Agent Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│               SHOPASSIST v3 — PRODUCTION AGENT                       │
│                                                                      │
│  User: "I was charged twice for order #5521. I want a refund."       │
│                                                                      │
│  ┌───────────┐                                                       │
│  │ SUPERVISOR│     Short-term memory: current conversation           │
│  │   Agent   │     Long-term memory: "Customer had issue 2mo ago"   │
│  └─────┬─────┘     Procedural: "Check payment logs before refund"    │
│        │                                                             │
│   ┌────┴────────────┐                                                │
│   │  Route by intent │                                               │
│   └──┬──────┬───────┘                                                │
│      │      │                                                        │
│      ▼      ▼                                                        │
│  ┌──────┐ ┌──────────┐                                               │
│  │Billing│ │ Returns  │  ← Specialist agents                        │
│  │Agent  │ │ Agent    │     (3-5 tools each)                         │
│  └──┬───┘ └──────────┘                                               │
│     │                                                                │
│     │  Plan:                                                         │
│     │  1. lookup_order("5521")                                       │
│     │  2. get_transactions(customer_id)                              │
│     │  3. analyze_duplicates(transactions)                           │
│     │  4. ── REFLECT: confirmed duplicate ──                         │
│     │  5. process_refund("5521", 49.99)   ← Under $100, auto-approve│
│     │  6. notify_customer(email, refund_details)                     │
│     │                                                                │
│     │  Safety Stack:                                                 │
│     │  ✅ Rate limiter: 4/5 calls used                               │
│     │  ✅ Permission: billing_agent authorized                       │
│     │  ✅ Amount $49.99 < $100 threshold                             │
│     │  ✅ Audit log: 6 entries recorded                              │
│     │                                                                │
│     ▼                                                                │
│  "I've confirmed the duplicate charge of $49.99 on order #5521      │
│   and processed a refund. You'll see it in 3-5 business days."      │
└──────────────────────────────────────────────────────────────────────┘
```

```python
# Production agent with all the pieces connected
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated, Literal
from langgraph.graph.message import add_messages

class ShopAssistState(TypedDict):
    messages: Annotated[list, add_messages]
    intent: str
    plan: list[dict]
    current_step: int
    memory_context: str        # Retrieved from long-term memory
    safety_log: list[str]

# Node: Retrieve relevant memory before processing
def enrich_with_memory(state: ShopAssistState) -> dict:
    user_msg = state["messages"][-1].content
    # Search long-term memory for similar past interactions
    past_context = memory.recall_similar_episodes(user_msg, k=2)
    # Get procedural rules
    rules = memory.get_procedural_rules()
    context = f"Past interactions: {past_context}\nRules: {rules}"
    return {"memory_context": context}

# Node: Classify intent with memory context
def classify_intent(state: ShopAssistState) -> dict:
    response = llm.invoke(
        f"Context: {state['memory_context']}\n"
        f"Classify intent: {state['messages'][-1].content}\n"
        f"Options: billing, shipping, returns, general"
    )
    return {"intent": response.content.strip().lower()}

# Node: Create execution plan
def create_plan(state: ShopAssistState) -> dict:
    plan = llm.invoke(
        f"Create a step-by-step plan for this {state['intent']} task: "
        f"{state['messages'][-1].content}\n"
        f"Memory context: {state['memory_context']}"
    )
    return {"plan": parse_plan(plan.content), "current_step": 0}

# Build the graph
graph = StateGraph(ShopAssistState)
graph.add_node("memory", enrich_with_memory)
graph.add_node("classify", classify_intent)
graph.add_node("plan", create_plan)
graph.add_node("execute", execute_step)  # Runs one step at a time
graph.add_node("reflect", reflect_on_result)
graph.add_node("respond", generate_response)

graph.add_edge(START, "memory")
graph.add_edge("memory", "classify")
graph.add_edge("classify", "plan")
graph.add_edge("plan", "execute")
graph.add_edge("execute", "reflect")
graph.add_conditional_edges("reflect", should_continue,
    {"continue": "execute", "replan": "plan", "done": "respond"})
graph.add_edge("respond", END)

# Compile with persistence
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)
```

### Agent Design Checklist

```
✅ Architecture: Single-agent for simple, multi-agent for complex
✅ Tools: Well-defined schemas, MCP for reusability
✅ Planning: Task decomposition with replanning on failure
✅ Memory: Short-term buffer + long-term vector store + episodic recall
✅ Safety: Sandboxing → Rate limits → Permissions → HITL → Audit logs
✅ Evaluation: Test suite with completion rate, tool accuracy, safety metrics
✅ Reliability: Graceful degradation, max step limits, timeout handling
✅ Observability: Structured logging, tracing (LangSmith), metrics dashboard
```

### 💡 Interview Insight

> **Q: Walk me through designing an AI agent system from scratch.**
>
> "I start with the **task analysis**: what does the agent need to accomplish? What tools does it need? What are the failure modes? For ShopAssist, I'd start with a **single-agent prototype** with 5-6 tools, test it on 50 representative queries, and measure completion rate. If tool accuracy drops below 80% or the agent struggles with routing, I split into a **supervisor + specialists** architecture using LangGraph. Each specialist gets a focused set of tools and system prompt. I add **memory** incrementally: conversation buffer first, then long-term vector storage, then episodic learning. Safety is non-negotiable from day one: rate limiting, permissions, audit logging. I build an **evaluation harness** with test cases covering happy paths, edge cases, and adversarial inputs. Finally, I add human-in-the-loop for high-risk actions and deploy behind a feature flag for gradual rollout."

---

## Quiz: Test Your Agent Knowledge

**Q1: In a supervisor multi-agent architecture, what is the supervisor's primary role?**
- A) Execute all tool calls directly
- B) Route tasks to the appropriate specialist agent ✅
- C) Store all conversation memory
- D) Generate the final user-facing response

**Q2: What does MCP (Model Context Protocol) standardize?**
- A) The format of LLM training data
- B) How agents discover and call tools from external servers ✅
- C) The tokenization algorithm used by models
- D) The prompt template format across frameworks

**Q3: In the plan-and-execute pattern, what happens when a step fails?**
- A) The entire task is aborted immediately
- B) The same step is retried infinitely
- C) The agent replans based on completed steps and the failure reason ✅
- D) The agent skips the step and continues with the next one

**Q4: Which memory type stores "last time this approach worked for a similar task"?**
- A) Short-term (conversation buffer)
- B) Semantic (facts and knowledge)
- C) Episodic (specific experiences with outcomes) ✅
- D) Procedural (learned workflow rules)

**Q5: In the agent safety stack, what is the purpose of rate limiting?**
- A) Ensure the agent responds quickly to users
- B) Prevent runaway loops and uncontrolled API cost ✅
- C) Encrypt tool call arguments
- D) Validate the agent's final response format

---

## Key Takeaways

1. **Agent = Observe + Reason + Plan + Act in a loop** — the LLM is the brain, tools are the hands, memory is the context.

2. **Start simple, scale to multi-agent** — use a single tool loop until you genuinely need routing, then graduate to supervisor/hierarchical patterns.

3. **MCP decouples tools from agents** — write tools once as MCP servers, use from any client. Think USB for AI.

4. **Planning with replanning beats blind execution** — decompose tasks into subtasks, reflect after each step, and replan when things go wrong.

5. **Memory is more than chat history** — short-term (buffer), long-term (vector store), episodic (past experiences), semantic (knowledge), and procedural (learned rules) each serve distinct purposes.

6. **LangGraph for production, CrewAI for prototypes** — choose your framework based on control needs, not hype.

7. **Safety is not optional** — sandboxing, rate limits, permissions, human-in-the-loop, and audit logging are all required for production agents.

8. **Evaluate like you test software** — build a regression suite with task completion rate, tool-use accuracy, efficiency, safety, and answer quality metrics.
