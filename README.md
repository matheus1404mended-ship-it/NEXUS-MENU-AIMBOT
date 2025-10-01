-- NEXUS EDU - Painel enxuto (sem bala Mágica / AutoFire), Hitbox ampliado e Noclip robusto
-- Use apenas no seu próprio jogo / ambiente de desenvolvimento.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- CONFIG
local AIM_FOV = 150
local AIM_SMOOTH = 1 -- 1 = instant, 0.2 = suave
local HITBOX_MULT = 6 -- agora maior; ajuste se quiser (6 = 6x)
-- FLAGS
local aimActive, espActive, hitboxActive, lowGraphicsActive, noclipActive = false,false,false,false,false

local originalSizes = {}         -- guarda tamanho original por parte
local originalCollision = {}     -- guarda CanCollide original por personagem (tabela de part -> bool)

-- GUI
local gui = Instance.new("ScreenGui")
gui.Name = "NexusEduGui"
gui.Parent = player:WaitForChild("PlayerGui")
gui.ResetOnSpawn = false

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
local function saveCollisionForCharacter(char)
    if not char then return end
    if originalCollision[char] then return end
    originalCollision[char] = {}
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            originalCollision[char][part] = part.CanCollide
        end
    end
end

local function restoreCollisionForCharacter(char)
    if not char or not originalCollision[char] then return end
    for part, canCollide in pairs(originalCollision[char]) do
        if part and part.Parent then
            pcall(function() part.CanCollide = canCollide end)
        end
    end
    originalCollision[char] = nil
end

local function applyNoclipToCharacter(char)
    if not char then return end
    -- salvar
    saveCollisionForCharacter(char)
    -- desativar colisão em todas as partes
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            pcall(function() part.CanCollide = false end)
        end
    end
    -- tentar manter humanoid em estado compatível com atravessar (cliente)
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid then
        pcall(function() humanoid:ChangeState(Enum.HumanoidStateType.Physics) end)
    end
end

local function removeNoclipFromCharacter(char)
    if not char then return end
    restoreCollisionForCharacter(char)
    -- tentar restaurar estado do humanoid
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid then
        -- tenta colocar em estado natural (se possível)
        pcall(function() humanoid:ChangeState(Enum.HumanoidStateType.GettingUp) end)
    end
end

noclipBtn.MouseButton1Click:Connect(function()
    noclipActive = not noclipActive
    noclipBtn.Text = noclipActive and "Desativar Noclip" or "Ativar Noclip"
    local char = player.Character or player.CharacterAdded:Wait()
    if noclipActive then
        applyNoclipToCharacter(char)
    else
        removeNoclipFromCharacter(char)
    end
end)

player.CharacterAdded:Connect(function(char)
    wait(0.1)
    if noclipActive then
        applyNoclipToCharacter(char)
    else
        -- garante estado normal (restaura caso tenha tabela antiga)
        removeNoclipFromCharacter(char)
    end
end)

-- Hitbox ampliado: agora tenta expandir as partes principais (mais confiável)
local function expandHitboxesForCharacter(char)
    if not char then return end
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            -- salvar tamanho original uma só vez
            if not originalSizes[part] then
                originalSizes[part] = part.Size
            end
            -- aplica escala (atenção: escalar pode afetar posicionamento/anim)
            local ok, _ = pcall(function()
                part.Size = originalSizes[part] * HITBOX_MULT
                -- tornar visível para debug (opcional): usar Neon/transparency
                -- part.Material = Enum.Material.Neon
                -- part.Transparency = 0.5
            end)
        end
    end
end

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

-- Aimbot: encontra parte visível mais próxima à center (prioriza Head/UpperTorso/Root)
local function getClosestTargetPart()
    local center = Vector2.new(camera.ViewportSize.X/2,camera.ViewportSize.Y/2)
    local best, dist
    for _, pl in pairs(Players:GetPlayers()) do
        if pl ~= player and pl.Character and pl.Character.Parent then
            local char = pl.Character
            local candidates = {"Head","UpperTorso","LowerTorso","Torso","HumanoidRootPart"}
            local foundPart
            for _, name in ipairs(candidates) do
                if char:FindFirstChild(name) then
                    foundPart = char[name]
                    break
                end
            end
            if foundPart then
                local pos, vis = camera:WorldToViewportPoint(foundPart.Position)
                if vis then
                    local mag = (Vector2.new(pos.X,pos.Y)-center).Magnitude
                    if mag < AIM_FOV and (not dist or mag < dist) then
                        best = foundPart
                        dist = mag
                    end
                end
            end
        end
    end
    return best
end

-- Loop principal
RunService.RenderStepped:Connect(function(dt)
    -- Aimbot (apenas câmera de treino)
    if aimActive then
        local targetPart = getClosestTargetPart()
        if targetPart and targetPart.Parent then
            local targetCFrame = CFrame.new(camera.CFrame.Position, targetPart.Position)
            if AIM_SMOOTH >= 1 then
                camera.CFrame = targetCFrame
            else
                camera.CFrame = camera.CFrame:Lerp(targetCFrame, AIM_SMOOTH)
            end
        end
    end

    -- ESP e Hitbox
    for _, pl in pairs(Players:GetPlayers()) do
        if pl ~= player and pl.Character then
            if espActive then
                applyESPToCharacter(pl.Character)
            else
                removeESPFromCharacter(pl.Character)
            end
            if hitboxActive then
                expandHitboxesForCharacter(pl.Character)
            end
        end
    end

    if not hitboxActive then
        restoreHitboxes()
    end

    -- Noclip: reaplicar continuamente no cliente para evitar partes recém-criadas voltarem a colidir
    if noclipActive then
        local char = player.Character
        if char then
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    if part.CanCollide then
                        pcall(function() part.CanCollide = false end)
                    end
                end
            end
            -- força humanoid Physics state para reduzir chances de bloqueio por colisão do servidor
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if humanoid then
                pcall(function() humanoid:ChangeState(Enum.HumanoidStateType.Physics) end)
            end
        end
    end
end)

-- Limpeza ao fechar/remoção da GUI
gui.AncestryChanged:Connect(function()
    if not gui:IsDescendantOf(game) then
        -- restaurar tudo
        local char = player.Character
        if char then
            removeNoclipFromCharacter(char)
        end
        restoreHitboxes()
        -- remover ESPs residuais
        for _, pl in pairs(Players:GetPlayers()) do
            if pl.Character then removeESPFromCharacter(pl.Character) end
        end
    end
end)

warn("NEXUS EDU PAINEL (sem BalaMágica/AutoFire). Hitbox ampliado e Noclip robusto carregados.")
