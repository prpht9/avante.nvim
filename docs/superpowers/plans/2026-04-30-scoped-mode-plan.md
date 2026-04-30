# Scoped Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `mode = "scoped"` as a first-class avante mode with four phases (brainstorm → planning → implementation → validation), explicit `/next` slash command for phase advancement, todo reset on phase change, and auto-injected artifact file paths between phases.

**Architecture:** `mode = "scoped"` bypasses agentic/legacy code paths. Phase state lives in `sidebar.phase` (persisted in `chat_history.phase`). Config drives all phase behavior via `scoped_phases` table. `llm.lua` injects date and artifact file paths dynamically per phase.

**Tech Stack:** Lua 5.1 (Neovim), plenary.nvim, busted (test runner via `make luatest`)

---

## File Map

| File | Change |
|---|---|
| `lua/avante/config.lua` | Mode alias, slash commands, scoped_phases |
| `lua/avante/sidebar.lua` | Default phase, PHASE_COMPLETE guard, set_phase todo reset, advance_phase end-of-workflow |
| `lua/avante/llm.lua` | Date injection, plan/spec file path injection |
| `lua/avante/utils/init.lua` | Add `get_latest_scoped_file(subdir)` |
| `tests/scoped_spec.lua` | Update for new phase names, add new behavior tests |

---

### Task 1: Update mode type alias in config.lua

**Files:**
- Modify: `lua/avante/config.lua:31`

- [ ] **Step 1: Update the type alias**

In `lua/avante/config.lua`, replace line 31:

```lua
---@alias avante.Mode "agentic" | "legacy"
```

with:

```lua
---@alias avante.Mode "agentic" | "legacy" | "scoped"
```

- [ ] **Step 2: Verify no typecheck errors introduced**

Run: `make luacheck 2>&1 | grep "avante.Mode" | head -10`

Expected: no new errors referencing `avante.Mode`

- [ ] **Step 3: Commit**

```bash
git add lua/avante/config.lua
git commit -m "feat(scoped): add 'scoped' to avante.Mode type alias"
```

---

### Task 2: Update slash commands in config.lua

**Files:**
- Modify: `lua/avante/config.lua:808-833`

- [ ] **Step 1: Replace slash_commands block**

In `lua/avante/config.lua`, replace the entire `slash_commands = { ... }` block (lines 808–833):

```lua
  ---@type AvanteSlashCommand[]
  slash_commands = {
    {
      name = "brainstorm",
      callback = function(sidebar) sidebar:set_phase("brainstorm") end,
    },
    {
      name = "planning",
      callback = function(sidebar) sidebar:set_phase("planning") end,
    },
    {
      name = "implementation",
      callback = function(sidebar) sidebar:set_phase("implementation") end,
    },
    {
      name = "validation",
      callback = function(sidebar) sidebar:set_phase("validation") end,
    },
    {
      name = "next",
      callback = function(sidebar) sidebar:advance_phase() end,
    },
  },
```

- [ ] **Step 2: Commit**

```bash
git add lua/avante/config.lua
git commit -m "feat(scoped): rename slash commands, add /next, remove /suggest"
```

---

### Task 3: Update scoped_phases in config.lua (_defaults block)

**Files:**
- Modify: `lua/avante/config.lua:838-869`

- [ ] **Step 1: Replace scoped_phases in M._defaults**

In `lua/avante/config.lua`, replace the `scoped_phases = { ... }` block inside `M._defaults` (lines 838–869):

```lua
  scoped_phases = {
    brainstorm = {
      prompt_suffix = [[Brainstorm phase: Explore ideas and clarify requirements. Ask questions only — no code, no file edits. Save the design doc to docs/superpowers/specs/{date}-<topic>-design.md when the idea is clear.]],
      enabled_tools = { "grep", "view", "think", "rag_search", "write_to_file", "read_file", "write_todos", "read_todos" },
      todo_scope = [[Brainstorm todos: High-level ideas and open questions only.]],
      next_phase = "planning",
    },
    planning = {
      prompt_suffix = [[Planning phase: Offer 2-3 implementation strategies. Write the final plan to docs/superpowers/plans/{date}-<topic>-plan.md. Only use write_to_file for the plan file.]],
      enabled_tools = { "write_to_file", "read_file", "view", "write_todos", "read_todos" },
      todo_scope = [[Planning todos: Structured implementation steps.]],
      next_phase = "implementation",
    },
    implementation = {
      prompt_suffix = [[Implementation phase: Read the most recent plan from docs/superpowers/plans/ and execute each todo step. Full tools available.]],
      enabled_tools = "all",
      todo_scope = [[Implementation todos: One code change per step, matching the plan.]],
      next_phase = "validation",
    },
    validation = {
      prompt_suffix = [[Validation phase: Write tests to validate the implementation. Run tests, fix failures, re-run. If one error persists after 3 fix attempts, stop and ask for guidance.]],
      enabled_tools = "all",
      todo_scope = [[Validation todos: Test cases to write and verify.]],
      next_phase = nil,
    },
  },
```

- [ ] **Step 2: Also update the setmetatable block (lines ~1158-1189) for consistency**

In `lua/avante/config.lua`, replace the `scoped_phases` table inside `setmetatable(M, { scoped_phases = { ... } })`:

```lua
  scoped_phases = {
    brainstorm = {
      prompt_suffix = [[Brainstorm phase: Explore ideas and clarify requirements. Ask questions only — no code, no file edits. Save the design doc to docs/superpowers/specs/{date}-<topic>-design.md when the idea is clear.]],
      enabled_tools = { "grep", "view", "think", "rag_search", "write_to_file", "read_file", "write_todos", "read_todos" },
      todo_scope = [[Brainstorm todos: High-level ideas and open questions only.]],
      next_phase = "planning",
    },
    planning = {
      prompt_suffix = [[Planning phase: Offer 2-3 implementation strategies. Write the final plan to docs/superpowers/plans/{date}-<topic>-plan.md. Only use write_to_file for the plan file.]],
      enabled_tools = { "write_to_file", "read_file", "view", "write_todos", "read_todos" },
      todo_scope = [[Planning todos: Structured implementation steps.]],
      next_phase = "implementation",
    },
    implementation = {
      prompt_suffix = [[Implementation phase: Read the most recent plan from docs/superpowers/plans/ and execute each todo step. Full tools available.]],
      enabled_tools = "all",
      todo_scope = [[Implementation todos: One code change per step, matching the plan.]],
      next_phase = "validation",
    },
    validation = {
      prompt_suffix = [[Validation phase: Write tests to validate the implementation. Run tests, fix failures, re-run. If one error persists after 3 fix attempts, stop and ask for guidance.]],
      enabled_tools = "all",
      todo_scope = [[Validation todos: Test cases to write and verify.]],
      next_phase = nil,
    },
  },
```

- [ ] **Step 3: Commit**

```bash
git add lua/avante/config.lua
git commit -m "feat(scoped): rename phases to implementation/validation, remove suggest, fix tools"
```

---

### Task 4: Add get_latest_scoped_file to utils/init.lua

**Files:**
- Modify: `lua/avante/utils/init.lua` (append before `return M`)

- [ ] **Step 1: Write the failing test**

In `tests/scoped_spec.lua`, add to the `describe("scoped mode")` block:

```lua
  it("utils.get_latest_scoped_file returns nil when no files", function()
    local result = utils.get_latest_scoped_file("plans")
    -- returns nil or a string path — both valid depending on local state
    assert.truthy(result == nil or type(result) == "string")
  end)
```

- [ ] **Step 2: Run test to verify it runs (not fails — it's a smoke test)**

Run: `make luatest 2>&1 | grep -A 3 "get_latest_scoped_file"`

Expected: test passes or the function doesn't exist yet (NameError)

- [ ] **Step 3: Add get_latest_scoped_file to utils/init.lua**

In `lua/avante/utils/init.lua`, insert before the final `return M` line (currently the last line):

```lua
---Returns the path of the most recently-named file (by YYYY-MM-DD prefix) in
---docs/superpowers/<subdir>/ relative to cwd, or nil if none exists.
---@param subdir string "plans" | "specs"
---@return string | nil
function M.get_latest_scoped_file(subdir)
  local dir = M.join_paths(vim.fn.getcwd(), "docs", "superpowers", subdir)
  if vim.fn.isdirectory(dir) == 0 then return nil end
  local files = vim.fn.glob(dir .. "/*.md", false, true)
  if not files or #files == 0 then return nil end
  table.sort(files)
  return files[#files]
end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `make luatest 2>&1 | grep -A 3 "get_latest_scoped_file"`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add lua/avante/utils/init.lua tests/scoped_spec.lua
git commit -m "feat(scoped): add get_latest_scoped_file util"
```

---

### Task 5: Update llm.lua phase prompt injection

**Files:**
- Modify: `lua/avante/llm.lua:463-468`

- [ ] **Step 1: Replace the phase injection block**

In `lua/avante/llm.lua`, replace lines 463–468:

```lua
  if opts.phase then
    local phase_cfg = Config.scoped_phases[opts.phase]
    if phase_cfg and phase_cfg.prompt_suffix then
      system_prompt = system_prompt .. "\n\n" .. phase_cfg.prompt_suffix
    end
  end
```

with:

```lua
  if opts.phase and Config.mode == "scoped" then
    local phase_cfg = Config.scoped_phases[opts.phase]
    if phase_cfg and phase_cfg.prompt_suffix then
      local today = os.date("%Y-%m-%d")
      local suffix = phase_cfg.prompt_suffix:gsub("{date}", today)
      if opts.phase == "implementation" or opts.phase == "validation" then
        local plan_path = Utils.get_latest_scoped_file("plans")
        local spec_path = Utils.get_latest_scoped_file("specs")
        if plan_path then suffix = suffix .. "\n\nPlan file: " .. plan_path end
        if spec_path then suffix = suffix .. "\nDesign spec: " .. spec_path end
      end
      system_prompt = system_prompt .. "\n\n" .. suffix
    end
  end
```

- [ ] **Step 2: Commit**

```bash
git add lua/avante/llm.lua
git commit -m "feat(scoped): inject date and artifact paths into phase prompts"
```

---

### Task 6: sidebar.lua — default phase for scoped mode

**Files:**
- Modify: `lua/avante/sidebar.lua:136`

- [ ] **Step 1: Update default phase initialization**

In `lua/avante/sidebar.lua`, replace line 136:

```lua
    phase = "normal",
```

with:

```lua
    phase = Config.mode == "scoped" and "brainstorm" or "normal",
```

- [ ] **Step 2: Update header to show SCOPED prefix when mode=scoped**

In `lua/avante/sidebar.lua`, replace line 1067:

```lua
  local header_text = Utils.icon("󰭻 ") .. "Avante [" .. string.upper(self.phase) .. "]"
```

with:

```lua
  local phase_label = Config.mode == "scoped" and ("SCOPED: " .. string.upper(self.phase)) or string.upper(self.phase)
  local header_text = Utils.icon("󰭻 ") .. "Avante [" .. phase_label .. "]"
```

- [ ] **Step 3: Update reload_chat_history to default to brainstorm in scoped mode**

In `lua/avante/sidebar.lua`, at line 2541, replace:

```lua
  if self.chat_history and self.chat_history.phase then self.phase = self.chat_history.phase end
```

with:

```lua
  if self.chat_history and self.chat_history.phase then
    self.phase = self.chat_history.phase
  elseif Config.mode == "scoped" then
    self.phase = "brainstorm"
  end
```

- [ ] **Step 4: Commit**

```bash
git add lua/avante/sidebar.lua
git commit -m "feat(scoped): default brainstorm phase, SCOPED: header prefix"
```

---

### Task 7: sidebar.lua — guard PHASE_COMPLETE detection

**Files:**
- Modify: `lua/avante/sidebar.lua:2852-2871`

- [ ] **Step 1: Wrap PHASE_COMPLETE block in mode guard**

In `lua/avante/sidebar.lua`, replace lines 2852–2871:

```lua
  -- Auto-advance phase if PHASE_COMPLETE detected
  local Utils = require("avante.utils")
  local history_messages = History.get_history_messages(self.chat_history)
  local last_msg = history_messages[#history_messages]
  if last_msg and last_msg.message.role == "assistant" then
    local content = type(last_msg.message.content) == "string" and last_msg.message.content
      or table.concat(
        vim.tbl_map(
          function(item) return type(item) == "string" and item or (item.text or "") end,
          vim.tbl_flatten(last_msg.message.content or {})
        ),
        ""
      )
    if Utils.parse_phase_complete(content) then
      vim.schedule(function()
        self:advance_phase()
        self:update_content("")
      end)
    end
  end
```

with:

```lua
  -- Auto-advance phase if PHASE_COMPLETE detected (non-scoped modes only)
  if Config.mode ~= "scoped" then
    local Utils = require("avante.utils")
    local history_messages = History.get_history_messages(self.chat_history)
    local last_msg = history_messages[#history_messages]
    if last_msg and last_msg.message.role == "assistant" then
      local content = type(last_msg.message.content) == "string" and last_msg.message.content
        or table.concat(
          vim.tbl_map(
            function(item) return type(item) == "string" and item or (item.text or "") end,
            vim.tbl_flatten(last_msg.message.content or {})
          ),
          ""
        )
      if Utils.parse_phase_complete(content) then
        vim.schedule(function()
          self:advance_phase()
          self:update_content("")
        end)
      end
    end
  end
```

- [ ] **Step 2: Commit**

```bash
git add lua/avante/sidebar.lua
git commit -m "feat(scoped): disable PHASE_COMPLETE auto-advance when mode=scoped"
```

---

### Task 8: sidebar.lua — set_phase clears todos, advance_phase handles end of workflow

**Files:**
- Modify: `lua/avante/sidebar.lua:3582-3599`

- [ ] **Step 1: Update set_phase and advance_phase**

In `lua/avante/sidebar.lua`, replace lines 3582–3599 (the existing `set_phase` and `advance_phase` functions):

```lua
Sidebar.set_phase = function(self, phase)
  self.phase = phase
  if self.chat_history then
    self.chat_history.phase = phase
    self.chat_history.todos = {}
    local Utils = require("avante.utils")
    local cfg = Utils.get_phase_config(phase)
    if cfg and cfg.prompt_suffix then
      self.chat_history.system_prompt = (self.chat_history.system_prompt or "") .. cfg.prompt_suffix
    end
  end
  self:save_history()
  self:create_todos_container()
end

Sidebar.advance_phase = function(self)
  local Utils = require("avante.utils")
  local cfg = Utils.get_phase_config(self.phase)
  if cfg and cfg.next_phase then
    self:set_phase(cfg.next_phase)
  elseif Config.mode == "scoped" then
    vim.notify("Scoped workflow complete. All phases done.", vim.log.levels.INFO)
  end
end
```

- [ ] **Step 2: Commit**

```bash
git add lua/avante/sidebar.lua
git commit -m "feat(scoped): reset todos on phase change, notify at workflow end"
```

---

### Task 9: Update tests/scoped_spec.lua

**Files:**
- Modify: `tests/scoped_spec.lua`

- [ ] **Step 1: Rewrite scoped_spec.lua with updated assertions**

Replace the entire contents of `tests/scoped_spec.lua` with:

```lua
-- Scoped mode tests
local utils = require("avante.utils")
local Config = require("avante.config")

describe("scoped mode config", function()
  it("scoped_phases has all four phases", function()
    local phases = Config.scoped_phases
    assert.truthy(phases)
    assert.truthy(phases.brainstorm)
    assert.truthy(phases.planning)
    assert.truthy(phases.implementation)
    assert.truthy(phases.validation)
  end)

  it("phase chain is correct", function()
    assert.same("planning", Config.scoped_phases.brainstorm.next_phase)
    assert.same("implementation", Config.scoped_phases.planning.next_phase)
    assert.same("validation", Config.scoped_phases.implementation.next_phase)
    assert.is_nil(Config.scoped_phases.validation.next_phase)
  end)

  it("prompt_suffix uses {date} placeholder", function()
    assert.truthy(Config.scoped_phases.brainstorm.prompt_suffix:match("{date}"))
    assert.truthy(Config.scoped_phases.planning.prompt_suffix:match("{date}"))
  end)

  it("validation has all tools enabled", function()
    assert.same("all", Config.scoped_phases.validation.enabled_tools)
  end)

  it("brainstorm and planning include write_todos", function()
    assert.truthy(vim.tbl_contains(Config.scoped_phases.brainstorm.enabled_tools, "write_todos"))
    assert.truthy(vim.tbl_contains(Config.scoped_phases.planning.enabled_tools, "write_todos"))
  end)
end)

describe("scoped mode utils", function()
  it("get_phase_tools brainstorm", function()
    local tools = utils.get_phase_tools("brainstorm")
    assert.truthy(vim.tbl_contains(tools, "grep"))
    assert.truthy(vim.tbl_contains(tools, "write_todos"))
  end)

  it("get_phase_tools implementation returns all", function()
    assert.same("all", utils.get_phase_tools("implementation"))
  end)

  it("get_phase_config implementation", function()
    local cfg = utils.get_phase_config("implementation")
    assert.truthy(cfg)
    assert.truthy(cfg.prompt_suffix:match("plan"))
  end)

  it("get_latest_scoped_file returns nil or string", function()
    local result = utils.get_latest_scoped_file("plans")
    assert.truthy(result == nil or type(result) == "string")
  end)
end)

describe("scoped mode sidebar", function()
  it("sidebar defaults to brainstorm when mode=scoped", function()
    Config.setup({ mode = "scoped" })
    local Sidebar = require("avante.sidebar")
    local tabpage = vim.api.nvim_get_current_tabpage()
    local sidebar = Sidebar:new(tabpage)
    assert.same("brainstorm", sidebar.phase)
  end)

  it("set_phase advances correctly", function()
    local Sidebar = require("avante.sidebar")
    local tabpage = vim.api.nvim_get_current_tabpage()
    local sidebar = Sidebar:new(tabpage)

    sidebar.chat_history = { todos = { { id = "1", content = "old todo", status = "todo" } } }
    sidebar:set_phase("planning")
    assert.same("planning", sidebar.phase)
    assert.same({}, sidebar.chat_history.todos)
  end)

  it("advance_phase from implementation goes to validation", function()
    local Sidebar = require("avante.sidebar")
    local tabpage = vim.api.nvim_get_current_tabpage()
    local sidebar = Sidebar:new(tabpage)

    sidebar.chat_history = { todos = {} }
    sidebar:set_phase("implementation")
    sidebar:advance_phase()
    assert.same("validation", sidebar.phase)
  end)

  it("advance_phase from validation is a no-op (end of workflow)", function()
    local Sidebar = require("avante.sidebar")
    local tabpage = vim.api.nvim_get_current_tabpage()
    local sidebar = Sidebar:new(tabpage)

    sidebar.chat_history = { todos = {} }
    sidebar:set_phase("validation")
    sidebar:advance_phase()
    assert.same("validation", sidebar.phase)
  end)
end)

describe("mode scoped setup", function()
  it("setup with mode='scoped' doesn't error", function()
    local ok, err = pcall(function() require("avante").setup({ mode = "scoped" }) end)
    if not ok then print("Scoped setup error:", vim.inspect(err)) end
    assert.is_true(ok)
  end)
end)
```

- [ ] **Step 2: Run tests**

Run: `make luatest 2>&1 | grep -E "PASS|FAIL|ERROR|scoped"`

Expected: all scoped tests PASS

- [ ] **Step 3: Commit**

```bash
git add tests/scoped_spec.lua
git commit -m "test(scoped): update spec for new phase names and behaviors"
```

---

### Task 10: Final integration smoke test

- [ ] **Step 1: Run full test suite**

Run: `make luatest 2>&1 | tail -30`

Expected: scoped tests pass, no new failures introduced in other specs

- [ ] **Step 2: Run luacheck**

Run: `make luacheck 2>&1 | grep -v "^$" | head -30`

Expected: no new errors (existing warnings about sidebar.lua W113 are pre-existing and acceptable)

- [ ] **Step 3: Verify scoped mode config roundtrip**

In a Neovim session: `:lua require("avante").setup({ mode = "scoped" })` then `:lua print(require("avante.config").mode)`

Expected output: `scoped`

- [ ] **Step 4: Verify slash commands registered**

In a Neovim session with scoped mode active, open Avante sidebar and type `/` — confirm `/brainstorm`, `/planning`, `/implementation`, `/validation`, `/next` appear in completions.

- [ ] **Step 5: Commit any final fixes**

```bash
git add -p
git commit -m "fix(scoped): integration fixes from smoke test"
```
