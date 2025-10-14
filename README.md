-- Serviços
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualUser = game:GetService("VirtualUser")
local JsFramework = require(ReplicatedStorage:WaitForChild("JsFramework"))

local LocalPlayer = Players.LocalPlayer

-- =========================
-- Flags gerais
-- =========================
local autoFarmOn = false
local autoTrialOn = false
local autoFunc25On = false
local autoAfkOn = false

local trialEnteredOnce = false
local lastTrialTrigger = 0
local savedToggleStates = nil


-- =========================
-- Utilidades
-- =========================
local function safeTeleportTo(cf)
    local char = LocalPlayer.Character
    if not char then return false end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return false end
    hrp.CFrame = cf
    RunService.Heartbeat:Wait()
    return true
end

local function normalize(s)
    s = tostring(s or "")
    s = s:lower():gsub("%s+", " ")
    return s
end
local function saveToggleStates()
    savedToggleStates = {
        autoFarmOn = autoFarmOn,
        autoFunc25On = autoFunc25On,
        NpcTp_Enabled = NpcTp_Enabled 
    }
    print("[Trial] Estados salvos:", savedToggleStates)
end

local function restoreToggleStates()
    if not savedToggleStates then return end

    autoFarmOn = savedToggleStates.autoFarmOn
    NpcTp_Enabled = savedToggleStates.NpcTp_Enabled
    autoFunc25On = savedToggleStates.autoFunc25On

    print("[Trial] Estados restaurados.")
end

local function monitorTrialResults()
    task.spawn(function()
        local gui = Players.LocalPlayer:WaitForChild("PlayerGui")
        local resultsFrame = gui:WaitForChild("_Modes"):WaitForChild("Frame"):WaitForChild("Trial"):WaitForChild("Results")

        while true do
            if resultsFrame.Visible then
                print("[Trial] Interface de recompensas visível. Dungeon finalizada.")
                restoreToggleStates()
                break
            end
            task.wait(1)
        end
    end)
end


-- =========================
-- Auto Farm NPCs
-- =========================
local npcRoot = Workspace:WaitForChild("__debris"):WaitForChild("__npcs"):WaitForChild("__client")
local idFolders = {"ID_1","ID_2","ID_3","ID_4","ID_5"}
local approachOffset = Vector3.new(0, 3, 3)
local delayBetweenTPs = 0.4

local function gatherTargets()
    local out = {}
    for _, folderName in ipairs(idFolders) do
        local folder = npcRoot:FindFirstChild(folderName)
        if folder then
            for _, npc in ipairs(folder:GetChildren()) do
                if npc:IsA("Model") then
                    local humanoid = npc:FindFirstChildOfClass("Humanoid")
                    local hrp = npc:FindFirstChild("HumanoidRootPart") or npc.PrimaryPart
                    local bar = npc:FindFirstChild("EnemyHealthBar")
                    local title = bar and bar:FindFirstChild("Title")
                    if humanoid and hrp and title and title:IsA("TextLabel") then
                        table.insert(out, {hrp = hrp, name = title.Text})
                    end
                end
            end
        end
    end
    return out
end

local function autoFarmLoop()
    while autoFarmOn do
        local npcs = gatherTargets()
        for _, entry in ipairs(npcs) do
            if not autoFarmOn then break end
            if entry.hrp and entry.hrp.Parent then
                safeTeleportTo(entry.hrp.CFrame * CFrame.new(approachOffset))
                task.wait(delayBetweenTPs)
            end
        end
        task.wait(0.2)
    end
end

local function findTrialTimer(trialType)
    local portals = Workspace:FindFirstChild("__Portals") or Workspace:FindFirstChild("_Portals")
    local portal = portals and (portals:FindFirstChild(trialType) or portals:FindFirstChild(trialType:lower()))
    local hud = portal and (portal:FindFirstChild("Hud") or portal:FindFirstChild("HUD"))
    local timer = hud and (hud:FindFirstChild("Timer") or hud:FindFirstChild("timer"))
    if timer and timer:IsA("TextLabel") then return timer end

    -- fallback genérico
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("TextLabel") then
            local t = normalize(obj.Text)
            if t:find("opening !") or t:find("opening!") then return obj end
        end
    end
    return nil
end


-- =========================
-- === NOVO: Monitor RoomLabel e executar Function #4
-- =========================
-- Variáveis de controle adicionadas:
local trialEnterTimestamp = nil
local roomWatcherRunning = false
local roomWatcherPaused = false -- quando true, watcher não inicia até nova trial

-- Função utilitária: tenta extrair número de um TextLabel (retorna número ou nil)
local function extractRoomNumberFromText(txt)
    if not txt then return nil end
    local n = tonumber(txt)
    if n then return n end
    local m = txt:match("(%d+)")
    if m then return tonumber(m) end
    return nil
end

-- Loop que verifica a RoomLabel a cada 20s depois de 60s da entrada na trial
local function startRoomWatcher()
    if roomWatcherRunning then return end
    if roomWatcherPaused then
        print("[RoomWatcher] Não vai iniciar porque está pausado até a próxima trial")
        return
    end
    roomWatcherRunning = true
    task.spawn(function()
        local elapsed = 0
        while elapsed < 60 and roomWatcherRunning do
            task.wait(1)
            elapsed += 1
        end
        if not roomWatcherRunning then return end

        local lastRoomNum = nil

        while roomWatcherRunning do
            local ok, roomLabel = pcall(function()
                local gui = Players.LocalPlayer:FindFirstChild("PlayerGui")
                if not gui then return nil end
                local modes = gui:FindFirstChild("_Modes")
                if not modes then return nil end
                local frame = modes:FindFirstChild("Frame")
                if not frame then return nil end
                local trial = frame:FindFirstChild("Trial")
                if not trial then return nil end
                local status = trial:FindFirstChild("Status")
                if not status then return nil end
                local label = status:FindFirstChild("RoomLabel")
                return label
            end)

            if ok and roomLabel and roomLabel:IsA("TextLabel") then
                local txt = roomLabel.Text
                local currentNum = extractRoomNumberFromText(txt)
                print(("[RoomWatcher] Room atual: %s (extraído: %s)"):format(txt, tostring(currentNum)))

                if lastRoomNum ~= nil and currentNum ~= nil and currentNum == lastRoomNum then
                    print("[RoomWatcher] Sala travada. Executando Auto Exit.")
                    pcall(function() JsFramework.Network:FireServer("Dungeon_RequestLeave") end)
                    restoreToggleStates()
                    roomWatcherRunning = false
                    return
                end

                if currentNum ~= nil then
                    lastRoomNum = currentNum
                end
            else
                warn("[RoomWatcher] Não encontrou RoomLabel no caminho esperado")
            end

            task.wait(20)
        end

        roomWatcherRunning = false
    end)
end

local function stopRoomWatcher()
    roomWatcherRunning = false
end

-- Flags separadas para cada trial
local autoTrialEasyOn = false
local autoTrialMediumOn = false

-- Função principal para cada tipo de trial
local function autoTrialLoop(trialType)
    task.spawn(function()
        while (trialType == "easy" and autoTrialEasyOn) or (trialType == "medium" and autoTrialMediumOn) do
            local timer = findTrialTimer(trialType)
            if timer then
                local txt = normalize(timer.Text)
                local isOpening = txt:find("opening !") or txt:find("opening!")
                if (isOpening) and (tick() - lastTrialTrigger > 3) then
                    lastTrialTrigger = tick()
                    if not trialEnteredOnce then
                        local ok, err = pcall(function()
                            JsFramework.Network:FireServer("Dungeon_RequestJoin", trialType)
                        end)

                        if ok then
                            print("[AutoTrial] Entrou na trial:", trialType)
                            trialEnteredOnce = true
                            trialEnterTimestamp = tick()
                            roomWatcherPaused = false
                            saveToggleStates()
                            monitorTrialResults()
                            startRoomWatcher()

                            -- Pausa Auto Farm por 10s e reativa depois de 60s
                            autoFarmOn = false
                            NpcTp_Enabled = false
                            print("[AutoTrial] Auto Farm desativado por 10s.")
                            task.delay(60, function()
                                autoFarmOn = true
                                print("[AutoTrial] Auto Farm reativado.")
                                task.spawn(autoFarmLoop)
                            end)

                            -- Auto Exit se RoomLabel não mudar
                            task.delay(60, function()
                                local gui = Players.LocalPlayer:FindFirstChild("PlayerGui")
                                local label = gui and gui:FindFirstChild("_Modes")
                                    and gui._Modes:FindFirstChild("Frame")
                                    and gui._Modes.Frame:FindFirstChild("Trial")
                                    and gui._Modes.Frame.Trial:FindFirstChild("Status")
                                    and gui._Modes.Frame.Trial.Status:FindFirstChild("RoomLabel")

                                if label and label:IsA("TextLabel") then
                                    local initialRoom = label.Text
                                    task.wait(20)
                                    if label.Text == initialRoom then
                                        print("[AutoTrial] Sala não mudou em 20s. Saindo da trial.")
                                        pcall(function() JsFramework.Network:FireServer("Dungeon_RequestLeave") end)
                                    end
                                end
                            end)

                            task.delay(50, function()
                                trialEnteredOnce = false
                                print("[AutoTrial] Resetado após 50s.")
                            end)
                        else
                            warn("[AutoTrial] Falha ao entrar na trial:", err)
                        end
                    end
                end
            end
            task.wait(0.3)
        end
        stopRoomWatcher()
    end)
end


-- =========================
-- Loop da Function 31 (InvokeServer no índice ajustável)
-- =========================
local function autoFunc25Loop()
    local funcs = ReplicatedStorage:WaitForChild("Communication"):WaitForChild("Functions")
    local remote = funcs:GetChildren()[18]
    if not remote then
        warn("[Func31] Remote 404.")
        return
    end

    local args = { false, "normal" }

    while autoFunc25On do
        local sucesso, retorno = pcall(function()
            return remote:InvokeServer(unpack(args))
        end)

        if not sucesso then
            warn("[Func31] Error call InvokeServer:", retorno)
        end

        task.wait(5)
    end
end

        -- =========================
        -- Anti-AFK (VirtualUser + nudge camera)
        -- =========================
        local afkIdledConn = nil

        local function nudgeCamera()
            local cam = workspace.CurrentCamera
            if not cam then return end
            VirtualUser:Button2Down(Vector2.new(0,0), cam.CFrame)
            task.wait(0.12 + math.random() * 0.15)
            VirtualUser:Button2Up(Vector2.new(0,0), cam.CFrame)
        end

        local function virtualClick()
            local cam = workspace.CurrentCamera
            if not cam then return end
            VirtualUser:Button2Down(Vector2.new(0,0), cam.CFrame)
            task.wait(0.85 + math.random() * 0.6)
            VirtualUser:Button2Up(Vector2.new(0,0), cam.CFrame)
        end

        local function enableAfk()
            if afkIdledConn then return end
            afkIdledConn = LocalPlayer.Idled:Connect(function()
                if math.random() < 0.6 then
                    virtualClick()
                else
                    nudgeCamera()
                end
            end)
            print("🟢 [AFK] ON")
        end

        local function disableAfk()
            if afkIdledConn then
                afkIdledConn:Disconnect()
                afkIdledConn = nil
            end
            print("🔴 [AFK] Off")
        end
        local VirtualUser = game:GetService("VirtualUser")

        local function screenClick()
            -- Simula clique na posição (500, 500) da tela
            VirtualUser:Button1Down(Vector2.new(500, 500))
            task.wait(0.1)
            VirtualUser:Button1Up(Vector2.new(500, 500))
            print("[AntiAFK] Clique na tela simulado")
        end

        -- Loop automático
        task.spawn(function()
            while true do
                task.wait(60) -- a cada 60 segundos
                screenClick()
            end
end)

-- =========================
-- AutoCall Function 21 (Dropdown + Toggle)
-- =========================
local function safeFmt(fmt, ...)
    local args = {...}
    for i = 1, #args do args[i] = tostring(args[i]) end
    return string.format(fmt, unpack(args))
end

local FUNCTION_INDEX = 21
local AUTOCALL_DELAY = 1
local AUTOCALL_MAX_CALLS = math.huge

local orderedNames = {
    "Ninja Rank",
    "Six Path's",
    "Vital Energy",
    "Kaioken Token",
    "Shadow Rank",
    "Vocation",
    "Vessel Crystal",
    "Respiration Token",
    "Elemental Mark",
     "Heart Gears"
}
local gachaMap = {
    ["Ninja Rank"] = "W1_1",
    ["Six Path's"] = "W1_2",
    ["Vital Energy"] = "W2_1",
    ["Kaioken Token"] = "W2_2",
    ["Shadow Rank"] = "W3_1",
    ["Vocation"] = "W3_2",
    ["Vessel Crystal"] = "W3_3",
    ["Respiration Token"] = "W4_1",
    ["Elemental Mark"] = "W4_2",
    ["Heart Gears"] = "W5_1"
}
local selectedName = orderedNames[1]
local selectedArg = gachaMap[selectedName]
local currentRunId = nil

local function getFunctionsFolder()
    local comm = ReplicatedStorage:FindFirstChild("Communication")
    return comm and comm:FindFirstChild("Functions")
end

local function getRemoteByIndex(idx)
    local f = getFunctionsFolder()
    return f and f:GetChildren()[idx]
end

local function callRemoteChild(child, args)
    if not child then return false, "child nil" end
    if child:IsA("RemoteFunction") then
        return pcall(function() return child:InvokeServer(unpack(args)) end)
    elseif child:IsA("RemoteEvent") then
        local ok, res = pcall(function() child:FireServer(unpack(args)) end)
        return ok, (ok and "fired" or res)
    end
    return false, "not remote"
end

local function autoCallLoop(runId)
    local child = getRemoteByIndex(FUNCTION_INDEX)
    if not child then
        warn("[AutoCallUI] Remote 404")
        currentRunId = nil
        return
    end
    local calls = 0
    while currentRunId == runId and calls < AUTOCALL_MAX_CALLS do
        if not selectedArg then break end
        local ok, res = callRemoteChild(child, {selectedArg})
        calls = calls + 1
        task.wait(AUTOCALL_DELAY)
    end
    currentRunId = nil
end

local function startAutoCall()
    if currentRunId then currentRunId = nil; task.wait(0.1) end
    local id = tostring(math.random(1,1e9)) .. "-" .. tostring(tick())
    currentRunId = id
    task.spawn(function() autoCallLoop(id) end)
end

local function stopAutoCall()
    currentRunId = nil
end

-- === Auto Spin Player Passives (Fluent Toggle) ===
local autoPlayerPassiveSpinOn = false
local autoPlayerPassiveThread
local PLAYER_SPIN_INTERVAL = 0.4 -- intervalo entre spins (mesmo padrão do script original)

local function startAutoPlayerPassiveSpin()
    if autoPlayerPassiveSpinOn then return end
    autoPlayerPassiveSpinOn = true
    autoPlayerPassiveThread = task.spawn(function()
        while autoPlayerPassiveSpinOn do
            local ok, res = pcall(function()
                return JsFramework.Network:InvokeServer("roll_p_passive")
            end)

            if ok then
                -- opcional: mostra notificação visual via JsFramework
                JsFramework.Signal.Fire("CustomNotification", "Rolled passive", {
                    BackColor = Color3.fromRGB(128, 255, 106),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Timer = 1
                })
            else
                warn("[AutoSpinPlayer] Falha ao chamar roll_p_passive:", tostring(res))
                JsFramework.Signal.Fire("CustomNotification", "Spin failed", {
                    BackColor = Color3.fromRGB(255, 108, 108),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Timer = 2
                })
            end

            task.wait(PLAYER_SPIN_INTERVAL)
        end
    end)
    print("[AutoSpinPlayer] Iniciado")
end

local function stopAutoPlayerPassiveSpin()
    autoPlayerPassiveSpinOn = false
    print("[AutoSpinPlayer] Parado")
end

-- =========================
-- Fluent UI (com Settings)
-- =========================
local FluentLoaderOk, Fluent = pcall(function()
    return loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
end)
if not FluentLoaderOk or not Fluent then
    warn("[Fluent] Unable to load Fluent module. Check your runner./HttpGet.")
else
    local Window = Fluent:CreateWindow({
        Title = "- Anime Elements Script -",
        SubTitle = "By Secret",
        TabWidth = 160,
        Size = UDim2.fromOffset(620, 460),
        Acrylic = true,
        Theme = "Dark",
        MinimizeKey = Enum.KeyCode.LeftControl
    })

    local Tabs = {
        Main = Window:AddTab({ Title = "Main", Icon = "hammer" }),
        Avatar = Window:AddTab({ Title = "Avatar", Icon = "users" }),
        Passives = Window:AddTab({ Title = "Passives", Icon = "swords" }),
        Gachas = Window:AddTab({ Title = "Gachas", Icon = "aperture" }),
        Respiration = Window:AddTab({ Title = "Respiration", Icon = "wind" }),
        Settings = Window:AddTab({ Title = "Settings", Icon = "settings" })
    }

    -- Toggles principais
    Tabs.Main:AddToggle("AutoFarm", {
        Title = "Auto Farm NPCs",
        Description = "Teleport to all nearby NPCs",
        Default = false,
        Callback = function(state)
            autoFarmOn = state
            if autoFarmOn then
                print("🟢 Auto Farm ON")
                task.spawn(autoFarmLoop)
            else
                print("🔴 Auto Farm OFF")
            end
        end
    })

    Tabs.Main:AddToggle("AutoTrialEasy", {
        Title = "Auto Trial Easy",
        Description = "Entra automaticamente na trial Easy",
        Default = false,
        Callback = function(state)
            autoTrialEasyOn = state
            if state then
                print("🟢 Auto Trial Easy ON")
                trialEnteredOnce = false
                lastTrialTrigger = 0
                autoTrialLoop("easy")
            else
                print("🔴 Auto Trial Easy OFF")
            end
        end
    })

    Tabs.Main:AddToggle("AutoTrialMedium", {
        Title = "Auto Trial Medium",
        Description = "Entra automaticamente na trial Medium",
        Default = false,
        Callback = function(state)
            autoTrialMediumOn = state
            if state then
                print("🟢 Auto Trial Medium ON")
                trialEnteredOnce = false
                lastTrialTrigger = 0
                autoTrialLoop("medium")
            else
                print("🔴 Auto Trial Medium OFF")
            end
        end
    })

    Tabs.Avatar:AddToggle("SpinAvatar", {
        Title = "Auto Spin Avatar",
        Description = "Spin Avatar Tokens",
        Default = false,
        Callback = function(state)
            autoFunc25On = state
            if autoFunc25On then
                print("🟢 Auto Spin Avatar ON")
                task.spawn(autoFunc25Loop)
            else
                print("🔴 Auto Spin Avatar OFF")
            end
        end
    })

    -- Dropdown + Toggle AutoCall
    Tabs.Gachas:AddDropdown("GachaSelect", {
        Title = "Select Gacha ",
        Description = "choose the gacha to spin",
        Values = orderedNames,
        Multi = false,
        Default = 1,
        Callback = function(value)
            selectedName = value
            selectedArg = gachaMap[value]
            print("[UI] Selected:", selectedName, "->", selectedArg)
        end
    })

    Tabs.Gachas:AddToggle("AutoCallToggle", {
        Title = "Auto Spin Gacha Selected",
        Description = "Active for Auto Spin",
        Default = false,
        Callback = function(state)
            if state then
                startAutoCall()
                print("🟢[UI] AutoGacha ON:", selectedName, "->", selectedArg)
            else
                stopAutoCall()
                print("🔴[UI] AutoGacha OFF")
            end
        end
    })

    -- NOVO: Dropdown + Botão Function 30 (nomes amigáveis → PIDs)
    local pidOptions = { "Arcanum", "Annihilation", "Destiny" }
    local pidMap = {
        Arcanum = "PID_13",
        Annihilation = "PID_14",
        Destiny = "PID_15",
    }
    local selectedPIDName = pidOptions[1]
    local selectedPID = pidMap[selectedPIDName]

    Tabs.Passives:AddDropdown("PIDSelect", {
        Title = "Select loadout",
        Description = "Chose passives for equip (Arcanum/Annihilation/Destiny)",
        Values = pidOptions,
        Multi = false,
        Default = 1,
        Callback = function(value)
            selectedPIDName = value
            selectedPID = pidMap[value]
            print(string.format("[UI] Selected: %s -> %s", selectedPIDName, selectedPID))
        end
    })

    -- Botão único para chamar Functions[31] com selectedPID (sem loop)
    Tabs.Passives:AddButton({
        Title = "Execute Function 31",
        Description = "Chama Functions[31] uma vez com o PID selecionado",
        Callback = function()
            local comm = ReplicatedStorage:FindFirstChild("Communication")
            local funcs = comm and comm:FindFirstChild("Functions")
            if not funcs then
                warn("[Func31 Button] Communication/Functions não encontrada")
                return
            end

            local children = funcs:GetChildren()
            local remote = children[31]
            if not remote then
                warn(string.format("[Func31 Button] Nenhum child no índice 31 (total %d)", #children))
                return
            end

            local ok, res = pcall(function()
                if remote:IsA("RemoteFunction") then
                    return remote:InvokeServer(selectedPID)
                elseif remote:IsA("RemoteEvent") then
                    remote:FireServer(selectedPID)
                    return true
                else
                    error("Objeto no índice 31 não é RemoteFunction nem RemoteEvent")
                end
            end)

            if ok then
                print(string.format("[Func31 Button] Chamado %s com %s. Retorno: %s",
                    tostring(remote.Name), tostring(selectedPID), tostring(res)))
            else
                warn(string.format("[Func31 Button] Erro ao chamar %s: %s",
                    tostring(remote.Name or "nil"), tostring(res)))
            end
        end
    })

    -- NOVO: Dropdown + Botão Function 17 (nomes amigáveis → IDs)
    local f17Options = {
        "Jyraua 30x Watter",
        "Etanor 25x Fire",
        "Satoru 20x Wind",
        "Esran 9x Earth",
        "Tremor 8x Watter",
        "Rengoru 6x",
        "Tantsumiki 7x Wind",
        "Gann 3x Eath",
        "Emirya 4x Water",
        "Sorro 2x Wind",
        "Akanu 1.5x Fire"
    }
    local f17IdMap = {
        ["Jyraua 30x Watter"] = "Jiraya",
        ["Etanor 25x Fire"] = "ID_9",
        ["Satoru 20x Wind"] = "ID_10",
        ["Esran 9x Earth"] = "ID_8",
        ["Tremor 8x Watter"] = "ID_7",
        ["Rengoru 6x"] = "ID_6",
        ["Tantsumiki 7x Wind"] = "ID_5",
        ["Gann 3x Eath"] = "ID_3",
        ["Emirya 4x Water"] = "ID_4",
        ["Sorro 2x Wind"] = "ID_2",
        ["Akanu 1.5x Fire"] = "ID_1"
    }
    local selectedF17Name = f17Options[1]
    local selectedF17ID = f17IdMap[selectedF17Name]

    Tabs.Avatar:AddDropdown("Func17IDSelect", {
        Title = "Select Your Avatar For Change",
        Description = "Change Your Avatar",
        Values = f17Options,
        Multi = false,
        Default = 1,
        Callback = function(value)
            selectedF17Name = value
            selectedF17ID = f17IdMap[value]
            print(string.format("[Func17] Selected: %s -> %s", selectedF17Name, selectedF17ID))
        end
    })

    Tabs.Avatar:AddButton({
        Title = "Change ur Avatar",
        Description = "Click for change ur avatar",
        Callback = function()
            local comm = ReplicatedStorage:WaitForChild("Communication")
            local funcs = comm:WaitForChild("Functions")
            local child = funcs:GetChildren()[17] -- índice 17

            if not child then
                warn("[Func17] Remote 404.")
                return
            end

            local ok, res = pcall(function()
                if child:IsA("RemoteFunction") then
                    return child:InvokeServer(selectedF17ID)
                elseif child:IsA("RemoteEvent") then
                    child:FireServer(selectedF17ID)
                    return true
                else
                    error("Objeto no índice 17 não é RemoteFunction nem RemoteEvent")
                end
            end)

            if ok then
                print(string.format("[Func17] Executed: %s (%s). Return: %s", selectedF17Name, selectedF17ID, tostring(res)))
            else
                warn("[Func17] Error:", res)
            end
        end
    })

    Tabs.Passives:AddToggle("AutoSpinPlayerPassives", {
        Title = "Auto Spin Player Passives",
        Description = "Liga/desliga auto-spin das passivas do Player",
        Default = false,
        Callback = function(state)
            if state then
                if autoPlayerPassiveSpinOn then return end
                startAutoPlayerPassiveSpin()
                print("🟢 Auto Spin Player ON")
            else
                stopAutoPlayerPassiveSpin()
                print("🔴 Auto Spin Player OFF")
            end
        end
    })

    -- NOVO: Dropdown + Botão Function 22 (nomes amigáveis → args duplos)
    local f22Options = {
        "Sun 10x Fire",
        "Love 5x Earth",
        "Mist 5x Watter",
        "Thunder 3x Wind",
        "Fire 2x Fire",
    }
    local f22ArgsMap = {
        ["Sun 10x Fire"]   = { "W4_1", "ID_8" },
        ["Love 5x Earth"]   = { "W4_1", "ID_7" },
        ["Mist 5x Watter"]  = { "W4_1", "ID_6" },
        ["Thunder 3x Wind"] = { "W4_1", "ID_5" },
        ["Fire 2x Fire"]    = { "W4_1", "ID_4" },
    }
    local selectedF22Name = f22Options[1]
    local selectedF22Args = f22ArgsMap[selectedF22Name]

    Tabs.Respiration:AddDropdown("Func22Select", {
        Title = "Select Your Respiration",
        Description = "Chose ur respiration for change",
        Values = f22Options,
        Multi = false,
        Default = 1,
        Callback = function(value)
            selectedF22Name = value
            selectedF22Args = f22ArgsMap[value]
            print(string.format("[Func22] Selected: %s -> %s, %s", selectedF22Name, tostring(selectedF22Args[1]), tostring(selectedF22Args[2])))
        end
    })

    Tabs.Respiration:AddButton({
        Title = "Change Your Respiration",
        Description = "Active change your respiration",
        Callback = function()
            local funcs = ReplicatedStorage:WaitForChild("Communication"):WaitForChild("Functions")
            local remote = funcs:GetChildren()[22] -- índice 22
            if not remote then
                warn("[Func22] Remote 404")
                return
            end
            local ok, res = pcall(function()
                if remote:IsA("RemoteFunction") then
                    return remote:InvokeServer(unpack(selectedF22Args))
                elseif remote:IsA("RemoteEvent") then
                    remote:FireServer(unpack(selectedF22Args))
                    return true
                else
                    error("Error...")
                end
            end)
            if ok then
                print(string.format("[Func22] Executed: %s -> %s, %s. Return: %s",
                    selectedF22Name, tostring(selectedF22Args[1]), tostring(selectedF22Args[2]), tostring(res)))
            else
                warn("[Func22] Error:", res)
            end
        end
    })

    -- === NPC Teleport (adicionado ao mesmo Hub, Tabs.Main) ===
    -- Função para obter nomes de NPCs vivos
    local function NpcTp_GetNpcNames()
        local names = {}
        local success, clientFolder = pcall(function()
            return workspace.__debris.__npcs.__client
        end)
        if not success or not clientFolder then return names end

        for _, worldFolder in ipairs(clientFolder:GetChildren()) do
            for _, npcFolder in ipairs(worldFolder:GetChildren()) do
                local hb = npcFolder:FindFirstChild("EnemyHealthBar")
                if hb and hb:FindFirstChild("Title") and hb.Title:IsA("TextLabel") then
                    local n = hb.Title.Text
                    if n and not table.find(names, n) then table.insert(names, n) end
                end
            end
        end
        return names
    end

    local NpcTp_Selected = {}
    local NpcTp_Multi = Tabs.Main:AddDropdown("NpcTp_Dropdown", {
        Title = "NPC Teleport",
        Description = "Selecione NPCs para teleportar",
        Values = NpcTp_GetNpcNames(),
        Multi = true,
        Default = {}
    })

    NpcTp_Multi:OnChanged(function(value)
        NpcTp_Selected = {}
        for name, state in pairs(value) do
            if state then table.insert(NpcTp_Selected, name) end
        end
    end)

    Tabs.Main:AddButton({
        Title = "🔄 Refresh NPCs",
        Description = "Atualiza lista de NPCs vivos",
        Callback = function()
            local newList = NpcTp_GetNpcNames()
            NpcTp_Multi:SetValues(newList)
            Fluent:Notify({ Title = "NPC Teleport", Content = "Lista atualizada", Duration = 3 })
        end
    })

    local NpcTp_Enabled = false
    local NpcTp_Toggle = Tabs.Main:AddToggle("NpcTp_Toggle", {
        Title = "Auto TP NPCs",
        Description = "Liga/desliga TP sequencial nos NPCs selecionados",
        Default = false
    })

    local function NpcTp_FindByName(name)
        local ok, clientFolder = pcall(function() return workspace.__debris.__npcs.__client end)
        if not ok or not clientFolder then return nil end
        for _, worldFolder in ipairs(clientFolder:GetChildren()) do
            for _, npcFolder in ipairs(worldFolder:GetChildren()) do
                local hb = npcFolder:FindFirstChild("EnemyHealthBar")
                if hb and hb:FindFirstChild("Title") and hb.Title:IsA("TextLabel") then
                    if hb.Title.Text == name then return npcFolder end
                end
            end
        end
        return nil
    end

    -- Sequência de teleporte corrigida para NPC Teleport
    local function NpcTp_Sequence()
        local PlayersSvc = game:GetService("Players")
        local LocalP = PlayersSvc.LocalPlayer

        NpcTp_Selected = NpcTp_Selected or {}

        while NpcTp_Enabled and #NpcTp_Selected > 0 do
            for _, name in ipairs(NpcTp_Selected) do
                if not NpcTp_Enabled then break end

                local npcFolder = NpcTp_FindByName(name)
                if not (npcFolder and npcFolder.Parent) then
                    task.wait(0.6)
                    continue
                end

                local char = LocalP.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local target = npcFolder:FindFirstChild("HumanoidRootPart") or npcFolder.PrimaryPart
                    if target then
                        pcall(function() hrp.CFrame = target.CFrame + Vector3.new(0, 5, 0) end)
                    end
                end

                -- espera até o NPC morrer (vida zerar) ou sumir
                local hb = npcFolder:FindFirstChild("EnemyHealthBar")
                if hb then
                    local healthValue = hb:FindFirstChild("Health")
                    if healthValue and healthValue:IsA("NumberValue") then
                        repeat
                            if not NpcTp_Enabled or not npcFolder.Parent then break end
                            local ok = pcall(function()
                                if healthValue.GetPropertyChangedSignal then
                                    healthValue:GetPropertyChangedSignal("Value"):Wait()
                                else
                                    healthValue.Changed:Wait()
                                end
                            end)
                            if not ok then task.wait(0.5) end
                        until (not healthValue.Parent) or healthValue.Value <= 0 or not NpcTp_Enabled or not npcFolder.Parent
                    else
                        local watchdog = 0
                        while npcFolder.Parent and NpcTp_Enabled do
                            local okHb = npcFolder:FindFirstChild("EnemyHealthBar")
                            if not okHb or not okHb:FindFirstChild("Title") then break end
                            task.wait(0.3)
                            watchdog = watchdog + 1
                            if watchdog > 100 then break end
                        end
                    end
                else
                    task.wait(0.7)
                end

                task.wait(0.2)
            end

            task.wait(0.2)
        end
    end

    -- conecta toggle (assegure-se de que NpcTp_Toggle já foi criado)
    NpcTp_Toggle:OnChanged(function(state)
        NpcTp_Enabled = state
        NpcTp_Toggle:SetTitle("Auto TP [" .. (state and "ON" or "OFF") .. "]")
        if state then
            task.spawn(NpcTp_Sequence)
        end
    end)

    -- Re-define/coloca AntiAFK toggle aqui (garante que exista apenas um handler)
    Tabs.Main:AddToggle("AntiAFK", {
        Title = "Anti AFK ",
        Default = true,
        Callback = function(state)
            autoAfkOn = state
            if autoAfkOn then
                enableAfk()
            else
                disableAfk()
            end
        end
    })

    -- Aba Settings (SaveManager + InterfaceManager)
    local SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua"))()
    local InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua"))()

    SaveManager:SetLibrary(Fluent)
    InterfaceManager:SetLibrary(Fluent)

    InterfaceManager:BuildInterfaceSection(Tabs.Settings)
    SaveManager:BuildConfigSection(Tabs.Settings)

    Window:SelectTab(1)
end

print("[Script] loaded successfully")
