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

### 4. Mortgage Capabilities (API & Tools)
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

### 5. Safety & Human-in-the-Loop
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

## What the Assistant Would Be Able To Do

Once configured, the assistant will be able to:
1.  **Conversational Interface**: Chat with you via the included Web UI.
2.  **Execute Mortgage Tasks**:
    *   "Check the status of the loan for John Doe" -> Agent logs into EMMA (via browser) or calls API -> Displays status.
    *   "Lock the rate for Loan #12345 at 6.5%" -> Agent asks "Are you sure you want to lock Loan #12345 at 6.5%?" -> User approves -> Agent performs the click/API call.
3.  **Document Handling**:
    *   "Upload this PDF to the loan file" -> Agent takes the file and uploads it via the browser/API.
4.  **Safety**:
    *   It will **refuse** to run unrelated commands (like `rm -rf` or checking the weather) if they are not allowed.
    *   It will **pause** and request your approval for sensitive actions defined in your policy.

## Conclusion

This project is an excellent foundation for a specialized Lending Assistant. It requires **no core code changes**—only configuration and the definition of your domain-specific "Skills" (API or Browser automation). The UI and security features are already built-in and well-suited for a loan officer's workflow.
