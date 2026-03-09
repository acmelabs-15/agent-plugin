---
title: ANALYSIS-037 Cross-Platform Hook Configuration Analysis
type: analysis
permalink: analysis/analysis-037-cross-platform-hook-configuration-analysis-1
tags:
- hooks
- cross-platform
- configuration
- events
- research
---

# ANALYSIS-037 Cross-Platform Hook Configuration Analysis

## 1. Objective and Scope

**Objective**: Map the hook/event systems across 7 AI coding assistant platforms to identify universal patterns, platform-specific variations, and feasibility of a canonical hook definition format.

**Scope**: Claude Code, Cursor, Windsurf, GitHub Copilot, OpenAI Codex CLI, Gemini CLI, Amazon Q Developer CLI. Covers hook events, configuration format, execution model, and cross-platform normalization.

## 2. Context

AI coding assistants have converged on hook systems as the mechanism for deterministic control over agent behavior. Hooks execute shell commands at lifecycle points, replacing non-deterministic LLM-based rules with predictable program execution. This analysis supports the agent-plugin spec by mapping what a universal hook definition must accommodate.

## 3. Approach

**Methodology**: Official documentation review, GitHub repository analysis, community guides, and changelog analysis for each platform.

**Tools Used**: WebSearch, WebFetch against official docs sites.

**Limitations**: Amazon Q Developer CLI is being sunset in favor of Kiro CLI (closed-source). Codex CLI has minimal hook support (notification only). Some platforms are in beta for hooks (Cursor). Exact internal execution details vary.

## 4. Platform-by-Platform Analysis

### 4.1 Claude Code

**Source**: https://code.claude.com/docs/en/hooks, https://code.claude.com/docs/en/hooks-guide

**Config Location**: `~/.claude/settings.json` (user), `.claude/settings.json` (project), `.claude/settings.local.json` (local), managed policy settings (org), plugin `hooks/hooks.json`, skill/agent frontmatter (YAML)

**Config Format**: JSON

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "regex_pattern",
        "hooks": [
          {
            "type": "command",
            "command": "shell-command-here",
            "timeout": 600,
            "async": false,
            "statusMessage": "Custom spinner text",
            "once": false
          }
        ]
      }
    ]
  }
}
```

**Hook Types**: `command`, `http`, `prompt`, `agent`

**18 Event Types**:

| Event | Matcher Target | Can Block? | Sync? |
|---|---|---|---|
| `SessionStart` | source: startup/resume/clear/compact | No | Yes |
| `UserPromptSubmit` | none | Yes (exit 2 or decision:block) | Yes |
| `PreToolUse` | tool name (regex) | Yes (exit 2 or permissionDecision:deny) | Yes |
| `PermissionRequest` | tool name (regex) | Yes (decision.behavior:deny) | Yes |
| `PostToolUse` | tool name (regex) | No (feedback only) | Yes |
| `PostToolUseFailure` | tool name (regex) | No | Yes |
| `Notification` | notification type | No | Yes |
| `SubagentStart` | agent type | No | Yes |
| `SubagentStop` | agent type | Yes (exit 2) | Yes |
| `Stop` | none | Yes (exit 2 or decision:block) | Yes |
| `TeammateIdle` | none | Yes (exit 2) | Yes |
| `TaskCompleted` | none | Yes (exit 2) | Yes |
| `InstructionsLoaded` | none | No | Async |
| `ConfigChange` | config source | Yes (exit 2) | Yes |
| `WorktreeCreate` | none | Yes (non-zero exit) | Yes |
| `WorktreeRemove` | none | No | Yes |
| `PreCompact` | trigger: manual/auto | No | Yes |
| `SessionEnd` | reason | No | Yes |

**Exit Codes**: 0 = allow/ok, 2 = block (blocking events only), other = non-blocking error

**Common Input Fields (stdin JSON)**:
- `session_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name`
- Agent context: `agent_id`, `agent_type` (when in subagent)

**Environment Variables**: `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `CLAUDE_ENV_FILE` (SessionStart only), `CLAUDE_CODE_REMOTE`

**Output**: stdout JSON with universal fields (`continue`, `stopReason`, `suppressOutput`, `systemMessage`) plus event-specific `hookSpecificOutput` with `hookEventName`

**Unique Features**: HTTP hooks (POST to URL), prompt hooks (LLM evaluation), agent hooks (multi-turn subagent verification), plugin bundled hooks, skill/agent frontmatter hooks, `updatedInput` to modify tool args, `CLAUDE_ENV_FILE` for env persistence, `once` field for single-fire hooks

---

### 4.2 Cursor

**Source**: https://cursor.com/docs/hooks, https://blog.gitbutler.com/cursor-hooks-deep-dive

**Config Location**: `.cursor/hooks.json` (project), `~/.cursor/hooks.json` (user), `/Library/Application Support/Cursor/hooks.json` or `/etc/cursor/hooks.json` (enterprise/system), cloud dashboard (enterprise)

**Config Format**: JSON with `version: 1`

```json
{
  "version": 1,
  "hooks": {
    "eventName": [
      {
        "command": "path/to/script",
        "type": "command",
        "timeout": 30,
        "loop_limit": 5,
        "failClosed": false,
        "matcher": "regex_or_string"
      }
    ]
  }
}
```

**Hook Types**: `command`, `prompt`

**19+ Event Types** (Agent + Tab):

Agent Events:
| Event | Description | Can Block? |
|---|---|---|
| `sessionStart` | Session begins | No (context injection) |
| `sessionEnd` | Session ends | No (fire-and-forget) |
| `preToolUse` | Before tool execution | Yes |
| `postToolUse` | After tool success | No |
| `postToolUseFailure` | After tool failure | No |
| `subagentStart` | Subagent spawned | Yes |
| `subagentStop` | Subagent finished | Yes (followup_message) |
| `beforeShellExecution` | Before shell command | Yes |
| `afterShellExecution` | After shell command | No |
| `beforeMCPExecution` | Before MCP tool | Yes |
| `afterMCPExecution` | After MCP tool | No |
| `beforeReadFile` | Before file read to LLM | Yes |
| `afterFileEdit` | After file edit | No |
| `beforeSubmitPrompt` | Before prompt to model | Yes |
| `preCompact` | Before compaction | No |
| `stop` | Agent loop done | Yes (followup_message) |
| `afterAgentResponse` | After assistant message | No |
| `afterAgentThought` | After thinking block | No |

Tab Events:
| Event | Description |
|---|---|
| `beforeTabFileRead` | Before tab reads file |
| `afterTabFileEdit` | After tab edits file |

**Exit Codes**: 0 = success, 2 = deny/block, other = fail-open (unless `failClosed: true`)

**Common Input Fields (stdin JSON)**:
- `conversation_id`, `generation_id`, `model`, `hook_event_name`, `cursor_version`, `workspace_roots`, `user_email`, `transcript_path`

**Environment Variables**: `CURSOR_PROJECT_DIR`, `CURSOR_VERSION`, `CURSOR_USER_EMAIL`, `CURSOR_TRANSCRIPT_PATH`, `CURSOR_CODE_REMOTE`, `CLAUDE_PROJECT_DIR` (compatibility alias)

**Output**: JSON with event-specific fields: `permission` (allow/deny/ask), `user_message`, `agent_message`, `updated_input`, `followup_message`, `env` (sessionStart)

**Unique Features**: `failClosed` field (block on hook crash/timeout), `loop_limit` for stop/subagentStop auto-followups (default 5, null=unlimited), Tab events for inline completions, separate shell/MCP/file-specific events, `afterAgentResponse`/`afterAgentThought` for output tracking, `CLAUDE_PROJECT_DIR` compatibility alias, enterprise cloud dashboard distribution

---

### 4.3 Windsurf (Cascade)

**Source**: https://docs.windsurf.com/windsurf/cascade/hooks

**Config Location**: `.windsurf/hooks.json` (workspace), `~/.codeium/windsurf/hooks.json` (user/IDE), `~/.codeium/hooks.json` (JetBrains), `/Library/Application Support/Windsurf/hooks.json` or `/etc/windsurf/hooks.json` (system)

**Config Format**: JSON (no version field)

```json
{
  "hooks": {
    "hook_event_name": [
      {
        "command": "executable with arguments",
        "show_output": true,
        "working_directory": "/optional/path"
      }
    ]
  }
}
```

**Hook Types**: `command` only

**12 Event Types**:

| Event | Description | Can Block? |
|---|---|---|
| `pre_read_code` | Before file read | Yes (exit 2) |
| `post_read_code` | After file read | No |
| `pre_write_code` | Before file write | Yes (exit 2) |
| `post_write_code` | After file write | No |
| `pre_run_command` | Before terminal command | Yes (exit 2) |
| `post_run_command` | After terminal command | No |
| `pre_mcp_tool_use` | Before MCP tool | Yes (exit 2) |
| `post_mcp_tool_use` | After MCP tool | No |
| `pre_user_prompt` | Before prompt processing | Yes (exit 2) |
| `post_cascade_response` | After agent response (markdown) | No |
| `post_cascade_response_with_transcript` | After response (full JSONL) | No |
| `post_setup_worktree` | After worktree creation | No |

**Exit Codes**: 0 = success, 2 = block (pre-hooks only), other = non-blocking error

**Common Input Fields (stdin JSON)**:
- `agent_action_name`, `trajectory_id`, `execution_id`, `timestamp`, `tool_info` (contains event-specific data)

**Environment Variables**: `ROOT_WORKSPACE_PATH` (post_setup_worktree only)

**Output**: stderr displayed when `show_output: true`. No structured JSON response protocol documented.

**Unique Features**: `show_output` toggle, `working_directory` field, enterprise MDM/dashboard distribution, separate transcript hook (`post_cascade_response_with_transcript` with JSONL output), granular file read/write separation, minimal field set (command, show_output, working_directory)

---

### 4.4 GitHub Copilot

**Source**: https://docs.github.com/en/copilot/reference/hooks-configuration, https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/use-hooks

**Config Location**: `.github/hooks/*.json` (repository, must be on default branch for coding agent), current working directory (CLI)

**Config Format**: JSON with `version: 1`

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [...],
    "sessionEnd": [...],
    "userPromptSubmitted": [...],
    "preToolUse": [...],
    "postToolUse": [...],
    "errorOccurred": [...]
  }
}
```

**Hook Definition**:
```json
{
  "type": "command",
  "bash": "./path/to/script.sh",
  "powershell": "./path/to/script.ps1",
  "cwd": "working_directory",
  "timeoutSec": 30,
  "env": { "KEY": "VALUE" },
  "comment": "description"
}
```

**Hook Types**: `command` only

**6 Event Types**:

| Event | Description | Can Block? |
|---|---|---|
| `sessionStart` | Agent session begins | No |
| `sessionEnd` | Agent session ends | No |
| `userPromptSubmitted` | User submits prompt | No |
| `preToolUse` | Before tool execution | Yes (permissionDecision: deny) |
| `postToolUse` | After tool execution | No |
| `errorOccurred` | Error during execution | No |

**Exit Codes**: Not explicitly documented. Scripts output JSON to stdout.

**Input/Output**: stdin JSON, stdout single-line JSON. `jq -c` (bash) or `ConvertTo-Json -Compress` (PowerShell) for output.

**Environment Variables**: `env` field in hook definition for custom env vars. No platform-specific env vars documented.

**Output**: preToolUse returns `{ "permissionDecision": "deny|allow|ask", "permissionDecisionReason": "..." }`. Currently only `deny` is processed; `ask` is not yet functional.

**Unique Features**: Separate `bash`/`powershell` command fields for cross-platform scripts, `comment` field for documentation, `env` field for per-hook environment variables, repository-based hooks (`.github/hooks/`), multiple hooks per event execute sequentially in array order

---

### 4.5 OpenAI Codex CLI

**Source**: https://developers.openai.com/codex/config-advanced/

**Config Location**: `~/.codex/config.toml` (TOML format)

**Config Format**: TOML

```toml
notify = ["python3", "/path/to/notify.py"]
```

**Hook Types**: Notification hook only (external program invocation)

**1 Event Type**:

| Event | Description | Can Block? |
|---|---|---|
| `agent-turn-complete` | Agent finishes a turn | No |

**Exit Codes**: Not applicable (fire-and-forget notification)

**Input**: Single JSON argument passed as command-line parameter (NOT stdin). Script parses `sys.argv[1]`.

**JSON Payload Fields**:
- `type` (event identifier), `thread-id`, `turn-id`, `cwd`, `input-messages`, `last-assistant-message`

**Environment Variables**: None documented for hooks.

**Output**: None processed. Fire-and-forget.

**Unique Features**: TOML configuration (only platform using TOML), JSON passed as CLI argument (not stdin), single event type only, separate `tui.notifications` for built-in terminal notifications. Extremely minimal hook system. Documentation states "currently only `agent-turn-complete`" suggesting future expansion planned.

---

### 4.6 Gemini CLI

**Source**: https://geminicli.com/docs/hooks/, https://geminicli.com/docs/hooks/reference/

**Config Location**: `.gemini/settings.json` (project), `~/.gemini/settings.json` (user), `/etc/gemini-cli/settings.json` (system), installed extensions

**Config Format**: JSON

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "regex_pattern",
        "sequential": true,
        "hooks": [
          {
            "type": "command",
            "command": "script.sh",
            "name": "friendly_name",
            "timeout": 60000,
            "description": "purpose"
          }
        ]
      }
    ]
  }
}
```

**Hook Types**: `command` only

**11 Event Types**:

| Event | Matcher Target | Can Block? |
|---|---|---|
| `SessionStart` | source: startup/resume/clear | No (advisory) |
| `SessionEnd` | reason: exit/clear/logout/other | No (best-effort) |
| `BeforeAgent` | none | Yes (decision:deny or exit 2) |
| `AfterAgent` | none | Yes (decision:deny triggers retry) |
| `BeforeModel` | none | Yes (exit 2 aborts turn) |
| `AfterModel` | none | Yes (per-stream-chunk replacement) |
| `BeforeToolSelection` | none | No (filter tools only) |
| `BeforeTool` | tool name (regex) | Yes (exit 2 blocks) |
| `AfterTool` | tool name (regex) | Yes (exit 2 hides result) |
| `PreCompress` | trigger: auto/manual | No (advisory) |
| `Notification` | notification_type | No (observability) |

**Exit Codes**: 0 = success (stdout parsed as JSON), 2 = system block, other = warning (non-fatal)

**Common Input Fields (stdin JSON)**:
- `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `timestamp` (ISO 8601)

**Environment Variables**: `GEMINI_PROJECT_DIR`, `GEMINI_SESSION_ID`, `GEMINI_CWD`, `CLAUDE_PROJECT_DIR` (compatibility alias)

**Output**: JSON with `systemMessage`, `suppressOutput`, `continue`, `stopReason`, `decision`, `reason`, plus event-specific `hookSpecificOutput` with `tool_input` (merge), `additionalContext`, `tailToolCallRequest`, `clearContext`

**Unique Features**: `BeforeModel` hook (intercept/mock LLM requests before they are sent), `AfterModel` hook (per-stream-chunk replacement), `BeforeToolSelection` hook (whitelist/filter tools before planner selects), `sequential` field on matcher group, `name` and `description` fields on handler, `CLAUDE_PROJECT_DIR` compatibility alias, `tailToolCallRequest` for chaining tools, `clearContext` for wiping LLM memory, timeout in milliseconds (60000ms default), extension-bundled hooks, `/hooks panel` management command

---

### 4.7 Amazon Q Developer CLI

**Source**: https://aws.github.io/amazon-q-developer-cli/agent-format.html, AWS blog posts

**Config Location**: Agent JSON files (custom agent definitions). Path not standardized like other platforms.

**Config Format**: JSON (within agent definition)

```json
{
  "hooks": {
    "agentSpawn": [
      {
        "command": "git branch --show-current"
      }
    ],
    "userPromptSubmit": [
      {
        "command": "git status --porcelain",
        "timeout_ms": 5000,
        "cache_ttl_seconds": 30
      }
    ],
    "preToolUse": [
      {
        "command": "validate-tool.sh",
        "matcher": "fs_write"
      }
    ]
  }
}
```

**Hook Types**: `command` only

**5 Event Types**:

| Event | Description | Can Block? |
|---|---|---|
| `agentSpawn` | Agent initialization | No (context injection, persists) |
| `userPromptSubmit` | User message submitted | No (context injection, per-prompt) |
| `preToolUse` | Before tool execution | Yes |
| `postToolUse` | After tool execution | No |
| `stop` | Assistant finishes responding | Yes |

**Exit Codes**: Not explicitly documented.

**Fields**: `command` (required), `timeout_ms` (default 30000), `max_output_size` (default 10240 bytes), `cache_ttl_seconds`, `matcher` (optional, for tool hooks)

**Output**: agentSpawn output persists in session context. userPromptSubmit output injected per-prompt only.

**Unique Features**: `timeout_ms` in milliseconds (unique field name), `max_output_size` for output truncation, `cache_ttl_seconds` for caching hook results, output injection model differs between agentSpawn (persistent) and userPromptSubmit (per-prompt). NOTE: Amazon Q CLI is being sunset; Kiro CLI is the successor (closed-source).

---

## 5. Cross-Platform Comparison Matrix

### 5.1 Hook Event Normalization

| Canonical Event | Claude Code | Cursor | Windsurf | Copilot | Codex CLI | Gemini CLI | Amazon Q |
|---|---|---|---|---|---|---|---|
| **Session Start** | `SessionStart` | `sessionStart` | -- | `sessionStart` | -- | `SessionStart` | `agentSpawn` |
| **Session End** | `SessionEnd` | `sessionEnd` | -- | `sessionEnd` | -- | `SessionEnd` | -- |
| **User Prompt** | `UserPromptSubmit` | `beforeSubmitPrompt` | `pre_user_prompt` | `userPromptSubmitted` | -- | `BeforeAgent` | `userPromptSubmit` |
| **Pre Tool Use** | `PreToolUse` | `preToolUse` | (split: see below) | `preToolUse` | -- | `BeforeTool` | `preToolUse` |
| **Post Tool Use** | `PostToolUse` | `postToolUse` | (split: see below) | `postToolUse` | -- | `AfterTool` | `postToolUse` |
| **Post Tool Failure** | `PostToolUseFailure` | `postToolUseFailure` | -- | `errorOccurred` | -- | -- | -- |
| **Pre Shell** | (via PreToolUse Bash) | `beforeShellExecution` | `pre_run_command` | (via preToolUse) | -- | (via BeforeTool) | -- |
| **Post Shell** | (via PostToolUse Bash) | `afterShellExecution` | `post_run_command` | (via postToolUse) | -- | (via AfterTool) | -- |
| **Pre File Read** | (via PreToolUse Read) | `beforeReadFile` | `pre_read_code` | -- | -- | (via BeforeTool) | -- |
| **Post File Edit** | (via PostToolUse Edit) | `afterFileEdit` | `post_write_code` | -- | -- | (via AfterTool) | -- |
| **Pre MCP Tool** | (via PreToolUse mcp__*) | `beforeMCPExecution` | `pre_mcp_tool_use` | -- | -- | (via BeforeTool mcp__*) | -- |
| **Post MCP Tool** | (via PostToolUse mcp__*) | `afterMCPExecution` | `post_mcp_tool_use` | -- | -- | (via AfterTool mcp__*) | -- |
| **Stop / Complete** | `Stop` | `stop` | -- | -- | `agent-turn-complete` | `AfterAgent` | `stop` |
| **Notification** | `Notification` | -- | -- | -- | -- | `Notification` | -- |
| **Subagent Start** | `SubagentStart` | `subagentStart` | -- | -- | -- | -- | -- |
| **Subagent Stop** | `SubagentStop` | `subagentStop` | -- | -- | -- | -- | -- |
| **Pre Compact** | `PreCompact` | `preCompact` | -- | -- | -- | `PreCompress` | -- |
| **Config Change** | `ConfigChange` | -- | -- | -- | -- | -- | -- |
| **Worktree** | `WorktreeCreate`/`WorktreeRemove` | -- | `post_setup_worktree` | -- | -- | -- | -- |
| **Agent Response** | -- | `afterAgentResponse` | `post_cascade_response` | -- | -- | `AfterAgent` | -- |
| **Agent Thought** | -- | `afterAgentThought` | -- | -- | -- | -- | -- |
| **Permission** | `PermissionRequest` | -- | -- | -- | -- | -- | -- |
| **Teammate Idle** | `TeammateIdle` | -- | -- | -- | -- | -- | -- |
| **Task Completed** | `TaskCompleted` | -- | -- | -- | -- | -- | -- |
| **Instructions** | `InstructionsLoaded` | -- | -- | -- | -- | -- | -- |
| **Before Model** | -- | -- | -- | -- | -- | `BeforeModel` | -- |
| **After Model** | -- | -- | -- | -- | -- | `AfterModel` | -- |
| **Tool Selection** | -- | -- | -- | -- | -- | `BeforeToolSelection` | -- |
| **Pre File Write** | (via PreToolUse Write) | -- | `pre_write_code` | -- | -- | (via BeforeTool) | -- |
| **Transcript** | -- | -- | `post_cascade_response_with_transcript` | -- | -- | -- | -- |

### 5.2 Event Naming Convention

| Platform | Convention | Example |
|---|---|---|
| Claude Code | PascalCase | `PreToolUse`, `SessionStart` |
| Cursor | camelCase | `preToolUse`, `sessionStart` |
| Windsurf | snake_case | `pre_write_code`, `post_run_command` |
| GitHub Copilot | camelCase | `preToolUse`, `sessionStart` |
| Codex CLI | kebab-case | `agent-turn-complete` |
| Gemini CLI | PascalCase | `BeforeTool`, `SessionStart` |
| Amazon Q | camelCase | `agentSpawn`, `userPromptSubmit` |

### 5.3 Configuration Structure Comparison

| Aspect | Claude Code | Cursor | Windsurf | Copilot | Codex CLI | Gemini CLI | Amazon Q |
|---|---|---|---|---|---|---|---|
| **Format** | JSON | JSON | JSON | JSON | TOML | JSON | JSON |
| **Version field** | No | Yes (1) | No | Yes (1) | N/A | No | No |
| **Top-level key** | `hooks` | `hooks` | `hooks` | `hooks` | `notify` | `hooks` | `hooks` |
| **Event grouping** | Event > MatcherGroup[] > Handler[] | Event > Handler[] | Event > Handler[] | Event > Handler[] | flat array | Event > MatcherGroup[] > Handler[] | Event > Handler[] |
| **Matcher location** | On matcher group | On handler | None | None | None | On matcher group | On handler |
| **Handler types** | command/http/prompt/agent | command/prompt | command | command | external program | command | command |
| **Timeout unit** | seconds | seconds | -- | seconds | -- | milliseconds | milliseconds |
| **Timeout field** | `timeout` | `timeout` | -- | `timeoutSec` | -- | `timeout` | `timeout_ms` |
| **Timeout default** | 600s (cmd), 30s (prompt), 60s (agent) | platform default | -- | 30s | -- | 60000ms (60s) | 30000ms (30s) |

### 5.4 Input/Output Comparison

| Aspect | Claude Code | Cursor | Windsurf | Copilot | Codex CLI | Gemini CLI | Amazon Q |
|---|---|---|---|---|---|---|---|
| **Input delivery** | stdin JSON | stdin JSON | stdin JSON | stdin JSON | CLI arg (sys.argv) | stdin JSON | stdout capture |
| **Output method** | stdout JSON + exit code | stdout JSON + exit code | stderr + exit code | stdout JSON | none | stdout JSON + exit code | stdout text |
| **Exit 0** | allow | allow | allow | success | N/A | allow | N/A |
| **Exit 2** | block | deny | block | N/A | N/A | system block | N/A |
| **Other exit** | non-blocking error | fail-open (or failClosed) | non-blocking error | N/A | N/A | warning | N/A |
| **JSON output** | hookSpecificOutput + decision | permission + messages | none | permissionDecision | none | hookSpecificOutput + decision | none |
| **Stdin no-output rule** | No (mixed ok) | No | No | No | N/A | Yes (MUST not print non-JSON) | N/A |

### 5.5 Environment Variables

| Variable Pattern | Claude Code | Cursor | Windsurf | Copilot | Gemini CLI |
|---|---|---|---|---|---|
| **Project dir** | `CLAUDE_PROJECT_DIR` | `CURSOR_PROJECT_DIR` | -- | -- | `GEMINI_PROJECT_DIR` |
| **Claude compat** | native | `CLAUDE_PROJECT_DIR` | -- | -- | `CLAUDE_PROJECT_DIR` |
| **Session ID** | in stdin JSON | in stdin JSON | in stdin JSON | -- | `GEMINI_SESSION_ID` + stdin |
| **Version** | -- | `CURSOR_VERSION` | -- | -- | -- |
| **User email** | -- | `CURSOR_USER_EMAIL` | -- | -- | -- |
| **Transcript** | in stdin JSON | `CURSOR_TRANSCRIPT_PATH` | -- | -- | in stdin JSON |
| **Remote flag** | `CLAUDE_CODE_REMOTE` | `CURSOR_CODE_REMOTE` | -- | -- | -- |
| **Env file** | `CLAUDE_ENV_FILE` | -- | -- | -- | -- |

---

## 6. Universal Events (Present in 4+ Platforms)

These events exist in some form across at least 4 of the 7 platforms:

1. **Pre Tool Use** (6/7): Claude Code, Cursor, Windsurf, Copilot, Gemini CLI, Amazon Q. Universal. Only Codex CLI lacks it.
2. **Post Tool Use** (6/7): Same platforms as Pre Tool Use.
3. **Session Start** (5/7): Claude Code, Cursor, Copilot, Gemini CLI, Amazon Q (as agentSpawn).
4. **User Prompt Submit** (6/7): Claude Code, Cursor, Windsurf, Copilot, Gemini CLI, Amazon Q. Named differently on each platform.
5. **Stop / Agent Complete** (5/7): Claude Code, Cursor, Codex CLI, Gemini CLI, Amazon Q. Core agentic loop event.
6. **Session End** (4/7): Claude Code, Cursor, Copilot, Gemini CLI.

## 7. Common Denominator Configuration

The minimum viable cross-platform hook definition uses these shared attributes:

```json
{
  "event": "pre_tool_use",
  "command": "path/to/script.sh",
  "matcher": "optional_tool_name_pattern",
  "timeout_ms": 30000
}
```

**Shared fields across all platforms with hooks**:
- `command` (string): Every platform uses a command/script path
- Event name (string): Every platform has named events
- Matcher/pattern (string, optional): 5/7 platforms support tool name filtering
- Timeout (number, optional): 5/7 platforms support timeouts

**NOT shared**:
- Exit code semantics (exit 2 = block is Claude Code/Cursor/Windsurf/Gemini only)
- JSON output format (varies significantly)
- Input delivery method (stdin JSON vs CLI arg vs stdout capture)
- Hook types beyond `command`
- Structured decision control
- Environment variables

## 8. Canonical Hook Definition Proposal

A universal hook definition that can be transformed into each platform's format:

```json
{
  "name": "block-dangerous-commands",
  "description": "Prevents destructive shell commands",
  "event": "pre_tool_use",
  "matcher": {
    "tool_name": "Bash|Shell|shell"
  },
  "handler": {
    "type": "command",
    "command": "./hooks/block-dangerous.sh",
    "timeout_ms": 30000
  },
  "behavior": {
    "can_block": true,
    "blocking_exit_code": 2,
    "input_format": "stdin_json",
    "output_format": "stdout_json"
  },
  "platform_overrides": {
    "claude_code": {
      "event": "PreToolUse",
      "matcher": "Bash"
    },
    "cursor": {
      "event": "beforeShellExecution",
      "matcher": null
    },
    "windsurf": {
      "event": "pre_run_command"
    },
    "copilot": {
      "event": "preToolUse"
    },
    "gemini": {
      "event": "BeforeTool",
      "matcher": "shell_.*"
    },
    "amazon_q": {
      "event": "preToolUse",
      "matcher": "execute_bash"
    }
  }
}
```

### Transformation Feasibility Assessment

| Transformation | Feasibility | Effort | Notes |
|---|---|---|---|
| Canonical to Claude Code | High | Low | Direct mapping, richest feature set |
| Canonical to Cursor | High | Low | Close alignment, camelCase conversion needed |
| Canonical to Windsurf | Medium | Medium | snake_case, split events (read/write/command/mcp), no JSON output protocol |
| Canonical to Copilot | Medium | Medium | Separate bash/powershell fields, .github/hooks/ location |
| Canonical to Gemini CLI | High | Low | PascalCase, nearly identical structure to Claude Code |
| Canonical to Codex CLI | Low | High | Only 1 event type, TOML format, CLI arg input |
| Canonical to Amazon Q | Medium | Medium | Limited events, being sunset for Kiro CLI |

## 9. Key Architectural Differences

### 9.1 Event Granularity Spectrum

```
Windsurf (most granular)                                     Codex CLI (least granular)
   12 events split by                                            1 event only
   file/shell/mcp/prompt
         |                                                           |
   Cursor (19+ events)    Claude Code (18)    Gemini (11)    Copilot (6)    Amazon Q (5)
   adds tab events,       unified tool        adds model     basic          basic
   agent thought/response events + matcher    intercept      lifecycle      lifecycle
```

### 9.2 Tool Event Strategy

Two approaches exist:
1. **Unified tool events + matchers** (Claude Code, Cursor partially, Copilot, Gemini, Amazon Q): Single `preToolUse`/`postToolUse` with matcher to filter by tool name
2. **Split tool events** (Windsurf, Cursor partially): Separate events per tool category (`pre_read_code`, `pre_write_code`, `pre_run_command`, `pre_mcp_tool_use`)

Cursor uses a hybrid: unified `preToolUse`/`postToolUse` plus specific `beforeShellExecution`, `beforeReadFile`, `afterFileEdit`, `beforeMCPExecution`.

### 9.3 Decision Control Patterns

| Pattern | Platforms | Mechanism |
|---|---|---|
| Exit code only | Windsurf | exit 0/2, stderr message |
| Exit code + JSON | Claude Code, Cursor, Gemini | exit 0 with JSON for fine control, exit 2 for simple block |
| JSON only | Copilot | stdout JSON with permissionDecision |
| Fire-and-forget | Codex CLI | No decision control |
| Mixed | Amazon Q | Output injected into context, some blocking |

## 10. Observations

- [fact] 6 of 7 platforms support pre-tool-use and post-tool-use hooks, making these the most universal hook events #cross-platform #hooks
- [fact] All 7 platforms use JSON for hook configuration except Codex CLI which uses TOML #configuration
- [fact] Exit code 2 = block is a convention shared by Claude Code, Cursor, Windsurf, and Gemini CLI (4/7 platforms) #convention
- [fact] Cursor provides a `CLAUDE_PROJECT_DIR` compatibility alias; Gemini CLI does the same, indicating Claude Code as the de facto standard #compatibility
- [fact] Claude Code has the richest hook system with 18 events and 4 hook types (command, http, prompt, agent) #architecture
- [fact] Windsurf uses the most granular event naming with separate read/write/command/mcp events #design-choice
- [decision] A canonical hook format is feasible for 5 of 7 platforms; Codex CLI (1 event, TOML) is the outlier #feasibility
- [insight] The industry is converging on stdin JSON input, stdout JSON output, and exit code 2 for blocking as the hook communication protocol #convergence
- [risk] Amazon Q CLI is being sunset in favor of closed-source Kiro CLI, reducing the addressable platform count to 6 #platform-risk
- [technique] Both Cursor and Gemini CLI include `CLAUDE_PROJECT_DIR` as a compatibility alias, suggesting hook scripts can be written once for Claude Code and reused #portability

## Relations

- implements [[REQ-002 Cross-Platform Hook Event Normalization]]
- relates_to [[TASK-007 Cross-Platform Hook Configuration]]
- relates_to [[TASK-004 Implement Shared Hook Script Infrastructure]]
- extends [[ANALYSIS-057 Cursor AI Capabilities Research]]