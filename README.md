-- NEXUS AIMBOT GOLD MOBILE OP - Painel EDUCATIVO + Noclip (adaptado para uso em seu jogo)
-- Use apenas no seu próprio jogo / ambiente de desenvolvimento.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UIS = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- CONFIG
local TARGET_PART = "Head"
local AIM_FOV = 150
local AIM_SMOOTH = 1 -- 1 = instant, 0.2 = suave
local HITBOX_MULT = 4 -- aumento do hitbox para treino
-- FLAGS
local aimActive, espActive, hitboxActive, magicBulletActive, autofireActive, lowGraphicsActive, noclipActive = false,false,false,false,false,false,false

local originalSizes = {}
local originalCollision = {} -- para restaurar colisões do jogador

-- GUI
local gui = Instance.new("ScreenGui")
gui.Parent = player:WaitForChild("PlayerGui")
gui.ResetOnSpawn = false
gui.Name = "NexusEduGui"

-- Open button
local openBtn = Instance.new("TextButton")
openBtn.Size = UDim2.new(0,100,0,40)
openBtn.Position = UDim2.new(0.85,0,0.03,0)
openBtn.BackgroundColor3 = Color3.fromRGB(120,90,20)
openBtn.Text = "Menu"
openBtn.Font = Enum.Font.Gotham
openBtn.TextScaled = true
openBtn.Parent = gui

-- Panel
local panel = Instance.new("ScrollingFrame")
panel.Size = UDim2.new(0,260,0,520)
panel.Position = UDim2.new(0.55,0,0.12,0)
panel.CanvasSize = UDim2.new(0,0,0,800)
panel.ScrollBarThickness = 6
panel.BackgroundColor3 = Color3.fromRGB(20,20,20)
panel.BorderSizePixel = 0
panel.Visible = false
panel.Parent = gui

-- Button Maker
local function makeBtn(txt,y)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0,240,0,44)
    b.Position = UDim2.new(0,10,0,y)
    b.BackgroundColor3 = Color3.fromRGB(40,40,40)
    b.Text = txt
    b.Font = Enum.Font.Gotham
    b.TextScaled = true
    b.TextColor3 = Color3.new(1,1,1)
    b.Parent = panel
    return b
end

-- Buttons
local y = 10
local aimBtn = makeBtn("Ativar Aimbot (Treino)",y) y+=54
local espBtn = makeBtn("Ativar ESP (Visual)",y) y+=54
local hitBtn = makeBtn("Ativar Hitbox (Treino)",y) y+=54
local magicBtn = makeBtn("Ativar Bala Mágica (DESATIVADO)",y) y+=54 -- placeholder
local fireBtn = makeBtn("Ativar AutoFire (DESATIVADO)",y) y+=54 -- placeholder
local graphicsBtn = makeBtn("Gráficos Super Low",y) y+=54
local noclipBtn = makeBtn("Ativar Noclip",y) y+=54
local closeBtn = makeBtn("Fechar Menu",y)

-- Toggles
openBtn.MouseButton1Click:Connect(function() panel.Visible = not panel.Visible end)
closeBtn.MouseButton1Click:Connect(function() panel.Visible = false end)

aimBtn.MouseButton1Click:Connect(function()
    aimActive = not aimActive
    aimBtn.Text = aimActive and "Desativar Aimbot (Treino)" or "Ativar Aimbot (Treino)"
end)
espBtn.MouseButton1Click:Connect(function()
    espActive = not espActive
    espBtn.Text = espActive and "Desativar ESP (Visual)" or "Ativar ESP (Visual)"
end)
hitBtn.MouseButton1Click:Connect(function()
    hitboxActive = not hitboxActive
    hitBtn.Text = hitboxActive and "Desativar Hitbox (Treino)" or "Ativar Hitbox (Treino)"
end)
magicBtn.MouseButton1Click:Connect(function()
    magicBulletActive = not magicBulletActive
    magicBtn.Text = magicBulletActive and "Desativar Bala Mágica (DESATIVADO)" or "Ativar Bala Mágica (DESATIVADO)"
end)
fireBtn.MouseButton1Click:Connect(function()
    autofireActive = not autofireActive
    fireBtn.Text = autofireActive and "Desativar AutoFire (DESATIVADO)" or "Ativar AutoFire (DESATIVADO)"
end)
graphicsBtn.MouseButton1Click:Connect(function()
    lowGraphicsActive = not lowGraphicsActive
    graphicsBtn.Text = lowGraphicsActive and "Restaurar Gráficos" or "Gráficos Super Low"

    local carsFolders = {Workspace:FindFirstChild("Cars"), Workspace:FindFirstChild("Vehicles")}

    for _, part in ipairs(Workspace:GetDescendants()) do
        if part:IsA("BasePart") then
            local isCar = false
            for _, folder in ipairs(carsFolders) do
                if folder and part:IsDescendantOf(folder) then
                    isCar = true
                    break
                end
            end
            if part.Parent and part.Parent.Name:lower():find("car") then
                isCar = true
            end

            if not isCar then
                if lowGraphicsActive then
                    part.Material = Enum.Material.Plastic
                    part.CastShadow = false
                    part.Reflectance = 0
                    part.Transparency = 0
                else
                    part.Material = Enum.Material.SmoothPlastic
                    part.CastShadow = true
                    part.Reflectance = 0
                    part.Transparency = 0
                end
            end

        elseif part:IsA("ParticleEmitter") or part:IsA("Beam") or part:IsA("Trail") then
            part.Enabled = not lowGraphicsActive
        elseif part:IsA("Decal") or part:IsA("Texture") then
            part.Transparency = lowGraphicsActive and 0.5 or 0
        end
    end

    -- Iluminação otimizada
    if lowGraphicsActive then
        Lighting.GlobalShadows = false
        Lighting.FogEnd = 50
        Lighting.Brightness = 1
        Lighting.OutdoorAmbient = Color3.fromRGB(100,100,100)
    else
        Lighting.GlobalShadows = true
        Lighting.FogEnd = 100000
        Lighting.Brightness = 2
        Lighting.OutdoorAmbient = Color3.fromRGB(255,255,255)
    end
end)

-- Noclip helper functions (aplica somente ao personagem local)
local function setCharacterCollision(char, enabled)
    if not char then return end
    local id = char:GetDebugId() -- identificador temporário (não-persistente) -- usado só para tabela
    -- armazenar/restaurar uma vez
    if enabled then
        -- restaurar colisões para os que foram alterados
        if originalCollision[char] then
            for part, canCollide in pairs(originalCollision[char]) do
                if part and part.Parent then
                    pcall(function() part.CanCollide = canCollide end)
                end
            end
            originalCollision[char] = nil
        end
    else
        -- salvar estados originais e aplicar noclip
        originalCollision[char] = originalCollision[char] or {}
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                if originalCollision[char][part] == nil then
                    originalCollision[char][part] = part.CanCollide
                end
                pcall(function() part.CanCollide = false end)
            end
        end
    end
end

-- Noclip toggle
noclipBtn.MouseButton1Click:Connect(function()
    noclipActive = not noclipActive
    noclipBtn.Text = noclipActive and "Desativar Noclip" or "Ativar Noclip"

    local char = player.Character or player.CharacterAdded:Wait()
    -- aplica ou restaura
    setCharacterCollision(char, not noclipActive)
end)

-- quando o personagem respawnar, garantir estado do noclip e limpar tabela antiga
player.CharacterAdded:Connect(function(char)
    -- pequena espera para partes existirem
    char:WaitForChild("Humanoid", 5)
    wait(0.1)
    if noclipActive then
        setCharacterCollision(char, false)
    else
        setCharacterCollision(char, true)
    end
end)

-- Aimbot inteligente (apenas câmera de treino, sem disparo remoto)
local function getClosest()
    local center = Vector2.new(camera.ViewportSize.X/2,camera.ViewportSize.Y/2)
    local best,dist
    for _,pl in pairs(Players:GetPlayers()) do
        if pl~=player and pl.Character and pl.Character:FindFirstChild(TARGET_PART) then
            local pos,vis = camera:WorldToViewportPoint(pl.Character[TARGET_PART].Position)
            if vis then
                local mag = (Vector2.new(pos.X,pos.Y)-center).Magnitude
                if mag<AIM_FOV and (not dist or mag<dist) then
                    best = pl.Character[TARGET_PART]
                    dist = mag
                end
            end
        end
    end
    return best
end

local function expandHitbox(char)
    if not char or not char:FindFirstChild(TARGET_PART) then return end
    local part = char[TARGET_PART]
    if not originalSizes[part] then originalSizes[part] = part.Size end
    part.Size = originalSizes[part]*HITBOX_MULT
    part.Material = Enum.Material.Neon
    part.Transparency = 0.5
end

-- Restaurar hitboxes quando desativar
local function restoreHitboxes()
    for part, size in pairs(originalSizes) do
        if part and part.Parent then
            pcall(function() part.Size = size end)
        end
    end
    originalSizes = {}
end

-- ESP visual (aplica Highlight local)
local function applyESPToCharacter(char)
    if not char then return end
    if not char:FindFirstChild("ESP_LOCAL") then
        local h = Instance.new("Highlight")
        h.Name = "ESP_LOCAL"
        h.FillTransparency = 1
        h.OutlineColor = Color3.new(1,1,0)
        h.Parent = char
    end
end

local function removeESPFromCharacter(char)
    if not char then return end
    local h = char:FindFirstChild("ESP_LOCAL")
    if h then h:Destroy() end
end

-- Loop principal: aimbot, esp e hitbox (execução local e segura)
RunService.RenderStepped:Connect(function(dt)
    -- Aimbot: apenas ajustar câmera para treino (sem atirar remotamente)
    if aimActive then
        local target = getClosest()
        if target and target.Parent then
            local targetCFrame = CFrame.new(camera.CFrame.Position, target.Position)
            if AIM_SMOOTH>=1 then
                camera.CFrame = targetCFrame
            else
                camera.CFrame = camera.CFrame:Lerp(targetCFrame,AIM_SMOOTH)
            end
        end
    end

    -- ESP + Hitbox: aplicar localmente (não faz network)
    for _,p in pairs(Players:GetPlayers()) do
        if p ~= player and p.Character then
            if espActive then
                applyESPToCharacter(p.Character)
            else
                removeESPFromCharacter(p.Character)
            end

            if hitboxActive then
                expandHitbox(p.Character)
            end
        end
    end

    if not hitboxActive then
        -- restaura se não ativo
        restoreHitboxes()
    end

    -- Noclip: garantir que o jogador local não colida enquanto ativo (proteção contínua)
    if noclipActive then
        local char = player.Character
        if char then
            -- reaplicar para qualquer parte nova
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    if part.CanCollide then
                        pcall(function() part.CanCollide = false end)
                    end
                end
            end
        end
    end
end)

-- Limpeza ao fechar (opcional)
gui.AncestryChanged:Connect(function()
    if not gui:IsDescendantOf(game) then
        -- restaurar estados
        local char = player.Character
        if char then setCharacterCollision(char, true) end
        restoreHitboxes()
    end
end)

warn("NEXUS EDU PAINEL carregado! Use apenas no seu jogo para testes/treino.")
