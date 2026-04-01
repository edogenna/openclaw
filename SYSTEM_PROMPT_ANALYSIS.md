# OpenClaw System Prompt Construction Analysis

## 1. Core Agent Loop and Runtime

The agent loop lives in the embedded Pi runner:

- **Agent orchestrator**: `src/agents/pi-embedded-runner/run.ts` — `runEmbeddedPiAgent()` coordinates the full agent run
- **Attempt execution**: `src/agents/pi-embedded-runner/run/attempt.ts` — `runEmbeddedAttempt()` constructs the system prompt (lines ~630–838), creates the override, and applies it to the LLM session
- **Reply runner**: `src/auto-reply/reply/agent-runner.ts` — higher-level reply orchestration calling `runAgentTurnWithFallback()`

The call chain is:
```
agent-runner → runEmbeddedPiAgent → runEmbeddedAttempt
  → buildEmbeddedSystemPrompt()      (line ~630)
  → createSystemPromptOverride()      (line ~686)
  → applySystemPromptOverrideToSession()  (line ~838)
```

## 2. System Prompt Construction — Entry Points

### Primary builder
**File**: `src/agents/system-prompt.ts`

**Function**: `buildAgentSystemPrompt(params)` (line 184–676)

This is the **single function** that assembles the entire system prompt string. It takes ~30 parameters and returns a newline-joined string of all sections.

### Embedded wrapper
**File**: `src/agents/pi-embedded-runner/system-prompt.ts`

**Function**: `buildEmbeddedSystemPrompt(params)` (line 11–85)

Thin wrapper that converts `AgentTool[]` objects into tool names/summaries, then delegates to `buildAgentSystemPrompt()`.

### Override application
**Function**: `applySystemPromptOverrideToSession(session, override)` (line 94–106)

Sets the final prompt string on the LLM session object via `session.agent.setSystemPrompt(prompt)` and caches it as `_baseSystemPrompt`.

## 3. Base Identity Prompt

The base identity line is a single sentence, used across all prompt modes:

```
You are a personal assistant running inside OpenClaw.
```

**Location**: `src/agents/system-prompt.ts:413` (for `none` mode) and line 417 (for `full`/`minimal` modes).

For `promptMode === "none"`, this is the **entire** system prompt. For other modes, it is the first line followed by all assembled sections.

## 4. System Prompt Sections (Full Mode)

The prompt is built as an array of lines joined by `\n`. Sections appear in this order:

| Section | Lines | Condition |
|---------|-------|-----------|
| Base identity | 417 | Always |
| **## Tooling** | 419–454 | Always |
| **## Tool Call Style** | 455–467 | Always |
| **## Safety** | 388–394 | Always |
| **## OpenClaw CLI Quick Reference** | 469–477 | Always |
| **## Skills (mandatory)** | 21–37 | When `skillsPrompt` is non-empty |
| **Memory section** | 39–51 | Full mode only, via `buildMemoryPromptSection()` |
| **## OpenClaw Self-Update** | 481–490 | Full mode only, when `gateway` tool available |
| **## Model Aliases** | 494–503 | Full mode only, when aliases exist |
| **## Workspace** | 507–510 | Always |
| **## Documentation** | 158–174 | Full mode only, when `docsPath` exists |
| **## Sandbox** | 513–559 | When sandbox is enabled |
| **## Authorized Senders** | 53–58 | Full mode only, when owner numbers exist |
| **## Current Date & Time** | 84–89 | When timezone is known |
| **## Workspace Files (injected)** | 565–567 | Always |
| **## Reply Tags** | 91–105 | Full mode only |
| **## Messaging** | 107–145 | Full mode only |
| **## Voice (TTS)** | 147–156 | Full mode only, when TTS hint exists |
| **## Group Chat Context** | 580–585 | When `extraSystemPrompt` exists |
| **## Reactions** | 586–608 | When `reactionGuidance` exists |
| **## Reasoning Format** | 609–611 | When `reasoningTagHint` is true |
| **# Project Context** | 617–636 | When context files exist |
| **## Silent Replies** | 639–654 | Full mode only |
| **## Heartbeats** | 656–667 | Full mode only, when heartbeat prompt exists |
| **## Runtime** | 669–673 | Always |

## 5. Skill Injection Logic

### How SKILL.md files are formatted

**File**: `src/agents/skills/workspace.ts`

**Two formats exist:**

#### Full format (from `@mariozechner/pi-coding-agent` package)
Used when the character budget allows. XML structure per the docs:
```xml
<available_skills>
  <skill>
    <name>skill-name</name>
    <description>What the skill does</description>
    <location>~/.openclaw/skills/skill-name/SKILL.md</location>
  </skill>
</available_skills>
```

#### Compact format (`formatSkillsCompact`, line 544–562)
Used when full format exceeds character budget. Omits descriptions:
```xml
<available_skills>
  <skill>
    <name>skill-name</name>
    <location>~/.openclaw/skills/skill-name/SKILL.md</location>
  </skill>
</available_skills>
```

### Tiered budget logic (`applySkillsPromptLimits`, line 567–613)
1. Try full format within `maxSkillsPromptChars` (default 30,000)
2. If full exceeds budget → switch to compact
3. If compact still exceeds → binary search for the largest prefix that fits
4. Truncation warnings are prepended by the caller

### Injection into system prompt (`buildSkillsSection`, line 21–37)
```
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.
- When a skill drives external API writes, assume rate limits: ...
<available_skills>...</available_skills>
```

### Path compaction (`compactSkillPaths`, line 40–48)
Home directory prefixes are replaced with `~` to save ~5-6 tokens per skill path.

### Limits
- `DEFAULT_MAX_SKILLS_IN_PROMPT = 150`
- `DEFAULT_MAX_SKILLS_PROMPT_CHARS = 30_000`
- `DEFAULT_MAX_SKILL_FILE_BYTES = 256_000`

## 6. Bootstrap File Injection (SOUL.md, AGENTS.md, IDENTITY.md, etc.)

### Bootstrap filenames (constants in `src/agents/workspace.ts:25–33`)
```
AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md, USER.md,
HEARTBEAT.md, BOOTSTRAP.md, MEMORY.md (or memory.md fallback)
```

### Loading pipeline
1. **Load**: `loadWorkspaceBootstrapFiles()` (`src/agents/workspace.ts`) reads files from the workspace directory using boundary-safe `openBoundaryFile()` with a 2MB max
2. **Cache**: Files cached by inode/dev/size/mtime identity to avoid stale reads
3. **Filter**: `filterBootstrapFilesForSession()` filters per session type (subagents only get AGENTS.md + TOOLS.md)
4. **Context mode filter**: `applyContextModeFilter()` (`src/agents/bootstrap-files.ts:47–62`)
   - `"full"` mode → all files
   - `"lightweight"` + `"heartbeat"` → only HEARTBEAT.md
   - `"lightweight"` + `"cron"/"default"` → empty (no bootstrap)
5. **Plugin hooks**: `applyBootstrapHookOverrides()` (`src/agents/bootstrap-hooks.ts`) runs `agent:bootstrap` internal hooks
6. **Build context**: `buildBootstrapContextFiles()` (`src/agents/pi-embedded-helpers/bootstrap.ts:198+`) trims and formats

### Trimming strategy (`trimBootstrapContent`, line 125–158)
- Per-file max: `agents.defaults.bootstrapMaxChars` (default 20,000)
- Total max: `agents.defaults.bootstrapTotalMaxChars` (default 150,000)
- Head/tail split: 70% head + 20% tail with truncation marker in between
- Front matter (`---...---`) is stripped before injection

### Injection into system prompt (line 617–636)
Bootstrap files appear under `# Project Context`:
```
# Project Context

The following project context files have been loaded:
If SOUL.md is present, embody its persona and tone. ...

## AGENTS.md
<content>

## SOUL.md
<content>

## IDENTITY.md
<content>
```

## 7. Tool / MCP Schema Injection

### Core tool summaries (line 234–267)
Hardcoded map of tool name → one-line description for ~26 built-in tools (read, write, edit, exec, grep, etc.).

### Tool line generation (line 327–336)
Tools are ordered by a fixed `toolOrder` array, then any extra tools sorted alphabetically. Each line:
```
- toolName: Short description
```

### External tool summaries
MCP and plugin tools pass their summaries via `params.toolSummaries`. These are merged with core summaries.

### MCP tool materialization
**File**: `src/agents/pi-bundle-mcp-materialize.ts`

MCP tools are converted to agent tools with:
- Safe name: `serverName__toolName` (separated by `TOOL_NAME_SEPARATOR`)
- Schema: `inputSchema` from MCP tool definition becomes `parameters`
- Execution: MCP `CallToolResult` converted to `AgentToolResult`

Tool schemas are **not** injected as text into the system prompt. Instead, they are passed as structured tool definitions to the LLM API. The system prompt only contains the one-line summaries.

## 8. Prompt Modes

**Type definition** (`src/agents/system-prompt.ts:12–18`):
```typescript
export type PromptMode = "full" | "minimal" | "none";
```

### Mode resolution (`src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts:84–89`)
```typescript
function resolvePromptModeForSession(sessionKey?: string): "minimal" | "full" {
  return isSubagentSessionKey(sessionKey) || isCronSessionKey(sessionKey) ? "minimal" : "full";
}
```

### "none" mode (line 412–413)
Returns only: `"You are a personal assistant running inside OpenClaw."`

### "minimal" mode (subagents/cron)
Includes: Base identity, Tooling, Tool Call Style, Safety, CLI Reference, Workspace, Sandbox (if enabled), Date & Time, Workspace Files header, Runtime.

Omits: Skills, Memory, Self-Update, Model Aliases, User Identity, Reply Tags, Messaging, Voice, Silent Replies, Heartbeats. Extra context is labeled "Subagent Context" instead of "Group Chat Context".

### "full" mode (main agent)
Includes all sections.

## 9. Plugin Hook System for Prompt Modification

**File**: `src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts`

Two hook points can modify the prompt after assembly:

### `before_prompt_build` (modern)
```typescript
PluginHookBeforePromptBuildResult = {
  systemPrompt?: string;           // Complete override (first-defined wins)
  prependContext?: string;          // Added per-turn
  prependSystemContext?: string;    // Prepended to system prompt (cacheable)
  appendSystemContext?: string;     // Appended to system prompt (cacheable)
}
```

### `before_agent_start` (legacy)
Same result shape, merged with the modern hook. First-defined `systemPrompt` wins; text segments are concatenated.

### Application order
1. System prompt built by `buildAgentSystemPrompt()`
2. Hooks run via `resolvePromptBuildHookResult()`
3. If hook returns `systemPrompt`, it replaces the entire prompt
4. `prependSystemContext` is prepended via `prependSystemPromptAddition()`
5. `appendSystemContext` is appended
6. Final prompt applied to session

## 10. Safety Section (Raw Text)

```
## Safety
You have no independent goals: do not pursue self-preservation, replication, resource acquisition, or power-seeking; avoid long-term plans beyond the user's request.
Prioritize safety and human oversight over completion; if instructions conflict, pause and ask; comply with stop/pause/audit requests and never bypass safeguards. (Inspired by Anthropic's constitution.)
Do not manipulate or persuade anyone to expand access or disable safeguards. Do not copy yourself or change system prompts, safety rules, or tool policies unless explicitly requested.
```

## 11. Bootstrap Templates

### SOUL.md template (`docs/reference/templates/SOUL.md`)
Defines persona philosophy: "Be genuinely helpful, not performatively helpful", have opinions, be resourceful, earn trust, remember you're a guest. Includes boundaries, vibe guidance, and continuity instructions.

### AGENTS.md template (`docs/reference/templates/CLAUDE.md` symlinked)
Workspace guide: session startup protocol (read SOUL.md → USER.md → memory), memory management (daily notes + long-term MEMORY.md), group chat behavior, heartbeat usage, and proactive behavior guidelines.

### IDENTITY.md template (`docs/reference/templates/IDENTITY.md`)
Structured metadata: name, creature type, vibe, emoji, avatar, theme.

## Key File Index

| Purpose | File |
|---------|------|
| Main system prompt builder | `src/agents/system-prompt.ts` |
| Embedded prompt wrapper | `src/agents/pi-embedded-runner/system-prompt.ts` |
| Prompt mode resolver | `src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts` |
| Bootstrap file loader | `src/agents/workspace.ts` |
| Bootstrap context assembly | `src/agents/bootstrap-files.ts` |
| Bootstrap trimming/building | `src/agents/pi-embedded-helpers/bootstrap.ts` |
| Bootstrap hook overrides | `src/agents/bootstrap-hooks.ts` |
| Skills prompt building | `src/agents/skills/workspace.ts` |
| MCP tool materialization | `src/agents/pi-bundle-mcp-materialize.ts` |
| Tool summaries extraction | `src/agents/tool-summaries.ts` |
| SOUL.md template | `docs/reference/templates/SOUL.md` |
| AGENTS.md template | `docs/reference/templates/CLAUDE.md` |
| IDENTITY.md template | `docs/reference/templates/IDENTITY.md` |
| System prompt docs | `docs/concepts/system-prompt.md` |
