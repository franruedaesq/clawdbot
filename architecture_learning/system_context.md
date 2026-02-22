# System Context (The "Why" and "Who")

## Core Purpose
OpenClaw is a personal, self-hosted AI assistant designed to run locally on user devices. It acts as a unified control plane that connects various messaging platforms (like WhatsApp, Telegram, Slack) with powerful AI models (OpenAI, Anthropic) to automate tasks, manage context, and provide a seamless assistant experience across desktop and mobile environments, all while prioritizing user privacy and control.

## User Personas
*   **Owner/Admin**: The primary user who installs, configures, and operates the OpenClaw instance. They manage API keys, select AI models, configure channels, and have full administrative control over the system's behavior.
*   **User**: Individuals who interact with the assistant through configured messaging channels. This can include family members or team members if the Owner explicitly grants access.
*   **Developer**: Technical users who extend the system's capabilities by creating custom skills, plugins, or integrating new services.

## External Dependencies
*   **LLM Providers**:
    *   OpenAI (ChatGPT, Codex)
    *   Anthropic (Claude Pro/Max)
*   **Speech & Audio Services**:
    *   ElevenLabs (Text-to-Speech for Voice Wake and Talk Mode)
*   **Messaging Platforms**:
    *   WhatsApp (via Baileys)
    *   Telegram (via grammY)
    *   Slack (via Bolt)
    *   Discord (via discord.js)
    *   Google Chat (Chat API)
    *   Signal (via signal-cli)
    *   BlueBubbles (iMessage integration)
    *   Microsoft Teams (Extension)
    *   Matrix (Extension)
    *   Zalo & Zalo Personal (Extensions)
*   **Networking & Access**:
    *   Tailscale (Serve/Funnel for secure remote access)
    *   SSH (Secure Shell for remote management)
*   **Browser Automation**:
    *   Chromium (via Puppeteer/Playwright for web-based tasks)
*   **Skill Registry**:
    *   ClawHub (for discovering and installing community skills)
*   **System Integrations**:
    *   Local OS Notification Systems (macOS, iOS, Android)

## Triggers
*   **Incoming Messages**: The system wakes up when it receives messages via webhooks or polling from connected platforms (e.g., a new Telegram message or Slack mention).
*   **Voice Activation**: "Voice Wake" hotword detection on companion apps (macOS, iOS, Android) triggers the assistant to listen.
*   **Scheduled Jobs**: Internal Cron jobs trigger the system to perform periodic tasks or checks.
*   **Manual CLI Commands**: The user can directly trigger actions or start sessions using the `openclaw` command-line interface.
*   **Email Hooks**: Incoming emails via Gmail Pub/Sub integration can trigger specific workflows.
*   **System Events**: Startup sequences, error conditions, or internal state changes that require the assistant's attention.
