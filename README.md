-- Script Unificado para executores (Delta, Fluxus)

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer

-- Criar pasta de Remotes (simulados)
local remotesFolder = ReplicatedStorage:FindFirstChild("QA_Remotes") or Instance.new("Folder")
remotesFolder.Name = "QA_Remotes"
remotesFolder.Parent = ReplicatedStorage

local function ensureRemote(name, className)
    local r = remotesFolder:FindFirstChild(name)
    if not r then
        r = Instance.new(className)
        r.Name = name
        r.Parent = remotesFolder
    end
    return r
end

-- Remotes simulados
local RE_Teleport     = ensureRemote("QA_Teleport", "RemoteEvent")
local RE_Buy          = ensureRemote("QA_Buy", "RemoteEvent")
local RE_Sell         = ensureRemote("QA_Sell", "RemoteEvent")
local RF_RequestState = ensureRemote("QA_RequestState", "RemoteFunction")

-- Catálogo
local Catalog = {
    {id=1, name="Smirk",     rarity="Common",    dps=5,   price=100},
    {id=2, name="Side Eye",  rarity="Common",    dps=7,   price=150},
    {id=3, name="Grimace",   rarity="Uncommon",  dps=15,  price=350},
    {id=4, name="Rizz",      rarity="Rare",      dps=35,  price=900},
    {id=5, name="Sigma",     rarity="Epic",      dps=75,  price=2200},
    {id=6, name="Chad",      rarity="Legendary", dps=160, price=5200},
}
local RarityOrder = { Common=1, Uncommon=2, Rare=3, Epic=4, Legendary=5 }

-- Estado do jogador
local State = {
    cash = 5000,
    inventory = {},
    capacity = 8,
}

-- Funções auxiliares
local function totalDps()
    local sum = 0
    for _, item in ipairs(State.inventory) do
        sum += item.dps
    end
    return sum
end

local function findLowest()
    local idx, min = nil, math.huge
    for i, item in ipairs(State.inventory) do
        if item.dps < min then
            min = item.dps
            idx = i
        end
    end
    return idx
end

local function getByRarity(rar)
    for _, item in ipairs(Catalog) do
        if item.rarity == rar then return item end
    end
end

-- Simulação de ações
RE_Teleport.OnClientEvent:Connect(function()
    local char = player.Character
    if char and char.PrimaryPart then
        char:SetPrimaryPartCFrame(CFrame.new(0, 5, 0))
    end
end)

RE_Buy.OnClientEvent:Connect(function(byRarity, rar)
    if #State.inventory >= State.capacity then
        local idx = findLowest()
        if idx then
            local sold = table.remove(State.inventory, idx)
            State.cash += math.floor(sold.price * 0.6)
        end
    end

    local item = byRarity and getByRarity(rar) or Catalog[math.random(1, #Catalog)]
    if item and State.cash >= item.price then
        State.cash -= item.price
        table.insert(State.inventory, item)
    end
end)

RE_Sell.OnClientEvent:Connect(function(sellLowest)
    if #State.inventory == 0 then return end
    local idx = sellLowest and findLowest() or #State.inventory
    local sold = table.remove(State.inventory, idx)
    State.cash += math.floor(sold.price * 0.6)
end)

RF_RequestState.OnClientInvoke = function()
    return {
        cash = State.cash,
        inventory = State.inventory,
        capacity = State.capacity,
        totalDps = totalDps(),
    }
end

-- Interface simples
local function createUI()
    local gui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
    gui.Name = "QA_UI"

    local frame = Instance.new("Frame", gui)
    frame.Size = UDim2.fromOffset(300, 360)
    frame.Position = UDim2.new(0, 20, 0.5, -180)

    local info = Instance.new("TextLabel", frame)
    info.Size = UDim2.new(1, -20, 0, 60)
    info.Position = UDim2.new(0, 10, 0, 10)
    info.TextWrapped = true

    local function mkBtn(text, y)
        local b = Instance.new("TextButton", frame)
        b.Size = UDim2.new(1, -20, 0, 30)
        b.Position = UDim2.new(0, 10, 0, y)
        b.Text = text
        return b
    end

    local btnTp      = mkBtn("Teleportar", 80)
    local btnBuy     = mkBtn("Comprar Aleatório", 120)
    local rarityBox  = Instance.new("TextBox", frame)
    rarityBox.Size = UDim2.new(1, -20, 0, 30)
    rarityBox.Position = UDim2.new(0, 10, 0, 160)
    rarityBox.PlaceholderText = "Raridade (Common, Rare...)"

    local btnBuyRar  = mkBtn("Comprar por Raridade", 200)
    local btnSell    = mkBtn("Vender Menor DPS", 240)
    local autoBtn    = mkBtn("Auto: OFF", 280)

    local auto = false

    local function refresh()
        info.Text = string.format("Cash: %d | DPS: %d\nInv: %d/%d",
            State.cash, totalDps(), #State.inventory, State.capacity)
    end

    btnTp.MouseButton1Click:Connect(function()
        RE_Teleport:FireClient(player)
        refresh()
    end)

    btnBuy.MouseButton1Click:Connect(function()
        RE_Buy:FireClient(player, false, "")
        refresh()
    end)

    btnBuyRar.MouseButton1Click:Connect(function()
        RE_Buy:FireClient(player, true, rarityBox.Text)
        refresh()
    end)

    btnSell.MouseButton1Click:Connect(function()
        RE_Sell:FireClient(player, true)
        refresh()
    end)

    autoBtn.MouseButton1Click:Connect(function()
        auto = not auto
        autoBtn.Text = "Auto: " .. (auto and "ON" or "OFF")
    end)

    task.spawn(function()
        while gui.Parent do
            if auto then
                RE_Buy:FireClient(player, false, "")
                refresh()
            end
            task.wait(2)
        end
    end)

    refresh()
end

createUI()
