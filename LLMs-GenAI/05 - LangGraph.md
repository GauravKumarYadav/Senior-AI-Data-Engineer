---
tags: [ai, langgraph, agents, phase-5]
phase: 5
status: not-started
priority: high
---

# 🕸️ LangGraph

> **Phase:** 5 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[04 - LangChain]], [[06 - AI Agents]], [[01 - LLM Fundamentals]], [[02 - Prompt Engineering]]

---

## Checklist

### Graph Architecture
- [ ] Why LangGraph: controllable, stateful, multi-step agent workflows
- [ ] `StateGraph`: define graph with typed state
- [ ] Nodes: Python functions that take state, return state updates
- [ ] Edges: connections between nodes (unconditional)
- [ ] Conditional edges: route based on state (branching logic)
- [ ] `START` and `END`: special nodes for graph entry/exit
- [ ] `add_node()`, `add_edge()`, `add_conditional_edges()`
- [ ] `graph.compile()` → runnable graph

### State Management
- [ ] State as `TypedDict`: define state schema with type hints
- [ ] Reducers: how state updates are merged (append to list, overwrite, custom)
- [ ] `Annotated[list, operator.add]` — list append reducer
- [ ] `MessagesState`: pre-built state for chat applications
- [ ] State channels: multiple independent state values
- [ ] Accessing state: node functions receive full state dict

### Agent Patterns
- [ ] ReAct agent: LLM → tool call → observation → LLM → ...
  - [ ] Create: `create_react_agent(model, tools)` helper
  - [ ] Custom: build with nodes (agent, tool_executor) + conditional edges
- [ ] Plan-and-Execute: planner creates steps → executor runs them → replan
- [ ] Reflection: agent evaluates own output, iterates to improve
- [ ] Multi-agent: supervisor routes to specialized agents
  - [ ] Supervisor pattern: one LLM decides which agent to invoke
  - [ ] Hierarchical: nested supervisors + worker agents
  - [ ] Collaborative: agents communicate via shared state

### Human-in-the-Loop
- [ ] `interrupt()`: pause graph execution, wait for human input
- [ ] Approval workflows: agent proposes action → human approves/rejects
- [ ] Editing state: human can modify state before resuming
- [ ] Review nodes: specific nodes designated for human review
- [ ] `Command`: resume graph with human-provided input

### Checkpointing & Persistence
- [ ] `MemorySaver`: in-memory checkpointer (development)
- [ ] `SqliteSaver`: persistent checkpoints to SQLite
- [ ] `PostgresSaver`: production-grade persistence
- [ ] Thread-based: each conversation gets a `thread_id`
- [ ] Time-travel debugging: resume from any previous checkpoint
- [ ] `get_state()`, `update_state()` — inspect and modify state

### Subgraphs & Composition
- [ ] Subgraphs: nest one graph inside another as a node
- [ ] Shared state: parent and subgraph can share state keys
- [ ] Parallel nodes: `Send()` — fan-out to multiple instances
- [ ] Map-reduce: parallel processing → aggregate results

### Streaming
- [ ] `stream()`: yield events as graph executes
- [ ] Stream modes: `values` (full state), `updates` (node outputs), `messages` (LLM tokens)
- [ ] `astream_events()`: fine-grained async streaming
- [ ] Token-by-token streaming from LLM nodes

### LangGraph Platform
- [ ] LangGraph Server: deploy graphs as APIs
- [ ] LangGraph Studio: visual debugger, step through execution
- [ ] LangGraph Cloud: hosted deployment
- [ ] API: `/threads`, `/runs`, `/assistants` endpoints

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] LangGraph docs: https://langchain-ai.github.io/langgraph/
- [ ] LangGraph tutorials: https://langchain-ai.github.io/langgraph/tutorials/
- [ ] LangGraph Academy (free course)
- [ ] Harrison Chase: LangGraph talks (YouTube)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Build a ReAct agent with LangGraph that can search and calculate
- Design a multi-agent system with a supervisor using LangGraph
- How does LangGraph differ from LangChain chains?
- Implement human-in-the-loop approval in a LangGraph agent
