# Pyromancer

**A security-first, AI-powered terminal emulator for macOS built with Metal and SwiftUI.**

Pyromancer is a native macOS terminal application by [Veilfire](https://veilfire.io) that pairs a high-performance GPU-rendered terminal with an autonomous AI agent system. Every feature is designed with a privacy-first philosophy: secrets never leave the Keychain, all AI actions are auditable, and destructive commands require explicit human approval.

---

## Table of Contents

- [Philosophy](#philosophy)
- [Security & Privacy](#security--privacy)
- [Terminal](#terminal)
- [AI Assistant](#ai-assistant)
- [Autonomous Agent](#autonomous-agent)
- [Intelligent Autocomplete](#intelligent-autocomplete)
- [Tabs & Navigation](#tabs--navigation)
- [Macros](#macros)
- [Preferences](#preferences)
- [Requirements](#requirements)

---

## Philosophy

1. **Security is not optional.** Every AI interaction passes through permission gates, risk classifiers, and a tamper-proof audit log. You are always in control of what runs in your terminal.

2. **Privacy by architecture.** API keys live exclusively in the macOS Keychain. Terminal output sent to AI providers is sanitized to remove secrets, tokens, and credentials before transmission. Autocomplete data is AES-256-GCM encrypted at rest.

3. **Performance without compromise.** The terminal renderer runs entirely on the GPU via Metal. The AI agent operates asynchronously without blocking your terminal. Autocomplete suggestions appear in under 5ms.

---

## Security & Privacy

### Human-In-The-Loop Permission System

Every command the AI wants to execute is classified by risk before it runs:

| Level | Examples | Behavior |
|-------|----------|----------|
| **Safe** | `ls`, `pwd`, `git status`, `cat` | Configurable: auto-approve or require approval |
| **Moderate** | `git commit`, `npm install`, `ssh` | Configurable: auto-approve or require approval |
| **Dangerous** | `rm -rf`, `docker rm`, `chmod 777` | Configurable: auto-approve or require approval |
| **Critical** | `sudo`, `dd`, `mkfs`, `defaults write` | Always requires approval + warning |

The permission dialog presents the exact command, the AI's stated intent, the risk level, and four approval scopes: **Deny**, **Allow Once**, **Allow for Session**, **Allow Forever**. Session grants are revocable at any time from the status bar.

### Auto-Approve Threshold

A configurable slider lets you set how much risk the agent can auto-approve without prompting -- from "every command requires approval" to auto-approving safe, moderate, or dangerous operations. An explicit opt-in "Allow All" mode is available with risk acceptance language.

### Secret Redaction

All terminal output is sanitized before reaching any AI provider:

- **Pattern-based detection**: AWS keys, GitHub PATs, JWTs, API keys, bearer tokens, connection strings, private key blocks
- **Keychain cross-reference**: Any string matching a stored Keychain secret is automatically redacted
- **Entropy analysis**: High-entropy strings near sensitive keywords are flagged
- **Environment variable filtering**: Sensitive env vars are never forwarded

Redacted content is replaced with `[REDACTED:type]` markers. The original content never leaves the local machine.

### Tamper-Proof Audit Log

Every security-relevant event is recorded in an HMAC-SHA256 signed JSONL log:

- AI provider invocations (provider, model, query hash)
- Permission decisions (command, risk level, grant/deny)
- Keychain access events
- Command executions with source attribution
- File access rule violations
- Workflow events

Each log entry is chained, making retroactive tampering detectable. Integrity verification is available in Preferences.

### Keychain-Only Secret Storage

Pyromancer never stores API keys, tokens, or credentials in files, UserDefaults, or environment variables. All sensitive material lives in the macOS Keychain with device-only access control, including:

- AI provider API keys (Anthropic, OpenAI, OpenRouter)
- User-defined secrets (with optional environment variable injection)
- Autocomplete encryption key (AES-256-GCM)
- Audit log HMAC signing key

### User-Defined Secrets

Define custom secrets and optionally inject them as environment variables into terminal sessions. Secret values are stored exclusively in the macOS Keychain. Each secret can be independently toggled for environment injection, and all injected values are still subject to redaction -- the AI never sees them.

---

## Terminal

### Metal GPU Renderer

The terminal display is rendered entirely on the GPU using Apple's Metal framework:

- Per-cell instanced rendering with position, texture, color, and attribute data
- Dynamic font texture atlas built at runtime
- Post-processing pipeline: chromatic aberration, bloom, scanlines, vignette, and color grading
- Ghost text overlay for inline autocomplete with configurable animations

### Terminal Emulation

- Full ANSI state machine (ground, escape, CSI, OSC, DCS states)
- 16 + 256 + 24-bit true color support
- Block, underline, and beam cursors with configurable fade animations
- Scroll regions for applications like `vim`, `less`, `htop`
- Configurable scrollback buffer (up to 100,000 lines)

### Shell Integration

- Automatic shell detection (`zsh`, `bash`, `sh`)
- OSC 133 semantic prompt detection
- Real-time working directory tracking
- Remote session detection with hostname display
- Password prompt recognition (suppresses logging)

### Themes

Four built-in themes plus full custom theme support:

- **Cyberpunk Neon** (default): Electric blues and hot pinks
- **Ember Dark**: Warm oranges and deep reds
- **Veilfire Stealth**: Muted grays with cyan accents
- **Custom**: User-defined text, background, cursor, and selection colors

All themes support configurable background opacity and glass effects (frosted or clear).

---

## AI Assistant

### Provider Support

| Provider | Models | Authentication |
|----------|--------|----------------|
| **Anthropic** | Claude Opus 4, Sonnet 4, Haiku 4 | API key |
| **OpenAI** | GPT-4o, GPT-4o-mini, o1-preview, o1-mini | API key |
| **OpenRouter** | 100+ models (Claude, GPT-4, Gemini, Llama, etc.) | OAuth browser sign-in or API key |

OpenRouter includes an integrated model browser for discovering and selecting from all available models.

### AI Sidebar

The AI panel opens via `Ctrl+Space` or the animated sparkles button in the tab bar:

- Resizable sidebar (280-800pt)
- Multi-line input with auto-expansion
- Streaming responses with real-time token display
- Terminal context automatically included and redacted
- Chat/Agent mode toggle

---

## Autonomous Agent

The agent system transforms Pyromancer from a terminal with AI chat into an AI-operated terminal. In agent mode, the AI autonomously executes multi-step workflows.

### Tools

| Tool | Purpose |
|------|---------|
| `execute_command` | Run a shell command and capture output |
| `send_input` | Send raw keystrokes for interactive programs (SSH, password prompts, etc.) |
| `wait_for_output` | Wait for a regex pattern to appear in terminal output |
| `read_terminal` | Read the current visible terminal state |
| `task_complete` | Signal the task is finished |

### Execution Modes

| Mode | Description |
|------|-------------|
| **Strict Step-by-Step** | Every tool call requires individual user approval |
| **Strict Workflow** | Entire workflow presented for single approval |
| **Autonomous** | Auto-approves up to a configurable risk level; human approval above that |

### Task Budget

Every agent task operates within a budget to prevent runaway execution:

- **Command limit**: Max tool calls per task (default: 50, configurable up to 500)
- **Time limit**: Max wall-clock time (default: 10 minutes, configurable up to 60)
- Visual progress tracking in the agent status bar

### Context Window Management

- Configurable window size (4K - 200K tokens)
- Real-time token usage displayed as a pie chart
- Color-coded usage: cyan (<70%), orange (70-90%), red (>90%)
- Intelligent output truncation to keep context lean

### File System Access Rules

Restrict the agent's file system access with per-path rules:

- **Read Only** or **Read & Write** per path
- Longest-prefix matching with symlink and relative path resolution
- Subshell/pipe commands flagged for manual review
- Rules injected into the agent's system prompt so it knows its boundaries

### Task Scheduler

Schedule agent tasks to run automatically:

| Schedule | Description |
|----------|-------------|
| **Interval** | Every N minutes (5-1440) |
| **Daily** | At a specific time each day |
| **Weekdays** | Monday through Friday |
| **Weekly** | On a specific day and time |

Each task has its own budget, enable/disable toggle, run tracking, and manual trigger.

### Workflow Capture & Locking

After a task's first run, Pyromancer captures the exact command sequence and presents it for review. Once accepted, all future runs are constrained to the same patterns -- the agent can't deviate. Command templates lock the base command and subcommand while allowing argument flexibility. Deviations are blocked and logged.

### Per-Tab Agents

Each terminal tab has its own independent agent instance, enabling multiple concurrent agent workflows with visual indicators for active agents.

---

## Intelligent Autocomplete

### Suggestion Sources

1. **Git context** -- Branch names, modified files, remotes (cached with TTL)
2. **Command history** -- Previously executed commands, session and imported
3. **Learned tokens** -- Commands, paths, usernames, flags learned from terminal output
4. **Per-host directories** -- Directory listings scoped per hostname
5. **Command knowledge base** -- 1,000+ subcommands for `git`, `docker`, `kubectl`, `brew`, `npm`, `cargo`, and more
6. **PATH binaries** -- All executables in `$PATH`
7. **File path completion** -- Context-aware file and directory completion
8. **Fuzzy matching** -- Typo correction via Damerau-Levenshtein distance

### Inline Ghost Text

Completions appear as semi-transparent ghost text after the cursor. Press Right Arrow to accept. Appearance is fully configurable: color, opacity, and animation modes (wave, pulse, color cycling, rainbow).

### Encrypted Persistence

All learned autocomplete data is encrypted at rest with AES-256-GCM. The encryption key is stored in the macOS Keychain. Fresh nonce per save, owner-only file permissions.

### Shell History Import

On first launch, existing shell history is imported from zsh, bash, or fish with automatic security filtering to exclude commands containing passwords, tokens, or API keys.

---

## Tabs & Navigation

Each tab maintains independent state: terminal, agent, shell detection, hostname tracking, remote session awareness, running command indicator, and unread output badge.

| Action | Shortcut |
|--------|----------|
| Toggle AI Sidebar | `Ctrl+Space` |
| New Tab | `Cmd+T` |
| Close Tab | `Cmd+W` |
| Next Tab | `Cmd+Shift+]` |
| Previous Tab | `Cmd+Shift+[` |
| Preferences | `Cmd+,` |

---

## Macros

Record and replay terminal keystrokes with timing information:

- Start recording with `/macro record <name>`
- Playback replays with original timing (delays capped at 2 seconds)
- Assign hotkeys for quick access
- Manage from Preferences or `/macro list`

### Slash Commands

| Command | Description |
|---------|-------------|
| `/help` | Show all available commands |
| `/search <query>` | Search terminal output using AI |
| `/explain [text]` | AI explains last terminal output |
| `/fix [description]` | AI suggests fix for last error |
| `/macro record\|play\|list\|stop [name]` | Macro operations |
| `/clear` | Clear AI conversation history |
| `/model [provider] [model]` | Switch AI provider/model |

---

## Preferences

Pyromancer's preferences are organized into nine tabs:

- **General** -- Shell, startup directory, font, cursor style and animation, padding, scrollback
- **Appearance** -- Themes, colors, vignette, background opacity, glass effects
- **AI** -- Provider and model selection, API keys, context limits, permission border customization
- **Agent** -- Execution mode, auto-approve level, budgets, file access rules, memory, reflection
- **Autocomplete** -- Ghost text appearance, animation modes, data management
- **Macros** -- Record, play, and manage macros with hotkey assignments
- **Key Mappings** -- Customize all keyboard shortcuts
- **Security** -- Auto-approve threshold, user-defined secrets, redaction, audit log viewer, command allowlists
- **Debug** -- Per-subsystem logging, live log viewer with filtering

---

## Requirements

- macOS 14.0+ (Sonoma)
- Apple Silicon or Intel Mac with Metal support

---

## License

Pyromancer is proprietary software by [Veilfire](https://veilfire.io). All rights reserved.
