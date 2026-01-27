# Feasibility Report: Springeq Lending Assistant Transformation

## Executive Summary

It is **highly feasible** to modify this project to become a specialized Lending Assistant for Springeq without changing the core codebase. The project's architecture (Clawdbot) is designed to be modular, driven by "Skills" and configurable security policies, which aligns perfectly with your requirements.

**Estimated Effort**: **Low to Medium** (Configuration & Skill Definition only).

## Detailed Feasibility Analysis

### 1. Removing Unrelated Capabilities
**Requirement**: "Reduce this project to an assistant that only takes care of [mortgage tasks]."

**Solution**:
The system loads "skills" (capabilities) from directories. You can restrict the agent to a specific set of skills via configuration, without deleting code.
- **Method**: Modify `~/.clawdbot/clawdbot.json` (or the project's default config) to disable bundled skills (like `weather`, `spotify`, etc.).
- **Configuration**:
  ```json
  {
    "skills": {
      "allowBundled": ["browser"] // Only allow specific bundled skills if needed
    }
  }
  ```
  Alternatively, you can explicitly disable specific skills:
  ```json
  {
    "skills": {
      "entries": {
        "weather": { "enabled": false },
        "spotify-player": { "enabled": false }
      }
    }
  }
  ```

### 2. User Interface (UI)
**Requirement**: "Does it have a UI? ... A terminal seems a bit hard for a loan officer."

**Solution**:
**Yes, the project includes a web-based User Interface.**
- **Existing Capabilities**: The project ships with a built-in "WebChat" and "Control UI" (found in `ui/`).
- **Features**:
  - **Conversational Stream**: It displays the conversation history, which is essential for a loan officer to see past actions ("Rate locked at 6.5%").
  - **Process Visibility**: The UI supports displaying "tool calls" (actions being processed) in real-time. When the agent interacts with the EMMA portal, the UI shows which tool is running (e.g., `browser.click`, `emma.lock_rate`) and its result.
  - **Confirmations**: The UI integrates with the messaging system. When the agent needs approval (configured via the "human-in-the-loop" settings), it sends a message asking for confirmation. The loan officer can simply type "yes" or "approve" in the chat interface.
- **Integration**: No extra coding is required to get a UI; it is part of the standard deployment.

### 3. Browser Automation (EMMA Portal)
**Requirement**: "Can this project login in in EMMA portal and click around to do stuff?"

**Solution**:
**Yes, the project has robust, built-in browser automation.**
- **Technology**: It uses Playwright (a powerful browser automation tool) under the hood (`src/browser/`).
- **Capabilities**:
  - **Login**: The agent can navigate to the EMMA portal login page, type credentials (securely injected via environment variables), and handle the login process.
  - **Interaction**: It can click buttons, fill forms, select dropdowns, and upload files (`setInputFiles`).
  - **Visuals**: It can take screenshots of the portal to confirm actions or show the status to the loan officer.
- **Implementation**: You would define these interactions as a "Skill" (e.g., `skills/emma/SKILL.md`) telling the agent: "To lock a rate, go to url X, click button Y...".

### 4. Web-Based Deployment (No Local Install)
**Requirement**: "Is it possible to have a web based assistant... [without installation for the user]?"

**Solution**:
**Yes, this project is fully deployable as a centralized web service.**
- **Server Deployment**: The project includes a `Dockerfile` and `docker-compose.yml`, making it ready for deployment on any cloud provider (AWS, GCP, Azure) or internal server.
- **Access**:
  - Users (Loan Officers) access the assistant purely via their web browser.
  - No installation is required on their local machines.
  - Remote access is supported securely via **Tailscale** (VPN) or by exposing the Web UI via a secure reverse proxy. The system supports authentication (`gateway.auth.mode`) to ensure only authorized loan officers can access it.
- **Headless Automation**: The Playwright browser automation runs entirely on the server ("headless" mode) inside the Docker container. The user interacts with the Chat UI, and the server performs the clicks/navigation on the EMMA portal in the background.

### 5. Mortgage Capabilities (API & Tools)
**Requirement**: "Generate loans, rate lock loans, price etc, consult loans, upload documents."

**Solution**:
You can add new capabilities by creating **Skills**. A Skill is simply a directory with a `SKILL.md` file that teaches the AI how to use specific CLI tools or APIs.
- **Method**: Create a `skills/springeq` directory.
- **Implementation**:
  - You will need a CLI tool or script (e.g., `emma-cli` or just `curl` commands) that interacts with the Springeq/EMMA backend.
  - Create `skills/springeq/SKILL.md` to define tools like `search_loan`, `lock_rate`, `upload_doc`.
  - **Example `SKILL.md`**:
    ```markdown
    # Springeq Lending Tools
    Use these tools to assist with loan origination in EMMA.

    ## Search Loans
    Find a loan by borrower name or loan number.
    `curl -X GET "https://api.springeq.com/loans?q={query}"`

    ## Lock Rate
    Lock the interest rate for a specific loan.
    `curl -X POST "https://api.springeq.com/loans/{id}/lock" -d '{"rate": {rate}}'`
    ```

### 6. Safety & Human-in-the-Loop
**Requirement**: "Limiting its capacity like requiring human in the loop for contacting other people or deleting files."

**Solution**:
The project has a built-in **Execution Security** system that supports exactly this.
- **Allowlist**: You can restrict the agent to *only* execute specific commands (e.g., your `emma-cli` or specific `curl` endpoints).
- **Human Approval**: You can configure the agent to **always ask for permission** before running specific commands.
- **Configuration**:
  In `clawdbot.json`:
  ```json
  {
    "tools": {
      "exec": {
        "security": "allowlist", // Only allow listed commands
        "ask": "on-miss"         // Ask if command is not in allowlist
      }
    }
  }
  ```
  For dangerous tasks (like "delete file" or "lock rate"), you can set `ask` to `"always"` for those specific patterns, or simply *not* include them in the allowlist so the system defaults to asking the user.

### 7. Hardware & System Requirements
**Requirement**: "Which hardware do we need to run it? Like ram processor OS memory?"

**Recommended Server Specs (for running the Gateway + Headless Browser):**
*   **Operating System**: Linux (Debian/Ubuntu recommended for Docker), macOS, or Windows (via WSL2).
*   **Processor (CPU)**: 2+ vCPUs (x64 or ARM64). No GPU required.
*   **Memory (RAM)**:
    *   **Minimum**: 2 GB (if only using API tools).
    *   **Recommended**: 4 GB+ (if using **Browser Automation** for EMMA). Headless Chrome/Playwright can consume significant memory depending on the complexity of the portal.
*   **Disk Space**: ~10 GB free space (for Docker images, logs, and temporary browser files).

### 8. Cloud Hosting & Cost Estimation (AWS)
**Requirement**: "Which kind of setup do we need and can you estimate costs like in AWS?"

For a production-grade deployment for your loan officers, we recommend a setup that balances performance with cost. Since the heavy AI processing happens via external APIs (Anthropic/OpenAI), the server costs are quite low.

#### Option A: Simple & Predictable (Amazon Lightsail) **[Recommended]**
Ideal for a single instance or small team.
*   **Setup**: Docker container on a Lightsail instance.
*   **Specs**: 4 GB RAM, 2 vCPUs, 80 GB SSD.
*   **Cost**: **~$20 / month** (Fixed).
*   **Pros**: Includes data transfer and storage; very easy to set up.

#### Option B: Flexible & Scalable (Amazon EC2)
Ideal if you already have an AWS VPC or need specific networking.
*   **Instance Type**: `t3.medium` (x86) or `t4g.medium` (ARM - better price/performance).
    *   2 vCPUs, 4 GB RAM.
*   **Compute Cost**: ~$25 - $30 / month (On-Demand).
*   **Storage (EBS)**: 20 GB gp3 volume (~$2 / month).
*   **Total Estimated**: **~$27 - $32 / month**.

### 9. Implementation Guide: From Local Docker to Production
**Requirement**: "Steps to follow... starting with this project... testing it locally but inside a docker... keep it working with EMMA."

#### Step 1: Local Docker Simulation
Simulate the AWS environment on your laptop.
1.  **Configure**: Create a `clawdbot.json` (or `.env` file) with your API keys (OpenAI/Anthropic).
2.  **Build & Run**:
    ```bash
    docker compose up --build
    ```
    This builds the same container image that will run in production.
3.  **Access**: Open your browser at `http://localhost:18789`. You are now interacting with the "Server" just like a loan officer would.

#### Step 2: Integrating EMMA (The "Skill")
1.  **Create Skill**: Add a file `skills/emma/SKILL.md` to your project.
2.  **Define Actions**:
    ```markdown
    # EMMA Portal Tools
    ## Login to EMMA
    Logs into the portal.
    `node scripts/emma-login.js` (or use browser tools directly)
    ```
3.  **Test**: In the Web UI (running in Docker), say "Login to EMMA". The container will launch a headless browser and attempt the action. You can view screenshots in the UI if you add a screenshot tool.

#### Step 3: Deploy to AWS
1.  **Launch Instance**: Start your Lightsail or EC2 instance.
2.  **Copy Files**: Upload your project (with the new `skills/emma` folder) to the server.
3.  **Run**: Execute `docker compose up -d` on the server.
4.  **Secure Access**: Set up Tailscale or an SSH tunnel so only your team can access the Web UI.

### 10. Model Selection & Cost/Risk Analysis
**Requirement**: "Using 'gpt-4o-mini' is wise? We want reasoning power... lending needs to be very precise."

You are correct that lending operations (Rate Locking, Pricing) require high precision. Relying solely on the cheapest model ("mini") introduces risk of hallucinations or logic errors. However, using the most expensive model for *everything* (e.g., navigating menus) is wasteful.

#### Model Comparison: Precision vs. Cost
Based on current data (and preview/leak specs for upcoming models):

| Model | Reasoning / Instruction Following | Speed | Est. Price (Input/Output per 1M) | Best Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **GPT-4o** (Standard) | **High (5/5)** | Fast | ~$2.50 / $10.00 | Complex decisions, Final verification. |
| **GPT-4o-mini** | Medium (3/5) | **Very Fast** | ~$0.15 / $0.60 | Routine tasks, Data extraction, UI navigation. |
| **GPT-5 mini** (Preview) | **High-Medium (3.5/5)** | **Lightning** | ~$0.25 / $2.00 | A balance of speed and better reasoning than 4o-mini. |
| **Gemini 2.0 Flash** | Medium-High (4/5) | **Extreme** | ~$0.10 / $0.40 | Large context analysis (documents), fast actions. |
| **Gemini 3 Flash** (Preview) | **High (Target)** | **Extreme** | (Unknown, likely ~$0.15) | Next-gen efficiency with improved reasoning. |

#### Recommended Strategy: The "Hybrid" Approach
Do not rely on a single model. Use the right tool for the job to balance cost and safety.

1.  **Tier 1: The "Doer" (Low Cost / High Speed)**
    *   **Model**: **GPT-5 mini** (if avail) or **Gemini 2.0 Flash**.
    *   **Role**: Navigating the EMMA UI, clicking buttons, filling simple forms, searching for loans.
    *   **Why**: These tasks are "mechanical" and don't require deep reasoning. Spending $10/million tokens on this is a waste.

2.  **Tier 2: The "Thinker" (High Precision / Safety)**
    *   **Model**: **GPT-4o** or **o1** (Reasoning models).
    *   **Role**:
        *   **Validating Data**: "Check this loan application against the uploaded PDF. Are the incomes identical?"
        *   **Critical Actions**: "Verify the rate lock terms (6.5%, 30yr) match the user's request exactly before clicking Confirm."
    *   **Why**: The cost of a mistake in lending (locking the wrong rate) is huge. The extra few cents for a "Reasoning" model here is insurance.

3.  **Human-in-the-Loop (Ultimate Safety)**
    *   Even with the best model, for irreversible actions (Delete, Lock Rate), the system should **pause and ask the human**:
    *   *Agent*: "I am about to LOCK the rate at 6.5%. Please confirm."
    *   *Human*: "Approve."

**Conclusion**: Start with **GPT-5 mini** or **Gemini 2.0 Flash** for general responsiveness. If you find it makes logic errors, switch *specific complex prompts* to use a larger model, or enforce human confirmation for all sensitive steps.
