# Scoped Mode Design

## Overview

`mode = "scoped"` is a first-class avante mode that replaces agentic/legacy behavior with a structured, phase-driven workflow. The user moves through four discrete phases — brainstorm, planning, implementation, validation — each with constrained tools, scoped todos, and a dedicated output artifact. Phase transitions are always explicit via slash commands; there is no automatic phase detection.

## Phases

| Phase | Purpose | Output artifact |
|---|---|---|
| `brainstorm` | Explore the idea, clarify requirements | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` |
| `planning` | Define implementation strategy, write step-by-step plan | `docs/superpowers/plans/YYYY-MM-DD-<topic>-plan.md` |
| `implementation` | Execute code changes from plan | (reads plan file) |
| `validation` | Write and run tests, loop until passing | (reads plan + spec) |

The date is injected automatically (current date at phase entry). The topic name is derived by the LLM from conversation context and written in kebab-case.

## Slash Commands

- `/brainstorm` — jump to brainstorm phase
- `/planning` — jump to planning phase
- `/implementation` — jump to implementation phase
- `/validation` — jump to validation phase
- `/next` — advance to the next phase in sequence

On any phase change (jump or `/next`): `self.phase` is updated, `chat_history.todos` is cleared, sidebar header updates to `[SCOPED: <PHASE>]`.

There is no PHASE_COMPLETE detection. Phase transitions are always user-initiated.

## Phase Config

Each phase entry in `config.lua` carries:

```lua
brainstorm = {
  prompt_suffix = "...",      -- injected into system prompt
  enabled_tools = { ... },    -- hard filter on available tools
  todo_scope = "...",         -- instruction for LLM todo creation
  next_phase = "planning",    -- target of /next
},
```

### Tool Access Per Phase

- **brainstorm**: `grep`, `view`, `think`, `rag_search` (read-only)
- **planning**: `write_to_file`, `read_file`, `view`
- **implementation**: all tools
- **validation**: all tools (must run tests and fix failures in the same phase)

### Prompt Suffix Behavior

Each `prompt_suffix` instructs the LLM:
- **brainstorm**: ideas and questions only, no code, save design to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- **planning**: offer 2-3 strategies, write final plan to `docs/superpowers/plans/YYYY-MM-DD-<topic>-plan.md`
- **implementation**: read plan file, execute changes per each todo
- **validation**: write tests, run them, loop on failures (max 3 attempts per error, then ask for guidance)

## Data Flow Between Phases

```
brainstorm → writes → docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md
planning   → writes → docs/superpowers/plans/YYYY-MM-DD-<topic>-plan.md
implementation → reads plan file path (injected into system prompt)
validation → reads plan + spec paths (injected into system prompt)
```

The plan file path is persisted in `chat_history` so it survives sidebar restarts. If a user jumps directly to implementation on an existing project (crash recovery or mid-workflow entry), the system looks for the most recent file in `docs/superpowers/plans/` and injects it automatically. If multiple plan files exist, the most recently modified one is used.

## Components

### `config.lua`
- Add `"scoped"` to `avante.Mode` type alias
- Rename `implement` → `implementation`, `testing` → `validation`
- Remove `suggest` phase
- Update all `prompt_suffix` strings with file path conventions
- Add `next_phase` chain: `brainstorm` → `planning` → `implementation` → `validation` → `nil`

### `sidebar.lua`
- Gate agentic/legacy code paths: `if Config.mode ~= "scoped" then ... end`
- Register slash commands: `/brainstorm`, `/planning`, `/implementation`, `/validation`, `/next`
- On phase change: clear `chat_history.todos`, update `self.phase`, update header
- Remove PHASE_COMPLETE response parsing when in scoped mode
- Default phase on first open (no history): `brainstorm`
- Persist last phase in `chat_history.phase` for crash recovery

### `llm.lua`
- Skip agentic todo-generation prompt when `Config.mode == "scoped"`
- Inject `os.date("%Y-%m-%d")` into prompt context for file naming
- Inject plan/spec file paths from `chat_history` when entering implementation or validation

### `utils/init.lua`
- Add `get_scoped_file_paths(phase, chat_history)` — returns relevant artifact paths for a given phase

## Todo Behavior

Todos are scoped to the current phase via `todo_scope` in the prompt. On phase advance, `chat_history.todos` is cleared so the new phase starts with an empty list. The LLM creates todos appropriate to the current phase only (no upfront implementation planning during brainstorm).

## Testing

- `scoped_spec.lua`: update to cover all 4 phases
- Phase jump via slash command resets todos
- `/next` from `validation` (last phase) is a no-op or shows "workflow complete" message
- Tool filter per phase returns correct set
- Plan/spec file paths are correctly injected into implementation and validation prompts
- Crash recovery: phase persisted in history, restored on reopen
