# Rust LangGraph

<div align="center">

**Graph-native LLM workflows in Rust — inspired by [LangGraph](https://github.com/langchain-ai/langgraph), built by the community**

[![Crates.io](https://img.shields.io/crates/v/rust-langgraph.svg)](https://crates.io/crates/rust-langgraph)
[![Documentation](https://docs.rs/rust-langgraph/badge.svg)](https://docs.rs/rust-langgraph)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

*Not affiliated with LangChain. This is an independent Rust library with a similar programming model.*

</div>

---

## Table of contents

1. [What this crate is](#what-this-crate-is)
2. [Who should use it](#who-should-use-it)
3. [Installation](#installation)
4. [Five-minute tutorial](#five-minute-tutorial)
5. [Core concepts](#core-concepts)
6. [Graph API reference](#graph-api-reference)
7. [LLMs and agents](#llms-and-agents)
8. [Feature flags](#feature-flags)
9. [Project layout](#project-layout)
10. [Examples](#examples)
11. [Docs for humans vs. tooling](#documentation)
12. [Comparison with Python LangGraph](#comparison-with-python-langgraph)
13. [License & acknowledgments](#license)

---

## What this crate is

**Rust LangGraph** (crate name: **`rust-langgraph`**, Rust import: **`rust_langgraph`**) helps you build **stateful workflows** as a **directed graph**:

- **Nodes** are async functions (or types implementing `Node`) that read and return **state**.
- **Edges** connect nodes: fixed edges or **conditional** edges that choose the next node from state.
- **Execution** follows a Pregel-style loop: run nodes, merge state, optionally **checkpoint**, repeat until done.

Use it for multi-step LLM apps, tool-calling agents, branching pipelines, and anything that fits “steps + shared state + optional loops.”

---

## Who should use it

| You want… | Use… |
|-----------|------|
| A small graph without LLMs | `StateGraph` + custom `State` |
| Chat + tools (ReAct-style) | `prebuilt` feature: `create_react_agent`, `Tool`, `ToolNode` |
| Local models | `ollama` feature: `llm::ollama::OllamaAdapter` |
| OpenAI / Anthropic APIs | `openai` / `anthropic` features under `rust_langgraph::llm` |
| [OpenRouter](https://openrouter.ai/docs/quickstart) (many providers, one API) | `openrouter` feature: `llm::openrouter::OpenRouterAdapter` |
| Persistence between runs | `MemorySaver` or DB backends (`sqlite` / `postgres` features) |

---

## Installation

**`Cargo.toml`:**

```toml
[dependencies]
rust-langgraph = "0.1"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
# Optional: use with `futures::StreamExt` when consuming `CompiledGraph::stream`
futures = "0.3"
```

**Import in Rust:**

```rust
use rust_langgraph::prelude::*;
```

Optional: enable features you need (see [Feature flags](#feature-flags)).

**Requirements:**

- Rust 2021 edition
- Async runtime: **Tokio** (the library is async-first)

---

## Five-minute tutorial

### 1. Define state

State must implement [`State`](https://docs.rs/rust-langgraph/latest/rust_langgraph/state/trait.State.html): `Clone`, `Serialize`/`Deserialize`, `Debug`, and **`merge`** (how updates from multiple nodes combine).

```rust
use rust_langgraph::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, Serialize, Deserialize)]
struct AppState {
    n: i32,
}

impl State for AppState {
    fn merge(&mut self, other: Self) -> Result<()> {
        self.n += other.n;
        Ok(())
    }
}
```

### 2. Build a graph

```rust
let mut graph = StateGraph::new();

graph.add_node("step", |state: AppState, _config: &Config| async move {
    Ok(AppState { n: state.n + 1 })
});

graph.set_entry_point("step");
graph.set_finish_point("step");

let mut app = graph.compile(None)?;
let out = app.invoke(AppState { n: 0 }, Config::default()).await?;
// out.n == 1
```

### 3. Conditional routing (optional)

Use `pregel::BranchResult`: `single("node_id")`, `end()`, or more advanced variants.

```rust
use rust_langgraph::pregel::BranchResult;

graph.add_conditional_edges("step", |state: &AppState| {
    let n = state.n;
    async move {
        if n >= 3 {
            Ok(BranchResult::end())
        } else {
            Ok(BranchResult::single("step"))
        }
    }
});
```

---

## Core concepts

### Mental model

1. You declare **nodes** by name and pass a closure or a type implementing `Node<S>`.
2. You connect nodes with **`add_edge`** or **`add_conditional_edges`**.
3. You set **`set_entry_point`** (where execution starts) and usually **`set_finish_point`** (terminal nodes).
4. **`compile`** produces a **`CompiledGraph`** you call **`invoke`** or **`stream`** on.

### State

- **`State`** — your domain data; **`merge`** defines reducer semantics when multiple writes occur.
- **`MessagesState`** — built-in chat history for LLM flows (`messages: Vec<Message>`).
- **`Message`**, **`ToolCall`** — roles `user`, `assistant`, `system`, `tool`; tool calls and tool results.

### Graph types

| Type | Role |
|------|------|
| `StateGraph<S>` | Builder: `add_node`, `add_edge`, `add_conditional_edges`, `compile` |
| `CompiledGraph<S>` | Runnable: `invoke`, `stream`, checkpoint helpers when configured |

### Checkpointing

Pass a **`BaseCheckpointSaver`** (e.g. **`MemorySaver`** with feature `memory-checkpoint`) into **`compile(Some(checkpointer))`**. Use **`Config::with_thread_id`** so each conversation/thread has isolated checkpoints.

### Streaming

Use **`CompiledGraph::stream`** with **`StreamMode`** and handle **`StreamEvent`** variants. Add **`futures`** to your app for **`StreamExt`**:

```rust
use futures::StreamExt;
use rust_langgraph::prelude::*;

// let mut app: CompiledGraph<MyState> = ...;
let mut stream = app
    .stream(initial_state, config, StreamMode::Values)
    .await?;

while let Some(event) = stream.next().await {
    match event? {
        StreamEvent::Values { data, .. } => {
            println!("{:?}", data);
        }
        _ => {}
    }
}
```

For token-level LLM streams, call **`ChatModel::stream`** on **`OllamaAdapter`** / **`OpenAIAdapter`** / **`OpenRouterAdapter`** / **`AnthropicAdapter`**.

---

## Graph API reference

| Method | Purpose |
|--------|---------|
| `StateGraph::new()` | Empty graph |
| `add_node(name, node)` | Register a node (`impl Node<S>` or closure) |
| `add_edge(from, to)` | Always go `from → to` |
| `add_conditional_edges(from, branch)` | `branch` returns `BranchResult` (next node(s) or end) |
| `set_entry_point(name)` | First node(s) to run |
| `set_finish_point(name)` | Mark terminal nodes |
| `compile(checkpointer)` | Build `CompiledGraph` |

**Node closure shape:**

```rust
|state: S, config: &Config| async move { Result<S> }
```

Use an explicit `&Config` parameter (not `_`) if the compiler complains about lifetimes in complex graphs.

---

## LLMs and agents

Enable features: **`ollama`**, **`openai`**, **`openrouter`**, **`anthropic`**, and often **`prebuilt`** for agents.

### Direct chat (no graph)

**Local (Ollama):**

```rust
use rust_langgraph::llm::ollama::OllamaAdapter;
use rust_langgraph::llm::ChatModel;
use rust_langgraph::state::Message;

let model = OllamaAdapter::new("llama3.1:8b");
let reply = model.invoke(&[Message::user("Hello")]).await?;
```

**OpenRouter** ([quickstart](https://openrouter.ai/docs/quickstart)) — OpenAI-compatible HTTP API; set `OPENROUTER_API_KEY` and use a router model id (e.g. `openai/gpt-4o-mini`):

```rust
use rust_langgraph::llm::openrouter::OpenRouterAdapter;
use rust_langgraph::llm::ChatModel;
use rust_langgraph::state::Message;

let model = OpenRouterAdapter::with_api_key(
    "openai/gpt-4o-mini",
    std::env::var("OPENROUTER_API_KEY").unwrap(),
);
let reply = model.invoke(&[Message::user("Hello")]).await?;
```

### ReAct agent (graph with `agent` ↔ `tools` loop)

1. Define **`Tool`** instances with **`Tool::new(...).with_schema(json_schema)`**.
2. Bind the same tools to the model: e.g. **`OllamaAdapter::new(...).with_tools(vec![t.to_tool_info(), ...])`**.
3. Call **`create_react_agent(model, tools)`** → get a **`CompiledGraph<MessagesState>`**.
4. **`invoke(MessagesState { messages: vec![Message::user("...")] }, Config::default())`**.

See **`examples/06_react_agent_ollama.rs`** for a full runnable flow.

### Validation

**`prebuilt::validate_chat_history`** checks that every assistant **`tool_calls`** entry has a matching **`tool`** message (aligned with common LangGraph-style rules).

---

## Feature flags

```toml
rust-langgraph = { version = "0.1", features = ["ollama", "prebuilt", "openai", "openrouter"] }
```

| Feature | What it enables |
|---------|------------------|
| `memory-checkpoint` | **Default.** In-memory `MemorySaver` |
| `sqlite` | SQLite checkpoint backend (`sqlx`) |
| `postgres` | PostgreSQL checkpoint backend |
| `openai` | `llm::openai::OpenAIAdapter` + `reqwest` + `async-openai` |
| `openrouter` | `llm::openrouter::OpenRouterAdapter` (OpenAI-compatible client → `https://openrouter.ai/api/v1`) |
| `anthropic` | `llm::anthropic::AnthropicAdapter` |
| `ollama` | `llm::ollama::OllamaAdapter` |
| `prebuilt` | `create_react_agent`, `Tool`, `ToolNode`, `validate_chat_history` |

---

## Project layout

```
src/
  lib.rs              # Crate root, prelude
  graph/              # StateGraph, CompiledGraph
  pregel/             # Execution engine, Branch, BranchResult
  state.rs            # State, Message, MessagesState
  nodes.rs            # Node trait
  checkpoint/         # Checkpoint types & saver trait
  checkpoint_backends/
  llm/                # ChatModel, Ollama / OpenAI / OpenRouter / Anthropic (feature-gated)
  prebuilt/           # ReAct agent, tools (feature-gated)
examples/             # Runnable examples (see table below)
tests/                # Integration tests (e.g. Ollama, --ignored)
```

**Full API details:** run `cargo doc --open` or visit [docs.rs/rust-langgraph](https://docs.rs/rust-langgraph).

---

## Examples

| Example | Command | Features |
|---------|---------|----------|
| Minimal graph | `cargo run --example simple_graph` | default |
| Branching | `cargo run --example conditional_edges` | default |
| Checkpoints | `cargo run --example checkpointing` | default |
| Streaming | `cargo run --example streaming` | default |
| Ollama chat | `cargo run --example ollama_chat` | `ollama` |
| ReAct + Ollama | `cargo run --example react_agent_ollama` | `ollama`, `prebuilt` |
| OpenRouter chat | `cargo run --example openrouter_chat` | `openrouter` |
| Custom state | `cargo run --example custom_state` | default |

```bash
cd rust-langgraph
cargo run --example simple_graph
cargo run --example ollama_chat --features ollama
cargo run --example react_agent_ollama --features ollama,prebuilt
set OPENROUTER_API_KEY=sk-or-v1-...
cargo run --example openrouter_chat --features openrouter
```

---

## Documentation

- **Human readers:** this README + **[`AGENTS.md`](AGENTS.md)** (also useful for contributors) + **rustdoc** (`cargo doc --no-deps --open`).
- **AI coding agents:** read **`AGENTS.md`** first — it lists crate name, features, patterns, and pitfalls so assistants generate correct `rust-langgraph` code.

---

## Comparison with Python LangGraph

This crate targets **similar ideas** (state graph, checkpoints, agents) but is a **separate implementation**. APIs and wire formats are aligned where practical; behavior may differ in edge cases. For the official Python stack, use LangChain’s LangGraph.

| Area | Rust LangGraph | Python LangGraph |
|------|----------------|------------------|
| Language | Rust | Python |
| Package | `rust-langgraph` / `rust_langgraph` | `langgraph` |
| Official? | Community | LangChain |

---

## Contributing

Issues and PRs are welcome. Please keep changes focused and match existing style.

---

## License

MIT — see [LICENSE](LICENSE).

## Acknowledgments

- Inspired by [LangGraph](https://github.com/langchain-ai/langgraph) (LangChain).
- Execution model influenced by Google’s Pregel.

## Links

- [docs.rs/rust-langgraph](https://docs.rs/rust-langgraph)
- [Examples](examples/)
