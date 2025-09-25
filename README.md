-- CheatDetector (ServerScriptService)
-- Detecta comportamento suspeito (teleporte/velocidade extrema, taxa de ação alta)
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local CheatFlagEvent = ReplicatedStorage:WaitForChild("CheatFlagEvent") -- server -> admin clients
local RequestAdminData = ReplicatedStorage:WaitForChild("RequestAdminData") -- admin client -> server

-- Ajuste estes limites conforme seu jogo (valores iniciais conservadores)
local SPEED_THRESHOLD = 200         -- studs/segundo (velocidade linear anormal)
local TELEPORT_DISTANCE = 60        -- distância entre amostras que indica possível teleporte
local SAMPLE_INTERVAL = 1.0         -- segundos entre amostras por jogador
local ACTIONS_PER_MINUTE_THRESHOLD = 300 -- ações (shots/hits) por minuto consideradas suspeitas

-- Armazenamento simples de histórico por jogador
local playerData = {}

local function flagPlayer(plr, reason, evidence)
    playerData[plr].flags = playerData[plr].flags or {}
    table.insert(playerData[plr].flags, {
        time = os.time(),
        reason = reason,
        evidence = evidence
    })
    -- Envia um evento para GUIs de admin conectadas com resumo
    CheatFlagEvent:FireAllClients({
        userId = plr.UserId,
        playerName = plr.Name,
        reason = reason,
        evidence = evidence,
        timestamp = os.time()
    })
    print(("[CheatDetector] Flagged %s (%d): %s"):format(plr.Name, plr.UserId, reason))
end

local function initPlayer(plr)
    playerData[plr] = {
        lastPos = nil,
        lastCheckTime = tick(),
        actionsCount = 0, -- ex.: contagem de "tiros" ou ações rápidas; você precisará incrementar onde fizer sentido
        flags = {}
    }

    -- exemplo: listener para quando jogador dispara uma RemoteEvent do jogo (se existir)
    -- Se seu jogo já usa um RemoteEvent de "playerAction", poderia incrementar actionsCount ali.
end

local function removePlayer(plr)
    playerData[plr] = nil
end

Players.PlayerAdded:Connect(function(plr)
    initPlayer(plr)
    plr.CharacterAdded:Connect(function(char)
        wait(0.5)
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if hrp then
            playerData[plr].lastPos = hrp.Position
            playerData[plr].lastCheckTime = tick()
        end
    end)
end)

Players.PlayerRemoving:Connect(removePlayer)

-- Periodic checker
spawn(function()
    while true do
        local now = tick()
        for plr, data in pairs(playerData) do
            if plr and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
                local hrp = plr.Character.HumanoidRootPart
                local pos = hrp.Position
                local dt = now - (data.lastCheckTime or now)
                if dt > 0 then
                    local dist = (pos - (data.lastPos or pos)).magnitude
                    local speed = dist / dt -- studs / second

                    -- Teleporte forte ou mudança muito grande
                    if dist > TELEPORT_DISTANCE then
                        flagPlayer(plr, "Teleport/large position jump", {dist = dist, dt = dt, speed = speed})
                    end

                    -- Velocidade anormal
                    if speed > SPEED_THRESHOLD then
                        flagPlayer(plr, "Excessive movement speed", {speed = speed, dist = dist, dt = dt})
                    end

                    data.lastPos = pos
                    data.lastCheckTime = now
                end

                -- Verifica taxa de ações: converte contador para taxa por minuto
                if data.actionsCount and data.actionsCount > 0 then
                    local ratePerMinute = (data.actionsCount / math.max(1, now - (data.rateStartTime or now))) * 60
                    if ratePerMinute > ACTIONS_PER_MINUTE_THRESHOLD then
                        flagPlayer(plr, "Excessive action rate", {ratePerMinute = ratePerMinute})
                    end
                    -- reset a cada ciclo para evitar flags duplicadas demais
                    data.actionsCount = 0
                    data.rateStartTime = now
                end
            end
        end
        wait(SAMPLE_INTERVAL)
    end
end)

-- API para outros scripts reportarem ações (ex: quando um jogador registra um "hit" no servidor)
-- Você pode chamar esta função de outros scripts do servidor do jogo.
local DetectorAPI = {}
function DetectorAPI.RecordAction(plr)
    if playerData[plr] then
        playerData[plr].actionsCount = (playerData[plr].actionsCount or 0) + 1
        if not playerData[plr].rateStartTime then
            playerData[plr].rateStartTime = tick()
        end
    end
end

-- Exponha no ServerStorage para outros scripts do servidor importarem (opcional)
local ServerStorage = game:GetService("ServerStorage")
local module = Instance.new("ModuleScript")
module.Name = "CheatDetectorAPI"
module.Source = "-- module created at runtime; ignore if unused"
module.Parent = ServerStorage
-- We keep the module approach simple: we'll just store a reference table
-- (alternatively, other server scripts can just require a ModuleScript you add manually)
module:SetAttribute("api", true) -- marker; not used programmatically here

-- Handler: quando admin GUI pede dados, devolvemos resumo (server-authoritative)
RequestAdminData.OnServerEvent:Connect(function(requestingPlayer)
    -- Verifique permissão do requestingPlayer antes de enviar dados
    -- Aqui usamos um simples controle por UserId. Troque pelos seus admins reais.
    local allowedAdmins = {
        [12345678] = true, -- <- substitua por seus IDs de admin
        -- ex: [98765432] = true
    }
    if not allowedAdmins[requestingPlayer.UserId] then
        return
    end

    local summary = {}
    for plr, data in pairs(playerData) do
        local entry = {
            userId = plr.UserId,
            name = plr.Name,
            flags = data.flags or {},
            lastPos = (plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") and plr.Character.HumanoidRootPart.Position) or nil
        }
        table.insert(summary, entry)
    end
    CheatFlagEvent:FireClient(requestingPlayer, {type = "adminData", data = summary})
end)

-- Exponha API para outros scripts (opcional)
_G.CheatDetectorAPI = DetectorAPI
