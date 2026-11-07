LinoriaLib — adding UI features

```
Library/
├── Library:Window(...)                     ← root window
├── Tabs.<Name>                             ← tab page
├── Tabs.<Name>:AddLeftGroupbox(name)       ← left column box
├── Tabs.<Name>:AddRightGroupbox(name)      ← right column box
│
├── Widgets (all return the widget object)
│   ├── groupbox:AddLabel(text)             ← text row
│   ├── groupbox:AddDivider()               ← horizontal rule
│   ├── groupbox:AddButton(text, cb)        ← click action
│   ├── groupbox:AddToggle(id, opts)        ← boolean
│   ├── groupbox:AddSlider(id, opts)        ← numeric range
│   ├── groupbox:AddDropdown(id, opts)      ← single/multi select
│   ├── groupbox:AddInput(id, opts)         ← text field
│   ├── groupbox:AddKeybind(id, opts)       ← key capture
│   └── widget:AddColorPicker(id, opts)     ← color (chained off a label)
│
├── Chaining patterns
│   ├── AddLabel(...):AddColorPicker(...)   ← picker needs a label parent
│   ├── AddButton(...):AddButton(...)       ← side-by-side buttons
│   └── AddButton(...):AddToggle(...)       ← button + toggle on one row
│
└── Options.<id>                            ← global registry for state
    ├── :GetValue()                         ← read current value
    ├── :SetValue(v)                        ← write + fire OnChanged
    ├── :OnChanged(cb)                      ← subscribe to changes
    └── :SetValueRGB(color)                 ← color-picker specific
```

---

Window + tabs (the skeleton)

```lua
local Library = loadstring(game:HttpGet('...'))()

local Window = Library:Window({
    Title = 'My Hub',
    Subtitle = 'v1.0',
    TabPadding = 8,
    MenuFadeTime = 0.2,
})

local Tabs = {
    Main     = Window:AddTab('Main'),
    Settings = Window:AddTab('Settings'),
}
```

Every widget below hangs off a groupbox, and every groupbox hangs off a tab. Columns are independent — you can build left first, then right, and they render side by side.

---

Groupbox (the container)

```lua
local mainBox = Tabs.Main:AddLeftGroupbox('Aimbot')
local sideBox = Tabs.Main:AddRightGroupbox('Visuals')
```

AddLeftGroupbox / AddRightGroupbox are the only two column constructors. Multiple groupboxes per side stack vertically.

---

Label (static text)

```lua
mainBox:AddLabel('Aimbot settings')
```

Labels are also the parent for color pickers (see that section) — the picker attaches to a label, not to the groupbox directly.

---

Divider (visual separator)

```lua
mainBox:AddDivider()
```

No arguments, returns nothing useful. Use to break up long groupboxes.

---

Button (click action)

```lua
mainBox:AddButton('Reset settings', function()
    print('clicked')
end)
```

Chaining two buttons on one row:

```lua
mainBox:AddButton('Save', function() print('save') end)
       :AddButton('Load', function() print('load') end)
```

The second AddButton on the returned widget places it beside the first.

---

Toggle (boolean)

```lua
mainBox:AddToggle('AimbotEnabled', {
    Text    = 'Enabled',
    Default = false,
    Tooltip = 'Toggle the aimbot on/off',
    Callback = function(value)
        print('aimbot:', value)
    end,
})
```

Read/write via Options:

```lua
if Options.AimbotEnabled:GetValue() then
    -- ...
end

Options.AimbotEnabled:SetValue(true)          -- fires Callback too
Options.AimbotEnabled:OnChanged(function(v)
    print('changed to', v)
end)
```

Button + toggle on one row (button visually attached to the toggle):

```lua
mainBox:AddButton('Reset', function() print('reset') end)
       :AddToggle('ShowReset', { Text = 'Show reset', Default = false })
```

---

Slider (numeric range)

```lua
mainBox:AddSlider('FOV', {
    Text     = 'FOV',
    Default  = 90,
    Min      = 10,
    Max      = 360,
    Rounding = 0,
    Compact  = false,   -- false = input box visible next to slider
    Callback = function(value) print('fov', value) end,
})
```

· Rounding = 0 → integers; Rounding = 2 → two decimals.
· Compact = true → hides the numeric input; useful for tight layouts.

Read:

```lua
local fov = Options.FOV:GetValue()   -- number
```

---

Dropdown (select)

Single-select:

```lua
mainBox:AddDropdown('Mode', {
    Text     = 'Mode',
    Values   = { 'Silent', 'Assist', 'Trigger' },
    Default  = 1,          -- index OR string
    Multi    = false,
    Callback = function(selected) print('mode:', selected) end,
})
```

Multi-select:

```lua
mainBox:AddDropdown('Bones', {
    Text    = 'Bones',
    Values  = { 'Head', 'Torso', 'Arms', 'Legs' },
    Default = 1,
    Multi   = true,        -- value becomes a table
    Callback = function(sel)
        for bone, on in next, sel do
            if on then print(bone) end
        end
    end,
})
```

Read / write:

```lua
local mode  = Options.Mode:GetValue()        -- 'Silent'
local bones = Options.Bones:GetValue()       -- { ['Head'] = true, ... }

Options.Mode:SetValue('Assist')              -- fires Callback
Options.Mode:SetValues({ 'a', 'b', 'c' })    -- replace the list at runtime
```

---

Input (text field)

```lua
mainBox:AddInput('WebhookURL', {
    Text        = 'Webhook',
    Default     = '',
    Placeholder = 'https://discord.com/api/...',
    Numeric     = false,       -- true = reject non-numeric input
    Finished    = true,        -- true = fire Callback only on Enter, false = every keystroke
    Callback    = function(text) print('url:', text) end,
})
```

Read:

```lua
local url = Options.WebhookURL:GetValue()   -- string
```

Finished = true is what you want for URLs, names, tokens — anything where firing on every keystroke is wasteful.

---

Keybind (key capture)

```lua
mainBox:AddKeybind('AimbotKey', {
    Text     = 'Aimbot key',
    Default  = Enum.KeyCode.E,
    Mode     = 'Toggle',   -- 'Toggle' | 'Hold' | 'Always'
    Callback = function()
        print('key pressed')
    end,
})
```

Read the current key and mode:

```lua
local keyObj = Options.AimbotKey:GetValue()
print(keyObj.Key, keyObj.Mode)   -- Enum.KeyCode.E, 'Toggle'
```

This is the one widget where you generally don't subscribe via OnChanged — the Callback fires on press, not on rebind. To react to a rebind specifically, hook Options.AimbotKey:OnChanged.

---

Color picker

Color pickers attach to a label (or any widget that supports AddColorPicker — labels are the common case):

```lua
mainBox:AddLabel('Accent color'):AddColorPicker('AccentColor', {
    Default          = Color3.fromRGB(185, 208, 250),
    Title            = 'Accent',
    Transparency     = 0,        -- 0 = opaque
    Callback         = function(c) print('accent', c) end,
})
```

Why the label parent. AddColorPicker returns a picker overlay; the label is the visible row you click. Without the label there's no anchor on the groupbox — you'd have a floating picker with no trigger.

Read / write:

```lua
local c = Options.AccentColor:GetValue()        -- Color3
Options.AccentColor:SetValueRGB(Color3.new(1, 0, 0))  -- fires Callback
```

SetValueRGB is color-picker specific — SetValue expects the picker's internal table shape, so always use SetValueRGB when scripting a color change.

---

The Options registry (why every widget has an ID)

Every Add* call with a string ID registers a proxy in the global Options table:

```lua
Options.AimbotEnabled:GetValue()      -- read
Options.AimbotEnabled:SetValue(v)     -- write + fire Callback
Options.AimbotEnabled:OnChanged(cb)   -- subscribe
```

Rules:

· ID must be unique. Registering the same ID twice silently overwrites.
· Widgets without IDs (labels, dividers, plain buttons) don't create Options entries — you can't script them, only click them.
· Options persists per-session only — it's an in-memory table. Persist to disk yourself (that's what ThemeManager:SaveCustomTheme does for colors).

---

Adding a feature — worked example

Say you want a "Fly speed" slider + "Enable fly" toggle on the Main tab.

```lua
-- 1. groupbox (only if you don't already have one on this side)
local flyBox = Tabs.Main:AddRightGroupbox('Fly')

-- 2. toggle
flyBox:AddToggle('FlyEnabled', {
    Text    = 'Enable fly',
    Default = false,
    Callback = function(v)
        -- your logic here
    end,
})

-- 3. slider
flyBox:AddSlider('FlySpeed', {
    Text     = 'Speed',
    Default  = 50,
    Min      = 1,
    Max      = 500,
    Rounding = 0,
    Callback = function(v)
        -- your logic here
    end,
})

-- 4. consume the state elsewhere
RunService.Heartbeat:Connect(function()
    if not Options.FlyEnabled:GetValue() then return end
    local speed = Options.FlySpeed:GetValue()
    -- apply to character...
end)
```

Three patterns to internalize:

1. UI layer — Add* builds the widget and registers its ID.
2. State layer — Options.<id>:GetValue() reads, :OnChanged subscribes.
3. Logic layer — your Heartbeat/hook/event loop reads Options and does the work.

Never put game logic inside Callback if it needs to run continuously — Callback only fires on change. Use it to set a cached variable, then read that variable in your loop.
