# Scoped Mode: Agentic Default - Implementation Plan (Strategy 1: Minimal Viable)

**Date**: 2026-05-01
**Strategy**: #1 (Fastest: Config + Sidebar init/advance + Slash /agentic)
**Est. Time**: 1-2h
**Files**: config.lua, sidebar.lua, slash_commands

## Step-by-Step Todos

### 1. Config Updates (lua/avante/config.lua)

- In `M._defaults.scoped_phases`, add:
  ```lua
  agentic = {
    prompt_suffix = "",  -- Standard agentic prompt
    enabled_tools = "all",
    todo_scope = nil,
    next_phase = nil,
  },
  default_phase = "agentic",
  ```
- Update `M.scoped_phases = vim.deepcopy(M._defaults.scoped_phases)` to include new.

### 2. Sidebar Phase Init (lua/avante/sidebar.lua)

- In `Sidebar:new()` or `init()`:
  ```lua
  self.current_phase = config.scoped_phases.default_phase or "agentic"
  self:apply_phase_config()  -- New helper: set prompt_suffix, tools, etc.
  ```
- Add `apply_phase_config()`:
  ```lua
  local phase = config.scoped_phases[self.current_phase]
  self.prompt_suffix = phase.prompt_suffix or ""
  -- Apply tools/todo/next_phase
  ```

### 3. Update advance_phase() (sidebar.lua)

```lua
function sidebar:advance_phase()
  local phase = config.scoped_phases[self.current_phase]
  if phase.next_phase then
    self:set_phase(phase.next_phase)
  else
    self:append_to_chat("No next phase in " .. self.current_phase .. ". Use /brainstorm for phased workflow!")
  end
end
```

### 4. Slash Commands (config.lua or sidebar.lua)

- Add to `slash_commands`:
  ```lua
  {
    name = "agentic",
    callback = function(sidebar) sidebar:set_phase("agentic") end,
  },
  ```

### 5. Basic Tests

- Manual: Open scoped chat → verify [PHASE: agentic], /next → hint msg, /brainstorm → phase switch.
- Optional: `tests/lua/avante/scoped-agentic-init_spec.lua` for phase init.

### 6. Validation

- Lint: `make luatest`.
- Scoped chat: Default agentic, test /next idle + /brainstorm entry.

## Risks/Mitigations

- Phase config missing: Fallback to "agentic".
- Legacy: No change (starts brainstorm if hardcoded).

**Ready for Implementation** - Proceed to phase?