-- Configurações Principais
local Settings = {
    -- O que é ativado?
    AimbotEnabled = true,
    EspEnabled = true,
    SpeedEnabled = false, -- Speed é o mais perigoso, use com moderação
    SpeedMultiplier = 1.5,
    AimbotSmoothness = 0.15, -- Valor baixo = mais suave (mais humano)
    EspColor = Color3.fromRGB(255, 50, 50),
    AimbotDistance = 200,
    MouseSensitivity = 0.01 -- Ajuste para o seu mouse
}

-- Variáveis Globais
local Player = game.Players.LocalPlayer
local Character = Player.Character or Player.CharacterAdded:Wait()
local HumanoidRootPart = Character:FindFirstChildWhichIsA("HumanoidRootPart")
local Humanoid = Character:FindFirstChildWhichIsA("Humanoid")
local Mouse = Player:GetMouse()
local RunService = game:GetService("RunService")

-- Armazena o último input do mouse para humanizar
local LastMousePosition = Vector2.new(0, 0)
local Camera = workspace.CurrentCamera

-- ==========================================
-- SISTEMA DE PAINEL (UI)
-- ==========================================
local UIService = game:GetService("UserInputService")
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AimbotMenu"
ScreenGui.Parent = Player.PlayerGui

-- Criar Frame do Painel
local PanelFrame = Instance.new("Frame")
PanelFrame.Size = UDim2.new(0, 250, 0, 500)
PanelFrame.Position = UDim2.new(0, 20, 0, 20)
PanelFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
PanelFrame.BorderColor3 = Color3.fromRGB(0, 0, 0)
PanelFrame.BorderSizePixel = 0
PanelFrame.BackgroundTransparency = 0
PanelFrame.Name = "MainPanel"
PanelFrame.Parent = ScreenGui

-- Título
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundTransparency = 1
Title.Text = "⚔️ FLOW BOT PANEL ⚔️"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.GothamBold
Title.Size = UDim2.new(1, -10, 0, 30)
Title.Position = UDim2.new(0, 5, 0, 5)
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = PanelFrame

-- Função para criar caixas de toggle/numberinput
local function createControl(name, type, defaultValue, label, color)
    local Container = Instance.new("Frame")
    Container.Size = UDim2.new(1, -20, 0, 40)
    Container.Position = UDim2.new(0, 0, 0, 10)
    Container.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
    Container.BorderSizePixel = 0
    Container.Parent = PanelFrame

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, 0, 1, 0)
    Label.BackgroundTransparency = 1
    Label.Text = label
    Label.TextColor3 = color
    Label.Font = Enum.Font.Gotham
    Label.Parent = Container

    if type == "Toggle" then
        local Toggle = Instance.new("TextButton")
        Toggle.Size = UDim2.new(1, -20, 0, 25)
        Toggle.Position = UDim2.new(0, 20, 0, 5)
        Toggle.BackgroundColor3 = defaultValue and Color3.fromRGB(50, 200, 100) or Color3.fromRGB(100, 100, 100)
        Toggle.Text = "ON"
        Toggle.TextColor3 = Color3.fromRGB(255, 255, 255)
        Toggle.Parent = Container
        
        -- Conectar ao Toggle
        Container.Name = name
        Container.DefaultValue = defaultValue
        Container.Selected = defaultValue
        
        local function UpdateToggle(value)
            Container.BackgroundColor3 = value and Color3.fromRGB(50, 200, 100) or Color3.fromRGB(100, 100, 100)
            Container.Text = value and "ON" or "OFF"
        end
        Toggle.MouseButton1Click:Connect(function()
            Container.Selected = not Container.Selected
            UpdateToggle(Container.Selected)
            Settings[name .. "Enabled"] = Container.Selected
        end)
        
        return Container
    elseif type == "NumberInput" then
        local Number = Instance.new("NumberInput")
        Number.Position = UDim2.new(0, 20, 0, 10)
        Number.Size = UDim2.new(1, -20, 1, 0)
        Number.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
        Number.Text = tostring(defaultValue)
        Number.Font = Enum.Font.Gotham
        Number.Parent = Container
        Container.Name = name
        Number.TextChanged:Connect(function()
            Settings[name .. "Value"] = Number.Value
        end)
        return Container
    end
end

-- Criar Controles no Painel
local AimbotBox = createControl("Aimbot", "Toggle", Settings.AimbotEnabled, "Aimbot", Color3.fromRGB(255, 255, 255))
local EspBox = createControl("Esp", "Toggle", Settings.EspEnabled, "Esp", Color3.fromRGB(255, 255, 255))
local SpeedBox = createControl("Speed", "Toggle", Settings.SpeedEnabled, "Speed (Risco!)", Color3.fromRGB(255, 215, 0))

-- Input de Sensibilidade
local Sensitivity = Instance.new("Frame")
Sensitivity.Size = UDim2.new(1, -20, 0, 40)
Sensitivity.Position = UDim2.new(0, 0, 0, 60)
Sensitivity.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
Sensitivity.BorderSizePixel = 0
Sensitivity.Parent = PanelFrame

local SensLabel = Instance.new("TextLabel")
SensLabel.Size = UDim2.new(1, 0, 1, 0)
SensLabel.BackgroundTransparency = 1
SensLabel.Text = "Mouse Sensitivity:"
SensLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
SensLabel.Font = Enum.Font.Gotham
SensLabel.Parent = Sensitivity

local SensNumber = Instance.new("NumberInput")
SensNumber.Position = UDim2.new(0, 10, 0, 5)
SensNumber.Size = UDim2.new(1, -10, 1, 0)
SensNumber.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
SensNumber.Text = tostring(Settings.MouseSensitivity)
SensNumber.Font = Enum.Font.Gotham
SensNumber.Parent = Sensitivity
SensNumber.Name = "SensitivityInput"

SensNumber.TextChanged:Connect(function()
    Settings.MouseSensitivity = SensNumber.Value
end)


-- ==========================================
-- LÓGICA DO AIMBOT
-- ==========================================
local function GetNearestEnemy()
    local enemies = workspace:FindFirstChild("Enemies") or {} -- Ajuste conforme a estrutura do jogo (ex: Teams)
    if not enemies then return end
    
    local Root = HumanoidRootPart.Position
    local NearestPos = nil
    local MinDistance = Settings.AimbotDistance
    
    for _, enemy in ipairs(enemies:GetChildren()) do
        local EnemyRoot = enemy:FindFirstChildWhichIsA("HumanoidRootPart")
        if EnemyRoot then
            local Dist = Root:DistanceTo(EnemyRoot.Position)
            if Dist < MinDistance then
                MinDistance = Dist
                NearestPos = EnemyRoot.Position
            end
        end
    end
    
    -- Se o jogo for baseado em Teams ou Partes soltas
    local AllParts = workspace:GetChildren()
    for _, part in ipairs(AllParts) do
        if part:IsA("BasePart") and part.Parent ~= workspace and part.Parent.Parent ~= HumanoidRootPart then
            -- Verifica se a part é um inimigo (ex: tem Humanoid ou cor vermelha)
            if part.Parent.Parent and part.Parent.Parent:FindFirstChildWhichIsA("Humanoid") then
                local Dist = Root:DistanceTo(part.Position)
                if Dist < MinDistance and not (part.Parent.Parent == Player.Character) then
                    MinDistance = Dist
                    NearestPos = part.Position
                end
            end
        end
    end
    
    return NearestPos
end

-- Lógica de MouseHumanize (Para parecer que você está jogando)
local function HumanizeMouse(targetPos)
    local Target = Camera:WorldToViewportPoint(targetPos)
    local Delta = Target - Mouse.Position
    
    -- Curva de movimento suave (Lerp simples para suavizar)
    local NewPosition = Mouse.Position + (Delta * Settings.AimbotSmoothness)
    return NewPosition
end

local function ApplyAimbot()
    local TargetPos = GetNearestEnemy()
    if TargetPos then
        local Target = Camera:WorldToViewportPoint(TargetPos)
        
        if TargetPos and Target then
            -- Aplica o movimento suave
            Mouse.Position = HumanizeMouse(TargetPos)
        end
    end
end

-- ==========================================
-- LÓGICA DO ESP
-- ==========================================
local function DrawEsp()
    local parts = workspace:GetDescendants()
    
    for _, part in ipairs(parts) do
        if part:IsA("BasePart") and part.Parent and part.Parent.Parent ~= workspace and part.Parent.Parent ~= HumanoidRootPart then
            -- Desenha caixas ao redor dos inimigos
            local Size = part.Size
            local Pos = part.Position
            
            -- Verifica se é inimigo (excluindo aliados ou o próprio)
            if part.Parent.Parent ~= Player.Character then
                local ScreenX, ScreenY = Camera:WorldToViewportPoint(Pos)
                local ScreenSize = Camera:WorldToViewportSize(Size)
                
                -- Só desenha se estiver na tela
                if ScreenX > 0 and ScreenX < Camera.ViewSizeX and ScreenY > 0 and ScreenY < Camera.ViewSizeY then
                    local Box = Instance.new("Frame")
                    Box.Size = UDim2.new(0, ScreenSize.X * 2, 0, ScreenSize.Y * 2)
                    Box.Position = UDim2.new(0, ScreenX - (ScreenSize.X), 0, ScreenY - (ScreenSize.Y))
                    Box.BackgroundColor3 = Settings.EspColor
                    Box.BackgroundTransparency = 0.7
                    Box.Name = "EspBox"
                    Box.Parent = Camera
                end
            end
        end
    end
end

-- ==========================================
-- LÓGICA DE SPEED
-- ==========================================
-- Nota: Speed real altera a velocidade de movimento local.
-- Para ser "bypass", o ideal é alterar apenas o cliente e ter um pequeno delay na aplicação.
local function ApplySpeed()
    if not Settings.SpeedEnabled or not HumanoidRootPart then return end
    
    -- Cria uma cópia da posição anterior
    local OldPos = HumanoidRootPart.CFrame
    
    -- Move a parte de forma suave para simular velocidade, sem mudar o CFrame do jogo original
    -- Isso ajuda a enganar alguns detectores simples, mas avançados ainda vão flaggar.
    local ForwardVector = HumanoidRootPart.CFrame.LookVector
    local NewPos = HumanoidRootPart.Position + (ForwardVector * (Settings.SpeedMultiplier - 1))
    
    HumanoidRootPart.Position = NewPos
    HumanoidRootPart.Velocity = (HumanoidRootPart.Velocity * Settings.SpeedMultiplier)
end

-- ==========================================
-- LOOP PRINCIPAL
-- ==========================================
RunService.RenderStepped:Connect(function()
    -- Lógica do Painel
    if AimbotBox and AimbotBox.Selected then
        ApplyAimbot()
    end
    
    if EspBox and EspBox.Selected then
        DrawEsp()
    end
    
    if SpeedBox and SpeedBox.Selected then
        ApplySpeed()
    end
end)

-- Ajuste de Sensibilidade do Mouse
Mouse.Sensitivity = Settings.MouseSensitivity

print("Bot Painel Iniciado com Sucesso!")
