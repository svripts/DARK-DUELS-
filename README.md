repeat task.wait() until game:IsLoaded()
-- ============================================================
-- OPIUM HUB - DARK DUELS STYLE
-- Complete redesign with card-based UI
-- ============================================================

local Players         = game:GetService("Players")
local RunService      = game:GetService("RunService")
local UIS             = game:GetService("UserInputService")
local TweenService    = game:GetService("TweenService")
local HttpService     = game:GetService("HttpService")
local Lighting        = game:GetService("Lighting")
local LP              = Players.LocalPlayer

local _isfile   = isfile   or (syn and syn.isfile)   or function() return false end
local _readfile = readfile  or (syn and syn.readfile)  or function() return nil  end
local _writefile= writefile or (syn and syn.writefile) or function() end
local getconnections = getconnections or get_signal_cons or getconnects or (syn and syn.get_signal_cons)

-- ============================================================
-- STATE
-- ============================================================
local State = {
    normalSpeed=63, carrySpeed=29.5, laggerSpeed=22.5, laggerCarrySpeed=52,
    aimbotSpeed=55,
    speedToggled=false, laggerEnabled=false,
    infJumpEnabled=false, antiRagdollEnabled=false, fpsBoostEnabled=false,
    guiVisible=true, uiLocked=false,
    isStealing=false, stealStartTime=nil, lastStealTick=0,
    autoLeftEnabled=false, autoRightEnabled=false,
    autoLeftPhase=1, autoRightPhase=1,
    medusaLastUsed=0, medusaDebounce=false, medusaCounterEnabled=false,
    batAimbotToggled=false, autoSwingEnabled=false,
    hittingCooldown=false,
    batCounterEnabled=false, batCounterDebounce=false,
    dropEnabled=false, batLockEnabled=false,
    lastMoveDir=Vector3.new(0,0,0),
    unwalkEnabled=false,
    _prevCarry=29.5, _prevSpeed=false,
    _laggerOriginalCarry=29.5,
    autoStealEnabled=false,
    duelLaggerActive=false,
    duelLaggerThread=nil,
    countdownActive=false,
    instaGrabEnabled=false,
}

local Keys = {
    speed      = Enum.KeyCode.Q,
    lagger     = Enum.KeyCode.R,
    autoLeft   = Enum.KeyCode.Z,
    autoRight  = Enum.KeyCode.C,
    drop       = Enum.KeyCode.H,
    tpDown     = Enum.KeyCode.V,
    aimbot     = Enum.KeyCode.E,
    unwalk     = Enum.KeyCode.U,
    guiHide    = Enum.KeyCode.LeftControl,
    duelLagger = Enum.KeyCode.F,
    instaReset = Enum.KeyCode.G,
    instaGrab  = Enum.KeyCode.T,
}

local Steal = {
    AutoStealEnabled=false, StealRadius=20, StealDuration=0.25,
    Data={}, plotCache={}, plotCacheTime={}, cachedPrompts={}, promptCacheTime=0,
}

local Presets = {}
local PRESET_FILE = "opiumhubPresets.json"
local LAST_PRESET_FILE = "opiumhubLastPreset.json"
local CONFIG_FILE = "opiumhubConfig.json"

local POS={
    L1=Vector3.new(-476.48,-6.28,92.73), L2=Vector3.new(-483.12,-4.95,94.80),
    R1=Vector3.new(-476.16,-6.52,25.62), R2=Vector3.new(-483.04,-5.09,23.14),
}

local Conns={autoSteal=nil,antiRag=nil,autoLeft=nil,autoRight=nil,aimbot=nil,anchor={},progress=nil,batCounter=nil,unwalk=nil,instaGrab=nil}

local PLOT_CACHE_DURATION=2
local PROMPT_CACHE_REFRESH=0.15
local STEAL_COOLDOWN=0.1
local MEDUSA_COOLDOWN=25
local DROP_AUTO_OFF_DELAY=0.15

local isTouchEnabled = UIS.TouchEnabled
local duelLaggerWaitTime = isTouchEnabled and 5.8 or 0.25

local h,hrp
local speedBoxRefs={}
local keybindBtnRefs={}
local toggleRefs={}
local mbGroup
local setAL, setAR, setAB

-- ============================================================
-- COLORS (GREEN DUELS STYLE)
-- ============================================================
local C = {
    bg          = Color3.fromRGB(8, 10, 18),
    cardBg      = Color3.fromRGB(14, 18, 28),
    cardBorder  = Color3.fromRGB(30, 50, 80),
    textMain    = Color3.fromRGB(240, 245, 255),
    textSub     = Color3.fromRGB(140, 170, 200),
    textGreen   = Color3.fromRGB(60, 180, 255),
    accent      = Color3.fromRGB(30, 144, 255),
    accentDark  = Color3.fromRGB(20, 80, 180),
    divider     = Color3.fromRGB(25, 35, 50),
    inputBg     = Color3.fromRGB(18, 22, 35),
    inputBorder = Color3.fromRGB(60, 180, 255),
    toggleOff   = Color3.fromRGB(35, 40, 50),
    toggleOn    = Color3.fromRGB(30, 144, 255),
    dotOff      = Color3.fromRGB(60, 70, 85),
    dotOn       = Color3.fromRGB(255, 255, 255),
    headerBg    = Color3.fromRGB(10, 14, 24),
    buttonBg    = Color3.fromRGB(25, 35, 50),
    buttonHover = Color3.fromRGB(40, 60, 90),
}

-- ============================================================
-- HELPER FUNCTIONS
-- ============================================================
local function getKeyName(kc)
    local n = kc.Name
    if n == "Unknown" then return "?" end
    if n == "LeftControl" then return "CTRL" end
    if n == "RightControl" then return "RCTL" end
    if n == "LeftShift" then return "SHFT" end
    if n == "Space" then return "SPC" end
    return n:sub(1, 4):upper()
end

local function mkCorner(p,r) 
    local c = Instance.new("UICorner", p)
    c.CornerRadius = UDim.new(0, r or 8)
    return c
end

local function mkStroke(p, col, th)
    local s = Instance.new("UIStroke", p)
    s.Color = col or C.cardBorder
    s.Thickness = th or 1
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    return s
end

local function makeDraggable(frame, handle, force)
    local src = handle or frame
    local dragging, dragInput, dragStart, startPos = false, nil, nil, nil
    src.InputBegan:Connect(function(inp)
        if State.uiLocked and not force then return end
        if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = inp.Position
            startPos = frame.Position
            inp.Changed:Connect(function()
                if inp.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    src.InputChanged:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then
            dragInput = inp
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if inp == dragInput and dragging and (force or not State.uiLocked) then
            local dx = inp.Position.X - dragStart.X
            local dy = inp.Position.Y - dragStart.Y
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + dx, startPos.Y.Scale, startPos.Y.Offset + dy)
        end
    end)
end

-- ============================================================
-- PRESET FUNCTIONS
-- ============================================================
local function buildPresetSnapshot()
    return {
        normalSpeed=State.normalSpeed, carrySpeed=State.carrySpeed, laggerSpeed=State.laggerSpeed,
        aimbotSpeed=State.aimbotSpeed,
        stealRadius=Steal.StealRadius, stealDuration=Steal.StealDuration,
        infJump=State.infJumpEnabled, antiRagdoll=State.antiRagdollEnabled,
        fpsBoost=State.fpsBoostEnabled, medusaCounter=State.medusaCounterEnabled,
        batCounter=State.batCounterEnabled, autoSteal=Steal.AutoStealEnabled,
    }
end

local function savePresetsFile()
    local ok, encoded = pcall(function() return HttpService:JSONEncode(Presets) end)
    if ok then pcall(function() _writefile(PRESET_FILE, encoded) end) end
end

local function loadPresetsFile()
    local hasFile = false
    pcall(function() hasFile = _isfile(PRESET_FILE) end)
    if not hasFile then return end
    local raw
    pcall(function() raw = _readfile(PRESET_FILE) end)
    if not raw then return end
    local ok, decoded = pcall(function() return HttpService:JSONDecode(raw) end)
    if ok and decoded then Presets = decoded end
end

local function saveLastPresetName(name)
    local ok, encoded = pcall(function() return HttpService:JSONEncode({lastPreset=name}) end)
    if ok then pcall(function() _writefile(LAST_PRESET_FILE, encoded) end) end
end

local function loadLastPresetName()
    local hasFile = false
    pcall(function() hasFile = _isfile(LAST_PRESET_FILE) end)
    if not hasFile then return nil end
    local raw
    pcall(function() raw = _readfile(LAST_PRESET_FILE) end)
    if not raw then return nil end
    local ok, decoded = pcall(function() return HttpService:JSONDecode(raw) end)
    if ok and decoded then return decoded.lastPreset end
    return nil
end

-- ============================================================
-- CONFIG SAVE/LOAD
-- ============================================================
local function saveConfig()
    local cfg = {
        normalSpeed = State.normalSpeed,
        carrySpeed = State.carrySpeed,
        laggerSpeed = State.laggerSpeed,
        aimbotSpeed = State.aimbotSpeed,
        stealRadius = Steal.StealRadius,
        stealDuration = Steal.StealDuration,
        speedKey = Keys.speed.Name,
        autoLeftKey = Keys.autoLeft.Name,
        autoRightKey = Keys.autoRight.Name,
        guiHideKey = Keys.guiHide.Name,
        dropKey = Keys.drop.Name,
        laggerKey = Keys.lagger.Name,
        tpDownKey = Keys.tpDown.Name,
        aimbotKey = Keys.aimbot.Name,
        unwalkKey = Keys.unwalk.Name,
        duelLaggerKey = Keys.duelLagger.Name,
        instaResetKey = Keys.instaReset.Name,
        instaGrabKey = Keys.instaGrab.Name,
    }
    local ok, enc = pcall(function() return HttpService:JSONEncode(cfg) end)
    if ok and enc then
        pcall(function() _writefile(CONFIG_FILE, enc) end)
    end
end

local function loadConfig()
    local has = false
    pcall(function() has = _isfile(CONFIG_FILE) end)
    if not has then return end
    local raw
    pcall(function() raw = _readfile(CONFIG_FILE) end)
    if not raw then return end
    local cfg
    local ok = pcall(function() cfg = HttpService:JSONDecode(raw) end)
    if not ok or not cfg then return end

    if cfg.normalSpeed then State.normalSpeed = cfg.normalSpeed end
    if cfg.carrySpeed then State.carrySpeed = cfg.carrySpeed end
    if cfg.laggerSpeed then State.laggerSpeed = cfg.laggerSpeed end
    if cfg.aimbotSpeed then State.aimbotSpeed = cfg.aimbotSpeed end
    if cfg.stealRadius then Steal.StealRadius = cfg.stealRadius end
    if cfg.stealDuration then Steal.StealDuration = cfg.stealDuration end

    local function tryKey(field, kt)
        if cfg[field] and Enum.KeyCode[cfg[field]] then
            Keys[kt] = Enum.KeyCode[cfg[field]]
        end
    end
    tryKey("speedKey", "speed")
    tryKey("autoLeftKey", "autoLeft")
    tryKey("autoRightKey", "autoRight")
    tryKey("guiHideKey", "guiHide")
    tryKey("dropKey", "drop")
    tryKey("laggerKey", "lagger")
    tryKey("tpDownKey", "tpDown")
    tryKey("aimbotKey", "aimbot")
    tryKey("unwalkKey", "unwalk")
    tryKey("duelLaggerKey", "duelLagger")
    tryKey("instaResetKey", "instaReset")
    tryKey("instaGrabKey", "instaGrab")
end

-- ============================================================
-- CLEANUP OLD GUIs
-- ============================================================
for _, name in pairs({"opiumhubV5_3"}) do
    pcall(function()
        local o = game:GetService("CoreGui"):FindFirstChild(name)
        if o then o:Destroy() end
    end)
    pcall(function()
        local o = LP:WaitForChild("PlayerGui"):FindFirstChild(name)
        if o then o:Destroy() end
    end)
end

-- ============================================================
-- ROOT GUI
-- ============================================================
local gui = Instance.new("ScreenGui")
gui.Name = "opiumhubV5_3"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.IgnoreGuiInset = true
pcall(function()
    if syn and syn.protect_gui then syn.protect_gui(gui) end
end)
pcall(function() gui.Parent = game:GetService("CoreGui") end)
if not gui.Parent then
    pcall(function() gui.Parent = LP.PlayerGui end)
end

local uiScaleObj = Instance.new("UIScale", gui)
uiScaleObj.Scale = 1.0

-- ============================================================
-- MAIN WINDOW
-- ============================================================
local WIN_W = 340
local WIN_H = 620

local mainOuter = Instance.new("Frame", gui)
mainOuter.Name = "MainOuter"
mainOuter.Size = UDim2.new(0, WIN_W, 0, WIN_H)
mainOuter.Position = UDim2.new(0, 8, 0, 8)
mainOuter.BackgroundColor3 = C.bg
mainOuter.BorderSizePixel = 0
mainOuter.ClipsDescendants = false
mkCorner(mainOuter, 16)
mkStroke(mainOuter, C.cardBorder, 1)
makeDraggable(mainOuter)

-- ============================================================
-- HEADER
-- ============================================================
local HEADER_H = 80
local headerFrame = Instance.new("Frame", mainOuter)
headerFrame.Size = UDim2.new(1, 0, 0, HEADER_H)
headerFrame.Position = UDim2.new(0, 0, 0, 0)
headerFrame.BackgroundColor3 = C.headerBg
headerFrame.BorderSizePixel = 0
headerFrame.ZIndex = 3
mkCorner(headerFrame, 16)

local titleLbl = Instance.new("TextLabel", headerFrame)
titleLbl.Size = UDim2.new(1, -40, 0, 32)
titleLbl.Position = UDim2.new(0, 20, 0, 18)
titleLbl.BackgroundTransparency = 1
titleLbl.Text = "GREEN DUELS"
titleLbl.TextColor3 = C.textGreen
titleLbl.Font = Enum.Font.GothamBlack
titleLbl.TextSize = 24
titleLbl.TextXAlignment = Enum.TextXAlignment.Left
titleLbl.ZIndex = 4

local subLbl = Instance.new("TextLabel", headerFrame)
subLbl.Size = UDim2.new(1, -40, 0, 18)
subLbl.Position = UDim2.new(0, 22, 0, 52)
subLbl.BackgroundTransparency = 1
subLbl.Text = "powered by luck ðŸ†"
subLbl.TextColor3 = C.textSub
subLbl.Font = Enum.Font.Gotham
subLbl.TextSize = 11
subLbl.TextXAlignment = Enum.TextXAlignment.Left
subLbl.ZIndex = 4

local minimizeBtn = Instance.new("TextButton", headerFrame)
minimizeBtn.Size = UDim2.new(0, 28, 0, 28)
minimizeBtn.Position = UDim2.new(1, -42, 0, 12)
minimizeBtn.BackgroundColor3 = C.cardBg
minimizeBtn.BorderSizePixel = 0
minimizeBtn.Text = "â€”"
minimizeBtn.TextColor3 = C.textSub
minimizeBtn.Font = Enum.Font.GothamBold
minimizeBtn.TextSize = 16
minimizeBtn.ZIndex = 5
mkCorner(minimizeBtn, 8)
mkStroke(minimizeBtn, C.cardBorder, 1)
minimizeBtn.MouseButton1Click:Connect(function()
    mainOuter.Visible = false
end)

local headerDiv = Instance.new("Frame", mainOuter)
headerDiv.Size = UDim2.new(1, 0, 0, 1)
headerDiv.Position = UDim2.new(0, 0, 0, HEADER_H)
headerDiv.BackgroundColor3 = C.divider
headerDiv.BorderSizePixel = 0
headerDiv.ZIndex = 3

-- ============================================================
-- SPEED VALUES SECTION
-- ============================================================
local SPEED_BAR_H = 100
local speedBar = Instance.new("Frame", mainOuter)
speedBar.Size = UDim2.new(1, 0, 0, SPEED_BAR_H)
speedBar.Position = UDim2.new(0, 0, 0, HEADER_H + 1)
speedBar.BackgroundColor3 = C.bg
speedBar.BorderSizePixel = 0
speedBar.ZIndex = 3

local function makeSpeedCard(label, stateKey, order)
    local card = Instance.new("Frame", speedBar)
    card.Size = UDim2.new(0.33, -6, 0.9, -8)
    card.Position = UDim2.new(order * 0.33 + 0.005, 4, 0.05, 0)
    card.BackgroundColor3 = C.cardBg
    card.BorderSizePixel = 0
    card.ZIndex = 4
    mkCorner(card, 12)
    mkStroke(card, C.cardBorder, 1)

    local lbl = Instance.new("TextLabel", card)
    lbl.Size = UDim2.new(1, 0, 0, 22)
    lbl.Position = UDim2.new(0, 0, 0, 10)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = C.textSub
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 10
    lbl.TextXAlignment = Enum.TextXAlignment.Center
    lbl.ZIndex = 5

    local valBox = Instance.new("TextBox", card)
    valBox.Size = UDim2.new(0.7, 0, 0, 36)
    valBox.Position = UDim2.new(0.15, 0, 0, 38)
    valBox.BackgroundColor3 = C.inputBg
    valBox.BorderSizePixel = 0
    valBox.Text = tostring(State[stateKey])
    valBox.TextColor3 = C.textGreen
    valBox.Font = Enum.Font.GothamBold
    valBox.TextSize = 22
    valBox.ClearTextOnFocus = false
    valBox.ZIndex = 5
    mkCorner(valBox, 8)
    mkStroke(valBox, C.inputBorder, 1)

    valBox.FocusLost:Connect(function()
        local n = tonumber(valBox.Text)
        if n and n > 0 and n <= 500 then
            State[stateKey] = n
            valBox.Text = tostring(n)
        else
            valBox.Text = tostring(State[stateKey])
        end
        task.spawn(saveConfig)
    end)
    speedBoxRefs[stateKey] = valBox
    return valBox
end

makeSpeedCard("NORMAL\nSPEED", "normalSpeed", 0)
makeSpeedCard("CARRY\nSPEED", "carrySpeed", 1)
makeSpeedCard("LAGGER\nSPEED", "laggerSpeed", 2)

local speedDiv = Instance.new("Frame", mainOuter)
speedDiv.Size = UDim2.new(1, 0, 0, 1)
speedDiv.Position = UDim2.new(0, 0, 0, HEADER_H + SPEED_BAR_H)
speedDiv.BackgroundColor3 = C.divider
speedDiv.BorderSizePixel = 0
speedDiv.ZIndex = 3

-- ============================================================
-- SCROLL AREA
-- ============================================================
local SCROLL_Y = HEADER_H + SPEED_BAR_H + 1
local scrollFrame = Instance.new("ScrollingFrame", mainOuter)
scrollFrame.Size = UDim2.new(1, 0, 1, -SCROLL_Y)
scrollFrame.Position = UDim2.new(0, 0, 0, SCROLL_Y)
scrollFrame.BackgroundColor3 = C.bg
scrollFrame.BackgroundTransparency = 0
scrollFrame.BorderSizePixel = 0
scrollFrame.ClipsDescendants = true
scrollFrame.ScrollBarThickness = 3
scrollFrame.ScrollBarImageColor3 = C.accentDark
scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
scrollFrame.ZIndex = 2

local scrollLL = Instance.new("UIListLayout", scrollFrame)
scrollLL.SortOrder = Enum.SortOrder.LayoutOrder
scrollLL.Padding = UDim.new(0, 8)

local scrollPad = Instance.new("UIPadding", scrollFrame)
scrollPad.PaddingLeft = UDim.new(0, 12)
scrollPad.PaddingRight = UDim.new(0, 12)
scrollPad.PaddingTop = UDim.new(0, 8)
scrollPad.PaddingBottom = UDim.new(0, 12)

local loCount = 0
local function LO()
    loCount = loCount + 1
    return loCount
end

-- ============================================================
-- SECTION CARD
-- ============================================================
local function makeSectionCard(title, icon)
    local card = Instance.new("Frame", scrollFrame)
    card.Size = UDim2.new(1, 0, 0, 0)
    card.AutomaticSize = Enum.AutomaticSize.Y
    card.BackgroundColor3 = C.cardBg
    card.BorderSizePixel = 0
    card.LayoutOrder = LO()
    card.ZIndex = 3
    mkCorner(card, 12)
    mkStroke(card, C.cardBorder, 1)

    local titleBar = Instance.new("Frame", card)
    titleBar.Size = UDim2.new(1, 0, 0, 40)
    titleBar.BackgroundColor3 = C.cardBg
    titleBar.BorderSizePixel = 0
    titleBar.ZIndex = 4
    mkCorner(titleBar, 12)

    local iconLbl = Instance.new("TextLabel", titleBar)
    iconLbl.Size = UDim2.new(0, 30, 1, 0)
    iconLbl.Position = UDim2.new(0, 12, 0, 0)
    iconLbl.BackgroundTransparency = 1
    iconLbl.Text = icon or "ðŸ“Œ"
    iconLbl.TextColor3 = C.textGreen
    iconLbl.Font = Enum.Font.Gotham
    iconLbl.TextSize = 18
    iconLbl.TextXAlignment = Enum.TextXAlignment.Center
    iconLbl.ZIndex = 5

    local titleLbl = Instance.new("TextLabel", titleBar)
    titleLbl.Size = UDim2.new(1, -50, 1, 0)
    titleLbl.Position = UDim2.new(0, 48, 0, 0)
    titleLbl.BackgroundTransparency = 1
    titleLbl.Text = title
    titleLbl.TextColor3 = C.textMain
    titleLbl.Font = Enum.Font.GothamBold
    titleLbl.TextSize = 15
    titleLbl.TextXAlignment = Enum.TextXAlignment.Left
    titleLbl.ZIndex = 5

    local div = Instance.new("Frame", card)
    div.Size = UDim2.new(1, -24, 0, 1)
    div.Position = UDim2.new(0, 12, 0, 40)
    div.BackgroundColor3 = C.divider
    div.BorderSizePixel = 0
    div.ZIndex = 4

    local content = Instance.new("Frame", card)
    content.Size = UDim2.new(1, 0, 0, 0)
    content.Position = UDim2.new(0, 0, 0, 45)
    content.AutomaticSize = Enum.AutomaticSize.Y
    content.BackgroundTransparency = 1
    content.BorderSizePixel = 0
    content.ZIndex = 3

    local contentList = Instance.new("UIListLayout", content)
    contentList.SortOrder = Enum.SortOrder.LayoutOrder
    contentList.Padding = UDim.new(0, 2)
    
