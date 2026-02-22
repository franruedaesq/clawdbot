# Data Flow & Sequence (The "Movement")

Static diagrams don't tell you how a system behaves. This file traces the exact path of data for critical features, helping you understand the step-by-step logic of the system's most important use cases.

**Concept: Tracing Execution Paths**
In software architecture, understanding data flow means knowing exactly which functions call which, where state is mutated, and how components hand off responsibility. We will look at three key flows: routing an incoming message, processing that message (the agent loop), and executing a tool.

---

## Flow 1: Multi-Agent Routing (The "Dispatcher")

**Goal**: Determine which Agent configuration (and specific Session) should handle an incoming event.

**Architecture Concept**: **Routing & Dispatching**. This is the logic that maps an external identifier (like a Telegram Chat ID) to an internal resource (an Agent Session). It allows one Gateway to host multiple distinct "personalities" or agents.

### The Happy Path

1.  **Incoming Event**: A webhook or WebSocket event arrives at a Channel Plugin (e.g., `src/telegram/bot.ts`).
2.  **Normalization**: The channel converts the platform-specific payload into a standard object.
3.  **Session Key Resolution**:
    *   The system calls `resolveSessionKey` (in `src/routing/session-key.ts`).
    *   It combines the Channel ID (e.g., "telegram") and the Account ID (e.g., specific bot token hash) and the Peer ID (user's ID).
    *   Result: A canonical string like `telegram:default:12345`.
4.  **Agent Resolution**:
    *   The system calls `resolveSessionAgentId` (in `src/agents/agent-scope.ts`).
    *   It checks `openclaw.json` configuration to see if this session key is mapped to a specific agent (e.g., "coding-assistant" vs "personal-bot").
    *   If no specific mapping exists, it defaults to the `main` agent.
5.  **Dispatch**: The message is handed off to the `Session` object associated with that key.

### Error Handling
*   **No Matching Agent**: If configuration is missing or invalid, the system typically falls back to the default agent or logs a warning.
*   **Invalid Session Key**: If a session key cannot be generated (e.g., missing metadata), the event is dropped to prevent "ghost" sessions.

---

## Flow 2: Message Processing (The "Pipeline")

**Goal**: The Agent receives a message, "thinks" about it, and replies.

**Architecture Concept**: **Event-Driven Pipeline**. The system reacts to an event (new message), processes it through a series of steps (Context Build -> Inference), and produces an output (Reply).

### The Happy Path

1.  **Ingestion**: `Gateway.handleIncomingMessage` receives the normalized message.
2.  **Session Locking**: The system acquires a lock on the session file (e.g., `sessions/telegram_default_12345.jsonl`) to ensure sequential processing.
3.  **Context Construction**:
    *   `AgentRuntime` reads the session history from the JSONL file.
    *   It appends the new User Message.
    *   It retrieves the System Prompt and available Tools for the resolved Agent.
4.  **Inference (The "Think")**:
    *   `AgentRuntime.runAgentLoop` calls the LLM Provider (e.g., `Anthropic.chat.completions.create`).
    *   The prompt (Context + Message) is sent to the model.
5.  **Response Handling**:
    *   **Text Response**: If the model returns text, `ReplyDispatcher` (in `src/auto-reply/reply/reply-dispatcher.ts`) streams it back to the channel.
    *   **Tool Call**: If the model requests a tool, we enter Flow 3 (see below).
6.  **Persistence**: The new User Message and Assistant Response are appended to the JSONL file.

### Error Handling
*   **Provider Failure**: If OpenAI/Anthropic is down, the `AgentRuntime` catches the exception. It may retry (exponential backoff) or send a "I'm having trouble connecting" message to the user.
*   **Context Limit Exceeded**: If the history is too long, a "Compaction" process is triggered to summarize older messages before the new inference step.

---

## Flow 3: Tool Execution (The "Side Effect")

**Goal**: The Agent executes a capability (like checking the weather or running code) and uses the result.

**Architecture Concept**: **Remote Procedure Call (RPC) / Command Pattern**. The LLM issues a command (Tool Call), the system executes it (Side Effect), and returns the result (Observation).

### The Happy Path

1.  **Tool Detection**: The LLM response contains a `tool_calls` array instead of (or alongside) text.
2.  **Tool Lookup**:
    *   `AgentRuntime` looks up the tool name in the Agent's enabled Skill set.
    *   It validates the arguments against the tool's JSON Schema.
3.  **Execution**:
    *   The specific tool handler is invoked (e.g., `BrowserTool.navigate`).
    *   If the tool is "sandboxed" (e.g., running in Docker), the execution is routed via RPC or a specific runner (e.g., `src/agents/bash-process-registry.ts`).
4.  **Result Capture**: The tool returns a value (string, JSON, or error).
5.  **Context Update**:
    *   A new message with role `tool` is added to the session history, containing the `tool_call_id` and the result.
6.  **Re-Inference**: The `runAgentLoop` triggers *another* call to the LLM, now with the tool result in the context, so the Agent can interpret the data and formulate a final answer for the user.

### Error Handling
*   **Tool Crash**: If the tool code throws an error, the system catches it and formats it as a standard "Tool Error" message in the conversation context. This allows the LLM to see that the tool failed and potentially try a different approach (e.g., "The file didn't exist, I'll try creating it first").
