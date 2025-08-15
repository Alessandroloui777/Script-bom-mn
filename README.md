-- LocalScript QA Unificado (coloque em StarterPlayerScripts ou StarterGui)

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Criar pasta de Remotes se não existir
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

local RE_Teleport = ensureRemote("QA_Teleport", "RemoteEvent")
local RE_Buy = ensureRemote("QA_Buy", "RemoteEvent")
local RE_Sell = ensureRemote("QA_Sell", "RemoteEvent")
local RF_State = ensureRemote("QA_RequestState", "RemoteFunction")

-- Catálogo de Brainrots
local Catalog = {
    {id=1, name="Common Smirk", rarity="Common", dps=5, price=100},
    {id=2, name="Side Eye", rarity="Common", dps=7, price=150},
    {id=3, name="Grimace", rarity="Uncommon", dps=15, price=350},
    {id=4, name="Rizz Master", rarity="Rare", dps=35, price=900},
    {id=5, name="Sigma Stare", rarity="Epic", dps=75, price=2200},
    {id=6, name="Giga Chad", rarity="Legendary", dps=160, price=5200},
}
local RarityOrder = {Common=1, Uncommon=2, Rare=3, Epic=4, Legendary=5}

-- Estado do jogador
local PlayerState = {
    cash = 5000,
    inventory = {},
}
local BASE_CAPACITY = 8
local BASE_CFRAME = CFrame.new(0, 5, 0)

-- Funções auxiliares
local function sortByRarityThenDps()
    local t = table.clone(Catalog)
    table.sort(t, function(a,b)
        if RarityOrder[a.rarity] == RarityOrder[b.rarity] then
            return a.dps > b.dps
        end
        return RarityOrder[a.rarity] > RarityOrder[b.rarity]
    end)
    return t
end

local function getByRarity(rar)
    for _, it in ipairs(Catalog) do
        if it.rarity == rar then return it end
    end
end

local function totalDps(inv)
    local s = 0
    for _, it in ipairs(inv) do s += it.dps end
    return s
end

local function findLowestDpsIndex(inv)
    local idx, minDps = nil, math.huge
    for i, it in ipairs(inv) do
        if it.dps < minDps then
            minDps = it.dps
            idx = i
        end
    end
    return idx
end

-- Eventos simulados (cliente)
RE_Teleport.OnClientEvent:Connect(function()
    local char = Players.LocalPlayer.Character
    if char and char.PrimaryPart then
        char:SetPrimaryPartCFrame(BASE_CFRAME)
    end
end)

-- Funções de compra e venda simuladas
RE_Buy.OnClientEvent:Connect(function(byRarity, wantedRarity)
    if #PlayerState.inventory >= BASE_CAPACITY then
        local idx = findLowestDpsIndex(PlayerState.inventory)
        if idx then
            local sold = table.remove(PlayerState.inventory, idx)
            PlayerState.cash += math.floor(sold.price * 0.6)
        end
    end
    local candidate
    if byRarity and wantedRarity then
        candidate = getByRarity(wantedRarity) or sortByRarityThenDps()[1]
    else
        candidate = sortByRarityThenDps()[1]
    end
    if candidate and PlayerState.cash >= candidate.price then
        PlayerState.cash -= candidate.price
        table.insert(PlayerState.inventory, candidate)
    end
end)

RE_Sell.OnClientEvent:Connect(function(sellLowest)
    if #PlayerState.inventory == 0 then return end
    local idx = sellLowest and findLowestDpsIndex(PlayerState.inventory) or #PlayerState.inventory
    local sold = table.remove(PlayerState.inventory, idx)
    PlayerState.cash += math.floor(sold.price * 0.6)
end)

RF_State.OnClientInvoke = function()
    return {
        cash = PlayerState.cash,
        inventory = PlayerState.inventory,
        capacity = BASE_CAPACITY,
        totalDps = totalDps(PlayerState.inventory),
    }
end

-- ======= UI =======
local function createUI()
    local player = Players.LocalPlayer

    local screen = Instance.new("ScreenGui")
    screen.Name = "QA_Tools"
    screen.Parent = player:WaitForChild("PlayerGui")

    local frame = Instance.new("Frame")
    frame.Size = UDim2.fromOffset(300, 360)
    frame.Position = UDim2.new(0, 20, 0.5, -180)
    frame.Parent = screen

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 32)
    title.Text = "Painel QA"
    title.Parent = frame

    local lblInfo = Instance.new("TextLabel")
    lblInfo.Size = UDim2.new(1, -20, 0, 60)
    lblInfo.Position = UDim2.new(0, 10, 0, 40)
    lblInfo.TextWrapped = true
    lblInfo.Parent = frame

    local function mkBtn(text, y)
        local b = Instance.new("TextButton")
        b.Size = UDim2.new(1, -20, 0, 30)
        b.Position = UDim2.new(0, 10, 0, y)
        b.Text = text
        b.Parent = frame
        return b
    end

    local btnTp = mkBtn("Teleportar à Base", 110)
    local btnBuyBest = mkBtn("Comprar Melhor", 150)
    local rarityBox = Instance.new("TextBox")
    rarityBox.Size = UDim2.new(1, -20, 0, 30)
    rarityBox.Position = UDim2.new(0, 10, 0, 190)
    rarityBox.PlaceholderText = "Raridade"
    rarityBox.Parent = frame
    local btnBuyRar = mkBtn("Comprar por Raridade", 230)
    local btnSellLow = mkBtn("Vender Menor DPS", 270)
    local autoToggle = mkBtn("Auto-Compra: OFF", 310)

    local auto = false
    local function refresh()
        local state = RF_State:Invoke()
        if state then
            lblInfo.Text = string.format("Cash: %d | DPS: %d\nInv: %d/%d",
                state.cash, state.totalDps, #state.inventory, state.capacity)
        end
    end

    btnTp.MouseButton1Click:Connect(function()
        RE_Teleport:FireClient(player)
        refresh()
    end)
    btnBuyBest.MouseButton1Click:Connect(function()
        RE_Buy:FireClient(player, false, nil)
        refresh()
    end)
    btnBuyRar.MouseButton1Click:Connect(function()
        RE_Buy:FireClient(player, true, rarityBox.Text)
        refresh()
    end)
    btnSellLow.MouseButton1Click:Connect(function()
        RE_Sell:FireClient(player, true)
        refresh()
    end)
    autoToggle.MouseButton1Click:Connect(function()
        auto = not auto
        autoToggle.Text = "Auto-Compra: " .. (auto and "ON" or "OFF")
    end)

    task.spawn(function()
        while screen.Parent do
            if auto then
                RE_Buy:FireClient(player, false, nil)
                refresh()
            end
            task.wait(2)
        end
    end)
    refresh()
end

createUI()
