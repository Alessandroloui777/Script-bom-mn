-- StarterPlayerScripts/QA_Unificado_Client.lua

local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local remotes = ReplicatedStorage:WaitForChild("QA_Remotes")

local RE_Teleport     = remotes:WaitForChild("QA_Teleport")
local RE_Buy          = remotes:WaitForChild("QA_Buy")
local RE_Sell         = remotes:WaitForChild("QA_Sell")
local RF_RequestState = remotes:WaitForChild("QA_RequestState")

-- Teleport local
RE_Teleport.OnClientEvent:Connect(function()
    local char = player.Character
    if char and char.PrimaryPart then
        char:SetPrimaryPartCFrame(CFrame.new(0,5,0))
    end
end)

-- Criação da UI
local function createUI()
    local screen = Instance.new("ScreenGui")
    screen.Name = "QA_Tools"
    screen.Parent = player:WaitForChild("PlayerGui")

    local frame = Instance.new("Frame")
    frame.Size     = UDim2.fromOffset(300, 360)
    frame.Position = UDim2.new(0,20, 0.5,-180)
    frame.Parent   = screen

    local title = Instance.new("TextLabel")
    title.Size    = UDim2.new(1,0, 0,32)
    title.Text    = "Painel QA"
    title.Parent  = frame

    local lblInfo = Instance.new("TextLabel")
    lblInfo.Size        = UDim2.new(1,-20,0,60)
    lblInfo.Position    = UDim2.new(0,10, 0,40)
    lblInfo.TextWrapped = true
    lblInfo.Parent      = frame

    local function mkBtn(txt, y)
        local b = Instance.new("TextButton")
        b.Size     = UDim2.new(1,-20,0,30)
        b.Position = UDim2.new(0,10,0,y)
        b.Text     = txt
        b.Parent   = frame
        return b
    end

    local btnTp      = mkBtn("Teleportar à Base", 110)
    local btnBuyBest = mkBtn("Comprar Melhor",     150)

    local rarityBox = Instance.new("TextBox")
    rarityBox.Size         = UDim2.new(1,-20,0,30)
    rarityBox.Position     = UDim2.new(0,10,0,190)
    rarityBox.PlaceholderText = "Raridade (Common, Uncommon...)"
    rarityBox.Parent       = frame

    local btnBuyRar = mkBtn("Comprar por Raridade", 230)
    local btnSellLow = mkBtn("Vender Menor DPS",    270)
    local autoToggle = mkBtn("Auto-Compra: OFF",    310)

    local auto = false

    -- Atualiza o painel de status
    local function refresh()
        local state = RF_RequestState:InvokeServer()
        lblInfo.Text = string.format(
            "Cash: %d | DPS: %d\nInv: %d/%d",
            state.cash,
            state.totalDps,
            #state.inventory,
            state.capacity
        )
    end

    -- Conexões dos botões
    btnTp.MouseButton1Click:Connect(function()
        RE_Teleport:FireServer()
        refresh()
    end)

    btnBuyBest.MouseButton1Click:Connect(function()
        RE_Buy:FireServer(false, "")
        refresh()
    end)

    btnBuyRar.MouseButton1Click:Connect(function()
        RE_Buy:FireServer(true, rarityBox.Text)
        refresh()
    end)

    btnSellLow.MouseButton1Click:Connect(function()
        RE_Sell:FireServer(true)
        refresh()
    end)

    autoToggle.MouseButton1Click:Connect(function()
        auto = not auto
        autoToggle.Text = "Auto-Compra: " .. (auto and "ON" or "OFF")
    end)

    -- Loop de auto-compra
    task.spawn(function()
        while screen.Parent do
            if auto then
                RE_Buy:FireServer(false, "")
                refresh()
            end
            task.wait(2)
        end
    end)

    -- Primeira atualização
    refresh()
end

createUI()
