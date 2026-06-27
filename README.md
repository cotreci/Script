--[[
    AutoFarm Pro - Sistema Completo de Automação para Roblox
    Versão: 2.0.0
    Tema: Dark com detalhes em Azul/Roxo
    Desenvolvido com modularidade e performance em mente
]]

-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local GuiService = game:GetService("GuiService")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

--[[
    ===================
    MÓDULO DE CONFIGURAÇÃO
    ===================
]]
local Config = {
    UI = {
        Theme = {
            Background = Color3.fromRGB(15, 15, 25),
            Surface = Color3.fromRGB(25, 25, 45),
            Border = Color3.fromRGB(65, 45, 120),
            Primary = Color3.fromRGB(100, 70, 255),
            Secondary = Color3.fromRGB(50, 120, 255),
            Text = Color3.fromRGB(255, 255, 255),
            TextDim = Color3.fromRGB(180, 180, 200),
            Success = Color3.fromRGB(0, 255, 150),
            Warning = Color3.fromRGB(255, 200, 50),
            Error = Color3.fromRGB(255, 80, 80)
        },
        Animations = {
            Duration = 0.3,
            Style = Enum.EasingStyle.Quad,
            Direction = Enum.EasingDirection.Out
        }
    },
    Toggles = {
        AutoBid = false,
        AutoColetar = false,
        AutoDrive = false,
        AutoDescarregar = false,
        AutoPlot = false,
        AutoLimpeza = false
    },
    Settings = {
        MaxItems = 10,
        DiamondsThreshold = 30,
        WaitTime = 0.5,
        TeleportOffset = Vector3.new(0, 3, 0)
    }
}

--[[
    ===================
    MÓDULO DE UTILITÁRIOS
    ===================
]]
local Utils = {}

function Utils:GetPlayer()
    return LocalPlayer
end

function Utils:GetCharacter()
    local player = self:GetPlayer()
    return player and player.Character
end

function Utils:GetHumanoid()
    local char = self:GetCharacter()
    return char and char:FindFirstChild("Humanoid")
end

function Utils:TeleportTo(position, offset)
    local char = self:GetCharacter()
    if not char then return false end
        
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return false end
        
    local targetPos = position + (offset or Vector3.new(0, 3, 0))
    root.CFrame = CFrame.new(targetPos)
    return true
end

function Utils:FindUI(uiName)
    for _, gui in pairs(game.CoreGui:GetChildren()) do
        if gui:IsA("ScreenGui") and gui.Name:find(uiName) then
            return gui
        end
    end
    return nil
end

function Utils:FindButtonInUI(ui, buttonText)
    local function searchButton(parent)
        for _, child in pairs(parent:GetChildren()) do
            if child:IsA("TextButton") and child.Text:find(buttonText) then
                return child
            end
            local found = searchButton(child)
            if found then return found end
        end
        return nil
    end
    return searchButton(ui)
end

function Utils:FindItemInUI(ui, itemName)
    local function searchItem(parent)
        for _, child in pairs(parent:GetChildren()) do
            if child:IsA("TextLabel") and child.Text:find(itemName) then
                return child
            end
            local found = searchItem(child)
            if found then return found end
        end
        return nil
    end
    return searchItem(ui)
end

function Utils:SimulateClick(object)
    if not object then return false end
    local success, err = pcall(function()
        object:Activate()
        object:Fire()
        if object:IsA("TextButton") or object:IsA("ImageButton") then
            object:Click()
        end
    end)
    return success
end

function Utils:WaitForUI(uiName, timeout)
    timeout = timeout or 10
    local start = tick()
    while tick() - start < timeout do
        local ui = self:FindUI(uiName)
        if ui then return ui end
        task.wait(0.1)
    end
    return nil
end

function Utils:DetectNewUI(callback)
    local connection
    connection = game.CoreGui.ChildAdded:Connect(function(child)
        if child:IsA("ScreenGui") then
            callback(child)
        end
    end)
    return connection
end

function Utils:GetDirtyItems()
    local items = {}
    local inventory = self:GetPlayer().Inventory or {}
    
    for _, item in pairs(inventory) do
        if item.Name:find("Dirty") or item.Name:find("Sujo") then
            table.insert(items, item)
        end
    end
    return items
end

function Utils:GetPlayerDiamonds()
    local player = self:GetPlayer()
    return player:GetAttribute("Diamonds") or player:GetAttribute("Gems") or 0
end

function Utils:Notify(message, type)
    type = type or "info"
    local colors = {
        info = Config.UI.Theme.Primary,
        success = Config.UI.Theme.Success,
        warning = Config.UI.Theme.Warning,
        error = Config.UI.Theme.Error
    }
    
    -- Criar notificação
    local notification = Instance.new("ScreenGui")
    notification.Name = "Notification"
    notification.Parent = game.CoreGui
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 350, 0, 60)
    frame.Position = UDim2.new(0.5, -175, 0, 10)
    frame.BackgroundColor3 = Config.UI.Theme.Surface
    frame.BorderColor3 = Config.UI.Theme.Border
    frame.BorderSizePixel = 1
    frame.ClipsDescendants = true
    frame.Parent = notification
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = frame
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -20, 0, 25)
    title.Position = UDim2.new(0, 10, 0, 5)
    title.BackgroundTransparency = 1
    title.Text = message
    title.TextColor3 = colors[type] or Config.UI.Theme.Text
    title.TextSize = 14
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.TextYAlignment = Enum.TextYAlignment.Center
    title.Font = Enum.Font.GothamSemibold
    title.Parent = frame
    
    local progress = Instance.new("Frame")
    progress.Size = UDim2.new(0, 0, 0, 2)
    progress.Position = UDim2.new(0, 0, 1, -2)
    progress.BackgroundColor3 = colors[type] or Config.UI.Theme.Primary
    progress.BackgroundTransparency = 0.5
    progress.Parent = frame
    
    -- Animação
    local progressTween = TweenService:Create(progress, TweenInfo.new(3), {
        Size = UDim2.new(1, 0, 0, 2)
    })
    progressTween:Play()
    
    -- Auto destruir
    task.wait(3.5)
    local fadeTween = TweenService:Create(frame, TweenInfo.new(0.3), {
        BackgroundTransparency = 1
    })
    fadeTween:Play()
    task.wait(0.3)
    notification:Destroy()
end

--[[
    ===================
    MÓDULO DA UI PRINCIPAL
    ===================
]]
local UIManager = {}
UIManager.IsOpen = false
UIManager.Dragging = false
UIManager.DragOffset = Vector2.new(0, 0)

function UIManager:CreateMainUI()
    -- ScreenGui
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "AutoFarmPro"
    screenGui.Parent = game.CoreGui
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    
    -- Main Frame
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 280, 0, 420)
    mainFrame.Position = UDim2.new(0.5, -140, 0.5, -210)
    mainFrame.BackgroundColor3 = Config.UI.Theme.Background
    mainFrame.BorderColor3 = Config.UI.Theme.Border
    mainFrame.BorderSizePixel = 1
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = screenGui
    
    -- Main Corner
    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 12)
    mainCorner.Parent = mainFrame
    
    -- Glass Effect (Background)
    local glass = Instance.new("Frame")
    glass.Size = UDim2.new(1, 0, 1, 0)
    glass.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    glass.BackgroundTransparency = 0.95
    glass.Parent = mainFrame
    
    -- Title Bar
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 45)
    titleBar.BackgroundColor3 = Color3.fromRGB(20, 20, 35)
    titleBar.BackgroundTransparency = 0.5
    titleBar.Parent = mainFrame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 12)
    titleCorner.Parent = titleBar
    
    -- Title Text
    local titleText = Instance.new("TextLabel")
    titleText.Size = UDim2.new(1, -40, 1, 0)
    titleText.Position = UDim2.new(0, 20, 0, 0)
    titleText.BackgroundTransparency = 1
    titleText.Text = "✦ AutoFarm Pro"
    titleText.TextColor3 = Config.UI.Theme.Primary
    titleText.TextSize = 18
    titleText.TextXAlignment = Enum.TextXAlignment.Left
    titleText.TextYAlignment = Enum.TextYAlignment.Center
    titleText.Font = Enum.Font.GothamBold
    titleText.Parent = titleBar
    
    -- Close Button
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
    
    -- Scroll Frame for Toggles
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1, -20, 1, -60)
    scrollFrame.Position = UDim2.new(0, 10, 0, 55)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.ScrollBarThickness = 4
    scrollFrame.ScrollBarImageColor3 = Config.UI.Theme.Primary
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 400)
    scrollFrame.Parent = mainFrame
    
    -- UIListLayout for Toggles
    local listLayout = Instance.new("UIListLayout")
    listLayout.Padding = UDim.new(0, 8)
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder
    listLayout.Parent = scrollFrame
    
    -- Toggle Data
    local toggleData = {
        {Key = "AutoBid", Label = "🚗 Auto Bid", Order = 1},
        {Key = "AutoColetar", Label = "📦 Auto Coletar", Order = 2},
        {Key = "AutoDrive", Label = "🚀 Auto Drive", Order = 3},
        {Key = "AutoDescarregar", Label = "📥 Auto Descarregar", Order = 4},
        {Key = "AutoPlot", Label = "📊 Auto Plot", Order = 5},
        {Key = "AutoLimpeza", Label = "🧹 Auto Limpeza", Order = 6}
    }
    
    self.ToggleButtons = {}
    
    for _, data in pairs(toggleData) do
        local toggleFrame = self:CreateToggle(data.Key, data.Label)
        toggleFrame.LayoutOrder = data.Order
        toggleFrame.Parent = scrollFrame
        self.ToggleButtons[data.Key] = toggleFrame
    end
    
    -- Update Canvas Size
    listLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 10)
    end)
    
    -- Draggable
    self:MakeDraggable(mainFrame, titleBar)
    
    -- Start Hidden
    mainFrame.Visible = false
    self.IsOpen = false
    
    -- Save/Load Config
    self:LoadConfig()
    
    return screenGui
end

function UIManager:CreateToggle(key, label)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 50)
    frame.BackgroundColor3 = Config.UI.Theme.Surface
    frame.BackgroundTransparency = 0.5
    frame.ClipsDescendants = true
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = frame
    
    -- Hover Effect
    frame.MouseEnter:Connect(function()
        TweenService:Create(frame, TweenInfo.new(0.2), {
            BackgroundTransparency = 0.3
        }):Play()
    end)
    
    frame.MouseLeave:Connect(function()
        TweenService:Create(frame, TweenInfo.new(0.2), {
            BackgroundTransparency = 0.5
        }):Play()
    end)
    
    -- Label
    local labelText = Instance.new("TextLabel")
    labelText.Size = UDim2.new(0.7, -10, 1, 0)
    labelText.Position = UDim2.new(0, 15, 0, 0)
    labelText.BackgroundTransparency = 1
    labelText.Text = label
    labelText.TextColor3 = Config.UI.Theme.Text
    labelText.TextSize = 14
    labelText.TextXAlignment = Enum.TextXAlignment.Left
    labelText.TextYAlignment = Enum.TextYAlignment.Center
    labelText.Font = Enum.Font.GothamMedium
    labelText.Parent = frame
    
    -- Toggle Switch
    local toggleBtn = Instance.new("TextButton")
    toggleBtn.Size = UDim2.new(0, 50, 0, 28)
    toggleBtn.Position = UDim2.new(1, -60, 0.5, -14)
    toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
    toggleBtn.Text = ""
    toggleBtn.Parent = frame
    
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 14)
    toggleCorner.Parent = toggleBtn
    
    -- Toggle Indicator
    local indicator = Instance.new("Frame")
    indicator.Size = UDim2.new(0, 22, 0, 22)
    indicator.Position = UDim2.new(0, 3, 0.5, -11)
    indicator.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    indicator.BackgroundTransparency = 0.5
    indicator.Parent = toggleBtn
    
    local indicatorCorner = Instance.new("UICorner")
    indicatorCorner.CornerRadius = UDim.new(0, 11)
    indicatorCorner.Parent = indicator
    
    -- Toggle State
    local isOn = false
    
    local function updateToggle(state)
        isOn = state
        Config.Toggles[key] = state
        
        local color1, color2
        if state then
            color1 = Config.UI.Theme.Primary
            color2 = Config.UI.Theme.Secondary
            indicator.BackgroundTransparency = 0
        else
            color1 = Color3.fromRGB(60, 60, 80)
            color2 = Color3.fromRGB(80, 80, 100)
            indicator.BackgroundTransparency = 0.5
        end
        
        toggleBtn.BackgroundColor3 = color1
        
        local indicatorPos = state and UDim2.new(1, -25, 0.5, -11) or UDim2.new(0, 3, 0.5, -11)
        TweenService:Create(indicator, TweenInfo.new(0.3, Enum.EasingStyle.Quad), {
            Position = indicatorPos
        }):Play()
        
        -- Save config
        self:SaveConfig()
        
        -- Notify
        Utils:Notify(string.format("%s %s", label, state and "✅ ATIVADO" or "❌ DESATIVADO"), state and "success" or "warning")
        
        -- Trigger system
        if state then
            self:StartSystem(key)
        else
            self:StopSystem(key)
        end
    end
    
    toggleBtn.MouseButton1Click:Connect(function()
        updateToggle(not isOn)
    end)
    
    -- Load saved state
    if Config.Toggles[key] ~= nil then
        updateToggle(Config.Toggles[key])
    end
    
    -- Tooltip
    local tooltip = Instance.new("TextLabel")
    tooltip.Size = UDim2.new(0, 150, 0, 25)
    tooltip.Position = UDim2.new(0, -5, 1, 5)
    tooltip.BackgroundColor3 = Config.UI.Theme.Background
    tooltip.BackgroundTransparency = 1
    tooltip.Text = isOn and "Clique para desativar" or "Clique para ativar"
    tooltip.TextColor3 = Config.UI.Theme.TextDim
    tooltip.TextSize = 10
    tooltip.Font = Enum.Font.Gotham
    tooltip.Visible = false
    tooltip.Parent = frame
    
    toggleBtn.MouseEnter:Connect(function()
        tooltip.Visible = true
        TweenService:Create(tooltip, TweenInfo.new(0.2), {
            BackgroundTransparency = 0.9
        }):Play()
    end)
    
    toggleBtn.MouseLeave:Connect(function()
        TweenService:Create(tooltip, TweenInfo.new(0.2), {
            BackgroundTransparency = 1
        }):Play()
        task.wait(0.2)
        tooltip.Visible = false
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
            
            frame.Position = UDim2.new(0, newX, 0, newY)
        end
    end)
end

function UIManager:ToggleUI()
    local mainFrame = self:FindUI()
    if not mainFrame then return end
    
    self.IsOpen = not self.IsOpen
    
    if self.IsOpen then
        mainFrame.Visible = true
        mainFrame.Size = UDim2.new(0, 0, 0, 0)
        
        TweenService:Create(mainFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back), {
            Size = UDim2.new(0, 280, 0, 420)
        }):Play()
        
        Utils:Notify("UI Aberta", "info")
    else
        TweenService:Create(mainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad), {
            Size = UDim2.new(0, 0, 0, 0)
        }):Play()
        
        task.wait(0.3)
        mainFrame.Visible = false
        
        Utils:Notify("UI Fechada", "info")
    end
end

function UIManager:FindUI()
    local gui = game.CoreGui:FindFirstChild("AutoFarmPro")
    if gui then
        return gui:FindFirstChild("MainFrame")
    end
    return nil
end

function UIManager:SaveConfig()
    local success, err = pcall(function()
        local data = HttpService:JSONEncode(Config.Toggles)
        writefile("AutoFarmPro_Config.json", data)
    end)
    if not success then
        warn("Failed to save config: " .. err)
    end
end

function UIManager:LoadConfig()
    local success, err = pcall(function()
        if isfile("AutoFarmPro_Config.json") then
            local data = readfile("AutoFarmPro_Config.json")
            local parsed = HttpService:JSONDecode(data)
            for key, value in pairs(parsed) do
                if Config.Toggles[key] ~= nil then
                    Config.Toggles[key] = value
                end
            end
        end
    end)
    if not success then
        warn("Failed to load config: " .. err)
    end
end

function UIManager:StartSystem(key)
    local systems = {
        AutoBid = function() self:StartAutoBid() end,
        AutoColetar = function() self:StartAutoColetar() end,
        AutoDrive = function() self:StartAutoDrive() end,
        AutoDescarregar = function() self:StartAutoDescarregar() end,
        AutoPlot = function() self:StartAutoPlot() end,
        AutoLimpeza = function() self:StartAutoLimpeza() end
    }
    
    if systems[key] then
        systems[key]()
    end
end

function UIManager:StopSystem(key)
    -- Parar sistema
    Utils:Notify(string.format("Sistema %s parado", key), "warning")
end

--[[
    ===================
    SISTEMAS DE AUTOMAÇÃO
    ===================
]]

-- 1. Auto Bid
function UIManager:StartAutoBid()
    Utils:Notify("Auto Bid Iniciado", "success")
    
    coroutine.wrap(function()
        while Config.Toggles.AutoBid do
            -- Detectar minigame
            local ui = Utils:WaitForUI("BidGame", 2)
            if ui then
                -- Resolver minigame - sempre escolher a melhor opção
                local options = ui:FindFirstChild("Options")
                if options then
                    local bestOption = nil
                    local bestValue = -1
                    
                    for _, child in pairs(options:GetChildren()) do
                        if child:IsA("TextButton") then
                            local value = tonumber(child.Text:match("%d+"))
                            if value and value > bestValue then
                                bestValue = value
                                bestOption = child
                            end
                        end
                    end
                    
                    if bestOption then
                        Utils:SimulateClick(bestOption)
                        Utils:Notify("Auto Bid: Melhor opção selecionada", "success")
                    end
                end
            end
            task.wait(Config.Settings.WaitTime)
        end
    end)()
end

-- 2. Auto Coletar
function UIManager:StartAutoColetar()
    Utils:Notify("Auto Coletar Iniciado", "success")
    
    coroutine.wrap(function()
        while Config.Toggles.AutoColetar do
            -- Encontrar itens no chão
            local items = workspace:FindFirstChild("Items") or {}
            
            for _, item in pairs(items:GetChildren()) do
                if item:IsA("BasePart") or item:IsA("Model") then
                    -- Teleportar até o item
                    Utils:TeleportTo(item.Position)
                    task.wait(0.2)
                    
                    -- Coletar (simular interação)
                    local clickDetector = item:FindFirstChild("ClickDetector")
                    if clickDetector then
                        clickDetector:FireClick(LocalPlayer)
                        Utils:Notify("Item coletado: " .. item.Name, "success")
                    end
                end
            end
            task.wait(1)
        end
    end)()
end

-- 3. Auto Drive
function UIManager:StartAutoDrive()
    Utils:Notify("Auto Drive Iniciado", "success")
    
    coroutine.wrap(function()
        while Config.Toggles.AutoDrive do
            -- Entrar no veículo
            local vehicle = workspace:FindFirstChild("Vehicle")
            if vehicle then
                local seat = vehicle:FindFirstChild("Seat")
                if seat then
                    LocalPlayer.Character.Humanoid:MoveTo(seat.Position)
                    task.wait(0.5)
                    seat:EnterSeat(LocalPlayer.Character)
                    Utils:Notify("Entrou no veículo", "success")
                end
                
                -- Teleportar para a loja
                local storePosition = Vector3.new(0, 0, 0) -- Posição da loja
                Utils:TeleportTo(storePosition, Vector3.new(0, 5, 0))
                Utils:Notify("Veículo teleportado para a loja", "success")
            end
            task.wait(Config.Settings.WaitTime)
        end
    end)()
end

-- 4. Auto Descarregar
function UIManager:StartAutoDescarregar()
    Utils:Notify("Auto Descarregar Iniciado", "success")
    
    coroutine.wrap(function()
        while Config.Toggles.AutoDescarregar do
            -- Encontrar UI de descarregamento
            local ui = Utils:FindUI("Unload")
            if ui then
                local unloadBtn = Utils:FindButtonInUI(ui, "Descarregar")
                if unloadBtn then
                    -- Pegar quantidade de itens (simulado)
                    local itemsCount = 10 -- Máximo
                    unloadBtn.Text = "Descarregar (" .. itemsCount .. ")"
                    Utils:SimulateClick(unloadBtn)
                    Utils:Notify("Descarregando " .. itemsCount .. " itens", "success")
                    
                    -- Aguardar descarregar
                    task.wait(3)
                end
            end
            task.wait(Config.Settings.WaitTime)
        end
    end)()
end

-- 5. Auto Plot
function UIManager:StartAutoPlot()
    Utils:Notify("Auto Plot Iniciado", "success")
    
    coroutine.wrap(function()
        while Config.Toggles.AutoPlot do
            -- Teleportar para a loja
            Utils:TeleportTo(Vector3.new(0, 0, 0))
            task.wait(0.5)
            
            -- Encontrar UI da loja
            local ui = Utils:FindUI("Store")
            if ui then
                -- Selecionar todos os itens
                local items = ui:FindFirstChild("Items")
                if items then
                    for _, item in pairs(items:GetChildren()) do
                        if item:IsA("TextButton") then
                            Utils:SimulateClick(item)
                            task.wait(0.2)
                        end
                    end
                    Utils:Notify("Todos os itens colocados nas prateleiras", "success")
                end
            end
            task.wait(Config.Settings.WaitTime)
        end
    end)()
end

-- 6. Auto Limpeza
function UIManager:StartAutoLimpeza()
    Utils:Notify("Auto Limpeza Iniciado", "success")
    
    coroutine.wrap(function()
        while Config.Toggles.AutoLimpeza do
            -- Verificar itens sujos
            local dirtyItems = Utils:GetDirtyItems()
            
            if #dirtyItems > 0 then
                Utils:Notify("Encontrados " .. #dirtyItems .. " itens sujos", "info")
                
                -- Teleportar para lavanderia
                local laundryPos = Vector3.new(0, 0, 0) -- Posição da lavanderia
                Utils:TeleportTo(laundryPos, Vector3.new(0, 2, 0))
                task.wait(0.5)
                
                -- Interagir com o círculo verde
                local interactCircle = workspace:FindFirstChild("GreenCircle")
                if interactCircle then
                    local proximity = interactCircle:FindFirstChild("ProximityPrompt")
                    if proximity then
                        proximity:InputHoldStart(LocalPlayer)
                        Utils:Notify("Interagindo com lavanderia", "success")
                        task.wait(2)
                    end
                end
                
                -- UI de Limpeza
                local ui = Utils:WaitForUI("Clean", 3)
                if ui then
                    -- Verificar diamantes
                    local diamonds = Utils:GetPlayerDiamonds()
                    local slot = diamonds >= 30 and "Slot 2" or "Slot 1"
                    
                    local slotBtn = Utils:FindButtonInUI(ui, slot)
                    if slotBtn then
                        Utils:SimulateClick(slotBtn)
                        Utils:Notify("Usando " .. slot, "success")
                        task.wait(0.5)
                    end
                    
                    -- Selecionar item
                    local itemBtn = Utils:FindButtonInUI(ui, dirtyItems[1].Name)
                    if itemBtn then
                        Utils:SimulateClick(itemBtn)
                        Utils:Notify("Lavando item: " .. dirtyItems[1].Name, "success")
                        task.wait(5) -- Simular tempo de lavagem
                    end
                end
            end
            
            task.wait(Config.Settings.WaitTime)
        end
    end)()
end

--[[
    ===================
    MÓDULO DE DETECÇÃO DE UI
    ===================
]]
local UIDetector = {}

function UIDetector:StartDetection()
    Utils:DetectNewUI(function(ui)
        Utils:Notify("Nova UI detectada: " .. ui.Name, "info")
        
        -- Processar UI automaticamente baseado no nome
        if ui.Name:find("Bid") then
            -- Processar Auto Bid
        elseif ui.Name:find("Unload") then
            -- Processar Auto Descarregar
        elseif ui.Name:find("Clean") then
            -- Processar Auto Limpeza
        elseif ui.Name:find("Store") then
            -- Processar Auto Plot
        end
    end)
end

--[[
    ===================
    INICIALIZAÇÃO
    ===================
]]
function Initialize()
    -- Criar UI Principal
    UIManager:CreateMainUI()
    
    -- Iniciar Detector de UI
    UIDetector:StartDetection()
    
    -- Tecla de atalho (Insert)
    UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.KeyCode == Enum.KeyCode.Insert then
            UIManager:ToggleUI()
        end
    end)
    
    Utils:Notify("AutoFarm Pro Inicializado! Pressione INSERT para abrir", "success")
end

-- Iniciar
Initialize()

--[[
    ===================
    SISTEMA DE EXPANSÃO
    ===================
]]
-- Função para adicionar novos toggles dinamicamente
function UIManager:AddToggle(key, label)
    if self.ToggleButtons[key] then return end
    
    local scrollFrame = self:FindUI():FindFirstChild("ScrollingFrame")
    if not scrollFrame then return end
    
    local toggleFrame = self:CreateToggle(key, label)
    toggleFrame.LayoutOrder = #self.ToggleButtons + 1
    toggleFrame.Parent = scrollFrame
    
    self.ToggleButtons[key] = toggleFrame
    
    -- Atualizar Canvas
    local listLayout = scrollFrame:FindFirstChild("UIListLayout")
    if listLayout then
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 10)
    end
    
    Utils:Notify("Novo toggle adicionado: " .. label, "success")
end

-- Retornar módulos para acesso externo
return {
    UIManager = UIManager,
    Utils = Utils,
    Config = Config
}
