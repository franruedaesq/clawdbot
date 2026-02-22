# Infrastructure & Deployment (The "Where")

Code is useless if it's not running somewhere. This file maps out the physical (or virtual) hardware where OpenClaw lives.

## Hosting: The "Local-First" Model

OpenClaw is designed to be **self-hosted**, meaning you own the infrastructure. Unlike SaaS products that run in a vendor's massive cloud, OpenClaw runs on your own machines.

### Deployment Targets
1.  **macOS (Primary Target)**:
    *   **Context**: The developer's laptop or a dedicated Mac Mini server.
    *   **Why**: Best support for local tool execution (AppleScript, native notifications, local browser automation) and Voice Wake features.
    *   **Setup**: Runs natively via `npm install -g openclaw` or from source.
2.  **Docker / Linux VPS**:
    *   **Context**: A cloud server (e.g., AWS Lightsail, DigitalOcean, Hetzner).
    *   **Why**: Always-on availability for handling Telegram/WhatsApp messages without needing a laptop open.
    *   **Setup**: Deployed via `docker-compose.yml`, which spins up the Gateway and CLI in isolated containers.

## Compute & Scale: Multi-Agent Concurrency

OpenClaw is a **Single-Process, Event-Driven Application**. It does not use Kubernetes or serverless functions like AWS Lambda.

*   **Runtime**: Node.js (v22+).
*   **Concurrency Model**:
    *   It relies on the **Node.js Event Loop** to handle concurrency.
    *   **Multi-Agent Support**: Even though it's a single process, OpenClaw can manage multiple active sessions simultaneously. For example, it can process a Telegram message for "Agent A" while simultaneously running a browser automation task for "Agent B".
    *   **Blocking Operations**: Heavy computations (like local vector search or image processing) are offloaded to asynchronous workers or external APIs to keep the gateway responsive.
*   **Scaling Strategy**:
    *   **Vertical Scaling**: Since it's a single process, the best way to handle more load is to give the machine more RAM and CPU (e.g., upgrading from a t3.micro to a t3.medium).
    *   **No Horizontal Scaling**: You generally do not run multiple instances of OpenClaw behind a load balancer. It is a stateful, personal assistant, not a stateless web app.

## Data Stores: Local & Simple

OpenClaw avoids complex external dependencies like Redis or PostgreSQL to keep deployment easy and portable.

1.  **Primary Database (JSONL)**:
    *   **What**: Session history is stored in newline-delimited JSON files (`.jsonl`) on the local disk.
    *   **Why**: Zero-config, human-readable, easy to backup/restore, and corruption-resistant (append-only).
2.  **Vector Store (SQLite)**:
    *   **What**: Semantic memory (embeddings) is stored in a local SQLite database using the `sqlite-vec` extension.
    *   **Why**: Provides powerful similarity search without needing a separate vector database server (like Pinecone or Weaviate).
3.  **Configuration**:
    *   **What**: `openclaw.json` (and `credentials.json`) are simple text files.
    *   **Why**: Easy to edit by hand or manage via dotfiles repositories.

## CI/CD Pipeline

*   **Build**: The project uses **pnpm** and **TypeScript**. The build process compiles TS to JS and bundles the UI assets using **Vite**.
*   **Testing**: GitHub Actions run the test suite (Vitest) on every push to ensure stability.
*   **Release**:
    *   **NPM**: Packages are published to the npm registry for easy installation (`npm install -g openclaw`).
    *   **Docker**: Images are built and pushed to a container registry (likely Docker Hub or GHCR), allowing users to pull the latest version with `docker pull`.
