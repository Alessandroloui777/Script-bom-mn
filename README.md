-- LocalScript em StarterPlayerScripts

local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService        = game:GetService("RunService")
local player            = Players.LocalPlayer

-- CONFIGURE AQUI os nomes reais dos Remotes do jogo
local REMOTE_FOLDER     = ReplicatedStorage:WaitForChild("Remotes")        -- ou o nome que tiver
local REMOTE_BUY        = REMOTE_FOLDER:WaitForChild("BuyBrainrot")       -- RemoteEvent para comprar
local REMOTE_SELL       = REMOTE_FOLDER:WaitForChild("SellBrainrot")      -- RemoteEvent para vender
local REMOTE_TELEPORT   = REMOTE_FOLDER:WaitForChild("TeleportToBase")    -- RemoteEvent para teleporte

-- ESTATÍSTICAS do jogador (ajuste conforme leaderstats do jogo)
local leaderstats       = player:WaitForChild("leaderstats")
local statCash          = leaderstats:WaitForChild("Cash")
local statRebirths      = leaderstats:WaitForChild("Rebirths")

-- PARÂMETROS
local BASE_CAPACITY     = 8        -- capacidade da base
local BUY_INTERVAL      = 2        -- segundos entre tentativas de compra
local TELEPORT_INTERVAL = 1        -- segundos entre teleports

-- Estado interno dos toggles
local autoBuy  = false
local autoTP   = false
local targetRarity = "Secreto"     -- mude para a raridade que quiser comprar

-- FUNÇÃO: encontra o Brainrot de menor DPS na sua base
local function getLowestDpsTool()
    local lowest, lowestDps = nil, math.huge
    for _, tool in ipairs(player.Backpack:GetChildren()) do
        if tool.Name:match("Brainrot") and tool:FindFirstChild("DPS") then
            local dps = tool.DPS.Value
            if dps < lowestDps then
                lowestDps = dps
                lowest = tool
            end
        end
    end
    return lowest
end

-- FUNÇÃO: compra um Brainrot de targetRarity
local function tryBuy()
    -- se base cheia
    if #player.Backpack:GetChildren() >= BASE_CAPACITY then
        -- vende o de menor DPS
        local lowTool = getLowestDpsTool()
        if lowTool then
            REMOTE_SELL:FireServer(lowTool)
            return
        end
    end

    -- dispara compra pelo servidor
    REMOTE_BUY:FireServer(targetRarity)
end

-- FUNÇÃO: teleporta e equipa
local function doTeleport()
    if not player.Character or not player.Character.PrimaryPart then return end
    REMOTE_TELEPORT:FireServer()
    -- equipa qualquer Brainrot na mão
    for _, tool in ipairs(player.Backpack:GetChildren()) do
        if tool:IsA("Tool") and tool.Name:match("Brainrot") then
            player.Character.Humanoid:EquipTool(tool)
            break
        end
    end
end

-- LOOP de Auto-Compra
spawn(function()
    while RunService.Heartbeat:Wait() do
        if autoBuy then
            tryBuy()
            task.wait(BUY_INTERVAL)
        else
            task.wait(0.1)
        end
    end
end)

-- LOOP de Auto-Teleport
spawn(function()
    while RunService.Heartbeat:Wait() do
        if autoTP then
            doTeleport()
            task.wait(TELEPORT_INTERVAL)
        else
            task.wait(0.1)
        end
    end
end)

-- UI SIMPLES para ligar/desligar e trocar raridade
local screenGui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
screenGui.Name = "BrainrotHelper"

local frame = Instance.new("Frame", screenGui)
frame.Size = UDim2.new(0, 200, 0, 140)
frame.Position = UDim2.new(0, 20, 0.5, -70)
frame.BackgroundTransparency = 0.3

local function mkButton(text, y)
    local btn = Instance.new("TextButton", frame)
    btn.Size = UDim2.new(1, -10, 0, 30)
    btn.Position = UDim2.new(0, 5, 0, y)
    btn.Text = text
    return btn
end

local buyBtn = mkButton("Auto-Buy: OFF",   5)
local tpBtn  = mkButton("Auto-TP: OFF",    40)
local box    = Instance.new("TextBox", frame)
box.Size = UDim2.new(1, -10, 0, 30)
box.Position = UDim2.new(0, 5, 0, 75)
box.PlaceholderText = "Digite raridade"
box.Text = targetRarity

buyBtn.MouseButton1Click:Connect(function()
    autoBuy = not autoBuy
    buyBtn.Text = "Auto-Buy: " .. (autoBuy and "ON" or "OFF")
end)

tpBtn.MouseButton1Click:Connect(function()
    autoTP = not autoTP
    tpBtn.Text = "Auto-TP: " .. (autoTP and "ON" or "OFF")
end)

box.FocusLost:Connect(function(enterPressed)
    if enterPressed and box.Text ~= "" then
        targetRarity = box.Text
    end
end)
