# Implementation Plan: Ollama in Gleam

This document provides the detailed technical specification for the rewrite of the Ollama platform into Gleam. It is guided by the principles and phases outlined in `roadmap.md`.

## 1. Core Technologies & Libraries

*   **Language**: [Gleam](https://gleam.run/)
*   **Web Server / API**: [Wisp](https://hex.pm/packages/wisp) (underlying the web UI and REST APIs for each SCS)
*   **CLI Framework**: [Lustre](https://hex.pm/packages/lustre) (for building the `ollama` command-line interface)
*   **JSON Handling**: [Gleason](https://hex.pm/packages/gleason)
*   **FFI (Foreign Function Interface)**: Gleam's native Erlang/Elixir FFI will be used to interface with a wrapper around `llama.cpp`.
*   **Build & Package Management**: Gleam's default build tool.

## 2. Project Structure (Phase 1 Monolith)

The initial project will be a monolith, but structured for an easy transition to SCS. The directory layout will strictly enforce the Hexagonal Architecture.

```
/ollama_gleam
├── gleam.toml
├── build/
├── src
│   ├── ollama # Application Core (The Hexagon)
│   │   ├── model.gleam        # Core domain types: Model, Modelfile, Template
│   │   ├── config.gleam       # Configuration logic
│   │   └── ports              # Interfaces for external interaction
│   │       ├── runner.gleam   # Port for the model inference engine
│   │       └── storage.gleam  # Port for model storage and retrieval
│   │
│   ├── adapters             # Implementations of the ports
│   │   ├── cli                # CLI Driving Adapter
│   │   │   └── main.gleam     # Entrypoint for the CLI app
│   │   ├── web                # Web Driving Adapter (API & UI)
│   │   │   ├── server.gleam   # Wisp server setup
│   │   │   ├── controller.gleam # API request handlers
│   │   │   └── viewmodel.gleam  # MVVM: Prepares data for the UI
│   │   ├── llama_runner.gleam # Driven Adapter for llama.cpp
│   │   └── file_storage.gleam # Driven Adapter for local filesystem storage
│   │
│   └── main.gleam             # Main entrypoint dispatching to CLI or Web
│
└── test
    ├── core_test.gleam
    └── adapters_test.gleam
```

## 3. Hexagonal Core: Ports Definition

The `src/ollama/ports/` directory defines the contract between the application core and the outside world. All types are illustrative and will be refined.

### `ports/storage.gleam`

This port defines how the application interacts with model storage, abstracting away the filesystem.

```gleam
import gleam/result
import ollama/model

pub type ModelStorage {
  // A function that returns a list of all models.
  list: fn() -> Result(List(model.Model), Error),

  // A function that retrieves a model by its tag.
  get: fn(String) -> Result(model.Model, Error),

  // A function that saves a model.
  save: fn(model.Model, BitString) -> Result(Nil, Error),

  // A function that deletes a model by its tag.
  delete: fn(String) -> Result(Nil, Error),
}
```

### `ports/runner.gleam`

This port defines the contract for running inference on a model.

```gleam
import gleam/io
import ollama/model

// An opaque type representing a loaded model instance.
pub type LoadedModel

pub type Runner {
  // A function to load a model into memory.
  load: fn(model.Model) -> Result(LoadedModel, Error),

  // A function to unload a model from memory.
  unload: fn(LoadedModel) -> Result(Nil, Error),

  // A function to generate a response from a prompt.
  generate: fn(LoadedModel, String) -> io.Stream(Result(String, Error)),
}
```

## 4. Architectural Patterns in Practice

### Hexagonal Architecture

*   **The Core (`src/ollama/`)**: Contains pure business logic and type definitions. It has no knowledge of Wisp, Lustre, or the filesystem. It only knows about the `ports`.
*   **Driving Adapters (`src/adapters/cli`, `src/adapters/web`)**: These initiate actions. The `cli` adapter calls core functions in response to user commands. The `web` adapter calls them in response to HTTP requests.
*   **Driven Adapters (`src/adapters/llama_runner.gleam`, `src/adapters/file_storage.gleam`)**: These are implementations of the ports defined in the core. The core application is given concrete implementations of these adapters at startup (dependency injection).

### MVVM (Model-View-ViewModel)

This pattern will be implemented in the `web` adapter for the UI.

*   **Model**: The core domain types defined in `src/ollama/model.gleam`.
*   **View**: A simple HTML/CSS/JavaScript frontend. It will be served by the Wisp server and will communicate with the backend via JSON over HTTP.
*   **ViewModel**: The `src/adapters/web/viewmodel.gleam` module. When the View needs data (e.g., a list of models), it makes an API call. The `controller.gleam` receives this, calls the core application logic, and then uses a function from `viewmodel.gleam` to format the core `Model` types into a JSON structure that is easy for the View to consume.

## 5. SCS Decomposition Plan (Post-Phase 1)

As we move to Phases 2 and 3, the single project will be split. The process will be:

1.  Create a new Gleam project for the new SCS (e.g., `runner_scs`).
2.  Copy the relevant core logic and adapters into the new project.
3.  The original project (now the `management_scs`) will remove the code that was moved.
4.  The `management_scs`'s CLI adapter will be updated to make an HTTP call to the new `runner_scs`'s API instead of calling the logic directly.

### SCS API Definitions (High-Level)

*   **Management SCS API**:
    *   `GET /api/models`: List all local models.
    *   `POST /api/pull`: Download a model from a remote registry.
    *   `DELETE /api/models/{tag}`: Delete a model.
*   **Runner SCS API**:
    *   `POST /api/generate`: Stream a response from a given model and prompt.
    *   `POST /api/chat`: A stateful version of the generate endpoint.
    *   `GET /api/loaded`: List models currently loaded in memory.
*   **Modelfile SCS API**:
    *   `POST /api/create`: Create a new model from a Modelfile.
    *   `POST /api/validate`: Validate the syntax of a Modelfile.

## 6. Testing Strategy

*   **Unit Tests (`test/core_test.gleam`)**: These will test the pure functions within the application core (`src/ollama/`). They will be fast and have no I/O.
*   **Integration Tests (`test/adapters_test.gleam`)**: These will test the adapters against mocked versions of the ports. For example, testing the `web` adapter by sending it mock HTTP requests and asserting on the responses, using a mocked `ModelStorage` port that returns predefined data.
*   **End-to-End (E2E) Tests**: A separate test suite (likely a shell script or a small test application) will be created to run the compiled `ollama` binary. It will execute CLI commands (`ollama pull`, `ollama run`) and hit the live API endpoints, asserting that the entire system works correctly. This is the final quality gate.
```
