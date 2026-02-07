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

## Complete Dependency Manifest

Before diving into steps, here is every dependency the hooks ecosystem requires, so you can see the full picture:

### System-Level Tools (installed via Homebrew)

| Tool | Version | Required? | What Needs It |
|------|---------|-----------|---------------|
| Xcode CLI Tools | Latest | **YES** | Homebrew, git, compilers — prerequisite for everything |
| Homebrew | Latest | **YES** | Package manager — installs all other tools |
| `git` | 2.x+ | **YES** | Every hook that checks repo state, all version control |
| `gh` | Latest | Recommended | `session_start.py` (fetches GitHub issues), PR workflows |
| `uv` | Latest | **YES** | **Critical** — runs every hook via PEP 723 `uv run --script` |
| `python3` | 3.11+ | **YES** | Installed by `uv` automatically, or via Homebrew as fallback |
| `node` | 20+ LTS | **YES** | Required to install Claude Code CLI via npm |
| `npm` | 10+ | **YES** | Comes with Node — installs Claude Code CLI globally |
| `bun` | Latest | Recommended | TypeScript projects, task-manager demo app |
| `just` | Latest | Optional | Command runner for install-and-maintain `justfile` |
| `ollama` | Latest | **YES** | Local LLM inference — no API key, no cost. M4 Neural Engine accelerated |
| `whisper-cpp` | Latest | **YES** | Local speech-to-text — no API key, Core ML accelerated on M4. ~27x real-time |
| `docker` | Latest | Optional | Containerized services (databases, CI testing). Not needed by hooks |

### Python Packages (auto-resolved by `uv` via PEP 723 headers)

> **How this works**: Every hook script has a `# dependencies = [...]` block in its header. When `uv run --script` executes the hook, `uv` automatically downloads, caches, and resolves these packages. You do NOT need to `pip install` them manually. First run of each hook is slightly slower (~2-5s) while uv caches; subsequent runs are instant.

| Package | Used By | Purpose |
|---------|---------|---------|
| `python-dotenv` | 16+ hooks, all status lines | Loads `.env` files into environment |
| `anthropic` | `utils/llm/anth.py`, `user_prompt_submit.py`, `subagent_stop.py` | Claude API calls (agent naming, task summaries) |
| `openai` | `utils/llm/oai.py`, `utils/tts/openai_tts.py` | OpenAI API calls, TTS voice generation |
| `elevenlabs` | `utils/tts/elevenlabs_tts.py` | High-quality text-to-speech |
| `pyttsx3` | `utils/tts/pyttsx3_tts.py` | Offline TTS (uses macOS AVFoundation — no API key needed) |

### Code Quality Tools (invoked via `uvx` — no install needed)

| Tool | Used By | Purpose |
|------|---------|---------|
| `ruff` | `validators/ruff_validator.py` | Python linting (invoked as `uvx ruff check`) |
| `ty` | `validators/ty_validator.py` | Python type checking (invoked as `uvx ty check`) |

> **`uvx` explained**: Like `npx` for Python. It downloads a tool into an isolated environment and runs it — no global install pollution. The validators call `uvx ruff` and `uvx ty`, so these tools are fetched on demand.

### API Keys (Environment Variables)

| Key | Required? | What It Enables |
|-----|-----------|-----------------|
| `ANTHROPIC_API_KEY` | **YES** (for Claude Code itself) | Claude Code authentication, hook LLM calls |
| `OPENAI_API_KEY` | Optional | OpenAI TTS, LLM-generated completion messages in `stop.py` |
| `ELEVENLABS_API_KEY` | Optional | Premium TTS voice feedback |
| `ENGINEER_NAME` | Optional | Personalized status line and session greetings |
| `OLLAMA_MODEL` | Optional | Override default Ollama model (default: `gpt-oss:20b`) |
| `OLLAMA_HOST` | Optional | Override Ollama endpoint (default: `http://localhost:11434`) |

### Accounts Required

| Service | Sign Up | Free Tier? | What It's For | Required? |
|---------|---------|------------|---------------|-----------|
| **Anthropic** | [console.anthropic.com](https://console.anthropic.com) | Pay-per-use only | Claude Code itself + `anth.py` hook LLM calls | **YES** |
| **GitHub** | [github.com](https://github.com) | Free | `gh auth login` — issue fetching, PR workflows | **YES** |
| **Ollama** | [ollama.com](https://ollama.com) | **No account needed** | Fully local LLM — no signup, no API key, no cost | N/A |
| **whisper.cpp** | [github.com/ggml-org/whisper.cpp](https://github.com/ggml-org/whisper.cpp) | **No account needed** | Fully local STT — no signup, no API key, no cost | N/A |
| **OpenAI** | [platform.openai.com](https://platform.openai.com) | $5 free credit (new accounts) | TTS voice, LLM completion messages in `stop.py` | Optional |
| **ElevenLabs** | [elevenlabs.io](https://elevenlabs.io) | 10k chars/month free | Premium TTS voices | Optional |

> **Minimum to function**: Anthropic account (for Claude Code) + GitHub account (free). With Ollama as a local fallback, you get agent naming and completion messages without any additional paid API keys.

### macOS-Specific Notes

| Concern | Status |
|---------|--------|
| Apple Silicon (ARM64) | All tools have native ARM64 builds. No Rosetta 2 needed. |
| Audio output for TTS | macOS AVFoundation is built-in. `pyttsx3` uses it automatically. |
| Audio input for STT | Built-in microphone + Core Audio. whisper.cpp uses it natively. Install `sox` for real-time mic recording. |
| Core ML (Neural Engine) | whisper.cpp and Ollama both leverage the M4 ANE for hardware acceleration. macOS Sonoma 14+ recommended. |
| File locking (`fcntl`) | Built into Python on macOS/Unix. Used by `tts_queue.py`. |
| SQLite | Built into `bun:sqlite` and Python stdlib. No install needed. |

## Step by Step Tasks

IMPORTANT: Execute every step in order, top to bottom.

---

### 1. Install Xcode Command Line Tools
**Why**: This is the true starting point on a fresh Mac. Xcode CLI Tools provide `git`, C/C++ compilers, and headers that Homebrew and many packages need to build. Without this, `brew install` will fail.

- Open Terminal.app (Cmd+Space, type "Terminal")
- Run:
  ```bash
  xcode-select --install
  ```
- A dialog will pop up — click **Install** and wait (~5-10 minutes, ~1.5GB download)
- Verify:
  ```bash
  xcode-select -p
  # Should output: /Library/Developer/CommandLineTools
  git --version
  # Should output: git version 2.x.x (Apple Git-xxx)
  ```

> **Note**: This gives you Apple's bundled `git`. We'll install a newer version via Homebrew next, but the bundled one is needed for Homebrew's own installation.

---

### 2. Install Homebrew
**Why**: Homebrew is the macOS package manager. Everything else installs through it.

- Run the Homebrew installer:
  ```bash
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  ```
- **Important** — the installer prints two commands at the end. Run them:
  ```bash
  echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
  eval "$(/opt/homebrew/bin/brew shellenv)"
  ```
  - The first line makes Homebrew available in future terminal sessions
  - The second line makes it available right now
- Verify:
  ```bash
  brew --version
  ```

---

### 3. Install Foundation Tools
**Why**: These are the runtime dependencies that hooks and projects need.

- Install all tools in one batch:
  ```bash
  brew install git gh uv python@3.13 node bun just ollama whisper-cpp
  ```
- What each tool does:
  | Tool | Purpose | Used By |
  |------|---------|---------|
  | `git` | Version control (newer than Apple's bundled version) | Every hook that checks repo state |
  | `gh` | GitHub CLI — PRs, issues, API access | `session_start.py` (fetches recent issues) |
  | `uv` | **Critical** — fast Python package manager + script runner | Every hook via `uv run --script` |
  | `python@3.13` | Python runtime (hooks require 3.11+) | All hook scripts |
  | `node` | JavaScript runtime (includes npm) | Required to install Claude Code CLI |
  | `bun` | Fast JS/TS runtime with built-in SQLite | TypeScript projects, task-manager demo |
  | `just` | Command runner (like make, but simpler) | install-and-maintain `justfile` |
  | `ollama` | Local LLM — runs on M4 Neural Engine, no API key | Agent naming, task summaries, completion messages |
  | `whisper-cpp` | Local speech-to-text — Core ML accelerated, no API key | Voice input, audio transcription |

- Start Ollama as a background service (auto-starts on boot):
  ```bash
  brew services start ollama
  ```

- Pull a model for hook use (choose based on your RAM):
  ```bash
  # 16GB M4 — small & fast, good for agent naming
  ollama pull llama3.2:3b

  # 24GB+ M4 Pro/Max — richer responses
  ollama pull llama3.2:8b
  ```

- Download a Whisper model for speech-to-text (choose based on accuracy needs):
  ```bash
  # Download the base model (~142MB) — good balance of speed and accuracy
  whisper-cpp-download-ggml-model base

  # Or the small model (~466MB) — better accuracy, still fast on M4
  whisper-cpp-download-ggml-model small
  ```

- Authenticate GitHub CLI:
  ```bash
  gh auth login
  ```
  - Select: **GitHub.com** > **HTTPS** > **Login with a web browser**
  - Follow the browser flow to authenticate

- Verify all tools:
  ```bash
  git --version
  gh --version
  uv --version
  python3 --version    # Should show 3.13.x
  node --version       # Should show v20+ or v22+
  npm --version
  bun --version
  just --version
  ollama list             # Should show your pulled model
  whisper-cpp --help       # Should show whisper.cpp usage
  ```

---

### 4. Install Claude Code CLI
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
  - This creates the `~/.claude/` directory structure automatically
  - Follow the authentication prompts to link your Anthropic account
  - Exit with `/exit` after setup completes

> **Important distinction**: Claude Desktop (the app you already have) is for chat conversations. Claude Code (the CLI, the `claude` command in Terminal) is for coding with hooks, sub-agents, status lines, and tool use. They are separate products that coexist. Hooks, sub-agents, and everything in this spec are CLI-only features.

---

### 5. Create Global Directory Structure
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

### 6. Set Up Global Environment Variables
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

### 7. Pre-Warm Python Dependency Cache
**Why**: Every hook declares its Python dependencies inline (PEP 723). The first time `uv run` executes a hook, it downloads and caches those packages — adding 2-5 seconds. Pre-warming ensures all hooks run instantly from the start.

- Pre-warm all packages the hooks will need:
  ```bash
  uv pip install --system python-dotenv anthropic openai elevenlabs pyttsx3
  ```
- **What each package does**:
  | Package | Size | What Uses It |
  |---------|------|-------------|
  | `python-dotenv` | ~30KB | Every hook and status line — loads `.env` files |
  | `anthropic` | ~2MB | Claude API calls in agent naming, task summaries |
  | `openai` | ~2MB | OpenAI API calls, TTS voice generation |
  | `elevenlabs` | ~1MB | Premium TTS (only if you have the API key) |
  | `pyttsx3` | ~100KB | Offline TTS using macOS AVFoundation (no API key) |

- **If you don't want to install globally**, you can skip this step. `uv run --script` will auto-resolve packages on first invocation per-hook. The tradeoff is a one-time ~3s delay per hook on first run.

- Verify ruff and ty are fetchable (used by per-project validators):
  ```bash
  uvx ruff --version
  uvx ty --version
  ```
  These download on first use and cache automatically. Running them once here pre-warms the cache.

---

### 8. Install Global Security Hook (PreToolUse)
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

### 9. Install Global Session & Lifecycle Hooks
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

### 10. Install Global Status Line
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

### 11. Wire Global Settings
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

### 12. Test the Global Setup
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

### 13. Clone the IndyDevDan Repos
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

### 14. Set Up the Install-and-Maintain Pattern
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

### 15. Local AI Configuration Reference (Ollama + Whisper)
**Note**: Both tools were installed in Step 3. This section provides detailed configuration and model management.

#### Ollama (Local LLM — Text Generation)

- **How hooks use Ollama**: The LLM utility at `.claude/hooks/utils/llm/ollama.py` connects to `http://localhost:11434/v1` using the OpenAI-compatible API. It uses the `OLLAMA_MODEL` env var (default: `gpt-oss:20b`). Add to your `~/.env`:
  ```bash
  OLLAMA_MODEL=llama3.2:3b    # Match the model you pulled in Step 3
  OLLAMA_HOST=http://localhost:11434  # Default, only change if custom port
  ```

- **M4 performance expectations (LLM)**:
  | Model | RAM Used | Tokens/sec | Good For |
  |-------|----------|-----------|----------|
  | llama3.2:3b | ~2GB | ~60 tok/s | Agent names, short completions |
  | llama3.2:8b | ~5GB | ~35 tok/s | Task summaries, completion messages |
  | gpt-oss:20b | ~12GB | ~15 tok/s | Richer responses (needs 24GB+ RAM) |

- Pull additional models later as needed:
  ```bash
  ollama pull gpt-oss:20b          # Larger model, M4 Pro/Max with 24GB+
  ```

- Verify Ollama is running:
  ```bash
  ollama list                      # Shows downloaded models
  curl http://localhost:11434/v1/models  # API responds
  ```

#### whisper.cpp (Local STT — Speech-to-Text)

- **What it is**: A C/C++ port of OpenAI's Whisper model, optimized for Apple Silicon. Uses Core ML and the M4 Neural Engine for hardware-accelerated inference. 100% local — audio never leaves your machine. MIT license, no account, no API key.

- **Why not the OpenAI Whisper API?**: The cloud API costs $0.006/min and sends your audio to OpenAI's servers. whisper.cpp is free, private, and on M4 hardware it's **faster than real-time**.

- **Available models** (downloaded in Step 3):
  | Model | Size | Speed on M4 | Accuracy | Best For |
  |-------|------|-------------|----------|----------|
  | tiny | ~75MB | ~27x real-time | Good | Quick commands, short phrases |
  | base | ~142MB | ~16x real-time | Better | General transcription (recommended start) |
  | small | ~466MB | ~8x real-time | Great | Meetings, detailed transcription |
  | medium | ~1.5GB | ~4x real-time | Excellent | High-accuracy professional use |
  | large-v3 | ~3GB | ~2x real-time | Best | Maximum accuracy, multi-language |

- Download additional models:
  ```bash
  whisper-cpp-download-ggml-model medium    # Higher accuracy
  whisper-cpp-download-ggml-model large-v3  # Maximum accuracy
  ```

- **Basic usage** — transcribe an audio file:
  ```bash
  # Transcribe a WAV file
  whisper-cpp -m /opt/homebrew/share/whisper-cpp/models/ggml-base.bin -f audio.wav

  # Transcribe with timestamps
  whisper-cpp -m /opt/homebrew/share/whisper-cpp/models/ggml-base.bin -f audio.wav -otxt

  # Real-time microphone input (requires sox for recording)
  brew install sox
  rec -c 1 -r 16000 -t wav - | whisper-cpp -m /opt/homebrew/share/whisper-cpp/models/ggml-base.bin -f -
  ```

- **Core ML acceleration** (optional, for maximum M4 performance):
  ```bash
  # Generate Core ML model for Neural Engine acceleration
  whisper-cpp-generate-coreml-model base
  ```
  This creates an ANE-optimized model that runs ~3x faster than the standard model on Apple Silicon.

- **Alternatives to consider later**:
  | Tool | Language | Notes |
  |------|----------|-------|
  | [WhisperKit](https://github.com/argmaxinc/WhisperKit) | Swift | Native Apple framework, real-time streaming |
  | [mlx-whisper](https://github.com/ml-explore/mlx-examples) | Python | Apple MLX framework, optimized for Apple Silicon |
  | `openai-whisper` | Python | Original open-source model, heavier deps |

#### The Local AI Stack Summary

With Ollama + whisper.cpp + pyttsx3, you have a complete **voice I/O pipeline** running entirely on your M4 with zero cloud dependencies:

```
Voice Input → [whisper.cpp: STT] → Text → [Ollama: LLM] → Response → [pyttsx3: TTS] → Audio Output
```

All three run locally, require no API keys, and leverage the M4's Neural Engine.

---

### 16. (Optional) Configure MCP Servers
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

### 17. (Optional) Set Up Observability
**Why**: `disler/claude-code-hooks-multi-agent-observability` adds a real-time monitoring dashboard for agent activity. Useful when running complex multi-agent workflows.

- Clone and review:
  ```bash
  cd ~/dev/claude-hooks-reference/claude-code-hooks-multi-agent-observability
  ```
- This repo provides hooks that stream agent events to a monitoring UI
- Install when you're comfortable with the basic hooks and want deeper visibility into multi-agent sessions

---

### 18. (Optional) Install Docker
**Why**: Docker is not needed by any hooks or Claude Code features, but many real-world projects use it for databases, microservices, CI pipeline testing, and devcontainers.

- Install Docker Desktop for Mac:
  ```bash
  brew install --cask docker
  ```
- Launch Docker Desktop from Applications (first launch requires granting permissions)
- Docker Desktop runs a background daemon (~1-2GB RAM when idle)
- Verify:
  ```bash
  docker --version
  docker compose version
  ```

- **When you need Docker**:
  | Use Case | Example |
  |----------|---------|
  | Running databases | `docker run -d postgres:16`, `docker run -d redis:7` |
  | Devcontainers | VS Code / Cursor remote container development |
  | CI testing | Running GitHub Actions locally with `act` |
  | MCP servers | Some MCP servers ship as Docker images |

- **When you don't need Docker**:
  - The hooks ecosystem is pure Python + uv — no containers
  - Ollama runs natively on macOS (no Docker wrapper)
  - SQLite is built into Bun and Python — no database container needed
  - Simple projects with no external service dependencies

> **Tip**: Docker Desktop uses ~2GB disk + ~1-2GB RAM as a background daemon. If disk/memory are a concern, install only when a project requires it. You can quit Docker Desktop when not in use.

---

### 19. Validate the Complete Setup
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

### Validation Commands (Layer 1 — Foundation)
```bash
# System prerequisite
xcode-select -p                  # Xcode CLI Tools installed

# Package manager
brew --version                   # Homebrew installed

# Core tools
git --version                    # Git (Homebrew version)
gh auth status                   # GitHub CLI authenticated
uv --version                     # UV package manager
python3 --version                # Python 3.11+ available
node --version                   # Node.js runtime
npm --version                    # npm (comes with Node)
bun --version                    # Bun runtime
just --version                   # Just command runner
claude --version                 # Claude Code CLI

# Python packages pre-warmed
python3 -c "import dotenv; print('python-dotenv OK')"
python3 -c "import anthropic; print('anthropic OK')"
python3 -c "import openai; print('openai OK')"

# Code quality tools reachable
uvx ruff --version               # Ruff linter
uvx ty --version                 # Ty type checker

# Ollama running (essential)
ollama list                      # Shows downloaded models
curl -s http://localhost:11434/v1/models | python3 -m json.tool  # API responds

# whisper.cpp installed (essential)
whisper-cpp --help               # Shows usage info
ls /opt/homebrew/share/whisper-cpp/models/  # Should show downloaded model(s)

# Docker (optional)
docker --version                 # Only if installed
docker compose version           # Only if installed
```

### Validation Commands (Layer 2 — Global Hooks)
```bash
# All global hook scripts compile
uv run python -m py_compile ~/.claude/hooks/pre_tool_use.py
uv run python -m py_compile ~/.claude/hooks/permission_request.py
uv run python -m py_compile ~/.claude/hooks/stop.py
uv run python -m py_compile ~/.claude/hooks/post_tool_use_failure.py

# Utils compile
uv run python -m py_compile ~/.claude/hooks/utils/llm/oai.py
uv run python -m py_compile ~/.claude/hooks/utils/llm/anth.py
uv run python -m py_compile ~/.claude/hooks/utils/llm/ollama.py
uv run python -m py_compile ~/.claude/hooks/utils/tts/openai_tts.py
uv run python -m py_compile ~/.claude/hooks/utils/tts/elevenlabs_tts.py
uv run python -m py_compile ~/.claude/hooks/utils/tts/pyttsx3_tts.py

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

1. **Xcode CLI Tools installed**: `xcode-select -p` returns `/Library/Developer/CommandLineTools`
2. **Foundation tools installed**: Homebrew, git, gh (authenticated), uv, python3 (3.11+), node, npm, bun, just, ollama, whisper-cpp, claude CLI all return valid version numbers
3. **Ollama running**: `ollama list` shows at least one downloaded model, `curl http://localhost:11434/v1/models` responds
4. **whisper.cpp working**: `whisper-cpp --help` succeeds and at least one model exists in `/opt/homebrew/share/whisper-cpp/models/`
5. **Python packages cached**: `python3 -c "import dotenv"` succeeds without error
6. **Code quality tools cached**: `uvx ruff --version` and `uvx ty --version` both succeed
7. **Global hooks directory exists**: `~/.claude/hooks/` contains pre_tool_use.py, permission_request.py, stop.py, post_tool_use_failure.py, and utils/ (with llm/ and tts/ subdirectories)
8. **Global settings wired**: `~/.claude/settings.json` is valid JSON with PreToolUse, PermissionRequest, Stop, and PostToolUseFailure hooks configured
9. **Status line works**: Running Claude Code shows a status line at the bottom of the terminal
10. **Security hook blocks**: Attempting `rm -rf /` or `.env` file access is blocked with exit code 2
11. **Logging works**: After a Claude Code session, `~/.claude/logs/` contains JSON log files
12. **Transcript saved**: The Stop hook saves a readable `chat.json` transcript after each session
13. **Per-project pattern available**: The install-and-maintain repo is cloned and `claude --init-only` succeeds
14. **All hook scripts compile**: Every `.py` file in `~/.claude/hooks/` passes `py_compile`
15. **Environment variables set**: `~/.env` exists with at least `ANTHROPIC_API_KEY`, permissions are `600`

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

- **macOS Arm64 (Apple Silicon)**: All tools listed have native ARM64 builds. No Rosetta 2 translation needed. The M4 chip runs everything natively.
- **Xcode CLI Tools are the real starting point**: On a fresh Mac, you can't even install Homebrew without them. Always do Step 1 first.
- **uv is the linchpin**: Every hook uses `#!/usr/bin/env -S uv run --script` — the PEP 723 pattern that lets each script declare its own dependencies inline. Without `uv`, zero hooks run. This is the single most important tool to install.
- **PEP 723 auto-resolution**: You technically don't need to pre-install Python packages (Step 6). `uv run --script` reads the `# dependencies = [...]` header and auto-resolves on first run. Pre-warming just avoids cold-start latency.
- **Claude Desktop vs Claude Code**: Claude Desktop (the macOS app you already have) is for chat conversations. Claude Code (the CLI, the `claude` command in Terminal) is for coding with hooks, sub-agents, and tool use. This entire spec is for the CLI. They coexist and use the same Anthropic account.
- **Global vs. project paths**: Global hooks should use `Path.home() / '.claude' / 'logs'` for logging. Project hooks use `Path.cwd() / 'logs'`. Be consistent to avoid polluting project directories with global log data.
- **Security first**: The PreToolUse security hook is the one hook that should ALWAYS be global. It protects every project from accidental destructive commands and `.env` leakage.
- **Incremental setup**: You don't need to do everything at once. Steps 1-12 give you a fully functional setup with global hooks and Ollama. Steps 13-19 are enhancements (per-project patterns, MCP, observability, Docker) you can add as needed.
- **Estimated disk usage**:
  | Component | Size |
  |-----------|------|
  | Xcode CLI Tools | ~1.5GB |
  | Homebrew + formulae | ~500MB |
  | Node.js + npm | ~100MB |
  | Bun | ~50MB |
  | uv + Python 3.13 | ~150MB |
  | Claude Code CLI | ~50MB |
  | Python packages (cached) | ~50MB |
  | Ollama + llama3.2:3b | ~2.5GB |
  | whisper.cpp + base model | ~200MB |
  | Docker Desktop (optional) | ~2GB |
  | **Total (essential, with Ollama 3b + Whisper base)** | **~5.2GB** |
  | **Total (with Docker + larger models)** | **~20GB** |
