# Prompt Inventory

## Scope

This repository is a flattened subset of a larger Claude Code source snapshot.
It contains the core CLI and prompt-plumbing files, but many referenced prompt
modules are not present in this checkout. This inventory separates:

1. Prompt text we can recover directly from the files in this repo.
2. Prompt assembly paths that explain how instructions reach the model.
3. Referenced prompt sources that are missing from this snapshot.

## Repo Functionality

Based on the available files, this project is a Claude Code-style CLI/runtime
for agentic software engineering:

- Interactive REPL and headless `--print` mode
- Tool use and permission-gated command execution
- Session/history management
- Multi-agent / teammate support
- MCP integration
- Agent, skill, and prompt-command loading
- Remote, bridge, and teleport flows

Primary files:

- `main.tsx`
- `QueryEngine.ts`
- `query.ts`
- `tools.ts`
- `commands.ts`
- `context.ts`

## Prompt Assembly Pipeline

### 1. CLI prompt inputs

`main.tsx` accepts both full replacement and appended system prompts:

- `--system-prompt`
- `--system-prompt-file`
- `--append-system-prompt`
- `--append-system-prompt-file`

Source: `main.tsx:1342-1388`

### 2. Runtime addenda appended in `main.tsx`

The runtime can append additional instruction layers for:

- teammate addenda
- custom agent instructions
- Claude-in-Chrome hints/prompts
- assistant addenda
- proactive mode
- agent memory merge prompts

Relevant sources:

- `main.tsx:1384-1387`
- `main.tsx:1549-1572`
- `main.tsx:2166-2168`
- `main.tsx:2203-2208`
- `main.tsx:2267-2270`

### 3. Final system prompt construction in `QueryEngine.ts`

The headless/SDK path assembles the final system prompt in this order:

1. `customSystemPrompt`, if present, otherwise `defaultSystemPrompt`
2. optional memory-mechanics prompt
3. `appendSystemPrompt`

Source: `QueryEngine.ts:286-325`

### 4. Final system context wrapping in `query.ts`

`query.ts` converts the assembled prompt into `fullSystemPrompt` by appending
runtime system context before the model call.

Source: `query.ts:449-450`

## Recovered Prompt Text

### Proactive mode prompt

Source: `main.tsx:2203`

```text
# Proactive Mode

You are in proactive mode. Take initiative — explore, act, and make progress without waiting for instructions.

Start by briefly greeting the user.

You will receive periodic <tick> prompts. These are check-ins. Do whatever seems most useful, or call Sleep if there's nothing to do. ${briefVisibility}
```

Notes:

- This is the strongest directly recoverable inline system instruction block.
- `${briefVisibility}` is filled with either:
  - `Call SendUserMessage at checkpoints to mark where things stand.`
  - `The user will see any text you output.`

Source for the visibility string: `main.tsx:2201`

### Custom agent wrapper

Source: `main.tsx:2167`

```text
# Custom Agent Instructions
${customPrompt}
```

Notes:

- The wrapper is present in this repo.
- The underlying `customPrompt` body depends on agent definitions and is not
  recoverable from this checkout.

### Git status context block

Source: `context.ts:96-103`

```text
This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation.

Current branch: ${branch}

Main branch (you will usually use this for PRs): ${mainBranch}

Git user: ${userName}

Status:
${truncatedStatus || '(clean)'}

Recent commits:
${log}
```

Notes:

- `Git user:` is conditional.
- This is runtime context, but it is clearly model-facing instruction/context.

### Date context block

Source: `context.ts:184-186`

```text
Today's date is ${getLocalISODate()}.
```

### Project onboarding instruction

Source: `projectOnboardingState.ts:28-35`

```text
Ask Claude to create a new app or clone a repository
Run /init to create a CLAUDE.md file with instructions for Claude
```

Notes:

- This is UI/onboarding copy rather than the main system prompt.
- It still reveals part of the instruction model: `CLAUDE.md` is expected to
  contain user/project instructions for Claude.

### Deep-link prefill warning

Source: `main.tsx:3794`

```text
Launched with a pre-filled prompt — review it before pressing Enter.
```

Notes:

- This is a safety banner, not a model instruction.
- It is included here because it describes an external prompt provenance path.

## Recovered Prompt-Carrying Mechanisms

### `CLAUDE.md` injection

`context.ts` loads `CLAUDE.md` content into `claudeMd` and includes it in user
context when enabled.

Source:

- `context.ts:170-176`
- `context.ts:184-186`

Implication:

- Project-local instruction files are a first-class prompt source.

### Memory prompt injection

When a caller supplies a custom system prompt and sets
`CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`, the runtime injects a memory-mechanics
prompt via `loadMemoryPrompt()`.

Source: `QueryEngine.ts:310-319`

Implication:

- There is at least one additional internal prompt file dedicated to memory
  usage semantics, but its body is not present here.

### Teammate prompt addendum

The runtime appends `TEAMMATE_SYSTEM_PROMPT_ADDENDUM` for tmux teammates.

Source: `main.tsx:1384-1387`

Implication:

- A dedicated teammate coordination prompt exists, but its module is missing
  from this snapshot.

### Claude-in-Chrome prompt/hint injection

The runtime appends either `chromeSystemPrompt` or a Chrome skill hint.

Source:

- `main.tsx:1542-1550`
- `main.tsx:1569-1572`

Implication:

- Chrome/browser integration uses dedicated prompt text, but the prompt module
  is not present here.

### Assistant addendum

The runtime appends `assistantModule.getAssistantSystemPromptAddendum()`.

Source: `main.tsx:2206-2208`

Implication:

- Assistant mode has its own addendum prompt, but the body is not available in
  this checkout.

## Prompt Command Surfaces

`commands.ts` confirms that prompt-type commands exist and are model-invocable.

Notable findings:

- `insights` is explicitly registered as a `type: 'prompt'` command.
- Skill-related filtering is based on `cmd.type === 'prompt'`.
- Prompt commands can come from bundled skills, user skills, plugins, MCP, and
  legacy command directories.

Useful sources:

- `commands.ts:190-200`
- `commands.ts:542-592`
- `commands.ts:645-674`

Implication:

- This repo clearly supports a larger prompt-command ecosystem, but most of the
  actual prompt bodies live outside the files present here.

## Prompt Plumbing Files

These files are important for analysis even when they do not contain large
literal prompt bodies:

- `Tool.ts:171-174`
  - defines `customSystemPrompt` and `appendSystemPrompt`
- `context.ts:22-31`
  - ant-only system prompt cache-breaker injection path
- `query.ts:588-589`
  - references `createDumpPromptsFetch(...)`, suggesting prompt payload capture
- `tools.ts:191`
  - notes that tool ordering must stay in sync for system prompt caching

## Referenced But Missing Prompt Sources

These modules are referenced by the available code and likely contain important
prompt text or prompt section definitions, but they are not present in this
checkout:

- `./utils/queryContext.js`
- `./memdir/memdir.js`
- `./utils/claudeInChrome/prompt.js`
- `./utils/swarm/teammatePromptAddendum.js`
- `./tools/BriefTool/prompt.js`
- `./tools/SleepTool/prompt.js`
- `./constants/systemPromptSections.js`
- assistant/agent prompt modules loaded through `assistantModule`
- custom agent definitions loaded outside this subset

## Analyst Notes

- The most important directly recoverable instruction text is the proactive
  mode block in `main.tsx`.
- The most important architectural finding is that this subset exposes prompt
  composition more clearly than prompt content.
- `CLAUDE.md`, custom agent prompts, teammate prompts, memory prompts, and
  assistant addenda are all first-class prompt layers.
- The default base system prompt is not present in this checkout.
- This inventory should be treated as a high-confidence map of prompt sources,
  not a complete dump of all original Anthropic instruction text.
