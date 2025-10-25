-- removi o guard _G.KalangoScriptLoaded (você pediu)
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UIS = game:GetService("UserInputService")
local Debris = game:GetService("Debris")

-- Carrega RedzUI
local success, RedzUILib = pcall(function()
    return loadstring(game:HttpGet("https://raw.githubusercontent.com/REDzHUB/RedzUILibrary/main/Source.lua"))()
end)

if not success then
    warn("Falha ao carregar RedzUI")
    -- não faço return forçado; se preferir que pare aqui, descomente a linha abaixo
    -- return
end

-- CLEANUP DE EXECUÇÕES ANTERIORES
-- Remove GUI anterior se existir (evita GUIs empilhadas)
local existingGui = LocalPlayer:FindFirstChild("PlayerGui") and LocalPlayer.PlayerGui:FindFirstChild("KalangoAdminGui")
if existingGui then
    existingGui:Destroy()
end

-- Desconecta listeners antigos se existirem (evita múltiplas conexões)
if _G.KalangoPlayerAddedConn and _G.KalangoPlayerAddedConn.Disconnect then
    pcall(function() _G.KalangoPlayerAddedConn:Disconnect() end)
    _G.KalangoPlayerAddedConn = nil
end
if _G.KalangoPlayerRemovingConn and _G.KalangoPlayerRemovingConn.Disconnect then
    pcall(function() _G.KalangoPlayerRemovingConn:Disconnect() end)
    _G.KalangoPlayerRemovingConn = nil
end

-- Reaproveita tabela global de crash toggles para evitar múltiplos loops
_G.KalangoCrashToggles = _G.KalangoCrashToggles or {}

-- Menu cinza custom
local gui = Instance.new("ScreenGui")
gui.Name = "KalangoAdminGui"
gui.Parent = LocalPlayer:WaitForChild("PlayerGui")
gui.ZIndexBehavior = Enum.ZIndexBehavior.Global

local menu = Instance.new("Frame")
menu.Size = UDim2.new(0,560,0,360)
menu.Position = UDim2.new(0.5,0,0.25,0)
menu.AnchorPoint = Vector2.new(0.5,0)
menu.BackgroundColor3 = Color3.fromRGB(36,36,36)
menu.Parent = gui

-- UICorner só embaixo
local menuCorner = Instance.new("UICorner")
menuCorner.CornerRadius = UDim.new(0.05,0)
menuCorner.Parent = menu

-- Barra preta arrastável
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1,0,0,40)
titleBar.BackgroundColor3 = Color3.fromRGB(0,0,0)
titleBar.Parent = menu
local titleBarCorner = Instance.new("UICorner")
titleBarCorner.CornerRadius = UDim.new(0.1,0)
titleBarCorner.Parent = titleBar

local title = Instance.new("TextLabel")
title.Text = "KALANGO ADMIN"
title.Font = Enum.Font.GothamBold
title.TextSize = 24
title.TextColor3 = Color3.new(1,1,1)
title.BackgroundTransparency = 1
title.Size = UDim2.new(1,0,1,0)
title.Parent = titleBar

-- Barra arrasta o menu
local dragging, dragInput, dragStart, startPos
titleBar.InputBegan:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then
        dragging=true
        dragStart=input.Position
        startPos=menu.Position
        input.Changed:Connect(function()
            if input.UserInputState==Enum.UserInputState.End then dragging=false end
        end)
    end
end)
titleBar.InputChanged:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseMovement or input.UserInputType==Enum.UserInputType.Touch then
        dragInput=input
    end
end)
UIS.InputChanged:Connect(function(input)
    if input==dragInput and dragging then
        local delta = input.Position-dragStart
        menu.Position=UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset+delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset+delta.Y
        )
    end
end)

-- Lista de jogadores
local rightContent = Instance.new("Frame")
rightContent.Position = UDim2.new(0,120,0,50)
rightContent.Size = UDim2.new(1,-140,1,-70)
rightContent.BackgroundColor3 = Color3.fromRGB(36,36,36)
rightContent.Parent = menu
local rightCorner = Instance.new("UICorner")
rightCorner.CornerRadius = UDim.new(0.05,0)
rightCorner.Parent = rightContent

local playersListBox = Instance.new("ScrollingFrame")
playersListBox.Size = UDim2.new(1,-20,0.4,0)
playersListBox.Position = UDim2.new(0,10,0,34)
playersListBox.BackgroundColor3 = Color3.fromRGB(48,48,48)
playersListBox.ScrollBarThickness = 6
playersListBox.Parent = rightContent
local playersLayout = Instance.new("UIListLayout",playersListBox)
playersLayout.SortOrder=Enum.SortOrder.LayoutOrder
playersLayout.Padding=UDim.new(0,6)

local function RefreshPlayers()
    for _,child in ipairs(playersListBox:GetChildren()) do
        if child:IsA("TextButton") then child:Destroy() end
    end
    for _,plr in ipairs(Players:GetPlayers()) do
        local btn=Instance.new("TextButton")
        btn.Size=UDim2.new(1,0,0,28)
        btn.Text=plr.DisplayName.." ("..plr.Name..")"
        btn.BackgroundColor3=Color3.fromRGB(24,24,24)
        btn.TextColor3=Color3.new(1,1,1)
        btn.Font=Enum.Font.Gotham
        btn.TextSize=16
        btn.Parent=playersListBox
        btn.MouseButton1Click:Connect(function()
            for _,b in ipairs(playersListBox:GetChildren()) do
                if b:IsA("TextButton") then b.BackgroundColor3=Color3.fromRGB(24,24,24) end
            end
            btn.BackgroundColor3=Color3.fromRGB(80,80,80)
            rightContent:SetAttribute("SelectedPlayer",plr.Name)
        end)
    end
    playersListBox.CanvasSize=UDim2.new(0,0,0,#Players:GetPlayers()*34)
end

-- conecta/desconecta PlayerAdded/Removing de forma segura (evita duplicar conn)
_G.KalangoPlayerAddedConn = Players.PlayerAdded:Connect(RefreshPlayers)
_G.KalangoPlayerRemovingConn = Players.PlayerRemoving:Connect(RefreshPlayers)
-- preenche agora
RefreshPlayers()

-- Funções Kick/Kill/Jail/Unjail
local optionsFrame = Instance.new("ScrollingFrame")
optionsFrame.Size = UDim2.new(1,-20,0.56,-10)
optionsFrame.Position = UDim2.new(0,10,0.42,0)
optionsFrame.BackgroundTransparency=1
optionsFrame.ScrollBarThickness=6
optionsFrame.Parent=rightContent
local optionsLayout = Instance.new("UIListLayout",optionsFrame)
optionsLayout.SortOrder=Enum.SortOrder.LayoutOrder
optionsLayout.Padding=UDim.new(0,16)

local jailTime=15
local jaulas={}

local function createOptionBtn(text,color,callback)
    local btn=Instance.new("TextButton")
    btn.Size=UDim2.new(1,0,0,38)
    btn.BackgroundColor3=color
    btn.Text=text
    btn.TextColor3=Color3.new(1,1,1)
    btn.Font=Enum.Font.GothamBold
    btn.TextSize=18
    btn.Parent=optionsFrame
    Instance.new("UICorner",btn).CornerRadius=UDim.new(0.15,0)
    btn.MouseButton1Click:Connect(callback)
    return btn
end

createOptionBtn("Kick",Color3.fromRGB(200,50,50),function()
    local selected=rightContent:GetAttribute("SelectedPlayer")
    if selected==LocalPlayer.Name then LocalPlayer:Kick("Você foi expulso") end
end)

createOptionBtn("Kill",Color3.fromRGB(50,120,200),function()
    local selected=rightContent:GetAttribute("SelectedPlayer")
    if selected==LocalPlayer.Name and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
        LocalPlayer.Character.Humanoid.Health=0
    end
end)

createOptionBtn("Jail",Color3.fromRGB(120,120,120),function()
    local selected=rightContent:GetAttribute("SelectedPlayer")
    if selected~=LocalPlayer.Name then return end
    local hrp=LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    jaulas[selected]={}
    local positions={Vector3.new(0,-3,0),Vector3.new(0,3,0),Vector3.new(-2.5,0,0),Vector3.new(2.5,0,0),Vector3.new(0,0,-2.5),Vector3.new(0,0,2.5)}
    for i,pos in ipairs(positions) do
        local p=Instance.new("Part")
        p.Anchored=true
        p.CanCollide=true
        p.Transparency=0.15
        if i<=2 then
            p.Size=Vector3.new(5,0.4,5)
        else
            p.Size=Vector3.new(0.4,6,5)
            if i>=5 then p.Size=Vector3.new(5,6,0.4) end
        end
        p.Position=hrp.Position+pos
        p.Color=Color3.fromRGB(60,60,60)
        p.Parent=workspace
        table.insert(jaulas[selected],p)
    end
    task.spawn(function()
        wait(jailTime)
        for _,p in ipairs(jaulas[selected]) do if p and p.Parent then p:Destroy() end end
        jaulas[selected]=nil
    end)
end)

createOptionBtn("Unjail",Color3.fromRGB(80,200,120),function()
    local selected=rightContent:GetAttribute("SelectedPlayer")
    if jaulas[selected] then
        for _,p in ipairs(jaulas[selected]) do if p and p.Parent then p:Destroy() end end
        jaulas[selected]=nil
    end
end)

-- Toggle Crash Player (integra com _G.KalangoCrashToggles para não duplicar loops)
local crashToggles = _G.KalangoCrashToggles

-- Se RedzUILib carregou, tenta criar tab (não é crítico)
local tab
if success and RedzUILib then
    local win = RedzUILib:Window("Kalango","Admin","v1.0")
    tab = win:Tab("Admin")
end

-- Se tab existir, usa toggle UI; caso contrário, cria botão simples dentro do optionsFrame para ligar/desligar crash
if tab and tab.Toggle then
    tab:Toggle("Crash player",false,function(state)
        local selected=rightContent:GetAttribute("SelectedPlayer")
        if not selected then return end
        crashToggles[selected]=state
        if state then
            task.spawn(function()
                while crashToggles[selected] do
                    for i=1,50 do
                        local p=Instance.new("Part")
                        p.Size=Vector3.new(1,1,1)
                        p.Anchored=true
                        p.CanCollide=false
                        p.Position=Vector3.new(math.random(-500,500),math.random(10,500),math.random(-500,500))
                        p.Transparency=1
                        p.Parent=workspace
                        Debris:AddItem(p,0.05)
                    end
                    task.wait()
                end
            end)
        end
    end)
else
    -- botão alternativo caso RedzUI não tenha carregado
    local crashBtn = createOptionBtn("Toggle Crash player", Color3.fromRGB(180,50,180), function()
        local selected=rightContent:GetAttribute("SelectedPlayer")
        if not selected then return end
        crashToggles[selected] = not crashToggles[selected]
        if crashToggles[selected] then
            task.spawn(function()
                while crashToggles[selected] do
                    for i=1,50 do
                        local p=Instance.new("Part")
                        p.Size=Vector3.new(1,1,1)
                        p.Anchored=true
                        p.CanCollide=false
                        p.Position=Vector3.new(math.random(-500,500),math.random(10,500),math.random(-500,500))
                        p.Transparency=1
                        p.Parent=workspace
                        Debris:AddItem(p,0.05)
                    end
                    task.wait()
                end
            end)
        end
    end)
end

-- salva a tabela global atualizada
_G.KalangoCrashToggles = crashToggles
