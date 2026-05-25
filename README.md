# erpnext.claude
.claude for ERPNext

# Quick Setup

**Step 1** Clone the repo in respective folder for example

Docker Devcontianer : `workspace/development`
Normal Development : In side the `frappe-bench`

In genral keep the .claude in the working directory for best practice

```sh
git clone https://github.com/Antony-M1/erpnext.claude.git
```

**Step 2** Rename the folder
If needed do manaual rename also
```sh
mv erpnext.claude .claude
```

GET START WITH CLAUDE
HAPPY CODING...

**Step 3**


# Claude Code — `.claude` Directory Guide for ERPNext

> The `.claude` directory is to Claude Code what `.github` is to GitHub Copilot — your project's AI configuration hub. Commit it to git and your whole team inherits the same setup.

---

## Directory Structure

```
your-erpnext-app/
├── CLAUDE.md                        ← Project memory (auto-loaded every session)
├── .mcp.json                        ← MCP server definitions
└── .claude/
    ├── settings.json                ← Permissions, hooks config, model
    ├── commands/                    ← Slash commands  e.g. /fix-migration
    │   ├── bench-restart.md
    │   ├── doctype-review.md
    │   └── fix-migration.md
    ├── hooks/                       ← Auto-run scripts on tool events
    │   ├── pre-write.sh
    │   └── post-write.sh
    ├── skills/                      ← Auto-triggered expertise packs
    │   └── frappe-patterns/
    │       └── SKILL.md
    └── agents/                      ← Specialized subagents
        └── migration-agent.md
```

---

## Features

| Feature | File/Location | What it does |
|---|---|---|
| **CLAUDE.md** | `./CLAUDE.md` | Project memory — stack, commands, conventions |
| **settings.json** | `.claude/settings.json` | Permissions, model, hooks wiring |
| **Slash Commands** | `.claude/commands/*.md` | Reusable prompt templates via `/command-name` |
| **Hooks** | `.claude/hooks/*.sh` + `settings.json` | Scripts triggered by Claude's tool events |
| **Skills** | `.claude/skills/<name>/SKILL.md` | Auto-triggered domain expertise packs |
| **Subagents** | `.claude/agents/*.md` | Specialist Claude sessions with isolated context |
| **MCP Servers** | `.mcp.json` | Live tool integrations (DB, GitHub, filesystem) |

---

## 1. CLAUDE.md — Project Memory

Auto-loaded every session. Tell Claude your stack, commands, and rules once — it remembers forever.

```markdown
# ERPNext Custom App — CLAUDE.md

## Stack
- Frappe Framework v15 + ERPNext v15
- Python 3.11, MariaDB 10.6, Redis
- Vue 3 (desk UI), Jinja2 (print formats)
- Bench CLI for all operations

## Commands
- `bench start`          — Start dev server (port 8000)
- `bench migrate`        — Run DB migrations after DocType changes
- `bench build`          — Rebuild JS/CSS assets
- `bench run-tests --app procurement_portal` — Run tests
- `bench reinstall`      — Full reset (ONLY on dev, ask first)

## Conventions
- DocType names: Title Case singular ("Purchase Request", not "purchase_requests")
- Controller files: snake_case matching DocType name
- Custom fields: prefix with app name e.g. `proc_approval_status`
- Never modify core ERPNext files — use hooks.py overrides
- All monetary fields: currency type, link to Company currency

## Architecture
- Custom app: apps/procurement_portal
- Overrides go in: procurement_portal/overrides/
- Client scripts: public/js/ (auto-bundled by bench build)
- Print formats: templates/print_formats/

## Rules
- Ask before running bench reinstall
- Always run bench migrate after DocType schema changes
- Validate with frappe.throw(), not Python raise
- Use frappe.db.get_value() not raw SQL selects
```

> **Tip:** You can also place `CLAUDE.md` inside subdirectories (e.g. `apps/procurement_portal/CLAUDE.md`) for app-specific context in a monorepo bench setup.

---

## 2. settings.json — Permissions & Hooks Wiring

Controls what Claude is allowed to do, which model to use, and registers your hooks.

```json
{
  "model": "claude-sonnet-4-20250514",

  "permissions": {
    "allow": [
      "Bash(bench migrate)",
      "Bash(bench build*)",
      "Bash(bench run-tests*)",
      "Bash(bench restart)",
      "Read(apps/procurement_portal/**)",
      "Write(apps/procurement_portal/**)"
    ],
    "deny": [
      "Bash(bench reinstall*)",
      "Write(apps/erpnext/**)",
      "Write(apps/frappe/**)"
    ]
  },

  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [{
          "type": "command",
          "command": ".claude/hooks/pre-write.sh"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{
          "type": "command",
          "command": ".claude/hooks/post-write.sh"
        }]
      }
    ]
  }
}
```

> **Note:** Project `settings.json` overrides global `~/.claude/settings.json`. CLI flags like `--permission-mode` override both for that session only.

---

## 3. Slash Commands — Reusable Prompt Templates

Store prompts as `.md` files in `.claude/commands/`. Filename becomes the command. Supports `$ARGUMENTS` for dynamic inputs.

### `.claude/commands/doctype-review.md` → `/doctype-review`

```markdown
---
name: doctype-review
description: Review a DocType controller for ERPNext conventions
---

Review the DocType controller at $ARGUMENTS for:

1. **Naming** — Does the class name match the DocType name exactly?
2. **Hooks** — Are validate(), before_save(), on_submit() used correctly?
3. **Permissions** — Uses frappe.has_permission(), not hardcoded roles
4. **DB calls** — No raw SQL; uses frappe.db.get_value/get_list
5. **Currency fields** — Linked to Company currency, not hardcoded INR/USD
6. **Error handling** — frappe.throw() with user-friendly messages
7. **Tests** — Corresponding test file exists in tests/

Provide a summary table and specific fixes.
```

**Usage:**
```
/doctype-review apps/procurement_portal/procurement_portal/doctype/purchase_request/purchase_request.py
```

### `.claude/commands/fix-migration.md` → `/fix-migration`

```markdown
---
name: fix-migration
description: Diagnose and fix a failing bench migrate
---

A `bench migrate` is failing. Diagnose the issue:

1. Run `bench migrate` and capture the traceback
2. Check if it's a MariaDB column type conflict
3. Check for duplicate custom field names
4. Look for missing link field targets
5. Propose and apply the minimal fix
6. Re-run `bench migrate` to confirm resolution

Do NOT run bench reinstall unless I explicitly approve it.
```

### `.claude/commands/bench-restart.md` → `/bench-restart`

```markdown
---
name: bench-restart
description: Safely restart all bench services
---

Restart bench services in this order:
1. `bench restart` — restart workers and scheduler
2. Wait 5 seconds
3. `bench build` — rebuild assets if $ARGUMENTS includes "build"
4. Confirm all services are back up
```

---

## 4. Hooks — Auto-Run Scripts on Tool Events

Scripts triggered by Claude's tool usage. Registered in `settings.json`.

| Hook type | When it fires | Exit code 2 effect |
|---|---|---|
| `PreToolUse` | Before every tool call | Denies the operation |
| `PostToolUse` | After every tool call | N/A (runs after) |
| `SessionStart` | When a session begins | — |
| `SessionEnd` | When a session ends | — |

### `.claude/hooks/pre-write.sh` — Block writes to core ERPNext

```bash
#!/bin/bash
# Exit 2 = DENY the write. Exit 0 = allow.

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('path',''))")

# Block writes to core frappe/erpnext
if echo "$FILE_PATH" | grep -qE "^apps/(frappe|erpnext)/"; then
  echo "Blocked: Do not edit core ERPNext files."
  echo "Use hooks.py or custom app overrides instead."
  exit 2
fi

# Warn if editing a migration file
if echo "$FILE_PATH" | grep -q "patches/"; then
  echo "Warning: Editing a migration patch file."
fi

exit 0
```

### `.claude/hooks/post-write.sh` — Auto-migrate after DocType JSON changes

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('path',''))")

# Auto-migrate when a DocType JSON is saved
if echo "$FILE_PATH" | grep -qE "doctype/.*\.json$"; then
  echo "DocType JSON changed — running bench migrate..."
  cd /home/frappe/frappe-bench && bench migrate --skip-failing 2>&1
fi
```

> **Make scripts executable:** `chmod +x .claude/hooks/*.sh`

---

## 5. Skills — Auto-Triggered Expertise Packs

Skills live in `.claude/skills/<name>/SKILL.md` and auto-activate when Claude detects relevant context — no manual invocation needed.

### `.claude/skills/frappe-patterns/SKILL.md`

```markdown
---
name: frappe-patterns
description: >
  Activate when working on any Frappe/ERPNext DocType,
  controller, client script, or hook. Enforces ERPNext
  coding patterns and Frappe API best practices.
tools:
  - Read
  - Bash
---

# Frappe Patterns Skill

You are an ERPNext/Frappe expert. Apply these patterns:

## DocType Controllers

```python
class PurchaseRequest(Document):
    def validate(self):
        self.validate_dates()
        self.calculate_totals()

    def validate_dates(self):
        if self.required_date < self.transaction_date:
            frappe.throw(_("Required Date cannot be before Transaction Date"))
```

## Frappe DB — preferred patterns

```python
# Use frappe.db.get_value
vendor = frappe.db.get_value("Supplier", filters, ["name", "supplier_name"])

# Use frappe.get_list for multiple records
items = frappe.get_list("Item", filters={"item_group": "Raw Material"},
                         fields=["name", "stock_uom"])

# Never raw SQL for simple reads
```

## Client Scripts

```javascript
frappe.ui.form.on('Purchase Request', {
    refresh(frm) {
        frm.add_custom_button(__('Create PO'), () => {
            frappe.call({ method: '...create_purchase_order', args: { doc: frm.doc } });
        });
    }
});
```
```

> **Tip:** Set `disable-model-invocation: true` in the frontmatter for manual-only invocation.

---

## 6. Subagents — Specialist Agents with Isolated Context

Separate Claude sessions launched by the main agent. Each has its own context window. Ideal for parallel work or keeping the main session clean.

### `.claude/agents/migration-agent.md`

```markdown
---
name: erpnext-migration-agent
description: >
  Specialist for writing and validating ERPNext migration patches.
  Invoke when creating a new patch, fixing a broken patch, or
  reviewing patch ordering in patches.txt.
model: claude-haiku-4-5-20251001
tools:
  - Read
  - Write
  - Bash
---

# ERPNext Migration Agent

You write and validate Frappe migration patches. Rules:

1. **Patch file location:** `apps/{app}/patches/{version}/{patch_name}.py`
2. **Patch function:** always named `execute()`
3. **Idempotent:** patches must be safe to run multiple times
4. **Register:** add to `patches.txt` with correct ordering

## Patch template

```python
import frappe

def execute():
    """
    Migration: Add approval_status field to Purchase Request
    Ticket: PROC-142
    """
    if not frappe.db.has_column("Purchase Request", "approval_status"):
        frappe.db.add_column("Purchase Request", "approval_status",
                             "varchar(140) DEFAULT 'Pending'")

    frappe.db.sql("""
        UPDATE `tabPurchase Request`
        SET approval_status = 'Approved'
        WHERE docstatus = 1 AND approval_status IS NULL
    """)
    frappe.db.commit()
```

After writing the patch, run `bench migrate` and confirm it exits 0.
```

> **Cost tip:** Use Haiku for subagents to save tokens. The main Claude (Sonnet/Opus) orchestrates; subagents do the heavy lifting in isolation.

---

## 7. MCP Servers — Live Tool Integrations

Defined in `.mcp.json` at the repo root. Committed to git so the whole team gets the same connections.

```json
{
  "mcpServers": {
    "mariadb": {
      "type": "stdio",
      "command": "mcp-server-mysql",
      "env": {
        "MYSQL_HOST": "localhost",
        "MYSQL_DATABASE": "_your_site_db",
        "MYSQL_USER": "frappe"
      }
    },

    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },

    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem",
        "/home/frappe/frappe-bench/apps"
      ]
    }
  }
}
```

> **With the MariaDB MCP connected**, you can ask: *"Show me all Purchase Requests submitted this month where approval_status is Pending"* — Claude queries live data directly.

---

## Scope: Project vs Global

| Scope | Location | Shared with team? |
|---|---|---|
| **Project** | `.claude/` and `CLAUDE.md` in your repo | Yes — commit to git |
| **Global** | `~/.claude/` on your machine | No — personal only |

Project settings override global settings. CLI flags override both for that session.

---

## Quick Reference

| You want to… | Use |
|---|---|
| Give Claude permanent project context | `CLAUDE.md` |
| Block Claude from touching core ERPNext | `settings.json` → `permissions.deny` |
| Run the same prompt repeatedly | `.claude/commands/your-command.md` |
| Auto-format/validate after every file write | `.claude/hooks/post-write.sh` |
| Enforce Frappe coding patterns automatically | `.claude/skills/frappe-patterns/SKILL.md` |
| Offload patch writing to a specialist agent | `.claude/agents/migration-agent.md` |
| Let Claude query live MariaDB data | `.mcp.json` → mariadb server |

