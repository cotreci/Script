--[[
    AutoFarm - Storage Hunters
    Script otimizado para o jogo Storage Hunters
]]

-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

-- ====== CONFIG ======
local Config = {
    Toggles = {
        AutoBid = false,
        AutoColetar = false,
        AutoDrive = false,
        AutoDescarregar = false,
        AutoPlot = false,
        AutoLimpeza = false
    }
}

-- ====== VARIÁVEIS GLOBAIS ======
local LastBid = 0
local IsBidding = false
local CollectedItems = {}

-- ====== UTILITÁRIOS ======

local function GetRoot()
    local char = LocalPlayer.Character
    if char then
        return char:FindFirstChild("HumanoidRootPart")
    end
    return nil
end

local function GetHumanoid()
    local char = LocalPlayer.Character
    if char then
        return char:FindFirstChild("Humanoid")
    end
    return nil
end

local function Teleport(pos)
    local root = GetRoot()
    if root and pos then
        pcall(function()
            root.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0))
            root.Velocity = Vector3.new(0, 0, 0)
        end)
        return true
    end
    return false
end

local function WalkTo(pos)
    local humanoid = GetHumanoid()
    if humanoid and pos then
        humanoid:MoveTo(pos)
        return true
    end
    return false
end

local function FindUI(namePattern)
    for _, gui in pairs(game.CoreGui:GetChildren()) do
        if gui:IsA("ScreenGui") and gui.Enabled then
            if string.lower(gui.Name):find(string.lower(namePattern)) then
                return gui
            end
        end
    end
    -- Procurar em PlayerGui também
    local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
    if playerGui then
        for _, gui in pairs(playerGui:GetChildren()) do
            if gui:IsA("ScreenGui") and gui.Enabled then
                if string.lower(gui.Name):find(string.lower(namePattern)) then
                    return gui
                end
            end
        end
    end
    return nil
end

local function FindAllUIs()
    local uis = {}
    for _, gui in pairs(game.CoreGui:GetChildren()) do
        if gui:IsA("ScreenGui") and gui.Enabled then
            table.insert(uis, gui)
        end
    end
    local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
    if playerGui then
        for _, gui in pairs(playerGui:GetChildren()) do
            if gui:IsA("ScreenGui") and gui.Enabled then
                table.insert(uis, gui)
            end
        end
    end
    return uis
end

local function FindButton(ui, textPattern)
    if not ui then return nil end
    for _, child in pairs(ui:GetDescendants()) do
        if (child:IsA("TextButton") or child:IsA("ImageButton")) and child.Visible then
            if child.Text and string.lower(child.Text):find(string.lower(textPattern)) then
                return child
            end
        end
    end
    return nil
end

local function FindAllButtons(ui)
    local buttons = {}
    if not ui then return buttons end
    for _, child in pairs(ui:GetDescendants()) do
        if (child:IsA("TextButton") or child:IsA("ImageButton")) and child.Visible then
            table.insert(buttons, child)
        end
    end
    return buttons
end

local function FindText(ui, textPattern)
    if not ui then return nil end
    for _, child in pairs(ui:GetDescendants()) do
        if (child:IsA("TextLabel") or child:IsA("TextButton")) and child.Visible then
            if child.Text and string.lower(child.Text):find(string.lower(textPattern)) then
                return child
            end
        end
    end
    return nil
end

local function Click(btn)
    if not btn then return false end
    pcall(function()
        btn:Click()
        btn:Activate()
        if btn.MouseButton1Click then
            btn.MouseButton1Click:Fire()
        end
    end)
    return true
end

local function ClickAtPosition(pos)
    if pos then
        VirtualInputManager:SendMouseButtonEvent(pos.X, pos.Y, 0, true, game, 0)
        task.wait(0.05)
        VirtualInputManager:SendMouseButtonEvent(pos.X, pos.Y, 0, false, game, 0)
        return true
    end
    return false
end

local function PressKey(key)
    VirtualInputManager:SendKeyEvent(true, key, false, game)
    task.wait(0.05)
    VirtualInputManager:SendKeyEvent(false, key, false, game)
end

local function Notify(msg, color)
    local gui = Instance.new("ScreenGui")
    gui.Name = "Notify"
    gui.Parent = game.CoreGui
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 320, 0, 35)
    frame.Position = UDim2.new(0.5, -160, 0.85, 0)
    frame.BackgroundColor3 = Color3.fromRGB(15, 15, 35)
    frame.BackgroundTransparency = 0.1
    frame.Parent = gui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame
    
    local text = Instance.new("TextLabel")
    text.Size = UDim2.new(1, -20, 1, 0)
    text.Position = UDim2.new(0, 10, 0, 0)
    text.BackgroundTransparency = 1
    text.Text = msg
    text.TextColor3 = color or Color3.fromRGB(255, 255, 255)
    text.TextSize = 12
    text.Font = Enum.Font.Gotham
    text.TextXAlignment = Enum.TextXAlignment.Left
    text.TextYAlignment = Enum.TextYAlignment.Center
    text.Parent = frame
    
    task.wait(2.5)
    gui:Destroy()
end

-- ====== SISTEMA DE BID ESPECÍFICO ======

-- Função para detectar o minigame de bid
local function DetectBidUI()
    -- Procurar por UIs relacionadas a leilão
    local uiNames = {"bid", "auction", "leilão", "storage", "container", "warehouse", "garage"}
    for _, name in pairs(uiNames) do
        local ui = FindUI(name)
        if ui then
            return ui
        end
    end
    
    -- Procurar por botões comuns em leilões
    local uis = FindAllUIs()
    for _, ui in pairs(uis) do
        local buttons = FindAllButtons(ui)
        for _, btn in pairs(buttons) do
            local text = string.lower(btn.Text or "")
            if string.find(text, "bid") or string.find(text, "$") or 
               string.find(text, "apostar") or string.find(text, "leilão") then
                return ui
            end
        end
    end
    return nil
end

-- Função para fazer o bid automaticamente
local function DoAutoBid()
    local ui = DetectBidUI()
    if not ui then return false end
    
    Notify("🎯 Leilão detectado!", Color3.fromRGB(255, 200, 0))
    
    -- Procurar botões de bid (geralmente são numerados ou com valores)
    local buttons = FindAllButtons(ui)
    local bidButtons = {}
    
    for _, btn in pairs(buttons) do
        local text = btn.Text or ""
        -- Verificar se é um botão de bid (contém $ ou número)
        if string.find(text, "$") or string.find(text, "%d+") then
            table.insert(bidButtons, btn)
        end
    end
    
    -- Ordenar por valor (maior primeiro)
    table.sort(bidButtons, function(a, b)
        local valA = tonumber(string.match(a.Text or "", "%d+")) or 0
        local valB = tonumber(string.match(b.Text or "", "%d+")) or 0
        return valA > valB
    end)
    
    -- Clicar no maior bid disponível
    if #bidButtons > 0 then
        local bestBid = bidButtons[1]
        local value = tonumber(string.match(bestBid.Text or "", "%d+")) or 0
        
        if value > LastBid then
            Click(bestBid)
            LastBid = value
            Notify("💰 Bid de $" .. value, Color3.fromRGB(0, 255, 100))
            return true
        end
    end
    
    -- Fallback: procurar botão "Bid" genérico
    local bidBtn = FindButton(ui, "bid") or FindButton(ui, "apostar")
    if bidBtn then
        Click(bidBtn)
        Notify("💰 Bid realizado!", Color3.fromRGB(0, 255, 100))
        return true
    end
    
    return false
end

-- ====== AUTO BID ======
local function AutoBid()
    Notify("🚗 Auto Bid iniciado", Color3.fromRGB(100, 200, 255))
    
    while Config.Toggles.AutoBid do
        task.wait(0.3)
        
        -- Verificar se está em um leilão
        local inAuction = false
        local uis = FindAllUIs()
        for _, ui in pairs(uis) do
            local text = FindText(ui, "leilão") or FindText(ui, "auction") or FindText(ui, "bid")
            if text then
                inAuction = true
                break
            end
        end
        
        if inAuction then
            DoAutoBid()
        else
            -- Se não estiver em leilão, procurar por containers para abrir
            local storageUI = FindUI("storage")
            if storageUI then
                local openBtn = FindButton(storageUI, "abrir") or FindButton(storageUI, "open")
                if openBtn then
                    Click(openBtn)
                    Notify("📦 Abrindo container...", Color3.fromRGB(255, 200, 0))
                    task.wait(1)
                end
            end
        end
    end
end

-- ====== AUTO COLETAR ======
local function AutoColetar()
    Notify("📦 Auto Coletar iniciado", Color3.fromRGB(100, 200, 255))
    
    while Config.Toggles.AutoColetar do
        task.wait(0.5)
        
        -- Procurar itens no chão (Storage Hunters tem itens espalhados)
        local foundItems = {}
        
        for _, obj in pairs(workspace:GetDescendants()) do
            if obj:IsA("BasePart") or obj:IsA("Model") then
                local name = string.lower(obj.Name)
                local hasClick = obj:FindFirstChild("ClickDetector") or 
                               (obj:IsA("Model") and obj:FindFirstChild("ClickDetector"))
                
                -- Verificar se é um item coletável
                if hasClick or string.find(name, "item") or 
                   string.find(name, "loot") or string.find(name, "drop") or
                   string.find(name, "collect") or string.find(name, "crate") or
                   string.find(name, "caixa") or string.find(name, "container") then
                    
                    local pos
                    if obj:IsA("Model") and obj.PrimaryPart then
                        pos = obj.PrimaryPart.Position
                    elseif obj:IsA("BasePart") then
                        pos = obj.Position
                    end
                    
                    if pos and not CollectedItems[obj] then
                        table.insert(foundItems, {obj = obj, pos = pos})
                    end
                end
            end
        end
        
        -- Coletar itens encontrados
        for _, item in pairs(foundItems) do
            Teleport(item.pos)
            task.wait(0.3)
            
            -- Tentar coletar via ClickDetector
            local cd = item.obj:FindFirstChild("ClickDetector")
            if cd then
                cd:FireClick(LocalPlayer)
                CollectedItems[item.obj] = true
                Notify("✅ Coletou: " .. item.obj.Name, Color3.fromRGB(0, 255, 100))
                task.wait(0.2)
            end
            
            -- Se não tiver ClickDetector, tentar via UI
            local ui = FindUI("collect") or FindUI("coletar")
            if ui then
                local btn = FindButton(ui, "coletar") or FindButton(ui, "collect") or FindButton(ui, "pegar")
                if btn then
                    Click(btn)
                    CollectedItems[item.obj] = true
                    Notify("✅ Coletou via UI", Color3.fromRGB(0, 255, 100))
                    task.wait(0.2)
                end
            end
        end
        
        -- Procurar UI de coleta em massa
        local ui = FindUI("loot") or FindUI("reward")
        if ui then
            local claimBtn = FindButton(ui, "reivindicar") or FindButton(ui, "claim") or FindButton(ui, "coletar")
            if claimBtn then
                Click(claimBtn)
                Notify("✅ Reivindicou recompensas", Color3.fromRGB(0, 255, 100))
            end
        end
    end
end

-- ====== AUTO DRIVE ======
local function AutoDrive()
    Notify("🚀 Auto Drive iniciado", Color3.fromRGB(100, 200, 255))
    
    while Config.Toggles.AutoDrive do
        task.wait(2)
        
        -- Procurar veículos (Storage Hunters tem caminhões/empilhadeiras)
        local vehicle = nil
        for _, obj in pairs(workspace:GetChildren()) do
            if obj:IsA("Model") then
                local name = string.lower(obj.Name)
                if string.find(name, "truck") or string.find(name, "caminhão") or 
                   string.find(name, "forklift") or string.find(name, "empilhadeira") or
                   string.find(name, "vehicle") or string.find(name, "carro") or
                   string.find(name, "van") or string.find(name, "carrinho") then
                    vehicle = obj
                    break
                end
            end
        end
        
        if vehicle then
            Notify("🚗 Veículo encontrado: " .. vehicle.Name, Color3.fromRGB(255, 200, 0))
            
            -- Procurar assento
            local seat = vehicle:FindFirstChild("Seat") or 
                        vehicle:FindFirstChild("DriverSeat") or
                        vehicle:FindFirstChild("VehicleSeat") or
                        vehicle:FindFirstChild("Driver")
            
            if seat then
                Teleport(seat.Position)
                task.wait(0.5)
                
                -- Entrar no veículo
                if seat:IsA("Seat") or seat:IsA("VehicleSeat") then
                    local char = LocalPlayer.Character
                    if char then
                        seat:EnterSeat(char)
                        Notify("✅ Entrou no veículo", Color3.fromRGB(0, 255, 100))
                        task.wait(1)
                    end
                end
                
                -- Procurar área de descarregamento
                local unloadPos = nil
                for _, obj in pairs(workspace:GetDescendants()) do
                    if obj:IsA("BasePart") then
                        local name = string.lower(obj.Name)
                        if string.find(name, "unload") or string.find(name, "descarregar") or
                           string.find(name, "delivery") or string.find(name, "entrega") or
                           string.find(name, "drop") or string.find(name, "zone") then
                            unloadPos = obj.Position
                            break
                        end
                    end
                end
                
                if unloadPos then
                    if vehicle.PrimaryPart then
                        vehicle.PrimaryPart.CFrame = CFrame.new(unloadPos)
                        Notify("✅ Veículo teleportado para descarregar", Color3.fromRGB(0, 255, 100))
                    end
                end
            end
        end
    end
end

-- ====== AUTO DESCARREGAR ======
local function AutoDescarregar()
    Notify("📥 Auto Descarregar iniciado", Color3.fromRGB(100, 200, 255))
    
    while Config.Toggles.AutoDescarregar do
        task.wait(1)
        
        -- Procurar UI de descarregamento
        local ui = FindUI("unload") or FindUI("descarregar") or FindUI("delivery") or FindUI("entrega")
        
        if ui then
            Notify("📋 UI de descarregamento encontrada", Color3.fromRGB(255, 200, 0))
            
            -- Procurar botão de descarregar
            local btn = FindButton(ui, "descarregar") or 
                       FindButton(ui, "unload") or
                       FindButton(ui, "entregar") or
                       FindButton(ui, "deliver") or
                       FindButton(ui, "confirmar")
            
            if btn then
                Click(btn)
                Notify("✅ Descarregando...", Color3.fromRGB(0, 255, 100))
                task.wait(2)
            end
            
            -- Tentar botões numerados (quantidade de itens)
            for i = 1, 10 do
                local numBtn = FindButton(ui, tostring(i))
                if numBtn then
                    Click(numBtn)
                    Notify("✅ Descarregou " .. i .. " itens", Color3.fromRGB(0, 255, 100))
                    break
                end
            end
        end
        
        -- Procurar área de descarregamento no mapa
        if not ui then
            for _, obj in pairs(workspace:GetDescendants()) do
                if obj:IsA("BasePart") then
                    local name = string.lower(obj.Name)
                    if string.find(name, "unload") or string.find(name, "descarregar") then
                        Teleport(obj.Position)
                        Notify("📍 Teleportado para área de descarregar", Color3.fromRGB(255, 200, 0))
                        task.wait(0.5)
                        break
                    end
                end
            end
        end
    end
end

-- ====== AUTO PLOT ======
local function AutoPlot()
    Notify("📊 Auto Plot iniciado", Color3.fromRGB(100, 200, 255))
    
    while Config.Toggles.AutoPlot do
        task.wait(1)
        
        -- Procurar UI da loja/estoque
        local ui = FindUI("shop") or FindUI("store") or FindUI("loja") or 
                   FindUI("storage") or FindUI("inventory") or FindUI("estoque")
        
        if ui then
            Notify("🏪 UI da loja/estoque encontrada", Color3.fromRGB(255, 200, 0))
            
            -- Procurar itens para colocar
            local itemsPlaced = 0
            local allButtons = FindAllButtons(ui)
            
            for _, btn in pairs(allButtons) do
                local text = btn.Text or ""
                -- Verificar se é um item (não é botão de ação)
                if not string.find(string.lower(text), "voltar") and
                   not string.find(string.lower(text), "close") and
                   not string.find(string.lower(text), "fechar") and
                   not string.find(string.lower(text), "sair") then
                    
                    if string.len(text) > 1 then
                        Click(btn)
                        itemsPlaced = itemsPlaced + 1
                        task.wait(0.15)
                    end
                end
            end
            
            -- Botão para confirmar colocação
            local confirmBtn = FindButton(ui, "colocar") or 
                              FindButton(ui, "place") or
                              FindButton(ui, "plot") or
                              FindButton(ui, "adicionar") or
                              FindButton(ui, "add") or
                              FindButton(ui, "confirmar")
            
            if confirmBtn then
                Click(confirmBtn)
                Notify("✅ " .. itemsPlaced .. " itens colocados", Color3.fromRGB(0, 255, 100))
            end
        end
    end
end

-- ====== AUTO LIMPEZA ======
local function AutoLimpeza()
    Notify("🧹 Auto Limpeza iniciado", Color3.fromRGB(100, 200, 255))
    
    while Config.Toggles.AutoLimpeza do
        task.wait(2)
        
        -- Verificar se há itens sujos
        local hasDirty = false
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        
        if playerGui then
            for _, gui in pairs(playerGui:GetChildren()) do
                if gui:IsA("ScreenGui") then
                    for _, child in pairs(gui:GetDescendants()) do
                        if child:IsA("TextLabel") or child:IsA("TextButton") then
                            local text = string.lower(child.Text or "")
                            if string.find(text, "dirty") or string.find(text, "sujo") or
                               string.find(text, "sujeira") or string.find(text, "sujar") then
                                hasDirty = true
                                Notify("🧹 Item sujo encontrado", Color3.fromRGB(255, 200, 0))
                                break
                            end
                        end
                    end
                end
                if hasDirty then break end
            end
        end
        
        if hasDirty then
            -- Procurar área de limpeza
            local cleanPos = nil
            for _, obj in pairs(workspace:GetDescendants()) do
                if obj:IsA("BasePart") then
                    local name = string.lower(obj.Name)
                    if string.find(name, "clean") or string.find(name, "limpeza") or
                       string.find(name, "wash") or string.find(name, "lavar") then
                        cleanPos = obj.Position
                        break
                    end
                end
            end
            
            if cleanPos then
                Teleport(cleanPos)
                Notify("📍 Teleportado para limpeza", Color3.fromRGB(255, 200, 0))
                task.wait(0.5)
            end
            
            -- Procurar UI de limpeza
            local ui = FindUI("clean") or FindUI("limpeza") or FindUI("lavar")
            
            if ui then
                Notify("🧹 UI de limpeza encontrada", Color3.fromRGB(255, 200, 0))
                
                -- Clicar em limpar item
                local cleanBtn = FindButton(ui, "limpar") or 
                                FindButton(ui, "clean") or
                                FindButton(ui, "lavar") or
                                FindButton(ui, "wash")
                
                if cleanBtn then
                    Click(cleanBtn)
                    Notify("🧼 Limpando item...", Color3.fromRGB(0, 200, 255))
                    task.wait(3)
                end
            end
        end
    end
end

-- ====== UI PRINCIPAL ======

local function CreateUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "AutoFarmSH"
    screenGui.Parent = game.CoreGui
    screenGui.ResetOnSpawn = false
    
    local main = Instance.new("Frame")
    main.Size = UDim2.new(0, 260, 0, 380)
    main.Position = UDim2.new(0.5, -130, 0.5, -190)
    main.BackgroundColor3 = Color3.fromRGB(12, 12, 28)
    main.BorderColor3 = Color3.fromRGB(100, 70, 255)
    main.BorderSizePixel = 1
    main.Parent = screenGui
    main.Visible = false
    
    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 10)
    mainCorner.Parent = main
    
    -- Título
    local title = Instance.new("Frame")
    title.Size = UDim2.new(1, 0, 0, 38)
    title.BackgroundColor3 = Color3.fromRGB(20, 20, 45)
    title.Parent = main
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 10)
    titleCorner.Parent = title
    
    local titleText = Instance.new("TextLabel")
    titleText.Size = UDim2.new(1, -40, 1, 0)
    titleText.Position = UDim2.new(0, 15, 0, 0)
    titleText.BackgroundTransparency = 1
    titleText.Text = "📦 Storage AutoFarm"
    titleText.TextColor3 = Color3.fromRGB(120, 80, 255)
    titleText.TextSize = 16
    titleText.TextXAlignment = Enum.TextXAlignment.Left
    titleText.TextYAlignment = Enum.TextYAlignment.Center
    titleText.Font = Enum.Font.GothamBold
    titleText.Parent = title
    
    -- Botão fechar
    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 26, 0, 26)
    close.Position = UDim2.new(1, -33, 0, 6)
    close.BackgroundColor3 = Color3.fromRGB(200, 40, 40)
    close.BackgroundTransparency = 0.7
    close.Text = "✕"
    close.TextColor3 = Color3.fromRGB(255, 255, 255)
    close.TextSize = 15
    close.Font = Enum.Font.GothamBold
    close.Parent = title
    
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 5)
    closeCorner.Parent = close
    
    local isOpen = false
    close.MouseButton1Click:Connect(function()
        isOpen = not isOpen
        main.Visible = isOpen
    end)
    
    -- Scroll
    local scroll = Instance.new("ScrollingFrame")
    scroll.Size = UDim2.new(1, -16, 1, -50)
    scroll.Position = UDim2.new(0, 8, 0, 44)
    scroll.BackgroundTransparency = 1
    scroll.ScrollBarThickness = 3
    scroll.ScrollBarImageColor3 = Color3.fromRGB(100, 70, 255)
    scroll.CanvasSize = UDim2.new(0, 0, 0, 350)
    scroll.Parent = main
    
    local list = Instance.new("UIListLayout")
    list.Padding = UDim.new(0, 5)
    list.SortOrder = Enum.SortOrder.LayoutOrder
    list.Parent = scroll
    
    -- Toggles específicos do Storage Hunters
    local toggleData = {
        {"AutoBid", "💰 Auto Bid"},
        {"AutoColetar", "📦 Auto Coletar"},
        {"AutoDrive", "🚚 Auto Drive"},
        {"AutoDescarregar", "📥 Auto Descarregar"},
        {"AutoPlot", "📊 Auto Plot"},
        {"AutoLimpeza", "🧹 Auto Limpeza"}
    }
    
    local systems = {
        AutoBid = AutoBid,
        AutoColetar = AutoColetar,
        AutoDrive = AutoDrive,
        AutoDescarregar = AutoDescarregar,
        AutoPlot = AutoPlot,
        AutoLimpeza = AutoLimpeza
    }
    
    for _, data in pairs(toggleData) do
        local key, label = data[1], data[2]
        
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(1, 0, 0, 42)
        frame.BackgroundColor3 = Color3.fromRGB(22, 22, 45)
        frame.BackgroundTransparency = 0.3
        frame.Parent = scroll
        
        local frameCorner = Instance.new("UICorner")
        frameCorner.CornerRadius = UDim.new(0, 6)
        frameCorner.Parent = frame
        
        frame.MouseEnter:Connect(function()
            TweenService:Create(frame, TweenInfo.new(0.2), {
                BackgroundTransparency = 0.1
            }):Play()
        end)
        frame.MouseLeave:Connect(function()
            TweenService:Create(frame, TweenInfo.new(0.2), {
                BackgroundTransparency = 0.3
            }):Play()
        end)
        
        local text = Instance.new("TextLabel")
        text.Size = UDim2.new(0.7, -10, 1, 0)
        text.Position = UDim2.new(0, 12, 0, 0)
        text.BackgroundTransparency = 1
        text.Text = label
        text.TextColor3 = Color3.fromRGB(240, 240, 255)
        text.TextSize = 13
        text.TextXAlignment = Enum.TextXAlignment.Left
        text.TextYAlignment = Enum.TextYAlignment.Center
        text.Font = Enum.Font.Gotham
        text.Parent = frame
        
        -- Switch
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 44, 0, 22)
        btn.Position = UDim2.new(1, -50, 0.5, -11)
        btn.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
        btn.Text = ""
        btn.Parent = frame
        
        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 11)
        btnCorner.Parent = btn
        
        local indicator = Instance.new("Frame")
        indicator.Size = UDim2.new(0, 16, 0, 16)
        indicator.Position = UDim2.new(0, 3, 0.5, -8)
        indicator.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
        indicator.BackgroundTransparency = 0.4
        indicator.Parent = btn
        
        local indCorner = Instance.new("UICorner")
        indCorner.CornerRadius = UDim.new(0, 8)
        indCorner.Parent = indicator
        
        local active = false
        
        btn.MouseButton1Click:Connect(function()
            active = not active
            Config.Toggles[key] = active
            
            if active then
                btn.BackgroundColor3 = Color3.fromRGB(100, 70, 255)
                indicator.BackgroundTransparency = 0
                indicator.Position = UDim2.new(1, -19, 0.5, -8)
                Notify("✅ " .. label .. " ATIVADO", Color3.fromRGB(0, 255, 100))
                
                if systems[key] then
                    coroutine.wrap(systems[key])()
                end
            else
                btn.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
                indicator.BackgroundTransparency = 0.4
                indicator.Position = UDim2.new(0, 3, 0.5, -8)
                Notify("❌ " .. label .. " DESATIVADO", Color3.fromRGB(255, 100, 100))
            end
        end)
        
        frame.LayoutOrder = #toggleData
    end
    
    -- Atualizar canvas
    list:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        scroll.CanvasSize = UDim2.new(0, 0, 0, list.AbsoluteContentSize.Y + 10)
    end)
    
    -- Info extra
    local info = Instance.new("TextLabel")
    info.Size = UDim2.new(1, -20, 0, 20)
    info.Position = UDim2.new(0, 10, 1, -25)
    info.BackgroundTransparency = 1
    info.Text = "🔄 Storage Hunters | INSERT"
    info.TextColor3 = Color3.fromRGB(150, 150, 200)
    info.TextSize = 10
    info.Font = Enum.Font.Gotham
    info.TextXAlignment = Enum.TextXAlignment.Center
    info.Parent = main
    
    -- Tecla Insert
    UserInputService.InputBegan:Connect(function(input)
        if input.KeyCode == Enum.KeyCode.Insert then
            isOpen = not isOpen
            main.Visible = isOpen
            if isOpen then
                Notify("📱 Storage AutoFarm Carregado!", Color3.fromRGB(100, 200, 255))
            end
        end
    end)
    
    Notify("🚀 Storage AutoFarm Carregado! Pressione INSERT", Color3.fromRGB(0, 255, 100))
    return screenGui
end

-- ====== INICIAR ======
CreateUI()
