# 🧠 MCP Agent Kit

<div align="center">

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/Python-3.11%2B-green.svg)](https://python.org)
[![MCP 1.0](https://img.shields.io/badge/MCP-1.0-orange.svg)](https://modelcontextprotocol.io)
[![CI](https://github.com/yourusername/mcp-agent-kit/actions/workflows/ci.yml/badge.svg)](/.github/workflows/ci.yml)
[![Coverage](https://img.shields.io/badge/coverage-94%25-brightgreen.svg)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

**A production-grade, modular framework for building AI agents using the Model Context Protocol (MCP).**

[Documentation](docs/) · [Examples](examples/) · [Contributing](CONTRIBUTING.md) · [Roadmap](ROADMAP.md)

</div>

---

## 🔍 Overview

MCP Agent Kit is a batteries-included agent infrastructure built on top of Anthropic's [Model Context Protocol](https://modelcontextprotocol.io). It solves three core problems that plague existing agent frameworks:

| Problem | Our Solution |
|---|---|
| Monolithic, hard-to-extend tool integrations | **Composable Skills** — plug in any capability in <10 lines |
| No standard inter-agent communication | **MCP-native Orchestrator** — structured multi-agent graphs |
| Black-box agent execution | **Built-in OpenTelemetry tracing** — every call, every token |

```
┌──────────────────────────────────────────────────────────┐
│                      Your Application                    │
├──────────────────────────────────────────────────────────┤
│                    MCP Agent Kit                         │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  Orchestrator│  │ Skills Reg.  │  │  Observability │  │
│  │  (DAG/Graph) │  │  (Registry)  │  │  (OTEL traces) │  │
│  └──────┬──────┘  └──────┬───────┘  └────────────────┘  │
│         │                │                               │
│  ┌──────▼──────────────▼───────────────────────────┐    │
│  │              Agent Runtime                       │    │
│  │  Memory · Tool Router · Retry · Sandbox          │    │
│  └────────────────────────────────────────────────-─┘    │
└──────────────────────────────────────────────────────────┘
         │                             │
    ┌────▼─────┐                 ┌─────▼─────┐
    │ MCP Tools│                 │  LLM APIs  │
    │(local/rmt)│                 │(Anthropic/│
    └──────────┘                 │  OpenAI)  │
                                 └───────────┘
```

---

## ✨ Key Features

### 🧩 Composable Skills System
Define reusable agent capabilities as typed Python classes. Skills are auto-discovered, versioned, and dependency-resolved at runtime.

```python
from mcp_agent_kit import skill, SkillContext

@skill(name="web_search", version="1.2.0", requires=["http_client"])
async def web_search(ctx: SkillContext, query: str, max_results: int = 5) -> list[SearchResult]:
    """Full-text web search with structured output and citation tracking."""
    return await ctx.tools.search(query, limit=max_results)
```

### ⚡ Zero-Boilerplate Agent Declaration
```yaml
# agents/research_agent.yaml
name: research-agent
version: "1.0"
model: claude-sonnet-4-20250514
skills:
  - web_search@1.2.0
  - summarize@2.0.0
  - cite_sources@1.0.0
memory:
  backend: chroma          # swap to redis/pinecone/in-memory with one line
  context_window: 8000
constraints:
  max_steps: 15
  timeout_seconds: 120
  budget_tokens: 50000
```

### 🔁 Multi-Agent Orchestration via DAG
```python
from mcp_agent_kit import AgentGraph, supervisor

graph = AgentGraph()

# Supervisor automatically routes tasks to best-fit sub-agents
@supervisor(graph, agents=["research", "writer", "critic"])
async def pipeline(task: str) -> FinalOutput:
    research = await graph.run("research", task)
    draft    = await graph.run("writer", research)
    return   await graph.run("critic", draft)
```

### 🧠 Pluggable Memory Backends
```python
# Swap ANY backend with one config change
memory = MemoryFactory.create(
    backend="chroma",          # or "redis" | "pinecone" | "in_memory" | "postgres"
    embedding_model="text-embedding-3-small",
    namespace="project-alpha",
)
```

### 📊 First-Class Observability
Every agent run emits structured OpenTelemetry spans with:
- Tool call latency + token usage per step
- Memory read/write operations
- Agent decision traces (which skill was chosen and why)
- Cost attribution per agent/task

```python
# Zero-config — just set your OTEL collector endpoint
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

### 🔒 Sandboxed Tool Execution
All tool invocations run in an isolated subprocess with configurable resource limits:
```python
sandbox = Sandbox(
    memory_mb=512,
    cpu_quota=0.5,
    network=NetworkPolicy.ALLOWLIST(["api.github.com"]),
    filesystem=FSPolicy.READ_ONLY,
)
```

---

## 🚀 Quick Start

### Installation

```bash
pip install mcp-agent-kit

# With optional extras
pip install "mcp-agent-kit[chroma,redis,observability]"
```

### Your First Agent (60 seconds)

```python
import asyncio
from mcp_agent_kit import Agent, AgentConfig

async def main():
    agent = Agent.from_config("agents/research_agent.yaml")

    result = await agent.run(
        "Summarize the latest research on transformer attention mechanisms"
    )

    print(result.output)
    print(f"Tokens used: {result.usage.total_tokens}")
    print(f"Steps taken: {result.steps}")

asyncio.run(main())
```

---

## 📁 Project Structure

```
mcp-agent-kit/
├── src/
│   ├── agents/           # Agent runtime and lifecycle management
│   │   ├── agent.py      # Core Agent class
│   │   ├── config.py     # YAML/JSON config loading + validation (Pydantic)
│   │   └── lifecycle.py  # Start, pause, resume, cancel semantics
│   ├── skills/           # Skills registry and loader
│   │   ├── registry.py   # Auto-discovery + version resolution
│   │   ├── base.py       # SkillContext, SkillResult types
│   │   └── builtins/     # 20+ built-in skills (search, code, memory, ...)
│   ├── tools/            # MCP tool adapters
│   │   ├── router.py     # Dynamic tool selection via embeddings
│   │   └── adapters/     # GitHub, Slack, Filesystem, HTTP, Shell
│   ├── memory/           # Memory backends
│   │   ├── factory.py    # Backend factory + config
│   │   └── backends/     # Chroma, Redis, Pinecone, Postgres, InMemory
│   ├── orchestrator/     # Multi-agent coordination
│   │   ├── graph.py      # DAG execution engine
│   │   ├── supervisor.py # Supervisor agent pattern
│   │   └── router.py     # Task-to-agent routing
│   ├── observability/    # Tracing, metrics, logging
│   │   ├── tracer.py     # OpenTelemetry instrumentation
│   │   ├── metrics.py    # Token + cost + latency tracking
│   │   └── dashboard/    # Local Streamlit dashboard
│   └── sandbox/          # Isolated tool execution
│       ├── sandbox.py    # Process isolation + resource limits
│       └── policies.py   # Network, FS, CPU/memory policies
├── tests/
│   ├── unit/             # Pure unit tests (no I/O)
│   ├── integration/      # Tests with real MCP servers (docker-compose)
│   └── e2e/              # Full agent run tests
├── examples/
│   ├── coding-assistant/ # GitHub Copilot-style coding agent
│   ├── research-bot/     # Deep research with citation tracking
│   └── multi-agent/      # Supervisor + worker pipeline example
├── docs/
│   ├── architecture/     # System design docs + diagrams
│   ├── api/              # Auto-generated API reference
│   └── guides/           # How-to guides
└── .github/
    ├── workflows/        # CI, release, security scanning
    └── ISSUE_TEMPLATE/   # Bug report + feature request templates
```

---

## 🏗️ Architecture Deep Dive

See [docs/architecture/DESIGN.md](docs/architecture/DESIGN.md) for the full design document including:
- Agent lifecycle state machine
- Skills dependency resolution algorithm
- Memory retrieval + context window packing strategy
- Multi-agent message passing protocol
- Sandbox isolation model

---

## 🧪 Testing

```bash
# Unit tests (fast, no external deps)
pytest tests/unit -v

# Integration tests (requires Docker)
docker-compose -f tests/docker-compose.yml up -d
pytest tests/integration -v

# Full e2e suite
pytest tests/e2e -v --timeout=120

# Coverage report
pytest --cov=src --cov-report=html
```

---

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for:
- Development setup
- Code style guide (we use `ruff` + `mypy`)
- How to add a new built-in skill
- How to add a new memory backend
- PR review process

---

## 📜 License

Apache 2.0 — see [LICENSE](LICENSE).

---

## 🙏 Acknowledgements

Built on top of:
- [Model Context Protocol](https://modelcontextprotocol.io) by Anthropic
- [LangGraph](https://github.com/langchain-ai/langgraph) — inspiration for the DAG orchestration model  
- [OpenTelemetry Python](https://github.com/open-telemetry/opentelemetry-python) — observability instrumentation
