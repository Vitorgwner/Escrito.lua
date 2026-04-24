-- ====================================================
--           DAVIDHUB v3 - by DavidHub
--   ESP Drawing completo + Aimbot FOV + Mais features
-- ====================================================

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local Workspace        = game:GetService("Workspace")
local CoreGui          = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local Camera      = Workspace.CurrentCamera

-- ====================================================
--                   CONFIGURAÇÕES
-- ====================================================

local Config = {
    -- Aimbot
    Aimbot        = false,
    AimbotFOV     = 200,
    AimbotSmooth  = 0.3,
    AimbotBone    = "Head",   -- "Head" ou "HumanoidRootPart"
    AimbotVisible = true,     -- só mira em alvos visíveis

    -- ESP
    ESP           = false,
    ESPBox        = true,     -- caixa 2D ao redor do player
    ESPName       = true,     -- nome do player
    ESPDist       = true,     -- distância
    ESPHealth     = true,     -- barra de vida
    ESPSkeleton   = false,    -- linhas de esqueleto
    ESPTracer     = false,    -- linha do chão até o player
    ESPMaxDist    = 500,
    ESPTeamColor  = false,    -- usa cor do time

    -- Highlight (SelectionBox 3D)
    Highlight     = false,

    -- Chams (cor no personagem)
    Chams         = false,

    -- Movement
    Fly           = false,
    FlySpeed      = 60,
    Noclip        = false,
    InfJump       = false,

    -- Speed
    Turbo         = false,
    TurboSpeed    = 50,

    -- Fling
    Fling         = false,
    FlingForce    = 1e6,

    -- Misc
    NoFall        = false,
    BunnyHop      = false,
    AntiAFK       = true,
    FullBright    = false,
    CrosshairDraw = false,
}

-- ====================================================
--                   UTILITÁRIOS
-- ====================================================

local function getCharacter(p)  return p and p.Character end
local function getRootPart(p)
    local c = getCharacter(p)
    return c and (c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("UpperTorso"))
end
local function getHumanoid(p)
    local c = getCharacter(p)
    return c and c:FindFirstChildOfClass("Humanoid")
end
local function getHead(p)
    local c = getCharacter(p)
    return c and c:FindFirstChild("Head")
end
local function isAlive(p)
    local h = getHumanoid(p)
    return h and h.Health > 0
end
local function getBone(p, bone)
    local c = getCharacter(p)
    return c and (c:FindFirstChild(bone) or c:FindFirstChild("HumanoidRootPart"))
end
local function isVisible(part)
    if not part then return false end
    local origin = Camera.CFrame.Position
    local dir    = (part.Position - origin)
    local ray    = Ray.new(origin, dir)
    local hit    = Workspace:FindPartOnRayWithIgnoreList(ray, {getCharacter(LocalPlayer), Camera})
    return hit == part or hit == nil or hit:IsDescendantOf(getCharacter(part.Parent.Parent) or Workspace)
end

-- Calcula bounding box 2D de um personagem
local function getCharacterBBox(character)
    local minX, minY = math.huge, math.huge
    local maxX, maxY = -math.huge, -math.huge
    local valid = false

    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            local corners = {
                part.CFrame * CFrame.new( part.Size.X/2,  part.Size.Y/2,  part.Size.Z/2),
                part.CFrame * CFrame.new(-part.Size.X/2,  part.Size.Y/2,  part.Size.Z/2),
                part.CFrame * CFrame.new( part.Size.X/2, -part.Size.Y/2,  part.Size.Z/2),
                part.CFrame * CFrame.new(-part.Size.X/2, -part.Size.Y/2,  part.Size.Z/2),
                part.CFrame * CFrame.new( part.Size.X/2,  part.Size.Y/2, -part.Size.Z/2),
                part.CFrame * CFrame.new(-part.Size.X/2,  part.Size.Y/2, -part.Size.Z/2),
                part.CFrame * CFrame.new( part.Size.X/2, -part.Size.Y/2, -part.Size.Z/2),
                part.CFrame * CFrame.new(-part.Size.X/2, -part.Size.Y/2, -part.Size.Z/2),
            }
            for _, cf in ipairs(corners) do
                local sp, onScreen = Camera:WorldToViewportPoint(cf.Position)
                if onScreen then
                    valid = true
                    if sp.X < minX then minX = sp.X end
                    if sp.Y < minY then minY = sp.Y end
                    if sp.X > maxX then maxX = sp.X end
                    if sp.Y > maxY then maxY = sp.Y end
                end
            end
        end
    end

    if not valid then return nil end
    return minX, minY, maxX - minX, maxY - minY
end

-- ====================================================
--          FOV CIRCLE (bola rosa) — Drawing API
-- ====================================================

local fovCircle
pcall(function()
    fovCircle = Drawing.new("Circle")
    fovCircle.Visible   = false
    fovCircle.Color     = Color3.fromRGB(220, 0, 200)
    fovCircle.Thickness = 2
    fovCircle.Filled    = false
    fovCircle.NumSides  = 64
    fovCircle.Transparency = 1
end)

-- Crosshair drawing
local crosshairLines = {}
pcall(function()
    for i = 1, 4 do
        local l = Drawing.new("Line")
        l.Visible   = false
        l.Color     = Color3.fromRGB(255, 255, 255)
        l.Thickness = 1.5
        l.Transparency = 1
        crosshairLines[i] = l
    end
end)

RunService.RenderStepped:Connect(function()
    -- FOV Circle
    if fovCircle then
        fovCircle.Visible      = Config.Aimbot
        fovCircle.Radius       = Config.AimbotFOV
        fovCircle.Position     = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
        fovCircle.Transparency = 1
    end

    -- Crosshair
    if #crosshairLines == 4 then
        local cx = Camera.ViewportSize.X / 2
        local cy = Camera.ViewportSize.Y / 2
        local sz = 12
        local gap = 4
        local visible = Config.CrosshairDraw
        -- left
        crosshairLines[1].Visible = visible
        crosshairLines[1].From = Vector2.new(cx - sz - gap, cy)
        crosshairLines[1].To   = Vector2.new(cx - gap,      cy)
        -- right
        crosshairLines[2].Visible = visible
        crosshairLines[2].From = Vector2.new(cx + gap, cy)
        crosshairLines[2].To   = Vector2.new(cx + sz + gap, cy)
        -- up
        crosshairLines[3].Visible = visible
        crosshairLines[3].From = Vector2.new(cx, cy - sz - gap)
        crosshairLines[3].To   = Vector2.new(cx, cy - gap)
        -- down
        crosshairLines[4].Visible = visible
        crosshairLines[4].From = Vector2.new(cx, cy + gap)
        crosshairLines[4].To   = Vector2.new(cx, cy + sz + gap)
    end
end)

-- ====================================================
--                     AIMBOT
-- ====================================================

local function getNearestTarget()
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    local closest, closestDist = nil, Config.AimbotFOV

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and isAlive(player) then
            local bone = getBone(player, Config.AimbotBone)
            if bone then
                if Config.AimbotVisible and not isVisible(bone) then continue end
                local sp, onScreen = Camera:WorldToViewportPoint(bone.Position)
                if onScreen then
                    local dist = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                    if dist < closestDist then
                        closestDist = dist
                        closest = bone
                    end
                end
            end
        end
    end
    return closest
end

RunService.Heartbeat:Connect(function()
    if Config.Aimbot then
        local target = getNearestTarget()
        if target then
            local tp, onScreen = Camera:WorldToViewportPoint(target.Position)
            if onScreen then
                local ray    = Camera:ViewportPointToRay(tp.X, tp.Y)
                local lookAt = Camera.CFrame.Position + ray.Direction * 100
                Camera.CFrame = Camera.CFrame:Lerp(
                    CFrame.new(Camera.CFrame.Position, lookAt),
                    Config.AimbotSmooth
                )
            end
        end
    end
end)

-- ====================================================
--              ESP DRAWING (Drawing API)
-- ====================================================

-- Esqueleto: pares de bones conectados
local SKELETON_PAIRS = {
    {"Head","UpperTorso"},
    {"UpperTorso","LowerTorso"},
    {"UpperTorso","LeftUpperArm"},
    {"LeftUpperArm","LeftLowerArm"},
    {"LeftLowerArm","LeftHand"},
    {"UpperTorso","RightUpperArm"},
    {"RightUpperArm","RightLowerArm"},
    {"RightLowerArm","RightHand"},
    {"LowerTorso","LeftUpperLeg"},
    {"LeftUpperLeg","LeftLowerLeg"},
    {"LeftLowerLeg","LeftFoot"},
    {"LowerTorso","RightUpperLeg"},
    {"RightUpperLeg","RightLowerLeg"},
    {"RightLowerLeg","RightFoot"},
}

local ESPObjects = {} -- [player] = { box={}, labels, health, skeleton, tracer }

local function newLine(color, thickness)
    local l = Drawing.new("Line")
    l.Visible     = false
    l.Color       = color or Color3.fromRGB(220,0,200)
    l.Thickness   = thickness or 1.5
    l.Transparency = 1
    return l
end

local function newText(size, color, outline)
    local t = Drawing.new("Text")
    t.Visible      = false
    t.Size         = size or 14
    t.Color        = color or Color3.fromRGB(255,255,255)
    t.Outline      = outline ~= false
    t.OutlineColor = Color3.fromRGB(0,0,0)
    t.Font         = Drawing.Fonts.GothamBold
    t.Transparency = 1
    return t
end

local function newQuad(color, thickness)
    local q = Drawing.new("Quad")
    q.Visible      = false
    q.Color        = color or Color3.fromRGB(220,0,200)
    q.Thickness    = thickness or 1.5
    q.Filled       = false
    q.Transparency = 1
    return q
end

local function getESPColor(player, dist)
    if Config.ESPTeamColor then
        pcall(function()
            local team = player.Team
            if team then return team.TeamColor.Color end
        end)
    end
    if dist < 50  then return Color3.fromRGB(255, 50,  50)  end
    if dist < 150 then return Color3.fromRGB(255, 200, 0)   end
    return Color3.fromRGB(220, 0, 200)
end

local function createESPForPlayer(player)
    if player == LocalPlayer then return end
    if ESPObjects[player] then
        -- clear old
        for _, obj in pairs(ESPObjects[player]) do
            if type(obj) == "table" then
                for _, o in pairs(obj) do pcall(function() o:Remove() end) end
            else
                pcall(function() obj:Remove() end)
            end
        end
    end

    local obj = {}

    -- 2D Box: 4 lines forming a rectangle
    obj.box = {
        newLine(Color3.fromRGB(220,0,200), 1.5), -- top
        newLine(Color3.fromRGB(220,0,200), 1.5), -- bottom
        newLine(Color3.fromRGB(220,0,200), 1.5), -- left
        newLine(Color3.fromRGB(220,0,200), 1.5), -- right
        -- Corner fills (thin white inner)
        newLine(Color3.fromRGB(0,0,0), 3), -- top shadow
        newLine(Color3.fromRGB(0,0,0), 3), -- bottom shadow
        newLine(Color3.fromRGB(0,0,0), 3), -- left shadow
        newLine(Color3.fromRGB(0,0,0), 3), -- right shadow
    }

    -- Name label
    obj.nameLabel = newText(13, Color3.fromRGB(255,255,255))

    -- Distance label
    obj.distLabel = newText(11, Color3.fromRGB(200,200,200))

    -- Health bar: 2 lines (background + fill)
    obj.healthBG  = newLine(Color3.fromRGB(0,0,0), 4)
    obj.healthFill= newLine(Color3.fromRGB(0,255,80), 3)

    -- Skeleton lines
    obj.skeleton = {}
    for _ = 1, #SKELETON_PAIRS do
        table.insert(obj.skeleton, newLine(Color3.fromRGB(255,255,255), 1))
    end

    -- Tracer
    obj.tracer = newLine(Color3.fromRGB(220,0,200), 1)

    ESPObjects[player] = obj
end

local function removeESPForPlayer(player)
    local obj = ESPObjects[player]
    if not obj then return end
    for _, v in pairs(obj) do
        if type(v) == "table" then
            for _, o in pairs(v) do pcall(function() o:Remove() end) end
        else
            pcall(function() v:Remove() end)
        end
    end
    ESPObjects[player] = nil
end

-- Init ESP for current players
for _, p in ipairs(Players:GetPlayers()) do
    createESPForPlayer(p)
end
Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.wait(1)
        createESPForPlayer(p)
    end)
    createESPForPlayer(p)
end)
Players.PlayerRemoving:Connect(removeESPForPlayer)

-- Main ESP render loop
RunService.RenderStepped:Connect(function()
    for _, player in ipairs(Players:GetPlayers()) do
        local obj = ESPObjects[player]
        if not obj then continue end

        local function hideAll()
            for _, v in pairs(obj) do
                if type(v) == "table" then
                    for _, o in pairs(v) do pcall(function() o.Visible = false end) end
                else
                    pcall(function() v.Visible = false end)
                end
            end
        end

        if player == LocalPlayer or not Config.ESP or not isAlive(player) then
            hideAll()
            continue
        end

        local myRoot  = getRootPart(LocalPlayer)
        local tgtRoot = getRootPart(player)
        if not myRoot or not tgtRoot then hideAll() continue end

        local dist = math.floor((myRoot.Position - tgtRoot.Position).Magnitude)
        if dist > Config.ESPMaxDist then hideAll() continue end

        local char = getCharacter(player)
        if not char then hideAll() continue end

        local color = getESPColor(player, dist)

        -- ---- 2D BOUNDING BOX ----
        local bx, by, bw, bh = getCharacterBBox(char)
        local boxVisible = Config.ESPBox and bx ~= nil

        -- Shadow lines (drawn first, behind)
        for i = 5, 8 do
            obj.box[i].Visible = false
        end

        if boxVisible then
            local pad = 2
            local x1, y1 = bx - pad, by - pad
            local x2, y2 = bx + bw + pad, by + bh + pad
            -- top
            obj.box[1].From = Vector2.new(x1, y1)
            obj.box[1].To   = Vector2.new(x2, y1)
            obj.box[1].Color = color
            obj.box[1].Visible = true
            -- bottom
            obj.box[2].From = Vector2.new(x1, y2)
            obj.box[2].To   = Vector2.new(x2, y2)
            obj.box[2].Color = color
            obj.box[2].Visible = true
            -- left
            obj.box[3].From = Vector2.new(x1, y1)
            obj.box[3].To   = Vector2.new(x1, y2)
            obj.box[3].Color = color
            obj.box[3].Visible = true
            -- right
            obj.box[4].From = Vector2.new(x2, y1)
            obj.box[4].To   = Vector2.new(x2, y2)
            obj.box[4].Color = color
            obj.box[4].Visible = true
        else
            for i = 1, 4 do obj.box[i].Visible = false end
        end

        -- ---- NAME LABEL ----
        if Config.ESPName and bx then
            local cx = bx + bw / 2
            obj.nameLabel.Text     = player.Name
            obj.nameLabel.Position = Vector2.new(cx, by - 18)
            obj.nameLabel.Color    = color
            obj.nameLabel.Center   = true
            obj.nameLabel.Visible  = true
        else
            obj.nameLabel.Visible = false
        end

        -- ---- DISTANCE LABEL ----
        if Config.ESPDist and bx then
            local cx = bx + bw / 2
            obj.distLabel.Text     = "[" .. dist .. "m]"
            obj.distLabel.Position = Vector2.new(cx, by + bh + 3)
            obj.distLabel.Color    = Color3.fromRGB(180,180,180)
            obj.distLabel.Center   = true
            obj.distLabel.Visible  = true
        else
            obj.distLabel.Visible = false
        end

        -- ---- HEALTH BAR ----
        local hum = getHumanoid(player)
        if Config.ESPHealth and bx and hum then
            local hp  = hum.Health
            local max = hum.MaxHealth
            local pct = math.clamp(hp / max, 0, 1)

            local barX  = bx - 6
            local barY1 = by
            local barY2 = by + bh

            -- background
            obj.healthBG.From    = Vector2.new(barX, barY1)
            obj.healthBG.To      = Vector2.new(barX, barY2)
            obj.healthBG.Visible = true

            -- fill
            local fillY2 = barY1 + (barY2 - barY1) * (1 - pct)
            -- health color: green → yellow → red
            local hColor
            if pct > 0.6 then
                hColor = Color3.fromRGB(0, 220, 80)
            elseif pct > 0.3 then
                hColor = Color3.fromRGB(255, 200, 0)
            else
                hColor = Color3.fromRGB(255, 50, 50)
            end

            obj.healthFill.From    = Vector2.new(barX, barY2)
            obj.healthFill.To      = Vector2.new(barX, fillY2)
            obj.healthFill.Color   = hColor
            obj.healthFill.Visible = true
        else
            obj.healthBG.Visible   = false
            obj.healthFill.Visible = false
        end

        -- ---- SKELETON ----
        for i, pair in ipairs(SKELETON_PAIRS) do
            local line = obj.skeleton[i]
            if Config.ESPSkeleton then
                local p1 = char:FindFirstChild(pair[1])
                local p2 = char:FindFirstChild(pair[2])
                if p1 and p2 then
                    local sp1, on1 = Camera:WorldToViewportPoint(p1.Position)
                    local sp2, on2 = Camera:WorldToViewportPoint(p2.Position)
                    if on1 and on2 then
                        line.From    = Vector2.new(sp1.X, sp1.Y)
                        line.To      = Vector2.new(sp2.X, sp2.Y)
                        line.Color   = Color3.fromRGB(255, 255, 255)
                        line.Visible = true
                    else
                        line.Visible = false
                    end
                else
                    line.Visible = false
                end
            else
                line.Visible = false
            end
        end

        -- ---- TRACER ----
        if Config.ESPTracer and tgtRoot then
            local sp, onScreen = Camera:WorldToViewportPoint(tgtRoot.Position)
            if onScreen then
                local bottomCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                obj.tracer.From    = bottomCenter
                obj.tracer.To      = Vector2.new(sp.X, sp.Y)
                obj.tracer.Color   = color
                obj.tracer.Visible = true
            else
                obj.tracer.Visible = false
            end
        else
            obj.tracer.Visible = false
        end
    end
end)

-- ====================================================
--              HIGHLIGHT 3D (SelectionBox)
-- ====================================================

local HighlightFolder = Instance.new("Folder")
HighlightFolder.Name   = "DavidHub_Highlight"
HighlightFolder.Parent = LocalPlayer.PlayerGui

local function refreshHighlight()
    for _, v in ipairs(HighlightFolder:GetChildren()) do v:Destroy() end
    if not Config.Highlight then return end
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local char = getCharacter(player)
            if char then
                local sb = Instance.new("SelectionBox")
                sb.Color3              = Color3.fromRGB(220, 0, 200)
                sb.LineThickness       = 0.04
                sb.SurfaceTransparency = 0.8
                sb.SurfaceColor3       = Color3.fromRGB(180, 0, 180)
                sb.Adornee             = char
                sb.Parent              = HighlightFolder
            end
        end
    end
end

-- ====================================================
--                    CHAMS
-- ====================================================

local function applyChams(player, enable)
    local char = getCharacter(player)
    if not char then return end
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            if enable then
                part.Material     = Enum.Material.Neon
                part.Color        = Color3.fromRGB(200, 0, 220)
            else
                part.Material = Enum.Material.SmoothPlastic
            end
        end
    end
end

local function refreshChams()
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then applyChams(p, Config.Chams) end
    end
end

-- ====================================================
--                       FLY
-- ====================================================

local flyConn
local function enableFly()
    local hum  = getHumanoid(LocalPlayer)
    local root = getRootPart(LocalPlayer)
    if not hum or not root then return end

    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(1e9,1e9,1e9)
    bv.Velocity = Vector3.zero
    bv.Parent   = root

    local bg = Instance.new("BodyGyro")
    bg.MaxTorque = Vector3.new(1e9,1e9,1e9)
    bg.D = 100
    bg.Parent = root

    hum.PlatformStand = true

    flyConn = RunService.Heartbeat:Connect(function()
        if not Config.Fly then
            bv:Destroy(); bg:Destroy()
            hum.PlatformStand = false
            if flyConn then flyConn:Disconnect() end
            return
        end
        local dir = Vector3.zero
        local uis = UserInputService
        if uis:IsKeyDown(Enum.KeyCode.W) then dir = dir + Camera.CFrame.LookVector end
        if uis:IsKeyDown(Enum.KeyCode.S) then dir = dir - Camera.CFrame.LookVector end
        if uis:IsKeyDown(Enum.KeyCode.A) then dir = dir - Camera.CFrame.RightVector end
        if uis:IsKeyDown(Enum.KeyCode.D) then dir = dir + Camera.CFrame.RightVector end
        if uis:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.new(0,1,0) end
        if uis:IsKeyDown(Enum.KeyCode.LeftControl) then dir = dir - Vector3.new(0,1,0) end
        bv.Velocity = dir.Magnitude > 0 and dir.Unit * Config.FlySpeed or Vector3.zero
        bg.CFrame   = Camera.CFrame
    end)
end

-- ====================================================
--               TURBO / NOCLIP / MISC
-- ====================================================

-- Infinite Jump
LocalPlayer.Character and LocalPlayer.Character:WaitForChild("Humanoid")
LocalPlayer.CharacterAdded:Connect(function(char)
    local hum = char:WaitForChild("Humanoid")
    hum.StateChanged:Connect(function(_, new)
        if Config.InfJump and new == Enum.HumanoidStateType.Jumping then
            task.wait(0.1)
            hum:ChangeState(Enum.HumanoidStateType.Jumping)
        end
    end)
end)

-- Anti-AFK
local vuf = LocalPlayer:FindFirstChildOfClass("VirtualUser")
if not vuf then
    vuf = Instance.new("VirtualUser")
    vuf.Parent = LocalPlayer
end
LocalPlayer.Idled:Connect(function()
    if Config.AntiAFK and vuf then
        vuf:CaptureController()
        vuf:ClickButton2(Vector2.new())
    end
end)

-- FullBright
local function setFullBright(enable)
    local lighting = game:GetService("Lighting")
    if enable then
        lighting.Brightness    = 10
        lighting.ClockTime     = 14
        lighting.FogEnd        = 100000
        lighting.GlobalShadows = false
        lighting.Ambient       = Color3.fromRGB(255,255,255)
    else
        lighting.Brightness    = 1
        lighting.ClockTime     = 14
        lighting.FogEnd        = 100000
        lighting.GlobalShadows = true
        lighting.Ambient       = Color3.fromRGB(70,70,70)
    end
end

RunService.Heartbeat:Connect(function()
    local hum = getHumanoid(LocalPlayer)
    if hum then
        hum.WalkSpeed = Config.Turbo and Config.TurboSpeed or 16
    end
    if Config.Noclip then
        local c = getCharacter(LocalPlayer)
        if c then
            for _, p in ipairs(c:GetDescendants()) do
                if p:IsA("BasePart") then p.CanCollide = false end
            end
        end
    end
    if Config.NoFall then
        local c = getCharacter(LocalPlayer)
        local hum2 = hum
        if hum2 and hum2:GetState() == Enum.HumanoidStateType.Freefall then
            hum2:ChangeState(Enum.HumanoidStateType.GettingUp)
        end
    end
end)

-- Bunny Hop
UserInputService.InputBegan:Connect(function(inp, gp)
    if gp then return end
    if inp.KeyCode == Enum.KeyCode.Space and Config.BunnyHop then
        local hum = getHumanoid(LocalPlayer)
        if hum then hum.Jump = true end
    end
end)

-- ====================================================
--                    FLING
-- ====================================================

local function flingNearest()
    local myRoot = getRootPart(LocalPlayer)
    if not myRoot then return end
    local closest, closestD = nil, math.huge
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and isAlive(p) then
            local r = getRootPart(p)
            if r then
                local d = (myRoot.Position - r.Position).Magnitude
                if d < closestD then closestD = d; closest = r end
            end
        end
    end
    if closest then
        local att = Instance.new("Attachment"); att.Parent = closest
        local vf  = Instance.new("VectorForce")
        vf.Attachment0 = att
        vf.Force = Vector3.new(
            math.random(-1,1) * Config.FlingForce,
            Config.FlingForce,
            math.random(-1,1) * Config.FlingForce
        )
        vf.Parent = closest
        task.delay(0.1, function() vf:Destroy(); att:Destroy() end)
    end
end

-- ====================================================
--                       GUI
-- ====================================================

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name           = "DavidHubGUI_v3"
ScreenGui.ResetOnSpawn   = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent         = LocalPlayer.PlayerGui

local MainFrame = Instance.new("Frame")
MainFrame.Size             = UDim2.new(0, 390, 0, 480)
MainFrame.Position         = UDim2.new(0.5, -195, 0.5, -240)
MainFrame.BackgroundColor3 = Color3.fromRGB(16, 16, 16)
MainFrame.BorderSizePixel  = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent           = ScreenGui
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 14)

local uiStroke = Instance.new("UIStroke")
uiStroke.Color     = Color3.fromRGB(200, 0, 220)
uiStroke.Thickness = 2
uiStroke.Parent    = MainFrame

-- Title bar
local TitleBar = Instance.new("Frame")
TitleBar.Size            = UDim2.new(1, 0, 0, 44)
TitleBar.BackgroundColor3= Color3.fromRGB(10,10,10)
TitleBar.BorderSizePixel = 0
TitleBar.Parent          = MainFrame

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size               = UDim2.new(1, -50, 1, 0)
TitleLabel.Position           = UDim2.new(0, 14, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text               = "DAVIDHUB  v3"
TitleLabel.Font               = Enum.Font.GothamBold
TitleLabel.TextSize           = 18
TitleLabel.TextColor3         = Color3.fromRGB(255,255,255)
TitleLabel.TextXAlignment     = Enum.TextXAlignment.Left
TitleLabel.Parent             = TitleBar

local CloseBtn = Instance.new("TextButton")
CloseBtn.Size            = UDim2.new(0, 28, 0, 28)
CloseBtn.Position        = UDim2.new(1, -36, 0, 8)
CloseBtn.BackgroundColor3= Color3.fromRGB(180,0,0)
CloseBtn.Text            = "✕"
CloseBtn.Font            = Enum.Font.GothamBold
CloseBtn.TextSize        = 13
CloseBtn.TextColor3      = Color3.fromRGB(255,255,255)
CloseBtn.Parent          = TitleBar
Instance.new("UICorner", CloseBtn).CornerRadius = UDim.new(0,6)
CloseBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
end)

-- Scrollable tab list (left side could be used; here we use top tabs with scroll)
local TabContainer = Instance.new("Frame")
TabContainer.Size            = UDim2.new(1, -20, 0, 34)
TabContainer.Position        = UDim2.new(0, 10, 0, 50)
TabContainer.BackgroundTransparency = 1
TabContainer.ClipsDescendants= true
TabContainer.Parent          = MainFrame

local tabLayout = Instance.new("UIListLayout")
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.Padding       = UDim.new(0, 5)
tabLayout.Parent        = TabContainer

-- Scrollable content
local ScrollFrame = Instance.new("ScrollingFrame")
ScrollFrame.Size             = UDim2.new(1, -20, 1, -96)
ScrollFrame.Position         = UDim2.new(0, 10, 0, 90)
ScrollFrame.BackgroundTransparency = 1
ScrollFrame.BorderSizePixel  = 0
ScrollFrame.ScrollBarThickness = 3
ScrollFrame.ScrollBarImageColor3 = Color3.fromRGB(180,0,220)
ScrollFrame.CanvasSize       = UDim2.new(0,0,0,0)
ScrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
ScrollFrame.Parent           = MainFrame

local contentLayout = Instance.new("UIListLayout")
contentLayout.Padding       = UDim.new(0, 8)
contentLayout.Parent        = ScrollFrame

local UIPadding = Instance.new("UIPaddingCss") -- fallback: UIPadding
pcall(function()
    local p = Instance.new("UIPadding")
    p.PaddingBottom = UDim.new(0, 8)
    p.Parent = ScrollFrame
end)

-- ====================================================
--               HELPER WIDGET BUILDERS
-- ====================================================

local function addLayout(parent)
    for _, c in ipairs(parent:GetChildren()) do
        if c:IsA("UIListLayout") then c:Destroy() end
    end
    local l = Instance.new("UIListLayout")
    l.Padding = UDim.new(0,8)
    l.Parent  = parent
end

local function createToggle(parent, label, state, callback)
    local btn = Instance.new("TextButton")
    btn.Size            = UDim2.new(1, 0, 0, 46)
    btn.BackgroundColor3= state and Color3.fromRGB(160,0,200) or Color3.fromRGB(34,34,34)
    btn.BorderSizePixel = 0
    btn.Text            = label .. ":  " .. (state and "ON ✓" or "OFF")
    btn.Font            = Enum.Font.GothamBold
    btn.TextSize        = 14
    btn.TextColor3      = Color3.fromRGB(255,255,255)
    btn.Parent          = parent
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0,8)

    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.BackgroundColor3 = state and Color3.fromRGB(160,0,200) or Color3.fromRGB(34,34,34)
        btn.Text = label .. ":  " .. (state and "ON ✓" or "OFF")
        callback(state)
    end)
    return btn
end

local function createSlider(parent, labelText, initVal, minVal, maxVal, step, onChange)
    local frame = Instance.new("Frame")
    frame.Size            = UDim2.new(1, 0, 0, 54)
    frame.BackgroundColor3= Color3.fromRGB(26, 26, 26)
    frame.BorderSizePixel = 0
    frame.Parent          = parent
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0,8)

    local topRow = Instance.new("Frame")
    topRow.Size            = UDim2.new(1,-16,0,18)
    topRow.Position        = UDim2.new(0,8,0,6)
    topRow.BackgroundTransparency = 1
    topRow.Parent          = frame

    local lbl = Instance.new("TextLabel")
    lbl.Size              = UDim2.new(0.65,0,1,0)
    lbl.BackgroundTransparency = 1
    lbl.Font              = Enum.Font.Gotham
    lbl.TextSize          = 12
    lbl.TextColor3        = Color3.fromRGB(200,200,200)
    lbl.TextXAlignment    = Enum.TextXAlignment.Left
    lbl.Text              = labelText
    lbl.Parent            = topRow

    local valLbl = Instance.new("TextLabel")
    valLbl.Size           = UDim2.new(0.35,0,1,0)
    valLbl.Position       = UDim2.new(0.65,0,0,0)
    valLbl.BackgroundTransparency = 1
    valLbl.Font           = Enum.Font.GothamBold
    valLbl.TextSize       = 12
    valLbl.TextColor3     = Color3.fromRGB(210,0,240)
    valLbl.TextXAlignment = Enum.TextXAlignment.Right
    valLbl.Text           = tostring(initVal)
    valLbl.Parent         = topRow

    local track = Instance.new("Frame")
    track.Size            = UDim2.new(1,-16,0,7)
    track.Position        = UDim2.new(0,8,0,32)
    track.BackgroundColor3= Color3.fromRGB(45,45,45)
    track.BorderSizePixel = 0
    track.Parent          = frame
    Instance.new("UICorner", track).CornerRadius = UDim.new(1,0)

    local fill = Instance.new("Frame")
    fill.BackgroundColor3= Color3.fromRGB(190,0,220)
    fill.BorderSizePixel = 0
    fill.Size            = UDim2.new((initVal-minVal)/(maxVal-minVal),0,1,0)
    fill.Parent          = track
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1,0)

    local thumb = Instance.new("Frame")
    thumb.Size            = UDim2.new(0,13,0,13)
    thumb.Position        = UDim2.new((initVal-minVal)/(maxVal-minVal),-6,0.5,-6)
    thumb.BackgroundColor3= Color3.fromRGB(255,255,255)
    thumb.BorderSizePixel = 0
    thumb.Parent          = track
    Instance.new("UICorner", thumb).CornerRadius = UDim.new(1,0)

    local value   = initVal
    local isDrag  = false

    local function setVal(v)
        v = math.clamp(math.round((v-minVal)/step)*step+minVal, minVal, maxVal)
        value = v
        local pct = (v-minVal)/(maxVal-minVal)
        fill.Size      = UDim2.new(pct,0,1,0)
        thumb.Position = UDim2.new(pct,-6,0.5,-6)
        valLbl.Text    = tostring(v)
        onChange(v)
    end

    thumb.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1
        or i.UserInputType == Enum.UserInputType.Touch then
            isDrag = true
        end
    end)
    track.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1
        or i.UserInputType == Enum.UserInputType.Touch then
            isDrag = true
            local pct = math.clamp((i.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
            setVal(minVal + pct*(maxVal-minVal))
        end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1
        or i.UserInputType == Enum.UserInputType.Touch then
            isDrag = false
        end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if isDrag then
            local inputX = i.Position.X
            local pct = math.clamp((inputX - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
            setVal(minVal + pct*(maxVal-minVal))
        end
    end)

    return frame, setVal
end

local function createSection(parent, text)
    local lbl = Instance.new("TextLabel")
    lbl.Size              = UDim2.new(1, 0, 0, 22)
    lbl.BackgroundTransparency = 1
    lbl.Font              = Enum.Font.GothamBold
    lbl.TextSize          = 11
    lbl.TextColor3        = Color3.fromRGB(180, 0, 220)
    lbl.TextXAlignment    = Enum.TextXAlignment.Left
    lbl.Text              = "▸  " .. string.upper(text)
    lbl.Parent            = parent
    return lbl
end

-- ====================================================
--              TAB CONTENT DEFINITIONS
-- ====================================================

local tabs = {"Aim", "ESP", "Visual", "Move", "Misc"}
local tabButtons = {}

local tabContents = {

    Aim = function(p)
        createSection(p, "Aimbot")
        createToggle(p, "Aimbot", Config.Aimbot, function(v) Config.Aimbot = v end)
        createSlider(p, "FOV  (bola rosa)", Config.AimbotFOV, 20, 600, 5, function(v) Config.AimbotFOV = v end)
        createSlider(p, "Smooth", math.floor(Config.AimbotSmooth*10), 1, 9, 1, function(v) Config.AimbotSmooth = v/10 end)
        createToggle(p, "Só visível", Config.AimbotVisible, function(v) Config.AimbotVisible = v end)

        createSection(p, "Osso alvo")
        -- Bone selector
        local boneFrame = Instance.new("Frame")
        boneFrame.Size            = UDim2.new(1,0,0,40)
        boneFrame.BackgroundColor3= Color3.fromRGB(26,26,26)
        boneFrame.BorderSizePixel = 0
        boneFrame.Parent          = p
        Instance.new("UICorner", boneFrame).CornerRadius = UDim.new(0,8)

        local headBtn = Instance.new("TextButton")
        headBtn.Size            = UDim2.new(0.5,-4,0,30)
        headBtn.Position        = UDim2.new(0,4,0.5,-15)
        headBtn.BackgroundColor3= Config.AimbotBone == "Head" and Color3.fromRGB(160,0,200) or Color3.fromRGB(44,44,44)
        headBtn.Text            = "Head"
        headBtn.Font            = Enum.Font.GothamBold
        headBtn.TextSize        = 13
        headBtn.TextColor3      = Color3.fromRGB(255,255,255)
        headBtn.Parent          = boneFrame
        Instance.new("UICorner", headBtn).CornerRadius = UDim.new(0,6)

        local bodyBtn = Instance.new("TextButton")
        bodyBtn.Size            = UDim2.new(0.5,-4,0,30)
        bodyBtn.Position        = UDim2.new(0.5,0,0.5,-15)
        bodyBtn.BackgroundColor3= Config.AimbotBone == "HumanoidRootPart" and Color3.fromRGB(160,0,200) or Color3.fromRGB(44,44,44)
        bodyBtn.Text            = "Body"
        bodyBtn.Font            = Enum.Font.GothamBold
        bodyBtn.TextSize        = 13
        bodyBtn.TextColor3      = Color3.fromRGB(255,255,255)
        bodyBtn.Parent          = boneFrame
        Instance.new("UICorner", bodyBtn).CornerRadius = UDim.new(0,6)

        headBtn.MouseButton1Click:Connect(function()
            Config.AimbotBone = "Head"
            headBtn.BackgroundColor3 = Color3.fromRGB(160,0,200)
            bodyBtn.BackgroundColor3 = Color3.fromRGB(44,44,44)
        end)
        bodyBtn.MouseButton1Click:Connect(function()
            Config.AimbotBone = "HumanoidRootPart"
            bodyBtn.BackgroundColor3 = Color3.fromRGB(160,0,200)
            headBtn.BackgroundColor3 = Color3.fromRGB(44,44,44)
        end)
    end,

    ESP = function(p)
        createSection(p, "ESP Drawing")
        createToggle(p, "ESP Ativo", Config.ESP, function(v)
            Config.ESP = v
        end)
        createToggle(p, "Caixa 2D", Config.ESPBox, function(v) Config.ESPBox = v end)
        createToggle(p, "Nome", Config.ESPName, function(v) Config.ESPName = v end)
        createToggle(p, "Distância", Config.ESPDist, function(v) Config.ESPDist = v end)
        createToggle(p, "Barra de Vida", Config.ESPHealth, function(v) Config.ESPHealth = v end)
        createToggle(p, "Esqueleto", Config.ESPSkeleton, function(v) Config.ESPSkeleton = v end)
        createToggle(p, "Tracer (linha)", Config.ESPTracer, function(v) Config.ESPTracer = v end)
        createToggle(p, "Cor do Time", Config.ESPTeamColor, function(v) Config.ESPTeamColor = v end)
        createSlider(p, "Dist. Máxima", Config.ESPMaxDist, 50, 1000, 50, function(v) Config.ESPMaxDist = v end)

        createSection(p, "3D / Chams")
        createToggle(p, "Highlight 3D", Config.Highlight, function(v)
            Config.Highlight = v
            refreshHighlight()
        end)
        createToggle(p, "Chams (Neon)", Config.Chams, function(v)
            Config.Chams = v
            refreshChams()
        end)
    end,

    Visual = function(p)
        createSection(p, "Visuais")
        createToggle(p, "Crosshair", Config.CrosshairDraw, function(v) Config.CrosshairDraw = v end)
        createToggle(p, "FullBright", Config.FullBright, function(v)
            Config.FullBright = v
            setFullBright(v)
        end)
    end,

    Move = function(p)
        createSection(p, "Movimento")
        createToggle(p, "Fly", Config.Fly, function(v)
            Config.Fly = v
            if v then enableFly() end
        end)
        createSlider(p, "Velocidade Fly", Config.FlySpeed, 10, 300, 10, function(v) Config.FlySpeed = v end)
        createToggle(p, "Turbo Speed", Config.Turbo, function(v) Config.Turbo = v end)
        createSlider(p, "WalkSpeed", Config.TurboSpeed, 16, 300, 5, function(v) Config.TurboSpeed = v end)
        createToggle(p, "Noclip", Config.Noclip, function(v) Config.Noclip = v end)
        createToggle(p, "Infinite Jump", Config.InfJump, function(v) Config.InfJump = v end)
        createToggle(p, "Bunny Hop", Config.BunnyHop, function(v) Config.BunnyHop = v end)
        createToggle(p, "No Fall Dmg", Config.NoFall, function(v) Config.NoFall = v end)

        createSection(p, "Fling")
        createToggle(p, "Auto Fling", Config.Fling, function(v) Config.Fling = v end)
        local fBtn = Instance.new("TextButton")
        fBtn.Size            = UDim2.new(1,0,0,46)
        fBtn.BackgroundColor3= Color3.fromRGB(160,0,200)
        fBtn.Text            = "⚡  FLING NEAREST"
        fBtn.Font            = Enum.Font.GothamBold
        fBtn.TextSize        = 14
        fBtn.TextColor3      = Color3.fromRGB(255,255,255)
        fBtn.BorderSizePixel = 0
        fBtn.Parent          = p
        Instance.new("UICorner", fBtn).CornerRadius = UDim.new(0,8)
        fBtn.MouseButton1Click:Connect(flingNearest)
        createSlider(p, "Força Fling", Config.FlingForce/1000, 100, 5000, 100, function(v) Config.FlingForce = v*1000 end)
    end,

    Misc = function(p)
        createSection(p, "Utilitários")
        createToggle(p, "Anti-AFK", Config.AntiAFK, function(v) Config.AntiAFK = v end)

        -- Teleport to nearest
        local tpBtn = Instance.new("TextButton")
        tpBtn.Size            = UDim2.new(1,0,0,46)
        tpBtn.BackgroundColor3= Color3.fromRGB(34,34,34)
        tpBtn.Text            = "📍  TP p/ mais próximo"
        tpBtn.Font            = Enum.Font.GothamBold
        tpBtn.TextSize        = 13
        tpBtn.TextColor3      = Color3.fromRGB(255,255,255)
        tpBtn.BorderSizePixel = 0
        tpBtn.Parent          = p
        Instance.new("UICorner", tpBtn).CornerRadius = UDim.new(0,8)
        tpBtn.MouseButton1Click:Connect(function()
            local myRoot = getRootPart(LocalPlayer)
            if not myRoot then return end
            local closest, closestD = nil, math.huge
            for _, pl in ipairs(Players:GetPlayers()) do
                if pl ~= LocalPlayer and isAlive(pl) then
                    local r = getRootPart(pl)
                    if r then
                        local d = (myRoot.Position - r.Position).Magnitude
                        if d < closestD then closestD = d; closest = r end
                    end
                end
            end
            if closest then
                myRoot.CFrame = closest.CFrame * CFrame.new(0, 0, 3)
            end
        end)

        -- Kill aura (loopkill via fling spam)
        createSection(p, "Kill Aura")
        createToggle(p, "Kill Aura", false, function(v)
            if v then
                task.spawn(function()
                    while v do
                        flingNearest()
                        task.wait(0.15)
                    end
                end)
            end
        end)

        createSection(p, "Info")
        local infoLbl = Instance.new("TextLabel")
        infoLbl.Size  = UDim2.new(1,0,0,54)
        infoLbl.BackgroundColor3 = Color3.fromRGB(22,22,22)
        infoLbl.BorderSizePixel  = 0
        infoLbl.Font  = Enum.Font.Gotham
        infoLbl.TextSize = 11
        infoLbl.TextColor3 = Color3.fromRGB(160,160,160)
        infoLbl.TextWrapped = true
        infoLbl.Text  = "RightAlt / Insert = abrir/fechar\nDavidHub v3 — ESP Drawing API\nUse apenas em servidores privados"
        infoLbl.Parent = p
        Instance.new("UICorner", infoLbl).CornerRadius = UDim.new(0,8)
    end,
}

-- ====================================================
--                  TAB SWITCHING
-- ====================================================

local function showTab(name)
    for _, c in ipairs(ScrollFrame:GetChildren()) do
        if c:IsA("GuiObject") or c:IsA("UIListLayout") or c:IsA("UIPadding") then c:Destroy() end
    end

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0,8)
    layout.Parent  = ScrollFrame

    local pad = Instance.new("UIPadding")
    pad.PaddingBottom = UDim.new(0,10)
    pad.Parent = ScrollFrame

    for tName, btn in pairs(tabButtons) do
        if tName == name then
            btn.BackgroundColor3 = Color3.fromRGB(160,0,200)
            btn.TextColor3       = Color3.fromRGB(255,255,255)
        else
            btn.BackgroundColor3 = Color3.fromRGB(32,32,32)
            btn.TextColor3       = Color3.fromRGB(150,150,150)
        end
    end

    if tabContents[name] then tabContents[name](ScrollFrame) end
end

for _, tabName in ipairs(tabs) do
    local btn = Instance.new("TextButton")
    btn.Size            = UDim2.new(0, 62, 1, 0)
    btn.BackgroundColor3= Color3.fromRGB(32,32,32)
    btn.Text            = tabName
    btn.Font            = Enum.Font.GothamBold
    btn.TextSize        = 12
    btn.TextColor3      = Color3.fromRGB(150,150,150)
    btn.BorderSizePixel = 0
    btn.Parent          = TabContainer
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0,7)

    tabButtons[tabName] = btn
    btn.MouseButton1Click:Connect(function() showTab(tabName) end)
end

showTab("Aim")

-- ====================================================
--                 DRAG (arrastar menu)
-- ====================================================

local isDragging, dragStart, startPos
TitleBar.InputBegan:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseButton1
    or inp.UserInputType == Enum.UserInputType.Touch then
        isDragging = true
        dragStart  = inp.Position
        startPos   = MainFrame.Position
    end
end)
UserInputService.InputChanged:Connect(function(inp)
    if isDragging then
        local d = inp.Position - dragStart
        MainFrame.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + d.X,
            startPos.Y.Scale, startPos.Y.Offset + d.Y
        )
    end
end)
UserInputService.InputEnded:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseButton1
    or inp.UserInputType == Enum.UserInputType.Touch then
        isDragging = false
    end
end)

-- ====================================================
--              TOGGLE GUI (RightAlt / Insert)
-- ====================================================

UserInputService.InputBegan:Connect(function(inp, gp)
    if gp then return end
    if inp.KeyCode == Enum.KeyCode.RightAlt
    or inp.KeyCode == Enum.KeyCode.Insert then
        MainFrame.Visible = not MainFrame.Visible
    end
end)

-- ====================================================
print("[DavidHub v3] ✓ Carregado com sucesso!")
print("[DavidHub v3] RightAlt / Insert = abrir/fechar")
print("[DavidHub v3] ESP: Box 2D + Nome + HP + Skeleton + Tracer")
print("[DavidHub v3] Aimbot: FOV ajustável + bone selector")
-- ====================================================
