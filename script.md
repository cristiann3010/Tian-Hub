-- OwnerToolsRayfield.lua
-- LocalScript: place in StarterPlayer > StarterPlayerScripts
-- Requires Rayfield UI library

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer

------------------------------------------------------
-- OWNER CHECK - REMOVE THIS SECTION COMPLETELY
-- Script will work for everyone
------------------------------------------------------

------------------------------------------------------
-- LOAD RAYFIELD
------------------------------------------------------
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "Owner Tools",
    LoadingTitle = "Owner Panel",
    LoadingSubtitle = "Loaded",
    ConfigurationSaving = {
        Enabled = false,
    },
    Discord = {
        Enabled = false,
    },
    KeySystem = false,
})

local MainTab = Window:CreateTab("Tools", 4483362458)

------------------------------------------------------
-- FLY FEATURE
------------------------------------------------------
local flying = false
local flySpeed = 50
local bodyVelocity, bodyGyro
local flyConnection
local flyLoop

local function startFly()
    local char = LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    -- Disable gravity on humanoid
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid.PlatformStand = true
    end

    bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.MaxForce = Vector3.new(1e5, 1e5, 1e5)
    bodyVelocity.Velocity = Vector3.new(0,0,0)
    bodyVelocity.Parent = hrp

    bodyGyro = Instance.new("BodyGyro")
    bodyGyro.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
    bodyGyro.Parent = hrp

    flyLoop = RunService.RenderStepped:Connect(function()
        if not flying then return end
        if not hrp or not hrp.Parent then return end
        
        local camera = workspace.CurrentCamera
        local moveDir = Vector3.new()
        
        -- WASD movement
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then
            moveDir = moveDir + (camera.CFrame.LookVector * Vector3.new(1, 0, 1))
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then
            moveDir = moveDir - (camera.CFrame.LookVector * Vector3.new(1, 0, 1))
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then
            moveDir = moveDir - (camera.CFrame.RightVector * Vector3.new(1, 0, 1))
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then
            moveDir = moveDir + (camera.CFrame.RightVector * Vector3.new(1, 0, 1))
        end
        
        -- Space and Shift for up/down
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
            moveDir = moveDir + Vector3.new(0, 1, 0)
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
            moveDir = moveDir - Vector3.new(0, 1, 0)
        end
        
        if moveDir.Magnitude > 0 then
            moveDir = moveDir.Unit * flySpeed
        end
        
        bodyVelocity.Velocity = moveDir
        bodyGyro.CFrame = camera.CFrame
    end)
end

local function stopFly()
    if flyLoop then
        flyLoop:Disconnect()
        flyLoop = nil
    end
    if bodyVelocity then 
        bodyVelocity:Destroy() 
        bodyVelocity = nil
    end
    if bodyGyro then 
        bodyGyro:Destroy() 
        bodyGyro = nil
    end
    
    local char = LocalPlayer.Character
    if char then
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.PlatformStand = false
        end
    end
end

MainTab:CreateToggle({
    Name = "Fly",
    CurrentValue = false,
    Flag = "FlyToggle",
    Callback = function(value)
        flying = value
        if flying then
            startFly()
        else
            stopFly()
        end
    end,
})

MainTab:CreateSlider({
    Name = "Fly Speed",
    Range = {10, 200},
    Increment = 5,
    CurrentValue = 50,
    Flag = "FlySpeedSlider",
    Callback = function(value)
        flySpeed = value
    end,
})

------------------------------------------------------
-- PLAYER HIGHLIGHT (ESP) - AUTO UPDATING
------------------------------------------------------
local highlights = {}
local highlightEnabled = false
local highlightColor = Color3.fromRGB(255, 0, 0)
local highlightTransparency = 0.5
local highlightUpdateConnection = nil

local function addHighlight(player)
    if player == LocalPlayer then return end
    if highlights[player] then return end
    if not player.Character then return end
    
    local h = Instance.new("Highlight")
    h.Name = "OwnerHighlight"
    h.FillColor = highlightColor
    h.OutlineColor = Color3.fromRGB(255, 255, 255)
    h.FillTransparency = highlightTransparency
    h.OutlineTransparency = 0
    h.Parent = player.Character
    highlights[player] = h
end

local function removeHighlight(player)
    if highlights[player] then
        highlights[player]:Destroy()
        highlights[player] = nil
    end
end

local function refreshHighlights()
    -- Remove highlights for players that no longer exist or don't have character
    for player, highlight in pairs(highlights) do
        if not player or not player.Parent or not player.Character then
            removeHighlight(player)
        end
    end
    
    -- Add highlights to all valid players
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and not highlights[player] then
            addHighlight(player)
        end
    end
end

local function toggleHighlight(value)
    highlightEnabled = value
    
    if highlightEnabled then
        -- Initial refresh
        refreshHighlights()
        
        -- Auto-update every 2 seconds
        highlightUpdateConnection = RunService.Heartbeat:Connect(function()
            if highlightEnabled then
                refreshHighlights()
            end
        end)
        
        -- Handle new players joining
        Players.PlayerAdded:Connect(function(player)
            if highlightEnabled then
                task.wait(0.5) -- Wait for character
                if player.Character then
                    addHighlight(player)
                end
            end
        end)
        
        -- Handle players leaving
        Players.PlayerRemoving:Connect(function(player)
            removeHighlight(player)
        end)
        
        -- Handle character respawns
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                player.CharacterAdded:Connect(function(char)
                    if highlightEnabled then
                        task.wait(0.5)
                        removeHighlight(player)
                        if char then
                            addHighlight(player)
                        end
                    end
                end)
            end
        end
        
    else
        -- Remove all highlights
        for player, _ in pairs(highlights) do
            removeHighlight(player)
        end
        
        if highlightUpdateConnection then
            highlightUpdateConnection:Disconnect()
            highlightUpdateConnection = nil
        end
    end
end

MainTab:CreateToggle({
    Name = "Highlight Players",
    CurrentValue = false,
    Flag = "HighlightToggle",
    Callback = function(value)
        toggleHighlight(value)
    end,
})

MainTab:CreateColorPicker({
    Name = "Highlight Color",
    Color = Color3.fromRGB(255, 0, 0),
    Flag = "HighlightColor",
    Callback = function(value)
        highlightColor = value
        for _, highlight in pairs(highlights) do
            highlight.FillColor = highlightColor
        end
    end,
})

MainTab:CreateSlider({
    Name = "Highlight Transparency",
    Range = {0, 1},
    Increment = 0.05,
    CurrentValue = 0.5,
    Flag = "HighlightTransparency",
    Callback = function(value)
        highlightTransparency = value
        for _, highlight in pairs(highlights) do
            highlight.FillTransparency = highlightTransparency
        end
    end,
})

------------------------------------------------------
-- AIMBOT FEATURE
------------------------------------------------------
local aimbotEnabled = false
local aimbotSmoothness = 0.2
local aimbotFOV = 90
local aimbotPart = "Head"
local aimbotLoopConnection = nil

local function getClosestPlayer()
    local camera = workspace.CurrentCamera
    local character = LocalPlayer.Character
    if not character then return nil end
    
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return nil end
    
    local closestPlayer = nil
    local closestDistance = aimbotFOV
    
    for _, targetPlayer in ipairs(Players:GetPlayers()) do
        if targetPlayer ~= LocalPlayer and targetPlayer.Character then
            local targetHumanoid = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
            if targetHumanoid and targetHumanoid.Health > 0 then
                local targetPart = targetPlayer.Character:FindFirstChild(aimbotPart)
                if targetPart then
                    local screenPos, onScreen = camera:WorldToViewportPoint(targetPart.Position)
                    if onScreen then
                        local distance = (targetPart.Position - rootPart.Position).Magnitude
                        if distance < closestDistance then
                            closestDistance = distance
                            closestPlayer = targetPlayer
                        end
                    end
                end
            end
        end
    end
    
    return closestPlayer
end

local function aimbotLoop()
    while aimbotEnabled do
        task.wait()
        if not aimbotEnabled then break end
        
        local targetPlayer = getClosestPlayer()
        if targetPlayer and targetPlayer.Character then
            local targetPart = targetPlayer.Character:FindFirstChild(aimbotPart)
            if targetPart then
                local camera = workspace.CurrentCamera
                local targetPosition = targetPart.Position
                local newCFrame = CFrame.new(camera.CFrame.Position, targetPosition)
                
                -- Smooth aim
                local smoothFactor = 1 - aimbotSmoothness
                camera.CFrame = camera.CFrame:Lerp(newCFrame, smoothFactor)
            end
        end
    end
end

local function toggleAimbot(value)
    aimbotEnabled = value
    if aimbotEnabled then
        task.spawn(aimbotLoop)
    end
end

MainTab:CreateToggle({
    Name = "Aimbot",
    CurrentValue = false,
    Flag = "AimbotToggle",
    Callback = function(value)
        toggleAimbot(value)
    end,
})

MainTab:CreateSlider({
    Name = "Aimbot Smoothness",
    Range = {0, 1},
    Increment = 0.05,
    CurrentValue = 0.2,
    Flag = "AimbotSmoothness",
    Callback = function(value)
        aimbotSmoothness = value
    end,
})

MainTab:CreateSlider({
    Name = "Aimbot FOV",
    Range = {30, 200},
    Increment = 5,
    CurrentValue = 90,
    Flag = "AimbotFOV",
    Callback = function(value)
        aimbotFOV = value
    end,
})

MainTab:CreateDropdown({
    Name = "Aimbot Target",
    Options = {"Head", "Torso", "HumanoidRootPart"},
    CurrentOption = "Head",
    Flag = "AimbotPart",
    Callback = function(option)
        aimbotPart = option
    end,
})

------------------------------------------------------
-- CLEANUP
------------------------------------------------------
LocalPlayer.CharacterAdded:Connect(function()
    if flying then
        task.wait(0.5)
        startFly()
    end
end)

Rayfield:Notify({
    Title = "Owner Tools Loaded",
    Content = "All features ready!",
    Duration = 3,
})

print("✅ Owner Tools loaded successfully!")
