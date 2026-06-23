-- TianHub.lua
-- LocalScript: place in StarterPlayer > StarterPlayerScripts
-- Requires Rayfield UI library

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer

------------------------------------------------------
-- LOAD RAYFIELD
------------------------------------------------------
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "Tian Hub",
    LoadingTitle = "Tian Hub",
    LoadingSubtitle = "Loaded Successfully",
    ConfigurationSaving = {
        Enabled = false,
    },
    Discord = {
        Enabled = false,
    },
    KeySystem = false,
})

local AutoFarmTab = Window:CreateTab("AutoFarm", 4483362458)

------------------------------------------------------
-- AUTO FARM VARIABLES
------------------------------------------------------
local autoFarmEnabled = false
local farmSpeed = 50
local farmLoop = nil
local currentVehicle = nil
local waypointIndex = 1
local waypoints = {}
local isTurning = false
local turnTimer = 0

-- Waypoint system - you can customize these points
local function setupWaypoints()
    waypoints = {
        {x = 0, z = 50},   -- Forward
        {x = 50, z = 50},  -- Right
        {x = 50, z = 0},   -- Back
        {x = 0, z = 0},    -- Left
    }
    waypointIndex = 1
end

setupWaypoints()

------------------------------------------------------
-- GET VEHICLE FUNCTIONS
------------------------------------------------------
local function getPlayerVehicle()
    local char = LocalPlayer.Character
    if not char then return nil end
    
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return nil end
    
    local seatPart = humanoid.SeatPart
    if seatPart and seatPart:IsA("VehicleSeat") then
        local vehicle = seatPart:FindFirstAncestorOfClass("Model")
        return vehicle, seatPart
    end
    return nil, nil
end

local function getVehicleRoot(vehicle)
    if not vehicle then return nil end
    return vehicle:FindFirstChild("PrimaryPart") or vehicle:FindFirstChild("HumanoidRootPart") or vehicle:FindFirstChildOfClass("Part")
end

------------------------------------------------------
-- AUTO FARM LOGIC
------------------------------------------------------
local function getTargetPosition()
    if #waypoints == 0 then
        setupWaypoints()
    end
    
    local waypoint = waypoints[waypointIndex]
    if not waypoint then
        waypointIndex = 1
        waypoint = waypoints[1]
    end
    
    -- Get player position for reference
    local char = LocalPlayer.Character
    if not char then return nil end
    
    local rootPart = char:FindFirstChild("HumanoidRootPart")
    if not rootPart then return nil end
    
    -- Return the target position with current Y
    return Vector3.new(waypoint.x, rootPart.Position.Y, waypoint.z)
end

local function nextWaypoint()
    waypointIndex = waypointIndex + 1
    if waypointIndex > #waypoints then
        waypointIndex = 1
    end
end

local function autoFarmStep()
    if not autoFarmEnabled then return end
    
    local vehicle, seat = getPlayerVehicle()
    if not vehicle or not seat then 
        -- Not in vehicle, try to find nearby vehicle to enter
        findAndEnterVehicle()
        return 
    end
    
    currentVehicle = vehicle
    
    -- Get vehicle root
    local root = getVehicleRoot(vehicle)
    if not root then return end
    
    -- Get target position
    local targetPos = getTargetPosition()
    if not targetPos then return end
    
    -- Calculate direction to waypoint
    local direction = (targetPos - root.Position)
    local distance = direction.Magnitude
    
    -- If we're close enough to waypoint, go to next
    if distance < 10 then
        nextWaypoint()
        return
    end
    
    -- Normalize direction (ignore Y for steering)
    local flatDirection = Vector3.new(direction.X, 0, direction.Z)
    if flatDirection.Magnitude > 0 then
        flatDirection = flatDirection.Unit
    end
    
    -- Get vehicle forward direction
    local forward = root.CFrame.LookVector
    local flatForward = Vector3.new(forward.X, 0, forward.Z)
    if flatForward.Magnitude > 0 then
        flatForward = flatForward.Unit
    end
    
    -- Calculate angle between forward and target direction
    local dot = flatForward.X * flatDirection.X + flatForward.Z * flatDirection.Z
    local angle = math.acos(math.clamp(dot, -1, 1))
    
    -- Cross product to determine turn direction
    local cross = flatForward.X * flatDirection.Z - flatForward.Z * flatDirection.X
    local steerValue = math.clamp(angle * 1.5, -1, 1) * (cross > 0 and 1 or -1)
    
    -- Apply controls
    seat.Throttle = math.min(1, farmSpeed / 100) -- Normalize speed
    
    -- Steering with smooth turning
    if math.abs(steerValue) > 0.1 then
        seat.Steer = steerValue
        isTurning = true
    else
        seat.Steer = 0
        isTurning = false
    end
    
    -- Anti-stuck system
    if root.AssemblyLinearVelocity.Magnitude < 5 then
        -- If stuck, give boost
        root.AssemblyLinearVelocity = root.CFrame.LookVector * 30
    end
    
    -- Prevent flipping
    if math.abs(root.Rotation.X) > 30 or math.abs(root.Rotation.Z) > 30 then
        root.CFrame = CFrame.new(root.Position) * CFrame.Angles(0, root.Rotation.Y, 0)
    end
end

------------------------------------------------------
-- FIND AND ENTER VEHICLE
------------------------------------------------------
local function findAndEnterVehicle()
    if not autoFarmEnabled then return end
    
    local char = LocalPlayer.Character
    if not char then return end
    
    local rootPart = char:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    
    -- Find nearest vehicle
    local nearestVehicle = nil
    local nearestDist = 50 -- Search radius
    
    for _, model in ipairs(Workspace:GetChildren()) do
        if model:IsA("Model") and model:FindFirstChild("VehicleSeat") then
            local vehicleRoot = getVehicleRoot(model)
            if vehicleRoot then
                local dist = (vehicleRoot.Position - rootPart.Position).Magnitude
                if dist < nearestDist then
                    nearestDist = dist
                    nearestVehicle = model
                end
            end
        end
    end
    
    -- Enter vehicle if found
    if nearestVehicle then
        local seat = nearestVehicle:FindFirstChild("VehicleSeat")
        if seat and seat:IsA("VehicleSeat") then
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid.Sit = true
                -- Move to seat position
                rootPart.CFrame = CFrame.new(seat.Position + Vector3.new(0, 3, 0))
                task.wait(0.5)
                humanoid:MoveTo(seat.Position)
            end
        end
    end
end

------------------------------------------------------
-- AUTO FARM LOOP
------------------------------------------------------
local function startAutoFarm()
    if farmLoop then return end
    
    autoFarmEnabled = true
    
    farmLoop = RunService.Heartbeat:Connect(function()
        if autoFarmEnabled then
            autoFarmStep()
        end
    end)
end

local function stopAutoFarm()
    autoFarmEnabled = false
    
    if farmLoop then
        farmLoop:Disconnect()
        farmLoop = nil
    end
    
    -- Stop vehicle
    local _, seat = getPlayerVehicle()
    if seat then
        seat.Throttle = 0
        seat.Steer = 0
    end
end

------------------------------------------------------
-- UI CONTROLS
------------------------------------------------------
AutoFarmTab:CreateToggle({
    Name = "Auto Farm",
    CurrentValue = false,
    Flag = "AutoFarmToggle",
    Callback = function(value)
        if value then
            startAutoFarm()
            Rayfield:Notify({
                Title = "Tian Hub",
                Content = "AutoFarm started!",
                Duration = 2,
            })
        else
            stopAutoFarm()
            Rayfield:Notify({
                Title = "Tian Hub",
                Content = "AutoFarm stopped!",
                Duration = 2,
            })
        end
    end,
})

AutoFarmTab:CreateSlider({
    Name = "Farm Speed",
    Range = {20, 100},
    Increment = 5,
    CurrentValue = 50,
    Flag = "FarmSpeed",
    Callback = function(value)
        farmSpeed = value
    end,
})

AutoFarmTab:CreateButton({
    Name = "Find Nearest Vehicle",
    Callback = function()
        findAndEnterVehicle()
        Rayfield:Notify({
            Title = "Tian Hub",
            Content = "Searching for nearest vehicle...",
            Duration = 2,
        })
    end,
})

-- Add waypoint management
AutoFarmTab:CreateButton({
    Name = "Add Current Position as Waypoint",
    Callback = function()
        local char = LocalPlayer.Character
        if char then
            local root = char:FindFirstChild("HumanoidRootPart")
            if root then
                table.insert(waypoints, {x = root.Position.X, z = root.Position.Z})
                Rayfield:Notify({
                    Title = "Tian Hub",
                    Content = string.format("Waypoint %d added at position (%.1f, %.1f)", #waypoints, root.Position.X, root.Position.Z),
                    Duration = 3,
                })
            end
        end
    end,
})

AutoFarmTab:CreateButton({
    Name = "Reset Waypoints",
    Callback = function()
        setupWaypoints()
        waypointIndex = 1
        Rayfield:Notify({
            Title = "Tian Hub",
            Content = "Waypoints reset to default",
            Duration = 2,
        })
    end,
})

-- Status display
AutoFarmTab:CreateParagraph({
    Title = "Status",
    Content = "Current Waypoint: " .. waypointIndex .. "/" .. #waypoints,
})

-- Force stop button
AutoFarmTab:CreateButton({
    Name = "STOP VEHICLE",
    Callback = function()
        local _, seat = getPlayerVehicle()
        if seat then
            seat.Throttle = 0
            seat.Steer = 0
        end
        Rayfield:Notify({
            Title = "Tian Hub",
            Content = "Vehicle has been stopped",
            Duration = 2,
        })
    end,
})

------------------------------------------------------
-- ENTER VEHICLE DETECTION
------------------------------------------------------
LocalPlayer.CharacterAdded:Connect(function(char)
    task.wait(1)
    -- Auto-restart farm if enabled
    if autoFarmEnabled then
        startAutoFarm()
    end
end)

------------------------------------------------------
-- FALLBACK - GET OUT OF VEHICLE IF STUCK
------------------------------------------------------
local function checkIfStuck()
    while wait(5) do
        if autoFarmEnabled then
            local vehicle, _ = getPlayerVehicle()
            if vehicle then
                local root = getVehicleRoot(vehicle)
                if root and root.AssemblyLinearVelocity.Magnitude < 1 then
                    -- Try to unstick by going backwards
                    local seat = vehicle:FindFirstChild("VehicleSeat")
                    if seat then
                        seat.Throttle = -1
                        task.wait(1)
                        seat.Throttle = 0
                    end
                end
            end
        end
    end
end

-- Start stuck checker
task.spawn(checkIfStuck)

------------------------------------------------------
-- NOTIFICATION
------------------------------------------------------
Rayfield:Notify({
    Title = "Tian Hub",
    Content = "System loaded! Get in a vehicle and start farming!",
    Duration = 4,
})

print("✅ Tian Hub loaded successfully!")
