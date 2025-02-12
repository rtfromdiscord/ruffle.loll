local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Stats = game:GetService("Stats")
local StarterGui = game:GetService("StarterGui")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

getgenv().Settings = {
    Keybind = Enum.KeyCode.Q,
    ControllerKeybind = Enum.KeyCode.ButtonY,
    AutoPrediction = false,
    ManualPrediction = 0.1157,
    Smoothness = 0,
    Resolver = true,
    HitboxSizeMultiplier = 3
}

local AimLockEnabled = false
local Target = nil
local ActiveHitbox = nil
local ExitConfirm = false
local TriggerBotEnabled = false
local SpeedEnabled = false

local function notify(title, message)
    StarterGui:SetCore("SendNotification", {
        Title = title,
        Text = message,
        Duration = 2
    })
end

notify("ruffle.lol", "ruffle.lol has loaded")

local function getPing()
    return Stats.Network.ServerStatsItem["Data Ping"]:GetValue() / 1000
end

local function resolveTarget(target)
    if not getgenv().Settings.Resolver or not target then return target.Position end
    local predictionTime = getgenv().Settings.AutoPrediction and getPing() * 3.5 or getgenv().Settings.ManualPrediction
    return target.Position + (target.Velocity * predictionTime)
end

local function snapAim(target)
    if target then
        Camera.CFrame = CFrame.lookAt(Camera.CFrame.Position, resolveTarget(target))
    end
end

local function createHitbox(target)
    if ActiveHitbox then
        ActiveHitbox:Destroy()
        ActiveHitbox = nil
    end

    if target and target.Parent then
        local hrp = target.Parent:FindFirstChild("HumanoidRootPart")
        if hrp then
            local hitbox = Instance.new("Part")
            hitbox.Name = "ExpandedHitbox"
            hitbox.Size = hrp.Size * getgenv().Settings.HitboxSizeMultiplier
            hitbox.Transparency = 0.4
            hitbox.CanCollide = false
            hitbox.Anchored = false
            hitbox.Massless = true
            hitbox.Parent = hrp
            hitbox.CFrame = hrp.CFrame
            hitbox.Color = Color3.fromRGB(0, 0, 255)

            local weld = Instance.new("WeldConstraint")
            weld.Part0 = hrp
            weld.Part1 = hitbox
            weld.Parent = hitbox

            ActiveHitbox = hitbox
        end
    end
end

local function removeHitbox()
    if ActiveHitbox then
        ActiveHitbox:Destroy()
        ActiveHitbox = nil
    end
end

local function getClosestTarget()
    local closest, minDist = nil, math.huge
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local part = player.Character:FindFirstChild("HumanoidRootPart")
            if part then
                local correctedPos = resolveTarget(part)
                local screenPos, onScreen = Camera:WorldToViewportPoint(correctedPos)

                if onScreen then
                    local dist = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude
                    if dist < minDist then
                        minDist = dist
                        closest = part
                    end
                end
            end
        end
    end
    return closest
end

local function drawBox()
    local box = Drawing.new("Square")
    box.Color = Color3.fromRGB(255, 255, 255)
    box.Thickness = 2
    box.Filled = false
    box.Visible = false

    RunService.RenderStepped:Connect(function()
        if AimLockEnabled and Target then
            local screenPos, onScreen = Camera:WorldToViewportPoint(Target.Position)
            if onScreen then
                box.Size = Vector2.new(50, 50)
                box.Position = Vector2.new(screenPos.X - 25, screenPos.Y - 25)
                box.Visible = true
            else
                box.Visible = false
            end
        else
            box.Visible = false
        end
    end)
end

drawBox()

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    if input.KeyCode == getgenv().Settings.Keybind or input.KeyCode == getgenv().Settings.ControllerKeybind then
        AimLockEnabled = not AimLockEnabled
        if AimLockEnabled then
            Target = getClosestTarget()
            if Target and Target.Parent then
                local targetPlayer = Players:GetPlayerFromCharacter(Target.Parent)
                if targetPlayer then
                    notify("ruffle.lol", "You have locked onto " .. targetPlayer.DisplayName)
                    createHitbox(Target)
                end
            end
        else
            notify("ruffle.lol", "Unlocked")
            Target = nil
            removeHitbox()
        end
    end
end)

local function confirmExit()
    if not ExitConfirm then
        notify("Exit Confirmation", "Press Z again within 3 seconds to last log")
        ExitConfirm = true
        task.wait(3)
        ExitConfirm = false
    else
        game.ReplicatedStorage.DefaultChatSystemChatEvents.SayMessageRequest:FireServer("last", "All")
        game:Shutdown()
    end
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    if input.KeyCode == Enum.KeyCode.Z then
        confirmExit()
    end
end)

local function triggerBot()
    while TriggerBotEnabled do
        task.wait(0.1) 
        local mouse = LocalPlayer:GetMouse()
        if mouse.Target and mouse.Target.Parent then
            local character = mouse.Target.Parent
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                mouse1press()
                task.wait(0.1)
                mouse1release()
            end
        end
    end
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    if input.KeyCode == Enum.KeyCode.E then
        TriggerBotEnabled = not TriggerBotEnabled
        if TriggerBotEnabled then
            notify("Triggerbot", "Enabled")
            task.spawn(triggerBot)
        else
            notify("Triggerbot", "Disabled")
        end
    end
end)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    if input.KeyCode == Enum.KeyCode.H then
        local character = LocalPlayer.Character
        if character then
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                SpeedEnabled = not SpeedEnabled
                humanoid.WalkSpeed = SpeedEnabled and 120 or 16
                notify("Speed Boost", SpeedEnabled and "Enabled" or "Disabled")
            end
        end
    end
end)

RunService.RenderStepped:Connect(function()
    if AimLockEnabled and Target then
        snapAim(Target)
    end
end)
