-- RaidManager (Server) - Colocar em ServerScriptService
-- Sistema de raid infinito que spawna ondas, aplica dano automático e recompensa jogadores.
-- Requisitos:
--   ReplicatedStorage.RaidTemplates.EnemyTemplate  (Model com Humanoid + HumanoidRootPart)
--   ReplicatedStorage.StartRaidEvent  (RemoteEvent)
--   ReplicatedStorage.RaidUpdateEvent (RemoteEvent) - usado para enviar status aos clients

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ServerStorage = game:GetService("ServerStorage")

-- Configurações (ajuste como quiser)
local TEMPLATES_FOLDER = ReplicatedStorage:WaitForChild("RaidTemplates")
local ENEMY_TEMPLATE = TEMPLATES_FOLDER:WaitForChild("EnemyTemplate") -- modelo base (clonar)
local START_EVENT = ReplicatedStorage:WaitForChild("StartRaidEvent")
local UPDATE_EVENT = ReplicatedStorage:WaitForChild("RaidUpdateEvent")

local SPAWN_POINT = workspace:FindFirstChild("RaidSpawn") or workspace -- onde os inimigos vão aparecer
local MAX_ACTIVE_PER_WAVE = 10             -- inimigos simultâneos por onda (inicial)
local WAVE_INTERVAL = 8                    -- segundos entre ondas (existem cooldowns internos)
local ENEMY_BASE_HEALTH = 200              -- health padrão (se o template tiver, será preservado)
local ENEMY_SCALE_PER_WAVE = 1.05          -- multiplicador de dificuldade por onda
local DAMAGE_PER_PLAYER_PER_SECOND = 20    -- dano aplicado aos inimigos por cada jogador participante
local REWARD_PER_PLAYER_PER_WAVE = 10      -- gold por jogador a cada onda completada
local SPAWN_OFFSET_RADIUS = 10             -- área de spawn em torno de SPAWN_POINT
local MAX_TOTAL_ACTIVE = 50                -- segurança: máximo total de NPCs ativos

-- Segurança / tolerância
local RUN_TIMEOUT = 60 * 60 * 3 -- 3 horas (safety); se quiser completamente infinito, aumente. Serve pra evitar loops permanentes acidentais.

-- Estado
local raidRunning = false
local participants = {} -- [player] = true
local currentWave = 0
local activeEnemies = {} -- lista de humanoids

-- Helpers
local function isPlayerValid(p)
    return p and p.Parent
end

local function broadcastStatus(statusTable)
    -- envia um table simples para os clients com info da raid
    -- ex: {running = true, wave = 5, activeEnemies = 12}
    pcall(function()
        UPDATE_EVENT:FireAllClients(statusTable)
    end)
end

local function randomSpawnCFrame(center)
    local x = (math.random() - 0.5) * 2 * SPAWN_OFFSET_RADIUS
    local z = (math.random() - 0.5) * 2 * SPAWN_OFFSET_RADIUS
    local y = 3
    local cf = CFrame.new(center.Position + Vector3.new(x, y, z))
    return cf
end

local function spawnEnemy(scaleMultiplier)
    local clone = ENEMY_TEMPLATE:Clone()
    clone.Name = "RaidEnemy"
    -- tenta definir humanoid health se existir
    local humanoid = clone:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid.MaxHealth = (humanoid.MaxHealth > 0 and humanoid.MaxHealth) or ENEMY_BASE_HEALTH
        humanoid.Health = humanoid.MaxHealth * (scaleMultiplier or 1)
    end
    -- posicione
    if SPAWN_POINT:IsA("BasePart") then
        clone:SetPrimaryPartCFrame(randomSpawnCFrame(SPAWN_POINT))
    elseif typeof(SPAWN_POINT) == "Instance" and SPAWN_POINT:IsA("Model") and SPAWN_POINT.PrimaryPart then
        clone:SetPrimaryPartCFrame(randomSpawnCFrame(SPAWN_POINT.PrimaryPart))
    else
        -- posição fallback: perto do workspace origin
        clone:SetPrimaryPartCFrame(CFrame.new(math.random(-10,10), 5, math.random(-10,10)))
    end

    clone.Parent = workspace
    -- registra humanoid
    local hum = clone:FindFirstChildOfClass("Humanoid")
    if hum then
        table.insert(activeEnemies, hum)
    end
    return clone
end

local function cleanupDeadEnemies()
    local alive = {}
    for _, hum in ipairs(activeEnemies) do
        if hum and hum.Parent and hum.Health > 0 then
            table.insert(alive, hum)
        else
            -- tenta destruir o model pai pra limpar
            if hum and hum.Parent then
                local m = hum.Parent
                pcall(function() m:Destroy() end)
            end
        end
    end
    activeEnemies = alive
end

local function rewardParticipants(amount)
    for pl,_ in pairs(participants) do
        if isPlayerValid(pl) then
            -- cria leaderstats Gold se não existir
            local stats = pl:FindFirstChild("leaderstats")
            if not stats then
                stats = Instance.new("Folder")
                stats.Name = "leaderstats"
                stats.Parent = pl
            end
            local gold = stats:FindFirstChild("Gold")
            if not gold then
                gold = Instance.new("IntValue")
                gold.Name = "Gold"
                gold.Value = 0
                gold.Parent = stats
            end
            gold.Value = gold.Value + amount
        end
    end
end

-- Dano contínuo: usa RunService.Heartbeat para aplicar dano por segundo proporcional ao número de players
local lastHeartbeat = tick()
RunService.Heartbeat:Connect(function(dt)
    if not raidRunning then return end
    -- atualiza lista de participantes válida
    for pl,_ in pairs(participants) do
        if not isPlayerValid(pl) then participants[pl] = nil end
    end
    local playerCount = 0
    for _,_ in pairs(participants) do playerCount = playerCount + 1 end
    if playerCount == 0 then
        -- ninguém participando -> encerra raid
        raidRunning = false
        broadcastStatus({running = false, reason = "Sem participantes"})
        return
    end

    if #activeEnemies == 0 then
        -- sem inimigos, nada a aplicar
        return
    end

    local damageThisFrame = DAMAGE_PER_PLAYER_PER_SECOND * playerCount * dt
    for i = #activeEnemies, 1, -1 do
        local hum = activeEnemies[i]
        if hum and hum.Parent and hum.Health > 0 then
            -- aplica dano
            hum:TakeDamage(damageThisFrame)
        else
            -- remove da tabela
            table.remove(activeEnemies, i)
        end
    end
end)

-- Função principal que gerencia as ondas (executa numa coroutine quando inicia)
local function runRaid(initiator)
    if raidRunning then return false, "Raid já em andamento." end
    if not isPlayerValid(initiator) then return false, "Iniciador inválido." end

    -- inicia
    raidRunning = true
    participants = {}
    participants[initiator] = true
    currentWave = 0
    local startTime = tick()

    broadcastStatus({running = true, wave = currentWave, activeEnemies = #activeEnemies})

    while raidRunning do
        if tick() - startTime > RUN_TIMEOUT then
            -- safety cutoff
            raidRunning = false
            break
        end

        -- incrementa onda
        currentWave = currentWave + 1
        -- calcula quantos inimigos spawnar (cresce com as waves)
        local enemiesToSpawn = math.min(MAX_ACTIVE_PER_WAVE * math.ceil(currentWave * 0.5), MAX_TOTAL_ACTIVE)
        local scaleMultiplier = ENEMY_SCALE_PER_WAVE ^ (currentWave - 1)

        -- spawn inicial (respeitando MAX_TOTAL_ACTIVE)
        for i = 1, enemiesToSpawn do
            if #activeEnemies >= MAX_TOTAL_ACTIVE then break end
            spawnEnemy(scaleMultiplier)
            wait(0.08) -- pequeno espaçamento entre spawns para evitar picos
        end

        broadcastStatus({running = true, wave = currentWave, activeEnemies = #activeEnemies})

        -- Espera a onda ser limpa (ou limite de tempo por onda)
        local waveStart = tick()
        local WAVE_MAX_DURATION = math.max(15, WAVE_INTERVAL * 4) -- segurança
        while tick() - waveStart < WAVE_MAX_DURATION and raidRunning do
            cleanupDeadEnemies()
            -- se todos mortos, break pra próxima onda
            if #activeEnemies == 0 then break end
            wait(1)
        end

        -- recompensa por onda
        rewardParticipants(REWARD_PER_PLAYER_PER_WAVE)

        broadcastStatus({running = true, wave = currentWave, activeEnemies = #activeEnemies})

        -- pequeno intervalo entre ondas
        wait(WAVE_INTERVAL)
    end

    -- finaliza e limpa tudo
    cleanupDeadEnemies()
    raidRunning = false
    participants = {}
    currentWave = 0
    broadcastStatus({running = false, wave = 0, activeEnemies = #activeEnemies})
    return true, "Raid finalizada."
end

-- API: players podem se juntar à raid enquanto ela estiver rodando
local function joinRaid(player)
    if not player then return end
    participants[player] = true
    broadcastStatus({running = raidRunning, wave = currentWave, activeEnemies = #activeEnemies})
end

-- RemoteEvent handlers
START_EVENT.OnServerEvent:Connect(function(player, action)
    -- action: "start", "stop", "join"
    if action == "start" then
        if raidRunning then
            START_EVENT:FireClient(player, false, "Raid já em andamento.")
            return
        end
        -- roda a raid em coroutine pra não bloquear
        spawn(function()
            local ok, msg = runRaid(player)
            START_EVENT:FireClient(player, ok, msg or "")
        end)
        START_EVENT:FireClient(player, true, "Raid iniciada.")
    elseif action == "stop" then
        raidRunning = false
        START_EVENT:FireClient(player, true, "Raid parada.")
    elseif action == "join" then
        if not raidRunning then
            START_EVENT:FireClient(player, false, "Não há raid ativa.")
            return
        end
        joinRaid(player)
        START_EVENT:FireClient(player, true, "Você entrou na raid.")
    end
end)

-- Cleanup on player removal
Players.PlayerRemoving:Connect(function(pl)
    participants[pl] = nil
end)

print("[RaidManager] carregado — pronto para raids infinitas (use StartRaidEvent:FireServer('start')).")
