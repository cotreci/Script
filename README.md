--[[
    AutoFarm Pro - Versão Melhorada com Debug
    Adicione este script em um LocalScript dentro de StarterPlayerScripts
]]

-- ===================
-- SERVIÇOS
-- ===================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local TeleportService = game:GetService("TeleportService")

-- ===================
-- CONFIGURAÇÕES
-- ===================
local Config = {
    Debug = true, -- Ativar logs de debug
    Toggles = {
        AutoBid = false,
        AutoColetar = false,
        AutoDrive = false,
        AutoDescarregar = false,
        AutoPlot = false,
        AutoLimpeza = false
    },
    Positions = {
        Store = Vector3.new(0, 5, 0), -- Ajuste para posição da loja
        Laundry = Vector3.new(0, 5, 0), -- Ajuste para posição da lavanderia
        Garage = Vector3.new(0, 5, 0), -- Ajuste para posição da garagem
        UnloadArea = Vector3.new(0, 5, 0), -- Ajuste para área de descarregamento
        PlotArea = Vector3.new(0, 5, 0) -- Ajuste para área das prateleiras
    }
}

-- ===================
-- SISTEMA DE DEBUG
-- ===================
local function DebugLog(message, type)
    if not Config.Debug then return end
    type = type or "INFO"
    local colors = {
        INFO = "\27[36m", -- Ciano
        SUCCESS = "\27[32m", -- Verde
        WARNING = "\27[33m", -- Amarelo
        ERROR = "\27[31m", -- Vermelho
        DEBUG = "\27[35m" -- Roxo
    }
    print(colors[type] or "\27[37m", string.format("[%s] %s", type, message), "\27[0m")
end

-- ===================
-- UTILITÁRIOS MELHORADOS
-- ===================
local Utils = {}

function Utils:GetCharacter()
    local char = LocalPlayer.Character
    if not char or not char.Parent then
        DebugLog("Personagem não encontrado!", "ERROR")
        return nil
    end
    return char
end

function Utils:GetHumanoid()
    local char = self:GetCharacter()
    if not char then return nil end
    return char:FindFirstChild("Humanoid")
end

function Utils:GetRootPart()
    local char = self:GetCharacter()
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart")
end

function Utils:TeleportTo(position, offset)
    local root = self:GetRootPart()
    if not root then 
        DebugLog("RootPart não encontrado!", "ERROR")
        return false 
    end
    
    if not position then
        DebugLog("Posição inválida!", "ERROR")
        return false
    end
    
    offset = offset or Vector3.new(0, 3, 0)
    local targetPos = position + offset
    
    DebugLog(string.format("Teleportando para: %.2f, %.2f, %.2f", targetPos.X, targetPos.Y, targetPos.Z), "DEBUG")
    
    local success, err = pcall(function()
        root.CFrame = CFrame.new(targetPos)
        root.Velocity = Vector3.new(0, 0, 0)
    end)
    
    if not success then
        DebugLog("Falha ao teleportar: " .. tostring(err), "ERROR")
        return false
    end
    
    return true
end

function Utils:FindAllUI()
    local uis = {}
    for _, gui in pairs(game.CoreGui:GetChildren()) do
        if gui:IsA("ScreenGui") and gui.Enabled then
            table.insert(uis, gui)
        end
    end
    return uis
end

function Utils:FindUIByName(namePattern)
    DebugLog("Procurando UI com padrão: " .. namePattern, "DEBUG")
    for _, gui in pairs(game.CoreGui:GetChildren()) do
        if gui:IsA("ScreenGui") and gui.Name:lower():find(namePattern:lower()) then
            DebugLog("UI encontrada: " .. gui.Name, "SUCCESS")
            return gui
        end
    end
    DebugLog("UI não encontrada: " .. namePattern, "WARNING")
    return nil
end

function Utils:FindButtonByText(ui, textPattern)
    if not ui then return nil end
    
    DebugLog("Procurando botão com texto: " .. textPattern, "DEBUG")
    
    local function search(parent)
        for _, child in pairs(parent:GetChildren()) do
            if child:IsA("TextButton") or child:IsA("ImageButton") then
                if child.Text and child.Text:lower():find(textPattern:lower()) then
                    DebugLog("Botão encontrado: " .. child.Text, "SUCCESS")
                    return child
                end
            end
            local found = search(child)
            if found then return found end
        end
        return nil
    end
    
    return search(ui)
end

function Utils:FindItemByText(ui, textPattern)
    if not ui then return nil end
    
    local function search(parent)
        for _, child in pairs(parent:GetChildren()) do
            if child:IsA("TextLabel") or child:IsA("TextButton") then
                if child.Text and child.Text:lower():find(textPattern:lower()) then
                    return child
                end
            end
            local found = search(child)
            if found then return found end
        end
        return nil
    end
    
    return search(ui)
end

function Utils:ClickButton(button)
    if not button then 
        DebugLog("Botão é nil!", "ERROR")
        return false 
    end
    
    DebugLog("Clicando no botão: " .. (button.Text or "Sem texto"), "DEBUG")
    
    local success, err = pcall(function()
        -- Tentativa 1: Método padrão
        if button:IsA("TextButton") or button:IsA("ImageButton") then
            button:Click()
            button:Activate()
        end
        
        -- Tentativa 2: MouseButton1Click
        if button.MouseButton1Click then
            button.MouseButton1Click:Fire()
        end
        
        -- Tentativa 3: VirtualInputManager (simula clique físico)
        if button.AbsolutePosition then
            local pos = button.AbsolutePosition
            local size = button.AbsoluteSize
            local center = pos + size / 2
            
            VirtualInputManager:SendMouseButtonEvent(center.X, center.Y, 0, true, game, 0)
            task.wait(0.05)
            VirtualInputManager:SendMouseButtonEvent(center.X, center.Y, 0, false, game, 0)
        end
    end)
    
    if not success then
        DebugLog("Erro ao clicar: " .. tostring(err), "ERROR")
        return false
    end
    
    return true
end

function Utils:WaitForUI(uiName, timeout)
    timeout = timeout or 5
    DebugLog("Aguardando UI: " .. uiName .. " (timeout: " .. timeout .. "s)", "DEBUG")
    
    local start = tick()
    while tick() - start < timeout do
        local ui = self:FindUIByName(uiName)
        if ui and ui.Enabled then
            DebugLog("UI encontrada: " .. uiName, "SUCCESS")
            return ui
        end
        task.wait(0.1)
    end
    
    DebugLog("Timeout - UI não encontrada: " .. uiName, "WARNING")
    return nil
end

function Utils:IsInGame()
    return LocalPlayer and LocalPlayer:FindFirstChild("PlayerGui") ~= nil
end

-- ===================
-- SISTEMA DE NOTIFICAÇÃO
-- ===================
function Utils:Notify(message, type)
    type = type or "info"
    DebugLog("NOTIFICAÇÃO: " .. message, type:upper())
    
    -- Criar notificação visual
    local gui = Instance.new("ScreenGui")
    gui.Name = "Notification"
    gui.Parent = game.CoreGui
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 300, 0, 50)
    frame.Position = UDim2.new(0.5, -150, 0, 10)
    frame.BackgroundColor3 = Color3.fromRGB(20, 20, 35)
    frame.BorderSizePixel = 0
    frame.BackgroundTransparency = 0.1
    frame.Parent = gui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = frame
    
    local border = Instance.new("Frame")
    border.Size = UDim2.new(1, 0, 0, 2)
    border.Position = UDim2.new(0, 0, 0, 0)
    border.BackgroundColor3 = type == "success" and Color3.fromRGB(0, 255, 100) or 
                              type == "warning" and Color3.fromRGB(255, 200, 0) or
                              type == "error" and Color3.fromRGB(255, 50, 50) or
                              Color3.fromRGB(100, 70, 255)
    border.Parent = frame
    
    local text = Instance.new("TextLabel")
    text.Size = UDim2.new(1, -20, 1, 0)
    text.Position = UDim2.new(0, 10, 0, 0)
    text.BackgroundTransparency = 1
    text.Text = message
    text.TextColor3 = Color3.fromRGB(255, 255, 255)
    text.TextSize = 13
    text.TextXAlignment = Enum.TextXAlignment.Left
    text.TextYAlignment = Enum.TextYAlignment.Center
    text.Font = Enum.Font.Gotham
    text.Parent = frame
    
    task.wait(3)
    frame:Destroy()
end

-- ===================
-- SISTEMAS DE AUTOMAÇÃO
-- ===================
local Systems = {}

-- 1. AUTO BID
function Systems:AutoBid()
    DebugLog("Iniciando Auto Bid...", "INFO")
    
    while Config.Toggles.AutoBid and RunService:IsRunning() do
        task.wait(0.5)
        
        -- Listar todas as UIs atuais para debug
        local allUI = Utils:FindAllUI()
        DebugLog(string.format("UIs encontradas: %d", #allUI), "DEBUG")
        
        for _, ui in pairs(allUI) do
            DebugLog("UI disponível: " .. ui.Name, "DEBUG")
            
            -- Procurar por UI de minigame
            if ui.Name:lower():find("bid") or ui.Name:lower():find("minigame") or 
               ui.Name:lower():find("garage") or ui.Name:lower():find("leilão") then
                
                DebugLog("UI de Bid encontrada: " .. ui.Name, "SUCCESS")
                
                -- Procurar botões de opção (1, 2, 3, 4, 5, etc)
                local optionsFound = 0
                for i = 1, 10 do
                    local btn = Utils:FindButtonByText(ui, tostring(i))
                    if btn then
                        optionsFound = optionsFound + 1
                        -- Pegar o maior número possível
                        if i >= 5 then -- Preferir números maiores
                            Utils:ClickButton(btn)
                            Utils:Notify("Auto Bid: Selecionado " .. i, "success")
                            break
                        end
                    end
                end
                
                if optionsFound == 0 then
                    -- Tentar encontrar botões comuns
                    local commonBtns = {"Bid", "Auction", "Leilão", "Apostar"}
                    for _, text in pairs(commonBtns) do
                        local btn = Utils:FindButtonByText(ui, text)
                        if btn then
                            Utils:ClickButton(btn)
                            Utils:Notify("Auto Bid: Clicado em " .. text, "success")
                            break
                        end
                    end
                end
            end
        end
    end
end

-- 2. AUTO COLETAR
function Systems:AutoColetar()
    DebugLog("Iniciando Auto Coletar...", "INFO")
    
    while Config.Toggles.AutoColetar and RunService:IsRunning() do
        task.wait(1)
        
        -- Procurar itens no workspace
        local itemsFound = 0
        local function searchItems(parent)
            for _, child in pairs(parent:GetChildren()) do
                if child:IsA("BasePart") or child:IsA("Model") then
                    if child.Name:lower():find("item") or 
                       child.Name:lower():find("colet") or
                       child.Name:lower():find("recurso") or
                       child:FindFirstChild("ClickDetector") then
                        
                        itemsFound = itemsFound + 1
                        DebugLog("Item encontrado: " .. child.Name, "DEBUG")
                        
                        -- Teleportar para o item
                        local pos = child:IsA("Model") and child.PrimaryPart and child.PrimaryPart.Position or child.Position
                        if pos then
                            Utils:TeleportTo(pos)
                            task.wait(0.3)
                            
                            -- Tentar coletar
                            local clickDetector = child:FindFirstChild("ClickDetector")
                            if clickDetector then
                                clickDetector:FireClick(LocalPlayer)
                                Utils:Notify("Coletando: " .. child.Name, "success")
                            end
                            
                            -- Tentar coletar via UI
                            local ui = Utils:FindUIByName("colet")
                            if ui then
                                local collectBtn = Utils:FindButtonByText(ui, "colet")
                                if collectBtn then
                                    Utils:ClickButton(collectBtn)
                                end
                            end
                        end
                    end
                end
                searchItems(child)
            end
        end
        
        searchItems(workspace)
        
        if itemsFound == 0 then
            DebugLog("Nenhum item encontrado para coletar", "WARNING")
        end
    end
end

-- 3. AUTO DRIVE
function Systems:AutoDrive()
    DebugLog("Iniciando Auto Drive...", "INFO")
    
    while Config.Toggles.AutoDrive and RunService:IsRunning() do
        task.wait(2)
        
        -- Procurar veículo
        local vehicle = nil
        for _, obj in pairs(workspace:GetChildren()) do
            if obj:IsA("Model") and (obj.Name:lower():find("veic") or obj.Name:lower():find("car") or obj.Name:lower():find("caminhão")) then
                vehicle = obj
                break
            end
        end
        
        if vehicle then
            DebugLog("Veículo encontrado: " .. vehicle.Name, "SUCCESS")
            
            -- Entrar no veículo
            local seat = vehicle:FindFirstChild("Seat") or vehicle:FindFirstChild("DriverSeat")
            if seat then
                Utils:TeleportTo(seat.Position, Vector3.new(0, 2, 0))
                task.wait(0.5)
                
                -- Tentar sentar
                if seat:IsA("Seat") then
                    local char = Utils:GetCharacter()
                    if char and char:FindFirstChild("Humanoid") then
                        char.Humanoid:MoveTo(seat.Position)
                        task.wait(0.5)
                        seat:EnterSeat(char)
                        Utils:Notify("Entrou no veículo", "success")
                    end
                end
                
                -- Teleportar veículo para a loja
                local storePos = Config.Positions.Store
                if vehicle.PrimaryPart then
                    vehicle.PrimaryPart.CFrame = CFrame.new(storePos)
                    Utils:Notify("Veículo teleportado para a loja", "success")
                end
            end
        else
            DebugLog("Nenhum veículo encontrado", "WARNING")
        end
    end
end

-- 4. AUTO DESCARREGAR
function Systems:AutoDescarregar()
    DebugLog("Iniciando Auto Descarregar...", "INFO")
    
    while Config.Toggles.AutoDescarregar and RunService:IsRunning() do
        task.wait(1)
        
        -- Teleportar para área de descarregamento
        Utils:TeleportTo(Config.Positions.UnloadArea)
        task.wait(0.5)
        
        -- Procurar UI de descarregamento
        local ui = Utils:FindUIByName("unload") or Utils:FindUIByName("descarregar")
        if ui then
            DebugLog("UI de descarregamento encontrada", "SUCCESS")
            
            local unloadBtn = Utils:FindButtonByText(ui, "descarregar") or 
                             Utils:FindButtonByText(ui, "unload")
            
            if unloadBtn then
                -- Tentar extrair quantidade
                local qtd = 10 -- Máximo padrão
                local qtdMatch = unloadBtn.Text:match("(%d+)")
                if qtdMatch then
                    qtd = tonumber(qtdMatch) or 10
                end
                
                Utils:ClickButton(unloadBtn)
                Utils:Notify("Descarregando " .. qtd .. " itens", "success")
                task.wait(2)
            else
                -- Procurar botões numerados
                for i = 1, 10 do
                    local btn = Utils:FindButtonByText(ui, tostring(i))
                    if btn then
                        Utils:ClickButton(btn)
                        Utils:Notify("Descarregando " .. i .. " itens", "success")
                        break
                    end
                end
            end
        else
            DebugLog("UI de descarregamento não encontrada", "WARNING")
        end
    end
end

-- 5. AUTO PLOT
function Systems:AutoPlot()
    DebugLog("Iniciando Auto Plot...", "INFO")
    
    while Config.Toggles.AutoPlot and RunService:IsRunning() do
        task.wait(1)
        
        -- Teleportar para área das prateleiras
        Utils:TeleportTo(Config.Positions.PlotArea)
        task.wait(0.5)
        
        -- Procurar UI da loja/prateleira
        local ui = Utils:FindUIByName("loja") or Utils:FindUIByName("shop") or 
                   Utils:FindUIByName("store") or Utils:FindUIByName("prateleira")
        
        if ui then
            DebugLog("UI da loja encontrada", "SUCCESS")
            
            -- Procurar botões de itens
            local itemsFound = 0
            local itemNames = {"Capybara", "Ferramenta", "Leite", "Ovos", "Aço", "Chave"}
            
            for _, name in pairs(itemNames) do
                local item = Utils:FindButtonByText(ui, name)
                if item then
                    Utils:ClickButton(item)
                    itemsFound = itemsFound + 1
                    task.wait(0.3)
                end
            end
            
            -- Procurar botão "Adicionar" ou "Colocar"
            local addBtn = Utils:FindButtonByText(ui, "adicionar") or 
                          Utils:FindButtonByText(ui, "colocar") or
                          Utils:FindButtonByText(ui, "plot")
            
            if addBtn then
                Utils:ClickButton(addBtn)
                Utils:Notify("Itens colocados na prateleira", "success")
            end
            
            if itemsFound == 0 then
                DebugLog("Nenhum item encontrado para colocar", "WARNING")
            end
        else
            DebugLog("UI da loja não encontrada", "WARNING")
        end
    end
end

-- 6. AUTO LIMPEZA
function Systems:AutoLimpeza()
    DebugLog("Iniciando Auto Limpeza...", "INFO")
    
    while Config.Toggles.AutoLimpeza and RunService:IsRunning() do
        task.wait(2)
        
        -- Verificar itens sujos no inventário
        local dirtyFound = false
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        
        if playerGui then
            for _, gui in pairs(playerGui:GetChildren()) do
                if gui:IsA("ScreenGui") then
                    local dirtyItem = Utils:FindItemByText(gui, "dirty") or 
                                     Utils:FindItemByText(gui, "sujo")
                    if dirtyItem then
                        dirtyFound = true
                        DebugLog("Item sujo encontrado: " .. dirtyItem.Text, "SUCCESS")
                        break
                    end
                end
            end
        end
        
        if dirtyFound then
            -- Teleportar para lavanderia
            Utils:TeleportTo(Config.Positions.Laundry)
            task.wait(0.5)
            
            -- Procurar interação (Pressione E)
            local interaction = workspace:FindFirstChild("Interaction") or 
                              workspace:FindFirstChild("Interact")
            
            if interaction then
                DebugLog("Interação encontrada", "SUCCESS")
                
                -- Simular pressionar E
                VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.E, false, game)
                task.wait(0.1)
                VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.E, false, game)
                Utils:Notify("Interagindo com lavanderia", "success")
                task.wait(1)
            end
            
            -- Procurar UI de limpeza
            local ui = Utils:WaitForUI("clean", 3) or Utils:WaitForUI("limpeza", 3)
            if ui then
                DebugLog("UI de limpeza encontrada", "SUCCESS")
                
                -- Verificar diamantes
                local diamonds = 0
                local diamondText = Utils:FindItemByText(ui, "diamond") or 
                                   Utils:FindItemByText(ui, "gem")
                if diamondText then
                    local match = diamondText.Text:match("(%d+)")
                    if match then
                        diamonds = tonumber(match) or 0
                    end
                end
                
                -- Escolher slot baseado nos diamantes
                local slot = diamonds >= 30 and "Slot 2" or "Slot 1"
                local slotBtn = Utils:FindButtonByText(ui, slot)
                if slotBtn then
                    Utils:ClickButton(slotBtn)
                    Utils:Notify("Usando " .. slot .. " para limpeza", "success")
                    task.wait(0.5)
                end
                
                -- Iniciar lavagem
                local washBtn = Utils:FindButtonByText(ui, "lavar") or 
                               Utils:FindButtonByText(ui, "wash") or
                               Utils:FindButtonByText(ui, "iniciar")
                if washBtn then
                    Utils:ClickButton(washBtn)
                    Utils:Notify("Lavagem iniciada", "success")
                    task.wait(5) -- Tempo de lavagem
                end
            end
        end
    end
end

-- ===================
-- UI PRINCIPAL
-- ===================
local UIManager = {}

function UIManager:CreateUI()
    DebugLog("Criando UI principal...", "INFO")
    
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "AutoFarmPro"
    screenGui.Parent = game.CoreGui
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 280, 0, 420)
    mainFrame.Position = UDim2.new(0.5, -140, 0.5, -210)
    mainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    mainFrame.BorderColor3 = Color3.fromRGB(65, 45, 120)
    mainFrame.BorderSizePixel = 1
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = screenGui
    mainFrame.Visible = false
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = mainFrame
    
    -- Title Bar
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 45)
    titleBar.BackgroundColor3 = Color3.fromRGB(20, 20, 35)
    titleBar.Parent = mainFrame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 12)
    titleCorner.Parent = titleBar
    
    local titleText = Instance.new("TextLabel")
    titleText.Size = UDim2.new(1, -40, 1, 0)
    titleText.Position = UDim2.new(0, 20, 0, 0)
    titleText.BackgroundTransparency = 1
    titleText.Text = "⚡ AutoFarm Pro"
    titleText.TextColor3 = Color3.fromRGB(100, 70, 255)
    titleText.TextSize = 18
    titleText.TextXAlignment = Enum.TextXAlignment.Left
    titleText.TextYAlignment = Enum.TextYAlignment.Center
    titleText.Font = Enum.Font.GothamBold
    titleText.Parent = titleBar
    
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 30, 0, 30)
    closeBtn.Position = UDim2.new(1, -35, 0, 7)
    closeBtn.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
    closeBtn.BackgroundTransparency = 0.8
    closeBtn.Text = "✕"
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.TextSize = 18
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.Parent = titleBar
    
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 6)
    closeCorner.Parent = closeBtn
    
    closeBtn.MouseButton1Click:Connect(function()
        self:ToggleUI()
    end)
    
    -- Scroll Frame
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1, -20, 1, -60)
    scrollFrame.Position = UDim2.new(0, 10, 0, 55)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.ScrollBarThickness = 4
    scrollFrame.ScrollBarImageColor3 = Color3.fromRGB(100, 70, 255)
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 400)
    scrollFrame.Parent = mainFrame
    
    local listLayout = Instance.new("UIListLayout")
    listLayout.Padding = UDim.new(0, 8)
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder
    listLayout.Parent = scrollFrame
    
    -- Toggles
    local toggleData = {
        {Key = "AutoBid", Label = "🚗 Auto Bid", Order = 1},
        {Key = "AutoColetar", Label = "📦 Auto Coletar", Order = 2},
        {Key = "AutoDrive", Label = "🚀 Auto Drive", Order = 3},
        {Key = "AutoDescarregar", Label = "📥 Auto Descarregar", Order = 4},
        {Key = "AutoPlot", Label = "📊 Auto Plot", Order = 5},
        {Key = "AutoLimpeza", Label = "🧹 Auto Limpeza", Order = 6}
    }
    
    for _, data in pairs(toggleData) do
        self:CreateToggle(scrollFrame, data.Key, data.Label, data.Order)
    end
    
    -- Debug Button (adicional)
    local debugBtn = Instance.new("TextButton")
    debugBtn.Size = UDim2.new(1, -20, 0, 30)
    debugBtn.Position = UDim2.new(0, 10, 1, -40)
    debugBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 50)
    debugBtn.Text = "🔍 Debug Info"
    debugBtn.TextColor3 = Color3.fromRGB(180, 180, 200)
    debugBtn.TextSize = 12
    debugBtn.Font = Enum.Font.Gotham
    debugBtn.Parent = mainFrame
    
    local debugCorner = Instance.new("UICorner")
    debugCorner.CornerRadius = UDim.new(0, 6)
    debugCorner.Parent = debugBtn
    
    debugBtn.MouseButton1Click:Connect(function()
        local allUI = Utils:FindAllUI()
        local msg = string.format("UIs ativas: %d\n", #allUI)
        for _, ui in pairs(allUI) do
            msg = msg .. " - " .. ui.Name .. "\n"
        end
        Utils:Notify(msg, "info")
    end)
    
    -- Update Canvas
    listLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 50)
    end)
    
    -- Draggable
    self:MakeDraggable(mainFrame, titleBar)
    
    self.MainFrame = mainFrame
    self.IsOpen = false
    
    DebugLog("UI criada com sucesso!", "SUCCESS")
end

function UIManager:CreateToggle(parent, key, label, order)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 50)
    frame.BackgroundColor3 = Color3.fromRGB(25, 25, 45)
    frame.BackgroundTransparency = 0.5
    frame.ClipsDescendants = true
    frame.LayoutOrder = order
    frame.Parent = parent
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = frame
    
    local labelText = Instance.new("TextLabel")
    labelText.Size = UDim2.new(0.7, -10, 1, 0)
    labelText.Position = UDim2.new(0, 15, 0, 0)
    labelText.BackgroundTransparency = 1
    labelText.Text = label
    labelText.TextColor3 = Color3.fromRGB(255, 255, 255)
    labelText.TextSize = 14
    labelText.TextXAlignment = Enum.TextXAlignment.Left
    labelText.TextYAlignment = Enum.TextYAlignment.Center
    labelText.Font = Enum.Font.GothamMedium
    labelText.Parent = frame
    
    local toggleBtn = Instance.new("TextButton")
    toggleBtn.Size = UDim2.new(0, 50, 0, 28)
    toggleBtn.Position = UDim2.new(1, -60, 0.5, -14)
    toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
    toggleBtn.Text = ""
    toggleBtn.Parent = frame
    
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 14)
    toggleCorner.Parent = toggleBtn
    
    local indicator = Instance.new("Frame")
    indicator.Size = UDim2.new(0, 22, 0, 22)
    indicator.Position = UDim2.new(0, 3, 0.5, -11)
    indicator.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    indicator.BackgroundTransparency = 0.5
    indicator.Parent = toggleBtn
    
    local indicatorCorner = Instance.new("UICorner")
    indicatorCorner.CornerRadius = UDim.new(0, 11)
    indicatorCorner.Parent = indicator
    
    local isOn = false
    
    local function updateToggle(state)
        isOn = state
        Config.Toggles[key] = state
        
        if state then
            toggleBtn.BackgroundColor3 = Color3.fromRGB(100, 70, 255)
            indicator.BackgroundTransparency = 0
            indicator.Position = UDim2.new(1, -25, 0.5, -11)
            Utils:Notify(label .. " ✅ ATIVADO", "success")
            
            -- Iniciar sistema
            local systems = {
                AutoBid = Systems.AutoBid,
                AutoColetar = Systems.AutoColetar,
                AutoDrive = Systems.AutoDrive,
                AutoDescarregar = Systems.AutoDescarregar,
                AutoPlot = Systems.AutoPlot,
                AutoLimpeza = Systems.AutoLimpeza
            }
            
            if systems[key] then
                coroutine.wrap(systems[key])()
            end
        else
            toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
            indicator.BackgroundTransparency = 0.5
            indicator.Position = UDim2.new(0, 3, 0.5, -11)
            Utils:Notify(label .. " ❌ DESATIVADO", "warning")
        end
    end
    
    toggleBtn.MouseButton1Click:Connect(function()
        updateToggle(not isOn)
    end)
    
    return frame
end

function UIManager:MakeDraggable(frame, dragHandle)
    local dragging = false
    local dragInput, dragStart, startPos
    
    dragHandle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    
    dragHandle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            dragInput = input
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            local screenSize = game:GetService("GuiService"):GetGuiInset()
            
            local newX = math.clamp(startPos.X.Offset + delta.X, -screenSize.X, screenSize.X)
            local newY = math.clamp(startPos.Y.Offset + delta.Y, -screenSize.Y, screenSize.Y)
            
            frame.P
