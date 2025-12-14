
local KEYAUTH_CONFIG = {
    Name = "ScriptRoblox",
    OwnerID = "tnrXMCnv1p",
    Secret = "526a68fcee23ec4df14badb3e5a4026aa265c03e7e1fbc08f5106e1448fc4828",
    Version = "1.0",
    API_URL = "https://keyauth.win/api/1.2/"
}

-- ===============================================
-- 2. INICIALIZAÇÃO DE SERVIÇOS ROBLOX
-- ===============================================
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService") -- NECESSÁRIO PARA KEYAUTH
local LocalPlayer = Players.LocalPlayer
local Workspace = game:GetService("Workspace")
local camera = Workspace.CurrentCamera
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CoreGui = game:GetService("CoreGui")

if not HttpService.HttpEnabled then
    warn("[KeyAuth] HttpService não está ativado. O script não pode ser executado.")
    return
end

-- Variáveis de controle de execução
local IsAuthenticated = false
local KeyAuthGui 

-- ===============================================
-- 3. FUNÇÃO DE VALIDAÇÃO DA CHAVE KEYAUTH
-- ===============================================

local function ValidateKey(key_input)
    local full_url = string.format(
        "%s?type=license&key=%s&name=%s&ownerid=%s&secret=%s&version=%s&hwid=%s",
        KEYAUTH_CONFIG.API_URL,
        HttpService:UrlEncode(key_input),
        HttpService:UrlEncode(KEYAUTH_CONFIG.Name),
        HttpService:UrlEncode(KEYAUTH_CONFIG.OwnerID),
        HttpService:UrlEncode(KEYAUTH_CONFIG.Secret),
        HttpService:UrlEncode(KEYAUTH_CONFIG.Version),
        HttpService:UrlEncode(LocalPlayer.UserId)
    )

    local success, response = pcall(function()
        return HttpService:GetAsync(full_url)
    end)

    if success then
        local data
        local decodeSuccess, decodedData = pcall(function()
            return HttpService:JSONDecode(response)
        end)

        if decodeSuccess and decodedData then
            data = decodedData
        else
             return false, "Resposta API inválida."
        end
        
        if data.success then
            return true, data
        else
            return false, data.message
        end
    else
        warn("[KeyAuth] Erro na Requisição HTTP:", response)
        return false, "Erro de conexão HTTP."
    end
end

-- ===============================================
-- 4. UI MODERNA DE LOGIN (KEYAUTH)
-- ===============================================

local function SetupModernKeyUI()
    -- Cria a ScreenGui no CoreGui e a protege
    KeyAuthGui = Instance.new("ScreenGui", CoreGui)
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(KeyAuthGui) end end)
    KeyAuthGui.Name = "KeyAuthLoader"
    
    -- FRAME PRINCIPAL (MODERNO)
    local MainFrame = Instance.new("Frame", KeyAuthGui)
    MainFrame.Size = UDim2.new(0, 350, 0, 200)
    MainFrame.Position = UDim2.new(0.5, -175, 0.5, -100)
    MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25) 
    MainFrame.BorderSizePixel = 0
    
    local FrameCorner = Instance.new("UICorner", MainFrame)
    FrameCorner.CornerRadius = UDim.new(0, 15)
    
    local FrameStroke = Instance.new("UIStroke", MainFrame)
    FrameStroke.Color = Color3.fromRGB(0, 150, 255)
    FrameStroke.Thickness = 2
    FrameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

    -- TÍTULO
    local Title = Instance.new("TextLabel", MainFrame)
    Title.Text = KEYAUTH_CONFIG.Name .. " - AUTENTICAÇÃO"
    Title.Size = UDim2.new(1, 0, 0, 40)
    Title.Position = UDim2.new(0, 0, 0, 10)
    Title.BackgroundTransparency = 1
    Title.TextColor3 = Color3.fromRGB(255, 255, 255)
    Title.Font = Enum.Font.GothamBold
    Title.TextSize = 20

    -- CAMPO DA CHAVE
    local KeyTextBox = Instance.new("TextBox", MainFrame)
    KeyTextBox.PlaceholderText = "Insira sua Chave de Licença"
    KeyTextBox.Text = ""
    KeyTextBox.Size = UDim2.new(0.8, 0, 0, 35)
    KeyTextBox.Position = UDim2.new(0.5, 0, 0.5, -17) 
    KeyTextBox.AnchorPoint = Vector2.new(0.5, 0.5)
    KeyTextBox.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    KeyTextBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    KeyTextBox.Font = Enum.Font.SourceSans
    KeyTextBox.TextSize = 16
    
    local KeyCorner = Instance.new("UICorner", KeyTextBox)
    KeyCorner.CornerRadius = UDim.new(0, 8)

    -- MENSAGEM DE STATUS
    local StatusLabel = Instance.new("TextLabel", MainFrame)
    StatusLabel.Text = "Aguardando chave..."
    StatusLabel.Size = UDim2.new(0.8, 0, 0, 20)
    StatusLabel.Position = UDim2.new(0.5, 0, 0.7, -5)
    StatusLabel.AnchorPoint = Vector2.new(0.5, 0.5)
    StatusLabel.BackgroundTransparency = 1
    StatusLabel.TextColor3 = Color3.fromRGB(150, 150, 170)
    StatusLabel.Font = Enum.Font.SourceSans
    StatusLabel.TextSize = 14

    -- BOTÃO DE LOGIN
    local LoginButton = Instance.new("TextButton", MainFrame)
    LoginButton.Text = "ENTRAR"
    LoginButton.Size = UDim2.new(0.8, 0, 0, 40)
    LoginButton.Position = UDim2.new(0.5, 0, 0.85, 0)
    LoginButton.AnchorPoint = Vector2.new(0.5, 0)
    LoginButton.BackgroundColor3 = Color3.fromRGB(0, 150, 255) 
    LoginButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    LoginButton.Font = Enum.Font.GothamBold
    LoginButton.TextSize = 18
    
    local ButtonCorner = Instance.new("UICorner", LoginButton)
    ButtonCorner.CornerRadius = UDim.new(0, 10)

    -- LÓGICA DO BOTÃO
    LoginButton.MouseButton1Click:Connect(function()
        local key = KeyTextBox.Text:gsub("%s+", "")
        if key == "" then return end

        LoginButton.Text = "VALIDANDO..."
        LoginButton.Active = false
        StatusLabel.Text = "Conectando ao KeyAuth..."
        StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 255) 

        coroutine.wrap(function()
            local success, result = ValidateKey(key)

            if success then
                StatusLabel.Text = "Autenticado! Iniciando..."
                StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 0) 
                IsAuthenticated = true
                wait(1)
                KeyAuthGui:Destroy()
                
            else
                -- CHAVE INVÁLIDA
                StatusLabel.TextColor3 = Color3.fromRGB(255, 50, 50) 
                StatusLabel.Text = "KEY INVALIDA!" 
                
                LoginButton.Text = "TENTAR NOVAMENTE"
                LoginButton.Active = true
            end
        end)()
    end)
end

-- ===============================================
-- 5. FUNÇÃO DE INÍCIO DO SCRIPT PRINCIPAL
-- (TODO O SEU CÓDIGO ORIGINAL ESTÁ AQUI DENTRO)
-- ===============================================

local function RunMainScript()
    
    -- O SEU CÓDIGO COMEÇA AQUI!

    -- Variáveis de controle de Puxar Itens (Auto Loot)
    local puxarItensToggle = false
    local puxarItensCoroutine
    local itens = {
        "AK47", "Uzi", "PARAFAL", "Glock 17", "Faca", "IA2", "G3", "Dinamite",
        "Hi Power", "Natalina", "HK416", "Lockpick", "Escudo", "Skate",
        "Saco de lixo", "Peca de Arma", "Tratamento", "AR-15", "PS5", "C4", "USP", "Xbox"
    }
    -- Variáveis globais para ESP
    local espNameEnabled = false
    local espTracersEnabled = false
    local espHitboxEnabled = false
    local espFullEnabled = false
    local espVidaEnabled = false 
    local espRGBEnabled = false 
    local espLabels = {}
    local espTracers = {}
    local espHealthBars = {} 

    local safePlayers = {} -- Tabela para armazenar jogadores que o aimbot deve ignorar (DEVE SER ADICIONADA)
    local isAiming = false -- Variável para controlar se o jogador está mirando (DEVE SER ADICIONADA)
    local aimbotConnection = nil -- Variável de conexão para o RenderStepped (DEVE SER ADICIONADA)

    -- GUI principal
    local gui = Instance.new("ScreenGui")
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
    gui.Name = "GNewStore"
    gui.ResetOnSpawn = false
    gui.Parent = CoreGui -- Mudado para CoreGui para consistência

    local menu = Instance.new("Frame")
    menu.Size = UDim2.new(0, 500, 0, 400)
    menu.Position = UDim2.new(0.5, -250, 0.4, -200)
    menu.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    menu.BorderSizePixel = 0
    menu.Active = true
    menu.Draggable = true
    menu.Parent = gui

    -- Borda arredondada
    local menuCorner = Instance.new("UICorner")
    menuCorner.CornerRadius = UDim.new(0, 12)
    menuCorner.Parent = menu

    local border = Instance.new("UIStroke")
    border.Color = Color3.fromRGB(100, 200, 255)
    border.Thickness = 2
    border.Parent = menu

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -40, 0, 40)
    title.Position = UDim2.new(0, 10, 0, 5)
    title.BackgroundTransparency = 1
    title.Text = "G NEW STORE MENU"
    title.TextColor3 = Color3.new(1, 1, 1)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 24
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = menu

    -- Botão fechar
    local closeBtn = Instance.new("TextButton")
    closeBtn.Text = "X"
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.TextSize = 16
    closeBtn.TextColor3 = Color3.new(1, 0.3, 0.3)
    closeBtn.Size = UDim2.new(0, 30, 0, 30)
    closeBtn.Position = UDim2.new(1, -35, 0, 5)
    closeBtn.BackgroundTransparency = 1
    closeBtn.Parent = menu

    -- Botão minimizar
    local minimizeBtn = Instance.new("TextButton")
    minimizeBtn.Text = "_"
    minimizeBtn.Font = Enum.Font.GothamBold
    minimizeBtn.TextSize = 16
    minimizeBtn.TextColor3 = Color3.new(1, 1, 0.3)
    minimizeBtn.Size = UDim2.new(0, 30, 0, 30)
    minimizeBtn.Position = UDim2.new(1, -70, 0, 5)
    minimizeBtn.BackgroundTransparency = 1
    minimizeBtn.Parent = menu

    closeBtn.MouseButton1Click:Connect(function()
    gui:Destroy()
    end)

    -- Botão flutuante "G"
    local reopenButton = Instance.new("TextButton")
    reopenButton.Size = UDim2.new(0, 40, 0, 40)
    reopenButton.Position = UDim2.new(0, 10, 1, -50)
    reopenButton.AnchorPoint = Vector2.new(0, 1)
    reopenButton.BackgroundColor3 = Color3.fromRGB(255, 140, 0)
    reopenButton.Text = "G"
    reopenButton.TextColor3 = Color3.new(1, 1, 1)
    reopenButton.Font = Enum.Font.GothamBold
    reopenButton.TextSize = 20
    reopenButton.Visible = false
    reopenButton.Parent = gui

    local reopenCorner = Instance.new("UICorner")
    reopenCorner.CornerRadius = UDim.new(1, 0)
    reopenCorner.Parent = reopenButton

    local reopenStroke = Instance.new("UIStroke")
    reopenStroke.Color = Color3.fromRGB(100, 200, 255)
    reopenStroke.Thickness = 2
    reopenStroke.Parent = reopenButton

    minimizeBtn.MouseButton1Click:Connect(function()
    menu.Visible = false
    reopenButton.Visible = true
    end)

    reopenButton.MouseButton1Click:Connect(function()
    menu.Visible = true
    reopenButton.Visible = false
    end)

    local tabsContainer = Instance.new("Frame")
    tabsContainer.Size = UDim2.new(1, -20, 0, 40)
    tabsContainer.Position = UDim2.new(0, 10, 0, 50)
    tabsContainer.BackgroundTransparency = 1
    tabsContainer.Parent = menu

    local contentContainer = Instance.new("Frame")
    contentContainer.Size = UDim2.new(1, -40, 1, -110)
    contentContainer.Position = UDim2.new(0, 20, 0, 90)
    contentContainer.BackgroundTransparency = 1
    contentContainer.Parent = menu

    local function createTabButton(name, posX)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 80, 1, 0)
        btn.Position = UDim2.new(0, posX, 0, 0)
        btn.Text = name
        btn.BackgroundTransparency = 1
        btn.TextColor3 = Color3.new(1, 1, 1)
        btn.Font = Enum.Font.Gotham
        btn.TextSize = 18
        btn.Parent = tabsContainer

        local corner = Instance.new("UICorner")  
        corner.CornerRadius = UDim.new(0, 6)  
        corner.Parent = btn  

        btn.MouseEnter:Connect(function()  
            btn.BackgroundTransparency = 0.2  
        end)  
        btn.MouseLeave:Connect(function()  
            btn.BackgroundTransparency = 1  
        end)  

        return btn
    end

    local espFrame = Instance.new("Frame")
    espFrame.Size = UDim2.new(1, 0, 1, 0)
    espFrame.BackgroundTransparency = 1
    espFrame.Parent = contentContainer

    local hitboxFrame = Instance.new("Frame")
    hitboxFrame.Size = UDim2.new(1, 0, 1, 0)
    hitboxFrame.BackgroundTransparency = 1
    hitboxFrame.Visible = false
    hitboxFrame.Parent = contentContainer 
    
    -- FIM DO SEU CÓDIGO ORIGINAL
end


-- ===============================================
-- 6. LOADER (GARANTE A ORDEM DE EXECUÇÃO)
-- ===============================================

SetupModernKeyUI()

-- Trava a execução do script até que IsAuthenticated seja verdadeiro (chave válida)
repeat
    wait(0.5)
until IsAuthenticated

-- Se a chave é válida, executa a função que contém o seu robô e a UI principal.
RunMainScript()

-- FIM DO SCRIPT

local tpFrame = Instance.new("Frame")
tpFrame.Size = UDim2.new(1, 0, 1, 0)
tpFrame.BackgroundTransparency = 1
tpFrame.Visible = false
tpFrame.Parent = contentContainer

-- Label "TP/NICK"
local tpLabel = Instance.new("TextLabel")
tpLabel.Size = UDim2.new(0, 120, 0, 40)
tpLabel.Position = UDim2.new(0, 0, 0, 0)
tpLabel.BackgroundTransparency = 1
tpLabel.Text = "TP/NICK"
tpLabel.Font = Enum.Font.Gotham
tpLabel.TextSize = 20
tpLabel.TextColor3 = Color3.new(1, 1, 1)
tpLabel.TextXAlignment = Enum.TextXAlignment.Left
tpLabel.Parent = tpFrame

-- Input Box
local tpInput = Instance.new("TextBox")
tpInput.Size = UDim2.new(0, 220, 0, 40)
tpInput.Position = UDim2.new(0, 130, 0, 0)
tpInput.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
tpInput.TextColor3 = Color3.new(1, 1, 1)
tpInput.Font = Enum.Font.Gotham
tpInput.TextSize = 18
tpInput.PlaceholderText = "Digite o nick aqui"
tpInput.ClearTextOnFocus = false
tpInput.Parent = tpFrame

local tpInputCorner = Instance.new("UICorner")
tpInputCorner.CornerRadius = UDim.new(0, 6)
tpInputCorner.Parent = tpInput

-- FUNÇÃO: Teleportar para Nick
local function teleportToNick()
    local targetNick = tpInput.Text
    local targetPlayer = Players:FindFirstChild(targetNick)
    
    if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local targetHRP = targetPlayer.Character.HumanoidRootPart
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            LocalPlayer.Character.HumanoidRootPart.CFrame = targetHRP.CFrame * CFrame.new(0, 3, 0) 
            print("Teleportado para: " .. targetNick)
        end
    else
        print("Jogador '" .. targetNick .. "' não encontrado ou sem Character.")
    end
end

-- Botão Enter
local enterBtn = Instance.new("TextButton")
enterBtn.Size = UDim2.new(0, 80, 0, 40)
enterBtn.Position = UDim2.new(0, 360, 0, 0)
enterBtn.BackgroundColor3 = Color3.fromRGB(100, 200, 255)
enterBtn.Text = "Teleportar"
enterBtn.TextColor3 = Color3.new(1, 1, 1)
enterBtn.Font = Enum.Font.GothamBold
enterBtn.TextSize = 18
enterBtn.Parent = tpFrame

local enterBtnCorner = Instance.new("UICorner")
enterBtnCorner.CornerRadius = UDim.new(0, 6)
enterBtnCorner.Parent = enterBtn

enterBtn.AutoButtonColor = true

enterBtn.MouseButton1Click:Connect(teleportToNick)

-- FUNÇÃO: Teleportar para o mais próximo
local function getClosestHRP()
    local localHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not localHRP then return nil end
    
    local closestPlayer = nil
    local closestDistance = math.huge
    
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local targetHRP = player.Character.HumanoidRootPart
            local distance = (localHRP.Position - targetHRP.Position).Magnitude
            
            if distance < closestDistance then
                closestDistance = distance
                closestPlayer = player
            end
        end
    end
    
    return closestPlayer and closestPlayer.Character.HumanoidRootPart
end

local function teleportToClosestPlayer()
    local closestHRP = getClosestHRP()
    local localHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    
    if closestHRP and localHRP then
        localHRP.CFrame = closestHRP.CFrame * CFrame.new(0, 3, 0)
        print("Teleportado para o jogador mais próximo.")
    else
        print("Nenhum outro jogador encontrado.")
    end
end

local tpProximoBtn = Instance.new("TextButton")
tpProximoBtn.Size = UDim2.new(0, 460, 0, 40)
tpProximoBtn.Position = UDim2.new(0, 0, 0, 50)
tpProximoBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
tpProximoBtn.TextColor3 = Color3.new(1, 1, 1)
tpProximoBtn.Font = Enum.Font.Gotham
tpProximoBtn.TextSize = 18
tpProximoBtn.Text = "TP PLAYER PROXIMO"
tpProximoBtn.Parent = tpFrame

local round = Instance.new("UICorner")
round.CornerRadius = UDim.new(0, 6)
round.Parent = tpProximoBtn

tpProximoBtn.MouseButton1Click:Connect(teleportToClosestPlayer)

-- TP/KILL
local function teleportAndKill()
    local closestHRP = getClosestHRP()
    local localHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")

    if closestHRP and localHRP then
        localHRP.CFrame = closestHRP.CFrame * CFrame.new(0, 3, 0)
        print("Teleportado para o jogador. Tentando matar...")
        
        local remoteFolder = ReplicatedStorage:FindFirstChild("RemoteNovos") or ReplicatedStorage
        local attackRemote = remoteFolder:FindFirstChild("DamageRequest") or remoteFolder:FindFirstChild("AttackRemote") 
        
        if attackRemote and attackRemote:IsA("RemoteEvent") then
            pcall(function()
                attackRemote:FireServer(closestHRP.Parent, 1000) 
                print("Tentativa de Kill Remota enviada.")
            end)
        else
            print("Remote de Kill não encontrado. Apenas teleportado.")
        end
    else
        print("Nenhum alvo encontrado para TP/KILL.")
    end
end

local tpKillBtn = Instance.new("TextButton")
tpKillBtn.Size = UDim2.new(0, 460, 0, 40)
tpKillBtn.Position = UDim2.new(0, 0, 0, 100)
tpKillBtn.BackgroundColor3 = Color3.fromRGB(150, 50, 50) 
tpKillBtn.TextColor3 = Color3.new(1, 1, 1)
tpKillBtn.Font = Enum.Font.Gotham
tpKillBtn.TextSize = 18
tpKillBtn.Text = "TP/KILL PROXIMO"
tpKillBtn.Parent = tpFrame

local roundKill = Instance.new("UICorner")
roundKill.CornerRadius = UDim.new(0, 6)
roundKill.Parent = tpKillBtn

tpKillBtn.MouseButton1Click:Connect(teleportAndKill)

local pvpFrame = Instance.new("Frame")
pvpFrame.Size = UDim2.new(1, 0, 1, 0)
pvpFrame.BackgroundTransparency = 1
pvpFrame.Visible = false
pvpFrame.Parent = contentContainer

local flyFrame = Instance.new("Frame")
flyFrame.Size = UDim2.new(1, 0, 1, 0)
flyFrame.BackgroundTransparency = 1
flyFrame.Visible = false
flyFrame.Parent = contentContainer

local espTabBtn = createTabButton("ESP", 0)
local pvpTabBtn = createTabButton("PVP", 90)
local flyTabBtn = createTabButton("Fly", 180)
local hitboxTabBtn = createTabButton("HitboxConfig", 270)
local tpTabBtn = createTabButton("TP'S", 360) 

local function switchTab(tabName)
    espFrame.Visible = tabName == "ESP"
    pvpFrame.Visible = tabName == "PVP"
    flyFrame.Visible = tabName == "Fly"
    hitboxFrame.Visible = tabName == "HitboxConfig"
    tpFrame.Visible = tabName == "TP'S"
    
    espTabBtn.BackgroundColor3 = tabName == "ESP" and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(50, 50, 50)  
    pvpTabBtn.BackgroundColor3 = tabName == "PVP" and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(50, 50, 50)  
    flyTabBtn.BackgroundColor3 = tabName == "Fly" and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(50, 50, 50)
    hitboxTabBtn.BackgroundColor3 = tabName == "HitboxConfig" and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(50, 50, 50)
    tpTabBtn.BackgroundColor3 = tabName == "TP'S" and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(50, 50, 50)
end

espTabBtn.MouseButton1Click:Connect(function()
switchTab("ESP")
end)

pvpTabBtn.MouseButton1Click:Connect(function()
switchTab("PVP")
end)

flyTabBtn.MouseButton1Click:Connect(function()
switchTab("Fly")
 end)

tpTabBtn.MouseButton1Click:Connect(function()
    switchTab("TP'S")
end)

hitboxTabBtn.MouseButton1Click:Connect(function()
    switchTab("HitboxConfig")
end)


local function createButton(parent, text, posY, callback)
local btn = Instance.new("TextButton")
btn.Size = UDim2.new(0, 460, 0, 40)
btn.Position = UDim2.new(0, 0, 0, posY)
btn.Text = text
btn.Font = Enum.Font.Gotham
btn.TextSize = 18
btn.TextColor3 = Color3.new(1, 1, 1)
btn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
btn.AutoButtonColor = true
btn.Parent = parent

local round = Instance.new("UICorner")  
round.CornerRadius = UDim.new(0, 6)  
round.Parent = btn  

btn.MouseButton1Click:Connect(function() callback(btn) end) 

return btn

end
-- ESP Functions --

local function createESPLabel(plr)
    if not plr.Character or not plr.Character:FindFirstChild("Head") then return end
    if espLabels[plr] then return end

    local head = plr.Character.Head  
    local billboard = Instance.new("BillboardGui", head)  
    billboard.Name = "ESPLabel"  
    billboard.Size = UDim2.new(0, 100, 0, 40)  
    billboard.AlwaysOnTop = true  
    billboard.Adornee = head  

    local label = Instance.new("TextLabel", billboard)  
    label.Size = UDim2.new(1, 0, 1, 0)  
    label.BackgroundTransparency = 1  
    label.TextColor3 = Color3.new(1, 0, 0)  
    label.Text = plr.Name  
    label.TextScaled = true  

    espLabels[plr] = billboard
end

local function removeESPLabel(plr)
    if espLabels[plr] then
        espLabels[plr]:Destroy()
        espLabels[plr] = nil
    end
end

local function createTracer(plr)
    if espTracers[plr] then return end
    if not plr.Character or not plr.Character:FindFirstChild("Head") then return end

    local tracer = Drawing.new("Line")  
    tracer.Color = Color3.new(1, 0, 0)  
    tracer.Thickness = 1.5  
    tracer.Transparency = 1  
    espTracers[plr] = tracer
end

local function removeTracer(plr)
    if espTracers[plr] then
        espTracers[plr]:Remove()
        espTracers[plr] = nil
    end
end

local function createHealthBar(plr)
    if not plr.Character or not plr.Character:FindFirstChild("Head") then return end
    if espHealthBars[plr] then return end

    local head = plr.Character.Head  
    local billboard = Instance.new("BillboardGui", head)  
    billboard.Name = "HealthBarESP"  
    billboard.Size = UDim2.new(0, 10, 0, 50)  
    billboard.Adornee = head  
    billboard.AlwaysOnTop = true
    billboard.ExtentsOffset = Vector3.new(-10, 0, 0) 

    local background = Instance.new("Frame", billboard)
    background.Size = UDim2.new(1, 0, 1, 0)
    background.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    background.BackgroundTransparency = 0.5

    local healthFill = Instance.new("Frame", background)
    healthFill.Name = "HealthFill"
    healthFill.Size = UDim2.new(1, 0, 1, 0)
    healthFill.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
    healthFill.AnchorPoint = Vector2.new(0, 1) 
    healthFill.Position = UDim2.new(0, 0, 1, 0)
    
    local humanoid = plr.Character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    
    if humanoid.MaxHealth > 0 then
        healthFill.Size = UDim2.new(1, 0, humanoid.Health / humanoid.MaxHealth, 0)
    end

    local function updateBarColor(health)
        if health > 50 then
            healthFill.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
        elseif health > 20 then
            healthFill.BackgroundColor3 = Color3.fromRGB(255, 255, 0)
        else
            healthFill.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
        end
    end

    updateBarColor(humanoid.Health)

    local connection
    connection = humanoid.HealthChanged:Connect(function(newHealth)
        if espVidaEnabled and humanoid.MaxHealth > 0 then
            healthFill.Size = UDim2.new(1, 0, newHealth / humanoid.MaxHealth, 0)
            updateBarColor(newHealth)
        else
            connection:Disconnect()
        end
    end)


    espHealthBars[plr] = billboard
end

local function removeHealthBar(plr)
    if espHealthBars[plr] then
        espHealthBars[plr]:Destroy()
        espHealthBars[plr] = nil
    end
end

local function enableHitbox(plr)
    if not plr.Character then return end
    for _, part in pairs(plr.Character:GetChildren()) do
        if part:IsA("BasePart") then
            for _, adornment in pairs(part:GetChildren()) do
                if adornment:IsA("BoxHandleAdornment") and adornment.Name == "ESPHitbox" then
                    adornment:Destroy()
                end
            end
        end
    end
    for _, part in pairs(plr.Character:GetChildren()) do
        if part:IsA("BasePart") and part.Massless == false and part.Name ~= "Handle" then
            local box = Instance.new("BoxHandleAdornment")
            box.Name = "ESPHitbox"
            box.Adornee = part
            box.AlwaysOnTop = true
            box.ZIndex = 10
            box.Size = part.Size + Vector3.new(0.1, 0.1, 0.1)
            box.Transparency = 0.5
            box.Color3 = Color3.fromRGB(255, 50, 50)
            box.Parent = part
        end
    end
end

local function disableHitbox(plr)
    if not plr.Character then return end
    for _, part in pairs(plr.Character:GetChildren()) do
        if part:IsA("BasePart") then
            for _, adornment in pairs(part:GetChildren()) do
                if adornment:IsA("BoxHandleAdornment") and adornment.Name == "ESPHitbox" then
                    adornment:Destroy()
                end
            end
        end
    end
end


local function updateESP()
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("Head") then
            
            local active = espFullEnabled or espNameEnabled
            if active then
                createESPLabel(plr)
            else
                removeESPLabel(plr)
            end

            active = espFullEnabled or espTracersEnabled
            if active then  
                createTracer(plr)  
            else  
                removeTracer(plr)  
            end  

            active = espFullEnabled or espHitboxEnabled
            if active then  
                enableHitbox(plr)  
            else  
                disableHitbox(plr)  
            end  
            
            active = espFullEnabled or espVidaEnabled
            if active and plr.Character:FindFirstChildOfClass("Humanoid") and plr.Character:FindFirstChildOfClass("Humanoid").MaxHealth > 0 then
                createHealthBar(plr)
            else
                removeHealthBar(plr) 
            end
            
        else  
            removeESPLabel(plr)  
            removeTracer(plr)  
            disableHitbox(plr)
            removeHealthBar(plr) 
        end  
    end
end

-- Loop de atualização a cada 2 segundos
task.spawn(function()
    while true do
        if espNameEnabled or espTracersEnabled or espHitboxEnabled or espFullEnabled or espVidaEnabled then
            updateESP()
        end
        task.wait(2)
    end
end)

local hue = 0 
RunService.RenderStepped:Connect(function()
    if not (espTracersEnabled or espFullEnabled) then return end

    if espRGBEnabled then
        hue = (hue + 0.01) % 1
        local rgbColor = Color3.fromHSV(hue, 1, 1) 
        
        for _, tracer in pairs(espTracers) do  
            tracer.Color = rgbColor
        end
    end

    for plr, tracer in pairs(espTracers) do  
        if plr.Character and plr.Character:FindFirstChild("Head") then  
            local headPos = Workspace.CurrentCamera:WorldToViewportPoint(plr.Character.Head.Position)  
            if headPos.Z > 0 then  
                tracer.From = Vector2.new(Workspace.CurrentCamera.ViewportSize.X / 2, Workspace.CurrentCamera.ViewportSize.Y)  
                tracer.To = Vector2.new(headPos.X, headPos.Y)  
                tracer.Visible = true  
            else  
                tracer.Visible = false  
            end  
        else  
            tracer.Visible = false  
        end  
    end

end)

-- Botões ESP

createButton(espFrame, "ESP Names", 0, function()
    espNameEnabled = not espNameEnabled
    updateESP()
end)

createButton(espFrame, "ESP Tracers", 50, function()
    espTracersEnabled = not espTracersEnabled
    updateESP()
end)

createButton(espFrame, "ESP Hitbox", 100, function()
    espHitboxEnabled = not espHitboxEnabled
    updateESP()
end)

-- Botão RGB
createButton(espFrame, "ATIVAR RGB", 150, function(btn)
    espRGBEnabled = not espRGBEnabled
    if espRGBEnabled then
        print("RGB ativado")
    else
        print("RGB desativado")
        for _, tracer in pairs(espTracers) do  
            tracer.Color = Color3.new(1, 0, 0)
        end
    end
end)


-- Lógica ESP FULL
createButton(espFrame, "ESP FULL", 200, function(btn)
    espFullEnabled = not espFullEnabled
    
    if espFullEnabled then
        espNameEnabled = true
        espTracersEnabled = true
        espHitboxEnabled = true
        espVidaEnabled = true 
        print("ESP FULL ativado: Name, Tracers, Hitbox, Vida ON")
    else
        espNameEnabled = false
        espTracersEnabled = false
        espHitboxEnabled = false
        espVidaEnabled = false
        print("ESP FULL desativado: Name, Tracers, Hitbox, Vida OFF")
    end
    updateESP()
end)


createButton(espFrame, "ESP Vida", 250, function()
    espVidaEnabled = not espVidaEnabled
    updateESP()
end)


-- Funções PVP
local autoCLActive = false
local vidaLimite = 10  
local autoCLConnection


-- ===================================================================
-- FUNÇÃO DE LIMPEZA DE NOTIFICAÇÕES (AGRESSIVA)
-- ===================================================================
local function limparNotificacoesAgressivo()
    local success, err = pcall(function()
        for _, gui in pairs(LocalPlayer.PlayerGui:GetChildren()) do
            if gui:IsA("ScreenGui") and gui.Name ~= "GNewStore" then -- Ignora o menu do script
                
                local found = false
                -- 1. Checa se o nome é o padrão de erro
                if gui.Name == "NotifyGui" or gui.Name == "ErrorGui" then
                     found = true
                end

                -- 2. Checa se a GUI tem algum elemento de texto com a palavra "Error" ou "#402"
                for _, descendant in pairs(gui:GetDescendants()) do
                    if descendant:IsA("TextLabel") and (string.find(descendant.Text, "Error", 1, true) or string.find(descendant.Text, "#402", 1, true)) then
                        found = true
                        break
                    end
                end
                
                if found then
                    gui:Destroy()
                    -- print("GUI de erro destruída: " .. gui.Name) -- Descomente para debug
                end
            end
        end
    end)
    if not success then
        warn("Failed to clear notifications aggressively: "..tostring(err))
    end
end


-- ===================================================================
-- FUNÇÃO LÓGICA: PUXAR ITENS (AUTO LOOT) - AJUSTADA PARA USAR O NOVO LIMPADOR
-- ===================================================================

local function findGenericRemote()
    local remotes = {}
    
    -- Remotes comuns para comunicação geral/chat
    local chatRemote = ReplicatedStorage:FindFirstChild("ChatEvent") or ReplicatedStorage:FindFirstChild("sayMessage")
    if chatRemote and chatRemote:IsA("RemoteEvent") then
        table.insert(remotes, chatRemote)
    end
    
    -- Tenta encontrar o remote específico de inventário se ainda estiver lá
    local invRequest = ReplicatedStorage:FindFirstChild("Modules") 
        and ReplicatedStorage.Modules:FindFirstChild("InvRemotes") 
        and ReplicatedStorage.Modules.InvRemotes:FindFirstChild("InvRequest")
    if invRequest and invRequest:IsA("RemoteFunction") then
         table.insert(remotes, invRequest) -- Coloca por último
    end

    -- Se não achou nenhum dos específicos, busca um remote aleatório
    if #remotes == 0 then
        for _, obj in pairs(ReplicatedStorage:GetChildren()) do
            if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
                table.insert(remotes, obj)
                break 
            end
        end
    end

    return remotes[1] 
end

local function togglePuxarItens(button)
    puxarItensToggle = not puxarItensToggle
    
    if puxarItensToggle then
        print("PUXAR ITENS ATIVADO (Tentando usar Remote Silencioso e Limpador Agressivo)")
        
        if button and button:IsA("TextButton") then 
            button.BackgroundColor3 = Color3.fromRGB(0, 150, 0) 
        end
        
        local genericRemote = findGenericRemote()
        
        if not genericRemote then
            print("AVISO: Nenhum Remote de comunicação encontrado. O Puxar Itens pode não funcionar.")
        end


        puxarItensCoroutine = coroutine.create(function()
            while puxarItensToggle do
                
                -- Limpa as notificações no início de cada ciclo do loop
                limparNotificacoesAgressivo() 
                
                if genericRemote then
                    for i, item in ipairs(itens) do
                        local argsPuxar = {"mudaInv", tostring(i), item, "1"} 
                        task.spawn(function()
                            pcall(function()
                                if genericRemote:IsA("RemoteFunction") then
                                    genericRemote:InvokeServer(unpack(argsPuxar))
                                elseif genericRemote:IsA("RemoteEvent") then
                                    genericRemote:FireServer(unpack(argsPuxar))
                                end
                            end)
                        end)
                    end
                else
                    print("Puxar Itens ativo, mas sem remote para usar.")
                end

                task.wait(0.1) 
            end
        end)
        coroutine.resume(puxarItensCoroutine)
    else
        print("PUXAR ITENS DESATIVADO")
        
        if button and button:IsA("TextButton") then 
            button.BackgroundColor3 = Color3.fromRGB(50, 50, 50) 
        end

        if puxarItensCoroutine then
            puxarItensCoroutine = nil
        end
    end
end


createButton(pvpFrame, "AUTO CL", 250, function()
    autoCLActive = not autoCLActive
    print("AUTO CL ativado:", autoCLActive)
    
    local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    local Humanoid = Character:WaitForChild("Humanoid")

    if autoCLActive then
        if autoCLConnection then
            autoCLConnection:Disconnect()
        end
        autoCLConnection = Humanoid.HealthChanged:Connect(function(health)
            if health <= vidaLimite then
                LocalPlayer:Kick("VOCE TOMOU KICK PELO AUTO CL")
            end
        end)
    else
        if autoCLConnection then
            autoCLConnection:Disconnect()
            autoCLConnection = nil
        end
    end
end)

local aimbotPCEnabled = false
local aiming = false
local lockedTarget = nil

local function getClosestToCrosshair()
    local closestPlayer = nil
    local shortestDistance = math.huge
    local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") then
            local head = player.Character.Head
            local screenPoint, onScreen = camera:WorldToViewportPoint(head.Position)
            if onScreen and screenPoint.Z > 0 then
                local distance = (Vector2.new(screenPoint.X, screenPoint.Y) - center).Magnitude
                if distance < 150 then
                    shortestDistance = distance
                    closestPlayer = player
                end
            end
        end
    end

    return closestPlayer
end

local aimbotPCConnection

local function toggleAimbotPC(button)
    aimbotPCEnabled = not aimbotPCEnabled
    if aimbotPCEnabled then
        print("Aimbot PC ativado")
    else
        print("Aimbot PC desativado")
        aiming = false
        lockedTarget = nil
    end
end

UserInputService.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 and aimbotPCEnabled then
        lockedTarget = getClosestToCrosshair()
        aiming = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        aiming = false
        lockedTarget = nil
    end
end)

aimbotPCConnection = RunService.RenderStepped:Connect(function()
    if aiming and lockedTarget and lockedTarget.Character and lockedTarget.Character:FindFirstChild("Head") then
        local head = lockedTarget.Character.Head.Position
        local camPos = camera.CFrame.Position
        camera.CFrame = CFrame.new(camPos, head)
    end
end)

createButton(pvpFrame, "AIMBOT PC", 0, toggleAimbotPC)

local aimbotConnection

local function getClosestTarget()
    local closestDistance = math.huge
    local closestPlayer = nil

    local localCharacter = LocalPlayer.Character
    if not localCharacter or not localCharacter:FindFirstChild("HumanoidRootPart") then
        return nil
    end
    local localPos = localCharacter.HumanoidRootPart.Position

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") and player.Character:FindFirstChildOfClass("Humanoid") then
            local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
            if humanoid.Health > 0 then
                local headPos = player.Character.Head.Position
                local dist = (headPos - localPos).Magnitude
                if dist < closestDistance then
                    closestDistance = dist
                    closestPlayer = player
                end
            end
        end
    end

    return closestPlayer
end

local function aimAt(targetPos)

    local cameraCFrame = camera.CFrame

    local direction = (targetPos - cameraCFrame.Position).Unit

    local newLookAt = CFrame.new(cameraCFrame.Position, cameraCFrame.Position + direction)

    local smoothFactor = 0.3 

    camera.CFrame = cameraCFrame:Lerp(newLookAt, smoothFactor)

end



local aimbotMobileToggle = false 



local function toggleAimbotMobile(button)

    aimbotMobileToggle = not aimbotMobileToggle

    

    if aimbotMobileToggle then

        print("Aimbot Mobile ativado")

        aimbotConnection = RunService.RenderStepped:Connect(function()

            local targetPlayer = getClosestTarget()

            if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("Head") then

                aimAt(targetPlayer.Character.Head.Position)

            end

        end)

    else

        print("Aimbot Mobile desativado")

        if aimbotConnection then

            aimbotConnection:Disconnect()

            aimbotConnection = nil

        end

    end

end



createButton(pvpFrame, "AIMBOT MOBILE", 50, toggleAimbotMobile)



local DETECTION_RADIUS = 10

local CheckInterval = 0.5

local detectados = {}



local remoteFolder = ReplicatedStorage:FindFirstChild("RemoteNovos") or ReplicatedStorage

local dsada = remoteFolder:FindFirstChild("bixobrabo") 



local function isDead(player)

    if player and player.Character then

        local humanoid = player.Character:FindFirstChildOfClass("Humanoid")

        if humanoid and humanoid.Health <= 0 then

            return true

        end

    end

    return false

end



local function distanceBetween(pos1, pos2)

    return (pos1 - pos2).Magnitude

end

local DETECTION_RADIUS = 10
local CheckInterval = 0.5
local detectados = {}

local remoteFolder = ReplicatedStorage:FindFirstChild("RemoteNovos") or ReplicatedStorage
local dsada = remoteFolder:FindFirstChild("bixobrabo") 

local function isDead(player)
    if player and player.Character then
        local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
        if humanoid and humanoid.Health <= 0 then
            return true
        end
    end
    return false
end

local function distanceBetween(pos1, pos2)
    return (pos1 - pos2).Magnitude
end

local revistarToggle = false
local revistarCoroutine

local function iniciarRevistar()
    revistarToggle = not revistarToggle
    
    if revistarToggle then
        print("REVISTAR ATIVADO")
        if revistarCoroutine then return end
        
        revistarCoroutine = coroutine.create(function()
            while revistarToggle do
                local char = LocalPlayer.Character
                if char and char:FindFirstChild("HumanoidRootPart") then
                    local pos = char.HumanoidRootPart.Position
                    for _, otherPlayer in pairs(Players:GetPlayers()) do
                        if otherPlayer ~= LocalPlayer and isDead(otherPlayer) and otherPlayer.Character and otherPlayer.Character:FindFirstChild("HumanoidRootPart") then
                            local otherPos = otherPlayer.Character.HumanoidRootPart.Position
                            if distanceBetween(pos, otherPos) <= DETECTION_RADIUS then
                                if not detectados[otherPlayer] then
                                    detectados[otherPlayer] = true
                                    if dsada then
                                        pcall(function()
                                            dsada:FireServer("/revistar") 
                                        end)
                                    end
                                end
                            else
                                detectados[otherPlayer] = nil
                            end
                        end
                    end
                end
                task.wait(CheckInterval)
            end
            revistarCoroutine = nil
        end)
        coroutine.resume(revistarCoroutine)
    else
        print("REVISTAR DESATIVADO")
    end
end

createButton(pvpFrame, "AUTO REVISTAR PC", 100, iniciarRevistar)

local miniUI = nil
createButton(pvpFrame, "AUTO REVISTAR UI", 150, function()
    if miniUI and miniUI.Parent then
        miniUI:Destroy()
        miniUI = nil
        print("Mini UI fechada.")
        return
    end

    print("Abrindo mini UI...")

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MiniUIFrame"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = game.CoreGui

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 320, 0, 190)
    frame.Position = UDim2.new(0.5, -160, 0.5, -95)
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.Draggable = true
    frame.Parent = screenGui

    local uiCorner = Instance.new("UICorner")
    uiCorner.CornerRadius = UDim.new(0, 16)
    uiCorner.Parent = frame

    local uiStroke = Instance.new("UIStroke")
    uiStroke.Thickness = 2
    uiStroke.Color = Color3.fromRGB(0, 102, 255)
    uiStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    uiStroke.Parent = frame

    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, 0, 0, 40)
    titleLabel.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    titleLabel.Text = "AUTO REVISTAR UI 🚀"
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.Font = Enum.Font.SourceSansBold
    titleLabel.TextSize = 22
    titleLabel.BorderSizePixel = 0
    titleLabel.Parent = frame

    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 16)
    titleCorner.Parent = titleLabel

    local innerButton = Instance.new("TextButton")
    innerButton.Size = UDim2.new(0.75, 0, 0, 60)
    innerButton.Position = UDim2.new(0.5, -120, 0.6, -30)
    innerButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    innerButton.Text = "REVISTAR"
    innerButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    innerButton.Font = Enum.Font.SourceSans
    innerButton.TextSize = 20
    innerButton.BorderSizePixel = 0
    innerButton.Parent = frame

    local buttonCorner = Instance.new("UICorner")
    buttonCorner.CornerRadius = UDim.new(0, 14)
    buttonCorner.Parent = innerButton

    local buttonStroke = Instance.new("UIStroke")
    buttonStroke.Thickness = 2
    buttonStroke.Color = Color3.fromRGB(0, 102, 255)
    buttonStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    buttonStroke.Parent = innerButton

    innerButton.MouseButton1Click:Connect(function()
        iniciarRevistar(innerButton) 
    end)

    miniUI = screenGui
end)

createButton(pvpFrame, "PUXAR ITENS", 200, togglePuxarItens)

-- FUNÇÃO: Teleportar e Tentar Matar (Atualizada: TP APENAS SE MATAR)
local function teleportAndKill()
    local closestHRP = getClosestHRP()
    local localHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")

    if closestHRP and localHRP then
        local targetPlayer = Players:GetPlayerFromCharacter(closestHRP.Parent)
        
        if not targetPlayer then
            print("Alvo não é um jogador válido.")
            return
        end

        print("Alvo encontrado: " .. targetPlayer.Name .. ". Tentando Kill Remota...")

        local remoteFolder = ReplicatedStorage:FindFirstChild("RemoteNovos") or ReplicatedStorage
        local attackRemote = remoteFolder:FindFirstChild("DamageRequest") or remoteFolder:FindFirstChild("AttackRemote")
        
        -- 1. Tenta a Kill Remota
        if attackRemote and attackRemote:IsA("RemoteEvent") then
            pcall(function()
                attackRemote:FireServer(closestHRP.Parent, 1000) -- Envia a requisição de dano
                print("Tentativa de Kill Remota enviada.")
            end)
        else
            print("Remote de Kill não encontrado. Apenas teleporte será tentado.")
            -- Se não tem remote, apenas teleporta (como era a função original)
            localHRP.CFrame = closestHRP.CFrame * CFrame.new(0, 3, 0)
            return
        end
        
        -- 2. Espera um pouco e verifica se o alvo morreu
        task.wait(0.2) 
        
        local targetHumanoid = targetPlayer.Character and targetPlayer.Character:FindFirstChildOfClass("Humanoid")
        
        if targetHumanoid and targetHumanoid.Health <= 0 then
            -- O ALVO MORREU (ou está morrendo) - Teleporta para ele
            localHRP.CFrame = closestHRP.CFrame * CFrame.new(0, 3, 0)
            print("TP/KILL SUCESSO: Teleportado para o jogador morto (" .. targetPlayer.Name .. ")!")
        else
            print("Alvo não morreu. Teleporte cancelado.")
        end

    else
        print("Nenhum alvo encontrado para TP/KILL.")
    end
end

local hue = 0 
RunService.RenderStepped:Connect(function()
    if not (espTracersEnabled or espFullEnabled) then return end

    if espRGBEnabled then
        hue = (hue + 0.01) % 1
        local rgbColor = Color3.fromHSV(hue, 1, 1) 
        
        for _, tracer in pairs(espTracers) do  
            tracer.Color = rgbColor
        end
    end

    for plr, tracer in pairs(espTracers) do  
        if plr.Character and plr.Character:FindFirstChild("Head") then  
            local headPos = Workspace.CurrentCamera:WorldToViewportPoint(plr.Character.Head.Position)  
            if headPos.Z > 0 then  
                tracer.From = Vector2.new(Workspace.CurrentCamera.ViewportSize.X / 2, Workspace.CurrentCamera.ViewportSize.Y)  
                tracer.To = Vector2.new(headPos.X, headPos.Y)  
                tracer.Visible = true  
            else  
                tracer.Visible = false  
            end  
        else  
            tracer.Visible = false  
        end  
    end

end)

-- Botões ESP

createButton(espFrame, "ESP Names", 0, function()
    espNameEnabled = not espNameEnabled
    updateESP()
end)

createButton(espFrame, "ESP Tracers", 50, function()
    espTracersEnabled = not espTracersEnabled
    updateESP()
end)

createButton(espFrame, "ESP Hitbox", 100, function()
    espHitboxEnabled = not espHitboxEnabled
    updateESP()
end)

-- Botão RGB
createButton(espFrame, "ATIVAR RGB", 150, function(btn)
    espRGBEnabled = not espRGBEnabled
    if espRGBEnabled then
        print("RGB ativado")
    else
        print("RGB desativado")
        for _, tracer in pairs(espTracers) do  
            tracer.Color = Color3.new(1, 0, 0)
        end
    end
end)


-- Lógica ESP FULL
createButton(espFrame, "ESP FULL", 200, function(btn)
    espFullEnabled = not espFullEnabled
    
    if espFullEnabled then
        espNameEnabled = true
        espTracersEnabled = true
        espHitboxEnabled = true
        espVidaEnabled = true 
        print("ESP FULL ativado: Name, Tracers, Hitbox, Vida ON")
    else
        espNameEnabled = false
        espTracersEnabled = false
        espHitboxEnabled = false
        espVidaEnabled = false
        print("ESP FULL desativado: Name, Tracers, Hitbox, Vida OFF")
    end
    updateESP()
end)


createButton(espFrame, "ESP Vida", 250, function()
    espVidaEnabled = not espVidaEnabled
    updateESP()
end)


-- Funções PVP
local autoCLActive = false
local vidaLimite = 10  
local autoCLConnection


-- ===================================================================
-- FUNÇÃO DE LIMPEZA DE NOTIFICAÇÕES (AGRESSIVA)
-- ===================================================================
local function limparNotificacoesAgressivo()
    local success, err = pcall(function()
        for _, gui in pairs(LocalPlayer.PlayerGui:GetChildren()) do
            if gui:IsA("ScreenGui") and gui.Name ~= "GNewStore" then -- Ignora o menu do script
                
                local found = false
                -- 1. Checa se o nome é o padrão de erro
                if gui.Name == "NotifyGui" or gui.Name == "ErrorGui" then
                     found = true
                end

                -- 2. Checa se a GUI tem algum elemento de texto com a palavra "Error" ou "#402"
                for _, descendant in pairs(gui:GetDescendants()) do
                    if descendant:IsA("TextLabel") and (string.find(descendant.Text, "Error", 1, true) or string.find(descendant.Text, "#402", 1, true)) then
                        found = true
                        break
                    end
                end
                
                if found then
                    gui:Destroy()
                    -- print("GUI de erro destruída: " .. gui.Name) -- Descomente para debug
                end
            end
        end
    end)
    if not success then
        warn("Failed to clear notifications aggressively: "..tostring(err))
    end
end


-- ===================================================================
-- FUNÇÃO LÓGICA: PUXAR ITENS (AUTO LOOT) - AJUSTADA PARA USAR O NOVO LIMPADOR
-- ===================================================================

local function findGenericRemote()
    local remotes = {}
    
    -- Remotes comuns para comunicação geral/chat
    local chatRemote = ReplicatedStorage:FindFirstChild("ChatEvent") or ReplicatedStorage:FindFirstChild("sayMessage")
    if chatRemote and chatRemote:IsA("RemoteEvent") then
        table.insert(remotes, chatRemote)
    end
    
    -- Tenta encontrar o remote específico de inventário se ainda estiver lá
    local invRequest = ReplicatedStorage:FindFirstChild("Modules") 
        and ReplicatedStorage.Modules:FindFirstChild("InvRemotes") 
        and ReplicatedStorage.Modules.InvRemotes:FindFirstChild("InvRequest")
    if invRequest and invRequest:IsA("RemoteFunction") then
         table.insert(remotes, invRequest) -- Coloca por último
    end

    -- Se não achou nenhum dos específicos, busca um remote aleatório
    if #remotes == 0 then
        for _, obj in pairs(ReplicatedStorage:GetChildren()) do
            if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
                table.insert(remotes, obj)
                break 
            end
        end
    end

    return remotes[1] 
end

local function togglePuxarItens(button)
    puxarItensToggle = not puxarItensToggle
    
    if puxarItensToggle then
        print("PUXAR ITENS ATIVADO (Tentando usar Remote Silencioso e Limpador Agressivo)")
        
        if button and button:IsA("TextButton") then 
            button.BackgroundColor3 = Color3.fromRGB(0, 150, 0) 
        end
        
        local genericRemote = findGenericRemote()
        
        if not genericRemote then
            print("AVISO: Nenhum Remote de comunicação encontrado. O Puxar Itens pode não funcionar.")
        end


        puxarItensCoroutine = coroutine.create(function()
            while puxarItensToggle do
                
                -- Limpa as notificações no início de cada ciclo do loop
                limparNotificacoesAgressivo() 
                
                if genericRemote then
                    for i, item in ipairs(itens) do
                        local argsPuxar = {"mudaInv", tostring(i), item, "1"} 
                        task.spawn(function()
                            pcall(function()
                                if genericRemote:IsA("RemoteFunction") then
                                    genericRemote:InvokeServer(unpack(argsPuxar))
                                elseif genericRemote:IsA("RemoteEvent") then
                                    genericRemote:FireServer(unpack(argsPuxar))
                                end
                            end)
                        end)
                    end
                else
                    print("Puxar Itens ativo, mas sem remote para usar.")
                end

                task.wait(0.1) 
            end
        end)
        coroutine.resume(puxarItensCoroutine)
    else
        print("PUXAR ITENS DESATIVADO")
        
        if button and button:IsA("TextButton") then 
            button.BackgroundColor3 = Color3.fromRGB(50, 50, 50) 
        end

        if puxarItensCoroutine then
            puxarItensCoroutine = nil
        end
    end
end


createButton(pvpFrame, "AUTO CL", 250, function()
    autoCLActive = not autoCLActive
    print("AUTO CL ativado:", autoCLActive)
    
    local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    local Humanoid = Character:WaitForChild("Humanoid")

    if autoCLActive then
        if autoCLConnection then
            autoCLConnection:Disconnect()
        end
        autoCLConnection = Humanoid.HealthChanged:Connect(function(health)
            if health <= vidaLimite then
                LocalPlayer:Kick("VOCE TOMOU KICK PELO AUTO CL")
            end
        end)
    else
        if autoCLConnection then
            autoCLConnection:Disconnect()
            autoCLConnection = nil
        end
    end
end)

local aimbotPCEnabled = false
local aiming = false
local lockedTarget = nil

local function getClosestToCrosshair()
    local closestPlayer = nil
    local shortestDistance = math.huge
    local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") then
            local head = player.Character.Head
            local screenPoint, onScreen = camera:WorldToViewportPoint(head.Position)
            if onScreen and screenPoint.Z > 0 then
                local distance = (Vector2.new(screenPoint.X, screenPoint.Y) - center).Magnitude
                if distance < 150 then
                    shortestDistance = distance
                    closestPlayer = player
                end
            end
        end
    end

    return closestPlayer
end

local aimbotPCConnection

local function toggleAimbotPC(button)
    aimbotPCEnabled = not aimbotPCEnabled
    if aimbotPCEnabled then
        print("Aimbot PC ativado")
    else
        print("Aimbot PC desativado")
        aiming = false
        lockedTarget = nil
    end
end

UserInputService.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 and aimbotPCEnabled then
        lockedTarget = getClosestToCrosshair()
        aiming = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        aiming = false
        lockedTarget = nil
    end
end)

aimbotPCConnection = RunService.RenderStepped:Connect(function()
    if aiming and lockedTarget and lockedTarget.Character and lockedTarget.Character:FindFirstChild("Head") then
        local head = lockedTarget.Character.Head.Position
        local camPos = camera.CFrame.Position
        camera.CFrame = CFrame.new(camPos, head)
    end
end)

createButton(pvpFrame, "AIMBOT PC", 0, toggleAimbotPC)

local aimbotConnection

local function getClosestTarget()
    local closestDistance = math.huge
    local closestPlayer = nil

    local localCharacter = LocalPlayer.Character
    if not localCharacter or not localCharacter:FindFirstChild("HumanoidRootPart") then
        return nil
    end
    local localPos = localCharacter.HumanoidRootPart.Position

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") and player.Character:FindFirstChildOfClass("Humanoid") then
            local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
            if humanoid.Health > 0 then
                local headPos = player.Character.Head.Position
                local dist = (headPos - localPos).Magnitude
                if dist < closestDistance then
                    closestDistance = dist
                    closestPlayer = player
                end
            end
        end
    end

    return closestPlayer
end

local function aimAt(targetPos)
    local cameraCFrame = camera.CFrame
    local direction = (targetPos - cameraCFrame.Position).Unit
    local newLookAt = CFrame.new(cameraCFrame.Position, cameraCFrame.Position + direction)
    local smoothFactor = 0.3 
    camera.CFrame = cameraCFrame:Lerp(newLookAt, smoothFactor)
end

local aimbotMobileToggle = false 

local function toggleAimbotMobile(button)
    aimbotMobileToggle = not aimbotMobileToggle
    
    if aimbotMobileToggle then
        print("Aimbot Mobile ativado")
        aimbotConnection = RunService.RenderStepped:Connect(function()
            local targetPlayer = getClosestTarget()
            if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("Head") then
                aimAt(targetPlayer.Character.Head.Position)
            end
        end)
    else
        print("Aimbot Mobile desativado")
        if aimbotConnection then
            aimbotConnection:Disconnect()
            aimbotConnection = nil
        end
    end
end

createButton(pvpFrame, "AIMBOT MOBILE", 50, toggleAimbotMobile)

local DETECTION_RADIUS = 10
local CheckInterval = 0.5
local detectados = {}

local remoteFolder = ReplicatedStorage:FindFirstChild("RemoteNovos") or ReplicatedStorage
local dsada = remoteFolder:FindFirstChild("bixobrabo") 

local function isDead(player)
    if player and player.Character then
        local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
        if humanoid and humanoid.Health <= 0 then
            return true
        end
    end
    return false
end

local function distanceBetween(pos1, pos2)
    return (pos1 - pos2).Magnitude
end

local revistarToggle = false
local revistarCoroutine

local function iniciarRevistar()
    revistarToggle = not revistarToggle
    
    if revistarToggle then
        print("REVISTAR ATIVADO")
        if revistarCoroutine then return end
        
        revistarCoroutine = coroutine.create(function()
            while revistarToggle do
                local char = LocalPlayer.Character
                if char and char:FindFirstChild("HumanoidRootPart") then
                    local pos = char.HumanoidRootPart.Position
                    for _, otherPlayer in pairs(Players:GetPlayers()) do
                        if otherPlayer ~= LocalPlayer and isDead(otherPlayer) and otherPlayer.Character and otherPlayer.Character:FindFirstChild("HumanoidRootPart") then
                            local otherPos = otherPlayer.Character.HumanoidRootPart.Position
                            if distanceBetween(pos, otherPos) <= DETECTION_RADIUS then
                                if not detectados[otherPlayer] then
                                    detectados[otherPlayer] = true
                                    if dsada then
                                        pcall(function()
                                            dsada:FireServer("/revistar") 
                                        end)
                                    end
                                end
                            else
                                detectados[otherPlayer] = nil
                            end
                        end
                    end
                end
                task.wait(CheckInterval)
            end
            revistarCoroutine = nil
        end)
        coroutine.resume(revistarCoroutine)
    else
        print("REVISTAR DESATIVADO")
    end
end

createButton(pvpFrame, "AUTO REVISTAR PC", 100, iniciarRevistar)

local miniUI = nil
createButton(pvpFrame, "AUTO REVISTAR UI", 150, function()
    if miniUI and miniUI.Parent then
        miniUI:Destroy()
        miniUI = nil
        print("Mini UI fechada.")
        return
    end

    print("Abrindo mini UI...")

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MiniUIFrame"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = game.CoreGui

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 320, 0, 190)
    frame.Position = UDim2.new(0.5, -160, 0.5, -95)
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.Draggable = true
    frame.Parent = screenGui

    local uiCorner = Instance.new("UICorner")
    uiCorner.CornerRadius = UDim.new(0, 16)
    uiCorner.Parent = frame

    local uiStroke = Instance.new("UIStroke")
    uiStroke.Thickness = 2
    uiStroke.Color = Color3.fromRGB(0, 102, 255)
    uiStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    uiStroke.Parent = frame

    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, 0, 0, 40)
    titleLabel.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    titleLabel.Text = "AUTO REVISTAR UI 🚀"
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.Font = Enum.Font.SourceSansBold
    titleLabel.TextSize = 22
    titleLabel.BorderSizePixel = 0
    titleLabel.Parent = frame

    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 16)
    titleCorner.Parent = titleLabel

    local innerButton = Instance.new("TextButton")
    innerButton.Size = UDim2.new(0.75, 0, 0, 60)
    innerButton.Position = UDim2.new(0.5, -120, 0.6, -30)
    innerButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    innerButton.Text = "REVISTAR"
    innerButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    innerButton.Font = Enum.Font.SourceSans
    innerButton.TextSize = 20
    innerButton.BorderSizePixel = 0
    innerButton.Parent = frame

    local buttonCorner = Instance.new("UICorner")
    buttonCorner.CornerRadius = UDim.new(0, 14)
    buttonCorner.Parent = innerButton

    local buttonStroke = Instance.new("UIStroke")
    buttonStroke.Thickness = 2
    buttonStroke.Color = Color3.fromRGB(0, 102, 255)
    buttonStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    buttonStroke.Parent = innerButton

    innerButton.MouseButton1Click:Connect(function()
        iniciarRevistar(innerButton) 
    end)

    miniUI = screenGui
end)

createButton(pvpFrame, "PUXAR ITENS", 200, togglePuxarItens)

-- Botões HitboxConfig
createButton(hitboxFrame, "Tamanho da hitbox 👇", 0, function()
    print("TamanhoAlterado (implemente a lógica)")
end)

-- Criando o slider container
local sliderContainer = Instance.new("Frame")
sliderContainer.Size = UDim2.new(0, 460, 0, 40)
sliderContainer.Position = UDim2.new(0, 0, 0, 50) -- 50 px abaixo do botão
sliderContainer.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
sliderContainer.Parent = hitboxFrame

local sliderCorner = Instance.new("UICorner")
sliderCorner.CornerRadius = UDim.new(0, 6)
sliderCorner.Parent = sliderContainer

-- Barra de fundo do slider
local sliderBar = Instance.new("Frame")
sliderBar.Size = UDim2.new(1, -40, 0, 6)
sliderBar.Position = UDim2.new(0, 20, 0, 17)
sliderBar.BackgroundColor3 = Color3.fromRGB(30,144,255)

sliderBar.Parent = sliderContainer
sliderBar.AnchorPoint = Vector2.new(0, 0.5)

local sliderBarCorner = Instance.new("UICorner")
sliderBarCorner.CornerRadius = UDim.new(0, 3)
sliderBarCorner.Parent = sliderBar

-- Botão que arrasta (o "tracinho")
local sliderHandle = Instance.new("TextButton")
sliderHandle.Size = UDim2.new(0, 20, 0, 20)
sliderHandle.Position = UDim2.new(0, 20, 0, 10)
sliderHandle.BackgroundColor3 = Color3.fromRGB(0,0,255)

sliderHandle.AutoButtonColor = false
sliderHandle.Text = ""
sliderHandle.Parent = sliderContainer

local handleCorner = Instance.new("UICorner")
handleCorner.CornerRadius = UDim.new(1, 0)
handleCorner.Parent = sliderHandle

-- Label que mostra o valor do slider
local sliderValueLabel = Instance.new("TextLabel")
sliderValueLabel.Size = UDim2.new(0, 50, 0, 20)
sliderValueLabel.Position = UDim2.new(1, -50, 0, 10)
sliderValueLabel.BackgroundTransparency = 1
sliderValueLabel.TextColor3 = Color3.new(1, 1, 1)
sliderValueLabel.Font = Enum.Font.GothamBold
sliderValueLabel.TextSize = 18
sliderValueLabel.Text = "0"
sliderValueLabel.Parent = sliderContainer

-- Valor atual do slider (0 a 100)
local sliderValue = 0

-- Função para atualizar a posição do handle e valor
local function updateSlider(inputPosX)
    local barStart = sliderBar.AbsolutePosition.X
    local barWidth = sliderBar.AbsoluteSize.X
    local handleWidth = sliderHandle.AbsoluteSize.X

    -- calcula a posição relativa centralizada
    local relativeX = math.clamp(inputPosX - barStart, 0, barWidth)
    local percent = math.clamp(relativeX / barWidth, 0, 1)

    sliderValue = math.floor(percent * 100)
    sliderValueLabel.Text = tostring(sliderValue)

    -- Centraliza o centro do handle na posição proporcional
    local handlePosX = (percent * (barWidth - handleWidth))
    sliderHandle.Position = UDim2.new(0, handlePosX, 0, 10)

    print("Hitbox size:", sliderValue)
end


local dragging = false

sliderHandle.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        updateSlider(input.Position.X)
    end
end)

-- Também permitir clicar na barra para mover o slider
sliderBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        updateSlider(input.Position.X)
        dragging = true
    end
end)

createButton(hitboxFrame, "Cor da hitbox", 100, function() print("CorAlterada (implemente a lógica)") end) -- Containerlocal flyPanelOpen = false

local walkSpeedLabel = Instance.new("TextLabel")
walkSpeedLabel.Size = UDim2.new(0, 120, 0, 36) -- opcional: maior pra acompanhar o input
walkSpeedLabel.Position = UDim2.new(0, 10, 0, 100)
walkSpeedLabel.BackgroundTransparency = 1
walkSpeedLabel.Text = "WALKSPEED"
walkSpeedLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
walkSpeedLabel.Font = Enum.Font.SourceSansBold
walkSpeedLabel.TextSize = 22 -- ← Aumenta o tamanho aqui
walkSpeedLabel.TextXAlignment = Enum.TextXAlignment.Left
walkSpeedLabel.Parent = flyFrame

local walkSpeedInput = Instance.new("TextBox")
walkSpeedInput.Size = UDim2.new(0, 160, 0, 36) -- aumentamos a largura e altura
walkSpeedInput.Position = UDim2.new(0, 140, 0, 100) -- empurrado mais pra direita
walkSpeedInput.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
walkSpeedInput.TextColor3 = Color3.fromRGB(255, 255, 255)
walkSpeedInput.PlaceholderText = "Digite valor"
walkSpeedInput.Text = ""
walkSpeedInput.Font = Enum.Font.SourceSans
walkSpeedInput.TextSize = 18
walkSpeedInput.BorderSizePixel = 0
walkSpeedInput.ClearTextOnFocus = false
walkSpeedInput.Parent = flyFrame

walkSpeedInput.FocusLost:Connect(function(enterPressed)
    local valor = tonumber(walkSpeedInput.Text)
    if valor then
        local char = LocalPlayer.Character
        if char and char:FindFirstChildOfClass("Humanoid") then
            char:FindFirstChildOfClass("Humanoid").WalkSpeed = valor
            print("WalkSpeed definido para:", valor)
        end
    else
        warn("Valor inválido digitado no campo WalkSpeed.")
    end
end)

-- Arredondamento
local inputCorner = Instance.new("UICorner")
inputCorner.CornerRadius = UDim.new(0, 8)
inputCorner.Parent = walkSpeedInput

-- Variável para controlar o estado do noclip
local noclipEnabled = false
local noclipConnection

-- Função para ativar/desativar noclip
local function toggleNoclip()
    noclipEnabled = not noclipEnabled
    local character = LocalPlayer.Character
    if not character then return end

    if noclipEnabled then
        print("Noclip ativado")
        noclipConnection = game:GetService("RunService").Stepped:Connect(function()
            if character then
                for _, part in pairs(character:GetChildren()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = false
                    end
                end
            end
        end)
    else
        print("Noclip desativado")
        if noclipConnection then
            noclipConnection:Disconnect()
            noclipConnection = nil
        end
        -- Restaurar CanCollide das partes ao desligar noclip
        if character then
            for _, part in pairs(character:GetChildren()) do
                if part:IsA("BasePart") then
                    part.CanCollide = true
                end
            end
        end
    end
end

-- Botão Noclip no menu flyFrame
createButton(flyFrame, "Noclip", 50, function()
    toggleNoclip()
end)

local function openFlyUI()
if flyPanelOpen then return end
flyPanelOpen = true

local player = LocalPlayer  
local char = player.Character or player.CharacterAdded:Wait()  
local hrp = char:WaitForChild("HumanoidRootPart")  

local flyActive = false  
local flySpeed = 60  
local flyConnection  

local flyGui = Instance.new("ScreenGui")  
flyGui.Name = "FlyGui"  
flyGui.Parent = game:GetService("CoreGui")  

local flyFrame = Instance.new("Frame")  
flyFrame.Size = UDim2.new(0, 220, 0, 140)  
flyFrame.Position = UDim2.new(0.5, -110, 0.6, -70)  
flyFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)  
flyFrame.Active = true  
flyFrame.Draggable = true  
flyFrame.Parent = flyGui  

local title = Instance.new("TextLabel")  
title.Size = UDim2.new(1, 0, 0, 30)  
title.BackgroundTransparency = 1  
title.Text = "🚀 Fly Controller"  
title.TextColor3 = Color3.new(1, 1, 1)  
title.Font = Enum.Font.GothamBold  
title.TextSize = 16  
title.Parent = flyFrame  

local statusLabel = Instance.new("TextLabel")  
statusLabel.Size = UDim2.new(1, -20, 0, 20)  
statusLabel.Position = UDim2.new(0, 10, 0, 30)  
statusLabel.BackgroundTransparency = 1  
statusLabel.TextColor3 = Color3.fromRGB(180, 180, 180)  
statusLabel.Font = Enum.Font.Gotham  
statusLabel.TextSize = 14  
statusLabel.Text = "Status: OFF"  
statusLabel.TextXAlignment = Enum.TextXAlignment.Left  
statusLabel.Parent = flyFrame  

local toggleBtn = Instance.new("TextButton")  
toggleBtn.Size = UDim2.new(0, 100, 0, 30)  
toggleBtn.Position = UDim2.new(0, 10, 0, 60)  
toggleBtn.Text = "Ativar Fly"  
toggleBtn.BackgroundColor3 = Color3.fromRGB(40, 80, 160)  
toggleBtn.TextColor3 = Color3.new(1, 1, 1)  
toggleBtn.Font = Enum.Font.Gotham  
toggleBtn.TextSize = 14  
toggleBtn.Parent = flyFrame  
Instance.new("UICorner", toggleBtn)  

local downBtn = Instance.new("TextButton")  
downBtn.Size = UDim2.new(0, 40, 0, 30)  
downBtn.Position = UDim2.new(0, 120, 0, 60)  
downBtn.Text = "-"  
downBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)  
downBtn.TextColor3 = Color3.new(1, 1, 1)  
downBtn.Font = Enum.Font.GothamBold  
downBtn.TextSize = 18  
downBtn.Parent = flyFrame  
Instance.new("UICorner", downBtn)  

local upBtn = Instance.new("TextButton")  
upBtn.Size = UDim2.new(0, 40, 0, 30)  
upBtn.Position = UDim2.new(0, 170, 0, 60)  
upBtn.Text = "+"  
upBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)  
upBtn.TextColor3 = Color3.new(1, 1, 1)  
upBtn.Font = Enum.Font.GothamBold  
upBtn.TextSize = 18  
upBtn.Parent = flyFrame  
Instance.new("UICorner", upBtn)  

local speedLabel = Instance.new("TextLabel")  
speedLabel.Size = UDim2.new(1, -20, 0, 20)  
speedLabel.Position = UDim2.new(0, 10, 0, 100)  
speedLabel.BackgroundTransparency = 1  
speedLabel.TextColor3 = Color3.fromRGB(180, 180, 180)  
speedLabel.Font = Enum.Font.Gotham  
speedLabel.TextSize = 14  
speedLabel.Text = "Velocidade: " .. tostring(flySpeed)  
speedLabel.TextXAlignment = Enum.TextXAlignment.Left  
speedLabel.Parent = flyFrame  

local function startFly()  
	if flyConnection then flyConnection:Disconnect() end  
	local bv = Instance.new("BodyVelocity")  
	bv.Name = "FlyBodyVelocity"  
	bv.MaxForce = Vector3.new(1e9, 1e9, 1e9)  
	bv.Velocity = Vector3.new()  
	bv.Parent = hrp  

	flyConnection = RunService.Heartbeat:Connect(function()  
		local moveVec = Vector3.new()  
		local cam = workspace.CurrentCamera  
		if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveVec += cam.CFrame.LookVector end  
		if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveVec -= cam.CFrame.LookVector end  
		if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveVec -= cam.CFrame.RightVector end  
		if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveVec += cam.CFrame.RightVector end  
		bv.Velocity = moveVec.Magnitude > 0 and moveVec.Unit * flySpeed or Vector3.new()  
	end)  
end  

local function stopFly()  
	if flyConnection then flyConnection:Disconnect() end  
	local bv = hrp:FindFirstChild("FlyBodyVelocity")  
	if bv then bv:Destroy() end  
end  

toggleBtn.MouseButton1Click:Connect(function()  
	flyActive = not flyActive  
	if flyActive then  
		toggleBtn.Text = "Desativar Fly"  
		statusLabel.Text = "Status: ON"  
		startFly()  
	else  
		toggleBtn.Text = "Ativar Fly"  
		statusLabel.Text = "Status: OFF"  
		stopFly()  
	end  
end)  

upBtn.MouseButton1Click:Connect(function()  
	flySpeed += 10  
	speedLabel.Text = "Velocidade: " .. flySpeed  
end)  

downBtn.MouseButton1Click:Connect(function()  
	flySpeed = math.max(10, flySpeed - 10)  
	speedLabel.Text = "Velocidade: " .. flySpeed  
end)  

title.InputBegan:Connect(function(input)  
	if input.UserInputType == Enum.UserInputType.MouseButton1 then  
		flyActive = false  
		stopFly()  
		flyGui:Destroy()  
		flyPanelOpen = false  
	end  
end)

end

-- Botão na aba FLY
createButton(flyFrame, "Abrir UI Fly", 0, function()
openFlyUI()
end)

-- Começar na aba ESP
switchTab("ESP")
