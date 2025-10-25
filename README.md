-- EnricoHub - Hub simples para executor (colar tudo em um script local no executor)
-- Recursos:
-- - Salvar posição (Save Position)
-- - Ir até posição salva (Go To Position)
-- - Fly (pulos infinitos via JumpRequest)
-- - Velocidade com input no formato "2x" e botão "Put" para aplicar
-- - Minimize / Maximize (➖ / ➕)
-- - Drag enquanto mantém pressionado o botão ↔️
-- Uso: cole inteiro em um executor (por exemplo Synapse/PlrExecutors) e execute.

-- Configurações rápidas
local StarterParent = game:GetService("CoreGui") -- usar CoreGui para executors
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
if not player then
    -- se executado num ambiente sem LocalPlayer, aborta
    return
end

-- Guarda estado
local savedCFrame = nil
local flyEnabled = false
local flyConnection = nil
local jumpConnection = nil
local defaultWalkSpeed = 16
local appliedMultiplier = 1

-- Estado do GUI (minimizado / maximizado)
local isMinimized = false
local originalSize, originalPosition = nil, nil

-- Drag state
local dragging = false
local dragOffset = Vector2.new(0,0)

-- Função utilitária pra obter personagem/humanoid/root
local function getCharacterParts()
    local char = player.Character or player.CharacterAdded and player.CharacterAdded:Wait()
    if not char then return nil end
    local hrp = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    return char, humanoid, hrp
end

-- Cria GUI
local function createGui()
    -- Limpa GUI anterior com mesmo nome
    local existing = StarterParent:FindFirstChild("EnricoHubGUI_for_Executor")
    if existing then existing:Destroy() end

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "EnricoHubGUI_for_Executor"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = StarterParent

    -- Frame principal
    local frame = Instance.new("Frame")
    frame.Name = "Main"
    frame.Size = UDim2.new(0, 320, 0, 200)
    frame.Position = UDim2.new(0.5, -160, 0.5, -100)
    frame.BackgroundColor3 = Color3.fromRGB(30,30,30)
    frame.BorderSizePixel = 0
    frame.Parent = screenGui

    local uicorner = Instance.new("UICorner", frame)
    uicorner.CornerRadius = UDim.new(0,8)

    -- Título / barra superior
    local titleBar = Instance.new("Frame", frame)
    titleBar.Name = "TitleBar"
    titleBar.Size = UDim2.new(1, 0, 0, 36)
    titleBar.Position = UDim2.new(0,0,0,0)
    titleBar.BackgroundTransparency = 1

    local title = Instance.new("TextLabel", titleBar)
    title.Size = UDim2.new(1, -96, 1, 0)
    title.Position = UDim2.new(0,8,0,0)
    title.BackgroundTransparency = 1
    title.Text = "EnricoHub"
    title.TextColor3 = Color3.fromRGB(220,220,220) -- será atualizado dinamicamente
    title.Font = Enum.Font.SourceSansBold
    title.TextSize = 18
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.TextYAlignment = Enum.TextYAlignment.Center

    -- Botões da direita na titlebar: Drag (↔️), Minimize (➖), Maximize (➕)
    local btnSize = UDim2.new(0,28,0,28)
    local btnY = UDim2.new(0, 0, 0.5, -14)

    local dragBtn = Instance.new("TextButton", titleBar)
    dragBtn.Name = "DragBtn"
    dragBtn.Size = btnSize
    dragBtn.Position = UDim2.new(1, -92, 0.5, -14)
    dragBtn.AnchorPoint = Vector2.new(0,0)
    dragBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
    dragBtn.Text = "↔️"
    dragBtn.Font = Enum.Font.SourceSansBold
    dragBtn.TextSize = 18
    dragBtn.TextColor3 = Color3.fromRGB(230,230,230)
    dragBtn.AutoButtonColor = true

    local minimizeBtn = Instance.new("TextButton", titleBar)
    minimizeBtn.Name = "MinimizeBtn"
    minimizeBtn.Size = btnSize
    minimizeBtn.Position = UDim2.new(1, -58, 0.5, -14)
    minimizeBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
    minimizeBtn.Text = "➖"
    minimizeBtn.Font = Enum.Font.SourceSansBold
    minimizeBtn.TextSize = 18
    minimizeBtn.TextColor3 = Color3.fromRGB(230,230,230)
    minimizeBtn.AutoButtonColor = true

    local maximizeBtn = Instance.new("TextButton", titleBar)
    maximizeBtn.Name = "MaximizeBtn"
    maximizeBtn.Size = btnSize
    maximizeBtn.Position = UDim2.new(1, -30, 0.5, -14)
    maximizeBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
    maximizeBtn.Text = "➕"
    maximizeBtn.Font = Enum.Font.SourceSansBold
    maximizeBtn.TextSize = 18
    maximizeBtn.TextColor3 = Color3.fromRGB(230,230,230)
    maximizeBtn.AutoButtonColor = true

    -- Label de status (abaixo da titlebar)
    local status = Instance.new("TextLabel", frame)
    status.Name = "Status"
    status.Size = UDim2.new(1, -16, 0, 20)
    status.Position = UDim2.new(0,8,0,44)
    status.BackgroundTransparency = 1
    status.Text = "Status: pronto"
    status.TextColor3 = Color3.fromRGB(170,170,170)
    status.Font = Enum.Font.SourceSans
    status.TextSize = 14
    status.TextXAlignment = Enum.TextXAlignment.Left

    -- Content frame: tudo que pode ser escondido ao minimizar
    local content = Instance.new("Frame", frame)
    content.Name = "Content"
    content.Size = UDim2.new(1, -16, 1, -76)
    content.Position = UDim2.new(0,8,0,68)
    content.BackgroundTransparency = 1

    -- Botão Salvar Posição
    local saveBtn = Instance.new("TextButton", content)
    saveBtn.Size = UDim2.new(0,140,0,34)
    saveBtn.Position = UDim2.new(0,0,0,0)
    saveBtn.Text = "Save Position"
    saveBtn.BackgroundColor3 = Color3.fromRGB(40,120,220)
    saveBtn.TextColor3 = Color3.fromRGB(255,255,255)
    saveBtn.Font = Enum.Font.SourceSansBold
    saveBtn.TextSize = 16
    saveBtn.AutoButtonColor = true

    -- Botão Ir Até Posição
    local gotoBtn = Instance.new("TextButton", content)
    gotoBtn.Size = UDim2.new(0,140,0,34)
    gotoBtn.Position = UDim2.new(0,152,0,0)
    gotoBtn.Text = "Go To Position"
    gotoBtn.BackgroundColor3 = Color3.fromRGB(45,170,80)
    gotoBtn.TextColor3 = Color3.fromRGB(255,255,255)
    gotoBtn.Font = Enum.Font.SourceSansBold
    gotoBtn.TextSize = 16
    gotoBtn.AutoButtonColor = true

    -- Botão Fly (pulos infinitos)
    local flyBtn = Instance.new("TextButton", content)
    flyBtn.Size = UDim2.new(0,140,0,34)
    flyBtn.Position = UDim2.new(0,0,0,42)
    flyBtn.Text = "Fly: OFF"
    flyBtn.BackgroundColor3 = Color3.fromRGB(200,100,40)
    flyBtn.TextColor3 = Color3.fromRGB(255,255,255)
    flyBtn.Font = Enum.Font.SourceSansBold
    flyBtn.TextSize = 16
    flyBtn.AutoButtonColor = true

    -- Caixa de input para velocidade
    local speedBox = Instance.new("TextBox", content)
    speedBox.Size = UDim2.new(0,160,0,28)
    speedBox.Position = UDim2.new(0,0,0,92)
    speedBox.PlaceholderText = "Ex: 2x"
    speedBox.Text = ""
    speedBox.ClearTextOnFocus = false
    speedBox.BackgroundColor3 = Color3.fromRGB(50,50,50)
    speedBox.TextColor3 = Color3.fromRGB(220,220,220)
    speedBox.Font = Enum.Font.SourceSans
    speedBox.TextSize = 16
    speedBox.TextXAlignment = Enum.TextXAlignment.Left

    -- Botão Put para aplicar velocidade
    local putBtn = Instance.new("TextButton", content)
    putBtn.Size = UDim2.new(0,140,0,28)
    putBtn.Position = UDim2.new(0,168,0,92)
    putBtn.Text = "Put"
    putBtn.BackgroundColor3 = Color3.fromRGB(100,80,200)
    putBtn.TextColor3 = Color3.fromRGB(255,255,255)
    putBtn.Font = Enum.Font.SourceSansBold
    putBtn.TextSize = 16
    putBtn.AutoButtonColor = true

    -- Guarda tamanho/posição originais para maximizar depois
    originalSize = frame.Size
    originalPosition = frame.Position

    -- Funções dos botões
    saveBtn.MouseButton1Click:Connect(function()
        local _, humanoid, hrp = getCharacterParts()
        if hrp then
            savedCFrame = hrp.CFrame
            status.Text = "Status: posição salva."
        else
            status.Text = "Status: personagem não encontrado."
        end
    end)

    gotoBtn.MouseButton1Click:Connect(function()
        local _, humanoid, hrp = getCharacterParts()
        if not savedCFrame then
            status.Text = "Status: nenhuma posição salva."
            return
        end
        if hrp then
            local ok, err = pcall(function()
                hrp.CFrame = savedCFrame + Vector3.new(0,2,0) -- teleporta 2 studs acima pra evitar prender no chão
            end)
            if ok then
                status.Text = "Status: teleportado para posição salva."
            else
                status.Text = "Status: erro ao teleportar: "..tostring(err)
            end
        else
            status.Text = "Status: personagem não encontrado."
        end
    end)

    local function enableFly()
        if flyEnabled then return end
        flyEnabled = true
        flyBtn.Text = "Fly: ON"
        status.Text = "Status: fly ativado (pulso infinito)."
        -- conecta JumpRequest para permitir pular sempre
        if jumpConnection then
            jumpConnection:Disconnect()
            jumpConnection = nil
        end
        jumpConnection = UserInputService.JumpRequest:Connect(function()
            local _, humanoid = getCharacterParts()
            if humanoid and humanoid.Health > 0 then
                -- Force change state to jumping
                pcall(function()
                    humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
                end)
            end
        end)
    end

    local function disableFly()
        if not flyEnabled then return end
        flyEnabled = false
        flyBtn.Text = "Fly: OFF"
        status.Text = "Status: fly desativado."
        if jumpConnection then
            jumpConnection:Disconnect()
            jumpConnection = nil
        end
    end

    flyBtn.MouseButton1Click:Connect(function()
        if not flyEnabled then
            enableFly()
        else
            disableFly()
        end
    end)

    -- Aplica velocidade com input tipo "2x"
    putBtn.MouseButton1Click:Connect(function()
        local text = speedBox.Text or ""
        -- extrai número (aceita floats) do texto
        local num = tonumber(string.match(text, "([%d%.]+)"))
        if not num then
            status.Text = "Status: entrada inválida. Use formato como '2x' (apenas número + x)."
            return
        end
        -- garante valor razoável (limite)
        if num <= 0 or num > 100 then
            status.Text = "Status: multiplicador fora do intervalo (0 < x <= 100)."
            return
        end
        local _, humanoid = getCharacterParts()
        if humanoid then
            -- guarda walk speed original se não guardado
            if defaultWalkSpeed == nil then
                defaultWalkSpeed = 16
            end
            appliedMultiplier = num
            local newSpeed = (defaultWalkSpeed or 16) * num
            local ok, err = pcall(function()
                humanoid.WalkSpeed = newSpeed
            end)
            if ok then
                status.Text = ("Status: velocidade aplicada: %sx (WalkSpeed = %s)"):format(num, tostring(newSpeed))
            else
                status.Text = "Status: erro ao aplicar velocidade: "..tostring(err)
            end
        else
            status.Text = "Status: humanoid não encontrado."
        end
    end)

    -- Pequena função pra manter defaultWalkSpeed atualizado quando humanoid muda
    player.CharacterAdded:Connect(function(char)
        wait(0.5)
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            -- atualiza defaultWalkSpeed sempre que personagem aparece
            defaultWalkSpeed = humanoid.WalkSpeed or 16
            -- reaplica multiplier se já estava aplicado
            if appliedMultiplier and appliedMultiplier ~= 1 then
                pcall(function() humanoid.WalkSpeed = defaultWalkSpeed * appliedMultiplier end)
            end
        end
    end)

    -- Minimize / Maximize behaviour
    local function minimize()
        if isMinimized then return end
        isMinimized = true
        -- esconde conteúdo e ajusta tamanho
        content.Visible = false
        status.Visible = false
        frame.Size = UDim2.new(0, 220, 0, 44)
    end

    local function maximize()
        if not isMinimized then
            -- se já maximizado, restaura posição/tamanho originais
            frame.Size = originalSize or UDim2.new(0,320,0,200)
            frame.Position = originalPosition or frame.Position
            return
        end
        isMinimized = false
        content.Visible = true
        status.Visible = true
        frame.Size = originalSize or UDim2.new(0,320,0,200)
    end

    -- Assign button actions
    minimizeBtn.MouseButton1Click:Connect(function()
        minimize()
    end)

    maximizeBtn.MouseButton1Click:Connect(function()
        maximize()
    end)

    -- Drag implementation: enquanto segurando o dragBtn com Mouse1, atualiza posição
    -- Utiliza UserInputService:GetMouseLocation() para obter coords em pixels
    dragBtn.MouseButton1Down:Connect(function()
        -- begin dragging
        dragging = true
        -- calcula offset entre canto superior-esquerdo do frame e mouse
        local mousePos = UserInputService:GetMouseLocation() -- Vector2 (pixels)
        -- Ajusta para considerar o offset do topo da tela (roblox adiciona 36 px no topo em alguns clients; GetMouseLocation já é em pixels relativos à viewport)
        local framePos = frame.AbsolutePosition
        dragOffset = Vector2.new(mousePos.X - framePos.X, mousePos.Y - framePos.Y)
    end)

    -- Quando soltar botão do mouse, para de arrastar
    local function stopDragging()
        dragging = false
    end
    dragBtn.MouseButton1Up:Connect(stopDragging)
    -- também parar se input ended em outro lugar (por touch ou mouse)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)

    -- Atualiza posição toda frame (RenderStepped) enquanto dragging == true
    local renderConn
    renderConn = RunService.RenderStepped:Connect(function()
        if dragging then
            local mousePos = UserInputService:GetMouseLocation()
            -- Converte para nova posição em pixels (canto superior esquerdo do frame)
            local newX = math.floor(mousePos.X - dragOffset.X)
            local newY = math.floor(mousePos.Y - dragOffset.Y)
            -- Opcional: limitar para dentro da tela (borda)
            local screenSize = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(1920,1080)
            local maxX = math.max(0, screenSize.X - frame.AbsoluteSize.X)
            local maxY = math.max(0, screenSize.Y - frame.AbsoluteSize.Y)
            newX = math.clamp(newX, 0, maxX)
            newY = math.clamp(newY, 0, maxY)
            -- Aplica posição em modo offset (UDim2 com escala 0)
            local ok, err = pcall(function()
                frame.Position = UDim2.new(0, newX, 0, newY)
            end)
            if not ok then
                -- em caso de erro, apenas para de arrastar
                dragging = false
            end
        end
    end)

    -- Animação RGB do título (mudando as cores)
    local colorConn
    do
        local hueSpeed = 0.12 -- velocidade do ciclo (ajuste conforme quiser)
        -- assegura que exista a função Color3.fromHSV (Roblox possui)
        colorConn = RunService.RenderStepped:Connect(function()
            -- tick() retorna tempo em segundos; pegamos hue entre 0 e 1
            local h = (tick() * hueSpeed) % 1
            -- usa saturação e valor cheios para cor viva
            local col = Color3.fromHSV(h, 0.9, 1)
            -- Aplica suavemente (pode ser direto)
            title.TextColor3 = col
        end)
    end

    -- Clean up when GUI destroyed (optional)
    screenGui.AncestryChanged:Connect(function()
        if not screenGui.Parent then
            if renderConn then renderConn:Disconnect() end
            if jumpConnection then jumpConnection:Disconnect() end
            if colorConn then colorConn:Disconnect() end
        end
    end)

    return screenGui
end

-- Cria GUI agora
local gui = createGui()

-- Mensagem final simples no output (opcional)
print("[EnricoHub] GUI criada. Use os botões para salvar/ir para posição, ativar fly, aplicar velocidade, minimizar/maximizar e arrastar (segure ↔️).")
