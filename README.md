# Rust LangGraph

<div align="center">

**Graph-native LLM workflows in Rust — inspired by LangGraph, built by the community**

[![Crates.io](https://img.shields.io/crates/v/rust-langgraph.svg)](https://crates.io/crates/rust-langgraph)
[![Documentation](https://docs.rs/rust-langgraph/badge.svg)](https://docs.rs/rust-langgraph)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

*This crate is **not** the official [LangGraph](https://github.com/langchain-ai/langgraph) from LangChain. It is an independent Rust port with a similar programming model.*

</div>

## Overview

**Rust LangGraph** (`rust-langgraph` on crates.io) is a community library for building stateful, multi-actor applications with Large Language Models (LLMs). It provides a graph-based execution model inspired by Google’s Pregel and by LangGraph’s ideas, with checkpointing, streaming, and conditional routing.

### Key Features

- **🔄 Stateful Execution**: Build graphs where nodes communicate through shared state with automatic merging
- **💾 Checkpointing**: Save and resume execution at any point with memory, SQLite, or PostgreSQL backends
- **📡 Streaming**: Stream events as the graph executes for real-time feedback
- **🔀 Conditional Logic**: Dynamic routing based on state with support for parallel execution
- **🔁 Cycles**: Create feedback loops and iterative processes
- **🧑‍💻 Human-in-the-Loop**: Pause execution for human input and resume seamlessly
- **🚀 High Performance**: Built in Rust for speed and safety
- **🔌 LLM Integration**: Built-in support for OpenAI, Anthropic, and Ollama

## Quick Start

Add **rust-langgraph** to your `Cargo.toml`:

```toml
[dependencies]
rust-langgraph = "0.1"
tokio = { version = "1", features = ["full"] }
```

### Simple Example

```rust
use rust_langgraph::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, Serialize, Deserialize)]
struct MyState {
    count: i32,
}

impl State for MyState {
    fn merge(&mut self, other: Self) -> Result<(), Error> {
        self.count += other.count;
        Ok(())
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a graph
    let mut graph = StateGraph::new();
    
    // Add a node
    graph.add_node("increment", |mut state: MyState, _config: &Config| async move {
        state.count += 1;
        Ok(state)
    });
    
    graph.set_entry_point("increment");
    graph.set_finish_point("increment");
    
    // Compile and run
    let mut app = graph.compile(None)?;
    let result = app.invoke(MyState { count: 0 }, Config::default()).await?;
    
    println!("Final count: {}", result.count); // Output: Final count: 1
    Ok(())
}
```

## Core Concepts

### StateGraph

The `StateGraph` is the main way to build graphs. It uses a builder pattern to declaratively define nodes and edges:

```rust
let mut graph = StateGraph::new();

graph.add_node("process", |state, _config| async move {
    // Your logic here
    Ok(state)
});

graph.set_entry_point("process");
graph.set_finish_point("process");

let app = graph.compile(None)?;
```

### Nodes

Nodes are the computational units. They take state and return updated state:

```rust
graph.add_node("my_node", |mut state: MyState, _config: &Config| async move {
    state.value += 1;
    Ok(state)
});
```

### Edges

Connect nodes with static or conditional edges:

```rust
// Static edge
graph.add_edge("node1", "node2");

// Conditional edge
graph.add_conditional_edges("node1", |state: &MyState| async move {
    if state.value > 10 {
        Ok(BranchResult::single("node2"))
    } else {
        Ok(BranchResult::single("node3"))
    }
});
```

### Checkpointing

Save and resume execution with checkpointing:

```rust
use std::sync::Arc;

// In-memory checkpointer (also available: SQLite, PostgreSQL)
let checkpointer = Arc::new(MemorySaver::new());
let app = graph.compile(Some(checkpointer))?;

// Execute with a thread ID for checkpoint isolation
let config = Config::new().with_thread_id("user-123");
let result = app.invoke(initial_state, config.clone()).await?;

// Resume from checkpoint
let snapshot = app.get_state(&config).await?;
```

### Streaming

Stream events as the graph executes:

```rust
let mut stream = app.stream(initial_state, config, StreamMode::Values).await?;

while let Some(event) = stream.next().await {
    match event? {
        StreamEvent::Values { data, .. } => {
            println!("State update: {:?}", data);
        }
        _ => {}
    }
}
```

## LLM Integration

LangGraph includes built-in adapters for popular LLM providers:

### Ollama (Local Models)

```rust
use rust_langgraph::llm::ollama::OllamaAdapter;
use rust_langgraph::llm::ChatModel;
use rust_langgraph::state::Message;

let adapter = OllamaAdapter::new("llama3.1:8b");
let messages = vec![Message::user("What is Rust?")];
let response = adapter.invoke(&messages).await?;

println!("Response: {}", response.content);
```

### OpenAI

```rust
use rust_langgraph::llm::openai::OpenAIAdapter;

let adapter = OpenAIAdapter::with_api_key("gpt-4", "sk-...");
let messages = vec![Message::user("Explain async Rust")];
let response = adapter.invoke(&messages).await?;
```

### Anthropic Claude

```rust
use rust_langgraph::llm::anthropic::AnthropicAdapter;

let adapter = AnthropicAdapter::with_api_key("sk-ant-...")
    .with_model("claude-3-5-sonnet-20241022");
    
let messages = vec![Message::user("Write a poem about Rust")];
let response = adapter.invoke(&messages).await?;
```

### Tool Calling

All adapters support function calling for agentic workflows:

```rust
use rust_langgraph::llm::ToolInfo;

let search_tool = ToolInfo::new(
    "search",
    "Search the web",
    serde_json::json!({
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Search query"}
        },
        "required": ["query"]
    })
);

let adapter = OllamaAdapter::new("llama3.1:8b")
    .with_tools(vec![search_tool]);
```

## Advanced Features

### ReAct Agent Pattern

Create an agent that can use tools with real LLMs:

```rust
use rust_langgraph::prebuilt::{create_react_agent, Tool};
use rust_langgraph::llm::ollama::OllamaAdapter;

let search_tool = Tool::new(
    "search",
    "Search for information online",
    |args| async move {
        let query = args["query"].as_str().unwrap_or("");
        // Your search implementation
        Ok(serde_json::json!({"results": format!("Found info about: {}", query)}))
    }
).with_schema(serde_json::json!({
    "type": "object",
    "properties": {
        "query": {"type": "string"}
    },
    "required": ["query"]
}));

// Create model with tools
let model = OllamaAdapter::new("llama3.1:8b")
    .with_tools(vec![search_tool.to_tool_info()]);

// Create ReAct agent
let agent = create_react_agent(model, vec![search_tool])?;

// Use the agent
let result = agent.invoke(
    MessagesState {
        messages: vec![Message::user("Search for Rust tutorials and summarize")]
    },
    Config::default()
).await?;
```

### Custom State Types

Define your own state with custom merge logic:

```rust
#[derive(Clone, Serialize, Deserialize)]
struct CustomState {
    counter: i32,
    items: Vec<String>,
}

impl State for CustomState {
    fn merge(&mut self, other: Self) -> Result<()> {
        self.counter += other.counter;
        self.items.extend(other.items);
        Ok(())
    }
}
```

## Feature Flags

Enable optional features in your `Cargo.toml`:

```toml
[dependencies]
rust-langgraph = { version = "0.1", features = ["sqlite", "openai", "prebuilt"] }
```

Available features:

- `memory-checkpoint` (default): In-memory checkpointing
- `sqlite`: SQLite checkpoint backend
- `postgres`: PostgreSQL checkpoint backend
- `openai`: OpenAI API integration
- `anthropic`: Anthropic API integration
- `ollama`: Ollama API integration
- `prebuilt`: Prebuilt agent patterns (ReAct, etc.)

## Architecture

Rust LangGraph uses a **channel-centric superstep execution** model:

1. **Channels** hold graph state with merge semantics
2. **Nodes** are triggered when their input channels are written
3. **Execution Loop** (Pregel):
   - Find triggered nodes
   - Execute nodes in parallel
   - Apply writes to channels
   - Save checkpoint
   - Repeat until no more triggers

This design enables:
- Automatic parallelization
- Deterministic execution
- Seamless checkpoint/resume
- Flexible state management

## Examples

Check the `examples/` directory for more examples:

- [`01_simple_graph.rs`](examples/01_simple_graph.rs) - Basic nodes and edges
- [`02_conditional_edges.rs`](examples/02_conditional_edges.rs) - Branching logic
- [`03_checkpointing.rs`](examples/03_checkpointing.rs) - Save and resume
- [`04_streaming.rs`](examples/04_streaming.rs) - Stream events in real-time
- [`05_ollama_chat.rs`](examples/05_ollama_chat.rs) - Simple Ollama chat integration
- [`06_react_agent_ollama.rs`](examples/06_react_agent_ollama.rs) - ReAct agent with tools and Ollama
- [`08_custom_state.rs`](examples/08_custom_state.rs) - Custom state types

Run an example:

```bash
# Basic examples
cargo run --example simple_graph

# LLM examples (requires Ollama running locally)
cargo run --example ollama_chat --features ollama
cargo run --example react_agent_ollama --features ollama,prebuilt
```

## Comparison with Python LangGraph

This Rust crate aims for API and behavior alignment where practical with Python LangGraph:

| Feature | Rust LangGraph | Python LangGraph |
|---------|----------------|------------------|
| StateGraph API | ✅ | ✅ |
| Checkpointing | ✅ | ✅ |
| Streaming | ✅ | ✅ |
| Conditional edges | ✅ | ✅ |
| Parallel execution | ✅ | ✅ |
| Human-in-the-loop | ✅ | ✅ |
| Checkpoint format | Compatible | Compatible |
| Performance | Faster | Slower |

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Inspired by [LangGraph](https://github.com/langchain-ai/langgraph) from LangChain
- Built on the Pregel execution model from Google

## Support

- [Documentation](https://docs.rs/rust-langgraph)
- [Examples](examples/)
- [Issue Tracker](https://github.com/yourusername/rust-langgraph/issues)
