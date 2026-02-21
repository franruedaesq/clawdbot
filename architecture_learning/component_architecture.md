# Component Architecture (The "How - Big Picture")

## Tech Stack Summary

*   **Frontend**: Built with **Lit** (web components) and bundled using **Vite**. It provides a lightweight, fast interface for the control plane.
*   **Backend**: **Node.js** (v22+) runtime using **TypeScript**. The project uses **pnpm** for package management and a monorepo structure.
*   **Persistence**:
    *   **Session State**: Primary conversation history is stored in **JSONL (JSON Lines)** files on the local filesystem. This ensures human-readable, append-only logs of all interactions.
    *   **Vector Memory**: Uses **SQLite** with the `sqlite-vec` extension for semantic search and long-term memory retrieval.
    *   **Configuration**: Stored as standard JSON/JSON5 files.
*   **Browser Automation**: Uses **Playwright** (Chromium) for web-based agent capabilities.

## Internal Modules/Services

OpenClaw follows a **Modular Monolith** architecture. While it runs as a single daemon process, its internal components are distinct and loosely coupled:

1.  **Gateway (The Core)**:
    *   Acts as the central control plane.
    *   Manages the lifecycle of all other components.
    *   Handles routing of messages between Channels and Agents.
    *   Exposes the WebSocket API for the UI and external clients.

2.  **Agent Runtime**:
    *   The "Brain" of the system.
    *   Manages the execution loop: Prompt construction -> LLM Inference -> Tool Execution.
    *   Handles context management (pruning, summarizing) and memory retrieval.

3.  **Channels**:
    *   Pluggable modules that connect to external messaging platforms (Discord, Slack, Telegram, WhatsApp, etc.).
    *   Each channel is responsible for normalizing incoming events into standard `AgentMessage` objects and converting agent responses back to platform-specific formats.

4.  **Skills & Tools**:
    *   A registry of capabilities the agent can use (e.g., `browser`, `code_interpreter`).
    *   Skills can be bundled (built-in) or dynamically loaded from the workspace.

5.  **Cron / Scheduler**:
    *   Manages scheduled tasks and background jobs.

## Communication

*   **WebSocket Control Plane**: The primary internal communication backbone. The Gateway hosts a WebSocket server that the UI, CLI, and even some internal components connect to for real-time updates and control.
*   **RPC (Remote Procedure Call)**: The Agent Runtime communicates with the Gateway and other components via an internal RPC mechanism. This allows the agent logic to be potentially decoupled or run in different contexts (like Docker containers) while still appearing local.
*   **HTTP Webhooks**: Used primarily by Channels to receive push events from external platforms (e.g., Slack Events API, Telegram Webhooks).
*   **Event Hooks**: An internal event bus system allows modules to subscribe to lifecycle events (e.g., `session_start`, `message_received`) without tight coupling.
