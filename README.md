-- ASTD Advanced Auto Farm Script
-- Created by Cascade AI

-- Services
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local VirtualUser = game:GetService("VirtualUser")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- Variables
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = workspace.CurrentCamera

-- Configuration
_G.Settings = {
    AutoFarm = false,
    AutoUpgrade = false,
    AutoSell = false,
    AutoAbility = false,
    AutoRaid = false,
    AutoInfinite = false,
    AutoStory = false,
    AutoChallenge = false,
    AutoCollectRewards = false,
    AutoSpinBanner = false,
    AutoEquipBest = false,
    WebhookEnabled = false,
    WebhookURL = "",
    SelectedUnits = {},
    UnitPlacementPositions = {},
    UpgradeOrder = {},
    SellAtWave = {15, 25, 40},
    MinimumMoneyKeep = 1000,
    PlacementDelay = 0.5,
    UpgradeDelay = 0.2,
    AbilityDelay = 0.2,
    AutoRetry = true,
    AutoNextLevel = true,
    AutoJoinRaid = true,
    AutoAcceptCarry = true,
    AutoClaimQuests = true,
    AutoClaimAchievements = true,
    AutoBuyShop = false,
    ShopItems = {},
    AntiAFK = true,
    Performance = {
        ReduceParticles = true,
        DisableAnimations = true,
        LowGraphics = true
    }
}

-- Constants
local DIFFICULTY_MULTIPLIERS = {
    ["Easy"] = 1,
    ["Normal"] = 1.5,
    ["Hard"] = 2,
    ["Extreme"] = 3,
    ["Nightmare"] = 4
}

local RARITY_COLORS = {
    ["Common"] = Color3.fromRGB(200, 200, 200),
    ["Rare"] = Color3.fromRGB(0, 100, 255),
    ["Epic"] = Color3.fromRGB(170, 0, 255),
    ["Legendary"] = Color3.fromRGB(255, 215, 0),
    ["Mythical"] = Color3.fromRGB(255, 0, 0)
}

-- Expanded Unit Data
local UnitData = {
    -- Story Mode Units
    ["Naruto"] = {
        Cost = 500,
        Upgrade_Costs = {700, 1500, 2500, 4000, 6000},
        Abilities = {
            ["Rasengan"] = {Cooldown = 15, Damage = 1000},
            ["Shadow Clone"] = {Cooldown = 20, Duration = 10}
        },
        Range = 15,
        AttackSpeed = 1.2,
        Rarity = "Epic"
    },
    ["Deku"] = {
        Cost = 450,
        Upgrade_Costs = {600, 1200, 2000, 3500, 5000},
        Abilities = {
            ["Detroit Smash"] = {Cooldown = 12, Damage = 800},
            ["Full Cowling"] = {Cooldown = 25, Duration = 8}
        },
        Range = 12,
        AttackSpeed = 1.0,
        Rarity = "Rare"
    },
    ["Goku"] = {
        Cost = 600,
        Upgrade_Costs = {800, 1600, 3000, 5000, 7000},
        Abilities = {
            ["Kamehameha"] = {Cooldown = 18, Damage = 1200},
            ["Instant Transmission"] = {Cooldown = 22, Duration = 12}
        },
        Range = 18,
        AttackSpeed = 1.1,
        Rarity = "Epic"
    },
    ["Ichigo"] = {
        Cost = 550,
        Upgrade_Costs = {750, 1400, 2200, 3800, 5500},
        Abilities = {
            ["Getsuga Tensho"] = {Cooldown = 15, Damage = 1000},
            ["Bankai"] = {Cooldown = 20, Duration = 10}
        },
        Range = 15,
        AttackSpeed = 1.2,
        Rarity = "Epic"
    },
    ["Luffy"] = {
        Cost = 500,
        Upgrade_Costs = {650, 1300, 2100, 3600, 5200},
        Abilities = {
            ["Gum-Gum Fruit"] = {Cooldown = 12, Damage = 800},
            ["Gear 4th"] = {Cooldown = 25, Duration = 8}
        },
        Range = 12,
        AttackSpeed = 1.0,
        Rarity = "Rare"
    },
    ["Saitama"] = {
        Cost = 800,
        Upgrade_Costs = {1000, 2000, 3500, 5500, 8000},
        Abilities = {
            ["Serious Punch"] = {Cooldown = 18, Damage = 1200},
            ["Consecutive Normal Punches"] = {Cooldown = 22, Duration = 12}
        },
        Range = 18,
        AttackSpeed = 1.1,
        Rarity = "Epic"
    }
}

-- Expanded Map Data
local MapData = {
    ["Leaf Village"] = {
        SpawnPoints = {
            Primary = Vector3.new(0, 0, 0),
            Secondary = {
                Vector3.new(10, 0, 10),
                Vector3.new(-10, 0, -10)
            }
        },
        RecommendedUnits = {"Naruto", "Deku", "Goku"},
        Difficulty = "Normal",
        WaveCount = 25,
        BossWaves = {15, 25},
        Rewards = {
            Coins = 1000,
            Gems = 50,
            Experience = 500
        }
    },
    ["UA High"] = {
        SpawnPoints = {
            Primary = Vector3.new(5, 0, 5),
            Secondary = {
                Vector3.new(15, 0, 15),
                Vector3.new(-5, 0, -5)
            }
        },
        RecommendedUnits = {"Deku", "Ichigo", "Saitama"},
        Difficulty = "Hard",
        WaveCount = 30,
        BossWaves = {20, 30},
        Rewards = {
            Coins = 1500,
            Gems = 75,
            Experience = 750
        }
    }
}

-- Utility Functions
local Utility = {}

function Utility.IsGameLoaded()
    return game:IsLoaded()
end

function Utility.WaitForChild(parent, childName, timeout)
    timeout = timeout or 5
    local startTime = tick()
    local child = parent:FindFirstChild(childName)
    
    while not child do
        if tick() - startTime > timeout then
            return nil
        end
        child = parent:FindFirstChild(childName)
        wait()
    end
    
    return child
end

function Utility.GetDistance(pos1, pos2)
    return (pos1 - pos2).Magnitude
end

function Utility.CreateTween(object, info, properties)
    local tween = TweenService:Create(object, info, properties)
    return tween
end

function Utility.SendWebhook(data)
    if not _G.Settings.WebhookEnabled or _G.Settings.WebhookURL == "" then return end
    
    local success, response = pcall(function()
        local response = syn.request({
            Url = _G.Settings.WebhookURL,
            Method = "POST",
            Headers = {
                ["Content-Type"] = "application/json"
            },
            Body = HttpService:JSONEncode(data)
        })
        return response
    end)
    
    return success and response
end

function Utility.SaveSettings()
    local success, err = pcall(function()
        writefile("ASTD_Settings.json", HttpService:JSONEncode(_G.Settings))
    end)
    return success
end

function Utility.LoadSettings()
    if not isfile("ASTD_Settings.json") then return false end
    
    local success, data = pcall(function()
        return HttpService:JSONDecode(readfile("ASTD_Settings.json"))
    end)
    
    if success then
        for k, v in pairs(data) do
            _G.Settings[k] = v
        end
        return true
    end
    return false
end

-- Game Functions
local Game = {}

function Game.GetCurrentWave()
    local waveNumber = Utility.WaitForChild(workspace, "WaveNumber")
    return waveNumber and waveNumber.Value or 0
end

function Game.GetPlayerMoney()
    local money = LocalPlayer:FindFirstChild("Money")
    return money and money.Value or 0
end

function Game.GetPlayerGems()
    local gems = LocalPlayer:FindFirstChild("Gems")
    return gems and gems.Value or 0
end

function Game.IsInGame()
    return workspace:FindFirstChild("WaveNumber") ~= nil
end

function Game.IsInLobby()
    return not Game.IsInGame()
end

function Game.GetAllUnits()
    local units = {}
    for _, unit in pairs(workspace.Units:GetChildren()) do
        if unit:FindFirstChild("Owner") and unit.Owner.Value == LocalPlayer then
            table.insert(units, unit)
        end
    end
    return units
end

function Game.GetUnitsByType(unitType)
    return table.filter(Game.GetAllUnits(), function(unit)
        return unit.Name == unitType
    end)
end

function Game.PlaceUnit(unitName, position)
    local args = {
        [1] = unitName,
        [2] = position
    }
    ReplicatedStorage.PlaceUnit:FireServer(unpack(args))
end

function Game.UpgradeUnit(unitInstance)
    local args = {
        [1] = unitInstance
    }
    ReplicatedStorage.UpgradeUnit:FireServer(unpack(args))
end

function Game.SellUnit(unitInstance)
    local args = {
        [1] = unitInstance
    }
    ReplicatedStorage.SellUnit:FireServer(unpack(args))
end

function Game.UseAbility(unitInstance)
    local args = {
        [1] = unitInstance
    }
    ReplicatedStorage.UseAbility:FireServer(unpack(args))
end

-- Auto Farm System
local AutoFarm = {}

function AutoFarm.Start()
    if _G.AutoFarmConnection then return end
    
    _G.AutoFarmConnection = RunService.Heartbeat:Connect(function()
        if not _G.Settings.AutoFarm then return end
        
        local currentWave = Game.GetCurrentWave()
        local currentMoney = Game.GetPlayerMoney()
        
        -- Place Units
        if currentMoney >= _G.Settings.MinimumMoneyKeep then
            for unitName, positions in pairs(_G.Settings.UnitPlacementPositions) do
                if UnitData[unitName] and currentMoney >= UnitData[unitName].Cost then
                    for _, pos in ipairs(positions) do
                        Game.PlaceUnit(unitName, pos)
                        wait(_G.Settings.PlacementDelay)
                    end
                end
            end
        end
        
        -- Upgrade Units
        if _G.Settings.AutoUpgrade then
            for _, unit in ipairs(Game.GetAllUnits()) do
                Game.UpgradeUnit(unit)
                wait(_G.Settings.UpgradeDelay)
            end
        end
        
        -- Use Abilities
        if _G.Settings.AutoAbility then
            for _, unit in ipairs(Game.GetAllUnits()) do
                if unit:FindFirstChild("Ability") and unit.Ability.Value then
                    Game.UseAbility(unit)
                    wait(_G.Settings.AbilityDelay)
                end
            end
        end
        
        -- Sell Units
        if _G.Settings.AutoSell then
            for _, wave in ipairs(_G.Settings.SellAtWave) do
                if currentWave == wave then
                    for _, unit in ipairs(Game.GetAllUnits()) do
                        Game.SellUnit(unit)
                        wait(_G.Settings.PlacementDelay)
                    end
                end
            end
        end
    end)
end

function AutoFarm.Stop()
    if _G.AutoFarmConnection then
        _G.AutoFarmConnection:Disconnect()
        _G.AutoFarmConnection = nil
    end
end

-- Auto Raid System
local AutoRaid = {}

function AutoRaid.JoinRaid()
    if not _G.Settings.AutoRaid then return end
    
    local args = {
        [1] = "JoinRaid"
    }
    ReplicatedStorage.RaidSystem:FireServer(unpack(args))
end

function AutoRaid.StartRaid()
    if not _G.Settings.AutoRaid then return end
    
    local args = {
        [1] = "StartRaid"
    }
    ReplicatedStorage.RaidSystem:FireServer(unpack(args))
end

-- Auto Quest System
local QuestSystem = {}

function QuestSystem.GetAvailableQuests()
    local quests = {}
    for _, quest in pairs(workspace.Quests:GetChildren()) do
        if quest:FindFirstChild("Available") and quest.Available.Value then
            table.insert(quests, quest)
        end
    end
    return quests
end

function QuestSystem.ClaimQuest(quest)
    local args = {
        [1] = "ClaimQuest",
        [2] = quest
    }
    ReplicatedStorage.QuestSystem:FireServer(unpack(args))
end

function QuestSystem.AutoClaimQuests()
    if not _G.Settings.AutoClaimQuests then return end
    
    for _, quest in pairs(QuestSystem.GetAvailableQuests()) do
        QuestSystem.ClaimQuest(quest)
        wait(0.5)
    end
end

-- Banner System
local BannerSystem = {}

function BannerSystem.SpinBanner(bannerName)
    local args = {
        [1] = "SpinBanner",
        [2] = bannerName
    }
    ReplicatedStorage.BannerSystem:FireServer(unpack(args))
end

function BannerSystem.AutoSpin()
    if not _G.Settings.AutoSpinBanner then return end
    
    while _G.Settings.AutoSpinBanner and Game.GetPlayerGems() >= 50 do
        BannerSystem.SpinBanner("Current")
        wait(1)
    end
end

-- Unit Management System
local UnitSystem = {}

function UnitSystem.GetUnitPower(unit)
    local power = 0
    if unit:FindFirstChild("Power") then
        power = unit.Power.Value
    end
    return power
end

function UnitSystem.GetInventoryUnits()
    local units = {}
    for _, unit in pairs(LocalPlayer.Inventory:GetChildren()) do
        if unit:IsA("Model") then
            table.insert(units, unit)
        end
    end
    return units
end

function UnitSystem.EquipBestUnits()
    if not _G.Settings.AutoEquipBest then return end
    
    local units = UnitSystem.GetInventoryUnits()
    table.sort(units, function(a, b)
        return UnitSystem.GetUnitPower(a) > UnitSystem.GetUnitPower(b)
    end)
    
    for i = 1, math.min(6, #units) do
        local args = {
            [1] = "EquipUnit",
            [2] = units[i]
        }
        ReplicatedStorage.UnitSystem:FireServer(unpack(args))
        wait(0.5)
    end
end

-- Shop System
local ShopSystem = {}

function ShopSystem.BuyItem(itemName)
    local args = {
        [1] = "BuyItem",
        [2] = itemName
    }
    ReplicatedStorage.ShopSystem:FireServer(unpack(args))
end

function ShopSystem.AutoBuyItems()
    if not _G.Settings.AutoBuyShop then return end
    
    for _, item in pairs(_G.Settings.ShopItems) do
        ShopSystem.BuyItem(item)
        wait(0.5)
    end
end

-- Performance Optimization
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

-- Anti AFK System
local AntiAFK = {}

function AntiAFK.Start()
    if _G.AntiAFKConnection then return end
    
    _G.AntiAFKConnection = game:GetService("Players").LocalPlayer.Idled:Connect(function()
        VirtualUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        wait(1)
        VirtualUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    end)
end

function AntiAFK.Stop()
    if _G.AntiAFKConnection then
        _G.AntiAFKConnection:Disconnect()
        _G.AntiAFKConnection = nil
    end
end

-- UI Library
local Library = loadstring(game:HttpGet("https://raw.githubusercontent.com/xHeptc/Kavo-UI-Library/main/source.lua"))()
local Window = Library.CreateLib("ASTD Advanced Auto Farm", "Ocean")

-- Main Tab
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

MainSection:NewToggle("Auto Upgrade", "Automatically upgrades units", function(state)
    _G.Settings.AutoUpgrade = state
end)

MainSection:NewToggle("Auto Ability", "Automatically uses unit abilities", function(state)
    _G.Settings.AutoAbility = state
end)

MainSection:NewToggle("Auto Sell", "Automatically sells units at specific waves", function(state)
    _G.Settings.AutoSell = state
end)

-- Units Tab
local UnitsTab = Window:NewTab("Units")
local UnitsSection = UnitsTab:NewSection("Unit Management")

UnitsSection:NewToggle("Auto Equip Best", "Automatically equips best units", function(state)
    _G.Settings.AutoEquipBest = state
    if state then
        UnitSystem.EquipBestUnits()
    end
end)

-- For each unit in UnitData
for unitName, data in pairs(UnitData) do
    UnitsSection:NewToggle(unitName, "Select "..unitName, function(state)
        if state then
            table.insert(_G.Settings.SelectedUnits, unitName)
        else
            table.remove(_G.Settings.SelectedUnits, table.find(_G.Settings.SelectedUnits, unitName))
        end
    end)
end

-- Raid Tab
local RaidTab = Window:NewTab("Raid")
local RaidSection = RaidTab:NewSection("Raid Settings")

RaidSection:NewToggle("Auto Raid", "Automatically joins and completes raids", function(state)
    _G.Settings.AutoRaid = state
end)

RaidSection:NewToggle("Auto Accept Carry", "Automatically accepts raid carries", function(state)
    _G.Settings.AutoAcceptCarry = state
end)

-- Banner Tab
local BannerTab = Window:NewTab("Banner")
local BannerSection = BannerTab:NewSection("Banner Settings")

BannerSection:NewToggle("Auto Spin", "Automatically spins banner", function(state)
    _G.Settings.AutoSpinBanner = state
    if state then
        BannerSystem.AutoSpin()
    end
end)

-- Settings Tab
local SettingsTab = Window:NewTab("Settings")
local SettingsSection = SettingsSection:NewSection("General Settings")

SettingsSection:NewToggle("Anti AFK", "Prevents getting kicked for being AFK", function(state)
    _G.Settings.AntiAFK = state
    if state then
        AntiAFK.Start()
    else
        AntiAFK.Stop()
    end
end)

SettingsSection:NewToggle("Reduce Particles", "Reduces particle effects for better performance", function(state)
    _G.Settings.Performance.ReduceParticles = state
    if state then
        Performance.OptimizeGame()
    end
end)

SettingsSection:NewToggle("Low Graphics", "Enables low graphics mode", function(state)
    _G.Settings.Performance.LowGraphics = state
    if state then
        Performance.OptimizeGame()
    end
end)

SettingsSection:NewButton("Save Settings", "Saves current settings", function()
    Utility.SaveSettings()
end)

SettingsSection:NewButton("Load Settings", "Loads saved settings", function()
    Utility.LoadSettings()
end)

-- Initialize
do
    Utility.LoadSettings() -- Load saved settings
    if _G.Settings.AntiAFK then
        AntiAFK.Start()
    end
    if _G.Settings.AutoFarm then
        AutoFarm.Start()
    end
    Performance.OptimizeGame()
end

-- Connections
game.Players.LocalPlayer.Idled:Connect(function()
    if _G.Settings.AntiAFK then
        VirtualUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        wait(1)
        VirtualUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    end
end)

game:GetService("RunService").Heartbeat:Connect(function()
    if _G.Settings.AutoClaimQuests then
        QuestSystem.AutoClaimQuests()
    end
    if _G.Settings.AutoBuyShop then
        ShopSystem.AutoBuyItems()
    end
end)
