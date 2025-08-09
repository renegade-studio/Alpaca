# Roadmap: The Great Gleam Rewrite of Ollama

This document outlines the high-level, strategic roadmap for rewriting the Ollama project from Go into Gleam. The goal is to create a more robust, maintainable, and scalable system by leveraging the strengths of the Gleam language and a modern architectural approach based on Self-Contained Systems (SCS), Hexagonal Architecture, and MVVM.

## Guiding Principles

*   **Robustness & Type Safety**: Leverage Gleam's static typing and functional paradigm to eliminate entire classes of runtime errors.
*   **Modularity & Autonomy**: Deconstruct the monolith into a few highly autonomous, independently deployable Self-Contained Systems.
*   **Clear Boundaries**: Use Hexagonal Architecture within each system to decouple core logic from external concerns (CLI, API, data stores).
*   **User-Centricity**: Each SCS will feature its own simple, functional web UI for management and interaction, applying MVVM principles.
*   **Incremental Adoption**: The rewrite is phased to allow for gradual implementation, testing, and deployment, delivering value at each stage.

---

## Phase 1: The Foundation (Gleam Monolith)

*Goal: Achieve feature parity with the existing Ollama application in a single, well-structured Gleam service. This lays the groundwork for future decomposition.*

*   **Q3 2025: Project Setup & Core Logic**
    *   [ ] Establish the Gleam project structure, build tooling, and dependency management.
    *   [ ] Begin porting the core domain logic: model file parsing, prompt templating, and configuration management.
    *   [ ] Implement the Hexagonal Core: Define the primary `ports` (Gleam type definitions and function signatures) for model runners, storage, and API interactions.

*   **Q4 2025: Adapter Implementation & API**
    *   [ ] Develop the "driving" adapters: a CLI matching the existing `ollama` commands and an HTTP server (the API adapter) providing the REST API.
    *   [ ] Develop "driven" adapters: an initial adapter for interacting with the `llama.cpp` backend. This will likely involve creating a Gleam wrapper around the C API.
    *   [ ] Implement an in-memory or simple file-based adapter for model storage.
    *   [ ] **Milestone**: A fully functional Gleam application that can download, manage, and run models, behaving identically to the original Ollama.

## Phase 2: The First Split (Inference as an SCS)

*Goal: Decompose the monolith by carving out the most critical component—the model inference engine—into its own Self-Contained System.*

*   **Q1 2026: The Runner SCS**
    *   [ ] Create a new, separate Gleam project: `runner_scs`.
    *   [ ] Move all inference-related logic (running models, handling prompts, managing active models) into this SCS.
    *   [ ] This SCS will have its own API, focused exclusively on inference tasks (`/api/generate`, `/api/chat`).
    *   [ ] Develop a simple web UI for the `runner_scs` to view currently loaded models and basic performance metrics (MVVM in action).

*   **Q2 2026: The Management SCS & UI Integration**
    *   [ ] The original monolith now becomes the `management_scs`. Its primary role is to manage the model lifecycle (`pull`, `create`, `rm`, `list`).
    *   [ ] The `management_scs` CLI (`ollama run`) will now delegate to the `runner_scs` API.
    *   [ ] **UI-level Integration**: The web UI for the `management_scs` will link directly to the `runner_scs` UI. There will be no direct backend-to-backend calls between the two systems.

## Phase 3: Full SCS Decomposition

*Goal: Complete the transition to a fully decentralized architecture by splitting out the remaining functionality.*

*   **Q3 2026: The Modelfile SCS**
    *   [ ] Carve out a new `modelfile_scs` from the `management_scs`.
    *   [ ] This new system is responsible for all logic related to `Modelfile` parsing, customization, and creation (`ollama create`).
    *   [ ] It will have its own API and a dedicated UI for building and testing custom models.

*   **Q4 2026: System Refinement & Asynchronous Communication**
    *   [ ] Refine the APIs of all three SCSs (`management`, `runner`, `modelfile`).
    *   [ ] Implement an asynchronous communication channel (e.g., using a message queue or event bus) for tasks like notifying the `runner_scs` when a new model has been downloaded by the `management_scs`. This maintains autonomy while allowing for eventual consistency.
    *   [ ] **Milestone**: The Ollama platform is now composed of multiple, independent, and resilient Self-Contained Systems.

## Phase 4: Ecosystem & Future Growth

*Goal: Solidify the platform and expand its capabilities.*

*   **2027 Onwards**:
    *   [ ] **Enhanced Tooling**: Develop comprehensive testing suites, CI/CD pipelines, and deployment scripts for the multi-SCS environment.
    *   [ ] **Pluggable Runtimes**: Refactor the `runner_scs` to support different model backends beyond `llama.cpp` via a clean adapter interface.
    *   [ ] **Community & Extensibility**: Publish the core Gleam libraries and document the APIs for each SCS to encourage community contributions and third-party integrations.
    *   [ ] **Observability**: Add structured logging, metrics, and tracing to each SCS to ensure the health of the distributed system can be monitored effectively.
