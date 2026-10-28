-- zerosuite rivals v2 · Velocity / Xeno · endurecido
-- imprime status antes de tocar em Drawing; GUI em xpcall; aim/esp sobrevivem sem GUI

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace  = game:GetService("Workspace")
local UIS        = game:GetService("UserInputService")

local LP  = Players.LocalPlayer
local Cam = Workspace.CurrentCamera

local genv = getgenv or function() return _G end
local function iround(v) return math.floor(v + 0.5) end
local function clamp(v, a, b) if v < a then return a elseif v > b then return b end return v end

-- ============================================================ status primeiro

local EXEC = "?"
pcall(function() if identifyexecutor then EXEC = identifyexecutor() end end)
local HAS_DRAW = (typeof(Drawing) == "table") and (Drawing.new ~= nil)

local function log(...)
    local ok, s = pcall(function()
        local parts = {}
        for i = 1, select("#", ...) do parts[i] = tostring((select(i, ...))) end
        return table.concat(parts, " ")
    end)
    print("[zerosuite] " .. (ok and s or "?"))
end

log("carregando ·", EXEC, "· drawing =", tostring(HAS_DRAW))

local CFG = {
    aim = true, aim_key = Enum.KeyCode.E, aim_part = "Head",
    fov = 130, smooth = 0.22, prediction = 0.045, aim_range = 450,
    visible_only = true, team_check = true, jitter = 0.0,
    esp = true, box = true, name = true, health = true, distance = true,
    tracer = false, head_dot = true, skeleton = false,
    esp_range = 500, esp_limit = 30, fov_circle = true,
    menu_key = Enum.KeyCode.RightShift, panic_key = Enum.KeyCode.F8,
}
local Stats = { fps = 0, target = "-" }

-- ============================================================ players

local SKELETON = {
    R6  = { {"Head","Torso"},{"Torso","Left Arm"},{"Torso","Right Arm"},{"Torso","Left Leg"},{"Torso","Right Leg"} },
    R15 = { {"Head","UpperTorso"},{"UpperTorso","LowerTorso"},
            {"LowerTorso","LeftUpperLeg"},{"LeftUpperLeg","LeftLowerLeg"},
            {"LowerTorso","RightUpperLeg"},{"RightUpperLeg","RightLowerLeg"},
            {"UpperTorso","LeftUpperArm"},{"LeftUpperArm","LeftLowerArm"},
            {"UpperTorso","RightUpperArm"},{"RightUpperArm","RightLowerArm"} },
}

local recs = {}

local function build(p)
    local rec = recs[p]
    if not rec then
        rec = { lines = {} }
        recs[p] = rec
    end
    rec.parts = nil
    local char = p.Character
    if not char then return rec end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    local head = char:FindFirstChild("Head")
    if not (hum and root and head) then return rec end

    local set = char:FindFirstChild("UpperTorso") and SKELETON.R15 or SKELETON.R6
    local bones = {}
    for _, pair in ipairs(set) do
        local a, b = char:FindFirstChild(pair[1]), char:FindFirstChild(pair[2])
        if a and b then bones[#bones + 1] = { a, b } end
    end
    rec.parts = { char = char, hum = hum, root = root, head = head, bones = bones }
    rec.label = (p.DisplayName ~= p.Name) and p.DisplayName or p.Name
    return rec
end

local function bind(p)
    local rec = build(p)
    if rec.ca then rec.ca:Disconnect() end
    if rec.cr then rec.cr:Disconnect() end
    rec.ca = p.CharacterAdded:Connect(function() task.defer(function() build(p) end) end)
    rec.cr = p.CharacterRemoving:Connect(function() rec.parts = nil end)
end

for _, p in ipairs(Players:GetPlayers()) do bind(p) end
Players.PlayerAdded:Connect(bind)
Players.PlayerRemoving:Connect(function(p) recs[p] = nil end)

local function my_root()
    local c = LP.Character
    return c and c:FindFirstChild("HumanoidRootPart")
end

local function same_team(p)
    if not CFG.team_check or not LP.Team then return false end
    return p.Team == LP.Team
end

local function project(pos)
    if not Cam then return nil end
    local s = Cam:WorldToViewportPoint(pos)
    if s.Z <= 0 then return nil end
    return s.X, s.Y
end

-- ============================================================ aim

local ray = RaycastParams.new()
ray.IgnoreWater = true
pcall(function() ray.FilterType = Enum.RaycastFilterType.Exclude end)
local rayf = { nil, nil }
ray.FilterDescendantsInstances = rayf

local function aim_part(rec)
    local parts = rec.parts
    if not parts then return nil end
    local p = parts.char:FindFirstChild(CFG.aim_part)
    if p and p:IsA("BasePart") then return p end
    return parts.head
end

local function pick()
    if not Cam then return nil end
    local root = my_root()
    if not root then return nil end
    local vp = Cam.ViewportSize
    local cx, cy = vp.X * 0.5, vp.Y * 0.5
    local campos = Cam.CFrame.Position
    local look = Cam.CFrame.LookVector
    local best, best_d2 = nil, CFG.fov * CFG.fov

    for p, rec in pairs(recs) do
        local parts = rec.parts
        if p ~= LP and parts and parts.hum and parts.hum.Health > 0 and not same_team(p) then
            local part = aim_part(rec)
            if part then
                local delta = part.Position - campos
                local dist = delta.Magnitude
                if dist > 4 and dist <= CFG.aim_range and delta.Unit:Dot(look) > 0.15 then
                    local sx, sy = project(part.Position)
                    if sx then
                        local dx, dy = sx - cx, sy - cy
                        local d2 = dx * dx + dy * dy
                        if d2 < best_d2 then
                            best_d2 = d2
                            best = { part = part, rec = rec, p = p }
                        end
                    end
                end
            end
        end
    end
    return best
end

local function aim_step(dt)
    if not CFG.aim or not Cam then Stats.target = "-" return end
    if not UIS:IsKeyDown(CFG.aim_key) then Stats.target = "-" return end

    local t = pick()
    if not t then Stats.target = "-" return end

    local campos = Cam.CFrame.Position
    if CFG.visible_only then
        rayf[1] = LP.Character
        rayf[2] = t.rec.parts.char
        if Workspace:Raycast(campos, t.part.Position - campos, ray) then
            Stats.target = "-"
            return
        end
    end

    Stats.target = t.rec.label or t.p.Name

    local pos = t.part.Position
    if CFG.prediction > 0 then
        pos = pos + (t.part.AssemblyLinearVelocity or Vector3.zero) * CFG.prediction
    end

    local alpha = 1 - math.exp(-dt / math.max(0.004, CFG.smooth))
    local desired = CFrame.lookAt(campos, pos)
    if CFG.jitter > 0 then
        local j = math.rad(CFG.jitter)
        desired = desired * CFrame.Angles((math.random() - 0.5) * j, (math.random() - 0.5) * j, 0)
    end
    Cam.CFrame = Cam.CFrame:Lerp(desired, alpha)
end

RunService.RenderStepped:Connect(function(dt)
    pcall(aim_step, dt)
    for p, rec in pairs(recs) do
        if not rec.parts and p.Character then build(p) end
    end
    Stats.fps = Stats.fps * 0.9 + (1 / math.max(dt, 0.001)) * 0.1
end)

-- ============================================================ drawing (tudo em pcall)

local D = {}          -- nós nomeados
local nodes = {}
local draw_ready = false

local function safe_new(kind)
    if not HAS_DRAW then return nil end
    local ok, obj = pcall(function() return Drawing.new(kind) end)
    if not ok or not obj then return nil end
    pcall(function()
        obj.Visible = false
        obj.Thickness = 1
        obj.Transparency = 1
    end)
    return obj
end

local function setup_drawing()
    local rect  = safe_new("Square")
    local text  = safe_new("Text")
    local line  = safe_new("Line")
    local circ  = safe_new("Circle")
    if not (rect and text and line) then
        log("Drawing incompleto: rect=", tostring(rect ~= nil), "text=", tostring(text ~= nil),
            "line=", tostring(line ~= nil))
        return false
    end

    local function add(o)
        if not o then return nil end
        local n = { o = o }
        nodes[#nodes + 1] = n
        return n
    end

    D.panel   = add(safe_new("Square"))
    D.header  = add(safe_new("Square"))
    D.stripe  = add(safe_new("Square"))
    D.title   = add(safe_new("Text"))
    D.sub     = add(safe_new("Text"))
    D.status  = add(safe_new("Text"))
    D.fov     = circ and add(circ)
    draw_ready = true
    return true
end

local function see(n) if n and not n.v then n.o.Visible = true; n.v = true end end
local function unsee(n) if n and n.v then n.o.Visible = false; n.v = false end end
local function tint(n, c) if n and n.c ~= c then n.o.Color = c; n.c = c end end

local function put_rect(n, x, y, w, h)
    if not n then return end
    x, y, w, h = iround(x), iround(y), math.max(1, iround(w)), math.max(1, iround(h))
    if n.x ~= x or n.y ~= y or n.w ~= w or n.h ~= h then
        pcallsafe(function()
            n.o.Position = Vector2.new(x, y)
            n.o.Size = Vector2.new(w, h)
        end)
        n.x, n.y, n.w, n.h = x, y, w, h
    end
    see(n)
end

local function put_text(n, x, y, s)
    if not n then return end
    x, y = iround(x), iround(y)
    if n.x ~= x or n.y ~= y then
        pcallsafe(function() n.o.Position = Vector2.new(x, y) end)
        n.x, n.y = x, y
    end
    if n.t ~= s then
        pcallsafe(function() n.o.Text = s end)
        n.t = s
    end
    see(n)
end

function pcallsafe(f) pcall(f) end

log("entrando na GUI")
setup_drawing()

-- painel mínimo: só desenha se o Drawing aguentou
local gui = { open = true, x = 120, y = 120 }
RunService.RenderStepped:Connect(function()
    if not draw_ready then return end
    pcallsafe(function()
        if not gui.open then
            for _, n in pairs(D) do unsee(n) end
            return
        end
        tint(D.panel, Color3.fromRGB(12, 10, 18))
        tint(D.header, Color3.fromRGB(22, 18, 32))
        tint(D.stripe, Color3.fromRGB(244, 63, 94))
        put_rect(D.panel, gui.x, gui.y, 320, 120)
        put_rect(D.header, gui.x, gui.y, 320, 34)
        put_rect(D.stripe, gui.x, gui.y, 3, 120)
        tint(D.title, Color3.fromRGB(244, 238, 248))
        tint(D.sub, Color3.fromRGB(253, 205, 211))
        tint(D.status, Color3.fromRGB(150, 132, 168))
        put_text(D.title, gui.x + 14, gui.y + 7, "ZEROSUITE")
        put_text(D.sub, gui.x + 14, gui.y + 40, "rivals · " .. EXEC)
        put_text(D.status, gui.x + 14, gui.y + 62,
            string.format("alvo: %s   fps: %d", Stats.target, iround(Stats.fps)))
        put_text(D.status, gui.x + 14, gui.y + 84,
            string.format("%s abre/fecha · %s mira · %s panic", "RShift", CFG.aim_key.Name, "F8"))
    end)
end)

UIS.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == CFG.menu_key then
        gui.open = not gui.open
    elseif input.KeyCode == CFG.panic_key then
        for _, n in ipairs(nodes) do pcall(function() n.o:Remove() end) end
        genv().rivals = nil
        log("panic: script descarregado")
    end
end)

genv().rivals = {
    cfg = CFG, stats = Stats,
    unload = function()
        for _, n in ipairs(nodes) do pcall(function() n.o:Remove() end) end
        genv().rivals = nil
    end,
}

log("pronto · drawing =", tostring(draw_ready), "· aim =", tostring(CFG.aim))
