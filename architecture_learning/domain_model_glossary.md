# Domain Model & Glossary (The "What")

## Glossary

*   **Agent**: An autonomous entity capable of performing tasks by interacting with an LLM, executing tools, and maintaining a conversation session.
*   **Session**: A continuous interaction thread between a user (or system) and an Agent. It stores the conversation history (context).
*   **Skill**: A bundle of capabilities (tools, environments, configurations) that an Agent can use. Skills often correspond to specific domains like "coding", "web browsing", or "data analysis".
*   **Tool**: A specific function or capability that an LLM can invoke, such as "readFile", "googleSearch", or "runCommand".
*   **Context**: The accumulated state and history of a Session, including messages, tool outputs, and system prompts, which is fed to the LLM to inform its next action.
*   **Turn**: A single cycle of User Input -> Agent Processing -> Agent Output.
*   **Provider**: An external service that supplies LLM capabilities (e.g., OpenAI, Anthropic).
*   **Gateway**: The central control plane that manages connections, routing, and the lifecycle of agents and sessions.

## Core Entities

### Agent Runtime

#### **Session**
The central unit of interaction.
*   `sessionId` (string): Unique identifier for the session.
*   `agentId` (string): The ID of the agent configuration running this session.
*   `messages` (Array<AgentMessage>): Chronological list of messages in the conversation (User, Assistant, System, Tool).
*   `status` (string): Current state of the session (e.g., "active", "idle").
*   `model` (string): The specific LLM model instance being used (e.g., "claude-3-opus").
*   `usage` (NormalizedUsage): Token usage statistics for the session.

#### **Message** (AgentMessage)
A single communication unit within a Session.
*   `role` (string): Who sent the message ("user", "assistant", "system", "tool").
*   `content` (string | Array): The actual text or multimodal content of the message.
*   `toolCalls` (Array?): If the message is a tool invocation request from the Assistant.
*   `toolCallId` (string?): If the message is a result from a tool, this links it back to the invocation.

#### **Agent Configuration** (AgentSummary/AgentsCreateParams)
Defines how an agent behaves.
*   `id` (string): Unique identifier.
*   `model` (string): Default model ID.
*   `systemPrompt` (string): The core instructions defining the agent's personality and rules.
*   `skills` (Array<string>): List of skill names enabled for this agent.
*   `params` (Object): Configuration parameters for the agent's runtime behavior.

### Skill System

#### **Skill** (SkillEntry)
A modular capability package.
*   `name` (string): Unique name of the skill.
*   `description` (string): What the skill does.
*   `tools` (Array<AnyAgentTool>): The actual functions exposed to the LLM.
*   `metadata` (OpenClawSkillMetadata): Configuration for installation, environment requirements, and display.
*   `install` (Array<SkillInstallSpec>): Instructions on how to install dependencies (e.g., via `brew`, `npm`).

#### **Tool** (AnyAgentTool)
An executable function.
*   `name` (string): The function name the LLM calls.
*   `description` (string): Instructions to the LLM on when and how to use this tool.
*   `parameters` (JSON Schema): Schema defining the arguments the tool accepts.
*   `handler` (Function): The actual code executed when the tool is called.

## Relationships

*   **One Agent has many Sessions**: An Agent configuration can spawn multiple independent conversation sessions.
*   **One Session has many Messages**: A Session is composed of a linear history of Messages.
*   **One Agent uses many Skills**: An Agent can be configured with multiple Skills to expand its capabilities.
*   **One Skill has many Tools**: A Skill is a collection of related Tools (e.g., a "Git" skill might have `git_commit`, `git_push`, `git_status`).
*   **One Session uses One Model**: While configurable, a single session typically runs against a specific Model instance at any given time (though this can be overridden).

## State Machines

### Session Lifecycle
*   **Created**: Session is initialized but has no history.
*   **Active**: Session is currently processing a turn (User input received, Agent "thinking" or executing tools).
*   **Idle**: Agent is waiting for user input.
*   **Compacting**: History is being summarized/pruned to fit within context limits.
*   **Closed/Archived**: Session is ended and persisted for history but no longer active.

### Agent Execution Loop (The "Turn")
1.  **Receive Input**: User message arrives.
2.  **Build Context**: System prompt + Session History + User Message are assembled.
3.  **Model Inference**: Context is sent to LLM.
4.  **Tool Selection**: LLM decides if a tool needs to be called.
    *   *If Tool Call*: Execute Tool -> Add Result to History -> Repeat Step 2.
    *   *If Text Response*: Stream response to User -> Go to Idle.
