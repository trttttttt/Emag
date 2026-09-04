-- ════════════════════════════════════════════════════════
--  MELAYU HUB — single script
--  If game.PlaceId matches, load Combat Hub.
--  Otherwise, load Quest Hub only.
-- ════════════════════════════════════════════════════════
local ALLOWED_PLACE_ID = 105440532661931

local Rayfield    = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
local HttpService = game:GetService("HttpService")

-- ════════════════════════════════════════════════════════
--  CUSTOM CONFIG SAVE/LOAD (manual, not Rayfield's built-in)
-- ════════════════════════════════════════════════════════
local CONFIG_FOLDER = "MelayuHub"
local CONFIG_MODE    = (game.PlaceId == ALLOWED_PLACE_ID) and "combat" or "quest"
local CONFIG_FILE    = CONFIG_FOLDER .. "/config_" .. CONFIG_MODE .. ".json"

if not isfolder(CONFIG_FOLDER) then
    makefolder(CONFIG_FOLDER)
end

local SavedConfig = {}
local ConfigReady = false
local ConfigHydrating = false
local ConfigDirty = false

local NON_PERSISTED_FLAGS = {
    VoteDifficultyDropdown = true,
    VoteLoopToggle = true,
    ReplayVoteToggle = true,
    PortalRemoteDropdown = true,
    PortalLoopToggle = true,
}

local function shouldPersistFlag(flagName)
    return not NON_PERSISTED_FLAGS[flagName]
end

local function cloneConfigValue(value)
    if type(value) ~= "table" then
        return value
    end

    local copy = {}
    for k, v in pairs(value) do
        copy[k] = cloneConfigValue(v)
    end
    return copy
end

local function loadConfigFile()
    local ok, result = pcall(function()
        if isfile(CONFIG_FILE) then
            return HttpService:JSONDecode(readfile(CONFIG_FILE))
        end
        return {}
    end)

    if ok and type(result) == "table" then
        SavedConfig = result
    else
        SavedConfig = {}
    end
end

local function writeConfigFile()
    if not ConfigReady then
        return false
    end

    local snapshot = cloneConfigValue(SavedConfig)
    local ok = pcall(function()
        writefile(CONFIG_FILE, HttpService:JSONEncode(snapshot))
    end)

    if ok then
        ConfigDirty = false
    end
    return ok
end

local function saveConfigValue(flagName, value)
    SavedConfig[flagName] = cloneConfigValue(value)
    ConfigDirty = true
    if ConfigReady then
        writeConfigFile()
    end
end

local function getSavedValue(flagName, default)
    if SavedConfig[flagName] ~= nil then
        return SavedConfig[flagName]
    end
    return default
end

local function getSavedDropdownValue(flagName, default)
    local saved = SavedConfig[flagName]
    if type(saved) == "table" then
        return cloneConfigValue(saved)
    elseif saved ~= nil then
        return { saved }
    end
    return cloneConfigValue(default)
end

local function getSavedToggleValue(flagName, default)
    if type(SavedConfig[flagName]) == "boolean" then
        return SavedConfig[flagName]
    end
    return default
end

local function persistFlag(flagName, value)
    if shouldPersistFlag(flagName) and ConfigReady and not ConfigHydrating then
        saveConfigValue(flagName, value)
    end
end

loadConfigFile()

local Players    = game:GetService("Players")
local RS         = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local LP         = Players.LocalPlayer

local function getChar()
    return LP.Character or LP.CharacterAdded:Wait()
end
local function getHum()
    local c = getChar()
    return c and c:FindFirstChildOfClass("Humanoid")
end
local function getHRP()
    local c = getChar()
    return c and c:FindFirstChild("HumanoidRootPart")
end

-- Shared enemy/NPC movement: CFrame-step teleport only. The character is moved
-- by repeated CFrame updates; within 50 studs it snaps beside the target.
local ENEMY_FLY_SPEED = math.clamp(tonumber(getSavedValue("EnemyFlySpeedSlider", 5)) or 5, 1, 200)
local ENEMY_TELEPORT_RADIUS = 50

-- CFrame-step movement only: no Tween/physics movement. Each iteration updates
-- the character CFrame toward the target. Within 50 studs, snap beside it.
local function moveToTarget(target, stopDistance, speed, teleportRadius, timeout)
    if not target or not target.Parent then return false end

    stopDistance = stopDistance or 3
    speed = math.max(tonumber(speed) or ENEMY_FLY_SPEED, 1)
    teleportRadius = teleportRadius or ENEMY_TELEPORT_RADIUS
    timeout = timeout or 12

    local started = os.clock()
    while target and target.Parent and os.clock() - started < timeout do
        local hrp = getHRP()
        local targetPart = target:FindFirstChild("HumanoidRootPart")
        if not targetPart and target:IsA("Model") then
            targetPart = target.PrimaryPart
        end
        if not targetPart and target:IsA("BasePart") then
            targetPart = target
        end

        if not hrp or not targetPart then
            task.wait(0.03)
        else
            local offset = targetPart.Position - hrp.Position
            local distance = offset.Magnitude

            if distance <= teleportRadius then
                hrp.CFrame = targetPart.CFrame * CFrame.new(0, 0, stopDistance)
                return true
            end

            -- Speed is expressed as studs per second; use a fixed small CFrame
            -- step so the movement remains controllable even at high speeds.
            local dt = 0.05
            local travel = math.min(speed * dt, math.max(distance - stopDistance, 0))
            local direction = distance > 0 and offset.Unit or Vector3.zero
            local nextPos = hrp.Position + direction * travel
            hrp.CFrame = CFrame.new(nextPos, targetPart.Position)
            task.wait(dt)
        end
    end

    return false
end

if game.PlaceId == ALLOWED_PLACE_ID then
    -- ════════════════════════════════════════════════════════
    --  COMBAT HUB (only in the target game)
    -- ════════════════════════════════════════════════════════
    local RunService  = game:GetService("RunService")
    local VirtualUser = game:GetService("VirtualUser")

    local InputEvent = RS.Remotes.Input
    local VoteEvent  = RS.Remotes.Events.DungeonInsideSync

    local Window = Rayfield:CreateWindow({
        Name            = "MELAYU HUB",
        LoadingTitle    = "MELAYU HUB",
        LoadingSubtitle = "Loading...",
        Theme           = "Amethyst",
        DisableRayfieldPrompts = false,
        DisableBuildWarnings    = false,
        ConfigurationSaving = {
            Enabled = false,
            FolderName = "MelayuHub",
            FileName = "CombatHub",
        },
    })

    -- ════════════════════════════════════════════════════════
    --  UNIVERSAL AUTO-SAVE (hooks every Flag automatically)
    -- ════════════════════════════════════════════════════════
    task.spawn(function()
        task.wait(2) -- UI must be fully built before config loading/saving.
        while not ConfigReady do
            task.wait(0.05)
        end

        local lastSnapshot = {}
        -- Seed snapshot from the hydrated UI once, so the backup loop cannot
        -- write transient/default values immediately after load.
        for flagName, element in pairs(Rayfield.Flags) do
            local ok, currentVal = pcall(function()
                if element.CurrentValue ~= nil then return element.CurrentValue end
                if element.CurrentOption ~= nil then return element.CurrentOption end
                if element.CurrentKeybind ~= nil then return element.CurrentKeybind end
                return nil
            end)
            if ok and currentVal ~= nil then
                local eok, encoded = pcall(function() return HttpService:JSONEncode(currentVal) end)
                if eok then lastSnapshot[flagName] = encoded end
            end
        end

        while true do
            task.wait(0.25)

            local changed = false
            for flagName, element in pairs(Rayfield.Flags) do
                local ok, currentVal = pcall(function()
                    if element.CurrentValue ~= nil then
                        return element.CurrentValue
                    elseif element.CurrentOption ~= nil then
                        return element.CurrentOption
                    elseif element.CurrentKeybind ~= nil then
                        return element.CurrentKeybind
                    end
                    return nil
                end)

                if ok and currentVal ~= nil then
                    local encodedOk, encoded = pcall(function()
                        return HttpService:JSONEncode(currentVal)
                    end)

                    if shouldPersistFlag(flagName) and encodedOk and lastSnapshot[flagName] ~= encoded then
                        lastSnapshot[flagName] = encoded
                        SavedConfig[flagName] = cloneConfigValue(currentVal)
                        changed = true
                    end
                end
            end

            if changed then
                writeConfigFile()
            elseif ConfigDirty then
                writeConfigFile()
            end
        end
    end)

    local function loadSavedFlag(flagName)
        local value = SavedConfig[flagName]
        local element = Rayfield.Flags[flagName]
        if value == nil or not element then
            return
        end

        pcall(function()
            if type(value) == "table" and element.MultipleOptions == false then
                element:Set(getSavedDropdownValue(flagName, {}))
            else
                element:Set(cloneConfigValue(value))
            end
        end)
    end

    local function loadSavedFlagsInOrder(order)
        local loaded = {}

        for _, flagName in ipairs(order) do
            if shouldPersistFlag(flagName) and SavedConfig[flagName] ~= nil and Rayfield.Flags[flagName] then
                loadSavedFlag(flagName)
                loaded[flagName] = true
            end
        end

        local remainder = {}
        for flagName, _ in pairs(SavedConfig) do
            if shouldPersistFlag(flagName) and not loaded[flagName] and Rayfield.Flags[flagName] then
                table.insert(remainder, flagName)
            end
        end
        table.sort(remainder)

        for _, flagName in ipairs(remainder) do
            loadSavedFlag(flagName)
        end
    end

    -- ── TAB 1: WEAPON ──
    local WeaponTab = Window:CreateTab("Weapon", "sword")

    local selectedWeapon   = getSavedValue("WeaponDropdown", nil)
    if type(selectedWeapon) == "table" then selectedWeapon = selectedWeapon[1] end
    local autoEquipActive  = getSavedToggleValue("AutoEquipToggle", false)
    local weaponWatchConn  = nil

    local function getBackpackTools()
        local names = {}
        local bp = LP:FindFirstChild("Backpack")
        if bp then
            for _, tool in ipairs(bp:GetChildren()) do
                if tool:IsA("Tool") then table.insert(names, tool.Name) end
            end
        end
        local char = LP.Character
        if char then
            local equipped = char:FindFirstChildOfClass("Tool")
            if equipped then
                local found = false
                for _, n in ipairs(names) do if n == equipped.Name then found = true break end end
                if not found then table.insert(names, equipped.Name) end
            end
        end
        table.sort(names)
        return #names > 0 and names or {"No Weapons Found"}
    end

    local function equipWeapon(name)
        if not name or name == "" or name == "No Weapons Found" then return end
        local hum = getHum()
        local char = getChar()
        if not hum then return end

        local currentTool = char:FindFirstChildOfClass("Tool")
        if currentTool and currentTool.Name == name then return end

        if currentTool then
            hum:UnequipTools()
            task.wait(0.05)
        end

        local bp = LP:FindFirstChild("Backpack")
        local tool = bp and bp:FindFirstChild(name)
        if tool then
            hum:EquipTool(tool)
        end
    end

    local WeaponDropdown
    WeaponDropdown = WeaponTab:CreateDropdown({
        Name = "Select Weapon",
        Options = getBackpackTools(),
        CurrentOption = getSavedDropdownValue("WeaponDropdown", {}),
        MultipleOptions = false,
        Flag = "WeaponDropdown",
        Callback = function(option)
            persistFlag("WeaponDropdown", option)
            local val = type(option) == "table" and option[1] or option
            if val and val ~= "No Weapons Found" then
                selectedWeapon = val
                persistFlag("WeaponDropdown", { val })
                if autoEquipActive then equipWeapon(selectedWeapon) end
            end
        end,
    })

    WeaponTab:CreateButton({
        Name = "Refresh Weapon List",
        Callback = function()
            local fresh = getBackpackTools()
            WeaponDropdown:Refresh(fresh, true)
            Rayfield:Notify({ Title = "Weapon", Content = "Refreshed: " .. #fresh .. " weapons", Duration = 2 })
        end,
    })

    WeaponTab:CreateToggle({
        Name = "Auto Equip & Re-Equip on Death",
        CurrentValue = getSavedToggleValue("AutoEquipToggle", false),
        Flag = "AutoEquipToggle",
        Callback = function(state)
            autoEquipActive = state
            persistFlag("AutoEquipToggle", state)

            if state then
                if selectedWeapon then equipWeapon(selectedWeapon) end

                if weaponWatchConn then weaponWatchConn:Disconnect(); weaponWatchConn = nil end
                weaponWatchConn = LP.CharacterAdded:Connect(function(char)
                    task.wait(1)
                    if autoEquipActive and selectedWeapon then
                        equipWeapon(selectedWeapon)
                    end
                end)

                Rayfield:Notify({ Title = "Weapon", Content = "Auto Equip ON", Duration = 2 })
            else
                if weaponWatchConn then weaponWatchConn:Disconnect(); weaponWatchConn = nil end
                Rayfield:Notify({ Title = "Weapon", Content = "Auto Equip OFF", Duration = 2 })
            end
        end,
    })

    -- ── Auto M1 ──
    -- Uses the exact weapon currently selected in Select Weapon.
    local autoM1Active = getSavedToggleValue("AutoM1Toggle", false)
    local autoM1Thread = nil

    local function fireSelectedWeaponM1()
        if not selectedWeapon then return end
        local char = LP.Character
        if not char then return end
        local tool = char:FindFirstChild(selectedWeapon)
        if not tool then return end

        pcall(function()
            InputEvent:FireServer("Tool", tool, "M1")
        end)
    end

    WeaponTab:CreateToggle({
        Name = "Auto M1",
        CurrentValue = autoM1Active,
        Flag = "AutoM1Toggle",
        Callback = function(state)
            persistFlag("AutoM1Toggle", state)
            autoM1Active = state

            if autoM1Thread then
                task.cancel(autoM1Thread)
                autoM1Thread = nil
            end

            if state then
                autoM1Thread = task.spawn(function()
                    while autoM1Active do
                        fireSelectedWeaponM1()
                        task.wait(0.1)
                    end
                end)
            end
        end,
    })

    -- ── TAB 2: MOB KILL (includes Skills) ──
    local MobTab = Window:CreateTab("Mob Kill", "swords")

    local mobKillActive = getSavedToggleValue("MobKillToggle", false)
    local mobKillThread  = nil
    local currentTarget   = nil

    local function findNearestEnemy()
        local hrp = getHRP()
        if not hrp then return nil end

        local enemiesFolder = workspace:FindFirstChild("Enemies")
        if not enemiesFolder then return nil end

        local closest, closestDist = nil, math.huge
        for _, mob in ipairs(enemiesFolder:GetChildren()) do
            local hum = mob:FindFirstChildOfClass("Humanoid")
            local mobHRP = mob:FindFirstChild("HumanoidRootPart")
            if hum and hum.Health > 0 and mobHRP then
                local dist = (hrp.Position - mobHRP.Position).Magnitude
                if dist < closestDist then
                    closest = mob
                    closestDist = dist
                end
            end
        end
        return closest
    end

    local function isTargetAlive(mob)
        if not mob then return false end
        if not mob.Parent then return false end

        local hum = mob:FindFirstChildOfClass("Humanoid")
        local mobHRP = mob:FindFirstChild("HumanoidRootPart")
        if not hum or not mobHRP then return false end
        if hum.Health <= 0 then return false end

        return true
    end

    local function fireM1()
        if not selectedWeapon then return end
        pcall(function()
            InputEvent:FireServer("Tool", LP.Character[selectedWeapon], "M1")
        end)
    end

    local TargetLabel = MobTab:CreateLabel("Target: None")

    MobTab:CreateToggle({
        Name = "Auto Kill Nearest Enemy",
        CurrentValue = getSavedToggleValue("MobKillToggle", false),
        Flag = "MobKillToggle",
        Callback = function(state)
            mobKillActive = state
            persistFlag("MobKillToggle", state)
            currentTarget = nil

            if state then
                if mobKillThread then task.cancel(mobKillThread); mobKillThread = nil end
                mobKillThread = task.spawn(function()
                    while mobKillActive do
                        if currentTarget == nil or not isTargetAlive(currentTarget) then
                            currentTarget = findNearestEnemy()
                        end

                        if currentTarget then
                            TargetLabel:Set("Target: " .. currentTarget.Name)

                            local hrp = getHRP()
                            local mobHRP = currentTarget:FindFirstChild("HumanoidRootPart")
                            moveToTarget(currentTarget, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.2)
                            fireM1()
                        else
                            TargetLabel:Set("Target: None")
                        end

                        task.wait(0.1)
                    end
                    TargetLabel:Set("Target: None")
                end)
                Rayfield:Notify({ Title = "Mob Kill", Content = "Auto Kill ON", Duration = 2 })
            else
                if mobKillThread then task.cancel(mobKillThread); mobKillThread = nil end
                currentTarget = nil
                TargetLabel:Set("Target: None")
                Rayfield:Notify({ Title = "Mob Kill", Content = "Auto Kill OFF", Duration = 2 })
            end
        end,
    })

    -- ── SKILLS (in Mob Kill tab) ──
    MobTab:CreateLabel("-- Skills --")

    local SKILL_DATA = {
        ["Z"] = Vector3.new(-150.63151550293, 7.4116439819336, 57.293502807617),
        ["X"] = Vector3.new(-150.63151550293, 8.032434463501, 65.726600646973),
        ["C"] = Vector3.new(-153.7455291748, 4.2512588500977, 56.925971984863),
        ["V"] = Vector3.new(-154.19039916992, 4.2512588500977, 63.616569519043),
        ["F"] = Vector3.new(-165.8377532959, 6.6581878662109, 48.863250732422),
    }

    local selectedSkills   = getSavedDropdownValue("SkillMultiDropdown", {})
    local skillCastActive  = getSavedToggleValue("SkillCastToggle", false)
    local skillCastThread  = nil
    local skillMode        = (getSavedDropdownValue("SkillModeDropdown", {"Always Loop"})[1]) or "Always Loop"
    local mobDetectRadius  = tonumber(getSavedValue("SkillRadiusSlider", 30)) or 30

    local function fireSkill(key)
        local pos = SKILL_DATA[key]
        if not pos or not selectedWeapon then return end
        pcall(function()
            InputEvent:FireServer("Tool", LP.Character[selectedWeapon], key, pos)
        end)
    end

    local function isMobNearby(radius)
        local hrp = getHRP()
        if not hrp then return false end
        local enemiesFolder = workspace:FindFirstChild("Enemies")
        if not enemiesFolder then return false end

        for _, mob in ipairs(enemiesFolder:GetChildren()) do
            local hum = mob:FindFirstChildOfClass("Humanoid")
            local mobHRP = mob:FindFirstChild("HumanoidRootPart")
            if hum and hum.Health > 0 and mobHRP then
                local dist = (hrp.Position - mobHRP.Position).Magnitude
                if dist <= radius then return true end
            end
        end
        return false
    end

    MobTab:CreateDropdown({
        Name = "Select Skills to Cast",
        Options = {"Z", "X", "C", "V", "F"},
        CurrentOption = getSavedDropdownValue("SkillMultiDropdown", {}),
        MultipleOptions = true,
        Flag = "SkillMultiDropdown",
        Callback = function(options)
            selectedSkills = {}
            for _, opt in ipairs(options) do table.insert(selectedSkills, opt) end
            persistFlag("SkillMultiDropdown", selectedSkills)
        end,
    })

    MobTab:CreateDropdown({
        Name = "Cast Mode",
        Options = {"Always Loop", "Only Near Mob"},
        CurrentOption = getSavedDropdownValue("SkillModeDropdown", {"Always Loop"}),
        MultipleOptions = false,
        Flag = "SkillModeDropdown",
        Callback = function(option)
            skillMode = type(option) == "table" and option[1] or option
            persistFlag("SkillModeDropdown", { skillMode })
        end,
    })

    MobTab:CreateSlider({
        Name = "Mob Detect Radius (Only Near Mob mode)",
        Range = {5, 150},
        Increment = 5,
        CurrentValue = getSavedValue("SkillRadiusSlider", 30),
        Flag = "SkillRadiusSlider",
        Callback = function(v)
            mobDetectRadius = v
            persistFlag("SkillRadiusSlider", v)
        end,
    })

    MobTab:CreateToggle({
        Name = "Auto Cast Selected Skills",
        CurrentValue = getSavedToggleValue("SkillCastToggle", false),
        Flag = "SkillCastToggle",
        Callback = function(state)
            skillCastActive = state
            persistFlag("SkillCastToggle", state)

            if state then
                if skillCastThread then task.cancel(skillCastThread); skillCastThread = nil end
                skillCastThread = task.spawn(function()
                    while skillCastActive do
                        if #selectedSkills > 0 then
                            local canCast = true
                            if skillMode == "Only Near Mob" then
                                canCast = isMobNearby(mobDetectRadius)
                            end

                            if canCast then
                                for _, key in ipairs(selectedSkills) do
                                    if not skillCastActive then break end
                                    fireSkill(key)
                                    task.wait(0.15)
                                end
                            end
                        end
                        task.wait(0.2)
                    end
                end)
                Rayfield:Notify({ Title = "Skills", Content = "Auto Cast ON", Duration = 2 })
            else
                if skillCastThread then task.cancel(skillCastThread); skillCastThread = nil end
                Rayfield:Notify({ Title = "Skills", Content = "Auto Cast OFF", Duration = 2 })
            end
        end,
    })

    -- ── TAB 3: DUNGEON ──
    local DungeonTab = Window:CreateTab("Dungeon", "castle")

    local selectedDifficulty = getSavedValue("VoteDifficultyDropdown", "Easy")
    if type(selectedDifficulty) == "table" then selectedDifficulty = selectedDifficulty[1] end
    local voteActive = getSavedToggleValue("VoteLoopToggle", false)
    local voteThread  = nil

    DungeonTab:CreateDropdown({
        Name = "Select Difficulty",
        Options = {"Easy", "Medium", "Hard", "Extreme"},
        CurrentOption = getSavedDropdownValue("VoteDifficultyDropdown", {"Easy"}),
        MultipleOptions = false,
        Flag = "VoteDifficultyDropdown",
        Callback = function(option)
            selectedDifficulty = type(option) == "table" and option[1] or option
            persistFlag("VoteDifficultyDropdown", { selectedDifficulty })
        end,
    })

    DungeonTab:CreateToggle({
        Name = "Auto Vote Loop",
        CurrentValue = getSavedToggleValue("VoteLoopToggle", false),
        Flag = "VoteLoopToggle",
        Callback = function(state)
            voteActive = state
            persistFlag("VoteLoopToggle", state)

            if state then
                if voteThread then task.cancel(voteThread); voteThread = nil end
                voteThread = task.spawn(function()
                    while voteActive do
                        pcall(function()
                            VoteEvent:FireServer("Vote", selectedDifficulty)
                        end)
                        task.wait(1)
                    end
                end)
                Rayfield:Notify({ Title = "Dungeon", Content = "Auto Vote: " .. selectedDifficulty, Duration = 2 })
            else
                if voteThread then task.cancel(voteThread); voteThread = nil end
                Rayfield:Notify({ Title = "Dungeon", Content = "Auto Vote OFF", Duration = 2 })
            end
        end,
    })

    DungeonTab:CreateToggle({
        Name = "Auto Replay Vote",
        CurrentValue = getSavedToggleValue("ReplayVoteToggle", false),
        Flag = "ReplayVoteToggle",
        Callback = function(state)
            _G.ReplayVoteActive = state
            persistFlag("ReplayVoteToggle", state)

            if state then
                if _G.ReplayVoteThread then task.cancel(_G.ReplayVoteThread) end
                _G.ReplayVoteThread = task.spawn(function()
                    while _G.ReplayVoteActive do
                        pcall(function()
                            VoteEvent:FireServer("ReplayVote")
                        end)
                        task.wait(1)
                    end
                end)
                Rayfield:Notify({ Title = "Dungeon", Content = "Auto Replay Vote ON", Duration = 2 })
            else
                if _G.ReplayVoteThread then task.cancel(_G.ReplayVoteThread); _G.ReplayVoteThread = nil end
                Rayfield:Notify({ Title = "Dungeon", Content = "Auto Replay Vote OFF", Duration = 2 })
            end
        end,
    })

    -- ── Auto Portal Loop (Create + Start every N seconds) ──
    DungeonTab:CreateSection("Portal Loop")

    local PORTAL_REMOTES = {
        "CursedChildPortal", "RealmBeyondHeavenPortal",
        "HouseOfSpidersPortal", "TimeSafePortal", "SpiderEstatePortal",
    }

    local selectedPortalRemote = getSavedValue("PortalRemoteDropdown", PORTAL_REMOTES[1])
    if type(selectedPortalRemote) == "table" then selectedPortalRemote = selectedPortalRemote[1] end
    local portalLoopActive     = getSavedToggleValue("PortalLoopToggle", false)
    local portalLoopThread     = nil
    _G.ReplayVoteActive = getSavedToggleValue("ReplayVoteToggle", false)

    DungeonTab:CreateDropdown({
        Name = "Select Portal",
        Options = PORTAL_REMOTES,
        CurrentOption = getSavedDropdownValue("PortalRemoteDropdown", { PORTAL_REMOTES[1] }),
        MultipleOptions = false,
        Flag = "PortalRemoteDropdown",
        Callback = function(option)
            selectedPortalRemote = type(option) == "table" and option[1] or option
            persistFlag("PortalRemoteDropdown", { selectedPortalRemote })
        end,
    })

    DungeonTab:CreateToggle({
        Name = "Auto Portal Loop",
        CurrentValue = getSavedToggleValue("PortalLoopToggle", false),
        Flag = "PortalLoopToggle",
        Callback = function(state)
            portalLoopActive = state
            persistFlag("PortalLoopToggle", state)

            if state then
                if portalLoopThread then task.cancel(portalLoopThread); portalLoopThread = nil end
                portalLoopThread = task.spawn(function()
                    local portalEvent = RS.Remotes.Events[selectedPortalRemote]
                    pcall(function()
                        portalEvent:FireServer("Create")
                    end)
                    task.wait(1)
                    while portalLoopActive do
                        pcall(function()
                            portalEvent:FireServer("Start")
                        end)
                        task.wait(0.5)
                    end
                end)
                Rayfield:Notify({ Title = "Dungeon", Content = "Portal Loop ON: " .. selectedPortalRemote, Duration = 2 })
            else
                if portalLoopThread then task.cancel(portalLoopThread); portalLoopThread = nil end
                Rayfield:Notify({ Title = "Dungeon", Content = "Portal Loop OFF", Duration = 2 })
            end
        end,
    })

    -- ── TAB 4: SETTINGS ──
    local SettingsTab = Window:CreateTab("Settings", "settings")

    local antiAfkActive = getSavedToggleValue("AntiAfkToggle", true)

    LP.Idled:Connect(function()
        if not antiAfkActive then return end
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new())
    end)

    SettingsTab:CreateToggle({
        Name = "Anti AFK",
        CurrentValue = getSavedToggleValue("AntiAfkToggle", true),
        Flag = "AntiAfkToggle",
        Callback = function(state)
            antiAfkActive = state
            persistFlag("AntiAfkToggle", state)
            Rayfield:Notify({ Title = "Settings", Content = state and "Anti AFK ON" or "Anti AFK OFF", Duration = 2 })
        end,
    })

    SettingsTab:CreateSection("Movement")

    -- Combat Hub CFrame movement setting is persisted by the custom config.
    SettingsTab:CreateSlider({
        Name = "Enemy CFrame Teleport Speed",
        Range = {1, 200},
        Increment = 1,
        CurrentValue = math.clamp(tonumber(getSavedValue("EnemyFlySpeedSlider", 5)) or 5, 1, 200),
        Flag = "EnemyFlySpeedSlider",
        Callback = function(value)
            ENEMY_FLY_SPEED = math.clamp(tonumber(value) or 5, 1, 200)
            persistFlag("EnemyFlySpeedSlider", ENEMY_FLY_SPEED)
        end,
    })

    -- ── Load all saved values now that every Flag exists ──
    task.spawn(function()
        task.wait(1)

        ConfigHydrating = true
        loadSavedFlagsInOrder({
            "WeaponDropdown",
            "AutoEquipToggle",
            "AutoM1Toggle",
            "MobKillToggle",
            "SkillMultiDropdown",
            "SkillModeDropdown",
            "SkillRadiusSlider",
            "SkillCastToggle",
            "VoteDifficultyDropdown",
            "VoteLoopToggle",
            "ReplayVoteToggle",
            "PortalRemoteDropdown",
            "PortalLoopToggle",
            "AntiAfkToggle",
            "EnemyFlySpeedSlider",
        })

        ConfigHydrating = false
        ConfigReady = true
        ConfigDirty = true
        writeConfigFile()

        -- Explicitly start saved combat features after config hydration.
        -- Restoring the UI value alone does not reliably execute Rayfield callbacks.
        if autoEquipActive then
            if selectedWeapon then equipWeapon(selectedWeapon) end
            if weaponWatchConn then weaponWatchConn:Disconnect(); weaponWatchConn = nil end
            weaponWatchConn = LP.CharacterAdded:Connect(function(char)
                task.wait(1)
                if autoEquipActive and selectedWeapon then equipWeapon(selectedWeapon) end
            end)
        end

        if autoM1Active then
            if autoM1Thread then task.cancel(autoM1Thread); autoM1Thread = nil end
            autoM1Thread = task.spawn(function()
                while autoM1Active do
                    fireSelectedWeaponM1()
                    task.wait(0.1)
                end
            end)
        end

        if mobKillActive then
            if mobKillThread then task.cancel(mobKillThread); mobKillThread = nil end
            mobKillThread = task.spawn(function()
                while mobKillActive do
                    if currentTarget == nil or not isTargetAlive(currentTarget) then
                        currentTarget = findNearestEnemy()
                    end
                    if currentTarget then
                        TargetLabel:Set("Target: " .. currentTarget.Name)
                        moveToTarget(currentTarget, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.2)
                        fireM1()
                    else
                        TargetLabel:Set("Target: None")
                    end
                    task.wait(0.1)
                end
                TargetLabel:Set("Target: None")
            end)
        end

        if skillCastActive then
            if skillCastThread then task.cancel(skillCastThread); skillCastThread = nil end
            skillCastThread = task.spawn(function()
                while skillCastActive do
                    if #selectedSkills > 0 then
                        local canCast = skillMode ~= "Only Near Mob" or isMobNearby(mobDetectRadius)
                        if canCast then
                            for _, key in ipairs(selectedSkills) do
                                if not skillCastActive then break end
                                fireSkill(key)
                                task.wait(0.15)
                            end
                        end
                    end
                    task.wait(0.2)
                end
            end)
        end

        if voteActive then
            if voteThread then task.cancel(voteThread); voteThread = nil end
            voteThread = task.spawn(function()
                while voteActive do
                    pcall(function() VoteEvent:FireServer("Vote", selectedDifficulty) end)
                    task.wait(1)
                end
            end)
        end

        if _G.ReplayVoteActive then
            if _G.ReplayVoteThread then task.cancel(_G.ReplayVoteThread) end
            _G.ReplayVoteThread = task.spawn(function()
                while _G.ReplayVoteActive do
                    pcall(function() VoteEvent:FireServer("ReplayVote") end)
                    task.wait(1)
                end
            end)
        end

        if portalLoopActive then
            if portalLoopThread then task.cancel(portalLoopThread); portalLoopThread = nil end
            portalLoopThread = task.spawn(function()
                local portalEvent = RS.Remotes.Events[selectedPortalRemote]
                pcall(function() portalEvent:FireServer("Create") end)
                task.wait(1)
                while portalLoopActive do
                    pcall(function() portalEvent:FireServer("Start") end)
                    task.wait(0.5)
                end
            end)
        end
    end)

else
    -- ════════════════════════════════════════════════════════
    --  QUEST HUB (any other game)
    -- ════════════════════════════════════════════════════════
    local InputEvent = nil
    pcall(function()
        InputEvent = RS.Remotes.Input
    end)

    local function fireM1()
        if not selectedWeapon then return end
        pcall(function()
            InputEvent:FireServer("Tool", LP.Character[selectedWeapon], "M1")
        end)
    end

    local Window = Rayfield:CreateWindow({
        Name            = "MELAYU HUB - Quest",
        LoadingTitle    = "MELAYU HUB",
        LoadingSubtitle = "Loading...",
        Theme           = "Amethyst",
        DisableRayfieldPrompts = false,
        DisableBuildWarnings    = false,
        ConfigurationSaving = {
            Enabled = false,
            FolderName = "MelayuHub",
            FileName = "QuestHub",
        },
    })

    -- ════════════════════════════════════════════════════════
    --  UNIVERSAL AUTO-SAVE (hooks every Flag automatically)
    -- ════════════════════════════════════════════════════════
    task.spawn(function()
        task.wait(2) -- UI must be fully built before config loading/saving.
        while not ConfigReady do
            task.wait(0.05)
        end

        local lastSnapshot = {}
        -- Seed snapshot from the hydrated UI once, so the backup loop cannot
        -- write transient/default values immediately after load.
        for flagName, element in pairs(Rayfield.Flags) do
            local ok, currentVal = pcall(function()
                if element.CurrentValue ~= nil then return element.CurrentValue end
                if element.CurrentOption ~= nil then return element.CurrentOption end
                if element.CurrentKeybind ~= nil then return element.CurrentKeybind end
                return nil
            end)
            if ok and currentVal ~= nil then
                local eok, encoded = pcall(function() return HttpService:JSONEncode(currentVal) end)
                if eok then lastSnapshot[flagName] = encoded end
            end
        end

        while true do
            task.wait(0.25)

            local changed = false
            for flagName, element in pairs(Rayfield.Flags) do
                local ok, currentVal = pcall(function()
                    if element.CurrentValue ~= nil then
                        return element.CurrentValue
                    elseif element.CurrentOption ~= nil then
                        return element.CurrentOption
                    elseif element.CurrentKeybind ~= nil then
                        return element.CurrentKeybind
                    end
                    return nil
                end)

                if ok and currentVal ~= nil then
                    local encodedOk, encoded = pcall(function()
                        return HttpService:JSONEncode(currentVal)
                    end)

                    if shouldPersistFlag(flagName) and encodedOk and lastSnapshot[flagName] ~= encoded then
                        lastSnapshot[flagName] = encoded
                        SavedConfig[flagName] = cloneConfigValue(currentVal)
                        changed = true
                    end
                end
            end

            if changed then
                writeConfigFile()
            elseif ConfigDirty then
                writeConfigFile()
            end
        end
    end)

    local function loadSavedFlag(flagName)
        local value = SavedConfig[flagName]
        local element = Rayfield.Flags[flagName]
        if value == nil or not element then
            return
        end

        pcall(function()
            if type(value) == "table" and element.MultipleOptions == false then
                element:Set(getSavedDropdownValue(flagName, {}))
            else
                element:Set(cloneConfigValue(value))
            end
        end)
    end

    local function loadSavedFlagsInOrder(order)
        local loaded = {}

        for _, flagName in ipairs(order) do
            if shouldPersistFlag(flagName) and SavedConfig[flagName] ~= nil and Rayfield.Flags[flagName] then
                loadSavedFlag(flagName)
                loaded[flagName] = true
            end
        end

        local remainder = {}
        for flagName, _ in pairs(SavedConfig) do
            if shouldPersistFlag(flagName) and not loaded[flagName] and Rayfield.Flags[flagName] then
                table.insert(remainder, flagName)
            end
        end
        table.sort(remainder)

        for _, flagName in ipairs(remainder) do
            loadSavedFlag(flagName)
        end
    end

    -- ── TAB 1: WEAPON (+ Skills) ──
    local WeaponTab = Window:CreateTab("Weapon", "sword")

    local selectedWeapon  = getSavedValue("WeaponDropdown", nil)
    if type(selectedWeapon) == "table" then selectedWeapon = selectedWeapon[1] end
    local autoEquipActive = getSavedToggleValue("AutoEquipToggle", false)
    local weaponWatchConn = nil

    local function getBackpackTools()
        local names = {}
        local bp = LP:FindFirstChild("Backpack")
        if bp then
            for _, tool in ipairs(bp:GetChildren()) do
                if tool:IsA("Tool") then table.insert(names, tool.Name) end
            end
        end
        local char = LP.Character
        if char then
            local equipped = char:FindFirstChildOfClass("Tool")
            if equipped then
                local found = false
                for _, n in ipairs(names) do if n == equipped.Name then found = true break end end
                if not found then table.insert(names, equipped.Name) end
            end
        end
        table.sort(names)
        return #names > 0 and names or {"No Weapons Found"}
    end

    local function equipWeapon(name)
        if not name or name == "" or name == "No Weapons Found" then return end
        local hum = getHum()
        local char = getChar()
        if not hum then return end

        local currentTool = char:FindFirstChildOfClass("Tool")
        if currentTool and currentTool.Name == name then return end

        if currentTool then
            hum:UnequipTools()
            task.wait(0.05)
        end

        local bp = LP:FindFirstChild("Backpack")
        local tool = bp and bp:FindFirstChild(name)
        if tool then
            hum:EquipTool(tool)
        end
    end

    local WeaponDropdown
    WeaponDropdown = WeaponTab:CreateDropdown({
        Name = "Select Weapon",
        Options = getBackpackTools(),
        CurrentOption = getSavedDropdownValue("WeaponDropdown", {}),
        MultipleOptions = false,
        Flag = "WeaponDropdown",
        Callback = function(option)
            persistFlag("WeaponDropdown", option)
            local val = type(option) == "table" and option[1] or option
            if val and val ~= "No Weapons Found" then
                selectedWeapon = val
                if autoEquipActive then equipWeapon(selectedWeapon) end
            end
        end,
    })

    WeaponTab:CreateButton({
        Name = "Refresh Weapon List",
        Callback = function()
            local fresh = getBackpackTools()
            WeaponDropdown:Refresh(fresh, true)
            Rayfield:Notify({ Title = "Weapon", Content = "Refreshed: " .. #fresh .. " weapons", Duration = 2 })
        end,
    })

    WeaponTab:CreateToggle({
        Name = "Auto Equip & Re-Equip on Death",
        CurrentValue = getSavedToggleValue("AutoEquipToggle", false),
        Flag = "AutoEquipToggle",
        Callback = function(state)
            autoEquipActive = state
            persistFlag("AutoEquipToggle", state)

            if state then
                if selectedWeapon then equipWeapon(selectedWeapon) end

                if weaponWatchConn then weaponWatchConn:Disconnect(); weaponWatchConn = nil end
                weaponWatchConn = LP.CharacterAdded:Connect(function(char)
                    task.wait(1)
                    if autoEquipActive and selectedWeapon then
                        equipWeapon(selectedWeapon)
                    end
                end)

                Rayfield:Notify({ Title = "Weapon", Content = "Auto Equip ON", Duration = 2 })
            else
                if weaponWatchConn then weaponWatchConn:Disconnect(); weaponWatchConn = nil end
                Rayfield:Notify({ Title = "Weapon", Content = "Auto Equip OFF", Duration = 2 })
            end
        end,
    })

    -- ── Auto M1 ──
    -- Uses the exact weapon currently selected in Select Weapon.
    local autoM1Active = getSavedToggleValue("AutoM1Toggle", false)
    local autoM1Thread = nil

    local function fireSelectedWeaponM1()
        if not selectedWeapon then return end
        local char = LP.Character
        if not char then return end
        local tool = char:FindFirstChild(selectedWeapon)
        if not tool then return end

        pcall(function()
            InputEvent:FireServer("Tool", tool, "M1")
        end)
    end

    WeaponTab:CreateToggle({
        Name = "Auto M1",
        CurrentValue = autoM1Active,
        Flag = "AutoM1Toggle",
        Callback = function(state)
            persistFlag("AutoM1Toggle", state)
            autoM1Active = state

            if autoM1Thread then
                task.cancel(autoM1Thread)
                autoM1Thread = nil
            end

            if state then
                autoM1Thread = task.spawn(function()
                    while autoM1Active do
                        fireSelectedWeaponM1()
                        task.wait(0.1)
                    end
                end)
            end
        end,
    })

    -- ── Skills (in Weapon tab) ──
    WeaponTab:CreateLabel("-- Skills --")

    local SKILL_DATA = {
        ["Z"] = Vector3.new(-150.63151550293, 7.4116439819336, 57.293502807617),
        ["X"] = Vector3.new(-150.63151550293, 8.032434463501, 65.726600646973),
        ["C"] = Vector3.new(-153.7455291748, 4.2512588500977, 56.925971984863),
        ["V"] = Vector3.new(-154.19039916992, 4.2512588500977, 63.616569519043),
        ["F"] = Vector3.new(-165.8377532959, 6.6581878662109, 48.863250732422),
    }

    local selectedSkills  = getSavedDropdownValue("SkillMultiDropdown", {})
    local skillCastActive = getSavedToggleValue("SkillCastToggle", false)
    local skillCastThread = nil
    local skillMode       = (getSavedDropdownValue("SkillModeDropdown", {"Always Loop"})[1]) or "Always Loop"
    local mobDetectRadius = tonumber(getSavedValue("SkillRadiusSlider", 30)) or 30

    local function fireSkill(key)
        local pos = SKILL_DATA[key]
        if not pos or not selectedWeapon then return end
        pcall(function()
            InputEvent:FireServer("Tool", LP.Character[selectedWeapon], key, pos)
        end)
    end

    local function isMobNearby(radius)
        local hrp = getHRP()
        if not hrp then return false end
        local enemiesFolder = workspace:FindFirstChild("Enemies")
        if not enemiesFolder then return false end

        for _, mob in ipairs(enemiesFolder:GetChildren()) do
            local hum = mob:FindFirstChildOfClass("Humanoid")
            local mobHRP = mob:FindFirstChild("HumanoidRootPart")
            if hum and hum.Health > 0 and mobHRP then
                local dist = (hrp.Position - mobHRP.Position).Magnitude
                if dist <= radius then return true end
            end
        end
        return false
    end

    WeaponTab:CreateDropdown({
        Name = "Select Skills to Cast",
        Options = {"Z", "X", "C", "V", "F"},
        CurrentOption = getSavedDropdownValue("SkillMultiDropdown", {}),
        MultipleOptions = true,
        Flag = "SkillMultiDropdown",
        Callback = function(options)
            selectedSkills = {}
            for _, opt in ipairs(options) do table.insert(selectedSkills, opt) end
            persistFlag("SkillMultiDropdown", selectedSkills)
        end,
    })

    WeaponTab:CreateDropdown({
        Name = "Cast Mode",
        Options = {"Always Loop", "Only Near Mob"},
        CurrentOption = getSavedDropdownValue("SkillModeDropdown", {"Always Loop"}),
        MultipleOptions = false,
        Flag = "SkillModeDropdown",
        Callback = function(option)
            skillMode = type(option) == "table" and option[1] or option
            persistFlag("SkillModeDropdown", { skillMode })
        end,
    })

    WeaponTab:CreateSlider({
        Name = "Mob Detect Radius (Only Near Mob mode)",
        Range = {5, 150},
        Increment = 5,
        CurrentValue = getSavedValue("SkillRadiusSlider", 30),
        Flag = "SkillRadiusSlider",
        Callback = function(v)
            mobDetectRadius = v
            persistFlag("SkillRadiusSlider", v)
        end,
    })

    WeaponTab:CreateToggle({
        Name = "Auto Cast Selected Skills",
        CurrentValue = getSavedToggleValue("SkillCastToggle", false),
        Flag = "SkillCastToggle",
        Callback = function(state)
            skillCastActive = state
            persistFlag("SkillCastToggle", state)

            if state then
                if skillCastThread then task.cancel(skillCastThread); skillCastThread = nil end
                skillCastThread = task.spawn(function()
                    while skillCastActive do
                        if #selectedSkills > 0 then
                            local canCast = true
                            if skillMode == "Only Near Mob" then
                                canCast = isMobNearby(mobDetectRadius)
                            end

                            if canCast then
                                for _, key in ipairs(selectedSkills) do
                                    if not skillCastActive then break end
                                    fireSkill(key)
                                    task.wait(0.15)
                                end
                            end
                        end
                        task.wait(0.2)
                    end
                end)
                Rayfield:Notify({ Title = "Skills", Content = "Auto Cast ON", Duration = 2 })
            else
                if skillCastThread then task.cancel(skillCastThread); skillCastThread = nil end
                Rayfield:Notify({ Title = "Skills", Content = "Auto Cast OFF", Duration = 2 })
            end
        end,
    })

    -- ── TAB 2: QUEST ──
    local QuestTab = Window:CreateTab("Quest", "scroll-text")

    -- ── Redeem Codes ──
    -- Uses the game's actual Functions.Input remote:
    -- InvokeServer("Code", <code>)
    local KNOWN_CODES = {
        "sorryforbugs9!!", "sorryforbugs10!!", "sorryforbugs13!!", "minislopupdate!!",
        "update1.9!!", "sorryfordelay6!!", "sorryforbugs14!!", "sorryforbugs15!!",
        "sorryforbugs16!!", "sorryfordelayyy!!", "update1.1!!", "update1.5!!",
        "update1.5part2!!", "update1.75!!", "sorryfordelay5!!", "18klikes!!",
        "19klikes!!", "dontknowwhattotypeforthiscode!!", "sorryfordelay4!!",
        "sorryforbugs12!!", "sorryfordroppotionbug!!", "balancechange1!!",
        "oceanupdate!!", "thanksfor17klikes!!", "4mvisits!!", "sorryfordelay3!!",
        "sorryforbugs11!!", "bugfixforupdate1.1!!", "sorryfordelayyy2!!",
        "thanksfor4kccu!!", "fireupdate!!",
    }

    local function redeemCode(code)
        local ok, result = pcall(function()
            return RS.Remotes.Functions.Input:InvokeServer("Code", code)
        end)
        return ok, result
    end

    QuestTab:CreateSection("Redeem Codes")

    QuestTab:CreateButton({
        Name = "Redeem All Known Codes",
        Description = "Redeems every code in the list using the Code remote",
        Callback = function()
            local successCount = 0
            for _, code in ipairs(KNOWN_CODES) do
                local ok = redeemCode(code)
                if ok then
                    successCount = successCount + 1
                end
                task.wait(0.2)
            end

            Rayfield:Notify({
                Title = "Codes",
                Content = "Sent " .. successCount .. "/" .. #KNOWN_CODES .. " codes.",
                Duration = 4,
            })
        end,
    })

    QuestTab:CreateDropdown({
        Name = "Select Code",
        Options = KNOWN_CODES,
        CurrentOption = {KNOWN_CODES[1]},
        MultipleOptions = false,
        Callback = function(option)
            local code = type(option) == "table" and option[1] or option
            if code then
                redeemCode(code)
            end
        end,
    })

    local AcceptEvent = nil
    pcall(function()
        AcceptEvent = RS.Remotes.Functions.Input
    end)

    local QuestData = nil
    local ok, result = pcall(function()
        return require(RS.Modules.Configurations.QuestData)
    end)
    if ok then QuestData = result end

    -- Generic / non-farmable Kill targets to exclude
    local EXCLUDED_TARGETS = {
        ["players"]        = true,
        ["bosses"]         = true,
        ["special bosses"] = true,
        ["world bosses"]   = true,
    }

    -- Every island PortalId from IslandsConfig, used as fallback
    -- teleport cycle when the quest's target enemy can't be found.
    local PORTAL_IDS = {
        "Legacy", "Starter", "JungleIsland", "IceIsland", "JujutsuAcademy",
        "HollowLand", "SlayerMansion", "TokyoGhoul", "RuinCity",
        "7thCompanyIsland", "FishmanIsland", "GraveIsland", "ACity", "UndyneTrial",
    }

    local function extractKillTargets(questTable, list)
        for identity, quest in pairs(questTable) do
            if type(quest) == "table" then
                if quest.Goal and quest.Goal.Type == "Kill" and quest.Goal.Target then
                    local tLower = quest.Goal.Target:lower()
                    if not EXCLUDED_TARGETS[tLower] then
                        table.insert(list, {
                            identity = identity,
                            label    = (quest.Name or identity) .. "  ->  " .. quest.Goal.Target,
                            target   = quest.Goal.Target,
                        })
                    end
                end
                if quest.Objectives then
                    for _, obj in ipairs(quest.Objectives) do
                        if obj.Type == "Kill" and obj.Target then
                            local tLower2 = obj.Target:lower()
                            if not EXCLUDED_TARGETS[tLower2] then
                                table.insert(list, {
                                    identity = identity,
                                    label    = (quest.Name or identity) .. "  ->  " .. obj.Target,
                                    target   = obj.Target,
                                })
                            end
                        end
                    end
                end
            end
        end
    end

    -- Only keep quests that actually require accepting via an NPC
    -- (i.e. workspace.NPCs has a child with the same name as the
    -- quest's identity key — that's how accept works in this game).
    local function requiresAccept(identity)
        local npcFolder = workspace:FindFirstChild("NPCs")
        if not npcFolder then return false end
        return npcFolder:FindFirstChild(identity) ~= nil
    end

    local killQuestList = {}
    local labelToTarget   = {}
    local labelToIdentity = {}

    local function buildQuestList()
        killQuestList = {}
        labelToTarget = {}
        labelToIdentity = {}

        if not QuestData then
            return {"QuestData failed to load"}
        end

        local rawList = {}
        for _, category in ipairs({QuestData.Main, QuestData.Daily, QuestData.Weekly, QuestData.Monthly}) do
            if category then
                extractKillTargets(category, rawList)
            end
        end

        -- Filter: keep only quests that require accepting via NPC
        for _, entry in ipairs(rawList) do
            if requiresAccept(entry.identity) then
                table.insert(killQuestList, entry)
            end
        end

        table.sort(killQuestList, function(a, b) return a.label < b.label end)

        local labels = {}
        for _, entry in ipairs(killQuestList) do
            table.insert(labels, entry.label)
            labelToTarget[entry.label]   = entry.target
            labelToIdentity[entry.label] = entry.identity
        end

        return #labels > 0 and labels or {"No Acceptable Kill Quests Found"}
    end

    local selectedQuestTarget   = nil
    local selectedQuestIdentity = nil
    local savedQuestLabel = getSavedValue("QuestDropdown", nil)
    if type(savedQuestLabel) == "table" then savedQuestLabel = savedQuestLabel[1] end
    if savedQuestLabel and labelToTarget[savedQuestLabel] then
        selectedQuestTarget = labelToTarget[savedQuestLabel]
        selectedQuestIdentity = labelToIdentity[savedQuestLabel]
    end
    local mobKillActive = getSavedToggleValue("QuestKillToggle", false)
    local mobKillThread  = nil
    local currentTarget   = nil

    local QuestDropdown
    QuestDropdown = QuestTab:CreateDropdown({
        Name = "Select Quest (Kill type)",
        Options = buildQuestList(),
        CurrentOption = getSavedDropdownValue("QuestDropdown", {}),
        MultipleOptions = false,
        Flag = "QuestDropdown",
        Callback = function(option)
            local val = type(option) == "table" and option[1] or option
            if val and labelToTarget[val] then
                selectedQuestTarget   = labelToTarget[val]
                selectedQuestIdentity = labelToIdentity[val]
                currentTarget = nil
                persistFlag("QuestDropdown", { val })
            end
        end,
    })

    -- Resolve the saved quest AFTER buildQuestList() populated labelToTarget.
    do
        local saved = getSavedValue("QuestDropdown", nil)
        if type(saved) == "table" then saved = saved[1] end
        if saved and labelToTarget[saved] then
            selectedQuestTarget = labelToTarget[saved]
            selectedQuestIdentity = labelToIdentity[saved]
        end
    end

    QuestTab:CreateButton({
        Name = "Refresh Quest List",
        Callback = function()
            local ok2, result2 = pcall(function()
                return require(RS.Modules.Configurations.QuestData)
            end)
            if ok2 then QuestData = result2 end

            local fresh = buildQuestList()
            QuestDropdown:Refresh(fresh, true)
            Rayfield:Notify({ Title = "Quest", Content = "Refreshed: " .. #killQuestList .. " quests", Duration = 2 })
        end,
    })

    -- ── Accept Quest (reusable function + button) ──
    local function acceptSelectedQuest()
        if not selectedQuestIdentity then
            Rayfield:Notify({ Title = "Quest", Content = "Select a quest first!", Duration = 3 })
            return false
        end

        local npcFolder = workspace:FindFirstChild("NPCs")
        local npc = npcFolder and npcFolder:FindFirstChild(selectedQuestIdentity)

        if not npc then
            Rayfield:Notify({ Title = "Quest", Content = "NPC not found for this quest.", Duration = 3 })
            return false
        end

        local success = pcall(function()
            AcceptEvent:InvokeServer("Quest", "Accept", npc, nil)
        end)

        if success then
            Rayfield:Notify({ Title = "Quest", Content = "Accepted: " .. selectedQuestIdentity, Duration = 2 })
        else
            Rayfield:Notify({ Title = "Quest", Content = "Failed to accept quest.", Duration = 3 })
        end
        return success
    end

    QuestTab:CreateButton({
        Name        = "Accept Selected Quest",
        Description = "Fires once — accepts the quest from its NPC",
        Callback = acceptSelectedQuest,
    })

    local TargetLabel = QuestTab:CreateLabel("Current Target: None")

    local function findQuestEnemy(targetName)
        if not targetName then return nil end
        local hrp = getHRP()
        if not hrp then return nil end

        local enemiesFolder = workspace:FindFirstChild("Enemies")
        if not enemiesFolder then
            warn("[QuestDebug] workspace.Enemies folder not found!")
            return nil
        end

        -- IMPORTANT: Quest targeting is EXACT.  Do not use substring/fuzzy
        -- matching here, otherwise target "A" can accidentally select "AB".
        local wanted = tostring(targetName):match("^%s*(.-)%s*$")
        local closest, closestDist = nil, math.huge
        local seenNames = {}

        for _, mob in ipairs(enemiesFolder:GetChildren()) do
            local hum = mob:FindFirstChildOfClass("Humanoid")
            local mobHRP = mob:FindFirstChild("HumanoidRootPart")
            table.insert(seenNames, mob.Name .. (hum and (" (HP:" .. hum.Health .. ")") or " (no humanoid)"))

            if hum and hum.Health > 0 and mobHRP then
                local actual = tostring(mob.Name):match("^%s*(.-)%s*$")
                if actual == wanted then
                    local dist = (hrp.Position - mobHRP.Position).Magnitude
                    if dist < closestDist then
                        closest = mob
                        closestDist = dist
                    end
                end
            end
        end

        if not closest then
            warn("[QuestDebug] EXACT target not found: '" .. tostring(targetName) .. "' | Found: " .. table.concat(seenNames, ", "))
        end

        return closest
    end

    local function isTargetAlive(mob)
        if not mob then return false end
        if not mob.Parent then return false end
        local hum = mob:FindFirstChildOfClass("Humanoid")
        local mobHRP = mob:FindFirstChild("HumanoidRootPart")
        if not hum or not mobHRP then return false end
        if hum.Health <= 0 then return false end
        return true
    end

    -- The quest loop is shared by both the UI callback and the saved-state
    -- restore path.  This prevents a toggle from looking ON while no worker
    -- thread is actually running after rejoin.
    local function startQuestKillLoop()
        if not mobKillActive or not selectedQuestTarget then return false end
        if mobKillThread then
            task.cancel(mobKillThread)
            mobKillThread = nil
        end

        acceptSelectedQuest()
        task.wait(0.3)

        mobKillThread = task.spawn(function()
            local notFoundTime = 0
            local portalIndex = 1
            local currentIsland = nil
            local QUEST_TARGET_WAIT = 8

            while mobKillActive do
                if not isTargetAlive(currentTarget) then
                    currentTarget = findQuestEnemy(selectedQuestTarget)
                end

                if currentTarget then
                    notFoundTime = 0
                    TargetLabel:Set("Current Target: " .. currentTarget.Name)
                    moveToTarget(currentTarget, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.25)
                    fireM1()
                else
                    -- Stay on the current island and wait for the EXACT target.
                    -- No teleport is allowed before the full 8-second grace period.
                    notFoundTime = notFoundTime + 0.1
                    TargetLabel:Set(
                        "Current Target: Waiting for exact '" .. tostring(selectedQuestTarget) .. "' (" ..
                        math.min(math.floor(notFoundTime * 10) / 10, QUEST_TARGET_WAIT) .. "/" .. QUEST_TARGET_WAIT .. "s)..."
                    )

                    if notFoundTime >= QUEST_TARGET_WAIT then
                        -- One final exact-name check immediately before changing islands.
                        local doubleCheck = findQuestEnemy(selectedQuestTarget)
                        if doubleCheck then
                            currentTarget = doubleCheck
                            notFoundTime = 0
                        else
                            currentIsland = PORTAL_IDS[portalIndex]
                            pcall(function()
                                RS.Remotes.TeleportToPortal:FireServer(currentIsland)
                            end)
                            TargetLabel:Set("Current Target: Not found after 8s, teleporting to " .. currentIsland .. "...")

                            portalIndex = portalIndex % #PORTAL_IDS + 1
                            notFoundTime = 0
                            currentTarget = nil
                            task.wait(0.8)
                        end
                    end
                end

                task.wait(0.1)
            end

            TargetLabel:Set("Current Target: None")
        end)

        return true
    end

    QuestTab:CreateToggle({
        Name = "Auto Kill Quest Target",
        CurrentValue = getSavedToggleValue("QuestKillToggle", false),
        Flag = "QuestKillToggle",
        Callback = function(state)
            mobKillActive = state
            persistFlag("QuestKillToggle", state)
            currentTarget = nil

            if state then
                if not selectedQuestTarget then
                    -- Saved UI can finish loading a moment after the control is
                    -- created, so re-resolve the saved quest before giving up.
                    local saved = getSavedValue("QuestDropdown", nil)
                    if type(saved) == "table" then saved = saved[1] end
                    if saved and labelToTarget[saved] then
                        selectedQuestTarget = labelToTarget[saved]
                        selectedQuestIdentity = labelToIdentity[saved]
                    end
                end

                if not selectedQuestTarget then
                    Rayfield:Notify({ Title = "Quest", Content = "Select a quest first!", Duration = 3 })
                    mobKillActive = false
                    persistFlag("QuestKillToggle", false)
                    return
                end

                startQuestKillLoop()
                Rayfield:Notify({ Title = "Quest", Content = "Auto Kill ON: " .. selectedQuestTarget, Duration = 2 })

                -- Keep the original skill linkage.
                pcall(function()
                    if Rayfield.Flags["SkillCastToggle"] then
                        Rayfield.Flags["SkillCastToggle"]:Set(true)
                    end
                end)
            else
                if mobKillThread then
                    task.cancel(mobKillThread)
                    mobKillThread = nil
                end
                currentTarget = nil
                TargetLabel:Set("Current Target: None")
                Rayfield:Notify({ Title = "Quest", Content = "Auto Kill OFF", Duration = 2 })

                pcall(function()
                    if Rayfield.Flags["SkillCastToggle"] then
                        Rayfield.Flags["SkillCastToggle"]:Set(false)
                    end
                end)
            end
        end,
    })

    -- ── AUTO STATS ──
    -- Each toggle repeatedly spends 30 points on its own stat.
    local AUTO_STAT_POINTS = 30
    local autoStatStates = {}
    local autoStatThreads = {}

    local function startAutoStat(statName)
        if autoStatThreads[statName] then
            task.cancel(autoStatThreads[statName])
            autoStatThreads[statName] = nil
        end

        autoStatThreads[statName] = task.spawn(function()
            while autoStatStates[statName] do
                pcall(function()
                    RS.Remotes.Functions.Input:InvokeServer(
                        "AddPoint",
                        statName,
                        AUTO_STAT_POINTS
                    )
                end)
                task.wait(0.25)
            end
            autoStatThreads[statName] = nil
        end)
    end

    local function createAutoStatToggle(displayName, statName)
        autoStatStates[statName] = getSavedToggleValue("AutoStat_" .. statName, false)

        QuestTab:CreateToggle({
            Name = "Auto " .. displayName,
            CurrentValue = autoStatStates[statName],
            Flag = "AutoStat_" .. statName,
            Callback = function(state)
                autoStatStates[statName] = state
                persistFlag("AutoStat_" .. statName, state)

                if state then
                    startAutoStat(statName)
                else
                    if autoStatThreads[statName] then
                        task.cancel(autoStatThreads[statName])
                        autoStatThreads[statName] = nil
                    end
                end
            end,
        })
    end

    QuestTab:CreateSection("Auto Stats")
    createAutoStatToggle("Strength", "Strength")
    createAutoStatToggle("Defense", "Defense")
    createAutoStatToggle("Weapon", "Weapon")
    createAutoStatToggle("Ability", "Ability")

    -- ── TAB: DUNGEON (Portal Loop) ──
    -- Quest Hub Dungeon settings are intentionally NOT auto-saved.
    local DungeonTab = Window:CreateTab("Dungeon", "castle")

    local PORTAL_REMOTES = {
        "CursedChildPortal", "RealmBeyondHeavenPortal",
        "HouseOfSpidersPortal", "TimeSafePortal", "SpiderEstatePortal",
    }

    local selectedPortalRemote = PORTAL_REMOTES[1]
    local portalLoopActive     = false
    local portalLoopThread     = nil

    DungeonTab:CreateDropdown({
        Name = "Select Portal",
        Options = PORTAL_REMOTES,
        CurrentOption = { PORTAL_REMOTES[1] },
        MultipleOptions = false,
        Flag = "PortalRemoteDropdown",
        Callback = function(option)
            selectedPortalRemote = type(option) == "table" and option[1] or option
        end,
    })

    DungeonTab:CreateToggle({
        Name = "Auto Portal Loop",
        CurrentValue = false,
        Flag = "PortalLoopToggle",
        Callback = function(state)
            portalLoopActive = state

            if state then
                if portalLoopThread then task.cancel(portalLoopThread); portalLoopThread = nil end
                portalLoopThread = task.spawn(function()
                    local portalEvent = RS.Remotes.Events[selectedPortalRemote]
                    pcall(function()
                        portalEvent:FireServer("Create")
                    end)
                    task.wait(1)
                    while portalLoopActive do
                        pcall(function()
                            portalEvent:FireServer("Start")
                        end)
                        task.wait(0.5)
                    end
                end)
                Rayfield:Notify({ Title = "Dungeon", Content = "Portal Loop ON: " .. selectedPortalRemote, Duration = 2 })
            else
                if portalLoopThread then task.cancel(portalLoopThread); portalLoopThread = nil end
                Rayfield:Notify({ Title = "Dungeon", Content = "Portal Loop OFF", Duration = 2 })
            end
        end,
    })

    -- ── TAB: BOSS SUMMON ──
    local BossTab = Window:CreateTab("Boss Summon", "skull")

    local NPC_BOSSES = {
        ["White Whisperer"]     = { "Blast", "Garou", "Flashy Flash" },
        ["The Whisperer"]       = { "Sosuke Aizen", "Ichigo Kurosaki", "Ichigo Kurosaki Bankai" },
        ["Gray Whisperer"]      = { "Ken Kaneki" },
        ["Yellow Whisperer"]    = { "Satoru Gojo", "Ryomen Sukuna" },
        ["Pink Whisperer"]      = { "Akaza" },
        ["PauPau Whisperer"]    = { "Chihora" },
        ["Fish Whisperer"]      = { "Fishman Captain" },
        ["Undyne Summon"]        = { "Undyne" },
        ["Griefbound Ferryman"]  = { "Solemn Lament" },
        ["Sacrifice Table"]     = { "One-Eyed Owl", "Cid Kagenou", "The Red Mist", "Demon Infernal", "Dio" },
    }

    local BOSS_ISLANDS = {
        ["Gray Whisperer"]     = "TokyoGhoul",
        ["Griefbound Ferryman"] = "GraveIsland",
        ["PauPau Whisperer"]   = "RuinCity",
        ["Pink Whisperer"]     = "SlayerMansion",
        ["Sacrifice Table"]    = "RuinCity",
        ["The Whisperer"]      = "HollowLand",
        ["Undyne Summon"]      = "UndyneTrial",
        ["White Whisperer"]    = "ACity",
        ["Yellow Whisperer"]   = "JujutsuAcademy",
        ["Fish Whisperer"]     = "FishmanIsland",
    }

    local NPC_LOCATIONS = {}
    for loc, _ in pairs(NPC_BOSSES) do table.insert(NPC_LOCATIONS, loc) end
    table.sort(NPC_LOCATIONS)

    local selectedBossLocation   = getSavedValue("BossLocationDropdown", NPC_LOCATIONS[1])
    if type(selectedBossLocation) == "table" then selectedBossLocation = selectedBossLocation[1] end
    if not NPC_BOSSES[selectedBossLocation] then selectedBossLocation = NPC_LOCATIONS[1] end

    local selectedBossName       = getSavedValue("BossNameDropdown", NPC_BOSSES[selectedBossLocation][1])
    if type(selectedBossName) == "table" then selectedBossName = selectedBossName[1] end
    local validBoss = false
    for _, bossName in ipairs(NPC_BOSSES[selectedBossLocation]) do
        if bossName == selectedBossName then validBoss = true break end
    end
    if not validBoss then selectedBossName = NPC_BOSSES[selectedBossLocation][1] end

    local selectedBossDifficulty = getSavedValue("BossDifficultyDropdown", "Normal")
    if type(selectedBossDifficulty) == "table" then selectedBossDifficulty = selectedBossDifficulty[1] end
    local validBossDifficulties = { Normal = true, Medium = true, Hard = true, Extreme = true }
    if not validBossDifficulties[selectedBossDifficulty] then selectedBossDifficulty = "Normal" end


    local BossDropdown

    BossTab:CreateDropdown({
        Name = "Boss Location",
        Options = NPC_LOCATIONS,
        CurrentOption = { selectedBossLocation },
        MultipleOptions = false,
        Flag = "BossLocationDropdown",
        Callback = function(option)
            selectedBossLocation = type(option) == "table" and option[1] or option
            persistFlag("BossLocationDropdown", { selectedBossLocation })
            local bosses = NPC_BOSSES[selectedBossLocation] or {}
            selectedBossName = bosses[1]
            persistFlag("BossNameDropdown", { selectedBossName })
            pcall(function()
                BossDropdown:Refresh(bosses, true)
            end)
        end,
    })

    BossDropdown = BossTab:CreateDropdown({
        Name = "Select Boss",
        Options = NPC_BOSSES[selectedBossLocation],
        CurrentOption = { selectedBossName },
        MultipleOptions = false,
        Flag = "BossNameDropdown",
        Callback = function(option)
            selectedBossName = type(option) == "table" and option[1] or option
            persistFlag("BossNameDropdown", { selectedBossName })
        end,
    })

    BossTab:CreateDropdown({
        Name = "Difficulty",
        Options = {"Normal", "Medium", "Hard", "Extreme"},
        CurrentOption = getSavedDropdownValue("BossDifficultyDropdown", {"Normal"}),
        MultipleOptions = false,
        Flag = "BossDifficultyDropdown",
        Callback = function(option)
            selectedBossDifficulty = type(option) == "table" and option[1] or option
            persistFlag("BossDifficultyDropdown", { selectedBossDifficulty })
        end,
    })

    BossTab:CreateButton({
        Name = "Summon Boss",
        Callback = function()
            local islandId = BOSS_ISLANDS[selectedBossLocation]
            if islandId then
                pcall(function()
                    RS.Remotes.TeleportToPortal:FireServer(islandId)
                end)
                task.wait(1)
            end

            local ok = pcall(function()
                if selectedBossLocation == "Fish Whisperer" then
                    RS.Remotes.Functions.Input:InvokeServer(
                        "AutoSpawnBoss",
                        selectedBossName,
                        true,
                        selectedBossLocation,
                        selectedBossDifficulty
                    )
                else
                    RS.Remotes.Functions.Input:InvokeServer(
                        "SpawnBoss",
                        selectedBossLocation,
                        selectedBossName,
                        selectedBossDifficulty
                    )
                end
            end)

            if ok then
                Rayfield:Notify({ Title = "Boss Summon", Content = "Summoned " .. selectedBossName .. " at " .. (islandId or selectedBossLocation), Duration = 3 })
            else
                Rayfield:Notify({ Title = "Boss Summon", Content = "Failed — check tickets/money.", Duration = 3 })
            end
        end,
    })

    local autoSummonActive = getSavedToggleValue("AutoSummonToggle", false)
    local autoSummonThread = nil

    -- Finds the summoned boss by exact name in workspace.Enemies (same folder as regular mobs)
    local function findBossInEnemies(bossName)
        local hrp = getHRP()
        if not hrp then return nil end
        local enemiesFolder = workspace:FindFirstChild("Enemies")
        if not enemiesFolder then return nil end

        local nameLower = bossName:lower()
        local closest, closestDist = nil, math.huge

        for _, mob in ipairs(enemiesFolder:GetChildren()) do
            local hum = mob:FindFirstChildOfClass("Humanoid")
            local mobHRP = mob:FindFirstChild("HumanoidRootPart")
            if hum and hum.Health > 0 and mobHRP then
                local mobNameLower = mob.Name:lower()
                if mobNameLower == nameLower then
                    local dist = (hrp.Position - mobHRP.Position).Magnitude
                    if dist < closestDist then
                        closest = mob
                        closestDist = dist
                    end
                end
            end
        end
        return closest
    end

    local function isBossAlive(mob)
        if not mob or not mob.Parent then return false end
        local hum = mob:FindFirstChildOfClass("Humanoid")
        if not hum then return false end
        return hum.Health > 0
    end

    BossTab:CreateToggle({
        Name = "Auto Summon Loop",
        CurrentValue = getSavedToggleValue("AutoSummonToggle", false),
        Flag = "AutoSummonToggle",
        Callback = function(state)
            autoSummonActive = state
            persistFlag("AutoSummonToggle", state)

            if state then
                if autoSummonThread then
                    task.cancel(autoSummonThread)
                    autoSummonThread = nil
                end

                autoSummonThread = task.spawn(function()
                    local islandId = BOSS_ISLANDS[selectedBossLocation]

                    if islandId then
                        pcall(function()
                            RS.Remotes.TeleportToPortal:FireServer(islandId)
                        end)
                        task.wait(1)
                    end

                    -- Fish Whisperer uses AutoSpawnBoss once. The server-side
                    -- auto spawn is stopped with the matching false call below.
                    if selectedBossLocation == "Fish Whisperer" then
                        pcall(function()
                            RS.Remotes.Functions.Input:InvokeServer(
                                "AutoSpawnBoss",
                                selectedBossName,
                                true,
                                selectedBossLocation,
                                selectedBossDifficulty
                            )
                        end)

                        while autoSummonActive do
                            local boss = findBossInEnemies(selectedBossName)
                            if boss then
                                while autoSummonActive and isBossAlive(boss) do
                                    moveToTarget(boss, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.25)
                                    fireM1()
                                    task.wait(0.1)
                                end
                            end
                            task.wait(0.5)
                        end
                    else
                        while autoSummonActive do
                            pcall(function()
                                RS.Remotes.Functions.Input:InvokeServer(
                                    "SpawnBoss",
                                    selectedBossLocation,
                                    selectedBossName,
                                    selectedBossDifficulty
                                )
                            end)

                            task.wait(0.8)

                            local boss = nil
                            local searchTime = 0
                            while autoSummonActive and not boss and searchTime < 10 do
                                boss = findBossInEnemies(selectedBossName)
                                if not boss then
                                    task.wait(0.2)
                                    searchTime = searchTime + 0.2
                                end
                            end

                            if boss then
                                while autoSummonActive and isBossAlive(boss) do
                                    moveToTarget(boss, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.25)
                                    fireM1()
                                    task.wait(0.1)
                                end
                            end

                            task.wait(0.2)
                        end
                    end
                end)

                Rayfield:Notify({ Title = "Boss Summon", Content = "Auto Summon + CFrame Kill ON: " .. selectedBossLocation, Duration = 2 })
            else
                if autoSummonThread then
                    task.cancel(autoSummonThread)
                    autoSummonThread = nil
                end

                if selectedBossLocation == "Fish Whisperer" then
                    -- Important: send false exactly once when the toggle is OFF.
                    pcall(function()
                        RS.Remotes.Functions.Input:InvokeServer(
                            "AutoSpawnBoss",
                            selectedBossName,
                            false,
                            selectedBossLocation,
                            selectedBossDifficulty
                        )
                    end)
                end

                Rayfield:Notify({ Title = "Boss Summon", Content = "Auto Summon OFF", Duration = 2 })
            end
        end,
    })

    -- ── TAB: SETTINGS (Codes) ──
    local SettingsTab = Window:CreateTab("Settings", "settings")

    SettingsTab:CreateSection("Movement")

    SettingsTab:CreateSlider({
        Name = "Enemy CFrame Teleport Speed",
        Range = {1, 200},
        Increment = 1,
        CurrentValue = math.clamp(tonumber(getSavedValue("EnemyFlySpeedSlider", 5)) or 5, 1, 200),
        Flag = "EnemyFlySpeedSlider",
        Callback = function(value)
            ENEMY_FLY_SPEED = math.clamp(tonumber(value) or 5, 1, 200)
            persistFlag("EnemyFlySpeedSlider", ENEMY_FLY_SPEED)
        end,
    })


    -- ════════════════════════════════════════════════════════
    -- ════════════════════════════════════════════════════════
    --  TAB: FIRE FORCE QUEST
    -- ════════════════════════════════════════════════════════
    local FireForceTab = Window:CreateTab("Fire Force", "flame")

    local FF_ISLANDS = {
        SEVENTH_COMPANY = "7thCompanyIsland",
        A_CITY          = "A-City",
        RUIN_CITY       = "RuinCity",
    }

    local function ffTeleport(islandId)
        pcall(function()
            RS.Remotes.TeleportToPortal:FireServer(islandId)
        end)
    end

    -- Finds a live NPC by exact/fuzzy name, optionally restricted to a
    -- given list of allowed island IDs the player must currently be on.
    -- (We can't directly read "which island am I on" without a signal for
    -- it, so for island-restricted searches we just search workspace.Enemies
    -- wherever we currently are — since we only teleport within the allowed
    -- island set for that quest, we never end up anywhere else.)
    local function findFFEnemy(targetName, maxDist)
        local hrp = getHRP()
        if not hrp then return nil end
        local enemiesFolder = workspace:FindFirstChild("Enemies")
        if not enemiesFolder then return nil end

        local nameLower = targetName:lower()
        local closest, closestDist = nil, math.huge

        for _, mob in ipairs(enemiesFolder:GetChildren()) do
            local hum = mob:FindFirstChildOfClass("Humanoid")
            local mobHRP = mob:FindFirstChild("HumanoidRootPart")
            if hum and hum.Health > 0 and mobHRP then
                local mobNameLower = mob.Name:lower()
                if mobNameLower == nameLower then
                    local dist = (hrp.Position - mobHRP.Position).Magnitude
                    if dist < closestDist and (not maxDist or dist <= maxDist) then
                        closest = mob
                        closestDist = dist
                    end
                end
            end
        end
        return closest
    end

    local function isFFTargetAlive(mob)
        if not mob or not mob.Parent then return false end
        local hum = mob:FindFirstChildOfClass("Humanoid")
        return hum ~= nil and hum.Health > 0
    end

    local function flyToAndKill(mob)
        moveToTarget(mob, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.25)
        fireM1()
    end

    local function fireProximityPrompt(prompt)
        if not prompt then return false end
        return pcall(function()
            if fireproximityprompt then
                fireproximityprompt(prompt)
            else
                prompt:InputHoldBegin()
                task.wait(prompt.HoldDuration or 0)
                prompt:InputHoldEnd()
            end
        end)
    end

    local FF_QUEST_TYPES = {
        "Ambush (Infernal Ambusher)",
        "Boss - Blast (White Whisperer)",
        "World Boss - Demon Infernal",
        "Lost Cat",
        "Hostage Rescue",
    }

    local selectedFFQuest = getSavedValue("FireForceQuestDropdown", FF_QUEST_TYPES[1])
    if type(selectedFFQuest) == "table" then selectedFFQuest = selectedFFQuest[1] end
    local validFFQuest = false
    for _, questName in ipairs(FF_QUEST_TYPES) do
        if questName == selectedFFQuest then validFFQuest = true break end
    end
    if not validFFQuest then selectedFFQuest = FF_QUEST_TYPES[1] end
    FireForceTab:CreateDropdown({
        Name = "Quest Type",
        Options = FF_QUEST_TYPES,
        CurrentOption = getSavedDropdownValue("FireForceQuestDropdown", { FF_QUEST_TYPES[1] }),
        MultipleOptions = false,
        Flag = "FireForceQuestDropdown",
        Callback = function(option)
            selectedFFQuest = type(option) == "table" and option[1] or option
            persistFlag("FireForceQuestDropdown", { selectedFFQuest })
        end,
    })

    local ffActive = getSavedToggleValue("FireForceToggle", false)
    local ffThread = nil

    -- ── Ambush: fire the Ambush request, then find+kill "Infernal Ambusher"
    -- only on 7th Company or Ruin City — never teleport to any other island.
    local function runAmbushLoop()
        local ambushIslands = { FF_ISLANDS.SEVENTH_COMPANY, FF_ISLANDS.RUIN_CITY }
        local islandCursor = 1

        while ffActive do
            pcall(function()
                RS.Remotes.Functions.Input:InvokeServer("FireForce", "Ambush")
            end)
            task.wait(0.5)

            local target = findFFEnemy("Infernal Ambusher")
            local searchTime = 0
            while ffActive and not target and searchTime < 6 do
                target = findFFEnemy("Infernal Ambusher")
                if not target then
                    task.wait(0.3)
                    searchTime = searchTime + 0.3
                end
            end

            if target then
                while ffActive and isFFTargetAlive(target) do
                    flyToAndKill(target)
                    task.wait(0.1)
                end
                -- dead — loop will immediately fire Ambush again
            else
                -- Not found on this island after searching — try the other
                -- allowed island (7th Company <-> Ruin City), never elsewhere.
                islandCursor = islandCursor % #ambushIslands + 1
                ffTeleport(ambushIslands[islandCursor])
                task.wait(1)
            end
        end
    end

    -- ── Boss: teleport to A City, summon Blast, find+kill, repeat.
    local function runBlastBossLoop()
        ffTeleport(FF_ISLANDS.A_CITY)
        task.wait(1.5)

        while ffActive do
            pcall(function()
                RS.Remotes.Functions.Input:InvokeServer("SpawnBoss", "White Whisperer", "Blast", "Normal")
            end)
            task.wait(1)

            local target = findFFEnemy("Blast")
            local searchTime = 0
            while ffActive and not target and searchTime < 8 do
                target = findFFEnemy("Blast")
                if not target then
                    task.wait(0.3)
                    searchTime = searchTime + 0.3
                end
            end

            if target then
                while ffActive and isFFTargetAlive(target) do
                    flyToAndKill(target)
                    task.wait(0.1)
                end
            end
        end
    end

    -- ── World Boss: summon Demon Infernal, search island by island until
    -- found (no restriction — this one can be anywhere), kill, repeat.
    local function runDemonInfernalLoop()
        local islandCursor = 1

        while ffActive do
            pcall(function()
                RS.Remotes.Functions.Input:InvokeServer("SpawnBoss", "Sacrifice Table", "Demon Infernal", "Normal")
            end)
            task.wait(1)

            local target = findFFEnemy("Demon Infernal")
            local searchTime = 0
            while ffActive and not target and searchTime < 6 do
                target = findFFEnemy("Demon Infernal")
                if not target then
                    task.wait(0.3)
                    searchTime = searchTime + 0.3
                end
            end

            if target then
                while ffActive and isFFTargetAlive(target) do
                    flyToAndKill(target)
                    task.wait(0.1)
                end
            else
                islandCursor = islandCursor % #PORTAL_IDS + 1
                ffTeleport(PORTAL_IDS[islandCursor])
                task.wait(1)
            end
        end
    end

    -- ── Lost Cat: go to 7th Company, fly to the cat, fire its prompt.
    local function runLostCatLoop()
        ffTeleport(FF_ISLANDS.SEVENTH_COMPANY)
        task.wait(1.5)

        while ffActive do
            local extra = workspace:FindFirstChild("Extra")
            local questFolder = extra and extra:FindFirstChild("FireForceQuest")
            local cat = questFolder and questFolder:FindFirstChild("Lost Cat")

            if cat then
                local catHRP = cat:FindFirstChild("HumanoidRootPart")
                if catHRP then
                    moveToTarget(cat, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 15)
                    task.wait(0.3)
                    local prompt = catHRP:FindFirstChild("ProximityPrompt")
                    fireProximityPrompt(prompt)
                end
            end
            task.wait(1)
        end
    end

    -- ── Hostage Rescue: go to Ruin City, find hostage, fire prompt, handle
    -- any Infernal Ambusher that spawns nearby, then go claim at Captain Burns.
    local function runHostageRescueLoop()
        ffTeleport(FF_ISLANDS.RUIN_CITY)
        task.wait(1.5)

        while ffActive do
            local extra = workspace:FindFirstChild("Extra")
            local questFolder = extra and extra:FindFirstChild("FireForceQuest")
            local hostage = questFolder and questFolder:FindFirstChild("Fire Force Hostage")

            if hostage then
                local hostageHRP = hostage:FindFirstChild("HumanoidRootPart")
                if hostageHRP then
                    moveToTarget(hostage, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 15)
                    task.wait(0.3)
                    local prompt = hostageHRP:FindFirstChild("ProximityPrompt")
                    fireProximityPrompt(prompt)
                    task.wait(0.5)

                    -- An Infernal Ambusher may spawn right after — only treat
                    -- it as ours if it's actually near us here at Ruin City
                    -- (maxDist stops us grabbing an Ambusher spawned elsewhere
                    -- on the map). Also give it a moment to spawn instead of
                    -- checking just once.
                    local AMBUSH_DETECT_RADIUS = 100
                    local ambusher = nil
                    local ambushSearchTime = 0
                    while ffActive and not ambusher and ambushSearchTime < 3 do
                        ambusher = findFFEnemy("Infernal Ambusher", AMBUSH_DETECT_RADIUS)
                        if not ambusher then
                            task.wait(0.2)
                            ambushSearchTime = ambushSearchTime + 0.2
                        end
                    end

                    if ambusher then
                        while ffActive and isFFTargetAlive(ambusher) do
                            flyToAndKill(ambusher)
                            task.wait(0.1)
                        end
                        task.wait(0.3)
                        if hostage.Parent and hostageHRP.Parent then
                            moveToTarget(hostage, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 15)
                            task.wait(0.3)
                            fireProximityPrompt(hostageHRP:FindFirstChild("ProximityPrompt"))
                        end
                    end
                end
            else
                -- Hostage gone (rescued) — go claim the reward from Captain Burns
                ffTeleport(FF_ISLANDS.SEVENTH_COMPANY)
                task.wait(1.5)

                local npcFolder = workspace:FindFirstChild("NPCs")
                local burns = npcFolder and npcFolder:FindFirstChild("Captain Burns")
                if burns then
                    local burnsHRP = burns:FindFirstChild("HumanoidRootPart")
                    if burnsHRP then
                        moveToTarget(burns, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 15)
                        task.wait(0.3)
                    end
                end

                pcall(function()
                    RS.Remotes.Functions.Input:InvokeServer("FireForce", "Claim", "HostageRescue")
                end)

                task.wait(1)
                ffTeleport(FF_ISLANDS.RUIN_CITY)
                task.wait(1.5)
            end
            task.wait(0.5)
        end
    end

    FireForceTab:CreateToggle({
        Name = "Auto Fire Force Quest",
        CurrentValue = getSavedToggleValue("FireForceToggle", false),
        Flag = "FireForceToggle",
        Callback = function(state)
            ffActive = state
            persistFlag("FireForceToggle", state)
            if state then
                if ffThread then task.cancel(ffThread); ffThread = nil end
                ffThread = task.spawn(function()
                    if selectedFFQuest == FF_QUEST_TYPES[1] then
                        runAmbushLoop()
                    elseif selectedFFQuest == FF_QUEST_TYPES[2] then
                        runBlastBossLoop()
                    elseif selectedFFQuest == FF_QUEST_TYPES[3] then
                        runDemonInfernalLoop()
                    elseif selectedFFQuest == FF_QUEST_TYPES[4] then
                        runLostCatLoop()
                    elseif selectedFFQuest == FF_QUEST_TYPES[5] then
                        runHostageRescueLoop()
                    end
                end)
                Rayfield:Notify({ Title = "Fire Force", Content = "Started: " .. selectedFFQuest, Duration = 3 })
            else
                if ffThread then task.cancel(ffThread); ffThread = nil end
                Rayfield:Notify({ Title = "Fire Force", Content = "Stopped", Duration = 2 })
            end
        end,
    })

    -- ════════════════════════════════════════════════════════
    --  TAB: TELEPORT
    -- ════════════════════════════════════════════════════════
    local TeleportTab = Window:CreateTab("Teleport", "map-pin")

    local ISLAND_LIST = {
        "Legacy", "Starter", "Jungle", "JujutsuAcademy", "HollowLand",
        "SlayerMansion", "TokyoGhoul", "7thCompanyIsland", "RuinCity",
        "FishmanIsland", "GraveIsland", "ACity", "IceIsland", "UndyneTrial",
    }

    local selectedIsland = getSavedValue("TeleportIslandDropdown", ISLAND_LIST[1])
    if type(selectedIsland) == "table" then selectedIsland = selectedIsland[1] end
    if not table.find(ISLAND_LIST, selectedIsland) then selectedIsland = ISLAND_LIST[1] end

    TeleportTab:CreateDropdown({
        Name = "Select Island",
        Options = ISLAND_LIST,
        CurrentOption = getSavedDropdownValue("TeleportIslandDropdown", { ISLAND_LIST[1] }),
        MultipleOptions = false,
        Flag = "TeleportIslandDropdown",
        Callback = function(option)
            selectedIsland = type(option) == "table" and option[1] or option
            persistFlag("TeleportIslandDropdown", { selectedIsland })
        end,
    })

    TeleportTab:CreateButton({
        Name = "Teleport",
        Callback = function()
            local ok = pcall(function()
                RS.Remotes.TeleportToPortal:FireServer(selectedIsland)
            end)
            if ok then
                Rayfield:Notify({ Title = "Teleport", Content = "Teleporting to " .. selectedIsland, Duration = 2 })
            else
                Rayfield:Notify({ Title = "Teleport", Content = "Failed to teleport.", Duration = 3 })
            end
        end,
    })

    -- ── Load all saved values now that every Flag exists ──
    task.spawn(function()
        task.wait(1)

        ConfigHydrating = true
        loadSavedFlagsInOrder({
            "WeaponDropdown",
            "AutoEquipToggle",
            "AutoM1Toggle",
            "SkillMultiDropdown",
            "SkillModeDropdown",
            "SkillRadiusSlider",
            "SkillCastToggle",
            "QuestDropdown",
            "QuestKillToggle",
            "AutoStat_Strength",
            "AutoStat_Defense",
            "AutoStat_Weapon",
            "AutoStat_Ability",
            "BossLocationDropdown",
            "BossNameDropdown",
            "BossDifficultyDropdown",
            "AutoSummonToggle",
            "FireForceQuestDropdown",
            "FireForceToggle",
            "TeleportIslandDropdown",
            "EnemyFlySpeedSlider",
        })

        ConfigHydrating = false
        ConfigReady = true
        ConfigDirty = true
        writeConfigFile()

        -- Re-start saved features after hydration. Rayfield:Set can restore the
        -- UI value without necessarily running the callback, so saved toggles
        -- must be activated explicitly here.
        for _, statName in ipairs({"Strength", "Defense", "Weapon", "Ability"}) do
            if autoStatStates[statName] then
                startAutoStat(statName)
            end
        end
        if autoEquipActive then
            if selectedWeapon then equipWeapon(selectedWeapon) end
            if weaponWatchConn then weaponWatchConn:Disconnect(); weaponWatchConn = nil end
            weaponWatchConn = LP.CharacterAdded:Connect(function(char)
                task.wait(1)
                if autoEquipActive and selectedWeapon then
                    equipWeapon(selectedWeapon)
                end
            end)
        end

        if autoM1Active then
            if autoM1Thread then task.cancel(autoM1Thread); autoM1Thread = nil end
            autoM1Thread = task.spawn(function()
                while autoM1Active do
                    fireSelectedWeaponM1()
                    task.wait(0.1)
                end
            end)
        end

        -- Re-resolve the saved quest after all dropdowns exist.
        do
            local saved = getSavedValue("QuestDropdown", nil)
            if type(saved) == "table" then saved = saved[1] end
            if saved and labelToTarget[saved] then
                selectedQuestTarget = labelToTarget[saved]
                selectedQuestIdentity = labelToIdentity[saved]
            end
        end

        if mobKillActive and selectedQuestTarget then
            startQuestKillLoop()
            -- Saved quest auto-kill should also restore its normal skill linkage.
            if skillCastActive then
                if skillCastThread then task.cancel(skillCastThread); skillCastThread = nil end
                skillCastThread = task.spawn(function()
                    while skillCastActive do
                        if #selectedSkills > 0 then
                            local canCast = skillMode ~= "Only Near Mob" or isMobNearby(mobDetectRadius)
                            if canCast then
                                for _, key in ipairs(selectedSkills) do
                                    if not skillCastActive then break end
                                    fireSkill(key)
                                    task.wait(0.15)
                                end
                            end
                        end
                        task.wait(0.2)
                    end
                end)
            end
        elseif skillCastActive then
            if skillCastThread then task.cancel(skillCastThread); skillCastThread = nil end
            skillCastThread = task.spawn(function()
                while skillCastActive do
                    if #selectedSkills > 0 then
                        local canCast = skillMode ~= "Only Near Mob" or isMobNearby(mobDetectRadius)
                        if canCast then
                            for _, key in ipairs(selectedSkills) do
                                if not skillCastActive then break end
                                fireSkill(key)
                                task.wait(0.15)
                            end
                        end
                    end
                    task.wait(0.2)
                end
            end)
        end

        if portalLoopActive then
            if portalLoopThread then task.cancel(portalLoopThread); portalLoopThread = nil end
            portalLoopThread = task.spawn(function()
                local portalEvent = RS.Remotes.Events[selectedPortalRemote]
                pcall(function() portalEvent:FireServer("Create") end)
                task.wait(1)
                while portalLoopActive do
                    pcall(function() portalEvent:FireServer("Start") end)
                    task.wait(0.5)
                end
            end)
        end

        if autoSummonActive then
            -- Keep the existing boss callback behavior by recreating the loop from saved state.
            if autoSummonThread then task.cancel(autoSummonThread); autoSummonThread = nil end
            autoSummonThread = task.spawn(function()
                local islandId = BOSS_ISLANDS[selectedBossLocation]
                if islandId then
                    pcall(function() RS.Remotes.TeleportToPortal:FireServer(islandId) end)
                    task.wait(1)
                end
                if selectedBossLocation == "Fish Whisperer" then
                    pcall(function()
                        RS.Remotes.Functions.Input:InvokeServer("AutoSpawnBoss", selectedBossName, true, selectedBossLocation, selectedBossDifficulty)
                    end)
                    while autoSummonActive do
                        local boss = findBossInEnemies(selectedBossName)
                        if boss then
                            while autoSummonActive and isBossAlive(boss) do
                                moveToTarget(boss, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.25)
                                fireM1()
                                task.wait(0.1)
                            end
                        end
                        task.wait(0.5)
                    end
                else
                    while autoSummonActive do
                        pcall(function() RS.Remotes.Functions.Input:InvokeServer("SpawnBoss", selectedBossLocation, selectedBossName, selectedBossDifficulty) end)
                        task.wait(0.8)
                        local boss, searchTime = nil, 0
                        while autoSummonActive and not boss and searchTime < 10 do
                            boss = findBossInEnemies(selectedBossName)
                            if not boss then task.wait(0.2); searchTime = searchTime + 0.2 end
                        end
                        if boss then
                            while autoSummonActive and isBossAlive(boss) do
                                moveToTarget(boss, 3, ENEMY_FLY_SPEED, ENEMY_TELEPORT_RADIUS, 0.25)
                                fireM1()
                                task.wait(0.1)
                            end
                        end
                        task.wait(0.2)
                    end
                end
            end)
        end

        if ffActive then
            if ffThread then task.cancel(ffThread); ffThread = nil end
            ffThread = task.spawn(function()
                if selectedFFQuest == FF_QUEST_TYPES[1] then
                    runAmbushLoop()
                elseif selectedFFQuest == FF_QUEST_TYPES[2] then
                    runBlastBossLoop()
                elseif selectedFFQuest == FF_QUEST_TYPES[3] then
                    runDemonInfernalLoop()
                elseif selectedFFQuest == FF_QUEST_TYPES[4] then
                    runLostCatLoop()
                elseif selectedFFQuest == FF_QUEST_TYPES[5] then
                    runHostageRescueLoop()
                end
            end)
        end
    end)
end
