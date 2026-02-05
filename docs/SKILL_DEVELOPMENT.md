# Skill Development Guide

Skills are the modular "plugins" that extend the capabilities of the Clawdbot agent. This guide explains how they work and how to create new ones, specifically tailored for custom use cases like the Springeq Lending Assistant.

## How Skills Work

A **Skill** is simply a directory containing a `SKILL.md` file (and optional scripts/assets). The agent reads this file to understand:
1.  **When** to use the skill (via `description` in Frontmatter).
2.  **How** to use the tools (via the Markdown body and code blocks).

When the agent loads, it scans the `skills/` directory. If a user's request matches a skill's description, the agent loads that skill's instructions into its context.

## Directory Structure

To add a new skill, create a folder in `skills/`.

```text
skills/
└── springeq/              <-- The Skill Directory
    ├── SKILL.md           <-- The Definition File (Required)
    ├── scripts/           <-- Helper scripts (Optional)
    │   └── login.js
    └── assets/            <-- Templates/Files (Optional)
```

## The `SKILL.md` Format

The file has two parts: **Frontmatter** (YAML) and **Body** (Markdown).

### 1. Frontmatter (YAML)
Located at the very top of the file between `---`.

```yaml
---
name: springeq
description: Manage Springeq mortgage loans (search, price, rate lock) and interact with the EMMA portal.
metadata: {"clawdbot":{"emoji":"🏠","requires":{"bins":["node","curl"]}}}
---
```

*   **name**: Unique identifier (e.g., `springeq`).
*   **description**: **Critical**. The AI reads this to decide if it should activate this skill. Be descriptive!
*   **metadata**: JSON5 configuration.
    *   `emoji`: Icon for the UI.
    *   `requires`: List of system tools needed (e.g., `curl`, `node`, `python`).

### 2. Body (Markdown)
This is the "User Manual" for the AI. It uses standard Markdown headers and code blocks to define actions.

**Principles:**
*   **Context**: Explain *what* the tools are for.
*   **Examples**: Show the exact command the AI should run.
*   **Variables**: Use `{variable}` syntax if needed, though the AI generally infers arguments.

## Example: Springeq Lending Assistant

Here is a complete example of how you would define the **Springeq** skill.

**File:** `skills/springeq/SKILL.md`

```markdown
---
name: springeq
description: Interact with Springeq/EMMA to manage loans, lock rates, and upload documents.
metadata: {"clawdbot":{"emoji":"🏠","requires":{"bins":["curl","node"]}}}
---

# Springeq Lending Assistant

Use these tools to perform actions in the EMMA portal.

## 1. Authentication

Before performing actions, ensure you are logged in.

```bash
# Check if logged in
node skills/springeq/scripts/auth_check.js

# Login (if check fails)
node skills/springeq/scripts/login.js
```

## 2. Loan Search

Find a loan by Borrower Name or Loan Number.

```bash
# Search by Loan Number
curl -s "https://api.springeq.com/v1/loans?loanNumber={loan_number}" \
  -H "Authorization: Bearer $EMMA_TOKEN"

# Search by Borrower Name
curl -s "https://api.springeq.com/v1/loans?borrower={name}" \
  -H "Authorization: Bearer $EMMA_TOKEN"
```

## 3. Rate Locking (⚠️ Sensitive)

**Rules:**
1.  **Verify**: Always verify the current rate with the user before locking.
2.  **Confirmation**: You MUST ask for explicit confirmation before executing the lock command.

```bash
# Get Current Pricing
curl -s "https://api.springeq.com/v1/loans/{id}/pricing" \
  -H "Authorization: Bearer $EMMA_TOKEN"

# Lock Rate (REQUIRES CONFIRMATION)
curl -X POST "https://api.springeq.com/v1/loans/{id}/lock" \
  -H "Authorization: Bearer $EMMA_TOKEN" \
  -d '{"rate": {rate}, "program": "{program}"}'
```

## 4. Browser Automation (Fallback)

If the API is unavailable, use the browser tool to navigate the portal manually.

```bash
# Open Loan in Browser (Headless)
node skills/springeq/scripts/browser_nav.js --url "https://emma.springeq.com/loan/{id}"
```
```

## Scripting (Optional)

If a task is too complex for a single `curl` command (e.g., a complex login flow), put the logic in a script in the `scripts/` folder.

**File:** `skills/springeq/scripts/login.js`

```javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('https://emma.springeq.com/login');
  await page.fill('#username', process.env.EMMA_USER);
  await page.fill('#password', process.env.EMMA_PASS);
  await page.click('#login-btn');
  // ... extract token ...
  console.log("Logged in successfully.");
  await browser.close();
})();
```

## Summary of Steps to Add Your Skill

1.  **Create Directory**: `mkdir -p skills/springeq/scripts`
2.  **Create Definition**: Write the `skills/springeq/SKILL.md` file (using the format above).
3.  **Add Scripts**: Write any Node.js or Bash scripts needed in `skills/springeq/scripts/`.
4.  **Restart**: Restart Clawdbot so it picks up the new skill.
5.  **Test**: Ask the bot: *"Help me find the loan for John Doe in Springeq."*
