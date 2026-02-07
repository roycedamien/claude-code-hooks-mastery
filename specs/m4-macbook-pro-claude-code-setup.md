# Plan: M4 MacBook Pro — Claude Code Hooks Developer Environment

## Task Description
Set up a brand-new M4 MacBook Pro as a complete Claude Code development environment, leveraging the IndyDevDan (disler) hooks ecosystem. The machine currently has only Claude Desktop and Chrome installed. This spec covers every step from installing Homebrew through a fully operational global hooks system, per-project bootstrapping via the install-and-maintain pattern, and optional enhancements like TTS, observability, and MCP servers.

## Objective
A fully functional M4 MacBook Pro where:
1. Claude Code CLI is installed with a global hooks layer (security, logging, session management)
2. Any new project can be bootstrapped with `claude --init` using the install-and-maintain pattern
3. The developer has a curated status line, safety guardrails, and optional TTS feedback
4. The setup is reproducible — another engineer could follow this spec and get the same result

## Problem Statement
A fresh Mac has none of the tooling Claude Code hooks depend on — no package manager, no Python, no Node.js, no `uv`, no `gh` CLI. The hooks in this repo use `$CLAUDE_PROJECT_DIR` (project-scoped), but a developer also needs a **global** baseline that protects every project (security hooks, logging, permission auditing). Without a spec, piecing together which hooks go global vs. per-project, which repos to pull from, and what order to install things is a trial-and-error process.

## Solution Approach
A three-layer architecture inspired by the IndyDevDan repos:

```
┌─────────────────────────────────────────────────┐
│  Layer 3: Per-Project Hooks (.claude/)          │
│  install-and-maintain pattern, project-specific │
│  agents, validators, custom commands            │
├─────────────────────────────────────────────────┤
│  Layer 2: Global Hooks (~/.claude/)             │
│  Security (PreToolUse), logging, session mgmt,  │
│  status line, permission auditing               │
├─────────────────────────────────────────────────┤
│  Layer 1: Foundation Tools                      │
│  Homebrew, uv, Bun, Node, git, gh, just,       │
│  Claude Code CLI                                │
└─────────────────────────────────────────────────┘
```

**How layers interact**: Claude Code merges settings from all levels. Global hooks run on every project. Per-project hooks add project-specific behavior on top. If both levels define the same hook event, **both** execute (global first, then project).

## What Is a Spec?

> If you've never worked with a spec before, here's the idea: a spec is a **blueprint** that turns a vague goal ("set up my laptop") into a concrete, ordered checklist. It answers three questions:
>
> 1. **What** are we building? (Objective + Acceptance Criteria)
> 2. **How** do we build it? (Step-by-step tasks with exact commands)
> 3. **How do we know it worked?** (Validation commands)
>
> You can hand this document to Claude Code with `/build` and it will execute it, or you can follow it manually. Either way, nothing is ambiguous.

## Relevant Files

### From This Repo (claude-code-hooks-mastery)
These are the source files we'll cherry-pick from for the global hooks layer:

- `.claude/hooks/pre_tool_use.py` — Security: blocks dangerous `rm` commands and `.env` file access
- `.claude/hooks/session_start.py` — Loads dev context (git branch, issues, TODO files) at session start
- `.claude/hooks/session_end.py` — Session cleanup and logging
- `.claude/hooks/stop.py` — Completion messages with optional TTS
- `.claude/hooks/notification.py` — Notification logging with optional TTS
- `.claude/hooks/permission_request.py` — Permission auditing, auto-allow read-only ops
- `.claude/hooks/pre_compact.py` — Transcript backup before context compaction
- `.claude/hooks/post_tool_use.py` — Tool execution logging and validation
- `.claude/hooks/post_tool_use_failure.py` — Error logging with structured context
- `.claude/hooks/subagent_start.py` — Subagent spawn logging
- `.claude/hooks/subagent_stop.py` — Subagent completion logging
- `.claude/hooks/setup.py` — Repository initialization and maintenance
- `.claude/hooks/user_prompt_submit.py` — Prompt validation, session data, agent naming
- `.claude/hooks/utils/` — LLM and TTS utility libraries
- `.claude/hooks/validators/` — Ruff and ty code quality validators
- `.claude/status_lines/status_line_v6.py` — Context window usage bar (recommended default)
- `.claude/settings.json` — Full hook wiring reference
- `.env.sample` — Environment variable template
- `.mcp.json.sample` — MCP server configuration template

### From disler/install-and-maintain
- `.claude/hooks/setup_init.py` — Deterministic project bootstrapping (deps, DB, env)
- `.claude/hooks/setup_maintenance.py` — Dependency updates, DB health checks
- `.claude/hooks/session_start.py` — Loads .env into CLAUDE_ENV_FILE
- `.claude/commands/install.md` — Agentic install slash command
- `.claude/commands/install-hil.md` — Human-in-the-loop interactive setup
- `.claude/commands/maintenance.md` — Maintenance slash command
- `justfile` — Entry point runner

### New Files to Create
- `~/.claude/settings.json` — Global hooks configuration
- `~/.claude/hooks/` — Global hook scripts directory
- `~/.claude/hooks/pre_tool_use.py` — Global security hook (adapted)
- `~/.claude/hooks/session_start.py` — Global session start (adapted)
- `~/.claude/hooks/permission_request.py` — Global permission auditing (adapted)
- `~/.claude/hooks/stop.py` — Global completion hook (adapted)
- `~/.claude/hooks/post_tool_use_failure.py` — Global error logging (adapted)
- `~/.claude/hooks/utils/` — Shared utility libraries
- `~/.claude/status_lines/status_line.py` — Global status line
- `~/.env` — Global API keys (gitignored, never committed)

## Implementation Phases

### Phase 1: Foundation (Layer 1)
Install all prerequisite tools on the fresh M4 Mac. This is pure terminal work — no hooks yet.

### Phase 2: Claude Code + Global Hooks (Layer 2)
Install Claude Code CLI, create `~/.claude/` directory structure, adapt the best generic hooks from this repo for global use, wire them in `~/.claude/settings.json`.

### Phase 3: Per-Project Pattern (Layer 3)
Set up the install-and-maintain bootstrapping pattern so any new project gets a `.claude/` directory with project-specific hooks, commands, and the three-tier setup flow.

### Phase 4: Optional Enhancements
TTS feedback, MCP servers, Ollama for local LLM, observability layer.

## Step by Step Tasks

IMPORTANT: Execute every step in order, top to bottom.

---

### 1. Install Homebrew
**Why**: Homebrew is the macOS package manager. Everything else installs through it.

- Open Terminal.app
- Run the Homebrew installer:
  ```bash
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  ```
- Follow the post-install instructions to add Homebrew to your PATH (the installer prints these):
  ```bash
  echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
  eval "$(/opt/homebrew/bin/brew shellenv)"
  ```
- Verify: `brew --version`

---

### 2. Install Foundation Tools
**Why**: These are the runtime dependencies that hooks and projects need.

- Install all tools in one batch:
  ```bash
  brew install git gh uv node bun just
  ```
- What each tool does:
  | Tool | Purpose | Used By |
  |------|---------|---------|
  | `git` | Version control | Everything |
  | `gh` | GitHub CLI (PRs, issues, API) | session_start.py (fetches recent issues) |
  | `uv` | Fast Python package manager | Every hook (PEP 723 single-file scripts) |
  | `node` | JavaScript runtime | npm-based projects |
  | `bun` | Fast JS/TS runtime | TypeScript projects, task-manager app |
  | `just` | Command runner | install-and-maintain justfile |

- Authenticate GitHub CLI:
  ```bash
  gh auth login
  ```
- Verify all tools:
  ```bash
  git --version && gh --version && uv --version && node --version && bun --version && just --version
  ```

---

### 3. Install Claude Code CLI
**Why**: The CLI is what actually runs hooks. Claude Desktop does NOT support hooks — only the CLI does.

- Install via npm (globally):
  ```bash
  npm install -g @anthropic-ai/claude-code
  ```
- Verify:
  ```bash
  claude --version
  ```
- Run Claude Code once to complete initial setup:
  ```bash
  claude
  ```
  - This creates `~/.claude/` directory structure
  - Follow the authentication prompts to link your Anthropic account
  - Exit with `/exit` after setup

> **Important distinction**: Claude Desktop (the app you already have) is for chat. Claude Code (the CLI) is for coding with hooks, sub-agents, and tool use. They are separate products. Hooks only work in the CLI.

---

### 4. Create Global Directory Structure
**Why**: Global hooks live in `~/.claude/` and apply to every project you open with Claude Code.

- Create the directory tree:
  ```bash
  mkdir -p ~/.claude/hooks/utils/llm
  mkdir -p ~/.claude/hooks/utils/tts
  mkdir -p ~/.claude/hooks/validators
  mkdir -p ~/.claude/status_lines
  mkdir -p ~/.claude/logs
  ```

---

### 5. Set Up Global Environment Variables
**Why**: Hooks that call LLM APIs or TTS services need API keys. These go in a dotfile, never in a repo.

- Create `~/.env` with your API keys:
  ```bash
  cat > ~/.env << 'EOF'
  ANTHROPIC_API_KEY=sk-ant-...
  OPENAI_API_KEY=sk-...
  ELEVENLABS_API_KEY=...
  ENGINEER_NAME=YourName
  EOF
  ```
- At minimum you need `ANTHROPIC_API_KEY`. The others are optional and enable:
  | Key | Enables |
  |-----|---------|
  | `ANTHROPIC_API_KEY` | Claude API calls in hooks (agent naming, summaries) |
  | `OPENAI_API_KEY` | OpenAI TTS, LLM-generated completion messages |
  | `ELEVENLABS_API_KEY` | High-quality TTS voice feedback |
  | `ENGINEER_NAME` | Personalized status line and session context |

- Secure the file:
  ```bash
  chmod 600 ~/.env
  ```

---

### 6. Install Global Security Hook (PreToolUse)
**Why**: This is the most important global hook. It blocks dangerous `rm -rf` commands and prevents `.env` file access across ALL projects. Adapted from `claude-code-hooks-mastery/.claude/hooks/pre_tool_use.py`.

- Copy the security hook to global location:
  ```bash
  cp /path/to/claude-code-hooks-mastery/.claude/hooks/pre_tool_use.py ~/.claude/hooks/pre_tool_use.py
  ```
- **What it does**:
  - Blocks `rm -rf /`, `rm -rf ~`, `rm -rf *` and similar destructive patterns
  - Blocks reading/writing `.env` files (protects your API keys from being leaked)
  - Logs all tool invocations to `logs/pre_tool_use.json`
  - Uses exit code 2 to block dangerous operations before they execute

- **Adaptation needed for global use**: Change the log path from `Path.cwd() / 'logs'` to `Path.home() / '.claude' / 'logs'` so global logs don't pollute project directories. Or keep per-project logging if you prefer visibility.

---

### 7. Install Global Session & Lifecycle Hooks
**Why**: These provide awareness of what's happening across all Claude Code sessions.

Copy and adapt these hooks from this repo to `~/.claude/hooks/`:

- **permission_request.py** — Audits every permission dialog. With `--log-only` flag, it just logs. You can later add auto-allow rules for read-only operations.
  ```bash
  cp /path/to/claude-code-hooks-mastery/.claude/hooks/permission_request.py ~/.claude/hooks/
  ```

- **stop.py** — Runs when Claude finishes a task. With `--chat` flag, it saves the conversation transcript as readable JSON. Optional `--notify` flag enables TTS announcements.
  ```bash
  cp /path/to/claude-code-hooks-mastery/.claude/hooks/stop.py ~/.claude/hooks/
  ```

- **post_tool_use_failure.py** — Logs tool failures with structured error context. Useful for debugging when hooks or tools break.
  ```bash
  cp /path/to/claude-code-hooks-mastery/.claude/hooks/post_tool_use_failure.py ~/.claude/hooks/
  ```

- **Copy the utils directory** (needed by stop.py for TTS/LLM):
  ```bash
  cp -r /path/to/claude-code-hooks-mastery/.claude/hooks/utils ~/.claude/hooks/
  ```

---

### 8. Install Global Status Line
**Why**: The status line shows real-time info at the bottom of your terminal while Claude Code runs — git branch, context window usage, cost tracking, etc.

- Copy the recommended status line (v6 — context window usage bar):
  ```bash
  cp /path/to/claude-code-hooks-mastery/.claude/status_lines/status_line_v6.py ~/.claude/status_lines/status_line.py
  ```
- **Available versions** (you can swap later):
  | Version | Shows |
  |---------|-------|
  | v1 | Basic git info |
  | v2 | Smart prompts with colors |
  | v3 | Agent session tracking |
  | v4 | Extended metadata |
  | v5 | Cost tracking ($) |
  | v6 | Context window usage bar (recommended) |
  | v7 | Session duration timer |
  | v8 | Token/cache statistics |
  | v9 | Minimal powerline style |

---

### 9. Wire Global Settings
**Why**: `~/.claude/settings.json` tells Claude Code which hooks to run and when. Without this file, the hook scripts are just idle Python files.

- Create `~/.claude/settings.json`:
  ```json
  {
    "permissions": {
      "allow": [
        "Bash(ls:*)",
        "Bash(git:*)",
        "Bash(gh:*)",
        "Bash(uv:*)",
        "Bash(npm:*)",
        "Bash(bun:*)",
        "Bash(find:*)",
        "Bash(grep:*)",
        "Bash(mkdir:*)",
        "Bash(cp:*)",
        "Bash(mv:*)",
        "Bash(touch:*)",
        "Bash(chmod:*)",
        "Write",
        "Edit"
      ],
      "deny": []
    },
    "statusLine": {
      "type": "command",
      "command": "uv run ~/.claude/status_lines/status_line.py",
      "padding": 0
    },
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "",
          "hooks": [
            {
              "type": "command",
              "command": "uv run ~/.claude/hooks/pre_tool_use.py"
            }
          ]
        }
      ],
      "PermissionRequest": [
        {
          "matcher": "",
          "hooks": [
            {
              "type": "command",
              "command": "uv run ~/.claude/hooks/permission_request.py --log-only"
            }
          ]
        }
      ],
      "Stop": [
        {
          "matcher": "",
          "hooks": [
            {
              "type": "command",
              "command": "uv run ~/.claude/hooks/stop.py --chat"
            }
          ]
        }
      ],
      "PostToolUseFailure": [
        {
          "matcher": "",
          "hooks": [
            {
              "type": "command",
              "command": "uv run ~/.claude/hooks/post_tool_use_failure.py"
            }
          ]
        }
      ]
    }
  }
  ```

- **What each hook does in this config**:
  | Hook Event | When It Fires | What This Config Does |
  |------------|---------------|----------------------|
  | `PreToolUse` | Before any tool executes | Blocks dangerous commands, logs all tool calls |
  | `PermissionRequest` | When Claude asks for permission | Logs the request for auditing |
  | `Stop` | When Claude finishes a response | Saves transcript as readable JSON |
  | `PostToolUseFailure` | When a tool call fails | Logs errors with structured context |

- **What's intentionally NOT in global** (these belong per-project):
  | Hook Event | Why Per-Project |
  |------------|-----------------|
  | `PostToolUse` | Validators (ruff, ty) are project-specific |
  | `UserPromptSubmit` | Agent naming and session data are per-project |
  | `Setup` | Init/maintenance logic differs per project |
  | `SessionStart` | Context loading (TODO files, issues) is per-project |
  | `SubagentStart/Stop` | Agent configurations differ per project |
  | `Notification` | TTS preferences vary |

---

### 10. Test the Global Setup
**Why**: Verify all hooks fire correctly before building the per-project layer.

- Navigate to any directory and launch Claude Code:
  ```bash
  cd /tmp && claude
  ```
- Ask Claude to do something simple:
  ```
  What files are in this directory?
  ```
- Check that logs were created:
  ```bash
  ls ~/.claude/logs/
  # Should see: pre_tool_use.json, stop.json
  ```
- Check the status line appears at the bottom of the terminal
- Exit Claude Code and verify the transcript was saved:
  ```bash
  cat ~/.claude/logs/stop.json | python3 -m json.tool | head -20
  ```

---

### 11. Clone the IndyDevDan Repos
**Why**: These are your reference implementations and source material for per-project hooks.

- Create a workspace directory:
  ```bash
  mkdir -p ~/dev/claude-hooks-reference
  cd ~/dev/claude-hooks-reference
  ```
- Clone the key repos:
  ```bash
  gh repo clone disler/claude-code-hooks-mastery
  gh repo clone disler/install-and-maintain
  gh repo clone disler/claude-code-damage-control
  gh repo clone disler/claude-code-hooks-multi-agent-observability
  ```

---

### 12. Set Up the Install-and-Maintain Pattern
**Why**: This is the per-project bootstrapping system. When you start a new project, you copy this `.claude/` skeleton and run `claude --init` to set everything up automatically.

- The pattern from `disler/install-and-maintain` uses three hooks:
  | Hook | Trigger | What It Does |
  |------|---------|--------------|
  | `Setup[init]` | `claude --init` | Installs deps, seeds DB, configures env |
  | `Setup[maintenance]` | `claude --maintenance` | Updates deps, runs DB health checks |
  | `SessionStart` | Every session | Loads `.env` into `CLAUDE_ENV_FILE` |

- The three-tier usage model:
  ```bash
  # Tier 1: Deterministic (CI-friendly, no LLM)
  claude --init-only

  # Tier 2: Agentic (hook runs, then Claude analyzes results)
  claude --init "/install"

  # Tier 3: Interactive (hook runs, Claude asks you 5 config questions)
  claude --init "/install true"
  ```

- To use this in a new project, you would:
  1. Copy the `.claude/` directory from install-and-maintain as a starting template
  2. Customize `setup_init.py` for your project's specific dependencies
  3. Add project-specific agents, commands, and validators
  4. Run `claude --init` to bootstrap

---

### 13. (Optional) Install Ollama for Local LLM
**Why**: Several hooks use an LLM fallback chain (OpenAI > Anthropic > Ollama). Ollama runs models locally on your M4 — no API key, no cost, no latency.

- Install Ollama:
  ```bash
  brew install ollama
  ```
- Start the service:
  ```bash
  ollama serve &
  ```
- Pull a small, fast model:
  ```bash
  ollama pull llama3.2:3b
  ```
- The hooks in this repo already have Ollama support built in (see `.claude/hooks/utils/llm/ollama.py`). The M4's Neural Engine makes local inference fast.

---

### 14. (Optional) Configure MCP Servers
**Why**: MCP (Model Context Protocol) servers extend Claude Code with external tool access — web scraping, TTS, database queries, etc.

- Create `~/.claude/.mcp.json` for global MCP servers:
  ```json
  {
    "mcpServers": {
      "ElevenLabs": {
        "command": "uvx",
        "args": ["elevenlabs-mcp"],
        "env": {
          "ELEVENLABS_API_KEY": "your-key-here"
        }
      }
    }
  }
  ```
- Other MCP servers to consider:
  | Server | What It Provides |
  |--------|-----------------|
  | `elevenlabs-mcp` | Text-to-speech in Claude responses |
  | `firecrawl-mcp` | Web scraping and search |
  | `just-prompt` (disler repo) | Unified multi-LLM provider access |

---

### 15. (Optional) Set Up Observability
**Why**: `disler/claude-code-hooks-multi-agent-observability` adds a real-time monitoring dashboard for agent activity. Useful when running complex multi-agent workflows.

- Clone and review:
  ```bash
  cd ~/dev/claude-hooks-reference/claude-code-hooks-multi-agent-observability
  ```
- This repo provides hooks that stream agent events to a monitoring UI
- Install when you're comfortable with the basic hooks and want deeper visibility into multi-agent sessions

---

### 16. Validate the Complete Setup
**Why**: Confirm every layer works together before using this for real work.

Run these checks in order:

```bash
# Layer 1: Foundation tools
brew --version
git --version
gh --version
uv --version
node --version
bun --version
just --version
claude --version

# Layer 2: Global hooks structure
ls ~/.claude/hooks/
ls ~/.claude/status_lines/
cat ~/.claude/settings.json | python3 -m json.tool

# Layer 2: Global hooks fire correctly
cd /tmp && claude -p "list files in this directory" 2>/dev/null
ls ~/.claude/logs/

# Layer 3: Per-project pattern works
cd ~/dev/claude-hooks-reference/install-and-maintain
claude --init-only
```

---

## Testing Strategy

### Smoke Tests (do these first)
1. Run `claude` in an empty directory — status line should appear, no errors
2. Ask Claude to read a file — `pre_tool_use.py` should log the event
3. Ask Claude to `rm -rf /` — should be BLOCKED by the security hook
4. Ask Claude to read `.env` — should be BLOCKED
5. Exit Claude — `stop.py` should save the transcript

### Integration Tests
1. Open a project with both global and per-project hooks — both should fire
2. Run `claude --init` in the install-and-maintain repo — setup hook should execute
3. Check `~/.claude/logs/` for structured JSON after a session

### Validation Commands (Layer 1)
```bash
brew --version          # Homebrew installed
git --version           # Git available
gh auth status          # GitHub CLI authenticated
uv --version            # UV package manager
node --version          # Node.js runtime
bun --version           # Bun runtime
just --version          # Just command runner
claude --version        # Claude Code CLI
```

### Validation Commands (Layer 2)
```bash
# All global hook scripts compile
uv run python -m py_compile ~/.claude/hooks/pre_tool_use.py
uv run python -m py_compile ~/.claude/hooks/permission_request.py
uv run python -m py_compile ~/.claude/hooks/stop.py
uv run python -m py_compile ~/.claude/hooks/post_tool_use_failure.py

# Settings is valid JSON
python3 -m json.tool ~/.claude/settings.json > /dev/null && echo "Valid JSON"

# Status line runs without error
echo '{}' | uv run ~/.claude/status_lines/status_line.py
```

### Validation Commands (Layer 3)
```bash
# Per-project hooks compile (from install-and-maintain)
cd ~/dev/claude-hooks-reference/install-and-maintain
uv run python -m py_compile .claude/hooks/setup_init.py
uv run python -m py_compile .claude/hooks/setup_maintenance.py
uv run python -m py_compile .claude/hooks/session_start.py
```

## Acceptance Criteria

1. **Foundation tools installed**: Homebrew, git, gh (authenticated), uv, node, bun, just, claude CLI all return valid version numbers
2. **Global hooks directory exists**: `~/.claude/hooks/` contains pre_tool_use.py, permission_request.py, stop.py, post_tool_use_failure.py, and utils/
3. **Global settings wired**: `~/.claude/settings.json` is valid JSON with PreToolUse, PermissionRequest, Stop, and PostToolUseFailure hooks configured
4. **Status line works**: Running Claude Code shows a status line at the bottom of the terminal
5. **Security hook blocks**: Attempting `rm -rf /` or `.env` file access is blocked with exit code 2
6. **Logging works**: After a Claude Code session, `~/.claude/logs/` contains JSON log files
7. **Transcript saved**: The Stop hook saves a readable `chat.json` transcript after each session
8. **Per-project pattern available**: The install-and-maintain repo is cloned and `claude --init-only` succeeds
9. **All hook scripts compile**: Every `.py` file in `~/.claude/hooks/` passes `py_compile`
10. **Environment variables set**: `~/.env` exists with at least `ANTHROPIC_API_KEY`

## Architecture Reference

### How Claude Code Resolves Settings
```
~/.claude/settings.json          ← User-global (this spec builds this)
  + .claude/settings.json        ← Project-level (committed to repo)
    + .claude/settings.local.json ← Local overrides (gitignored)
      = Merged settings             ← What Claude Code actually uses
```

Hooks from all levels execute. If global defines `PreToolUse` and a project also defines `PreToolUse`, **both run** (global first).

### Hook Communication Model
```
stdin (JSON) → Hook Script → stdout (JSON, optional) + exit code
                                │
                                ├── exit 0 = success, continue
                                ├── exit 1 = error (logged, continues)
                                └── exit 2 = BLOCK (stops the tool call)
```

### IndyDevDan Repos Quick Reference
| Repo | Stars | Use For |
|------|-------|---------|
| claude-code-hooks-mastery | 2,656 | All 13 hook types, sub-agents, status lines |
| install-and-maintain | 65 | Setup/maintenance bootstrapping pattern |
| claude-code-damage-control | 354 | Safety guardrails |
| claude-code-hooks-multi-agent-observability | 960 | Real-time monitoring |
| claude-code-is-programmable | 288 | Programmatic Claude Code usage |
| single-file-agents | 424 | Compact agent patterns |
| just-prompt | 705 | Multi-provider LLM MCP server |

## Notes

- **macOS Arm64**: All tools listed support Apple Silicon natively. No Rosetta needed.
- **uv is critical**: Every hook uses `#!/usr/bin/env -S uv run --script` — this is the PEP 723 pattern that lets each script declare its own dependencies inline. Without `uv`, no hooks run.
- **Claude Desktop vs Claude Code**: Claude Desktop (the app) is for chat. Claude Code (the CLI, `claude` command) is for coding with hooks. This entire spec is for the CLI. They can coexist.
- **Global vs. project paths**: Global hooks should use `Path.home() / '.claude' / 'logs'` for logging. Project hooks use `Path.cwd() / 'logs'`. Be consistent.
- **Security first**: The PreToolUse security hook is the one hook that should ALWAYS be global. It protects every project from accidental destructive commands.
- **Incremental setup**: You don't need to do everything at once. Phase 1 + Phase 2 (steps 1-10) give you a fully functional setup. Phase 3 and 4 (steps 11-15) are enhancements you can add later.
- **Estimated disk usage**: ~500MB total (Homebrew, Node, Bun, uv, Ollama model). The hooks themselves are negligible.
