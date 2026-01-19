-- Farm + Proximity (HoldDuration-only) + Ring parts + AutoGrab
repeat task.wait() until game:IsLoaded()

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ProximityPromptService = game:GetService("ProximityPromptService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local lp = Players.LocalPlayer
local reps = ReplicatedStorage
local rf = reps:FindFirstChild("RemoteFunctions") or reps.RemoteFunctions

-- Farming functions
local function upgradespeed()
    local remote = rf and rf:FindFirstChild("UpgradeSpeed")
    if remote then pcall(function() remote:InvokeServer(5) end) end
end

local function rebirth()
    local remote = rf and rf:FindFirstChild("Rebirth")
    if remote then pcall(function() remote:InvokeServer() end) end
end

local function upgrade()
    local remote = rf and rf:FindFirstChild("UpgradeBase")
    if remote then pcall(function() remote:InvokeServer() end) end
end

local function upgradec()
    local remote = rf and rf:FindFirstChild("UpgradeCarry")
    if remote then pcall(function() remote:InvokeServer() end) end
end

local function upgradeb(slot)
    local remote = rf and rf:FindFirstChild("UpgradeBrainrot")
    if remote then pcall(function() remote:InvokeServer(slot) end) end
end

local function getbase()
    local bases = Workspace:FindFirstChild("Bases")
    if not bases then return nil end
    for _, v in pairs(bases:GetChildren()) do
        if v:IsA("Model") and v.GetAttribute and v:GetAttribute("Holder") == lp.UserId then
            return v
        end
    end
    return nil
end

local function upgradeallb()
    local base = getbase()
    if not base then return end
    local slots = base:FindFirstChild("Slots")
    if not slots then return end
    for _, v in pairs(slots:GetChildren()) do
        if v:IsA("Model") and v.Name:lower():find("slot") and v:FindFirstChildWhichIsA("Tool") then
            pcall(function() upgradeb(v.Name) end)
            task.wait(0.1)
        end
    end
end

local function speedchanger()
    local speed = getgenv().Scv or 16
    pcall(function() lp:SetAttribute("CurrentSpeed", speed) end)
end

-- Optional safety: move to safe area if carry limit reached
local function checkCarryLimit()
    local plrgui = lp and lp:FindFirstChild("PlayerGui")
    if not plrgui then return end
    for _, v in pairs(plrgui:GetDescendants()) do
        if v:IsA("TextLabel") and v.Text == "Carry limit reached!" then
            local safearea = Workspace:FindFirstChild("Misc") and Workspace.Misc:FindFirstChild("Ground")
            if safearea and safearea:IsA("BasePart") and lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") then
                pcall(function()
                    lp.Character.HumanoidRootPart.CFrame = safearea.CFrame + CFrame.new(0, 5, 0)
                end)
            end
        end
    end
end

-- Proximity: only set HoldDuration = 0, preserve MaxActivationDistance
local ProxOriginals = {}
local ProxAddedConn

local function applyProxTo(prompt)
    if not prompt or not prompt:IsA("ProximityPrompt") then return end
    if ProxOriginals[prompt] == nil then
        pcall(function() ProxOriginals[prompt] = prompt.HoldDuration end)
    end
    pcall(function() prompt.HoldDuration = 0 end)
end

local function enableProx()
    for _, obj in pairs(Workspace:GetDescendants()) do
        if obj:IsA("ProximityPrompt") then
            applyProxTo(obj)
        end
    end
    if not ProxAddedConn then
        ProxAddedConn = Workspace.DescendantAdded:Connect(function(desc)
            if desc:IsA("ProximityPrompt") then
                applyProxTo(desc)
            end
        end)
    end
end

local function disableProx()
    if ProxAddedConn then
        ProxAddedConn:Disconnect()
        ProxAddedConn = nil
    end
    local list = {}
    for prompt in pairs(ProxOriginals) do table.insert(list, prompt) end
    for _, prompt in ipairs(list) do
        local orig = ProxOriginals[prompt]
        if prompt and prompt.Parent and orig ~= nil then
            pcall(function() prompt.HoldDuration = orig end)
        end
        ProxOriginals[prompt] = nil
    end
end

-- UI (WindUI)
local WindUILoader = "https://github.com/Footagesus/WindUI/releases/latest/download/main.lua"
local WindUI = loadstring(game:HttpGet(WindUILoader))()
WindUI:SetNotificationLower(true)

local Window = WindUI:CreateWindow({
    Title = "Escape Tsunami For Brainrots!",
    Icon = "sparkles",
    Author = "by DZ"
})
Window:SetToggleKey(Enum.KeyCode.L)

local Main = Window:Tab({ Title = "Farm", Icon = "sparkles", Locked = false })
Main:Select()

Main:Toggle({ Title = "Auto upgrade speed", Icon = "sparkles", Type = "Toggle", Value = false, Callback = function(state) getgenv().Aus = state end })
Main:Toggle({ Title = "Auto upgrade carry", Icon = "sparkles", Type = "Toggle", Value = false, Callback = function(state) getgenv().Auc = state end })
Main:Toggle({ Title = "Auto rebirth", Icon = "sparkles", Type = "Toggle", Value = false, Callback = function(state) getgenv().Ar = state end })
Main:Toggle({ Title = "Auto upgrade base", Icon = "sparkles", Type = "Toggle", Value = false, Callback = function(state) getgenv().Aub = state end })
Main:Toggle({ Title = "Update brainrots", Icon = "sparkles", Type = "Toggle", Value = false, Callback = function(state) getgenv().Aub2 = state end })
Main:Toggle({ Title = "Speed changer", Icon = "sparkles", Type = "Toggle", Value = false, Callback = function(state) getgenv().Sc = state end })
Main:Slider({ Title = "Speed", Step = 1, Value = { Min = 16, Max = 1000, Default = 16 }, Callback = function(value) getgenv().Scv = tonumber(value) or 16 end })

Main:Toggle({
    Title = "insta interactúe",
    Icon = "sparkles",
    Type = "Toggle",
    Value = false,
    Callback = function(state)
        getgenv().Proximity = state
        if state then pcall(enableProx) else pcall(disableProx) end
    end
})

-- Ring visual + AutoGrab (ignores specific patterns)
local SEGMENTS = 36
local FILL_FACTOR = 0.6
local HEIGHT = 0.12
local THICKNESS = 0.25
local Y_OFFSET = 0.12
local COLOR = Color3.fromRGB(0, 170, 255)
local TRANSPARENCY = 0.35

local BLOCK_PATTERNS = {
    "sell brainrot",
    "colocar brainrot",
    "steal brainrot",
    "recoge brainrot"
}

local FIRE_COOLDOWN = 0.9
local MAX_RETRIES = 6
local RETRY_DELAY = 0.12

local ringParts = {}
local char = lp and lp.Character
local hrp = char and char:FindFirstChild("HumanoidRootPart")
local lastFired = {}
local RingRenderConn, RingHeartbeatConn, PromptShownConn, CharAddedConn

local function normalize(str)
    if not str then return "" end
    local ok, s = pcall(function() return tostring(str):lower() end)
    if not ok then return "" end
    return s:gsub("%s+", " ")
end

local function createRingParts()
    if #ringParts > 0 then return end
    for i = 1, SEGMENTS do
        local part = Instance.new("Part")
        part.Name = "DZ_ReachRingSeg"
        part.Anchored = true
        part.CanCollide = false
        part.CanTouch = false
        part.CanQuery = false
        part.Size = Vector3.new(1, HEIGHT, THICKNESS)
        part.Material = Enum.Material.Neon
        part.Color = COLOR
        part.Transparency = TRANSPARENCY
        part.Parent = Workspace
        table.insert(ringParts, part)
    end
end

local function destroyRingParts()
    for _, part in ipairs(ringParts) do
        if part and part.Parent then part:Destroy() end
    end
    ringParts = {}
end

local function getCurrentReach()
    local maxd = 0
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("ProximityPrompt") then
            local ok, d = pcall(function() return v.MaxActivationDistance end)
            d = (ok and d) and d or 0
            if d > maxd then maxd = d end
        end
    end
    return maxd
end

local function updateRing(hrpPos, radius)
    if not radius or radius <= 0.01 then radius = 0.5 end
    local angleStep = (2 * math.pi) / SEGMENTS
    for i, part in ipairs(ringParts) do
        if not part then break end
        local theta = (i - 1) * angleStep
        local arcLen = radius * angleStep
        local segLen = math.max(0.001, arcLen * FILL_FACTOR)
        local px = hrpPos.X + math.cos(theta) * radius
        local pz = hrpPos.Z + math.sin(theta) * radius
        local py = hrpPos.Y + Y_OFFSET
        part.Size = Vector3.new(segLen, HEIGHT, THICKNESS)
        local look = CFrame.new(Vector3.new(px, py, pz), Vector3.new(hrpPos.X, py, hrpPos.Z))
        part.CFrame = look * CFrame.Angles(0, math.pi/2, 0)
    end
end

local function onCharacterAdded(c)
    destroyRingParts()
    char = c
    hrp = char:WaitForChild("HumanoidRootPart", 6)
    createRingParts()
end

local function isBlockedPrompt(prompt)
    if not prompt then return false end
    local fields = {}
    pcall(function() table.insert(fields, prompt.Name) end)
    pcall(function() table.insert(fields, prompt.ActionText) end)
    pcall(function() table.insert(fields, prompt.ObjectText) end)
    pcall(function() if prompt.Parent then table.insert(fields, prompt.Parent.Name) end end)
    pcall(function() if prompt.Parent and prompt.Parent.Parent then table.insert(fields, prompt.Parent.Parent.Name) end end)
    for _, raw in ipairs(fields) do
        local s = normalize(raw)
        for _, pat in ipairs(BLOCK_PATTERNS) do
            if s:find(pat, 1, true) then return true end
        end
    end
    return false
end

local function safeFirePrompt(prompt)
    if not prompt then return end
    if isBlockedPrompt(prompt) then return end
    local now = tick()
    if lastFired[prompt] and (now - lastFired[prompt]) < FIRE_COOLDOWN then return end
    lastFired[prompt] = now
    pcall(function() fireproximityprompt(prompt) end)
    spawn(function()
        for i = 1, MAX_RETRIES do
            task.wait(RETRY_DELAY)
            if not prompt or not prompt.Parent or not prompt:IsDescendantOf(game) then break end
            local ok, enabled = pcall(function() return prompt.Enabled end)
            if not ok or not enabled then break end
            pcall(function() fireproximityprompt(prompt) end)
        end
    end)
end

local function heartbeatCleanup()
    for k,_ in pairs(lastFired) do
        if not k or (type(k) == "userdata" and not k:IsDescendantOf(game)) then
            lastFired[k] = nil
        end
    end
end

local function renderUpdate()
    if not hrp or not hrp.Parent then
        if lp and lp.Character then hrp = lp.Character:FindFirstChild("HumanoidRootPart") end
        return
    end
    local reach = getCurrentReach()
    updateRing(hrp.Position, reach)
end

local function promptShownHandler(prompt)
    if not prompt then return end
    safeFirePrompt(prompt)
end

local function enableRingAutoGrab()
    if RingRenderConn or PromptShownConn or RingHeartbeatConn or CharAddedConn then return end
    char = lp and lp.Character
    if char then hrp = char:FindFirstChild("HumanoidRootPart") end
    createRingParts()
    CharAddedConn = lp.CharacterAdded:Connect(onCharacterAdded)
    RingRenderConn = RunService.RenderStepped:Connect(renderUpdate)
    PromptShownConn = ProximityPromptService.PromptShown:Connect(promptShownHandler)
    RingHeartbeatConn = RunService.Heartbeat:Connect(heartbeatCleanup)
end

local function disableRingAutoGrab()
    if PromptShownConn then PromptShownConn:Disconnect() PromptShownConn = nil end
    if RingRenderConn then RingRenderConn:Disconnect() RingRenderConn = nil end
    if RingHeartbeatConn then RingHeartbeatConn:Disconnect() RingHeartbeatConn = nil end
    if CharAddedConn then CharAddedConn:Disconnect() CharAddedConn = nil end
    destroyRingParts()
    lastFired = {}
    hrp = nil
    char = lp and lp.Character
end

Main:Toggle({
    Title = "Auto grab [beta]",
    Icon = "sparkles",
    Type = "Toggle",
    Value = false,
    Callback = function(state)
        getgenv().AutoGrabBeta = state
        if state then pcall(enableRingAutoGrab) else pcall(disableRingAutoGrab) end
    end
})

-- Main loop (farming)
spawn(function()
    while true do
        task.wait(0.6)
        if getgenv().Aus then spawn(function() pcall(upgradespeed) end) end
        if getgenv().Auc then spawn(function() pcall(upgradec) end) end
        if getgenv().Ar then spawn(function() pcall(rebirth) end) end
        if getgenv().Aub then spawn(function() pcall(upgrade) end) end
        if getgenv().Aub2 then spawn(function() pcall(upgradeallb) end) end
        if getgenv().Sc then spawn(function() pcall(speedchanger) end) end
    end
end)

-- Cleanup on close
game:BindToClose(function()
    pcall(disableRingAutoGrab)
    pcall(disableProx)
    destroyRingParts()
end)

print("Script cargado: Proximity HoldDuration-only, Ring y AutoGrab integrados")
