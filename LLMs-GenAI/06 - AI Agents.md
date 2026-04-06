---
tags: [ai, agents, phase-5]
phase: 5
status: not-started
priority: high
---

# 🤖 AI Agents

> **Phase:** 5 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[05 - LangGraph]], [[04 - LangChain]], [[02 - Prompt Engineering]], [[01 - LLM Fundamentals]]

---

## Checklist

### Agent Architectures
- [ ] Single-agent: one LLM with tools, ReAct loop
- [ ] Multi-agent: multiple specialized LLMs collaborating
- [ ] Hierarchical: supervisor → sub-supervisors → workers
- [ ] Peer-to-peer: agents communicate directly (less common)
- [ ] Sequential: pipeline of agents, each handles one step
- [ ] DAG-based: complex workflows with branching (LangGraph)

### Tool Use
- [ ] Function calling: model outputs structured tool invocations
- [ ] Tool selection: model decides which tool based on query + tool descriptions
- [ ] Tool composition: chain tool calls for multi-step tasks
- [ ] MCP (Model Context Protocol): standardized tool/resource protocol
  - [ ] MCP servers: expose tools, resources, prompts
  - [ ] MCP clients: LLM applications that consume MCP servers
  - [ ] Transport: stdio, SSE — local vs remote
- [ ] Custom tools: API wrappers, database queries, code execution
- [ ] Tool errors: graceful handling, retry logic, fallback tools

### Planning & Reasoning
- [ ] Task decomposition: break complex tasks into sub-tasks
- [ ] Plan-and-execute: generate plan → execute steps → replan on failure
- [ ] Self-reflection: agent evaluates own output quality
- [ ] Iterative refinement: multiple passes to improve answer
- [ ] Self-critique: agent identifies mistakes, proposes corrections
- [ ] Backtracking: abandon failed paths, try alternatives

### Memory for Agents
- [ ] Short-term memory: conversation context, working memory (context window)
- [ ] Long-term memory: vector store for past interactions, learned preferences
- [ ] Episodic memory: recall specific past interactions/tasks
- [ ] Semantic memory: general knowledge accumulated over time
- [ ] Procedural memory: learned workflows, SOPs
- [ ] Memory management: what to store, when to retrieve, when to forget

### Multi-Agent Frameworks
- [ ] LangGraph: graph-based, most control, production-ready (see [[05 - LangGraph]])
- [ ] CrewAI: role-based agents, crew/task model, simpler API
  - [ ] Agents: role, goal, backstory, tools
  - [ ] Tasks: description, expected output, agent assignment
  - [ ] Crew: agents + tasks + process (sequential/hierarchical)
- [ ] AutoGen (Microsoft): conversable agents, group chat pattern
- [ ] Comparison: flexibility vs ease of use vs production readiness

### Agent Evaluation
- [ ] Task completion rate: did the agent achieve the goal?
- [ ] Tool-use accuracy: correct tool, correct arguments
- [ ] Efficiency: number of steps, tokens used, latency
- [ ] Safety: did the agent stay within bounds?
- [ ] Robustness: handling edge cases, adversarial inputs
- [ ] Benchmarks: SWE-bench (coding), GAIA (general), ToolBench (tools)
- [ ] Human evaluation: quality of final output, user satisfaction

### Agent Safety & Reliability
- [ ] Sandboxing: restrict agent's execution environment
- [ ] Permission systems: what tools/actions require approval
- [ ] Rate limiting: prevent runaway agent loops
- [ ] Output validation: check agent outputs before acting
- [ ] Audit logging: track all agent decisions and actions
- [ ] Fallback strategies: escalate to human on uncertainty

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] LangGraph docs: agent architectures
- [ ] CrewAI docs: https://docs.crewai.com/
- [ ] AutoGen docs: https://microsoft.github.io/autogen/
- [ ] MCP docs: https://modelcontextprotocol.io/
- [ ] Lilian Weng: "LLM Powered Autonomous Agents" (blog)
- [ ] Andrew Ng: AI Agents course (DeepLearning.AI)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Design a multi-agent system for automated code review
- How do you evaluate agent performance?
- Compare LangGraph vs CrewAI — when to use each?
- What is MCP and how does it standardize tool use?
