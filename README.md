-- Services  
local Players = game:GetService("Players")  
local RunService = game:GetService("RunService")  
local UserInputService = game:GetService("UserInputService")  
local TextService = game:GetService("TextService")  
  
-- Player reference  
local player = Players.LocalPlayer  
  
-- Script State  
local ScriptEnabled = false
local originalSizes = {}  
local auraConnection = nil  
local antiRagdollConnection = nil  
local antiFlingConnection = nil  
local lastAttackTime = 0  
local targetPlayerNames = ""  
local headSitPlayerNames = ""
local avoidPlayerNames = ""  
local targetList = {}  
local headSitTargetList = {}
local avoidList = {}  
local isCharacterSafe = true  
  
-- Configuration
local attacksPerSecond = 100000
local attackCooldown = 0  
local attackRange = 50 
  
-- Remote paths  
local HitRemote = game:GetService("ReplicatedStorage")  
    :WaitForChild("Packages")  
    :WaitForChild("Knit")  
    :WaitForChild("Services")  
    :WaitForChild("CombatService")  
    :WaitForChild("RF")  
    :WaitForChild("Hit")  
  

-- ==================== HELPER FUNCTIONS ====================
local function updateTargetList()
    targetList = {}
    if targetPlayerNames == "" then return end
    for name in targetPlayerNames:gmatch("[^,^]+") do
        local trimmedName = name:gsub("^%s*(.-)%s*$", "%1")
        if trimmedName ~= "" then
            targetList[trimmedName:lower()] = true
        end
    end
end

local function updateHeadSitTargetList()
    headSitTargetList = {}
    if headSitPlayerNames == "" then return end
    for name in headSitPlayerNames:gmatch("[^,^]+") do
        local trimmedName = name:gsub("^%s*(.-)%s*$", "%1")
        if trimmedName ~= "" then
            headSitTargetList[trimmedName:lower()] = true
        end
    end
end

local function updateAvoidList()
    avoidList = {}
    if avoidPlayerNames == "" then return end
    for name in avoidPlayerNames:gmatch("[^,^]+") do
        local trimmedName = name:gsub("^%s*(.-)%s*$", "%1")
        if trimmedName ~= "" then
            avoidList[trimmedName:lower()] = true
        end
    end
end

-- ==================== ANTI-FLING SYSTEM ====================  
local function setupAntiFling()  
    if antiFlingConnection then antiFlingConnection:Disconnect() end  
    antiFlingConnection = RunService.Heartbeat:Connect(function()  
        if not player.Character then return end  
        local humanoidRootPart = player.Character:FindFirstChild("HumanoidRootPart")  
        if not humanoidRootPart then return end  
        local velocity = humanoidRootPart.Velocity  
        local speed = velocity.Magnitude  
        if speed > 100 then  
            humanoidRootPart.Velocity = velocity * 0.8  
            local antiForce = -velocity.Unit * math.min(speed * 0.5, 200)  
            humanoidRootPart.Velocity = humanoidRootPart.Velocity + antiForce  
        end  
        local angularVelocity = humanoidRootPart.AssemblyAngularVelocity  
        if angularVelocity.Magnitude > 20 then  
            humanoidRootPart.AssemblyAngularVelocity = Vector3.new(0, 0, 0)  
        end  
    end)  
end  
  
-- ==================== ANTI-RAGDOLL SYSTEM ====================  
local function setupAntiRagdoll()  
    if antiRagdollConnection then antiRagdollConnection:Disconnect() end  
    antiRagdollConnection = RunService.Heartbeat:Connect(function()  
        if not player.Character then return end  
        local humanoid = player.Character:FindFirstChildOfClass("Humanoid")  
        if not humanoid then return end  
        if humanoid:GetState() == Enum.HumanoidStateType.FallingDown or   
           humanoid:GetState() == Enum.HumanoidStateType.Ragdoll then  
            humanoid:ChangeState(Enum.HumanoidStateType.Running)  
        end  
        if humanoid.Health <= 0 then  
            humanoid.Health = 1  
        end  
    end)  
end  
  
-- ==================== GRAB/FALL FIXES ====================  
local function setupGrabFallFixes()  
    local character = player.Character  
    if not character then return end  
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")  
    if not humanoidRootPart then return end  
    local lastYPosition = humanoidRootPart.Position.Y  
    local fallStartTime = 0  
    local isFalling = false  
    RunService.Heartbeat:Connect(function()  
        if not character or not humanoidRootPart then return end  
        local currentY = humanoidRootPart.Position.Y  
        local velocity = humanoidRootPart.Velocity.Y  
        if velocity < -50 and currentY < lastYPosition then  
            if not isFalling then  
                isFalling = true  
                fallStartTime = tick()  
            end  
            if tick() - fallStartTime > 0.5 then  
                humanoidRootPart.Velocity = Vector3.new(  
                    humanoidRootPart.Velocity.X,  
                    math.max(velocity, -50),  
                    humanoidRootPart.Velocity.Z  
                )  
                if currentY > 1000 then  
                    humanoidRootPart.CFrame = CFrame.new(  
                        humanoidRootPart.Position.X,  
                        100,  
                        humanoidRootPart.Position.Z  
                    )  
                end  
            end  
        else  
            isFalling = false  
        end  
        lastYPosition = currentY  
        local humanoid = character:FindFirstChildOfClass("Humanoid")  
        if humanoid then  
            if humanoid.PlatformStand then humanoid.PlatformStand = false end  
            if humanoid.Sit then humanoid.Sit = false end  
            humanoid:ChangeState(Enum.HumanoidStateType.Running)  
        end  
    end)  
end  
  
-- ==================== CHARACTER SAFETY SYSTEM ====================  
local function ensureCharacterSafety()  
    if not player.Character then return end  
    local humanoid = player.Character:FindFirstChildOfClass("Humanoid")  
    if humanoid then  
        if humanoid.Health <= 0 then humanoid.Health = 100 end  
        humanoid.AutoRotate = true  
        humanoid.AutoJumpEnabled = true  
        if humanoid.WalkSpeed < 16 then humanoid.WalkSpeed = 16 end  
    end  
end  


-- ==================== PROFESSIONAL GUI CREATOR ====================
local function CreateProfessionalGUI()
    local player = game.Players.LocalPlayer
    local TweenService = game:GetService("TweenService")
    local UserInputService = game:GetService("UserInputService")

    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "PAIN_MODIFIED_GUI"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ScreenGui.IgnoreGuiInset = true

    local MainFrame = Instance.new("Frame")
    MainFrame.Size = UDim2.new(0, 420, 0, 620)
    MainFrame.Position = UDim2.new(0.5, -210, 0.5, -310)
    MainFrame.BackgroundColor3 = Color3.fromRGB(8, 8, 14)
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = false
    MainFrame.Parent = ScreenGui
    Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 24)

    -- Outer glow frame
    local GlowFrame = Instance.new("Frame")
    GlowFrame.Size = UDim2.new(1, 20, 1, 20)
    GlowFrame.Position = UDim2.new(0, -10, 0, -10)
    GlowFrame.BackgroundColor3 = Color3.fromRGB(40, 80, 200)
    GlowFrame.BackgroundTransparency = 0.82
    GlowFrame.BorderSizePixel = 0
    GlowFrame.ZIndex = 0
    GlowFrame.Parent = MainFrame
    Instance.new("UICorner", GlowFrame).CornerRadius = UDim.new(0, 32)

    -- Inner clipping frame
    local InnerClip = Instance.new("Frame")
    InnerClip.Size = UDim2.new(1, 0, 1, 0)
    InnerClip.BackgroundTransparency = 1
    InnerClip.ClipsDescendants = true
    InnerClip.ZIndex = 1
    InnerClip.Parent = MainFrame
    Instance.new("UICorner", InnerClip).CornerRadius = UDim.new(0, 24)

    local BgGrad = Instance.new("UIGradient")
    BgGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0,   Color3.fromRGB(16, 16, 28)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(10, 10, 18)),
        ColorSequenceKeypoint.new(1,   Color3.fromRGB(6,  6,  12))
    }
    BgGrad.Rotation = 135
    BgGrad.Parent = InnerClip

    -- Top accent bar
    local TopBar = Instance.new("Frame")
    TopBar.Size = UDim2.new(1, 0, 0, 3)
    TopBar.BackgroundColor3 = Color3.fromRGB(80, 140, 255)
    TopBar.BorderSizePixel = 0
    TopBar.ZIndex = 5
    TopBar.Parent = InnerClip
    Instance.new("UICorner", TopBar).CornerRadius = UDim.new(0, 4)
    local TopBarGrad = Instance.new("UIGradient", TopBar)
    TopBarGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0,    Color3.fromRGB(0,0,0)),
        ColorSequenceKeypoint.new(0.15, Color3.fromRGB(60,120,255)),
        ColorSequenceKeypoint.new(0.5,  Color3.fromRGB(160,80,255)),
        ColorSequenceKeypoint.new(0.85, Color3.fromRGB(60,120,255)),
        ColorSequenceKeypoint.new(1,    Color3.fromRGB(0,0,0))
    }

    -- Border stroke
    local BorderStroke = Instance.new("UIStroke")
    BorderStroke.Color = Color3.fromRGB(50, 80, 180)
    BorderStroke.Transparency = 0.5
    BorderStroke.Thickness = 1.5
    BorderStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    BorderStroke.Parent = MainFrame

    -- ===== TITLE BAR =====
    local TitleBar = Instance.new("Frame")
    TitleBar.Size = UDim2.new(1, 0, 0, 58)
    TitleBar.BackgroundColor3 = Color3.fromRGB(12, 12, 22)
    TitleBar.BorderSizePixel = 0
    TitleBar.ZIndex = 2
    TitleBar.Parent = InnerClip
    Instance.new("UICorner", TitleBar).CornerRadius = UDim.new(0, 24)
    local TitleBarGrad = Instance.new("UIGradient", TitleBar)
    TitleBarGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 20, 38)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(8,  8,  16))
    }
    TitleBarGrad.Rotation = 90

    local TitleSep = Instance.new("Frame")
    TitleSep.Size = UDim2.new(1, 0, 0, 1)
    TitleSep.Position = UDim2.new(0, 0, 1, -1)
    TitleSep.BackgroundColor3 = Color3.fromRGB(255,255,255)
    TitleSep.BorderSizePixel = 0
    TitleSep.ZIndex = 3
    TitleSep.Parent = TitleBar
    local sepGrad = Instance.new("UIGradient", TitleSep)
    sepGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0,   Color3.fromRGB(0,0,0)),
        ColorSequenceKeypoint.new(0.2, Color3.fromRGB(60,120,255)),
        ColorSequenceKeypoint.new(0.8, Color3.fromRGB(140,60,255)),
        ColorSequenceKeypoint.new(1,   Color3.fromRGB(0,0,0))
    }

    local TitleDot = Instance.new("Frame")
    TitleDot.Size = UDim2.new(0, 9, 0, 9)
    TitleDot.Position = UDim2.new(0, 16, 0.45, -4)
    TitleDot.BackgroundColor3 = Color3.fromRGB(80, 180, 255)
    TitleDot.BorderSizePixel = 0
    TitleDot.ZIndex = 3
    TitleDot.Parent = TitleBar
    Instance.new("UICorner", TitleDot).CornerRadius = UDim.new(1, 0)

    local Title = Instance.new("TextLabel")
    Title.Size = UDim2.new(1, -120, 0, 28)
    Title.Position = UDim2.new(0, 32, 0, 8)
    Title.BackgroundTransparency = 1
    Title.Text = "FROSTY"
    Title.TextColor3 = Color3.fromRGB(220, 225, 255)
    Title.TextSize = 20
    Title.Font = Enum.Font.GothamBold
    Title.TextXAlignment = Enum.TextXAlignment.Left
    Title.ZIndex = 3
    Title.Parent = TitleBar
    local TitleGrad = Instance.new("UIGradient", Title)
    TitleGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0,   Color3.fromRGB(100,200,255)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(210,210,255)),
        ColorSequenceKeypoint.new(1,   Color3.fromRGB(160,90,255))
    }

    local SubTitle = Instance.new("TextLabel")
    SubTitle.Size = UDim2.new(1, -120, 0, 16)
    SubTitle.Position = UDim2.new(0, 32, 0, 34)
    SubTitle.BackgroundTransparency = 1
    SubTitle.Text = ""
    SubTitle.TextColor3 = Color3.fromRGB(60, 90, 140)
    SubTitle.TextSize = 11
    SubTitle.Font = Enum.Font.Gotham
    SubTitle.TextXAlignment = Enum.TextXAlignment.Left
    SubTitle.ZIndex = 3
    SubTitle.Parent = TitleBar

    -- Window buttons
    local function createTopButton(text, posX, color)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 32, 0, 32)
        btn.Position = UDim2.new(1, posX, 0.5, -16)
        btn.Text = text
        btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 16
        btn.BackgroundColor3 = color
        btn.BorderSizePixel = 0
        btn.ZIndex = 4
        btn.Parent = TitleBar
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)
        local g = Instance.new("UIGradient", btn)
        g.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0, color:Lerp(Color3.new(1,1,1), 0.18)),
            ColorSequenceKeypoint.new(1, color)
        }
        g.Rotation = 90
        local s = Instance.new("UIStroke", btn)
        s.Color = color:Lerp(Color3.new(1,1,1), 0.35)
        s.Transparency = 0.5
        s.Thickness = 1
        btn.MouseEnter:Connect(function()
            TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundColor3 = color:Lerp(Color3.new(1,1,1), 0.28) }):Play()
        end)
        btn.MouseLeave:Connect(function()
            TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundColor3 = color }):Play()
        end)
        return btn
    end

    local CloseButton    = createTopButton("×", -44, Color3.fromRGB(200, 50, 50))
    local MinimizeButton = createTopButton("−", -82, Color3.fromRGB(35, 35, 55))

    -- ===== TAB BAR =====
    local TabBar = Instance.new("Frame")
    TabBar.Size = UDim2.new(1, -24, 0, 36)
    TabBar.Position = UDim2.new(0, 12, 0, 62)
    TabBar.BackgroundColor3 = Color3.fromRGB(10, 10, 18)
    TabBar.BorderSizePixel = 0
    TabBar.ZIndex = 2
    TabBar.Parent = InnerClip
    Instance.new("UICorner", TabBar).CornerRadius = UDim.new(0, 12)
    local TabBarStroke = Instance.new("UIStroke", TabBar)
    TabBarStroke.Color = Color3.fromRGB(30, 35, 65)
    TabBarStroke.Transparency = 0.3
    TabBarStroke.Thickness = 1
    local TabList = Instance.new("UIListLayout", TabBar)
    TabList.FillDirection = Enum.FillDirection.Horizontal
    TabList.HorizontalAlignment = Enum.HorizontalAlignment.Center
    TabList.VerticalAlignment = Enum.VerticalAlignment.Center
    TabList.Padding = UDim.new(0, 4)

    -- ===== PAGE CONTAINER =====
    local PageContainer = Instance.new("Frame")
    PageContainer.Size = UDim2.new(1, 0, 1, -104)
    PageContainer.Position = UDim2.new(0, 0, 0, 104)
    PageContainer.BackgroundTransparency = 1
    PageContainer.ClipsDescendants = true
    PageContainer.ZIndex = 2
    PageContainer.Parent = InnerClip

    local function makePage()
        local p = Instance.new("Frame")
        p.Size = UDim2.new(1, 0, 1, 0)
        p.BackgroundTransparency = 1
        p.Visible = false
        p.ZIndex = 2
        p.Parent = PageContainer
        return p
    end

    local PageMain    = makePage()
    local PageTargets = makePage()
    local PageStatus  = makePage()
    local PageVisuals = makePage()

    -- ===== TAB SYSTEM =====
    local tabs = {}
    local activeTab = nil
    local tabColors = {
        active   = Color3.fromRGB(60, 120, 255),
        inactive = Color3.fromRGB(18, 18, 32)
    }

    local function createTab(labelText, page, icon)
        local tab = Instance.new("TextButton")
        tab.Size = UDim2.new(0, 85, 0, 28)
        tab.BackgroundColor3 = tabColors.inactive
        tab.Text = icon .. "  " .. labelText
        tab.TextColor3 = Color3.fromRGB(90, 100, 140)
        tab.Font = Enum.Font.GothamBold
        tab.TextSize = 12
        tab.BorderSizePixel = 0
        tab.ZIndex = 3
        tab.Parent = TabBar
        Instance.new("UICorner", tab).CornerRadius = UDim.new(0, 9)
        local tabStroke = Instance.new("UIStroke", tab)
        tabStroke.Color = Color3.fromRGB(30, 35, 65)
        tabStroke.Transparency = 0.6
        tabStroke.Thickness = 1
        local tabData = { button = tab, page = page, stroke = tabStroke }
        table.insert(tabs, tabData)
        tab.MouseButton1Click:Connect(function()
            for _, t in pairs(tabs) do
                t.page.Visible = false
                TweenService:Create(t.button, TweenInfo.new(0.2), {
                    BackgroundColor3 = tabColors.inactive,
                    TextColor3 = Color3.fromRGB(80, 90, 130)
                }):Play()
                TweenService:Create(t.stroke, TweenInfo.new(0.2), {
                    Transparency = 0.6,
                    Color = Color3.fromRGB(30, 35, 65)
                }):Play()
            end
            page.Visible = true
            TweenService:Create(tab, TweenInfo.new(0.2), {
                BackgroundColor3 = tabColors.active,
                TextColor3 = Color3.fromRGB(220, 235, 255)
            }):Play()
            TweenService:Create(tabStroke, TweenInfo.new(0.2), {
                Transparency = 0,
                Color = Color3.fromRGB(100, 160, 255)
            }):Play()
            activeTab = tabData
        end)
        return tabData
    end

    local tabMain    = createTab("Main",    PageMain,    "⚡")
    local tabTargets = createTab("Targets", PageTargets, "🎯")
    local tabStatus  = createTab("Status",  PageStatus,  "📊")
    local tabVisuals = createTab("Visuals", PageVisuals, "🎨")

    -- Activate Main tab by default
    PageMain.Visible = true
    tabMain.button.BackgroundColor3 = tabColors.active
    tabMain.button.TextColor3 = Color3.fromRGB(220, 235, 255)
    tabMain.stroke.Transparency = 0
    tabMain.stroke.Color = Color3.fromRGB(100, 160, 255)

    -- ===== DRAGGING =====
    local dragging, dragStart, startPos = false, nil, nil
    TitleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = MainFrame.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    -- ===== MINIMIZE =====
    local minimized = false
    MinimizeButton.MouseButton1Click:Connect(function()
        minimized = not minimized
        TweenService:Create(MainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad), {
            Size = minimized and UDim2.new(0, 420, 0, 58) or UDim2.new(0, 420, 0, 620)
        }):Play()
        PageContainer.Visible = not minimized
        TabBar.Visible = not minimized
        MinimizeButton.Text = minimized and "+" or "−"
    end)

    -- ===== CLOSE =====
    CloseButton.MouseButton1Click:Connect(function()
        TweenService:Create(MainFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Size = UDim2.new(0, 420, 0, 0)
        }):Play()
        task.wait(0.25)
        ScreenGui:Destroy()
    end)

    ScreenGui.Parent = player:WaitForChild("PlayerGui")

    return PageMain, PageTargets, PageStatus, PageVisuals, ScreenGui
end

-- ==================== PROFESSIONAL CONTROL CREATORS ====================

local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

-- BUTTON
local function CreateToggleButton(parent, text, yPosition)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0.9, 0, 0, 52)
    button.Position = UDim2.new(0.05, 0, yPosition, 0)
    button.BackgroundColor3 = Color3.fromRGB(12, 10, 18)
    button.Text = text
    button.TextColor3 = Color3.fromRGB(255, 80, 80)
    button.Font = Enum.Font.GothamBold
    button.TextSize = 22
    button.BorderSizePixel = 0
    button.ZIndex = 3
    button.Parent = parent
    Instance.new("UICorner", button).CornerRadius = UDim.new(0, 16)

    local stroke = Instance.new("UIStroke", button)
    stroke.Color = Color3.fromRGB(220, 60, 60)
    stroke.Thickness = 1.5
    stroke.Transparency = 0.4
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

    button.MouseButton1Down:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.08), { Size = UDim2.new(0.88, 0, 0, 50) }):Play()
    end)
    button.MouseButton1Up:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.08), { Size = UDim2.new(0.9, 0, 0, 52) }):Play()
    end)

    return button
end

-- SLIDER
local function CreateSlider(parent, labelText, minValue, maxValue, defaultValue, yPosition, valueChangedCallback)
    local sliderFrame = Instance.new("Frame")
    sliderFrame.Name = "SliderFrame"
    sliderFrame.Size = UDim2.new(0.9, 0, 0, 68)
    sliderFrame.Position = UDim2.new(0.05, 0, yPosition, 0)
    sliderFrame.BackgroundColor3 = Color3.fromRGB(11, 11, 20)
    sliderFrame.BorderSizePixel = 0
    sliderFrame.ZIndex = 3
    sliderFrame.Parent = parent
    Instance.new("UICorner", sliderFrame).CornerRadius = UDim.new(0, 16)

    local frameBorder = Instance.new("UIStroke", sliderFrame)
    frameBorder.Color = Color3.fromRGB(30, 40, 80)
    frameBorder.Thickness = 1
    frameBorder.Transparency = 0.3

    local frameGrad = Instance.new("UIGradient", sliderFrame)
    frameGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, Color3.fromRGB(18, 18, 32)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(8,   8, 14))
    }
    frameGrad.Rotation = 135

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.72, 0, 0, 22)
    label.Position = UDim2.new(0, 14, 0, 7)
    label.BackgroundTransparency = 1
    label.Text = labelText
    label.TextColor3 = Color3.fromRGB(170, 180, 215)
    label.Font = Enum.Font.Gotham
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 4
    label.Parent = sliderFrame

    local valueLabel = Instance.new("TextLabel")
    valueLabel.Size = UDim2.new(0.25, 0, 0, 22)
    valueLabel.Position = UDim2.new(0.74, 0, 0, 7)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = tostring(defaultValue)
    valueLabel.TextColor3 = Color3.fromRGB(80, 180, 255)
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextSize = 14
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right
    valueLabel.ZIndex = 4
    valueLabel.Parent = sliderFrame

    local sliderContainer = Instance.new("Frame")
    sliderContainer.Size = UDim2.new(1, -28, 0, 30)
    sliderContainer.Position = UDim2.new(0, 14, 0, 32)
    sliderContainer.BackgroundColor3 = Color3.fromRGB(7, 7, 13)
    sliderContainer.BorderSizePixel = 0
    sliderContainer.ZIndex = 4
    sliderContainer.Parent = sliderFrame
    Instance.new("UICorner", sliderContainer).CornerRadius = UDim.new(0, 12)

    local sliderTrack = Instance.new("Frame")
    sliderTrack.Size = UDim2.new(1, -16, 0, 5)
    sliderTrack.Position = UDim2.new(0, 8, 0.5, -2)
    sliderTrack.BackgroundColor3 = Color3.fromRGB(25, 28, 52)
    sliderTrack.BorderSizePixel = 0
    sliderTrack.ZIndex = 5
    sliderTrack.Parent = sliderContainer
    Instance.new("UICorner", sliderTrack).CornerRadius = UDim.new(1, 0)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(0, 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(70, 150, 255)
    fill.BorderSizePixel = 0
    fill.ZIndex = 6
    fill.Parent = sliderTrack
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
    local fillGrad = Instance.new("UIGradient", fill)
    fillGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, Color3.fromRGB(50, 100, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(120, 200, 255))
    }

    local sliderThumb = Instance.new("Frame")
    sliderThumb.Size = UDim2.new(0, 20, 0, 20)
    sliderThumb.Position = UDim2.new(0, -10, 0.5, -10)
    sliderThumb.BackgroundColor3 = Color3.fromRGB(210, 230, 255)
    sliderThumb.BorderSizePixel = 0
    sliderThumb.ZIndex = 7
    sliderThumb.Parent = sliderContainer
    Instance.new("UICorner", sliderThumb).CornerRadius = UDim.new(1, 0)
    local thumbStroke = Instance.new("UIStroke", sliderThumb)
    thumbStroke.Color = Color3.fromRGB(80, 160, 255)
    thumbStroke.Thickness = 1.5
    thumbStroke.Transparency = 0.15

    local currentValue = defaultValue
    local draggingSlider = false

    local function updateSlider(value)
        currentValue = math.clamp(value, minValue, maxValue)
        local percentage = (currentValue - minValue) / (maxValue - minValue)
        TweenService:Create(sliderThumb, TweenInfo.new(0.1, Enum.EasingStyle.Quad), {
            Position = UDim2.new(percentage, -10, 0.5, -10)
        }):Play()
        TweenService:Create(fill, TweenInfo.new(0.1), { Size = UDim2.new(percentage, 0, 1, 0) }):Play()
        valueLabel.Text = tostring(math.floor(currentValue))
        if valueChangedCallback then valueChangedCallback(currentValue) end
    end

    sliderThumb.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = true
            TweenService:Create(sliderThumb, TweenInfo.new(0.1), { Size = UDim2.new(0, 24, 0, 24) }):Play()
        end
    end)

    sliderContainer.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = true
            local clickPos   = input.Position.X - sliderContainer.AbsolutePosition.X
            local trackStart = sliderTrack.AbsolutePosition.X - sliderContainer.AbsolutePosition.X
            local trackWidth = sliderTrack.AbsoluteSize.X
            local percentage = math.clamp((clickPos - trackStart) / trackWidth, 0, 1)
            updateSlider(minValue + percentage * (maxValue - minValue))
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local relX       = input.Position.X - sliderContainer.AbsolutePosition.X
            local trackStart = sliderTrack.AbsolutePosition.X - sliderContainer.AbsolutePosition.X
            local trackWidth = sliderTrack.AbsoluteSize.X
            local percentage = math.clamp((relX - trackStart) / trackWidth, 0, 1)
            updateSlider(minValue + percentage * (maxValue - minValue))
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = false
            TweenService:Create(sliderThumb, TweenInfo.new(0.1), { Size = UDim2.new(0, 20, 0, 20) }):Play()
        end
    end)

    updateSlider(defaultValue)
    return sliderFrame
end

-- TEXTBOX
local function CreateTextBox(parent, placeholderText, yPosition, textChangedCallback)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(0.9, 0, 0, 44)
    container.Position = UDim2.new(0.05, 0, yPosition, 0)
    container.BackgroundColor3 = Color3.fromRGB(10, 10, 18)
    container.BorderSizePixel = 0
    container.ZIndex = 3
    container.Parent = parent
    Instance.new("UICorner", container).CornerRadius = UDim.new(0, 14)

    local containerGrad = Instance.new("UIGradient", container)
    containerGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, Color3.fromRGB(16, 16, 30)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(8,   8, 14))
    }
    containerGrad.Rotation = 90

    local stroke = Instance.new("UIStroke", container)
    stroke.Color = Color3.fromRGB(32, 38, 72)
    stroke.Thickness = 1
    stroke.Transparency = 0.25

    local accent = Instance.new("Frame")
    accent.Size = UDim2.new(0, 3, 0.5, 0)
    accent.Position = UDim2.new(0, 1, 0.25, 0)
    accent.BackgroundColor3 = Color3.fromRGB(50, 110, 220)
    accent.BorderSizePixel = 0
    accent.ZIndex = 4
    accent.Parent = container
    Instance.new("UICorner", accent).CornerRadius = UDim.new(1, 0)

    local textBox = Instance.new("TextBox")
    textBox.Size = UDim2.new(1, -22, 1, -10)
    textBox.Position = UDim2.new(0, 16, 0, 5)
    textBox.BackgroundTransparency = 1
    textBox.Text = ""
    textBox.PlaceholderText = placeholderText
    textBox.PlaceholderColor3 = Color3.fromRGB(60, 72, 115)
    textBox.TextColor3 = Color3.fromRGB(200, 210, 240)
    textBox.Font = Enum.Font.Gotham
    textBox.TextSize = 13
    textBox.TextXAlignment = Enum.TextXAlignment.Left
    textBox.ClearTextOnFocus = false
    textBox.ZIndex = 4
    textBox.Parent = container

    textBox.Focused:Connect(function()
        TweenService:Create(stroke, TweenInfo.new(0.2), { Color = Color3.fromRGB(70,150,255), Transparency = 0 }):Play()
        TweenService:Create(accent, TweenInfo.new(0.2), { BackgroundColor3 = Color3.fromRGB(80,160,255) }):Play()
    end)
    textBox.FocusLost:Connect(function()
        TweenService:Create(stroke, TweenInfo.new(0.2), { Color = Color3.fromRGB(32,38,72), Transparency = 0.25 }):Play()
        TweenService:Create(accent, TweenInfo.new(0.2), { BackgroundColor3 = Color3.fromRGB(50,110,220) }):Play()
        if textChangedCallback then textChangedCallback(textBox.Text) end
    end)

    return textBox
end

-- ==================== MAIN GUI SETUP ====================

local pageMain, pageTargets, pageStatus, pageVisuals, screenGui = CreateProfessionalGUI()

-- ========== PAGE 1: MAIN ==========
local toggleButton = CreateToggleButton(pageMain, "OFF", 0.04)
toggleButton.TextColor3 = Color3.fromRGB(255, 60, 60)
toggleButton.BackgroundColor3 = Color3.fromRGB(40, 8, 8)

local divLabel1 = Instance.new("TextLabel")
divLabel1.Size = UDim2.new(0.9, 0, 0, 18)
divLabel1.Position = UDim2.new(0.05, 0, 0.22, 0)
divLabel1.BackgroundTransparency = 1
divLabel1.Text = "AURAKILL"
divLabel1.TextColor3 = Color3.fromRGB(50, 80, 140)
divLabel1.Font = Enum.Font.GothamBold
divLabel1.TextSize = 11
divLabel1.TextXAlignment = Enum.TextXAlignment.Left
divLabel1.ZIndex = 3
divLabel1.Parent = pageMain

local apsSlider = CreateSlider(pageMain, "Attacks Per Second:", 1, 100000, 10000, 0.28, function(value)
    attacksPerSecond = value
    attackCooldown = 0
end)

local rangeSlider = CreateSlider(pageMain, "Attack Range (studs):", 1, 250, 50, 0.53, function(value)
    attackRange = value
end)

-- ========== PAGE 2: TARGETS ==========
local divLabel2 = Instance.new("TextLabel")
divLabel2.Size = UDim2.new(0.9, 0, 0, 18)
divLabel2.Position = UDim2.new(0.05, 0, 0.03, 0)
divLabel2.BackgroundTransparency = 1
divLabel2.Text = "PLAYER LISTS"
divLabel2.TextColor3 = Color3.fromRGB(50, 80, 140)
divLabel2.Font = Enum.Font.GothamBold
divLabel2.TextSize = 11
divLabel2.TextXAlignment = Enum.TextXAlignment.Left
divLabel2.ZIndex = 3
divLabel2.Parent = pageTargets

local targetTextBox = CreateTextBox(pageTargets, "Target players (comma separated)", 0.10, function(text)
    targetPlayerNames = text
    updateTargetList()
end)

local headSitTextBox = CreateTextBox(pageTargets, "Head sit targets (comma separated)", 0.26, function(text)
    headSitPlayerNames = text
    updateHeadSitTargetList()
end)

local avoidTextBox = CreateTextBox(pageTargets, "Avoid players (comma separated)", 0.42, function(text)
    avoidPlayerNames = text
    updateAvoidList()
end)

-- Info tip card
local infoTip = Instance.new("Frame")
infoTip.Size = UDim2.new(0.9, 0, 0, 42)
infoTip.Position = UDim2.new(0.05, 0, 0.62, 0)
infoTip.BackgroundColor3 = Color3.fromRGB(12, 18, 35)
infoTip.BorderSizePixel = 0
infoTip.ZIndex = 3
infoTip.Parent = pageTargets
Instance.new("UICorner", infoTip).CornerRadius = UDim.new(0, 12)
local infoStroke = Instance.new("UIStroke", infoTip)
infoStroke.Color = Color3.fromRGB(30, 50, 100)
infoStroke.Transparency = 0.4
infoStroke.Thickness = 1
local infoLabel = Instance.new("TextLabel")
infoLabel.Size = UDim2.new(1, -16, 1, 0)
infoLabel.Position = UDim2.new(0, 10, 0, 0)
infoLabel.BackgroundTransparency = 1
infoLabel.Text = "💡  Separate names with commas.\nLeave blank to target all players."
infoLabel.TextColor3 = Color3.fromRGB(70, 100, 160)
infoLabel.Font = Enum.Font.Gotham
infoLabel.TextSize = 11
infoLabel.TextWrapped = true
infoLabel.TextXAlignment = Enum.TextXAlignment.Left
infoLabel.ZIndex = 4
infoLabel.Parent = infoTip

-- ========== PAGE 3: STATUS ==========

-- Status card
local statusContainer = Instance.new("Frame")
statusContainer.Size = UDim2.new(0.9, 0, 0, 48)
statusContainer.Position = UDim2.new(0.05, 0, 0.04, 0)
statusContainer.BackgroundColor3 = Color3.fromRGB(10, 12, 22)
statusContainer.BorderSizePixel = 0
statusContainer.ZIndex = 3
statusContainer.Parent = pageStatus
Instance.new("UICorner", statusContainer).CornerRadius = UDim.new(0, 14)
local statusGrad = Instance.new("UIGradient", statusContainer)
statusGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(16, 22, 42)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(7,   8, 14))
}
statusGrad.Rotation = 90
local statusStroke = Instance.new("UIStroke", statusContainer)
statusStroke.Color = Color3.fromRGB(35, 100, 200)
statusStroke.Transparency = 0.35
statusStroke.Thickness = 1

local statusDot = Instance.new("Frame")
statusDot.Size = UDim2.new(0, 9, 0, 9)
statusDot.Position = UDim2.new(0, 14, 0.5, -4)
statusDot.BackgroundColor3 = Color3.fromRGB(60, 210, 120)
statusDot.BorderSizePixel = 0
statusDot.ZIndex = 4
statusDot.Parent = statusContainer
Instance.new("UICorner", statusDot).CornerRadius = UDim.new(1, 0)

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, -38, 1, 0)
statusLabel.Position = UDim2.new(0, 32, 0, 0)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = "Status: Ready"
statusLabel.TextColor3 = Color3.fromRGB(140, 200, 255)
statusLabel.Font = Enum.Font.GothamBold
statusLabel.TextSize = 14
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.ZIndex = 4
statusLabel.Parent = statusContainer

-- Stats card
local statsContainer = Instance.new("Frame")
statsContainer.Size = UDim2.new(0.9, 0, 0, 200)
statsContainer.Position = UDim2.new(0.05, 0, 0.22, 0)
statsContainer.BackgroundColor3 = Color3.fromRGB(10, 10, 18)
statsContainer.BorderSizePixel = 0
statsContainer.ZIndex = 3
statsContainer.Parent = pageStatus
Instance.new("UICorner", statsContainer).CornerRadius = UDim.new(0, 16)
local statsGrad = Instance.new("UIGradient", statsContainer)
statsGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(14, 16, 30)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(7,   8, 12))
}
statsGrad.Rotation = 135
local statsStroke = Instance.new("UIStroke", statsContainer)
statsStroke.Color = Color3.fromRGB(30, 45, 90)
statsStroke.Transparency = 0.25
statsStroke.Thickness = 1

local statsHeaderBar = Instance.new("Frame")
statsHeaderBar.Size = UDim2.new(1, 0, 0, 32)
statsHeaderBar.BackgroundColor3 = Color3.fromRGB(12, 14, 26)
statsHeaderBar.BorderSizePixel = 0
statsHeaderBar.ZIndex = 4
statsHeaderBar.Parent = statsContainer
Instance.new("UICorner", statsHeaderBar).CornerRadius = UDim.new(0, 16)
-- Fix bottom corners
local headerFix = Instance.new("Frame")
headerFix.Size = UDim2.new(1, 0, 0, 16)
headerFix.Position = UDim2.new(0, 0, 1, -16)
headerFix.BackgroundColor3 = Color3.fromRGB(12, 14, 26)
headerFix.BorderSizePixel = 0
headerFix.ZIndex = 4
headerFix.Parent = statsHeaderBar

local statsHeaderLabel = Instance.new("TextLabel")
statsHeaderLabel.Size = UDim2.new(1, -20, 1, 0)
statsHeaderLabel.Position = UDim2.new(0, 14, 0, 0)
statsHeaderLabel.BackgroundTransparency = 1
statsHeaderLabel.Text = "LIVE STATS"
statsHeaderLabel.TextColor3 = Color3.fromRGB(80, 130, 220)
statsHeaderLabel.Font = Enum.Font.GothamBold
statsHeaderLabel.TextSize = 11
statsHeaderLabel.TextXAlignment = Enum.TextXAlignment.Left
statsHeaderLabel.ZIndex = 5
statsHeaderLabel.Parent = statsHeaderBar

local headerLine = Instance.new("Frame")
headerLine.Size = UDim2.new(0.35, 0, 0, 1)
headerLine.Position = UDim2.new(0, 14, 1, -1)
headerLine.BackgroundColor3 = Color3.fromRGB(60, 120, 255)
headerLine.BorderSizePixel = 0
headerLine.ZIndex = 5
headerLine.Parent = statsHeaderBar
local hlGrad = Instance.new("UIGradient", headerLine)
hlGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(80,160,255)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0,0,0))
}

local statsLabel = Instance.new("TextLabel")
statsLabel.Size = UDim2.new(1, -24, 1, -40)
statsLabel.Position = UDim2.new(0, 14, 0, 36)
statsLabel.BackgroundTransparency = 1
statsLabel.Text = "APS: 400\nRange: 50\nAnti-Ragdoll: OFF\nAnti-Fling: OFF\nTarget: All\nHead: None\nAvoid: None"
statsLabel.TextColor3 = Color3.fromRGB(165, 175, 210)
statsLabel.Font = Enum.Font.Gotham
statsLabel.TextSize = 13
statsLabel.TextXAlignment = Enum.TextXAlignment.Left
statsLabel.TextYAlignment = Enum.TextYAlignment.Top
statsLabel.TextWrapped = true
statsLabel.LineHeight = 1.5
statsLabel.ZIndex = 4
statsLabel.Parent = statsContainer

-- Keybind card
local keybindContainer = Instance.new("Frame")
keybindContainer.Size = UDim2.new(0.9, 0, 0, 36)
keybindContainer.Position = UDim2.new(0.05, 0, 0.82, 0)
keybindContainer.BackgroundColor3 = Color3.fromRGB(10, 10, 18)
keybindContainer.BorderSizePixel = 0
keybindContainer.ZIndex = 3
keybindContainer.Parent = pageStatus
Instance.new("UICorner", keybindContainer).CornerRadius = UDim.new(0, 12)
local keybindGrad = Instance.new("UIGradient", keybindContainer)
keybindGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(14, 16, 30)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(7,   8, 12))
}
keybindGrad.Rotation = 90
local keybindStroke = Instance.new("UIStroke", keybindContainer)
keybindStroke.Color = Color3.fromRGB(35, 55, 120)
keybindStroke.Transparency = 0.45
keybindStroke.Thickness = 1

local keyBadge = Instance.new("Frame")
keyBadge.Size = UDim2.new(0, 22, 0, 22)
keyBadge.Position = UDim2.new(1, -30, 0.5, -11)
keyBadge.BackgroundColor3 = Color3.fromRGB(22, 40, 90)
keyBadge.BorderSizePixel = 0
keyBadge.ZIndex = 4
keyBadge.Parent = keybindContainer
Instance.new("UICorner", keyBadge).CornerRadius = UDim.new(0, 6)
local keyBadgeStroke = Instance.new("UIStroke", keyBadge)
keyBadgeStroke.Color = Color3.fromRGB(60, 120, 220)
keyBadgeStroke.Transparency = 0.3
keyBadgeStroke.Thickness = 1
local keyBadgeLabel = Instance.new("TextLabel")
keyBadgeLabel.Size = UDim2.new(1, 0, 1, 0)
keyBadgeLabel.BackgroundTransparency = 1
keyBadgeLabel.Text = "E"
keyBadgeLabel.TextColor3 = Color3.fromRGB(100, 175, 255)
keyBadgeLabel.Font = Enum.Font.GothamBold
keyBadgeLabel.TextSize = 12
keyBadgeLabel.ZIndex = 5
keyBadgeLabel.Parent = keyBadge

local keybindLabel = Instance.new("TextLabel")
keybindLabel.Size = UDim2.new(1, -46, 1, 0)
keybindLabel.Position = UDim2.new(0, 14, 0, 0)
keybindLabel.BackgroundTransparency = 1
keybindLabel.Text = "Toggle Key:"
keybindLabel.TextColor3 = Color3.fromRGB(90, 120, 175)
keybindLabel.Font = Enum.Font.GothamBold
keybindLabel.TextSize = 12
keybindLabel.TextXAlignment = Enum.TextXAlignment.Left
keybindLabel.ZIndex = 4
keybindLabel.Parent = keybindContainer

-- ========== PAGE 4: VISUALS ==========

-- Color blink services (integrated)
local _ReplicatedStorage = game:GetService("ReplicatedStorage")
local _Remotes      = _ReplicatedStorage:WaitForChild("Remotes")
local _ColorRemote  = _Remotes:WaitForChild("UpdateRPColor")
local _BioColorRemote = _Remotes:WaitForChild("UpdateBioColor")

local blinkSpeed = 0.003
local blinkEnabled = false
local blinkConnection = nil
local blinkColorIndex = 1
local blinkTimer = 0
local isBioFunc = _BioColorRemote:IsA("RemoteFunction")

local blinkColors = {
    Color3.fromRGB(255, 255, 255),
    Color3.fromRGB(0,   0,   0),
}

local function getNextBlinkColor()
    local c = blinkColors[blinkColorIndex]
    blinkColorIndex = blinkColorIndex % #blinkColors + 1
    return c
end

-- Section label
local visLabel = Instance.new("TextLabel")
visLabel.Size = UDim2.new(0.9, 0, 0, 18)
visLabel.Position = UDim2.new(0.05, 0, 0.03, 0)
visLabel.BackgroundTransparency = 1
visLabel.Text = "NAME COLOR"
visLabel.TextColor3 = Color3.fromRGB(50, 80, 140)
visLabel.Font = Enum.Font.GothamBold
visLabel.TextSize = 11
visLabel.TextXAlignment = Enum.TextXAlignment.Left
visLabel.ZIndex = 3
visLabel.Parent = pageVisuals

-- Color preview card
local colorPreviewCard = Instance.new("Frame")
colorPreviewCard.Size = UDim2.new(0.9, 0, 0, 54)
colorPreviewCard.Position = UDim2.new(0.05, 0, 0.10, 0)
colorPreviewCard.BackgroundColor3 = Color3.fromRGB(10, 10, 18)
colorPreviewCard.BorderSizePixel = 0
colorPreviewCard.ZIndex = 3
colorPreviewCard.Parent = pageVisuals
Instance.new("UICorner", colorPreviewCard).CornerRadius = UDim.new(0, 14)
local cpStroke = Instance.new("UIStroke", colorPreviewCard)
cpStroke.Color = Color3.fromRGB(35, 40, 80)
cpStroke.Transparency = 0.3
cpStroke.Thickness = 1
local cpGrad = Instance.new("UIGradient", colorPreviewCard)
cpGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(16, 16, 30)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(8,   8, 14))
}
cpGrad.Rotation = 90

local cpText = Instance.new("TextLabel")
cpText.Size = UDim2.new(0.5, 0, 1, 0)
cpText.Position = UDim2.new(0, 14, 0, 0)
cpText.BackgroundTransparency = 1
cpText.Text = "Preview"
cpText.TextColor3 = Color3.fromRGB(150, 160, 200)
cpText.Font = Enum.Font.Gotham
cpText.TextSize = 13
cpText.TextXAlignment = Enum.TextXAlignment.Left
cpText.ZIndex = 4
cpText.Parent = colorPreviewCard

local colorBox = Instance.new("Frame")
colorBox.Size = UDim2.new(0, 38, 0, 30)
colorBox.Position = UDim2.new(1, -52, 0.5, -15)
colorBox.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
colorBox.BorderSizePixel = 0
colorBox.ZIndex = 4
colorBox.Parent = colorPreviewCard
Instance.new("UICorner", colorBox).CornerRadius = UDim.new(0, 8)
local cbStroke = Instance.new("UIStroke", colorBox)
cbStroke.Color = Color3.fromRGB(80, 80, 140)
cbStroke.Transparency = 0.4
cbStroke.Thickness = 1

-- Toggle blink button
local blinkToggleBtn = Instance.new("TextButton")
blinkToggleBtn.Size = UDim2.new(0.9, 0, 0, 52)
blinkToggleBtn.Position = UDim2.new(0.05, 0, 0.27, 0)
blinkToggleBtn.BackgroundColor3 = Color3.fromRGB(40, 8, 8)
blinkToggleBtn.Text = "OFF"
blinkToggleBtn.TextColor3 = Color3.fromRGB(255, 60, 60)
blinkToggleBtn.Font = Enum.Font.GothamBold
blinkToggleBtn.TextSize = 15
blinkToggleBtn.BorderSizePixel = 0
blinkToggleBtn.ZIndex = 3
blinkToggleBtn.Parent = pageVisuals
Instance.new("UICorner", blinkToggleBtn).CornerRadius = UDim.new(0, 16)

local blinkBtnStroke = Instance.new("UIStroke", blinkToggleBtn)
blinkBtnStroke.Color = Color3.fromRGB(200, 50, 50)
blinkBtnStroke.Thickness = 1.5
blinkBtnStroke.Transparency = 0.4
blinkBtnStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

local blinkBtnGrad = Instance.new("UIGradient", blinkToggleBtn)
blinkBtnGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(35, 10, 10)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(18,  5,  5))
}
blinkBtnGrad.Rotation = 90

local blinkAccent = Instance.new("Frame") -- kept as dummy to avoid nil errors in setBlinkState
blinkAccent.Size = UDim2.new(0, 0, 0, 0)
blinkAccent.BackgroundTransparency = 1
blinkAccent.Parent = blinkToggleBtn

-- Info card
local blinkInfoCard = Instance.new("Frame")
blinkInfoCard.Size = UDim2.new(0.9, 0, 0, 50)
blinkInfoCard.Position = UDim2.new(0.05, 0, 0.46, 0)
blinkInfoCard.BackgroundColor3 = Color3.fromRGB(12, 18, 35)
blinkInfoCard.BorderSizePixel = 0
blinkInfoCard.ZIndex = 3
blinkInfoCard.Parent = pageVisuals
Instance.new("UICorner", blinkInfoCard).CornerRadius = UDim.new(0, 12)
local biStroke = Instance.new("UIStroke", blinkInfoCard)
biStroke.Color = Color3.fromRGB(30, 50, 100)
biStroke.Transparency = 0.4
biStroke.Thickness = 1
local blinkInfoLabel = Instance.new("TextLabel")
blinkInfoLabel.Size = UDim2.new(1, -16, 1, 0)
blinkInfoLabel.Position = UDim2.new(0, 10, 0, 0)
blinkInfoLabel.BackgroundTransparency = 1
blinkInfoLabel.Text = "Flashes name between white and black."
blinkInfoLabel.TextColor3 = Color3.fromRGB(70, 100, 160)
blinkInfoLabel.Font = Enum.Font.Gotham
blinkInfoLabel.TextSize = 11
blinkInfoLabel.TextWrapped = true
blinkInfoLabel.TextXAlignment = Enum.TextXAlignment.Left
blinkInfoLabel.ZIndex = 4
blinkInfoLabel.Parent = blinkInfoCard

-- Blink toggle logic
local function setBlinkState(on)
    blinkEnabled = on
    if on then
        blinkToggleBtn.Text = "ON"
        blinkToggleBtn.TextColor3 = Color3.fromRGB(60, 255, 100)
        blinkToggleBtn.BackgroundColor3 = Color3.fromRGB(8, 40, 12)
        blinkBtnStroke.Color = Color3.fromRGB(50, 200, 80)
        blinkAccent.BackgroundColor3 = Color3.fromRGB(60, 255, 100)
        blinkBtnGrad.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0, Color3.fromRGB(10, 35, 12)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(5,  18,  6))
        }
        if blinkConnection then blinkConnection:Disconnect() end
        blinkTimer = 0
        blinkColorIndex = 1
        blinkConnection = RunService.Heartbeat:Connect(function(dt)
            if not blinkEnabled then return end
            blinkTimer += dt
            if blinkTimer >= blinkSpeed then
                blinkTimer = 0
                local newColor = getNextBlinkColor()
                if colorBox and colorBox.Parent then
                    colorBox.BackgroundColor3 = newColor
                end
                pcall(function() _ColorRemote:FireServer(newColor) end)
                pcall(function()
                    if isBioFunc then
                        _BioColorRemote:InvokeServer(newColor)
                    else
                        _BioColorRemote:FireServer(newColor)
                    end
                end)
            end
        end)
    else
        blinkToggleBtn.Text = "OFF"
        blinkToggleBtn.TextColor3 = Color3.fromRGB(255, 60, 60)
        blinkToggleBtn.BackgroundColor3 = Color3.fromRGB(40, 8, 8)
        blinkBtnStroke.Color = Color3.fromRGB(200, 50, 50)
        blinkAccent.BackgroundColor3 = Color3.fromRGB(255, 60, 60)
        blinkBtnGrad.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0, Color3.fromRGB(35, 10, 10)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(18,  5,  5))
        }
        if blinkConnection then blinkConnection:Disconnect() blinkConnection = nil end
    end
end

blinkToggleBtn.MouseButton1Click:Connect(function()
    setBlinkState(not blinkEnabled)
end)

blinkToggleBtn.MouseEnter:Connect(function()
    TweenService:Create(blinkToggleBtn, TweenInfo.new(0.15), {
        BackgroundColor3 = blinkEnabled and Color3.fromRGB(10, 50, 16) or Color3.fromRGB(55, 12, 12)
    }):Play()
end)
blinkToggleBtn.MouseLeave:Connect(function()
    TweenService:Create(blinkToggleBtn, TweenInfo.new(0.15), {
        BackgroundColor3 = blinkEnabled and Color3.fromRGB(8, 40, 12) or Color3.fromRGB(40, 8, 8)
    }):Play()
end)
blinkToggleBtn.MouseButton1Down:Connect(function()
    TweenService:Create(blinkToggleBtn, TweenInfo.new(0.08), { Size = UDim2.new(0.88, 0, 0, 50) }):Play()
end)
blinkToggleBtn.MouseButton1Up:Connect(function()
    TweenService:Create(blinkToggleBtn, TweenInfo.new(0.08), { Size = UDim2.new(0.9, 0, 0, 52) }):Play()
end)

-- ==================== TARGETING LOGIC ====================

-- ==================== LOCKED SAFE LIST ====================
-- These players are ALWAYS protected and cannot be removed
local LOCKED_SAFE_LIST = {
    ["replix_rip"] = true,
}

local function isAvoided(playerObj)
    local playerNameLower = playerObj.Name:lower()
    local displayNameLower = playerObj.DisplayName:lower()
    -- Always check locked safe list first (cannot be bypassed)
    for lockedName in pairs(LOCKED_SAFE_LIST) do
        if playerNameLower:find(lockedName, 1, true) or displayNameLower:find(lockedName, 1, true) then
            return true
        end
    end
    -- Exact match from user avoid list
    if avoidList[playerNameLower] or avoidList[displayNameLower] then return true end
    -- Partial match from user avoid list
    for avoidName in pairs(avoidList) do
        if playerNameLower:find(avoidName, 1, true) or displayNameLower:find(avoidName, 1, true) then
            return true
        end
    end
    return false
end

local function isPlayerTarget(playerObj)  
    if playerObj == player then return false end  
    if isAvoided(playerObj) then return false end
    if targetPlayerNames == "" then return true end  
    local playerNameLower = playerObj.Name:lower()  
    local displayNameLower = playerObj.DisplayName:lower()  
    if targetList[playerNameLower] or targetList[displayNameLower] then return true end  
    for targetName in pairs(targetList) do  
        if playerNameLower:find(targetName, 1, true) or displayNameLower:find(targetName, 1, true) then  
            return true  
        end  
    end  
    return false  
end  
  
local function getHeadSitTarget()  
    if headSitPlayerNames == "" then return nil end  
    for _, playerObj in pairs(Players:GetPlayers()) do  
        if playerObj == player then continue end  
        if isAvoided(playerObj) then continue end
        local playerNameLower = playerObj.Name:lower()  
        local displayNameLower = playerObj.DisplayName:lower()  
        if headSitTargetList[playerNameLower] or headSitTargetList[displayNameLower] then return playerObj end  
        for targetName in pairs(headSitTargetList) do  
            if playerNameLower:find(targetName, 1, true) or displayNameLower:find(targetName, 1, true) then  
                return playerObj  
            end  
        end  
    end  
    return nil  
end  
  
-- ==================== UPDATE STATS ====================  
local function updateStats()  
    local targetText = targetPlayerNames == "" and "All players" or targetPlayerNames  
    local headText   = headSitPlayerNames == "" and "None" or headSitPlayerNames  
    local avoidText  = avoidPlayerNames == "" and "None" or avoidPlayerNames  
    statsLabel.Text = string.format(
        "APS: %d\nRange: %d\nAnti-Ragdoll: %s\nAnti-Fling: %s\nTarget: %s\nHead: %s\nAvoid: %s",   
        math.floor(attacksPerSecond),   
        math.floor(attackRange),   
        ScriptEnabled and "ON" or "OFF",  
        ScriptEnabled and "ON" or "OFF",  
        targetText, headText, avoidText
    )
end  
  
RunService.Heartbeat:Connect(updateStats)  


-- ==================== TOGGLE SCRIPT ====================
local function toggleScript()
    ScriptEnabled = not ScriptEnabled

    if ScriptEnabled then
        toggleButton.Text = "ON"
        toggleButton.TextColor3 = Color3.fromRGB(60, 255, 100)
        toggleButton.BackgroundColor3 = Color3.fromRGB(8, 40, 12)

        statusLabel.Text = "Status: Instant Kill"
        statusLabel.TextColor3 = Color3.fromRGB(80, 255, 80)
        statusDot.BackgroundColor3 = Color3.fromRGB(80, 255, 80)

        setupAntiRagdoll()
        setupAntiFling()
        setupGrabFallFixes()
        ensureCharacterSafety()

        for _, p in pairs(Players:GetPlayers()) do
            if p ~= player and p.Character and isPlayerTarget(p) then
                local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                if hrp then
                    originalSizes[hrp] = hrp.Size
                    hrp.Size = Vector3.new(40, 40, 40)
                end
            end
        end

        auraConnection = RunService.Heartbeat:Connect(function()
            if not ScriptEnabled then return end
            if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end

            local myHRP = player.Character.HumanoidRootPart

            -- HEAD SIT
            local headTarget = getHeadSitTarget()
            if headTarget and headTarget.Character then
                local head = headTarget.Character:FindFirstChild("Head")
                if head and (head.Position - myHRP.Position).Magnitude <= attackRange then
                    myHRP.CFrame = CFrame.new(head.Position + Vector3.new(0, 3, 0))
                end
            end

            -- KILL AURA - fire every single heartbeat, no cooldown
            if targetPlayerNames ~= "" then
                for _, p in pairs(Players:GetPlayers()) do
                    if not isPlayerTarget(p) then continue end
                    if not p.Character then continue end
                    local hum = p.Character:FindFirstChild("Humanoid")
                    local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                    if hum and hrp and hum.Health > 0 then
                        local dist = (hrp.Position - myHRP.Position).Magnitude
                        if dist <= attackRange then
                            HitRemote:InvokeServer(hum, vector.create(myHRP.Position.X, myHRP.Position.Y, myHRP.Position.Z))
                        end
                    end
                end
            else
                local closest, closestDist = nil, attackRange
                for _, p in pairs(Players:GetPlayers()) do
                    if not isPlayerTarget(p) then continue end
                    if not p.Character then continue end
                    local hum = p.Character:FindFirstChild("Humanoid")
                    local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                    if hum and hrp and hum.Health > 0 then
                        local dist = (hrp.Position - myHRP.Position).Magnitude
                        if dist <= attackRange and dist < closestDist then
                            closestDist = dist
                            closest = p
                        end
                    end
                end
                if closest and closest.Character and closest.Character:FindFirstChild("Humanoid") then
                    HitRemote:InvokeServer(closest.Character.Humanoid, vector.create(myHRP.Position.X, myHRP.Position.Y, myHRP.Position.Z))
                end
            end

            ensureCharacterSafety()
        end)
    else
        toggleButton.Text = "OFF"
        toggleButton.TextColor3 = Color3.fromRGB(255, 60, 60)
        toggleButton.BackgroundColor3 = Color3.fromRGB(40, 8, 8)

        statusLabel.Text = "Status: Disabled"
        statusLabel.TextColor3 = Color3.fromRGB(255, 150, 150)
        statusDot.BackgroundColor3 = Color3.fromRGB(200, 60, 60)

        if auraConnection then auraConnection:Disconnect() end
        if antiRagdollConnection then antiRagdollConnection:Disconnect() end
        if antiFlingConnection then antiFlingConnection:Disconnect() end

        for hrp, oldSize in pairs(originalSizes) do
            if hrp and hrp.Parent then hrp.Size = oldSize end
        end
        originalSizes = {}
        lastAttackTime = 0
    end
end

toggleButton.MouseButton1Click:Connect(toggleScript)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.E then
        toggleScript()
    end
end)

screenGui.Destroying:Connect(function()
    if auraConnection then auraConnection:Disconnect() end
    if antiRagdollConnection then antiRagdollConnection:Disconnect() end
    if antiFlingConnection then antiFlingConnection:Disconnect() end
    if blinkConnection then blinkConnection:Disconnect() end
end)

updateTargetList()
updateHeadSitTargetList()
updateAvoidList()
updateStats()

print("==============================================")
print(FROSTY 10/10 MAXIMUM SPEED Kill Aura")
print("Loaded Successfully!")
print("⚡ FIXED TARGET LOOP - No Slowdown")
print("⚡ INSTANT KILL - Guaranteed Hits")
print("==============================================")
