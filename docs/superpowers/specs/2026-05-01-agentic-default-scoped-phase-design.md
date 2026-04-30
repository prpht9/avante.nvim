# Agentic Default Scoped Phase Design

## Overview
Add a new `agentic` phase to `scoped_phases` that mirrors "agentic" mode: full tool access (`enabled_tools = \"all\""), no `next_phase`, minimal `prompt_suffix`, free-form `todo_scope`. Make it the configurable default for scoped mode via `scoped_phases.default_phase = \"agentic\"`. Add `/agentic` slash command for switching.

## Goals
- Default to full-capability \"agentic\" phase in scoped mode (instead of \"brainstorm\").
- Allow user-configurable default phase.
- Enable switching to/from structured phases (/brainstorm etc.) via slash commands.
- No phase chaining from agentic; user controls transitions.

## Config Changes (lua/avante/config.lua)
1. Add to `M._defaults.scoped_phases`:
   ```lua
   default_phase = \"agentic\",
   agentic = {
     prompt_suffix = \"Agentic phase: Full tools available. No restrictions. Use /agentic to confirm, or switch phases with /brainstorm etc.\",
     enabled_tools = \"all\",
     todo_scope = \"Agentic todos: Free-form tasks with complete agent capabilities.\",
     next_phase = nil,
   },
   ```
   - Place as first key so legacy code falls back naturally.
   - Update `M.scoped_phases = vim.deepcopy(M._defaults.scoped_phases)`.

2. Add to `slash_commands`:
   ```lua
   {
     name = \"agentic\",
     callback = function(sidebar) sidebar:set_phase(\"agentic\") end,
   },
   ```

## Code Changes (lua/avante/sidebar.lua)
1. In `Sidebar:new(id)`:
   ```lua
   self.phase = Config.mode == \"scoped\" and (Config.scoped_phases.default_phase or \"brainstorm\") or \"normal\",
   ```
   - Uses new config field; falls back to \"brainstorm\" if unset.

## Usage Flow
- Scoped mode starts in \"agentic\" (full tools, no limits).
- `/brainstorm` → structured phases (chained via `/next`).
- `/agentic` → back to full mode anytime.
- User config: `require(\"avante\").setup({ scoped_phases = { default_phase = \"brainstorm\" } })` to override.

## Validation
- Test scoped mode defaults to agentic (full tools).
- `/agentic` sets phase correctly.
- Phase suffix appears in prompts (llm.lua uses `self.phase`).
- No regressions in existing phases.
- Config override works.

## Edge Cases
- Legacy configs without `default_phase`: falls to \"brainstorm\".
- Invalid phase name: log warn, default \"agentic\".
- ACP providers: phase metadata preserved.

Ready for implementation.