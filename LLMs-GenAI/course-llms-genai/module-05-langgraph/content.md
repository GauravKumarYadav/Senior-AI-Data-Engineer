# Module 5: LangGraph

> **Scenario**: ShopAssist's `AgentExecutor` works for simple queries, but it can't handle complex workflows: multi-step returns requiring approval, routing tickets to specialist agents, or retrying failed tool calls with fallback strategies. LangGraph gives you a **graph-based runtime** where you define nodes (actions), edges (transitions), and state — enabling cycles, branching, human-in-the-loop, and multi-agent orchestration.

---

## Screen 1: Why LangGraph — Beyond AgentExecutor

### The Limits of AgentExecutor

```python
# AgentExecutor is a simple loop:
#   1. LLM decides action
#   2. Execute tool
#   3. Feed result back to LLM
#   4. Repeat until LLM says "done"

# ❌ Problems:
# - No control over the loop — the LLM decides everything
# - Can't add human approval before destructive actions
# - No branching — can't route to different sub-workflows
# - No parallel execution — tools run one at a time
# - Hard to debug — opaque "agent_scratchpad" blob
# - No persistence — crash = lose entire conversation state
```

### LangGraph Solves This

```
┌──────────────────────────────────────────────────────────────────┐
│            AgentExecutor              LangGraph                  │
│                                                                  │
│  ┌─────┐    ┌─────┐                ┌─────┐    ┌─────┐          │
│  │ LLM │───▶│Tool │                │ LLM │───▶│Tools│          │
│  │     │◀───│     │                │     │    │     │          │
│  └──┬──┘    └─────┘                └──┬──┘    └──┬──┘          │
│     │                                 │          │              │
│     ▼                                 ▼          ▼              │
│  "Done!" (LLM decides)          ┌─────────┐  ┌──────┐         │
│                                  │ Human   │  │Route │         │
│  Simple loop — LLM has          │ Approval│  │      │         │
│  full control                   └────┬────┘  └──┬───┘         │
│                                      │          │              │
│  No branching                        ▼          ▼              │
│  No human gates               ┌──────────┐ ┌──────────┐       │
│  No persistence               │Specialist│ │Specialist│       │
│  No parallel paths            │ Agent A  │ │ Agent B  │       │
│                               └──────────┘ └──────────┘       │
│                                                                │
│                           Graph = full control over flow       │
└──────────────────────────────────────────────────────────────────┘
```

### When to Use What

| Feature | AgentExecutor | LangGraph |
|---|---|---|
| Simple Q&A with tools | ✅ Perfect | Overkill |
| Multi-step approval flows | ❌ Can't do | ✅ interrupt() |
| Parallel tool execution | ❌ Sequential | ✅ Send() / fan-out |
| Multi-agent routing | ❌ Single agent | ✅ Supervisor pattern |
| Persistent conversations | ❌ In-memory only | ✅ Checkpointing |
| Deterministic workflows | ❌ LLM decides all | ✅ You control edges |
| Time-travel / debugging | ❌ No state history | ✅ Replay from checkpoint |
| Production deployment | ⚠️ Limited | ✅ LangGraph Platform |

### 💡 Interview Insight

> **Q: When would you choose LangGraph over a simple AgentExecutor?**
>
> "I start with `AgentExecutor` for prototyping — it's simple and covers single-agent tool use well. I move to LangGraph when I need any of these: (1) **human-in-the-loop** — our ShopAssist agent must get customer confirmation before processing refunds over $50; (2) **deterministic routing** — billing issues go to the billing specialist, shipping issues to the shipping specialist, not just 'LLM figures it out'; (3) **persistence** — if the server crashes mid-conversation, LangGraph's checkpointing lets me resume exactly where I left off; (4) **multi-agent collaboration** — a supervisor agent delegates to specialists, which is impossible with a single AgentExecutor. The mental model shift is from 'loop' to 'graph' — nodes are actions, edges are transitions, and you explicitly define the control flow."

---

## Screen 2: StateGraph Fundamentals

### The Core Concepts

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# Step 1: Define your state schema
class SupportState(TypedDict):
    customer_message: str
    category: str
    response: str

# Step 2: Define node functions (each takes state, returns partial update)
def classify(state: SupportState) -> dict:
    """Classify the customer message into a category."""
    message = state["customer_message"].lower()
    if "order" in message or "track" in message or "ship" in message:
        return {"category": "shipping"}
    elif "return" in message or "refund" in message:
        return {"category": "returns"}
    else:
        return {"category": "general"}

def handle_shipping(state: SupportState) -> dict:
    return {"response": f"Let me track your order. Your package is on its way!"}

def handle_returns(state: SupportState) -> dict:
    return {"response": "I can help with your return. Our policy allows returns within 30 days."}

def handle_general(state: SupportState) -> dict:
    return {"response": "I'd be happy to help! Could you provide more details?"}

# Step 3: Build the graph
graph = StateGraph(SupportState)

# Add nodes
graph.add_node("classify", classify)
graph.add_node("handle_shipping", handle_shipping)
graph.add_node("handle_returns", handle_returns)
graph.add_node("handle_general", handle_general)

# Add edges
graph.add_edge(START, "classify")  # Start → classify

# Conditional edge: route based on category
def route_by_category(state: SupportState) -> str:
    category = state["category"]
    if category == "shipping":
        return "handle_shipping"
    elif category == "returns":
        return "handle_returns"
    else:
        return "handle_general"

graph.add_conditional_edges("classify", route_by_category)

# All handlers → END
graph.add_edge("handle_shipping", END)
graph.add_edge("handle_returns", END)
graph.add_edge("handle_general", END)

# Step 4: Compile and run
app = graph.compile()
result = app.invoke({"customer_message": "Where is my order ORD-7788?"})
print(result)
# {'customer_message': 'Where is my order ORD-7788?',
#  'category': 'shipping',
#  'response': 'Let me track your order. Your package is on its way!'}
```

### Graph Visualization

```
┌─────────────────────────────────────────────────────┐
│              SHOPASSIST ROUTING GRAPH                │
│                                                     │
│                   ┌─────────┐                       │
│                   │  START   │                       │
│                   └────┬────┘                        │
│                        │                             │
│                        ▼                             │
│                  ┌──────────┐                        │
│                  │ classify  │                        │
│                  └─────┬────┘                        │
│                        │                             │
│           ┌────────────┼────────────┐                │
│           │            │            │                │
│     "shipping"    "returns"    "general"             │
│           │            │            │                │
│           ▼            ▼            ▼                │
│    ┌───────────┐ ┌──────────┐ ┌──────────┐          │
│    │ handle_   │ │ handle_  │ │ handle_  │          │
│    │ shipping  │ │ returns  │ │ general  │          │
│    └─────┬─────┘ └────┬─────┘ └────┬─────┘          │
│          │             │            │                │
│          └─────────────┼────────────┘                │
│                        │                             │
│                        ▼                             │
│                   ┌─────────┐                        │
│                   │   END   │                        │
│                   └─────────┘                        │
└─────────────────────────────────────────────────────┘
```

### Key Rules

```
┌──────────────────────────────────────────────────────────────┐
│  LANGGRAPH RULES:                                            │
│                                                              │
│  1. Nodes are functions: (state) → partial state update      │
│  2. Edges are transitions: fixed or conditional              │
│  3. State accumulates: each node's return merges into state  │
│  4. START → first node, last node → END                      │
│  5. Conditional edges: function returns the NEXT node name   │
│  6. Cycles are allowed: node A → B → A (for retry loops)    │
│  7. compile() validates the graph before execution           │
└──────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: Explain StateGraph's execution model.**
>
> "A `StateGraph` is a finite state machine where nodes are Python functions and edges define transitions. The state is a `TypedDict` that flows through the graph — each node receives the full state and returns a **partial update** (only the keys it wants to change). LangGraph merges these updates into the state object. Edges can be fixed (`add_edge`) or conditional (`add_conditional_edges`) where a routing function inspects the state and returns the name of the next node. Execution starts at `START`, follows edges, and terminates at `END`. The critical insight is that cycles are allowed — you can have a node that loops back to a previous node, which is impossible with simple sequential chains. This enables retry loops, iterative refinement, and ReAct-style agent loops."

---

## Screen 3: State Management — Reducers & MessagesState

### The Problem: List Accumulation

```python
# Default behavior: each node OVERWRITES state keys
class State(TypedDict):
    messages: list  # ❌ Each node replaces the entire list!

def node_a(state):
    return {"messages": [AIMessage("Hello!")]}

def node_b(state):
    return {"messages": [AIMessage("How can I help?")]}
    # ❌ This REPLACES messages — node_a's message is gone!
```

### Reducers with Annotated Types

```python
from typing import Annotated, TypedDict
from langchain_core.messages import AnyMessage
import operator

# Use Annotated + operator.add to APPEND instead of replace
class SupportState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]  # Append!
    customer_id: str                                      # Overwrite (default)
    tool_results: Annotated[list[dict], operator.add]     # Append!

def node_a(state: SupportState) -> dict:
    return {"messages": [AIMessage(content="Hello!")]}
    # ✅ APPENDS to messages list

def node_b(state: SupportState) -> dict:
    return {"messages": [AIMessage(content="How can I help?")]}
    # ✅ messages is now [AIMessage("Hello!"), AIMessage("How can I help?")]
```

### MessagesState — Built-in Chat State

```python
from langgraph.graph import MessagesState

# MessagesState is a pre-built TypedDict with messages: Annotated[list, add_messages]
# add_messages is a smart reducer that:
# - Appends new messages
# - Updates existing messages by ID (for tool results)
# - Handles message deduplication

class ShopAssistState(MessagesState):
    # Inherit messages with smart reducer, add custom fields
    customer_id: str
    order_id: str | None
    escalated: bool

# This is equivalent to:
# class ShopAssistState(TypedDict):
#     messages: Annotated[list[AnyMessage], add_messages]
#     customer_id: str
#     order_id: str | None
#     escalated: bool
```

### Custom Reducers

```python
from typing import Annotated

def merge_dicts(existing: dict, new: dict) -> dict:
    """Custom reducer that deep-merges dictionaries."""
    merged = existing.copy()
    merged.update(new)
    return merged

class AnalyticsState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    metrics: Annotated[dict, merge_dicts]  # Custom merge behavior

# Each node can add metrics without losing previous ones:
# node_a returns {"metrics": {"sentiment": "positive"}}
# node_b returns {"metrics": {"category": "shipping"}}
# Result: {"metrics": {"sentiment": "positive", "category": "shipping"}}
```

### 💡 Interview Insight

> **Q: How does state management work in LangGraph, and what are reducers?**
>
> "LangGraph state is a `TypedDict` where each key has a **reducer** that defines how updates are merged. By default, new values overwrite old ones — which is fine for scalar fields like `customer_id`. But for lists like chat messages, you want to **append**, not replace. You specify this with `Annotated[list, operator.add]`. LangGraph provides `MessagesState` as a built-in base class with a smart `add_messages` reducer that handles appending, updating by ID, and deduplication. For ShopAssist, I'd extend `MessagesState` with custom fields: `customer_id`, `order_id`, `escalated` flag. The reducer pattern is crucial because multiple nodes may write to the same key — without proper reducers, you'd lose data as each node overwrites the previous value."

---

## Screen 4: Building a ReAct Agent

### Hand-Built ReAct Agent

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage

# Tools from Module 4
tools = [track_order, search_knowledge_base, process_return, escalate_to_human]

llm = ChatOpenAI(model="gpt-4o", temperature=0).bind_tools(tools)

# Node 1: Call the LLM
def call_model(state: MessagesState) -> dict:
    system = SystemMessage(content="You are ShopAssist, a helpful support agent.")
    messages = [system] + state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}

# Node 2: Execute tools (ToolNode handles this automatically)
tool_node = ToolNode(tools)

# Routing: does the LLM want to call tools?
def should_continue(state: MessagesState) -> str:
    last_message = state["messages"][-1]
    if last_message.tool_calls:    # LLM requested tool calls
        return "tools"
    return END                      # No tools → we're done

# Build the graph
graph = StateGraph(MessagesState)
graph.add_node("agent", call_model)
graph.add_node("tools", tool_node)

graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "agent")  # After tools → back to agent (the CYCLE)

app = graph.compile()
```

### The ReAct Loop Visualized

```
┌──────────────────────────────────────────────────────┐
│              REACT AGENT GRAPH                        │
│                                                       │
│              ┌─────────┐                              │
│              │  START   │                              │
│              └────┬────┘                               │
│                   │                                    │
│                   ▼                                    │
│            ┌──────────┐                                │
│    ┌──────▶│  agent   │                                │
│    │       │ (LLM)    │                                │
│    │       └─────┬────┘                                │
│    │             │                                     │
│    │      has_tool_calls?                              │
│    │       ╱          ╲                                │
│    │     Yes           No                              │
│    │      │             │                              │
│    │      ▼             ▼                              │
│    │  ┌────────┐   ┌─────────┐                        │
│    │  │ tools  │   │   END   │                        │
│    │  │(execute│   └─────────┘                        │
│    │  │ tools) │                                       │
│    │  └───┬────┘                                       │
│    │      │                                            │
│    └──────┘  ◀── THE CYCLE (tools → agent → tools...) │
│                                                       │
└──────────────────────────────────────────────────────┘
```

### Using the Prebuilt ReAct Agent

```python
from langgraph.prebuilt import create_react_agent

# One-liner that creates the same graph as above!
agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=tools,
    prompt="You are ShopAssist, a helpful e-commerce support agent.",
)

# Same interface
result = agent.invoke({
    "messages": [HumanMessage(content="Where is order ORD-7788?")]
})

for msg in result["messages"]:
    print(f"{msg.type}: {msg.content[:100]}")
# human: Where is order ORD-7788?
# ai: [tool_call: track_order(order_id="ORD-7788")]
# tool: {"order_id": "ORD-7788", "status": "shipped", "eta": "2026-04-08"}
# ai: Your order ORD-7788 has been shipped and is expected to arrive on April 8, 2026!
```

### Hand-Built vs create_react_agent

| Aspect | Hand-Built | create_react_agent |
|---|---|---|
| Lines of code | ~25 | 5 |
| Customization | Full control | Limited hooks |
| Adding human approval | Easy — add node | Need to modify |
| Custom routing logic | Your conditional edges | Predefined |
| Extra state fields | Extend TypedDict | Use state_schema param |
| Learning value | High — understand internals | Low — black box |
| Production use | When you need custom flow | Simple tool-use agents |

### 💡 Interview Insight

> **Q: Explain how a ReAct agent works in LangGraph.**
>
> "A ReAct agent is a two-node cyclic graph: an `agent` node (LLM call) and a `tools` node (tool execution). The agent node calls the LLM with the conversation history. If the LLM returns `tool_calls`, the conditional edge routes to the `tools` node, which executes the requested tools and adds `ToolMessage` results to the state. The edge from `tools` goes back to `agent` — this is the cycle. The agent sees the tool results, decides if it needs more tools or can answer, and either loops again or routes to `END`. The key difference from AgentExecutor is that this is an explicit graph — I can add nodes between `agent` and `tools` (like a validation step), add a human approval gate before destructive tools, or replace the routing logic entirely. `create_react_agent` is a convenience wrapper that builds this exact graph, but I prefer hand-building when I need custom control flow."

---

## Screen 5: Agent Patterns — Plan-and-Execute, Reflection, Self-RAG

### Pattern 1: Plan-and-Execute

```python
# The agent creates a plan, executes each step, then replans if needed
# Good for: complex multi-step tasks like "Process a return, issue a refund,
#           and send a confirmation email"

class PlanExecuteState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    plan: list[str]              # Steps to execute
    current_step: int            # Which step we're on
    results: Annotated[list[str], operator.add]  # Results from each step
    final_response: str

def planner(state: PlanExecuteState) -> dict:
    """LLM creates a plan of action steps."""
    response = planner_llm.invoke([
        SystemMessage(content="""Create a step-by-step plan to handle this request.
        Return a JSON list of steps."""),
        *state["messages"],
    ])
    plan = json.loads(response.content)
    return {"plan": plan, "current_step": 0}

def executor(state: PlanExecuteState) -> dict:
    """Execute the current step in the plan."""
    step = state["plan"][state["current_step"]]
    result = executor_chain.invoke({"step": step, "context": state["results"]})
    return {
        "results": [result],
        "current_step": state["current_step"] + 1,
    }

def replanner(state: PlanExecuteState) -> dict:
    """Check if plan needs adjustment based on results so far."""
    response = replanner_llm.invoke([
        SystemMessage(content="Review the plan and results. Adjust if needed."),
        HumanMessage(content=f"Plan: {state['plan']}\nResults: {state['results']}"),
    ])
    # Returns updated plan or signals completion
    return {"plan": json.loads(response.content)}

def should_continue(state: PlanExecuteState) -> str:
    if state["current_step"] >= len(state["plan"]):
        return "replanner"  # Finished all steps — check if we need more
    return "executor"       # More steps to execute

def should_end(state: PlanExecuteState) -> str:
    if "DONE" in state["plan"]:
        return END
    return "executor"  # Revised plan — continue executing

# Graph: planner → executor ↔ replanner → END
```

```
┌───────────────────────────────────────────────────────────┐
│            PLAN-AND-EXECUTE PATTERN                       │
│                                                           │
│  ┌──────────┐    ┌──────────┐    ┌───────────┐           │
│  │ Planner  │───▶│ Executor │───▶│ Replanner │           │
│  │          │    │          │    │           │           │
│  │ "1. Look │    │ Execute  │    │ "Plan     │           │
│  │  up order│    │ step 1   │    │  still    │           │
│  │  2. Check│    │ Execute  │    │  good? Or │──▶ END    │
│  │  policy  │    │ step 2   │    │  adjust?" │           │
│  │  3. Init │    │ ...      │    │           │           │
│  │  return" │    │          │◀───│ Revised!  │           │
│  └──────────┘    └──────────┘    └───────────┘           │
└───────────────────────────────────────────────────────────┘
```

### Pattern 2: Reflection

```python
# Generate → Critique → Revise loop
# Good for: drafting customer emails, policy documents, complex responses

class ReflectionState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    draft: str
    critique: str
    iteration: int

def generate(state: ReflectionState) -> dict:
    """Generate or revise a draft response."""
    if state.get("critique"):
        prompt = f"Revise this draft based on feedback:\n\nDraft: {state['draft']}\n\nFeedback: {state['critique']}"
    else:
        prompt = f"Draft a response to: {state['messages'][-1].content}"
    
    response = llm.invoke([HumanMessage(content=prompt)])
    return {"draft": response.content, "iteration": state.get("iteration", 0) + 1}

def critique(state: ReflectionState) -> dict:
    """Critique the draft for accuracy, tone, and completeness."""
    response = llm.invoke([
        SystemMessage(content="""Critique this customer support response.
        Check: accuracy, empathy, completeness, actionability.
        If it's good, respond with 'APPROVED'. Otherwise, provide specific feedback."""),
        HumanMessage(content=state["draft"]),
    ])
    return {"critique": response.content}

def should_revise(state: ReflectionState) -> str:
    if "APPROVED" in state["critique"] or state["iteration"] >= 3:
        return END
    return "generate"  # Loop back for revision

graph = StateGraph(ReflectionState)
graph.add_node("generate", generate)
graph.add_node("critique", critique)
graph.add_edge(START, "generate")
graph.add_edge("generate", "critique")
graph.add_conditional_edges("critique", should_revise)
app = graph.compile()
```

### Pattern 3: Self-RAG

```python
# Decide whether to retrieve → retrieve → grade relevance → 
# generate → check for hallucination → check answer quality

class SelfRAGState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    question: str
    documents: list[Document]
    generation: str
    needs_retrieval: bool
    is_relevant: bool
    is_grounded: bool

def decide_retrieval(state: SelfRAGState) -> dict:
    """Decide if the question needs document retrieval."""
    response = llm.invoke([
        SystemMessage(content="Does this question need document retrieval? Respond YES or NO."),
        HumanMessage(content=state["question"]),
    ])
    return {"needs_retrieval": "YES" in response.content.upper()}

def retrieve(state: SelfRAGState) -> dict:
    docs = retriever.invoke(state["question"])
    return {"documents": docs}

def grade_documents(state: SelfRAGState) -> dict:
    """Grade whether retrieved documents are relevant to the question."""
    relevant_docs = []
    for doc in state["documents"]:
        response = llm.invoke([
            SystemMessage(content="Is this document relevant to the question? YES or NO."),
            HumanMessage(content=f"Question: {state['question']}\nDocument: {doc.page_content}"),
        ])
        if "YES" in response.content.upper():
            relevant_docs.append(doc)
    return {"documents": relevant_docs, "is_relevant": len(relevant_docs) > 0}

def generate(state: SelfRAGState) -> dict:
    context = "\n".join(doc.page_content for doc in state["documents"])
    response = llm.invoke([
        SystemMessage(content=f"Answer based on context:\n{context}"),
        HumanMessage(content=state["question"]),
    ])
    return {"generation": response.content}

def check_hallucination(state: SelfRAGState) -> dict:
    """Verify the answer is grounded in the retrieved documents."""
    response = llm.invoke([
        SystemMessage(content="Is this answer supported by the documents? YES or NO."),
        HumanMessage(content=f"Documents: {state['documents']}\nAnswer: {state['generation']}"),
    ])
    return {"is_grounded": "YES" in response.content.upper()}

# Route: hallucinated → regenerate, grounded → END
```

### 💡 Interview Insight

> **Q: Compare Plan-and-Execute, Reflection, and Self-RAG patterns. When do you use each?**
>
> "These are three fundamental agent patterns. **Plan-and-Execute** is for multi-step tasks where the sequence matters — like 'look up the order, check if it's eligible for return, process the return, email confirmation.' The planner creates a structured plan, the executor runs each step, and the replanner adjusts if something goes wrong. **Reflection** is for quality-critical outputs — when drafting a legal response or a complex customer email, the generate-critique-revise loop catches errors a single pass would miss. I cap iterations at 3 to prevent infinite loops. **Self-RAG** adds intelligence to retrieval — instead of always retrieving, the agent decides if retrieval is needed, grades the relevance of retrieved docs, and checks for hallucination in the final answer. For ShopAssist, I'd use Plan-and-Execute for complex workflows (multi-item returns), Reflection for escalation emails to VIP customers, and Self-RAG for the knowledge base Q&A pipeline."

---

## Screen 6: Multi-Agent Architectures

### Supervisor Pattern

```python
from langgraph.graph import StateGraph, MessagesState, START, END

# Specialist agents
billing_agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[lookup_invoice, process_refund, apply_credit],
    prompt="You are a billing specialist agent.",
)

shipping_agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[track_order, update_address, request_reshipment],
    prompt="You are a shipping specialist agent.",
)

product_agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[search_products, check_inventory, compare_products],
    prompt="You are a product specialist agent.",
)

# Supervisor decides which specialist to delegate to
def supervisor(state: MessagesState) -> dict:
    response = supervisor_llm.invoke([
        SystemMessage(content="""You are the ShopAssist supervisor.
        Route the customer's request to the right specialist:
        - "billing" for charges, refunds, invoices
        - "shipping" for tracking, delivery, address changes
        - "product" for product info, comparisons, inventory
        - "DONE" when the customer's issue is fully resolved.
        
        Respond with ONLY the specialist name or DONE."""),
        *state["messages"],
    ])
    return {"next": response.content.strip().lower()}

def route_supervisor(state: dict) -> str:
    return state.get("next", END)

# Build supervisor graph
graph = StateGraph(MessagesState)
graph.add_node("supervisor", supervisor)
graph.add_node("billing", billing_agent)
graph.add_node("shipping", shipping_agent)
graph.add_node("product", product_agent)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route_supervisor, {
    "billing": "billing",
    "shipping": "shipping",
    "product": "product",
    "done": END,
})
# After each specialist, return to supervisor for next decision
graph.add_edge("billing", "supervisor")
graph.add_edge("shipping", "supervisor")
graph.add_edge("product", "supervisor")

app = graph.compile()
```

### Multi-Agent Architecture Comparison

```
┌──────────────────────────────────────────────────────────────────────┐
│                    MULTI-AGENT PATTERNS                              │
│                                                                      │
│  SUPERVISOR                HIERARCHICAL            COLLABORATIVE     │
│  ┌──────────┐             ┌──────────┐            ┌─────┐           │
│  │Supervisor│             │  Super-  │            │  A  │───▶───┐   │
│  │          │             │ Supervisor│           └──┬──┘       │   │
│  └──┬─┬─┬──┘             └──┬────┬──┘               │          │   │
│     │ │ │                    │    │                   ▼          ▼   │
│     ▼ ▼ ▼                   ▼    ▼               ┌─────┐   ┌─────┐│
│   ┌─┐┌─┐┌─┐          ┌──────┐┌──────┐           │  B  │   │  C  ││
│   │A││B││C│          │ Sub  ││ Sub  │           └──┬──┘   └──┬──┘│
│   └─┘└─┘└─┘          │ Sup1 ││ Sup2 │              │          │   │
│                       └┬──┬─┘└┬──┬─┘              ▼          ▼   │
│  Router agent         │  │   │  │            ┌─────────────────┐ │
│  delegates to         ▼  ▼   ▼  ▼            │   Shared State  │ │
│  specialists       ┌─┐┌─┐ ┌─┐┌─┐            └─────────────────┘ │
│                    │a││b│ │c││d│                                  │
│                    └─┘└─┘ └─┘└─┘           Agents pass work      │
│                                             to each other         │
│  Best: clear       Best: large orgs       Best: creative tasks   │
│  specialization    with teams              brainstorming          │
│                                                                    │
│  SWARM                                                             │
│  ┌─────┐    ┌─────┐    ┌─────┐                                    │
│  │  A  │◀──▶│  B  │◀──▶│  C  │   Each agent can hand off to any  │
│  └─────┘    └─────┘    └─────┘   other agent dynamically.         │
│       ◀────────────────────▶     No central coordinator.           │
│                                                                    │
│  Best: flexible, OpenAI-style agent swarms                        │
└──────────────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: How would you design a multi-agent system for ShopAssist?**
>
> "I'd use the **Supervisor pattern** — it's the most controllable. A supervisor agent receives every customer message and routes to specialists: billing (refunds, charges, invoices), shipping (tracking, delivery, address), and product (catalog, inventory, comparisons). Each specialist is a `create_react_agent` with domain-specific tools. After the specialist responds, control returns to the supervisor who decides: is the issue resolved (→ END), does another specialist need to weigh in (→ route again), or does this need human escalation? The key design decisions: (1) the supervisor uses a cheaper model (GPT-4o-mini) since it's just routing; (2) specialists use GPT-4o for complex reasoning; (3) all agents share the same message history via `MessagesState` so context isn't lost during handoffs; (4) I add guardrails — the supervisor can't call more than 3 specialists per conversation to prevent infinite delegation loops."

---

## Screen 7: Human-in-the-Loop & Checkpointing

### The interrupt() Function

```python
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import MemorySaver

class RefundState(MessagesState):
    refund_amount: float
    approved: bool | None

def calculate_refund(state: RefundState) -> dict:
    """Calculate the refund amount."""
    return {"refund_amount": 89.99}

def request_approval(state: RefundState) -> dict:
    """Pause execution and ask for human approval."""
    amount = state["refund_amount"]
    
    # interrupt() PAUSES the graph and returns control to the caller
    approval = interrupt(
        f"Refund of ${amount:.2f} requires approval. Approve? (yes/no)"
    )
    
    return {"approved": approval == "yes"}

def process_refund(state: RefundState) -> dict:
    if state["approved"]:
        return {"messages": [AIMessage(content=f"Refund of ${state['refund_amount']:.2f} processed!")]}
    return {"messages": [AIMessage(content="Refund cancelled per supervisor decision.")]}

# Build graph
graph = StateGraph(RefundState)
graph.add_node("calculate", calculate_refund)
graph.add_node("approve", request_approval)
graph.add_node("process", process_refund)

graph.add_edge(START, "calculate")
graph.add_edge("calculate", "approve")
graph.add_edge("approve", "process")
graph.add_edge("process", END)

# Compile with checkpointing (required for interrupt)
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# --- Usage ---
config = {"configurable": {"thread_id": "refund-001"}}

# Run until interrupt
result = app.invoke(
    {"messages": [HumanMessage(content="I want a refund for order ORD-7788")]},
    config=config,
)
# Graph PAUSES at request_approval node
# Returns: interrupt value = "Refund of $89.99 requires approval..."

# Resume with human decision
result = app.invoke(
    Command(resume="yes"),  # Human approves
    config=config,
)
# Graph RESUMES from where it paused
# Output: "Refund of $89.99 processed!"
```

### Checkpointing & Persistence

```python
from langgraph.checkpoint.memory import MemorySaver     # Dev only — in-memory
from langgraph.checkpoint.sqlite import SqliteSaver      # SQLite file
from langgraph.checkpoint.postgres import PostgresSaver  # Production

# MemorySaver — development and testing
memory_saver = MemorySaver()

# SqliteSaver — single-server persistence
sqlite_saver = SqliteSaver.from_conn_string("shopassist.db")

# PostgresSaver — production multi-server
postgres_saver = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost:5432/shopassist"
)

# Compile with checkpointer
app = graph.compile(checkpointer=postgres_saver)

# thread_id isolates conversations
config_alice = {"configurable": {"thread_id": "alice-session-001"}}
config_bob = {"configurable": {"thread_id": "bob-session-002"}}

# Alice and Bob have independent conversation states
app.invoke({"messages": [HumanMessage(content="Hi!")]}, config=config_alice)
app.invoke({"messages": [HumanMessage(content="Hello!")]}, config=config_bob)
```

### Time Travel — Replay from Checkpoint

```python
# Get all checkpoints for a conversation
checkpoints = list(app.get_state_history(config_alice))

for cp in checkpoints:
    print(f"Step: {cp.metadata['step']}, Node: {cp.metadata.get('source')}")
    print(f"  State: {cp.values['messages'][-1].content[:50]}...")

# Replay from a specific checkpoint (undo/redo)
# Get an earlier state
earlier_state = checkpoints[2]  # Go back 2 steps

# Update state and resume from that point
app.update_state(config_alice, earlier_state.values)
result = app.invoke(None, config=config_alice)  # Continue from updated state
```

### Cross-Thread Memory (Store)

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# Store persists data ACROSS threads — user preferences, past interactions
# Namespace: ("customers", customer_id)

def personalize(state: MessagesState, *, store) -> dict:
    """Load customer preferences from cross-thread store."""
    customer_id = state.get("customer_id", "unknown")
    
    # Retrieve stored preferences
    prefs = store.search(("customers", customer_id))
    if prefs:
        pref_text = f"Customer preferences: {prefs[0].value}"
    else:
        pref_text = "No stored preferences."
    
    response = llm.invoke([
        SystemMessage(content=f"Personalize response using: {pref_text}"),
        *state["messages"],
    ])
    return {"messages": [response]}

# After resolving an issue, save preferences for next time
def save_preferences(state: MessagesState, *, store) -> dict:
    store.put(
        ("customers", state["customer_id"]),
        "prefs",
        {"preferred_contact": "email", "vip": True, "past_issues": ["shipping_delay"]},
    )
    return {}
```

### 💡 Interview Insight

> **Q: How do you implement human-in-the-loop workflows with LangGraph?**
>
> "LangGraph's `interrupt()` function is purpose-built for this. When a node calls `interrupt()`, the graph **pauses execution**, saves the complete state to the checkpointer, and returns the interrupt value to the caller. The caller (your API server) can then present this to a human — 'Approve refund of $89.99?' When the human responds, you call `app.invoke(Command(resume="yes"), config)` and the graph **resumes exactly where it paused** with the human's input. This requires a checkpointer — `MemorySaver` for dev, `PostgresSaver` for production. The beauty is that the graph can pause for minutes, hours, or days. For ShopAssist, I'd add interrupt points before: refunds over $50, account deletions, and escalation routing. The `thread_id` in config isolates each conversation, so thousands of paused conversations can coexist."

---

## Screen 8: Subgraphs, Parallelism & Streaming

### Subgraphs — Nested Graphs

```python
# Complex agents can be composed from smaller subgraphs

# Subgraph 1: Return processing workflow
return_graph = StateGraph(ReturnState)
return_graph.add_node("validate_order", validate_order)
return_graph.add_node("check_eligibility", check_eligibility)
return_graph.add_node("create_label", create_return_label)
return_graph.add_edge(START, "validate_order")
return_graph.add_edge("validate_order", "check_eligibility")
return_graph.add_edge("check_eligibility", "create_label")
return_graph.add_edge("create_label", END)
return_subgraph = return_graph.compile()

# Subgraph 2: Refund processing workflow
refund_graph = StateGraph(RefundState)
refund_graph.add_node("calculate", calculate_refund)
refund_graph.add_node("approve", request_approval)
refund_graph.add_node("process", execute_refund)
refund_graph.add_edge(START, "calculate")
refund_graph.add_edge("calculate", "approve")
refund_graph.add_edge("approve", "process")
refund_graph.add_edge("process", END)
refund_subgraph = refund_graph.compile()

# Parent graph uses subgraphs as nodes
parent_graph = StateGraph(MessagesState)
parent_graph.add_node("router", route_request)
parent_graph.add_node("returns", return_subgraph)   # Subgraph as node!
parent_graph.add_node("refunds", refund_subgraph)   # Subgraph as node!
parent_graph.add_edge(START, "router")
parent_graph.add_conditional_edges("router", route_decision)
parent_graph.add_edge("returns", END)
parent_graph.add_edge("refunds", END)
```

### Send() — Map-Reduce Fan-Out

```python
from langgraph.types import Send

# Process multiple items in parallel (e.g., multi-item return)
class MultiReturnState(TypedDict):
    items: list[dict]           # Items to process
    results: Annotated[list[dict], operator.add]  # Results accumulate

def fan_out_items(state: MultiReturnState) -> list[Send]:
    """Create a parallel task for each item."""
    return [
        Send("process_single_item", {"item": item})
        for item in state["items"]
    ]

def process_single_item(state: dict) -> dict:
    """Process one item return (runs in parallel for each item)."""
    item = state["item"]
    # Check eligibility, create label, etc.
    return {"results": [{"item_id": item["id"], "status": "return_initiated"}]}

def aggregate_results(state: MultiReturnState) -> dict:
    """Combine all parallel results into a final response."""
    return {"messages": [AIMessage(
        content=f"Processed {len(state['results'])} returns successfully."
    )]}

graph = StateGraph(MultiReturnState)
graph.add_node("process_single_item", process_single_item)
graph.add_node("aggregate", aggregate_results)

graph.add_conditional_edges(START, fan_out_items)  # Fan-out with Send()
graph.add_edge("process_single_item", "aggregate")  # All converge
graph.add_edge("aggregate", END)
```

### Streaming

```python
# LangGraph supports multiple stream modes

app = graph.compile(checkpointer=MemorySaver())
config = {"configurable": {"thread_id": "stream-demo"}}

# Mode 1: "values" — full state after each node
async for state in app.astream(
    {"messages": [HumanMessage(content="Track ORD-7788")]},
    config=config,
    stream_mode="values",
):
    print(f"State has {len(state['messages'])} messages")

# Mode 2: "updates" — only the changes from each node
async for update in app.astream(
    {"messages": [HumanMessage(content="Track ORD-7788")]},
    config=config,
    stream_mode="updates",
):
    for node_name, node_output in update.items():
        print(f"Node '{node_name}' output: {node_output}")

# Mode 3: "messages" — stream individual LLM tokens
async for event in app.astream(
    {"messages": [HumanMessage(content="Track ORD-7788")]},
    config=config,
    stream_mode="messages",
):
    message, metadata = event
    if isinstance(message, AIMessageChunk):
        print(message.content, end="", flush=True)  # Token-by-token!

# Multiple stream modes at once
async for event in app.astream(
    {"messages": [HumanMessage(content="Track ORD-7788")]},
    config=config,
    stream_mode=["values", "messages"],
):
    # event is a tuple: (stream_mode, data)
    pass
```

### 💡 Interview Insight

> **Q: How do you handle parallel processing and streaming in LangGraph?**
>
> "LangGraph offers `Send()` for fan-out parallelism — like processing a multi-item return where each item is checked independently. You return a list of `Send` objects from a conditional edge, and LangGraph runs all target nodes in parallel, collecting results via reducers. For streaming, LangGraph has four modes: `values` (full state snapshots), `updates` (delta changes per node), `messages` (token-by-token LLM output), and `custom` (user-defined events). In production, I use `messages` mode for the frontend to display typing indicators and token-by-token responses, while logging `updates` mode to our observability stack for debugging. The key is that streaming works through the entire graph — even through subgraphs and tool calls — without any extra code."

---

## Screen 9: LangGraph Platform — Production Deployment

### LangGraph Server

```python
# langgraph.json — configuration for LangGraph Platform
# {
#   "graphs": {
#     "shopassist": "./agent.py:graph"
#   },
#   "dependencies": ["langchain-openai", "langchain-chroma"],
#   "env": {
#     "OPENAI_API_KEY": "",
#     "LANGSMITH_API_KEY": ""
#   }
# }

# agent.py — your graph definition
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import create_react_agent

graph = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=tools,
    prompt="You are ShopAssist.",
)

# LangGraph Server exposes this as an API:
# POST /threads                    — Create conversation thread
# POST /threads/{id}/runs          — Run the graph
# POST /threads/{id}/runs/stream   — Stream graph execution
# GET  /threads/{id}/state         — Get current state
# POST /threads/{id}/state         — Update state (for human-in-the-loop)
# GET  /threads/{id}/history       — Get state history (time travel)
```

### Interacting with LangGraph Server

```python
from langgraph_sdk import get_client

client = get_client(url="http://localhost:8123")

# Create a thread (conversation)
thread = await client.threads.create()

# Run the agent
result = await client.runs.create(
    thread_id=thread["thread_id"],
    graph_id="shopassist",
    input={"messages": [{"role": "user", "content": "Track ORD-7788"}]},
)

# Stream responses
async for event in client.runs.stream(
    thread_id=thread["thread_id"],
    graph_id="shopassist",
    input={"messages": [{"role": "user", "content": "Track ORD-7788"}]},
    stream_mode="messages",
):
    print(event.data, end="")

# Human-in-the-loop: get pending interrupts
state = await client.threads.get_state(thread["thread_id"])
if state.get("pending_interrupts"):
    # Resume with approval
    await client.runs.create(
        thread_id=thread["thread_id"],
        graph_id="shopassist",
        command={"resume": "yes"},
    )
```

### LangGraph Studio — Visual Debugger

```
┌──────────────────────────────────────────────────────────────┐
│              LANGGRAPH STUDIO                                │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Graph Visualization (live)                             │ │
│  │                                                         │ │
│  │    [START] ──▶ [supervisor] ──▶ [billing] ──▶ [END]    │ │
│  │                    │               ▲                    │ │
│  │                    ▼               │                    │ │
│  │               [shipping] ──────────┘                    │ │
│  │                                                         │ │
│  │    ● Current node highlighted in blue                   │ │
│  │    ● Completed nodes in green                           │ │
│  │    ● Error nodes in red                                 │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────────────────────┐ │
│  │  State Inspector  │  │  Step-by-Step Execution          │ │
│  │                    │  │                                  │ │
│  │  messages: [...]   │  │  Step 1: supervisor (120ms)     │ │
│  │  customer_id: abc  │  │    → Routed to "billing"        │ │
│  │  category: billing │  │  Step 2: billing (1.2s)         │ │
│  │  escalated: false  │  │    → Called lookup_invoice()     │ │
│  │                    │  │  Step 3: supervisor (90ms)       │ │
│  │  [Edit State]     │  │    → "DONE"                      │ │
│  └──────────────────┘  └──────────────────────────────────┘ │
│                                                              │
│  Features:                                                   │
│  • Real-time graph execution visualization                   │
│  • State inspection and editing at any step                  │
│  • Time-travel: replay from any checkpoint                   │
│  • Fork: create branches from any state                      │
│  • Interrupt handling: approve/reject in UI                   │
└──────────────────────────────────────────────────────────────┘
```

### Deployment Options

| Option | Best For | Scaling | Cost |
|---|---|---|---|
| Local (langgraph dev) | Development | None | Free |
| Self-hosted Server | On-prem, air-gapped | Manual | Infra cost |
| LangGraph Cloud | Managed production | Auto-scaling | Usage-based |

### 💡 Interview Insight

> **Q: How would you deploy a LangGraph agent to production?**
>
> "For production at ShopAssist, I'd deploy on LangGraph Cloud or self-hosted LangGraph Server with PostgreSQL checkpointing. The architecture: LangGraph Server exposes a REST/streaming API that my FastAPI frontend consumes. Each customer conversation gets a `thread_id` mapping to their support ticket. Checkpointing to PostgreSQL ensures crash recovery — if a server restarts mid-conversation, it resumes from the last checkpoint. For human-in-the-loop, the API exposes pending interrupts that my supervisor dashboard polls. I'd use LangGraph Studio during development for visual debugging — seeing the graph execute step-by-step with state inspection is invaluable for catching routing bugs. Monitoring goes through LangSmith: every graph execution is traced with node-level latency, token costs, and tool call success rates. Scaling is horizontal — LangGraph Server is stateless (state lives in PostgreSQL), so I can run multiple instances behind a load balancer."

---

## Module 5 Quiz

**1. What is the primary advantage of LangGraph over AgentExecutor?**

- A) LangGraph is faster at making LLM calls
- B) LangGraph provides explicit graph-based control flow with cycles, branching, human-in-the-loop, and persistence ✅
- C) LangGraph uses less memory
- D) LangGraph doesn't require an LLM
- E) LangGraph automatically chooses the best model

**2. In LangGraph state management, what does `Annotated[list, operator.add]` achieve?**

- A) It validates that the list contains only integers
- B) It makes the list immutable
- C) It defines a reducer that APPENDS new values to the list instead of replacing it ✅
- D) It sorts the list after each update
- E) It limits the list to a maximum length

**3. In the Supervisor multi-agent pattern, what happens after a specialist agent completes its task?**

- A) The graph ends immediately
- B) Control returns to the supervisor, who decides the next action (another specialist or END) ✅
- C) The specialist automatically routes to another specialist
- D) The customer is asked to rate the interaction
- E) All other specialists run in parallel

**4. How does `interrupt()` enable human-in-the-loop workflows?**

- A) It sends an email to a human reviewer
- B) It pauses graph execution, saves state to the checkpointer, and returns control to the caller who can later resume with `Command(resume=...)` ✅
- C) It creates a timeout that waits for human input
- D) It switches the LLM to a human-supervised mode
- E) It logs the decision for later human review

**5. Which LangGraph feature enables parallel processing of multiple items (e.g., processing a multi-item return)?**

- A) RunnableParallel
- B) ToolNode with multiple tools
- C) Send() for map-reduce fan-out ✅
- D) MessagesState with reducers
- E) Subgraphs with shared state

---

## Key Takeaways

1. **LangGraph replaces AgentExecutor when you need control** — cycles for retry loops, conditional edges for routing, `interrupt()` for human approval, and checkpointing for crash recovery. Start with AgentExecutor, graduate to LangGraph when complexity demands it.

2. **StateGraph = nodes + edges + state**. Nodes are functions that receive and return partial state updates. Edges define transitions (fixed or conditional). State is a `TypedDict` with optional reducers for merge behavior.

3. **Reducers are critical for list accumulation**. Without `Annotated[list, operator.add]`, each node overwrites the message list. `MessagesState` provides this out of the box with a smart `add_messages` reducer.

4. **The ReAct pattern is a two-node cycle**: agent (LLM) → tools → agent → ... until the LLM stops requesting tools. `create_react_agent` builds this automatically, but hand-building gives you insertion points for custom logic.

5. **Three core agent patterns**: Plan-and-Execute for multi-step workflows, Reflection for quality-critical outputs, Self-RAG for intelligent retrieval with hallucination checks. Choose based on your use case, not trend.

6. **Supervisor pattern is the go-to multi-agent architecture** — a router agent delegates to specialist agents with domain-specific tools. Simpler to debug and control than collaborative or swarm patterns.

7. **Human-in-the-loop requires checkpointing**. `interrupt()` pauses the graph, the checkpointer saves state, and `Command(resume=value)` continues execution. PostgresSaver for production, MemorySaver for development.

8. **LangGraph Platform (Server + Studio + Cloud)** provides production-grade deployment: REST API for integration, visual debugging for development, PostgreSQL persistence for durability, and auto-scaling for traffic.
