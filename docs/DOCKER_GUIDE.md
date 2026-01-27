# Docker Deployment Guide for Springeq Lending Assistant

This guide explains how to run Clawdbot in Docker, specifically configured for the **Springeq Lending Assistant** use case. This setup includes:
1.  **Browser Automation Support**: For accessing the EMMA portal.
2.  **Skill Mounting**: Injecting your custom Springeq tools.
3.  **Web UI Access**: Using the assistant via a browser.
4.  **Security**: Best practices for production.

---

## 1. The Challenge: Browser Dependencies

The standard `Dockerfile` is optimized for size and does not include the heavy system libraries required to run a browser (Chromium/Playwright). Since your Springeq skill requires opening `https://emma.springeq.com`, we need a custom image.

### Create `Dockerfile.springeq`

Create this file in the root of your project. It extends the base image and adds the necessary browser dependencies.

```dockerfile
# Dockerfile.springeq
FROM node:22-bookworm

# 1. Install system dependencies for Playwright (Chromium)
# We use the official Playwright script to install system deps
RUN npx playwright install-deps chromium

# 2. Copy the project files (standard build)
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY patches ./patches
COPY scripts ./scripts

# 3. Install Node dependencies
RUN npm install -g pnpm && pnpm install --frozen-lockfile

# 4. Copy source and build
COPY . .
RUN pnpm build
ENV CLAWDBOT_PREFER_PNPM=1
RUN pnpm ui:install && pnpm ui:build

# 5. Install the browser binary itself
RUN npx playwright install chromium

# 6. Security: Run as non-root
USER node
ENV NODE_ENV=production

CMD ["node", "dist/index.js"]
```

---

## 2. Docker Compose Configuration

Create a `docker-compose.springeq.yml` to define the service, volumes, and secrets.

```yaml
services:
  clawdbot:
    build:
      context: .
      dockerfile: Dockerfile.springeq
    environment:
      # --- Authentication & Secrets ---
      # Tokens for AI Models (OpenAI, Anthropic, etc.)
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      OPENAI_API_KEY: ${OPENAI_API_KEY}

      # Springeq Credentials (passed to the agent)
      EMMA_USER: ${EMMA_USER}
      EMMA_PASS: ${EMMA_PASS}

      # --- Configuration ---
      CLAWDBOT_CONFIG_DIR: /home/node/.clawdbot
      HOME: /home/node

    volumes:
      # 1. Persist Configuration (Memory, Chat History)
      - ./data/config:/home/node/.clawdbot

      # 2. MOUNT YOUR SKILLS
      # This maps your local 'skills' folder to the container's workspace.
      # The agent will look for skills in /home/node/clawd/skills
      - ./skills:/home/node/clawd/skills

    ports:
      # Expose the Web UI
      - "18789:18789"

    restart: unless-stopped
```

---

## 3. How to Run It

### Step 1: Prepare Directories
Create the folders for data and your skills.

```bash
mkdir -p data/config
mkdir -p skills/springeq
```

*Tip: Follow the `docs/SKILL_DEVELOPMENT.md` guide to populate `skills/springeq` with your `SKILL.md` and scripts.*

### Step 2: Configure Secrets
Create a `.env` file with your keys. **Never commit this file.**

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-proj-xxx
EMMA_USER=loan.officer@example.com
EMMA_PASS=secretPassword123
```

### Step 3: Build and Start
Run the container using your custom Compose file.

```bash
docker compose -f docker-compose.springeq.yml up -d --build
```

### Step 4: Access the UI
Open your browser and navigate to:
**http://localhost:18789**

You will see the Clawdbot Chat Interface.

---

## 4. Verifying the Capabilities

1.  **Check the UI**: Confirm the page loads.
2.  **Check the Skill**: Type *"Help me login to Springeq"* in the chat.
    *   The agent should recognize the command from your `skills/springeq/SKILL.md`.
    *   It will execute the browser script inside the container.
    *   **Note**: Because the browser is "headless" (running inside Docker), you won't see a Chrome window pop up on your screen. However, the agent can take screenshots if you add that capability to your skill, or simply report back "Login successful".

---

## 5. Security Considerations

When running in a production or semi-production environment:

1.  **User Permissions**: The Dockerfile uses `USER node` (UID 1000). This prevents the process from having root access to the container, limiting the damage if code execution is compromised.
2.  **Network Exposure**:
    *   The configuration exposes port `18789`.
    *   **Do not** forward this port on your router or cloud firewall to the public internet without protection.
    *   **Recommended**: Use a VPN (like Tailscale) to access the UI securely from remote locations, or put it behind a reverse proxy (Nginx/Caddy) with Basic Auth if deploying to the cloud.
3.  **Secrets**:
    *   Pass credentials (like `EMMA_PASS`) strictly via environment variables (`.env`).
    *   Do not hardcode passwords in `SKILL.md` or scripts.
4.  **Skill Isolation**:
    *   The agent runs commands defined in your Skills. Ensure your `SKILL.md` only exposes necessary actions.
    *   The `exec-approvals` system (documented in Feasibility Report) applies here too. You can configure the agent to ask for permission before running `curl` or `node` commands.
