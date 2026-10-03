print("[HV] Script started")
local Players = game:GetService("Players")
print("[HV] Players service loaded")
local RS = game:GetService("ReplicatedStorage")
print("[HV] RS service loaded")
local UIS = game:GetService("UserInputService")
print("[HV] UIS service loaded")
local TweenService = game:GetService("TweenService")
print("[HV] TweenService loaded")
local HttpService = game:GetService("HttpService")
print("[HV] HttpService loaded")

local LP = Players.LocalPlayer
print("[HV] LocalPlayer:", LP)
local PlayerGui = LP:WaitForChild("PlayerGui")
print("[HV] PlayerGui loaded")
-- ========== PANDA ==========
local PANDA_SERVICE = "hussvalleys"
local keyOk = false
local PUSL = nil

local function loadPanda()
    local ok, lib = pcall(function()
        return loadstring(game:HttpGet("https://secure.pandauth.com/pv4/lib"))()
    end)
    if ok and lib then
        PUSL = lib
        PUSL.configure({ serviceId = PANDA_SERVICE })
        return true
    end
    return false
end

local function getKeyUrl()
    if not PUSL then loadPanda() end
    if PUSL and PUSL.getKeyUrl then return PUSL.getKeyUrl() end
    return "https://pandadevelopment.net"
end

local function validateKey(key)
    if not key or key == "" then return false, "empty key" end
    key = key:gsub("%s+", "")
    if not PUSL and not loadPanda() then return false, "panda load failed" end
    local result = PUSL.validate(key)
    if result and result.success then
        keyOk = true
        pcall(function()
            if writefile then writefile("hv_panda_key.txt", key) end
        end)
        return true, "ok"
    end
    return false, "invalid key"
end

task.spawn(function()
    loadPanda()
    pcall(function()
        if readfile then
            local saved = readfile("hv_panda_key.txt")
            if saved and saved ~= "" then validateKey(saved) end
        end
    end)
    if getgenv and getgenv().HVKey then validateKey(getgenv().HVKey) end
end)

-- ========== STATE ==========
local State = {
    Speed = false,
    InfDash = false,
    CatchTP = false,
}

local TARGET_MAX = 40
local SPEED_GATE = 28
local CATCH_RANGE = 120
local CATCH_COOLDOWN = 0.28
local LEAD_TIME = 0.18
local LEAD_EXTRA = 3
local lastCatch = 0

local function getHRP()
    local char = LP.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function flat(v)
    return Vector3.new(v.X, 0, v.Z)
end

local function getGameAction()
    local ok, ev = pcall(function()
        return RS.ChickenOrHero.Game.GameAction
    end)
    return ok and ev or nil
end

-- ========== MOVEMENT ==========
local function setupMovement()
    local ok, Movement = pcall(function()
        return RS:WaitForChild("ChickenOrHero"):WaitForChild("Movement")
    end)
    if not ok then return end

    local Model = require(Movement:WaitForChild("MovementModel"))

    local oldCanBoost = Model.canBoost
    Model.canBoost = function(state, profile, onGround)
        if State.InfDash and keyOk then
            if state then
                state.cooldown = 0
                state.recoveryRemaining = 0
                state.armed = true
            end
            return true
        end
        return oldCanBoost(state, profile, onGround)
    end

    local oldStep = Model.step
    Model.step = function(state, ...)
        if State.InfDash and keyOk and state then
            state.cooldown = 0
            state.recoveryRemaining = 0
        end
        local result = oldStep(state, ...)
        if State.Speed and keyOk and state and state.outputSpeed then
            local s = state.outputSpeed
            if s >= SPEED_GATE then
                state.outputSpeed = math.min(s * 1.08, TARGET_MAX)
                if state.speed then
                    state.speed = math.min(state.speed * 1.08, TARGET_MAX)
                end
            end
        end
        return result
    end
end

task.spawn(setupMovement)

-- ========== CATCH ==========
local function getNearestRunner()
    local myHRP = getHRP()
    if not myHRP then return nil end
    local best, bestDist = nil, CATCH_RANGE
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and plr:GetAttribute("GameRole") == "Runner" then
            local char = plr.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hrp and hum and hum.Health > 0 then
                local st = plr:GetAttribute("RunState") or char:GetAttribute("RunState")
                if st ~= "Caught" and st ~= "Safe" then
                    local d = (hrp.Position - myHRP.Position).Magnitude
                    if d < bestDist then
                        bestDist = d
                        best = hrp
                    end
                end
            end
        end
    end
    return best
end

local function fireTackle(myHRP, target)
    local dir = flat(target.Position - myHRP.Position)
    if dir.Magnitude < 0.05 then dir = flat(target.CFrame.LookVector) end
    if dir.Magnitude > 0.05 then dir = dir.Unit end
    local action = getGameAction()
    if not action then return end
    pcall(function()
        action:FireServer("Tackle", {
            id = HttpService:GenerateGUID(false),
            at = workspace:GetServerTimeNow(),
            speed = math.max(flat(myHRP.AssemblyLinearVelocity).Magnitude, 28),
            direction = dir,
            position = myHRP.Position,
        })
    end)
end

local function predictPos(target)
    local vel = target.AssemblyLinearVelocity
    local hv = Vector3.new(vel.X, 0, vel.Z)
    local pred = target.Position + hv * LEAD_TIME
    if hv.Magnitude > 1 then pred = pred + hv.Unit * LEAD_EXTRA end
    return Vector3.new(pred.X, target.Position.Y, pred.Z)
end

local function doCatch()
    if not State.CatchTP or not keyOk then return end
    if tick() - lastCatch < CATCH_COOLDOWN then return end
    local myHRP = getHRP()
    if not myHRP then return end
    local target = getNearestRunner()
    if not target then return end
    lastCatch = tick()
    myHRP.CFrame = CFrame.new(predictPos(target))
    task.spawn(function()
        for i = 1, 4 do
            task.wait(0.025)
            local hrp = getHRP()
            if not hrp or not target or not target.Parent then break end
            hrp.CFrame = CFrame.new(predictPos(target))
            fireTackle(hrp, target)
        end
    end)
end

UIS.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.F then
        doCatch()
    elseif input.KeyCode == Enum.KeyCode.Space and State.CatchTP and keyOk then
        if LP:GetAttribute("GameRole") == "Catcher" then
            doCatch()
        end
    end
end)

-- ========== UI ==========
local function makeUI()
    local old = PlayerGui:FindFirstChild("HV_Assist")
    if old then old:Destroy() end

    local sg = Instance.new("ScreenGui")
    sg.Name = "HV_Assist"
    sg.ResetOnSpawn = false
    sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    sg.Parent = PlayerGui

    local openBtn = Instance.new("TextButton")
    openBtn.Size = UDim2.new(0, 48, 0, 48)
    openBtn.Position = UDim2.new(0, 16, 0.5, -24)
    openBtn.BackgroundColor3 = Color3.fromRGB(28, 28, 34)
    openBtn.Text = "HV"
    openBtn.TextColor3 = Color3.fromRGB(230, 230, 240)
    openBtn.Font = Enum.Font.GothamBold
    openBtn.TextSize = 15
    openBtn.Parent = sg
    Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0, 12)

    local panel = Instance.new("Frame")
    panel.Size = UDim2.new(0, 380, 0, 420)
    panel.Position = UDim2.new(0.5, -190, 0.5, -210)
    panel.BackgroundColor3 = Color3.fromRGB(20, 20, 24)
    panel.BackgroundTransparency = 0.06
    panel.Parent = sg
    Instance.new("UICorner", panel).CornerRadius = UDim.new(0, 14)
    local ps = Instance.new("UIStroke", panel)
    ps.Color = Color3.fromRGB(55, 55, 70)
    ps.Transparency = 0.4

    -- DRAG
    local header = Instance.new("Frame")
    header.Size = UDim2.new(1, 0, 0, 50)
    header.BackgroundTransparency = 1
    header.Parent = panel

    local dragging, dragStart, startPos
    header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = panel.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    header.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            if dragging then
                local delta = input.Position - dragStart
                panel.Position = UDim2.new(
                    startPos.X.Scale, startPos.X.Offset + delta.X,
                    startPos.Y.Scale, startPos.Y.Offset + delta.Y
                )
            end
        end
    end)

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -60, 1, 0)
    title.Position = UDim2.new(0, 18, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "Huss Valley Assist"
    title.TextColor3 = Color3.fromRGB(240, 240, 250)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = header

    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 32, 0, 32)
    closeBtn.Position = UDim2.new(1, -44, 0.5, -16)
    closeBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 48)
    closeBtn.Text = "✕"
    closeBtn.TextColor3 = Color3.fromRGB(180, 180, 195)
    closeBtn.Font = Enum.Font.Gotham
    closeBtn.TextSize = 16
    closeBtn.Parent = header
    Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 8)

    -- AUTH PAGE
    local authPage = Instance.new("Frame")
    authPage.Size = UDim2.new(1, -32, 1, -66)
    authPage.Position = UDim2.new(0, 16, 0, 54)
    authPage.BackgroundTransparency = 1
    authPage.Parent = panel

    local authCard = Instance.new("Frame")
    authCard.Size = UDim2.new(1, 0, 0, 200)
    authCard.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
    authCard.Parent = authPage
    Instance.new("UICorner", authCard).CornerRadius = UDim.new(0, 10)

    local aTitle = Instance.new("TextLabel")
    aTitle.Size = UDim2.new(1, -24, 0, 24)
    aTitle.Position = UDim2.new(0, 14, 0, 14)
    aTitle.BackgroundTransparency = 1
    aTitle.Text = "Panda Auth"
    aTitle.TextColor3 = Color3.fromRGB(235, 235, 245)
    aTitle.Font = Enum.Font.GothamMedium
    aTitle.TextSize = 15
    aTitle.TextXAlignment = Enum.TextXAlignment.Left
    aTitle.Parent = authCard

    local aDesc = Instance.new("TextLabel")
    aDesc.Size = UDim2.new(1, -24, 0, 36)
    aDesc.Position = UDim2.new(0, 14, 0, 40)
    aDesc.BackgroundTransparency = 1
    aDesc.Text = "Get a key, paste it below, then authenticate to unlock."
    aDesc.TextColor3 = Color3.fromRGB(140, 140, 160)
    aDesc.Font = Enum.Font.Gotham
    aDesc.TextSize = 12
    aDesc.TextWrapped = true
    aDesc.TextXAlignment = Enum.TextXAlignment.Left
    aDesc.Parent = authCard

    local keyBox = Instance.new("TextBox")
    keyBox.Size = UDim2.new(1, -28, 0, 34)
    keyBox.Position = UDim2.new(0, 14, 0, 86)
    keyBox.BackgroundColor3 = Color3.fromRGB(16, 16, 20)
    keyBox.PlaceholderText = "Paste key here..."
    keyBox.Text = ""
    keyBox.TextColor3 = Color3.fromRGB(230, 230, 240)
    keyBox.PlaceholderColor3 = Color3.fromRGB(100, 100, 120)
    keyBox.Font = Enum.Font.Gotham
    keyBox.TextSize = 13
    keyBox.ClearTextOnFocus = false
    keyBox.Parent = authCard
    Instance.new("UICorner", keyBox).CornerRadius = UDim.new(0, 6)

    local getKeyBtn = Instance.new("TextButton")
    getKeyBtn.Size = UDim2.new(0.48, -6, 0, 34)
    getKeyBtn.Position = UDim2.new(0, 14, 0, 132)
    getKeyBtn.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
    getKeyBtn.Text = "Get Key"
    getKeyBtn.TextColor3 = Color3.fromRGB(230, 230, 240)
    getKeyBtn.Font = Enum.Font.GothamMedium
    getKeyBtn.TextSize = 13
    getKeyBtn.Parent = authCard
    Instance.new("UICorner", getKeyBtn).CornerRadius = UDim.new(0, 6)

    local authBtn = Instance.new("TextButton")
    authBtn.Size = UDim2.new(0.48, -6, 0, 34)
    authBtn.Position = UDim2.new(0.52, 0, 0, 132)
    authBtn.BackgroundColor3 = Color3.fromRGB(50, 130, 210)
    authBtn.Text = "Authenticate"
    authBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    authBtn.Font = Enum.Font.GothamMedium
    authBtn.TextSize = 13
    authBtn.Parent = authCard
    Instance.new("UICorner", authBtn).CornerRadius = UDim.new(0, 6)

    local aStatus = Instance.new("TextLabel")
    aStatus.Size = UDim2.new(1, -28, 0, 18)
    aStatus.Position = UDim2.new(0, 14, 0, 174)
    aStatus.BackgroundTransparency = 1
    aStatus.Text = ""
    aStatus.TextColor3 = Color3.fromRGB(200, 140, 100)
    aStatus.Font = Enum.Font.Gotham
    aStatus.TextSize = 12
    aStatus.TextXAlignment = Enum.TextXAlignment.Left
    aStatus.Parent = authCard

    getKeyBtn.MouseButton1Click:Connect(function()
        local url = getKeyUrl()
        pcall(function() if setclipboard then setclipboard(url) end end)
        aStatus.Text = "Link copied — open in browser"
        aStatus.TextColor3 = Color3.fromRGB(120, 200, 140)
        print("[Panda] " .. tostring(url))
    end)

    -- FEATURES PAGE
    local featPage = Instance.new("ScrollingFrame")
    featPage.Size = UDim2.new(1, -32, 1, -66)
    featPage.Position = UDim2.new(0, 16, 0, 54)
    featPage.BackgroundTransparency = 1
    featPage.BorderSizePixel = 0
    featPage.ScrollBarThickness = 4
    featPage.Visible = false
    featPage.CanvasSize = UDim2.new(0, 0, 0, 480)
    featPage.Parent = panel

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 14)
    layout.Parent = featPage

    local function addFeature(name, desc, onToggle)
        local card = Instance.new("Frame")
        card.Size = UDim2.new(1, -4, 0, 96)
        card.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
        card.Parent = featPage
        Instance.new("UICorner", card).CornerRadius = UDim.new(0, 10)

        local n = Instance.new("TextLabel")
        n.Size = UDim2.new(1, -70, 0, 22)
        n.Position = UDim2.new(0, 14, 0, 12)
        n.BackgroundTransparency = 1
        n.Text = name
        n.TextColor3 = Color3.fromRGB(235, 235, 245)
        n.Font = Enum.Font.GothamMedium
        n.TextSize = 14
        n.TextXAlignment = Enum.TextXAlignment.Left
        n.Parent = card

        local d = Instance.new("TextLabel")
        d.Size = UDim2.new(1, -28, 0, 48)
        d.Position = UDim2.new(0, 14, 0, 38)
        d.BackgroundTransparency = 1
        d.Text = desc
        d.TextColor3 = Color3.fromRGB(140, 140, 160)
        d.Font = Enum.Font.Gotham
        d.TextSize = 12
        d.TextWrapped = true
        d.TextXAlignment = Enum.TextXAlignment.Left
        d.TextYAlignment = Enum.TextYAlignment.Top
        d.Parent = card

        local toggle = Instance.new("TextButton")
        toggle.Size = UDim2.new(0, 46, 0, 26)
        toggle.Position = UDim2.new(1, -58, 0, 12)
        toggle.BackgroundColor3 = Color3.fromRGB(55, 55, 65)
        toggle.Text = ""
        toggle.Parent = card
        Instance.new("UICorner", toggle).CornerRadius = UDim.new(1, 0)

        local knob = Instance.new("Frame")
        knob.Size = UDim2.new(0, 20, 0, 20)
        knob.Position = UDim2.new(0, 3, 0.5, -10)
        knob.BackgroundColor3 = Color3.fromRGB(240, 240, 245)
        knob.Parent = toggle
        Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

        local on = false
        toggle.MouseButton1Click:Connect(function()
            on = not on
            local tw = TweenInfo.new(0.15, Enum.EasingStyle.Quad)
            TweenService:Create(toggle, tw, {
                BackgroundColor3 = on and Color3.fromRGB(50, 160, 90) or Color3.fromRGB(55, 55, 65)
            }):Play()
            TweenService:Create(knob, tw, {
                Position = on and UDim2.new(1, -23, 0.5, -10) or UDim2.new(0, 3, 0.5, -10)
            }):Play()
            onToggle(on)
        end)
        return card
    end

    -- SPEED card + slider
    local speedCard = addFeature("Speed Upgrade",
        "Raises max speed. Use with Inf Dash so you don't get snapped.",
        function(v) State.Speed = v end)

    speedCard.Size = UDim2.new(1, -4, 0, 150)

    local sliderLabel = Instance.new("TextLabel")
    sliderLabel.Size = UDim2.new(1, -28, 0, 18)
    sliderLabel.Position = UDim2.new(0, 14, 0, 92)
    sliderLabel.BackgroundTransparency = 1
    sliderLabel.Text = "Max Speed: " .. TARGET_MAX
    sliderLabel.TextColor3 = Color3.fromRGB(180, 180, 200)
    sliderLabel.Font = Enum.Font.Gotham
    sliderLabel.TextSize = 12
    sliderLabel.TextXAlignment = Enum.TextXAlignment.Left
    sliderLabel.Parent = speedCard

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, -28, 0, 8)
    track.Position = UDim2.new(0, 14, 0, 118)
    track.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
    track.Parent = speedCard
    Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((TARGET_MAX - 34) / (50 - 34), 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(50, 140, 220)
    fill.Parent = track
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

    local knobBtn = Instance.new("TextButton")
    knobBtn.Size = UDim2.new(0, 16, 0, 16)
    knobBtn.Position = UDim2.new((TARGET_MAX - 34) / (50 - 34), -8, 0.5, -8)
    knobBtn.BackgroundColor3 = Color3.fromRGB(240, 240, 250)
    knobBtn.Text = ""
    knobBtn.Parent = track
    Instance.new("UICorner", knobBtn).CornerRadius = UDim.new(1, 0)

    local sliding = false
    local minSp, maxSp = 34, 50

    local function setSlider(alpha)
        alpha = math.clamp(alpha, 0, 1)
        TARGET_MAX = math.floor(minSp + (maxSp - minSp) * alpha + 0.5)
        fill.Size = UDim2.new(alpha, 0, 1, 0)
        knobBtn.Position = UDim2.new(alpha, -8, 0.5, -8)
        sliderLabel.Text = "Max Speed: " .. TARGET_MAX
    end

    knobBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            sliding = true
        end
    end)
    track.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            sliding = true
            local rel = (input.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X
            setSlider(rel)
        end
    end)
    UIS.InputChanged:Connect(function(input)
        if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local rel = (input.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X
            setSlider(rel)
        end
    end)
    UIS.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            sliding = false
        end
    end)

    addFeature("Inf Dash",
        "No dash cooldown. Great for dodging and for higher speed without snaps.",
        function(v) State.InfDash = v end)

    addFeature("Strong Catch",
        "Press F to TP in front of enemies. Spam Space as Catcher — burst tackle.",
        function(v) State.CatchTP = v end)

    layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        featPage.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 20)
    end)

    local function showFeatures()
        authPage.Visible = false
        featPage.Visible = true
    end

    authBtn.MouseButton1Click:Connect(function()
        local ok, msg = validateKey(keyBox.Text)
        if ok then
            aStatus.Text = "Authenticated"
            aStatus.TextColor3 = Color3.fromRGB(100, 200, 130)
            task.wait(0.3)
            showFeatures()
        else
            aStatus.Text = msg
            aStatus.TextColor3 = Color3.fromRGB(220, 120, 100)
        end
    end)

    task.spawn(function()
        task.wait(0.6)
        if keyOk then showFeatures() end
    end)

    local uiOpen = true
    openBtn.MouseButton1Click:Connect(function()
        uiOpen = not uiOpen
        panel.Visible = uiOpen
    end)
    closeBtn.MouseButton1Click:Connect(function()
        uiOpen = false
        panel.Visible = false
    end)
end

makeUI()
print("HV Assist · draggable + speed slider")
