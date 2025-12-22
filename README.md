local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local Fluent = loadstring(game:HttpGet("https://files.catbox.moe/pj2qb9.lua"))()
local Window = Fluent:CreateWindow({
    Title = "🥋 KARATEKA HUB V5",
    SubTitle = "by karatekaff | Enhanced Edition", 
    TabWidth = 160, 
    Theme = "VSC Dark High Contrast",
    Acrylic = false,
    Size = UDim2.fromOffset(580, 400), 
    MinimizeKey = Enum.KeyCode.End
})

local Config = {
    Aimbot = {
        Enabled = false,
        FOVSize = 150,
        Speed = 5,
        MaxDistance = 1000,
        ShowFOV = true,
        AimPart = "Head",
        Mode = "Camera",
        WallCheck = false,
        SmoothingType = "Linear",
        PredictionEnabled = false,
        PredictionAmount = 0.1,
    },
    
    ESP = {
        Enabled = false,
        ShowTracers = false,
        ShowSkeleton = false,
        ShowAura = false,
        ShowBoxes = false,
        ShowNames = false,
        ShowDistance = false,
        MaxDistance = 2000,
        TeamCheck = false,
    },
    
    Visual = {
        RainbowMode = true,
        ESPColor = Color3.fromRGB(255, 255, 255),
        FOVColor = Color3.fromRGB(255, 255, 255),
        TracerColor = Color3.fromRGB(255, 0, 0),
        SkeletonColor = Color3.fromRGB(255, 255, 255),
        AuraColor = Color3.fromRGB(0, 255, 255),
    },
    
    Misc = {
        NoRecoil = false,
        NoSpread = false,
        InfiniteStamina = false,
        WalkSpeed = 16,
        JumpPower = 50,
        SpeedEnabled = false,
        JumpEnabled = false,
    },
}

local IgnoredPlayers = {}
local ESPObjects = {}
local Connections = {}

local function GetRainbowColor()
    local hue = (tick() % 5) / 5
    return Color3.fromHSV(hue, 1, 1)
end

local function IsVisible(targetPart)
    if not Config.Aimbot.WallCheck then return true end
    
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin).Unit * 1000
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {LocalPlayer.Character, targetPart.Parent}
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    
    local result = workspace:Raycast(origin, direction, raycastParams)
    
    if result then
        local distToHit = (result.Position - origin).Magnitude
        local distToTarget = (targetPart.Position - origin).Magnitude
        return distToTarget < distToHit
    end
    
    return true
end

local function GetVelocity(character)
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    if humanoidRootPart then
        return humanoidRootPart.AssemblyLinearVelocity
    end
    return Vector3.new(0, 0, 0)
end

local function PredictPosition(targetPart, velocity)
    if not Config.Aimbot.PredictionEnabled then
        return targetPart.Position
    end
    return targetPart.Position + (velocity * Config.Aimbot.PredictionAmount)
end

local function GetClosestEnemy()
    local closest = nil
    local bestDistance = math.huge
    local mouseLocation = UserInputService:GetMouseLocation()
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and not IgnoredPlayers[player] then
            local character = player.Character
            local targetPart = character:FindFirstChild(Config.Aimbot.AimPart)
            
            if targetPart then
                
                if not IsVisible(targetPart) then
                    continue
                end
                
                local partPosition = targetPart.Position
                local screenPosition, onScreen = Camera:WorldToViewportPoint(partPosition)
                
                if onScreen then
                    local screenVector = Vector2.new(screenPosition.X, screenPosition.Y)
                    local cursorDistance = (screenVector - mouseLocation).Magnitude
                    local worldDistance = (Camera.CFrame.Position - partPosition).Magnitude
                    
                    
                    if cursorDistance < Config.Aimbot.FOVSize and 
                       worldDistance <= Config.Aimbot.MaxDistance and 
                       cursorDistance < bestDistance then
                        bestDistance = cursorDistance
                        closest = player
                    end
                end
            end
        end
    end
    
    return closest
end


local function CreateESP(player)
    if ESPObjects[player] then return end
    
    local espData = {
        Tracer = Drawing.new("Line"),
        Aura = Drawing.new("Circle"),
        Box = Drawing.new("Square"),
        Skeleton = {},
    }
    
    
    espData.Tracer.Thickness = 2
    espData.Tracer.Color = Config.Visual.TracerColor
    espData.Tracer.Transparency = 1
    
    
    espData.Aura.Thickness = 2
    espData.Aura.NumSides = 32
    espData.Aura.Radius = 8
    espData.Aura.Filled = false
    espData.Aura.Transparency = 1
    
    
    espData.Box.Thickness = 2
    espData.Box.Color = Config.Visual.ESPColor
    espData.Box.Filled = false
    espData.Box.Transparency = 1
    
    
    for i = 1, 6 do
        espData.Skeleton[i] = Drawing.new("Line")
        espData.Skeleton[i].Thickness = 1.5
        espData.Skeleton[i].Color = Config.Visual.SkeletonColor
        espData.Skeleton[i].Transparency = 1
    end
    
    ESPObjects[player] = espData
end

local function UpdateESP()
    for player, espData in pairs(ESPObjects) do
        if not player or not player.Character then
            espData.Tracer.Visible = false
            espData.Aura.Visible = false
            espData.Box.Visible = false
            for _, line in ipairs(espData.Skeleton) do
                line.Visible = false
            end
            continue
        end
        
        local character = player.Character
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        local head = character:FindFirstChild("Head")
        
        if not rootPart then
            espData.Tracer.Visible = false
            espData.Aura.Visible = false
            espData.Box.Visible = false
            for _, line in ipairs(espData.Skeleton) do
                line.Visible = false
            end
            continue
        end
        
        local distance = (Camera.CFrame.Position - rootPart.Position).Magnitude
        
        if distance > Config.ESP.MaxDistance then
            espData.Tracer.Visible = false
            espData.Aura.Visible = false
            espData.Box.Visible = false
            for _, line in ipairs(espData.Skeleton) do
                line.Visible = false
            end
            continue
        end
        
        
        if Config.Visual.RainbowMode then
            local rainbowColor = GetRainbowColor()
            espData.Tracer.Color = rainbowColor
            espData.Aura.Color = rainbowColor
            espData.Box.Color = rainbowColor
            for _, line in ipairs(espData.Skeleton) do
                line.Color = rainbowColor
            end
        end
        
        
        if Config.ESP.ShowTracers and Config.ESP.Enabled then
            local screenPos, onScreen = Camera:WorldToViewportPoint(rootPart.Position)
            if onScreen then
                espData.Tracer.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                espData.Tracer.To = Vector2.new(screenPos.X, screenPos.Y)
                espData.Tracer.Visible = true
            else
                espData.Tracer.Visible = false
            end
        else
            espData.Tracer.Visible = false
        end
        
        
        if Config.ESP.ShowAura and Config.ESP.Enabled and head then
            local headPos, headVisible = Camera:WorldToViewportPoint(head.Position)
            if headVisible then
                espData.Aura.Position = Vector2.new(headPos.X, headPos.Y)
                espData.Aura.Visible = true
            else
                espData.Aura.Visible = false
            end
        else
            espData.Aura.Visible = false
        end
        
        
        if Config.ESP.ShowSkeleton and Config.ESP.Enabled then
            local parts = {
                head = character:FindFirstChild("Head"),
                torso = character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso"),
                leftArm = character:FindFirstChild("Left Arm") or character:FindFirstChild("LeftUpperArm"),
                rightArm = character:FindFirstChild("Right Arm") or character:FindFirstChild("RightUpperArm"),
                leftLeg = character:FindFirstChild("Left Leg") or character:FindFirstChild("LeftUpperLeg"),
                rightLeg = character:FindFirstChild("Right Leg") or character:FindFirstChild("RightUpperLeg"),
            }
            
            if parts.head and parts.torso then
                local connections = {
                    {parts.head, parts.torso, 1},
                    {parts.torso, parts.leftArm, 2},
                    {parts.torso, parts.rightArm, 3},
                    {parts.torso, parts.leftLeg, 4},
                    {parts.torso, parts.rightLeg, 5},
                }
                
                for _, connection in ipairs(connections) do
                    local part1, part2, index = connection[1], connection[2], connection[3]
                    if part1 and part2 then
                        local pos1, vis1 = Camera:WorldToViewportPoint(part1.Position)
                        local pos2, vis2 = Camera:WorldToViewportPoint(part2.Position)
                        
                        if vis1 and vis2 then
                            espData.Skeleton[index].From = Vector2.new(pos1.X, pos1.Y)
                            espData.Skeleton[index].To = Vector2.new(pos2.X, pos2.Y)
                            espData.Skeleton[index].Visible = true
                        else
                            espData.Skeleton[index].Visible = false
                        end
                    else
                        espData.Skeleton[index].Visible = false
                    end
                end
            else
                for _, line in ipairs(espData.Skeleton) do
                    line.Visible = false
                end
            end
        else
            for _, line in ipairs(espData.Skeleton) do
                line.Visible = false
            end
        end
    end
end

local function RemoveESP(player)
    if ESPObjects[player] then
        ESPObjects[player].Tracer:Remove()
        ESPObjects[player].Aura:Remove()
        ESPObjects[player].Box:Remove()
        for _, line in ipairs(ESPObjects[player].Skeleton) do
            line:Remove()
        end
        ESPObjects[player] = nil
    end
end


local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness = 2
FOVCircle.NumSides = 64
FOVCircle.Filled = false
FOVCircle.Transparency = 1

local aimHeld = false

local function SmoothAim(currentPos, targetPos, smoothing, smoothType)
    if smoothType == "Exponential" then
        return currentPos:Lerp(targetPos, 1 / smoothing)
    else -- Linear
        local delta = targetPos - currentPos
        local distance = delta.Magnitude
        if distance > 1 then
            return currentPos + (delta.Unit * math.min(distance, distance / smoothing))
        end
        return targetPos
    end
end

local function UpdateAimbot()
    if not Config.Aimbot.Enabled or not aimHeld then return end
    
    local target = GetClosestEnemy()
    if not target or not target.Character then return end
    
    local targetPart = target.Character:FindFirstChild(Config.Aimbot.AimPart)
    if not targetPart then return end
    
    local velocity = GetVelocity(target.Character)
    local targetPosition = PredictPosition(targetPart, velocity)
    local smoothing = math.max(1, Config.Aimbot.Speed)
    
    if Config.Aimbot.Mode == "Camera" then
        
        local mousePos = UserInputService:GetMouseLocation()
        local partPos = Camera:WorldToViewportPoint(targetPosition)
        local delta = Vector2.new(partPos.X - mousePos.X, partPos.Y - mousePos.Y)
        
        if delta.Magnitude > 1 then
            if mousemoverel then
                pcall(function()
                    mousemoverel(delta.X / smoothing, delta.Y / smoothing)
                end)
            end
        end
        
        
        local currentCFrame = Camera.CFrame
        local lookAtCFrame = CFrame.lookAt(currentCFrame.Position, targetPosition)
        Camera.CFrame = currentCFrame:Lerp(lookAtCFrame, 1 / smoothing)
        
    elseif Config.Aimbot.Mode == "Mouse" then
        
        local mousePos = UserInputService:GetMouseLocation()
        local partPos = Camera:WorldToViewportPoint(targetPosition)
        local delta = Vector2.new(partPos.X - mousePos.X, partPos.Y - mousePos.Y)
        
        if delta.Magnitude > 1 then
            if mousemoverel then
                pcall(function()
                    mousemoverel(delta.X / smoothing, delta.Y / smoothing)
                end)
            end
        end
    end
end


local function ApplyNoRecoil()
    if not Config.Misc.NoRecoil then return end
    
    local character = LocalPlayer.Character
    if not character then return end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid.CameraOffset = Vector3.new(0, 0, 0)
    end
end

local function ApplySpeedHack()
    local character = LocalPlayer.Character
    if not character then return end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid and Config.Misc.SpeedEnabled then
        humanoid.WalkSpeed = Config.Misc.WalkSpeed
    end
end

local function ApplyJumpHack()
    local character = LocalPlayer.Character
    if not character then return end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid and Config.Misc.JumpEnabled then
        humanoid.JumpPower = Config.Misc.JumpPower
    end
end


local Tabs = {
    Aimbot = Window:AddTab({Title = "🎯 Aimbot", Icon = "target"}),
    ESP = Window:AddTab({Title = "👁️ ESP", Icon = "eye"}),
    Players = Window:AddTab({Title = "👥 Players", Icon = "users"}),
    Misc = Window:AddTab({Title = "⚙️ Misc", Icon = "settings"}),
    Settings = Window:AddTab({Title = "🎨 Visual", Icon = "palette"}),
}


local AimbotSection = Tabs.Aimbot:AddSection("Aimbot Settings")

Tabs.Aimbot:AddToggle("AimbotEnabled", {
    Title = "Enable Aimbot",
    Description = "Toggle aimbot on/off",
    Default = Config.Aimbot.Enabled,
    Callback = function(value)
        Config.Aimbot.Enabled = value
    end
})

Tabs.Aimbot:AddToggle("ShowFOV", {
    Title = "Show FOV Circle",
    Description = "Display the field of view circle",
    Default = Config.Aimbot.ShowFOV,
    Callback = function(value)
        Config.Aimbot.ShowFOV = value
    end
})

Tabs.Aimbot:AddSlider("FOVSize", {
    Title = "FOV Size",
    Description = "Adjust the field of view radius",
    Default = Config.Aimbot.FOVSize,
    Min = 20,
    Max = 600,
    Rounding = 0,
    Callback = function(value)
        Config.Aimbot.FOVSize = value
    end
})

Tabs.Aimbot:AddSlider("AimbotSpeed", {
    Title = "Smoothing",
    Description = "Adjust aimbot smoothing (lower = faster)",
    Default = Config.Aimbot.Speed,
    Min = 1,
    Max = 100,
    Rounding = 0,
    Callback = function(value)
        Config.Aimbot.Speed = value
    end
})

Tabs.Aimbot:AddSlider("MaxDistance", {
    Title = "Max Distance",
    Description = "Maximum targeting distance",
    Default = Config.Aimbot.MaxDistance,
    Min = 200,
    Max = 5000,
    Rounding = 0,
    Callback = function(value)
        Config.Aimbot.MaxDistance = value
    end
})

local TargetSection = Tabs.Aimbot:AddSection("Target Settings")

Tabs.Aimbot:AddDropdown("AimPart", {
    Title = "Aim Part",
    Description = "Select which body part to target",
    Values = {"Head", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg"},
    Default = Config.Aimbot.AimPart,
    Callback = function(value)
        Config.Aimbot.AimPart = value
    end
})

Tabs.Aimbot:AddDropdown("AimbotMode", {
    Title = "Aimbot Mode",
    Description = "Select aimbot movement mode",
    Values = {"Camera", "Mouse"},
    Default = Config.Aimbot.Mode,
    Callback = function(value)
        Config.Aimbot.Mode = value
    end
})

Tabs.Aimbot:AddToggle("WallCheck", {
    Title = "Wall Check",
    Description = "Ignore targets behind walls",
    Default = Config.Aimbot.WallCheck,
    Callback = function(value)
        Config.Aimbot.WallCheck = value
    end
})

Tabs.Aimbot:AddToggle("PredictionEnabled", {
    Title = "Prediction",
    Description = "Predict moving targets",
    Default = Config.Aimbot.PredictionEnabled,
    Callback = function(value)
        Config.Aimbot.PredictionEnabled = value
    end
})

Tabs.Aimbot:AddSlider("PredictionAmount", {
    Title = "Prediction Amount",
    Description = "Adjust prediction strength",
    Default = Config.Aimbot.PredictionAmount,
    Min = 0,
    Max = 1,
    Rounding = 2,
    Callback = function(value)
        Config.Aimbot.PredictionAmount = value
    end
})


local ESPMainSection = Tabs.ESP:AddSection("ESP Features")

Tabs.ESP:AddToggle("ESPEnabled", {
    Title = "Enable ESP",
    Description = "Toggle ESP system",
    Default = Config.ESP.Enabled,
    Callback = function(value)
        Config.ESP.Enabled = value
    end
})

Tabs.ESP:AddToggle("ShowTracers", {
    Title = "Tracers",
    Description = "Show lines to players",
    Default = Config.ESP.ShowTracers,
    Callback = function(value)
        Config.ESP.ShowTracers = value
    end
})

Tabs.ESP:AddToggle("ShowSkeleton", {
    Title = "Skeleton",
    Description = "Show player skeletons",
    Default = Config.ESP.ShowSkeleton,
    Callback = function(value)
        Config.ESP.ShowSkeleton = value
    end
})

Tabs.ESP:AddToggle("ShowAura", {
    Title = "Aura (Hotkey: P)",
    Description = "Show aura circle around heads",
    Default = Config.ESP.ShowAura,
    Callback = function(value)
        Config.ESP.ShowAura = value
    end
})

Tabs.ESP:AddSlider("ESPDistance", {
    Title = "ESP Distance",
    Description = "Maximum ESP render distance",
    Default = Config.ESP.MaxDistance,
    Min = 100,
    Max = 5000,
    Rounding = 0,
    Callback = function(value)
        Config.ESP.MaxDistance = value
    end
})


local PlayersSection = Tabs.Players:AddSection("Player Management")

Tabs.Players:AddParagraph({
    Title = "Ignore Players",
    Content = "Click on players below to ignore them in aimbot"
})

local PlayerList = Tabs.Players:AddSection("Connected Players")

local function UpdatePlayerList()
    -- This would need to be implemented with Fluent's dynamic content system
    -- For now, we'll create a simple text display
    local playerNames = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local status = IgnoredPlayers[player] and "[IGNORED]" or "[ACTIVE]"
            table.insert(playerNames, status .. " " .. player.Name)
        end
    end
end


local MiscSection = Tabs.Misc:AddSection("Gameplay Modifications")

Tabs.Misc:AddToggle("NoRecoil", {
    Title = "No Recoil",
    Description = "Remove weapon recoil",
    Default = Config.Misc.NoRecoil,
    Callback = function(value)
        Config.Misc.NoRecoil = value
    end
})

Tabs.Misc:AddToggle("SpeedEnabled", {
    Title = "Speed Hack",
    Description = "Enable custom walk speed",
    Default = Config.Misc.SpeedEnabled,
    Callback = function(value)
        Config.Misc.SpeedEnabled = value
        if not value then
            local humanoid = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid.WalkSpeed = 16
            end
        end
    end
})

Tabs.Misc:AddSlider("WalkSpeed", {
    Title = "Walk Speed",
    Description = "Adjust movement speed",
    Default = Config.Misc.WalkSpeed,
    Min = 16,
    Max = 200,
    Rounding = 0,
    Callback = function(value)
        Config.Misc.WalkSpeed = value
    end
})

Tabs.Misc:AddToggle("JumpEnabled", {
    Title = "Jump Hack",
    Description = "Enable custom jump power",
    Default = Config.Misc.JumpEnabled,
    Callback = function(value)
        Config.Misc.JumpEnabled = value
        if not value then
            local humanoid = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid.JumpPower = 50
            end
        end
    end
})

Tabs.Misc:AddSlider("JumpPower", {
    Title = "Jump Power",
    Description = "Adjust jump height",
    Default = Config.Misc.JumpPower,
    Min = 50,
    Max = 200,
    Rounding = 0,
    Callback = function(value)
        Config.Misc.JumpPower = value
    end
})


local VisualSection = Tabs.Settings:AddSection("Visual Settings")

Tabs.Settings:AddToggle("RainbowMode", {
    Title = "Rainbow Mode",
    Description = "Enable rainbow colors for ESP and FOV",
    Default = Config.Visual.RainbowMode,
    Callback = function(value)
        Config.Visual.RainbowMode = value
    end
})

local InfoSection = Tabs.Settings:AddSection("Information")

Tabs.Settings:AddParagraph({
    Title = "Hotkeys",
    Content = "• END - Toggle Menu\n• P - Toggle Aura ESP\n• Left Click - Aim (when aimbot enabled)"
})

Tabs.Settings:AddParagraph({
    Title = "Features",
    Content = "• Advanced aimbot with prediction\n• Multiple ESP options\n• Wall check system\n• Player ignore list\n• Speed & jump modifications"
})



Connections.InputBegan = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        aimHeld = true
    elseif input.KeyCode == Enum.KeyCode.P then
        Config.ESP.ShowAura = not Config.ESP.ShowAura
        Fluent:Notify({
            Title = "Aura ESP",
            Content = Config.ESP.ShowAura and "Enabled" or "Disabled",
            Duration = 2
        })
    end
end)

Connections.InputEnded = UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        aimHeld = false
    end
end)


Connections.PlayerAdded = Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then
        CreateESP(player)
        UpdatePlayerList()
    end
end)

Connections.PlayerRemoving = Players.PlayerRemoving:Connect(function(player)
    RemoveESP(player)
    IgnoredPlayers[player] = nil
    UpdatePlayerList()
end)


Connections.CharacterAdded = LocalPlayer.CharacterAdded:Connect(function(character)
    character:WaitForChild("Humanoid")
    wait(0.1)
    ApplySpeedHack()
    ApplyJumpHack()
end)


Connections.RenderStepped = RunService.RenderStepped:Connect(function()
    
    FOVCircle.Position = UserInputService:GetMouseLocation()
    FOVCircle.Radius = Config.Aimbot.FOVSize
    FOVCircle.Visible = Config.Aimbot.ShowFOV
    
    if Config.Visual.RainbowMode then
        FOVCircle.Color = GetRainbowColor()
    else
        FOVCircle.Color = Config.Visual.FOVColor
    end
    
    
    UpdateESP()
    
    
    UpdateAimbot()
end)


Connections.Heartbeat = RunService.Heartbeat:Connect(function()
    ApplyNoRecoil()
    ApplySpeedHack()
    ApplyJumpHack()
end)

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end

UpdatePlayerList()

Fluent:Notify({
    Title = "🥋 KARATEKA HUB V5",
    Content = "Successfully loaded! Press END to toggle menu.",
    Duration = 5
})
