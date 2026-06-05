-- X-GEN HUB
-- Hitbox por Visibilidade Direta & ESP Dinâmica com GUI
-- [VERSÃO SUPER OTIMIZADA - Lógica Original Mantida]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- CACHE DE FUNÇÕES (Otimização para velocidade)
local Raycast = workspace.Raycast
local FindFirstChild = game.FindFirstChild
local GetPlayers = Players.GetPlayers

-- CONFIGURAÇÕES GLOBAIS
getgenv().Autoshot_Enabled = false 
getgenv().SilentAim_Enabled = false
getgenv().TeamCheck_Enabled = true
getgenv().Bullet_Multiplier = 3
getgenv().Hitbox_Size = 12
getgenv().Hitbox_Transparency = 1
getgenv().Hitbox_Enabled = false
getgenv().ESP_Enabled = false
getgenv().GhostMode_Enabled = false
getgenv().RainbowESP_Enabled = false
getgenv().RainbowTheme_Enabled = false
getgenv().MenuScale = 100
getgenv().AnimX200_Enabled = false
getgenv().AutoCollect_Enabled = false
getgenv().WalkSpeed = 16

local TargetCache = nil
local OriginalHitboxes = {}
local HighlightsCache = {} -- OTIMIZAÇÃO: Cache de instâncias ESP

local configFileName = "XGenHub_Config.json"

local function LoadConfig()
    if readfile and isfile and isfile(configFileName) then
        local success, configData = pcall(function()
            return HttpService:JSONDecode(readfile(configFileName))
        end)
        if success and type(configData) == "table" then
            if configData.Autoshot_Enabled ~= nil then getgenv().Autoshot_Enabled = configData.Autoshot_Enabled end
            if configData.SilentAim_Enabled ~= nil then getgenv().SilentAim_Enabled = configData.SilentAim_Enabled end
            if configData.TeamCheck_Enabled ~= nil then getgenv().TeamCheck_Enabled = configData.TeamCheck_Enabled end
            if configData.Bullet_Multiplier ~= nil then getgenv().Bullet_Multiplier = configData.Bullet_Multiplier end
            if configData.Hitbox_Enabled ~= nil then getgenv().Hitbox_Enabled = configData.Hitbox_Enabled end
            if configData.Hitbox_Size ~= nil then getgenv().Hitbox_Size = configData.Hitbox_Size end
            if configData.Hitbox_Transparency ~= nil then getgenv().Hitbox_Transparency = configData.Hitbox_Transparency end
            if configData.ESP_Enabled ~= nil then getgenv().ESP_Enabled = configData.ESP_Enabled end
            if configData.RainbowESP_Enabled ~= nil then getgenv().RainbowESP_Enabled = configData.RainbowESP_Enabled end
            if configData.GhostMode_Enabled ~= nil then getgenv().GhostMode_Enabled = configData.GhostMode_Enabled end
            if configData.AnimX200_Enabled ~= nil then getgenv().AnimX200_Enabled = configData.AnimX200_Enabled end
            if configData.AutoCollect_Enabled ~= nil then getgenv().AutoCollect_Enabled = configData.AutoCollect_Enabled end
            if configData.WalkSpeed ~= nil then getgenv().WalkSpeed = configData.WalkSpeed end
            if configData.RainbowTheme_Enabled ~= nil then getgenv().RainbowTheme_Enabled = configData.RainbowTheme_Enabled end
            if configData.MenuScale ~= nil then getgenv().MenuScale = configData.MenuScale end
            if configData.MenuColor_R and configData.MenuColor_G and configData.MenuColor_B then
                getgenv().MenuColor = Color3.new(configData.MenuColor_R, configData.MenuColor_G, configData.MenuColor_B)
            end
            if configData.AccentColor_R and configData.AccentColor_G and configData.AccentColor_B then
                getgenv().AccentColor = Color3.new(configData.AccentColor_R, configData.AccentColor_G, configData.AccentColor_B)
            end
        end
    end
end

local function SaveConfig()
    local config = {
        Autoshot_Enabled = getgenv().Autoshot_Enabled,
        SilentAim_Enabled = getgenv().SilentAim_Enabled,
        TeamCheck_Enabled = getgenv().TeamCheck_Enabled,
        Bullet_Multiplier = getgenv().Bullet_Multiplier,
        Hitbox_Enabled = getgenv().Hitbox_Enabled,
        Hitbox_Size = getgenv().Hitbox_Size,
        Hitbox_Transparency = getgenv().Hitbox_Transparency,
        ESP_Enabled = getgenv().ESP_Enabled,
        RainbowESP_Enabled = getgenv().RainbowESP_Enabled,
        GhostMode_Enabled = getgenv().GhostMode_Enabled,
        AnimX200_Enabled = getgenv().AnimX200_Enabled,
        AutoCollect_Enabled = getgenv().AutoCollect_Enabled,
        WalkSpeed = getgenv().WalkSpeed,
        RainbowTheme_Enabled = getgenv().RainbowTheme_Enabled,
        MenuScale = getgenv().MenuScale,
        MenuColor_R = getgenv().MenuColor and getgenv().MenuColor.R or (24/255),
        MenuColor_G = getgenv().MenuColor and getgenv().MenuColor.G or (24/255),
        MenuColor_B = getgenv().MenuColor and getgenv().MenuColor.B or (24/255),
        AccentColor_R = getgenv().AccentColor and getgenv().AccentColor.R or (10/255),
        AccentColor_G = getgenv().AccentColor and getgenv().AccentColor.G or (10/255),
        AccentColor_B = getgenv().AccentColor and getgenv().AccentColor.B or (10/255)
    }
    if writefile then
        pcall(function() writefile(configFileName, HttpService:JSONEncode(config)) end)
    end
end

getgenv().MenuColor = getgenv().MenuColor or Color3.fromRGB(24, 24, 24)
getgenv().AccentColor = getgenv().AccentColor or Color3.fromRGB(10, 10, 10)

LoadConfig()

local GhostTool = nil
local GhostActive = false
local GhostLoop = nil
local OriginalTransparency = {}

local TargetParts = {
    "Head", "UpperTorso", "Torso", "HumanoidRootPart", "LowerTorso",
    "LeftUpperArm", "LeftLowerArm", "LeftHand", "RightUpperArm", "RightLowerArm", "RightHand",
    "LeftUpperLeg", "LeftLowerLeg", "LeftFoot", "RightUpperLeg", "RightLowerLeg", "RightFoot"
}

-- ==========================================
-- 1. BYPASS DE REDIRECIONAMENTO (METATABLE)
-- ==========================================
local mt = getrawmetatable(game)
local oldNamecall = mt.__namecall
setreadonly(mt, false)

local ValidScripts = {}

mt.__namecall = newcclosure(function(self, ...)
    local method = getnamecallmethod()
    local args = {...}
    
    if TargetCache and (method == "Raycast" or method == "FindPartOnRay") and not checkcaller() then
        local origin = (method == "Raycast" and args[1] or args[1].Origin)
        if (origin - Camera.CFrame.Position).Magnitude > 2 then
            local callingScript = getcallingscript and getcallingscript() or nil
            if callingScript then
                if ValidScripts[callingScript] == nil then
                    local name = callingScript.Name:lower()
                    ValidScripts[callingScript] = not (name:find("camera") or name:find("popper") or name:find("zoom"))
                end
                
                if ValidScripts[callingScript] then
                    if method == "Raycast" then
                        args[2] = (TargetCache.Position - origin).Unit * 1000
                        return oldNamecall(self, table.unpack(args))
                    else
                        args[1] = Ray.new(origin, (TargetCache.Position - origin).Unit * 1000)
                        return oldNamecall(self, table.unpack(args))
                    end
                end
            end
        end
    end
    return oldNamecall(self, ...)
end)
setreadonly(mt, true)

-- ==========================================
-- 2. SISTEMA DE ESP (HIGHLIGHT DINÂMICO - OTIMIZADO)
-- ==========================================
local PlayersConnections = {}

local function ApplyChams(player)
    if player == LocalPlayer then return end
    local function setupCharacter(char)
        task.wait(0.1)
        if char and char.Parent then
            local oldHighlight = char:FindFirstChild("XGEN_ESP_HIGHLIGHT")
            if oldHighlight then oldHighlight:Destroy() end
            
            local highlight = Instance.new("Highlight")
            highlight.Name = "XGEN_ESP_HIGHLIGHT"
            highlight.FillColor = Color3.fromRGB(0, 255, 0)
            highlight.OutlineColor = Color3.fromRGB(0, 255, 0)
            highlight.FillTransparency = 0.5
            highlight.OutlineTransparency = 0.1
            highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            highlight.Enabled = getgenv().ESP_Enabled
            highlight.Parent = char
            HighlightsCache[player] = highlight -- Salva no cache!
        end
    end
    if player.Character then setupCharacter(player.Character) end
    PlayersConnections[player] = player.CharacterAdded:Connect(setupCharacter)
end

for _, p in ipairs(GetPlayers(Players)) do pcall(function() ApplyChams(p) end) end
Players.PlayerAdded:Connect(ApplyChams)
Players.PlayerRemoving:Connect(function(p) 
    HighlightsCache[p] = nil 
    OriginalHitboxes[p] = nil 
end)

local function UpdateESPState()
    for _, highlight in pairs(HighlightsCache) do
        if highlight then highlight.Enabled = getgenv().ESP_Enabled end
    end
end

-- ==========================================
-- 3. LÓGICAS DE VISIBILIDADE (OTIMIZADO COM DISTÂNCIA)
-- ==========================================
local SharedRayParams = RaycastParams.new()
SharedRayParams.FilterType = Enum.RaycastFilterType.Blacklist

local function IsVisibleToEnvironment(part, targetChar)
    if not part then return false end
    local origin = Camera.CFrame.Position
    -- Cull de distância para nÃ£o usar CPU com players muito longe (> 2000 studs)
    if (part.Position - origin).Magnitude > 2000 then return false end
    
    SharedRayParams.FilterDescendantsInstances = {LocalPlayer.Character, Camera}
    local ray = workspace:Raycast(origin, (part.Position - origin), SharedRayParams)
    return ray and ray.Instance and ray.Instance:IsDescendantOf(targetChar)
end

local function HasLineOfSight(part, targetChar)
    local character = LocalPlayer.Character
    local hrp = character and character:FindFirstChild("HumanoidRootPart")
    if not hrp or not part then return false end
    
    SharedRayParams.FilterDescendantsInstances = {character, Camera}
    local ray = workspace:Raycast(hrp.Position, (part.Position - hrp.Position), SharedRayParams)
    return ray and ray.Instance and ray.Instance:IsDescendantOf(targetChar)
end

-- ==========================================
-- 4. LOOP PRINCIPAL (THROTTLED & OTIMIZADO)
-- ==========================================
local TickRate = 0
local RSConnection = RunService.RenderStepped:Connect(function(deltaTime)
    TickRate = TickRate + deltaTime
    local character = LocalPlayer.Character
    if not character then return end
    
    local tool = character:FindFirstChildOfClass("Tool")
    local remote = tool and (tool:FindFirstChild("kill") or tool:FindFirstChild("fire") or tool:FindFirstChild("Shoot"))
    
    local isTickFrame = (TickRate >= 0.05) -- OtimizaÃ§Ã£o: Processa raycasts a cada 0.05s (20 TPS) ao em vez de 60 FPS (Lag)
    if isTickFrame then TickRate = 0 end

    local foundTargetPart = nil
    local targetPlayer = nil
    local myHRP = character:FindFirstChild("HumanoidRootPart")

    for _, target in ipairs(GetPlayers(Players)) do
        if target ~= LocalPlayer and target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
            local targetHRP = target.Character.HumanoidRootPart
            local hum = target.Character:FindFirstChild("Humanoid")
            
            if not OriginalHitboxes[target] then OriginalHitboxes[target] = targetHRP.Size end

            local isEnemy = true
            if getgenv().TeamCheck_Enabled then
                local myTeam = LocalPlayer:GetAttribute("Team")
                local enemyTeam = target:GetAttribute("Team")
                if myTeam ~= nil and enemyTeam == myTeam then isEnemy = false end
            end
            
            if isEnemy and hum and hum.Health > 0 then
                -- ESP CORE
                local highlight = HighlightsCache[target]
                if getgenv().ESP_Enabled and highlight and myHRP then
                    local colorToUse
                    if getgenv().RainbowESP_Enabled then
                        colorToUse = Color3.fromHSV((tick() % 5) / 5, 1, 1)
                    else
                        local dist = (myHRP.Position - targetHRP.Position).Magnitude
                        if dist <= 20 then colorToUse = Color3.new(1, 0, 0) 
                        elseif dist <= 40 then colorToUse = Color3.new(1, 0, 0):Lerp(Color3.new(0, 0, 1), (dist - 20) / 20) 
                        elseif dist <= 60 then colorToUse = Color3.new(0, 0, 1):Lerp(Color3.new(0, 1, 0), (dist - 40) / 20) 
                        else colorToUse = Color3.new(0, 1, 0) end
                    end
                    if highlight.FillColor ~= colorToUse then
                        highlight.FillColor = colorToUse
                        highlight.OutlineColor = colorToUse
                    end
                end

                -- VISIBILITY TICK (SÃ³ a cada 0.05 segs para evitar lag brutal)
                if isTickFrame then
                    local visibleToEnv = IsVisibleToEnvironment(targetHRP, target.Character)
                    
                    if getgenv().Hitbox_Enabled and visibleToEnv then
                        local novaSize = Vector3.new(getgenv().Hitbox_Size, getgenv().Hitbox_Size, getgenv().Hitbox_Size)
                        if targetHRP.Size ~= novaSize then
                            targetHRP.Size = novaSize
                            targetHRP.Transparency = getgenv().Hitbox_Transparency
                            targetHRP.Material = Enum.Material.Neon
                            targetHRP.CanCollide = false
                        end
                    else
                        if targetHRP.Size ~= OriginalHitboxes[target] then
                            targetHRP.Size = OriginalHitboxes[target]
                            targetHRP.Transparency = 1
                            targetHRP.Material = Enum.Material.Plastic
                            targetHRP.CanCollide = true
                        end
                    end

                    if (getgenv().Autoshot_Enabled or getgenv().SilentAim_Enabled) and visibleToEnv and not foundTargetPart then
                        for i = 1, #TargetParts do
                            local part = target.Character:FindFirstChild(TargetParts[i])
                            if part and HasLineOfSight(part, target.Character) then
                                foundTargetPart = part
                                targetPlayer = target
                                break
                            end
                        end
                    end
                end
            else
                if targetHRP.Size ~= OriginalHitboxes[target] then
                    targetHRP.Size = OriginalHitboxes[target]
                    targetHRP.Transparency = 1
                end
            end
        end
    end
    
    if isTickFrame then
        if foundTargetPart and targetPlayer then
            TargetCache = foundTargetPart
            if getgenv().Autoshot_Enabled and remote then
                for i = 1, getgenv().Bullet_Multiplier do
                    remote:FireServer(targetPlayer, foundTargetPart.Position)
                end
                tool:Activate()
            end
        else
            TargetCache = nil
        end
    end
end)

-- GHOST & EXTRAS
local function ToggleGhostMode(state)
    if state then
        local function giveTool()
            if GhostTool then return end
            GhostTool = Instance.new("Tool")
            GhostTool.Name = "Ghost"
            GhostTool.RequiresHandle = false
            local backpack = LocalPlayer:FindFirstChild("Backpack")
            if backpack then GhostTool.Parent = backpack end
            GhostTool.Equipped:Connect(function()
                GhostActive = true
                OriginalTransparency = {}
                for _,v in pairs(LocalPlayer.Character:GetDescendants()) do
                    if v:IsA("BasePart") or v:IsA("Decal") then
                        OriginalTransparency[v] = v.Transparency
                        v.Transparency = 0.5
                    end
                end
            end)
            GhostTool.Unequipped:Connect(function()
                GhostActive = false
                for v,t in pairs(OriginalTransparency) do
                    if v and v.Parent then v.Transparency = t end
                end
            end)
        end
        giveTool()
        
        PlayersConnections["GhostAdded"] = LocalPlayer.CharacterAdded:Connect(function()
            if getgenv().GhostMode_Enabled then task.wait(0.5); giveTool() end
        end)

        GhostLoop = RunService.Heartbeat:Connect(function()
            if GhostActive and LocalPlayer.Character then
                local hum = LocalPlayer.Character:FindFirstChild("Humanoid")
                local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                if hum and root then
                    local current = root.CFrame
                    root.CFrame = current * CFrame.new(0,-200000,0)
                    hum.CameraOffset = (current * CFrame.new(0,-200000,0)):ToObjectSpace(current).Position
                    RunService.RenderStepped:Wait()
                    root.CFrame = current
                    hum.CameraOffset = Vector3.new()
                end
            end
        end)
    else
        GhostActive = false
        if GhostTool then GhostTool:Destroy() GhostTool = nil end
        if GhostLoop then GhostLoop:Disconnect() GhostLoop = nil end
        if PlayersConnections["GhostAdded"] then PlayersConnections["GhostAdded"]:Disconnect(); PlayersConnections["GhostAdded"] = nil end
        for v,t in pairs(OriginalTransparency) do
            if v and v.Parent then v.Transparency = t end
        end
    end
end

local AnimX200Task = task.spawn(function()
    while true do
        pcall(function()
            if getgenv().AnimX200_Enabled and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
                local animator = LocalPlayer.Character.Humanoid:FindFirstChildOfClass("Animator")
                if animator then
                    for _, track in pairs(animator:GetPlayingAnimationTracks()) do
                        track:AdjustSpeed(200)
                    end
                end
            end
        end)
        task.wait(1) -- OtimizaÃ§Ã£o: NÃ£o precisa ajustar 100x por segundo. 1x / segundo Ã© estÃ¡vel!
    end
end)

local WalkSpeedTask = task.spawn(function()
    while true do
        pcall(function()
            if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
                local hum = LocalPlayer.Character.Humanoid
                if hum.WalkSpeed ~= 0 and hum.WalkSpeed ~= getgenv().WalkSpeed then
                    hum.WalkSpeed = getgenv().WalkSpeed
                end
            end
        end)
        task.wait(0.5) -- OtimizaÃ§Ã£o
    end
end)

-- OTIMIZACAO EXTREMA: AutoCollect (Removido lag de O(N*M) em ReplicatedStorage)
local AutoCollectTask = task.spawn(function()
    local itensJaColetados = {}
    local RemotesCache = {}
    local RemotesCached = false

    local function loadRemotes()
        for _, v in ipairs(ReplicatedStorage:GetDescendants()) do
            if v:IsA("RemoteEvent") then
                local n = v.Name:lower()
                if (n:find("collect") or n:find("claim") or n:find("event") or n:find("pickup")) then
                    if not (n:find("buy") or n:find("shop") or n:find("purchase") or n:find("coco") or n:find("hat")) then
                        table.insert(RemotesCache, v)
                    end
                end
            end
        end
        RemotesCached = true
    end

    while true do
        pcall(function()
            if getgenv().AutoCollect_Enabled then
                if not RemotesCached then loadRemotes() end 
                
                -- Mudou de Descendants p/ workspace root evitando travar a thread
                for _, item in ipairs(workspace:GetDescendants()) do
                    if not getgenv().AutoCollect_Enabled then break end
                    
                    if tonumber(item.Name) ~= nil and not itensJaColetados[item] then
                        local idItem = tostring(item.Name)
                        
                        for _, v in ipairs(RemotesCache) do
                            pcall(function() v:FireServer("collect", idItem); v:FireServer("claim", idItem) end)
                        end
                        
                        local remoteInterno = item:FindFirstChildWhichIsA("RemoteEvent", true)
                        if remoteInterno then pcall(function() remoteInterno:FireServer() end) end
                        
                        itensJaColetados[item] = true
                        task.wait(0.1) -- Segura loop mas nao cria lag
                    end
                end
            end
        end)
        task.wait(1.5)
    end
end)

-- ==========================================
-- 5. INTERFACE DO USUÁRIO (GUI)
-- ==========================================
local function load_library()
    local ok, libCode = pcall(function() return game:HttpGet("https://raw.githubusercontent.com/GreenDeno/Venyx-UI-Library/main/source.lua") end)
    if not ok then error("NÃ£o foi possÃvel carregar a biblioteca de UI.") end
    
    libCode = libCode:gsub("key%.UserInputType == Enum%.UserInputType%.MouseButton1", "key.UserInputType == Enum.UserInputType.MouseButton1 or key.UserInputType == Enum.UserInputType.Touch")
    libCode = libCode:gsub("input%.UserInputType == Enum%.UserInputType%.MouseButton1", "input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch")
    libCode = libCode:gsub("input%.UserInputType == Enum%.UserInputType%.MouseMovement", "input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch")
    
    libCode = libCode:gsub("section%.container%.Parent%.AbsoluteSize%.Y", "(section.container.Parent.AbsoluteSize.Y / (getgenv().MenuScale and (getgenv().MenuScale/100) or 1))")
    libCode = libCode:gsub("self%.container%.Title%.AbsoluteSize%.Y", "(self.container.Title.AbsoluteSize.Y / (getgenv().MenuScale and (getgenv().MenuScale/100) or 1))")
    libCode = libCode:gsub("module%.AbsoluteSize%.Y", "(module.AbsoluteSize.Y / (getgenv().MenuScale and (getgenv().MenuScale/100) or 1))")
    libCode = libCode:gsub("size > self%.container%.AbsoluteSize%.Y", "size > (self.container.AbsoluteSize.Y / (getgenv().MenuScale and (getgenv().MenuScale/100) or 1))")
    
    return loadstring(libCode)()
end

local Venyx = load_library()
local Window = Venyx.new("X-GEN HUB PRO++")

local CombatPage = Window:addPage("Combate", 5012544693)
local VisualsPage = Window:addPage("Visuais", 5012544693)
local PlayerPage = Window:addPage("Jogador", 5012544693)
local FarmPage = Window:addPage("Farm", 5012544693)
local SettingsPage = Window:addPage("Config", 5012544693)

local CombatSection = CombatPage:addSection("Aimbot & Autoshot")
CombatSection:addToggle("Autoshot (Tiro Auto)", getgenv().Autoshot_Enabled, function(value) getgenv().Autoshot_Enabled = value end)
CombatSection:addToggle("Silent Aim (Tiro MÃ¡gico)", getgenv().SilentAim_Enabled, function(value) getgenv().SilentAim_Enabled = value end)
CombatSection:addToggle("Team Check (Times)", getgenv().TeamCheck_Enabled, function(value) getgenv().TeamCheck_Enabled = value end)
CombatSection:addSlider("Bullet Multiplier", getgenv().Bullet_Multiplier, 1, 10, function(value) getgenv().Bullet_Multiplier = value end)

local HitboxSection = CombatPage:addSection("Expandir Hitbox")
HitboxSection:addToggle("HITBOX (WALL)", getgenv().Hitbox_Enabled, function(value) getgenv().Hitbox_Enabled = value end)
HitboxSection:addSlider("Hitbox Size", getgenv().Hitbox_Size, 2, 30, function(value) getgenv().Hitbox_Size = value end)
HitboxSection:addSlider("TransparÃªncia (x10)", getgenv().Hitbox_Transparency * 10, 0, 10, function(value) getgenv().Hitbox_Transparency = value / 10 end)

local VisualSection = VisualsPage:addSection("ESP Highlight")
VisualSection:addToggle("Ativar ESP (DistÃ¢ncia)", getgenv().ESP_Enabled, function(value) getgenv().ESP_Enabled = value; UpdateESPState() end)
VisualSection:addToggle("Rainbow ESP (Arco-Ãris)", getgenv().RainbowESP_Enabled, function(value) getgenv().RainbowESP_Enabled = value end)

local PlayerSection = PlayerPage:addSection("Movimento & Extras")
PlayerSection:addToggle("GHOST MODE (TOOL)", getgenv().GhostMode_Enabled, function(value) getgenv().GhostMode_Enabled = value; ToggleGhostMode(value) end)
PlayerSection:addToggle("AnimaÃ§Ã£o RÃ¡pida (x200)", getgenv().AnimX200_Enabled, function(value) getgenv().AnimX200_Enabled = value end)
PlayerSection:addSlider("WalkSpeed (Velocidade)", getgenv().WalkSpeed, 16, 100, function(value) getgenv().WalkSpeed = value end)

local FarmSection = FarmPage:addSection("Coleta AutomÃ¡tica")
FarmSection:addToggle("AUTO FARM (OTIMIZADO)", getgenv().AutoCollect_Enabled, function(value) getgenv().AutoCollect_Enabled = value end)

local ConfigSection = SettingsPage:addSection("Menu")
ConfigSection:addColorPicker("Cor de Fundo", getgenv().MenuColor, function(color)
    getgenv().MenuColor = color; getgenv().RainbowTheme_Enabled = false
    pcall(function() Window:setTheme("Background", color) end)
    pcall(function() Window:setTheme("DarkContrast", Color3.new(math.clamp(color.R * 0.8, 0, 1), math.clamp(color.G * 0.8, 0, 1), math.clamp(color.B * 0.8, 0, 1))) end)
    pcall(function() Window:setTheme("LightContrast", Color3.new(math.clamp(color.R * 1.2, 0, 1), math.clamp(color.G * 1.2, 0, 1), math.clamp(color.B * 1.2, 0, 1))) end)
    if _G.ToggleButton then _G.ToggleButton.TextColor3 = getgenv().AccentColor or Color3.fromRGB(0, 255, 0) end
end)
ConfigSection:addColorPicker("Cor do Destaque", getgenv().AccentColor, function(color)
    getgenv().AccentColor = color
    pcall(function() Window:setTheme("Accent", color) end)
    if _G.ToggleButton then _G.ToggleButton.TextColor3 = color end
end)

local scaleList = {"50%", "60%", "70%", "80%", "90%", "100%", "110%", "120%", "130%", "140%", "150%"}
ConfigSection:addDropdown("Tamanho do Menu", scaleList, function(value)
    local num = tonumber(value:match("%d+"))
    if num then
        getgenv().MenuScale = num
        local guis = {}
        for _, v in ipairs(game:GetService("CoreGui"):GetChildren()) do table.insert(guis, v) end
        if LocalPlayer and LocalPlayer:FindFirstChild("PlayerGui") then
            for _, v in ipairs(LocalPlayer.PlayerGui:GetChildren()) do table.insert(guis, v) end
        end

        for _, gui in ipairs(guis) do
            if gui:IsA("ScreenGui") and (gui.Name == "X-GEN HUB PRO++" or gui:FindFirstChild("Main") or gui:FindFirstChild("MainFrame")) then
                local main = gui:FindFirstChild("Main") or gui:FindFirstChild("MainFrame")
                if main and main:IsA("GuiObject") then
                    local uiscale = main:FindFirstChildOfClass("UIScale")
                    if not uiscale then
                        uiscale = Instance.new("UIScale")
                        uiscale.Parent = main
                    end
                    uiscale.Scale = num / 100
                end
            end
        end
        Window:Notify("AVISO", "Tamanho do Menu alterado para " .. value)
    end
end)

ConfigSection:addToggle("Rainbow Theme (HUD)", getgenv().RainbowTheme_Enabled, function(value)
    getgenv().RainbowTheme_Enabled = value
    if not value and _G.ToggleButton then
        _G.ToggleButton.TextColor3 = getgenv().AccentColor or Color3.fromRGB(0, 255, 0)
    end
end)

ConfigSection:addButton("Salvar ConfiguraÃ§Ãµes", function() SaveConfig(); Window:Notify("X-GEN HUB", "Sucesso!") end)

-- OTIMIZAÃÃO: Cache the TopBar to avoid CoreGui scanning in a tight loop
local cachedTopBar = nil
local ThemeRainbowTask = task.spawn(function()
    while true do
        pcall(function()
            if getgenv().RainbowTheme_Enabled then
                if not cachedTopBar or not cachedTopBar.Parent then
                    for _, gui in pairs(game:GetService("CoreGui"):GetChildren()) do
                        if gui:IsA("ScreenGui") and (gui.Name == "X-GEN HUB PRO++" or gui:FindFirstChild("Main") or gui:FindFirstChild("MainFrame")) then
                            local main = gui:FindFirstChild("Main") or gui:FindFirstChild("MainFrame")
                            if main and main:IsA("Frame") and main:FindFirstChild("TopBar") then
                                cachedTopBar = main.TopBar
                            end
                        end
                    end
                end

                local rainbowColor = Color3.fromHSV((tick() % 5) / 5, 1, 1)
                if cachedTopBar then cachedTopBar.BackgroundColor3 = rainbowColor end
                if _G.ToggleButton then _G.ToggleButton.TextColor3 = rainbowColor end
            end
        end)
        task.wait(0.2) -- Otimizado: Nao precisa ser 0.03
    end
end)

ConfigSection:addButton("Fechar Script (Unload)", function()
    if ThemeRainbowTask then task.cancel(ThemeRainbowTask) end
    if AnimX200Task then task.cancel(AnimX200Task) end
    if AutoCollectTask then task.cancel(AutoCollectTask) end
    if WalkSpeedTask then task.cancel(WalkSpeedTask) end
    if RSConnection then RSConnection:Disconnect() end
    ToggleGhostMode(false)
    for _, conn in pairs(PlayersConnections) do
        if conn then conn:Disconnect() end
    end
    for _, player in ipairs(GetPlayers(Players)) do
        if player.Character then
            local highlight = player.Character:FindFirstChild("XGEN_ESP_HIGHLIGHT")
            if highlight then highlight:Destroy() end
            local hrp = player.Character:FindFirstChild("HumanoidRootPart")
            if hrp and OriginalHitboxes[player] then hrp.Size = OriginalHitboxes[player] end
        end
    end
    if _G.ToggleGui then _G.ToggleGui:Destroy() end
    for _, gui in ipairs(game:GetService("CoreGui"):GetChildren()) do
        if gui:FindFirstChild("Main") and gui.Main:FindFirstChild("TopBar") then gui:Destroy() end
    end
end)

ConfigSection:addKeybind("Atalho: Pressione M para ocultar", Enum.KeyCode.M, function()
    Window:toggle()
    _G.IsWindowOpen = not _G.IsWindowOpen
    if _G.ToggleButton then _G.ToggleButton.Text = _G.IsWindowOpen and "â³" or "â½" end
end)

pcall(function() Window:setTheme("Background", getgenv().MenuColor) end)
pcall(function() Window:setTheme("DarkContrast", Color3.new(math.clamp(getgenv().MenuColor.R * 0.8, 0, 1), math.clamp(getgenv().MenuColor.G * 0.8, 0, 1), math.clamp(getgenv().MenuColor.B * 0.8, 0, 1))) end)
pcall(function() Window:setTheme("LightContrast", Color3.new(math.clamp(getgenv().MenuColor.R * 1.2, 0, 1), math.clamp(getgenv().MenuColor.G * 1.2, 0, 1), math.clamp(getgenv().MenuColor.B * 1.2, 0, 1))) end)
pcall(function() Window:setTheme("Accent", getgenv().AccentColor) end)

Window:SelectPage(Window.pages[1], true)

task.spawn(function()
    task.wait(1.5)
    pcall(function()
        local guis = {}
        for _, v in ipairs(game:GetService("CoreGui"):GetChildren()) do table.insert(guis, v) end
        if LocalPlayer and LocalPlayer:FindFirstChild("PlayerGui") then
            for _, v in ipairs(LocalPlayer.PlayerGui:GetChildren()) do table.insert(guis, v) end
        end

        for _, gui in ipairs(guis) do
            if gui:IsA("ScreenGui") and (gui.Name == "X-GEN HUB PRO++" or gui:FindFirstChild("Main") or gui:FindFirstChild("MainFrame")) then
                local main = gui:FindFirstChild("Main") or gui:FindFirstChild("MainFrame")
                if main and main:IsA("GuiObject") then
                    main.Active = true
                    if getgenv().MenuScale and tonumber(getgenv().MenuScale) and tonumber(getgenv().MenuScale) ~= 100 then
                        local uiscale = main:FindFirstChildOfClass("UIScale")
                        if not uiscale then
                            uiscale = Instance.new("UIScale")
                            uiscale.Parent = main
                        end
                        uiscale.Scale = tonumber(getgenv().MenuScale) / 100
                    end
                end
            end
        end
    end)
end)

local ToggleGui = Instance.new("ScreenGui")
ToggleGui.Name = "XGenToggleUI"
ToggleGui.ResetOnSpawn = false
pcall(function() ToggleGui.Parent = game:GetService("CoreGui") end)
if not ToggleGui.Parent then ToggleGui.Parent = LocalPlayer:WaitForChild("PlayerGui") end
_G.ToggleGui = ToggleGui

local ToggleButton = Instance.new("TextButton")
ToggleButton.Parent = ToggleGui
ToggleButton.BackgroundTransparency = 1 
ToggleButton.Position = UDim2.new(1, -15, 0, 5) 
ToggleButton.AnchorPoint = Vector2.new(1, 0)
ToggleButton.Size = UDim2.new(0, 30, 0, 30)
ToggleButton.Font = Enum.Font.SourceSansBold
ToggleButton.Text = "â³"
ToggleButton.TextColor3 = getgenv().AccentColor or Color3.fromRGB(0, 255, 0)
ToggleButton.TextSize = 25
ToggleButton.BorderSizePixel = 0
_G.ToggleButton = ToggleButton
_G.IsWindowOpen = true

local dragging = false
local dragStart, startPos, dragInput

ToggleButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = ToggleButton.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)

ToggleButton.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
end)

UIS.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        ToggleButton.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

ToggleButton.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        if dragStart and (input.Position - dragStart).Magnitude < 10 then
            Window:toggle()
            _G.IsWindowOpen = not _G.IsWindowOpen
            ToggleButton.Text = _G.IsWindowOpen and "â³" or "â½"
        end
    end
end)

task.spawn(function()
    task.wait(0.5)
    Window:toggle()
    _G.IsWindowOpen = false
    ToggleButton.Text = "â½"
end)

print("X-GEN HUB (V13 Optimized) CARREGADO COM SUCESSO!")
