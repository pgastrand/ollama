# Ollama

Get up and running with large language models locally. Ollama provides a simple CLI and a REST API for running, managing, and chatting with LLMs and multimodal models on your own hardware.

## Table of Contents
- [Features](#features)
- [Architecture](#architecture)
- [Installation](#installation)
- [Usage](#usage)
- [API Reference](#api-reference)
- [CLI Reference](#cli-reference)
- [Configuration](#configuration)
- [Development](#development)
- [Testing](#testing)
- [UML Diagrams](#uml-diagrams)
- [Contributing](#contributing)
- [License](#license)

## Features

- **Local LLM Inference** -- Run large language models entirely on your own hardware with GPU acceleration
- **Multimodal Support** -- Chat with vision models that understand images, and generate images with diffusion models
- **OpenAI-Compatible API** -- Drop-in replacement for `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, and more
- **Anthropic-Compatible API** -- Supports `/v1/messages` for Claude-style integrations
- **Model Management** -- Pull, push, create, copy, and delete models via CLI or API
- **Streaming-First Design** -- NDJSON streaming for real-time token generation
- **Tool Calling** -- Models can invoke tools with structured JSON output
- **Thinking / Reasoning** -- Support for reasoning models with configurable depth
- **Multiple Backends** -- CUDA, ROCm, Metal (macOS), Vulkan, and MLX acceleration
- **Subprocess Isolation** -- Runner processes are isolated from the server for stability

## Architecture

Ollama is written primarily in Go with C/C++ backends for model inference. The architecture separates concerns into distinct layers:

- **CLI (`cmd/`)** -- Cobra-based commands that communicate with the server over HTTP
- **HTTP Server (`server/`)** -- Gin-based web server exposing native, OpenAI, and Anthropic APIs
- **Scheduler (`server/sched.go`)** -- Manages GPU memory, model loading/unloading, and request queuing
- **LLM Runner (`llm/`)** -- Spawns subprocess runners (`ollama runner`) that expose a local HTTP API
- **Model Abstraction (`model/`)** -- Plugin-based model architectures with GGUF tensor auto-loading
- **ML Backend (`ml/`)** -- Abstracted tensor operations via the GGML backend
- **Manifest Storage (`manifest/`)** -- Docker-inspired content-addressable blob storage

## Installation

### Prerequisites

- Go 1.26+
- C/C++ compiler (Clang on macOS, GCC/Clang on Linux, MSVC or TDM-GCC on Windows)
- CMake 3.25+
- Optional: CUDA SDK, ROCm, Vulkan SDK, or Metal toolchain (macOS Apple Silicon)

### Build from Source

```bash
# Clone the repository
git clone https://github.com/ollama/ollama.git
cd ollama

# Build native backends
cmake -B build
cmake --build build

# Run the server
go run . serve
```

### Docker

```bash
docker build .

# Build with ROCm support
docker build --build-arg FLAVOR=rocm .
```

### Pre-built Binaries

Download pre-built binaries for macOS, Linux, and Windows from [ollama.com/download](https://ollama.com/download).

## Usage

### Start the Server

```bash
ollama serve
```

The server listens on `127.0.0.1:11434` by default.

### Run a Model

```bash
ollama run llama3.2
```

### Pull a Model

```bash
ollama pull llama3.2
```

### List Local Models

```bash
ollama list
```

### Generate Text (API)

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Why is the sky blue?"
}'
```

### Chat (API)

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {"role": "user", "content": "Why is the sky blue?"}
  ]
}'
```

## API Reference

### Native Ollama API (`/api/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/generate` | Text generation |
| POST | `/api/chat` | Chat completion |
| POST | `/api/embed` | Batch embeddings |
| POST | `/api/embeddings` | Single embedding |
| POST | `/api/pull` | Download a model |
| POST | `/api/push` | Upload a model |
| POST | `/api/create` | Create a model from a Modelfile |
| POST | `/api/copy` | Copy a model |
| DELETE | `/api/delete` | Delete a model |
| POST | `/api/show` | Show model details |
| GET | `/api/tags` | List local models |
| GET | `/api/ps` | List running models |

### OpenAI-Compatible API (`/v1/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/chat/completions` | Chat completions |
| POST | `/v1/completions` | Text completions |
| POST | `/v1/embeddings` | Embeddings |
| GET | `/v1/models` | List models |
| POST | `/v1/images/generations` | Image generation |
| POST | `/v1/audio/transcriptions` | Audio transcription |

### Anthropic-Compatible API

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/messages` | Messages API |

### Key Request Types

- `GenerateRequest` -- Text generation with prompt, images, options, and format controls
- `ChatRequest` -- Multi-turn chat with messages, tools, and thinking options
- `EmbedRequest` -- Batch text embedding with truncation and dimension controls
- `Options` -- Model hyperparameters (temperature, top_p, num_ctx, etc.)

## CLI Reference

| Command | Description |
|---------|-------------|
| `ollama` | Interactive TUI launcher |
| `ollama serve` | Start the HTTP API server |
| `ollama run MODEL [PROMPT]` | Chat or generate with a model |
| `ollama create MODEL` | Create a model from a Modelfile |
| `ollama pull MODEL` | Download a model from the registry |
| `ollama push MODEL` | Upload a model to the registry |
| `ollama list` | List locally available models |
| `ollama ps` | List currently loaded models |
| `ollama show MODEL` | Display model details |
| `ollama stop MODEL` | Unload a model from memory |
| `ollama cp SRC DST` | Copy a model |
| `ollama rm MODEL...` | Delete local models |
| `ollama signin` | Sign in to ollama.com |
| `ollama signout` | Sign out from ollama.com |
| `ollama launch` | Launch third-party integrations |

## Configuration

Ollama is configured through environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_HOST` | `127.0.0.1:11434` | Bind address for the server |
| `OLLAMA_MODELS` | `~/.ollama/models` | Model storage path |
| `OLLAMA_NUM_PARALLEL` | `1` | Number of parallel request slots |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | Maximum concurrently loaded models |
| `OLLAMA_FLASH_ATTENTION` | `false` | Enable flash attention |
| `OLLAMA_KV_CACHE_TYPE` | -- | KV cache quantization type |
| `OLLAMA_GPU_OVERHEAD` | -- | VRAM overhead reservation |
| `OLLAMA_DEBUG` | `false` | Enable debug logging |
| `OLLAMA_EXPERIMENT` | -- | Comma-separated list of experiments |

## Development

### Project Structure

```
ollama/
  api/          -- API request/response types and Go HTTP client
  cmd/          -- CLI commands (Cobra + Bubbletea TUI)
  server/       -- HTTP server, scheduler, and route handlers
  llm/          -- LLM runner subprocess management
  model/        -- Model architecture implementations and interfaces
  ml/           -- ML backend abstraction (GGML)
  manifest/     -- Model manifest and blob storage
  tokenizer/    -- Text tokenization
  template/     -- Prompt template rendering
  parser/       -- Output parsers (thinking blocks, tool calls)
  kvcache/      -- Key-value cache management
  runner/       -- Runner dispatch (llama, ollama, imagegen, mlx)
  auth/         -- Authentication utilities
  discover/     -- Hardware capability detection
  fs/           -- Filesystem and GGUF parsing
  convert/      -- Model format conversion tools
  integration/  -- Integration tests
  x/            -- Experimental features
```

### Common Tasks

```bash
# Run the server
go run . serve

# Run tests
go test ./...

# Build native backends
cmake -B build && cmake --build build

# Run the linter
golangci-lint run
```

## Testing

```bash
# Run all tests
go test ./...

# Run integration tests (requires built backends)
go test ./integration/...

# Run with verbose output
go test -v ./...

# Run a specific package
go test ./server/...
```

## UML Diagrams

### Component Architecture

```mermaid
flowchart TB
    subgraph CLI["CLI Layer"]
        Cobra["Cobra Commands"]
        TUI["Bubbletea TUI"]
    end

    subgraph API["API Layer"]
        NativeAPI["Native API /api/*"]
        OpenAI["OpenAI API /v1/*"]
        Anthropic["Anthropic API /v1/messages"]
    end

    subgraph Server["Server Core"]
        Gin["Gin Router"]
        Scheduler["Model Scheduler"]
        ModelCache["Model Cache"]
        Registry["Registry Client"]
    end

    subgraph LLM["LLM Execution"]
        LLMServer["llmServer"]
        RunnerProc["Runner Subprocess"]
        Tokenizer["Tokenizer"]
        Parser["Output Parser"]
    end

    subgraph Model["Model Engine"]
        ModelInterface["Model Interface"]
        Backend["ML Backend (GGML)"]
        Tensors["Tensor Ops"]
    end

    subgraph Storage["Storage"]
        ManifestStore["Manifest Store"]
        Blobs["Content-Addressable Blobs"]
    end

    Cobra -->|HTTP| Gin
    TUI -->|HTTP| Gin
    Gin --> NativeAPI
    Gin --> OpenAI
    Gin --> Anthropic
    NativeAPI --> Scheduler
    OpenAI --> Scheduler
    Anthropic --> Scheduler
    Scheduler --> LLMServer
    Scheduler --> ModelCache
    LLMServer -->|spawns| RunnerProc
    RunnerProc -->|loads| ModelInterface
    RunnerProc --> Tokenizer
    RunnerProc --> Parser
    ModelInterface --> Backend
    Backend --> Tensors
    Scheduler -->|reads| ManifestStore
    ManifestStore --> Blobs
    Registry -->|pull/push| ManifestStore
```

### Class Diagram -- Core Types

```mermaid
classDiagram
    class Server {
        +addr net.Addr
        +sched *Scheduler
        +defaultNumCtx int
        +requestLogger *inferenceRequestLogger
        +modelCaches *modelCaches
        +Serve() error
        +scheduleRunner() (llm.LlamaServer, *Model, *api.Options, error)
    }

    class Scheduler {
        +runners map[string]*runnerRef
        +Unload(model string)
        +processPending()
    }

    class llmServer {
        +cmd *exec.Cmd
        +addr string
        +Ping() error
        +Completion() error
        +Embedding() error
    }

    class Model {
        <<interface>>
        +Forward(ml.Context, input.Batch) (ml.Tensor, error)
        +Backend() ml.Backend
        +Config() config
    }

    class MultimodalProcessor {
        <<interface>>
        +EncodeMultimodal(ml.Context, []byte) ([]input.Multimodal, error)
        +PostTokenize([]input.Input) []input.Input
    }

    class Base {
        +backend ml.Backend
        +config config
    }

    class Backend {
        <<interface>>
        +Close()
        +Load(ctx, progress) error
        +BackendMemory() BackendMemory
        +Get(name string) Tensor
        +NewContext() Context
        +BackendDevices() []DeviceInfo
    }

    class Tensor {
        <<interface>>
    }

    class Context {
        <<interface>>
    }

    class GenerateRequest {
        +Model string
        +Prompt string
        +Images []ImageData
        +Stream *bool
        +Options map[string]any
    }

    class ChatRequest {
        +Model string
        +Messages []Message
        +Tools []Tool
        +Think *ThinkValue
    }

    class Message {
        +Role string
        +Content string
        +Thinking string
        +Images []ImageData
        +ToolCalls []ToolCall
    }

    class Options {
        +Temperature float32
        +TopP float32
        +TopK int
        +NumCtx int
        +NumPredict int
        +RepeatPenalty float32
    }

    class Manifest {
        +SchemaVersion int
        +MediaType string
        +Config Descriptor
        +Layers []Descriptor
    }

    Server --> Scheduler
    Server --> llmServer
    Scheduler --> llmServer
    llmServer --> Model
    Model <|.. Base
    MultimodalProcessor <|.. Base
    Model --> Backend
    Backend --> Tensor
    Backend --> Context
    Server --> GenerateRequest
    Server --> ChatRequest
    ChatRequest --> Message
    GenerateRequest --> Options
    ChatRequest --> Options
    Server --> Manifest
```

### Sequence Diagram -- Chat Request Flow

```mermaid
sequenceDiagram
    actor User
    participant CLI as CLI / curl
    participant Gin as Gin Router
    participant Sched as Scheduler
    participant LLMServer as llmServer
    participant Runner as Runner Subprocess
    participant Model as Model Engine
    participant Backend as ML Backend

    User->>CLI: POST /api/chat
    CLI->>Gin: HTTP request
    Gin->>Gin: Validate request
    Gin->>Sched: scheduleRunner(model, caps, opts)
    Sched->>Sched: Check model cache
    alt Model not loaded
        Sched->>LLMServer: NewLlamaServer(model)
        LLMServer->>Runner: Start subprocess
        Runner->>Model: Load GGUF weights
        Model->>Backend: Initialize backend
        Backend-->>Runner: Ready
        Runner-->>LLMServer: HTTP on ephemeral port
    end
    Sched-->>Gin: runner, model, opts
    Gin->>LLMServer: Chat completion request
    LLMServer->>Runner: Forward HTTP request
    Runner->>Model: Tokenize + Forward pass
    Model->>Backend: Execute graph
    Backend-->>Model: logits tensor
    Model-->>Runner: output tokens
    Runner->>Runner: Sample next token
    Runner-->>LLMServer: NDJSON stream chunk
    LLMServer-->>Gin: Stream chunk
    Gin-->>CLI: NDJSON chunk
    CLI-->>User: Display token
    loop Until generation complete
        Runner-->>LLMServer: Next chunk
        LLMServer-->>Gin: Next chunk
        Gin-->>CLI: Next chunk
        CLI-->>User: Display token
    end
    Runner-->>LLMServer: [DONE]
    LLMServer-->>Gin: Final chunk
    Gin-->>CLI: Final response
    CLI-->>User: Done
```

### State Diagram -- Model Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Unloaded: Server starts
    Unloaded --> Loading: Request arrives
    Loading --> Loaded: Runner ready
    Loaded --> Processing: Active request
    Processing --> Loaded: Request complete
    Loaded --> Unloading: Keep-alive expired
    Loaded --> Unloading: Memory pressure
    Unloading --> Unloaded: Process terminated
    Loading --> Error: Load failure
    Error --> Unloaded: Cleanup
```

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for detailed guidelines.

- Open an issue to discuss non-trivial changes before submitting a PR
- Write commit messages in the format: `package: short description`
- Include tests that validate behavior, not implementation
- Keep dependencies to a minimum
- Reach out on [Discord](https://discord.gg/ollama) if you need help

## License

MIT License. See [LICENSE](./LICENSE) for details.
