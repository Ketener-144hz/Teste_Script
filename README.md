local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local VirtualUser = game:GetService("VirtualUser")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")

_G.Settings = {
    AutoFarm = false,
    AutoQuest = false,
    AutoCollectDrops = false,
    AutoEquipBest = false,
    AutoUpgrade = false,
    AutoBuySkills = false,
    AutoUseSkills = false,
    AutoRebirth = false,
    WebhookEnabled = false,
    WebhookURL = "",
    SelectedAreas = {},
    FarmingSpeed = 1,
    Performance = {
        ReduceParticles = true,
        DisableAnimations = true,
        LowGraphics = true
    }
}

local AREAS = {
    ["Starter Area"] = {
        Position = Vector3.new(0, 0, 0),
        RequiredLevel = 0,
        Mobs = {"Slime", "Goblin", "Wolf"}
    },
    ["Forest"] = {
        Position = Vector3.new(100, 0, 100),
        RequiredLevel = 10,
        Mobs = {"Forest Spider", "Tree Spirit", "Forest Golem"}
    },
    ["Desert"] = {
        Position = Vector3.new(-100, 0, 100),
        RequiredLevel = 25,
        Mobs = {"Sand Worm", "Mummy", "Desert Bandit"}
    }
}

local SKILLS = {
    ["Basic Attack"] = {
        Cost = 0,
        Cooldown = 0,
        Damage = 10
    },
    ["Fire Ball"] = {
        Cost = 1000,
        Cooldown = 5,
        Damage = 50
    },
    ["Ice Spike"] = {
        Cost = 2500,
        Cooldown = 8,
        Damage = 75
    }
}

local Utility = {}

function Utility.GetDistance(pos1, pos2)
    return (pos1 - pos2).Magnitude
end

function Utility.CreateTween(object, info, properties)
    local tween = TweenService:Create(object, info, properties)
    return tween
end

function Utility.Teleport(position)
    if typeof(position) == "CFrame" then
        HumanoidRootPart.CFrame = position
    else
        HumanoidRootPart.CFrame = CFrame.new(position)
    end
end

function Utility.GetClosestMob()
    local closest = nil
    local maxDistance = math.huge
    
    for _, mob in pairs(workspace.Mobs:GetChildren()) do
        if mob:FindFirstChild("HumanoidRootPart") and mob:FindFirstChild("Humanoid") and mob.Humanoid.Health > 0 then
            local distance = Utility.GetDistance(HumanoidRootPart.Position, mob.HumanoidRootPart.Position)
            if distance < maxDistance then
                maxDistance = distance
                closest = mob
            end
        end
    end
    
    return closest
end

function Utility.CollectDrops()
    for _, drop in pairs(workspace.Drops:GetChildren()) do
        if drop:IsA("BasePart") then
            Utility.Teleport(drop.Position)
            wait(0.1)
        end
    end
end

local Game = {}

function Game.GetPlayerLevel()
    return LocalPlayer.Stats.Level.Value
end

function Game.GetPlayerGold()
    return LocalPlayer.Stats.Gold.Value
end

function Game.GetPlayerExp()
    return LocalPlayer.Stats.Experience.Value
end

function Game.Attack(mob)
    local args = {
        [1] = "Attack",
        [2] = mob
    }
    ReplicatedStorage.GameEvents.Combat:FireServer(unpack(args))
end

function Game.UseSkill(skillName)
    local args = {
        [1] = "UseSkill",
        [2] = skillName
    }
    ReplicatedStorage.GameEvents.Skills:FireServer(unpack(args))
end

function Game.BuySkill(skillName)
    local args = {
        [1] = "BuySkill",
        [2] = skillName
    }
    ReplicatedStorage.GameEvents.Shop:FireServer(unpack(args))
end

function Game.Rebirth()
    local args = {
        [1] = "Rebirth"
    }
    ReplicatedStorage.GameEvents.Rebirth:FireServer(unpack(args))
end

local AutoFarm = {}

function AutoFarm.Start()
    if _G.AutoFarmConnection then return end
    
    _G.AutoFarmConnection = RunService.Heartbeat:Connect(function()
        if not _G.Settings.AutoFarm then return end
        
        local mob = Utility.GetClosestMob()
        if mob then
            Utility.Teleport(mob.HumanoidRootPart.CFrame * CFrame.new(0, 0, 3))
            Game.Attack(mob)
            
            if _G.Settings.AutoUseSkills then
                for skillName, _ in pairs(SKILLS) do
                    Game.UseSkill(skillName)
                end
            end
        end
        
        if _G.Settings.AutoCollectDrops then
            Utility.CollectDrops()
        end
        
        wait(_G.Settings.FarmingSpeed)
    end)
end

function AutoFarm.Stop()
    if _G.AutoFarmConnection then
        _G.AutoFarmConnection:Disconnect()
        _G.AutoFarmConnection = nil
    end
end

local Performance = {}

function Performance.OptimizeGame()
    if _G.Settings.Performance.ReduceParticles then
        for _, v in pairs(game:GetDescendants()) do
            if v:IsA("ParticleEmitter") or v:IsA("Fire") or v:IsA("Sparkles") then
                v.Enabled = false
            end
        end
    end
    
    if _G.Settings.Performance.DisableAnimations then
        for _, v in pairs(game:GetDescendants()) do
            if v:IsA("Animation") or v:IsA("AnimationController") then
                v:Destroy()
            end
        end
    end
    
    if _G.Settings.Performance.LowGraphics then
        settings().Rendering.QualityLevel = 1
        sethiddenproperty(game.Lighting, "Technology", 2)
        settings().Rendering.MeshPartDetailLevel = Enum.MeshPartDetailLevel.Level04
    end
end

local Library = loadstring(game:HttpGet("https://raw.githubusercontent.com/xHeptc/Kavo-UI-Library/main/source.lua"))()
local Window = Library.CreateLib("Cryptic Auto Farm", "Ocean")

local MainTab = Window:NewTab("Main")
local MainSection = MainTab:NewSection("Farming")

MainSection:NewToggle("Auto Farm", "Toggles auto farming", function(state)
    _G.Settings.AutoFarm = state
    if state then
        AutoFarm.Start()
    else
        AutoFarm.Stop()
    end
end)

MainSection:NewToggle("Auto Collect Drops", "Automatically collects drops", function(state)
    _G.Settings.AutoCollectDrops = state
end)

MainSection:NewSlider("Farming Speed", "Adjusts the farming speed", 0.1, 2, 1, 0.1, function(value)
    _G.Settings.FarmingSpeed = value
end)

local SkillsTab = Window:NewTab("Skills")
local SkillsSection = SkillsTab:NewSection("Skills Management")

SkillsSection:NewToggle("Auto Use Skills", "Automatically uses skills", function(state)
    _G.Settings.AutoUseSkills = state
end)

SkillsSection:NewToggle("Auto Buy Skills", "Automatically buys skills", function(state)
    _G.Settings.AutoBuySkills = state
end)

local AreasTab = Window:NewTab("Areas")
local AreasSection = AreasTab:NewSection("Area Selection")

for areaName, data in pairs(AREAS) do
    AreasSection:NewToggle(areaName, "Level "..data.RequiredLevel.." required", function(state)
        if state then
            table.insert(_G.Settings.SelectedAreas, areaName)
        else
            table.remove(_G.Settings.SelectedAreas, table.find(_G.Settings.SelectedAreas, areaName))
        end
    end)
end

local SettingsTab = Window:NewTab("Settings")
local SettingsSection = SettingsTab:NewSection("Performance Settings")

SettingsSection:NewToggle("Reduce Particles", "Reduces particle effects", function(state)
    _G.Settings.Performance.ReduceParticles = state
    Performance.OptimizeGame()
end)

SettingsSection:NewToggle("Low Graphics", "Enables low graphics mode", function(state)
    _G.Settings.Performance.LowGraphics = state
    Performance.OptimizeGame()
end)

Performance.OptimizeGame()

LocalPlayer.Idled:Connect(function()
    VirtualUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    wait(1)
    VirtualUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
end)

LocalPlayer.CharacterAdded:Connect(function(char)
    Character = char
    HumanoidRootPart = char:WaitForChild("HumanoidRootPart")
end)
