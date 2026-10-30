--=====================================================================
-- ZERO // universal hub
-- language: Lua (Luau) | target: Roblox client | env: Xeno (Android/Win)
-- modules: aimbot | esp | fly | teleport book | protection | unload
-- no RemoteEvents fired. aim/esp are render-side only = zero server footprint.
--=====================================================================

if type(getgenv) == "function" and type(getgenv().ZERO_CLEANUP) == "function" then
	pcall(getgenv().ZERO_CLEANUP)
end

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS        = game:GetService("UserInputService")
local Tween      = game:GetService("TweenService")
local Http       = game:GetService("HttpService")
local Workspace  = game:GetService("Workspace")

local LP     = Players.LocalPlayer
local unpack = table.unpack or unpack

local CONN, DRAW = {}, {}
local function track(c) CONN[#CONN + 1] = c; return c end
local function bind(sig, fn) return track(sig:Connect(fn)) end

local COL = {
	bg      = Color3.fromRGB(15, 15, 19),
	panel   = Color3.fromRGB(21, 21, 27),
	panel2  = Color3.fromRGB(29, 29, 38),
	stroke  = Color3.fromRGB(44, 44, 58),
	text    = Color3.fromRGB(236, 236, 245),
	dim     = Color3.fromRGB(138, 138, 160),
	accent  = Color3.fromRGB(138, 92, 255),
	accent2 = Color3.fromRGB(80, 205, 255),
	off     = Color3.fromRGB(48, 48, 62),
	danger  = Color3.fromRGB(255, 84, 104),
}

local DrawingLib
pcall(function() DrawingLib = Drawing end)

local function new(class, props, parent)
	local inst = Instance.new(class)
	for k, v in next, props do inst[k] = v end
	if parent then inst.Parent = parent end
	return inst
end
local function corner(inst, r) return new("UICorner", { CornerRadius = UDim.new(0, r or 8) }, inst) end
local function outline(inst, c, t, tr)
	return new("UIStroke", { Color = c or COL.stroke, Thickness = t or 1, Transparency = tr or 0.35 }, inst)
end

local function viewport()
	local cam = Workspace.CurrentCamera
	return (cam and cam.ViewportSize) or Vector2.new(1280, 720)
end
local function compute_scale()
	local vp = viewport()
	return math.clamp(math.min(vp.X / 600, vp.Y / 450), 0.5, 1)
end

--========================================================================
-- UI ROOT
--========================================================================
local PARENT
do
	if type(gethui) == "function" then
		local ok, h = pcall(gethui)
		if ok and h then PARENT = h end
	end
	if not PARENT then
		local ok, pg = pcall(function() return LP:FindFirstChildOfClass("PlayerGui") end)
		PARENT = (ok and pg) or LP:WaitForChild("PlayerGui")
	end
end

local gui = new("ScreenGui", {
	Name = "z" .. tostring(math.random(1000000, 9999999)),
	ResetOnSpawn = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 9999,
}, PARENT)

pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
pcall(function() if protect_gui then protect_gui(gui) end end)
pcall(function() if protectgui then protectgui(gui) end end)

--========================================================================
-- SHARED UI PIECES
--========================================================================
local ORD = setmetatable({}, { __mode = "k" })
local function next_order(parent)
	ORD[parent] = (ORD[parent] or 0) + 1
	return ORD[parent]
end

-- drag: MouseMovement chega em InputObject novo, nunca no mesmo do InputBegan.
-- por isso o estado é só o booleano, sem comparar objeto de input.
local function draggable(frame, handle, scale_get)
	local drag, start, start_pos, start_abs = false, nil, nil, nil

	handle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			drag = true
			start = Vector2.new(input.Position.X, input.Position.Y)
			start_pos = frame.Position
			start_abs = frame.AbsolutePosition
		end
	end)

	track(UIS.InputChanged:Connect(function(input)
		if not drag then return end
		if input.UserInputType ~= Enum.UserInputType.MouseMovement
		and input.UserInputType ~= Enum.UserInputType.Touch then return end

		local s = scale_get() or 1
		local vp = viewport()
		local size = frame.AbsoluteSize

		local raw = Vector2.new(input.Position.X, input.Position.Y) - start
		local target = start_abs + raw

		-- trava: pelo menos 60px de largura e o cabeçalho sempre alcançáveis
		local x = math.clamp(target.X, -(size.X - 60), vp.X - 60)
		local y = math.clamp(target.Y, 36, vp.Y - 46)

		local d = (Vector2.new(x, y) - start_abs) / s
		frame.Position = UDim2.new(
			start_pos.X.Scale, start_pos.X.Offset + d.X,
			start_pos.Y.Scale, start_pos.Y.Offset + d.Y
		)
	end))

	track(UIS.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			drag = false
			start, start_abs = nil, nil
		end
	end))
end

local function hdr_button(header, text, x, cb)
	local b = new("TextButton", {
		Size = UDim2.fromOffset(26, 26),
		Position = UDim2.new(1, x, 0.5, -13),
		BackgroundColor3 = COL.panel2,
		Text = text,
		Font = Enum.Font.GothamBold,
		TextSize = 15,
		TextColor3 = COL.dim,
		AutoButtonColor = false,
	}, header)
	corner(b, 7)
	b.MouseButton1Click:Connect(cb)
	return b
end

--========================================================================
-- STATE
--========================================================================
local aim = {
	enabled = false, held = false, fov = 150, smooth = 4,
	part = "Head", team = false, wall = false, show_fov = true,
	key = Enum.KeyCode.E,
}
local esp = {
	enabled = false, box = true, name = true, health = true,
	dist = true, tracer = false, team = false, max_dist = 900,
}
local fly  = { enabled = false, speed = 90, vert = 0 }
local misc = { antiafk = false }
local tp   = { step = false, list = {}, file = "zero_teleports.json" }

--========================================================================
-- MAIN WINDOW
--========================================================================
local win = new("Frame", {
	Size = UDim2.fromOffset(540, 392),
	Position = UDim2.new(0.5, 0, 0.5, 0),
	AnchorPoint = Vector2.new(0.5, 0.5),
	BackgroundColor3 = COL.bg,
	BorderSizePixel = 0,
	Active = true,
}, gui)
corner(win, 14)
outline(win, COL.stroke, 1, 0.2)
local win_scale = new("UIScale", { Scale = compute_scale() }, win)
draggable(win, win, function() return win_scale.Scale end)

local header = new("Frame", { Size = UDim2.new(1, 0, 0, 46), BackgroundTransparency = 1 }, win)
local dot = new("Frame", {
	Size = UDim2.fromOffset(7, 7),
	Position = UDim2.new(0, 16, 0.5, -3),
	BackgroundColor3 = COL.accent,
}, header)
corner(dot, 4)
new("TextLabel", {
	Size = UDim2.new(0, 90, 1, 0), Position = UDim2.new(0, 30, 0, 0),
	BackgroundTransparency = 1, Text = "ZERO", Font = Enum.Font.GothamBold,
	TextSize = 16, TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
}, header)
new("TextLabel", {
	Size = UDim2.new(0, 120, 1, 0), Position = UDim2.new(0, 78, 0, 1),
	BackgroundTransparency = 1, Text = "hub", Font = Enum.Font.GothamMedium,
	TextSize = 10, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left,
}, header)

local unload_dummy -- forward: atribuído no fim

local dock = new("TextButton", {
	Size = UDim2.fromOffset(52, 52),
	Position = UDim2.new(0, 28, 0, 90),
	BackgroundColor3 = COL.bg, Text = "Z",
	Font = Enum.Font.GothamBold, TextSize = 20, TextColor3 = COL.accent,
	AutoButtonColor = false, Visible = false,
}, gui)
corner(dock, 13)
outline(dock, COL.accent, 1, 0.35)
draggable(dock, dock, function() return 1 end)
dock.MouseButton1Click:Connect(function()
	win.Visible = true
	dock.Visible = false
	win_scale.Scale = compute_scale()
end)

hdr_button(header, "–", -68, function()
	win.Visible = false
	dock.Visible = true
end)
hdr_button(header, "×", -36, function()
	if unload_dummy then unload_dummy() end
end)

-- sidebar
local side = new("Frame", {
	Size = UDim2.new(0, 116, 1, -64),
	Position = UDim2.new(0, 14, 0, 52),
	BackgroundTransparency = 1,
}, win)
new("UIListLayout", { Padding = UDim.new(0, 5), SortOrder = Enum.SortOrder.LayoutOrder }, side)

local body = new("Frame", {
	Size = UDim2.new(1, -150, 1, -64),
	Position = UDim2.new(0, 138, 0, 52),
	BackgroundTransparency = 1,
}, win)

local PAGES, TABS = {}, {}

local function make_page(order)
	local p = new("ScrollingFrame", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ScrollBarThickness = 3,
		ScrollBarImageColor3 = COL.stroke,
		CanvasSize = UDim2.new(0, 0, 0, 0),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
		ScrollingDirection = Enum.ScrollingDirection.Y,
		Visible = false,
		LayoutOrder = order,
	}, body)
	new("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, p)
	new("UIPadding", { PaddingRight = UDim.new(0, 6), PaddingBottom = UDim.new(0, 8) }, p)
	PAGES[#PAGES + 1] = p
	return p
end

local function select_page(page)
	for _, p in ipairs(PAGES) do p.Visible = (p == page) end
	for _, t in ipairs(TABS) do
		local on = (t.page == page)
		Tween:Create(t.btn, TweenInfo.new(0.14), {
			BackgroundColor3 = on and COL.panel2 or COL.panel,
		}):Play()
		t.lbl.TextColor3 = on and COL.text or COL.dim
		t.ind.Visible = on
	end
end

local function make_tab(label, order, page)
	local b = new("TextButton", {
		Size = UDim2.new(1, 0, 0, 36),
		BackgroundColor3 = COL.panel,
		Text = "", AutoButtonColor = false,
		LayoutOrder = order,
	}, side)
	corner(b, 8)
	local ind = new("Frame", {
		Size = UDim2.new(0, 3, 0, 16),
		Position = UDim2.new(0, 0, 0.5, -8),
		BackgroundColor3 = COL.accent, Visible = false,
	}, b)
	corner(ind, 2)
	local lbl = new("TextLabel", {
		Size = UDim2.new(1, -22, 1, 0),
		Position = UDim2.new(0, 14, 0, 0),
		BackgroundTransparency = 1,
		Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left,
	}, b)
	b.MouseButton1Click:Connect(function() select_page(page) end)
	TABS[#TABS + 1] = { btn = b, lbl = lbl, ind = ind, page = page }
	return b
end

--========================================================================
-- COMPONENTS
--========================================================================
local function section(parent, text)
	return new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 26),
		BackgroundTransparency = 1,
		Text = string.upper(text),
		Font = Enum.Font.GothamBold, TextSize = 10,
		TextColor3 = COL.dim,
		TextXAlignment = Enum.TextXAlignment.Left,
		LayoutOrder = next_order(parent),
	}, parent)
end

local function toggle(parent, label, default, cb)
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 34),
		BackgroundColor3 = COL.panel,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -64, 1, 0), Position = UDim2.new(0, 12, 0, 0),
		BackgroundTransparency = 1, Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local btn = new("TextButton", {
		Size = UDim2.fromOffset(38, 20),
		Position = UDim2.new(1, -50, 0.5, -10),
		BackgroundColor3 = COL.off, Text = "", AutoButtonColor = false,
	}, row)
	corner(btn, 10)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(14, 14),
		Position = UDim2.new(0, 3, 0.5, -7),
		BackgroundColor3 = COL.text,
	}, btn)
	corner(knob, 7)
	local state = default and true or false
	local function render()
		Tween:Create(btn, TweenInfo.new(0.14), {
			BackgroundColor3 = state and COL.accent or COL.off,
		}):Play()
		Tween:Create(knob, TweenInfo.new(0.14), {
			Position = state and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7),
		}):Play()
	end
	render()
	local function set(v)
		state = v and true or false
		render()
		if cb then cb(state) end
	end
	btn.MouseButton1Click:Connect(function() set(not state) end)
	return set
end

local function slider(parent, label, min, max, default, step, cb)
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 48),
		BackgroundColor3 = COL.panel,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -110, 0, 18), Position = UDim2.new(0, 12, 0, 7),
		BackgroundTransparency = 1, Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local value_lbl = new("TextLabel", {
		Size = UDim2.new(0, 90, 0, 18), Position = UDim2.new(1, -102, 0, 7),
		BackgroundTransparency = 1, Text = tostring(default),
		Font = Enum.Font.GothamBold, TextSize = 12,
		TextColor3 = COL.accent, TextXAlignment = Enum.TextXAlignment.Right,
	}, row)
	local hit = new("TextButton", {
		Size = UDim2.new(1, -24, 0, 16),
		Position = UDim2.new(0, 12, 0, 27),
		BackgroundTransparency = 1, Text = "",
	}, row)
	local line = new("Frame", {
		Size = UDim2.new(1, 0, 0, 3),
		Position = UDim2.new(0, 0, 0.5, -1),
		BackgroundColor3 = COL.off,
	}, hit)
	corner(line, 2)
	local fill = new("Frame", { Size = UDim2.new(0, 0, 1, 0), BackgroundColor3 = COL.accent }, line)
	corner(fill, 2)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(12, 12),
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(0, 0, 0.5, 0),
		BackgroundColor3 = COL.text,
	}, hit)
	corner(knob, 6)

	local value = default
	local function apply(v, fire)
		v = math.clamp(v, min, max)
		if step and step > 0 then
			v = math.floor((v - min) / step + 0.5) * step + min
		end
		v = math.clamp(v, min, max)
		value = v
		local a = (v - min) / (max - min)
		fill.Size = UDim2.new(a, 0, 1, 0)
		knob.Position = UDim2.new(a, 0, 0.5, 0)
		value_lbl.Text = (step and step < 1)
			and string.format("%.2f", v)
			or tostring(math.floor(v + 0.5))
		if fire and cb then cb(v) end
	end
	apply(default, false)

	local dragging = false
	local function from_x(x)
		local a = (x - hit.AbsolutePosition.X) / math.max(hit.AbsoluteSize.X, 1)
		apply(min + math.clamp(a, 0, 1) * (max - min), true)
	end
	hit.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			from_x(input.Position.X)
		end
	end)
	track(UIS.InputChanged:Connect(function(input)
		if dragging
		and (input.UserInputType == Enum.UserInputType.MouseMovement
		or input.UserInputType == Enum.UserInputType.Touch) then
			from_x(input.Position.X)
		end
	end))
	track(UIS.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end))
	return apply
end

local function cycle(parent, label, options, default_idx, cb)
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 34),
		BackgroundColor3 = COL.panel,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(0.55, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
		BackgroundTransparency = 1, Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local idx = default_idx or 1
	local value_lbl = new("TextLabel", {
		Size = UDim2.new(0.45, -16, 1, 0), Position = UDim2.new(0.55, 0, 0, 0),
		BackgroundTransparency = 1, Text = options[idx],
		Font = Enum.Font.GothamBold, TextSize = 12,
		TextColor3 = COL.accent, TextXAlignment = Enum.TextXAlignment.Right,
	}, row)
	local click = new("TextButton", {
		Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "",
	}, row)
	click.MouseButton1Click:Connect(function()
		idx = idx % #options + 1
		value_lbl.Text = options[idx]
		if cb then cb(options[idx], idx) end
	end)
	return value_lbl
end

local function button(parent, label, cb, color)
	local b = new("TextButton", {
		Size = UDim2.new(1, 0, 0, 34),
		BackgroundColor3 = color or COL.panel2,
		Text = label,
		Font = Enum.Font.GothamBold, TextSize = 11,
		TextColor3 = COL.text, AutoButtonColor = false,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(b, 8)
	b.MouseButton1Click:Connect(function() if cb then cb() end end)
	return b
end

local function hold_button(parent, label, down, up)
	local b = new("TextButton", {
		Size = UDim2.new(0.48, 0, 1, 0),
		BackgroundColor3 = COL.panel2,
		Text = label,
		Font = Enum.Font.GothamBold, TextSize = 12,
		TextColor3 = COL.text, AutoButtonColor = false,
	}, parent)
	corner(b, 8)
	b.MouseButton1Down:Connect(down)
	b.MouseButton1Up:Connect(up)
	b.MouseLeave:Connect(up)
	return b
end

--========================================================================
-- AIMBOT
--========================================================================
local aim_page = make_page(1)
make_tab("AIMBOT", 1, aim_page)

section(aim_page, "aimbot")
local aim_btn_visible
toggle(aim_page, "Enabled", false, function(v)
	aim.enabled = v
	if aim_btn_visible then aim_btn_visible(v) end
end)
cycle(aim_page, "Aim key", { "E", "Q", "F", "C", "V", "T" }, 1, function(v)
	aim.key = Enum.KeyCode[v]
end)
slider(aim_page, "FOV (px)", 40, 400, 150, 5, function(v) aim.fov = v end)
slider(aim_page, "Smooth (1 = snap)", 1, 20, 4, 1, function(v) aim.smooth = v end)
cycle(aim_page, "Target part", { "Head", "Torso" }, 1, function(v) aim.part = v end)
toggle(aim_page, "Team check", false, function(v) aim.team = v end)
toggle(aim_page, "Wall check (raycast)", false, function(v) aim.wall = v end)
toggle(aim_page, "Show FOV circle", true, function(v) aim.show_fov = v end)

local aim_button = new("TextButton", {
	Size = UDim2.fromOffset(68, 68),
	Position = UDim2.new(1, -92, 1, -104),
	BackgroundColor3 = COL.panel,
	BackgroundTransparency = 0.2,
	Text = "AIM",
	Font = Enum.Font.GothamBold, TextSize = 12,
	TextColor3 = COL.text, AutoButtonColor = false, Visible = false,
}, gui)
corner(aim_button, 34)
outline(aim_button, COL.accent, 1, 0.4)
aim_btn_visible = function(v) aim_button.Visible = v end

aim_button.MouseButton1Down:Connect(function()
	aim.held = true
	Tween:Create(aim_button, TweenInfo.new(0.08), { BackgroundColor3 = COL.accent }):Play()
end)
aim_button.MouseButton1Up:Connect(function()
	aim.held = false
	Tween:Create(aim_button, TweenInfo.new(0.08), { BackgroundColor3 = COL.panel }):Play()
end)

local fov_circle
if DrawingLib then
	local ok, c = pcall(function() return DrawingLib.new("Circle") end)
	if ok and c then
		fov_circle = c
		c.Thickness = 1
		c.NumSides = 64
		c.Color = COL.accent
		c.Transparency = 0.45
		c.Filled = false
		c.Visible = false
		DRAW[#DRAW + 1] = c
	end
end

local ray_params = RaycastParams.new()
ray_params.FilterType = Enum.RaycastFilterType.Exclude

local function target_part(char)
	if aim.part == "Head" then return char:FindFirstChild("Head") end
	return char:FindFirstChild("UpperTorso")
		or char:FindFirstChild("Torso")
		or char:FindFirstChild("HumanoidRootPart")
end

local function is_visible(from, part)
	local exclude = {}
	if LP.Character then exclude[#exclude + 1] = LP.Character end
	exclude[#exclude + 1] = part.Parent
	ray_params.FilterDescendantsInstances = exclude
	return Workspace:Raycast(from, part.Position - from, ray_params) == nil
end

local function best_target()
	local cam = Workspace.CurrentCamera
	if not cam then return nil end
	local vp = cam.ViewportSize
	local center = Vector2.new(vp.X / 2, vp.Y / 2)
	local origin = cam.CFrame.Position
	local best, best_d = nil, nil
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LP then
			local skip = false
			if aim.team and p.Team ~= nil and LP.Team ~= nil and p.Team == LP.Team then
				skip = true
			end
			local char = p.Character
			if not skip and char then
				local hum = char:FindFirstChildOfClass("Humanoid")
				local part = target_part(char)
				if hum and part and hum.Health > 0 then
					local sp, on = cam:WorldToViewportPoint(part.Position)
					if on and sp.Z > 0 then
						local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
						if d <= aim.fov and (not best_d or d < best_d) then
							if not aim.wall or is_visible(origin, part) then
								best, best_d = part, d
							end
						end
					end
				end
			end
		end
	end
	return best
end

bind(RunService.RenderStepped, function(dt)
	local cam = Workspace.CurrentCamera
	if not cam then return end

	if fov_circle then
		local vp = cam.ViewportSize
		fov_circle.Position = Vector2.new(vp.X / 2, vp.Y / 2)
		fov_circle.Radius = aim.fov
		fov_circle.Visible = aim.enabled and aim.show_fov
	end

	if not (aim.enabled and aim.held) then return end
	local part = best_target()
	if not part then return end

	local goal = CFrame.lookAt(cam.CFrame.Position, part.Position)
	local alpha = 1 - math.pow(1 - (1 / math.max(aim.smooth, 1)), dt * 60)
	cam.CFrame = cam.CFrame:Lerp(goal, math.clamp(alpha, 0, 1))
end)

bind(UIS.InputBegan, function(input, processed)
	if processed then return end
	if aim.enabled and input.KeyCode == aim.key then aim.held = true end
end)
bind(UIS.InputEnded, function(input)
	if input.KeyCode == aim.key then aim.held = false end
end)

--========================================================================
-- ESP
--========================================================================
local esp_page = make_page(2)
make_tab("ESP", 2, esp_page)

section(esp_page, "esp")
toggle(esp_page, "Enabled", false, function(v) esp.enabled = v end)
toggle(esp_page, "Box", true, function(v) esp.box = v end)
toggle(esp_page, "Name", true, function(v) esp.name = v end)
toggle(esp_page, "Health bar", true, function(v) esp.health = v end)
toggle(esp_page, "Distance", true, function(v) esp.dist = v end)
toggle(esp_page, "Tracer", false, function(v) esp.tracer = v end)
toggle(esp_page, "Team check", false, function(v) esp.team = v end)
slider(esp_page, "Max distance", 50, 2000, 900, 50, function(v) esp.max_dist = v end)

if not DrawingLib then
	local note = new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 46),
		BackgroundColor3 = COL.panel,
		Text = "Drawing lib ausente — fallback: Highlight + Billboard",
		Font = Enum.Font.GothamMedium, TextSize = 11,
		TextColor3 = COL.dim, TextWrapped = true,
		LayoutOrder = next_order(esp_page),
	}, esp_page)
	corner(note, 8)
end

local esp_cache = {}

local function esp_make(player)
	local e = { player = player }
	if DrawingLib then
		local ok, set = pcall(function()
			return {
				box    = DrawingLib.new("Square"),
				name   = DrawingLib.new("Text"),
				hp_bg  = DrawingLib.new("Square"),
				hp     = DrawingLib.new("Square"),
				dist   = DrawingLib.new("Text"),
				tracer = DrawingLib.new("Line"),
			}
		end)
		if ok and set then
			set.box.Filled       = false
			set.box.Thickness    = 1
			set.box.Transparency = 0
			set.hp_bg.Filled      = true
			set.hp_bg.Color       = Color3.new(0, 0, 0)
			set.hp_bg.Transparency = 0.35
			set.hp.Filled    = true
			set.hp.Color     = COL.accent2
			set.name.Center      = true
			set.name.Outline     = true
			set.name.OutlineColor = Color3.new(0, 0, 0)
			set.name.Size = 13
			set.name.Font = 2
			set.dist.Center       = true
			set.dist.Outline      = true
			set.dist.OutlineColor = Color3.new(0, 0, 0)
			set.dist.Size = 12
			set.dist.Font = 2
			set.tracer.Thickness = 1
			e.draw = set
			for _, o in pairs(set) do DRAW[#DRAW + 1] = o end
			return e
		end
	end
	e.fallback = true
	return e
end

local function esp_drop(e)
	if e.draw then
		for _, o in pairs(e.draw) do pcall(function() o:Remove() end) end
	end
	if e.hl then pcall(function() e.hl:Destroy() end) end
	if e.bg then pcall(function() e.bg:Destroy() end) end
end

local function esp_ensure_inst(e, char)
	if e.hl and e.hl.Parent == char then return end
	if e.hl then pcall(function() e.hl:Destroy() end) end
	if e.bg then pcall(function() e.bg:Destroy() end) end
	e.hl = new("Highlight", {
		Adornee = char,
		FillTransparency = 0.75,
		OutlineTransparency = 0,
		DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
	}, char)
	local head = char:FindFirstChild("Head") or char:FindFirstChildWhichIsA("BasePart")
	e.bg = new("BillboardGui", {
		Adornee = head,
		Size = UDim2.fromOffset(140, 20),
		StudsOffset = Vector3.new(0, 2.4, 0),
		AlwaysOnTop = true,
		LightInfluence = 0,
	}, char)
	e.lbl = new("TextLabel", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		Text = "",
		Font = Enum.Font.GothamBold,
		TextSize = 12,
		TextColor3 = Color3.new(1, 1, 1),
		TextStrokeTransparency = 0,
	}, e.bg)
end

local function esp_tick()
	local cam = Workspace.CurrentCamera
	if not cam then return end
	local vp = cam.ViewportSize
	local origin = cam.CFrame.Position
	local seen = {}

	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LP then
			seen[p] = true
			local e = esp_cache[p]
			if not e then
				e = esp_make(p)
				esp_cache[p] = e
			end
			local char = p.Character
			local hum  = char and char:FindFirstChildOfClass("Humanoid")
			local hrp  = char and char:FindFirstChild("HumanoidRootPart")
			local teammate = (p.Team ~= nil and LP.Team ~= nil and p.Team == LP.Team)

			local show = esp.enabled and char and hum and hrp and hum.Health > 0
			if show and esp.team and teammate then show = false end
			local dist = (char and hrp) and (origin - hrp.Position).Magnitude or 0
			if show and dist > esp.max_dist then show = false end

			local col = teammate and COL.accent2 or COL.accent

			if e.draw then
				if not show then
					for _, o in pairs(e.draw) do o.Visible = false end
				else
					local head = char:FindFirstChild("Head") or hrp
					local hs, on1 = cam:WorldToViewportPoint(head.Position + Vector3.new(0, 0.6, 0))
					local fs, on2 = cam:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3.2, 0))
					if on1 and on2 and hs.Z > 0 then
						local h = math.abs(fs.Y - hs.Y)
						local w = h * 0.52
						local x, y = hs.X - w / 2, hs.Y

						e.draw.box.Position = Vector2.new(x, y)
						e.draw.box.Size = Vector2.new(w, h)
						e.draw.box.Color = col
						e.draw.box.Visible = esp.box

						local frac = math.clamp(hum.Health / math.max(hum.MaxHealth, 1), 0, 1)
						e.draw.hp_bg.Position = Vector2.new(x - 5, y)
						e.draw.hp_bg.Size = Vector2.new(2, h)
						e.draw.hp.Position = Vector2.new(x - 5, y + h * (1 - frac))
						e.draw.hp.Size = Vector2.new(2, h * frac)
						e.draw.hp.Color = COL.accent2
						e.draw.hp_bg.Visible = esp.health
						e.draw.hp.Visible = esp.health

						e.draw.name.Position = Vector2.new(hs.X, y - 13)
						e.draw.name.Text = p.Name
						e.draw.name.Color = Color3.new(1, 1, 1)
						e.draw.name.Visible = esp.name

						e.draw.dist.Position = Vector2.new(hs.X, y + h + 2)
						e.draw.dist.Text = tostring(math.floor(dist))
						e.draw.dist.Color = col
						e.draw.dist.Visible = esp.dist

						e.draw.tracer.From = Vector2.new(vp.X / 2, vp.Y)
						e.draw.tracer.To = Vector2.new(hs.X, y + h)
						e.draw.tracer.Color = col
						e.draw.tracer.Visible = esp.tracer
					else
						for _, o in pairs(e.draw) do o.Visible = false end
					end
				end
			else
				if show then
					esp_ensure_inst(e, char)
					e.hl.FillColor = col
					e.hl.OutlineColor = col
					if e.lbl then
						e.lbl.Text = string.format("%s  %d%%  %d",
							p.Name,
							math.floor((hum.Health / math.max(hum.MaxHealth, 1)) * 100),
							math.floor(dist))
					end
				else
					if e.hl then e.hl.Enabled = false end
					if e.bg then e.bg.Enabled = false end
				end
				if show then
					if e.hl then e.hl.Enabled = true end
					if e.bg then e.bg.Enabled = true end
				end
			end
		end
	end

	for p, e in pairs(esp_cache) do
		if not seen[p] then
			esp_drop(e)
			esp_cache[p] = nil
		end
	end
end

local esp_acc = 0
bind(RunService.RenderStepped, function(dt)
	esp_acc = esp_acc + dt
	if esp_acc < 0.02 then return end
	esp_acc = 0
	esp_tick()
end)

--========================================================================
-- FLY  (CFrame puro — nenhuma instância criada, nada pra scanner achar)
--========================================================================
local fly_page = make_page(3)
make_tab("FLY", 3, fly_page)

section(fly_page, "fly")
toggle(fly_page, "Enabled", false, function(v)
	fly.enabled = v
	if not v then fly.vert = 0 end
end)
slider(fly_page, "Speed", 10, 400, 90, 5, function(v) fly.speed = v end)

local vertical_row = new("Frame", {
	Size = UDim2.new(1, 0, 0, 34),
	BackgroundTransparency = 1,
	LayoutOrder = next_order(fly_page),
}, fly_page)
local up_btn = hold_button(vertical_row, "▲  UP", function()
	fly.vert = 1
end, function()
	if fly.vert == 1 then fly.vert = 0 end
end)
up_btn.Position = UDim2.new(0, 0, 0, 0)
local dn_btn = hold_button(vertical_row, "▼  DOWN", function()
	fly.vert = -1
end, function()
	if fly.vert == -1 then fly.vert = 0 end
end)
dn_btn.Position = UDim2.new(0.52, 0, 0, 0)

local fly_note = new("TextLabel", {
	Size = UDim2.new(1, 0, 0, 30),
	BackgroundTransparency = 1,
	Text = "thumbstick/WASD move · ▲▼ or Space/LShift vertical",
	Font = Enum.Font.GothamMedium, TextSize = 10,
	TextColor3 = COL.dim, TextWrapped = true,
	LayoutOrder = next_order(fly_page),
}, fly_page)

bind(UIS.InputBegan, function(input, processed)
	if processed or not fly.enabled then return end
	if input.KeyCode == Enum.KeyCode.Space then fly.vert = 1 end
	if input.KeyCode == Enum.KeyCode.LeftShift then fly.vert = -1 end
end)
bind(UIS.InputEnded, function(input)
	if input.KeyCode == Enum.KeyCode.Space and fly.vert == 1 then fly.vert = 0 end
	if input.KeyCode == Enum.KeyCode.LeftShift and fly.vert == -1 then fly.vert = 0 end
end)

bind(RunService.RenderStepped, function(dt)
	if not fly.enabled then return end
	local char = LP.Character
	local hrp  = char and char:FindFirstChild("HumanoidRootPart")
	local hum  = char and char:FindFirstChildOfClass("Humanoid")
	if not hrp or not hum or hum.Health <= 0 then return end

	local dir = hum.MoveDirection
	if fly.vert ~= 0 then
		dir = dir + Vector3.new(0, fly.vert, 0)
	end
	if dir.Magnitude < 0.01 then
		pcall(function() hrp.AssemblyLinearVelocity = Vector3.zero end)
		return
	end
	local step = dir.Unit * fly.speed * dt
	pcall(function()
		hrp.AssemblyLinearVelocity = Vector3.zero
		hrp.CFrame = hrp.CFrame + step
	end)
end)

--========================================================================
-- MISC / PROTECTION
--========================================================================
local kick_orig = nil
local function set_kick_guard(on)
	if type(hookfunction) ~= "function" then return end
	pcall(function()
		if on and not kick_orig then
			kick_orig = hookfunction(LP.Kick, function() end)
		elseif (not on) and kick_orig then
			hookfunction(LP.Kick, kick_orig)
			kick_orig = nil
		end
	end)
end

local misc_page = make_page(4)
make_tab("MISC", 4, misc_page)

section(misc_page, "protection")
toggle(misc_page, "Anti-AFK", false, function(v)
	misc.antiafk = v
end)
toggle(misc_page, "Block client-side kick", false, set_kick_guard)

section(misc_page, "session")
local unload_btn = button(misc_page, "UNLOAD / REMOVE", function()
	if unload_dummy then unload_dummy() end
end, COL.danger)

bind(LP.Idled, function()
	if not misc.antiafk then return end
	pcall(function()
		local vu = game:GetService("VirtualUser")
		vu:CaptureController()
		vu:ClickButton2(Vector2.new())
	end)
end)

--========================================================================
-- TELEPORT BOOK
--========================================================================
local function tp_load()
	pcall(function()
		if type(isfile) == "function" and isfile(tp.file) then
			tp.list = Http:JSONDecode(readfile(tp.file)) or {}
		end
	end)
end
local function tp_save()
	pcall(function()
		if type(writefile) == "function" then
			writefile(tp.file, Http:JSONEncode(tp.list))
		end
	end)
end
tp_load()

local tpw = new("Frame", {
	Size = UDim2.fromOffset(304, 392),
	Position = UDim2.new(0.5, 300, 0.5, 0),
	AnchorPoint = Vector2.new(0, 0.5),
	BackgroundColor3 = COL.bg,
	BorderSizePixel = 0,
	Visible = false,
	Active = true,
}, gui)
corner(tpw, 14)
outline(tpw, COL.stroke, 1, 0.2)
local tp_scale = new("UIScale", { Scale = compute_scale() }, tpw)
draggable(tpw, tpw, function() return tp_scale.Scale end)

local tp_header = new("Frame", { Size = UDim2.new(1, 0, 0, 44), BackgroundTransparency = 1 }, tpw)
local tp_dot = new("Frame", {
	Size = UDim2.fromOffset(7, 7),
	Position = UDim2.new(0, 16, 0.5, -3),
	BackgroundColor3 = COL.accent2,
}, tp_header)
corner(tp_dot, 4)
new("TextLabel", {
	Size = UDim2.new(0, 180, 1, 0), Position = UDim2.new(0, 30, 0, 0),
	BackgroundTransparency = 1, Text = "TELEPORTS", Font = Enum.Font.GothamBold,
	TextSize = 14, TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
}, tp_header)
hdr_button(tp_header, "×", -36, function() tpw.Visible = false end)

local tp_body = new("Frame", {
	Size = UDim2.new(1, -24, 1, -56),
	Position = UDim2.new(0, 12, 0, 48),
	BackgroundTransparency = 1,
}, tpw)
new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, tp_body)

local name_box = new("TextBox", {
	Size = UDim2.new(1, -74, 0, 32),
	BackgroundColor3 = COL.panel,
	Text = "",
	PlaceholderText = "nome do lugar...",
	PlaceholderColor3 = COL.dim,
	Font = Enum.Font.GothamMedium, TextSize = 12,
	TextColor3 = COL.text,
	ClearTextOnFocus = false,
	LayoutOrder = 1,
}, tp_body)
corner(name_box, 8)
new("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10) }, name_box)

local save_btn = new("TextButton", {
	Size = UDim2.fromOffset(68, 32),
	Position = UDim2.new(1, -68, 0, 0),
	BackgroundColor3 = COL.accent,
	Text = "SAVE",
	Font = Enum.Font.GothamBold, TextSize = 11,
	TextColor3 = Color3.new(1, 1, 1), AutoButtonColor = false,
}, tp_body)
corner(save_btn, 8)

local step_toggle
do
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 32),
		BackgroundColor3 = COL.panel,
		LayoutOrder = 2,
	}, tp_body)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -60, 1, 0), Position = UDim2.new(0, 10, 0, 0),
		BackgroundTransparency = 1, Text = "Step teleport (menos flag)",
		Font = Enum.Font.GothamMedium, TextSize = 12,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local b = new("TextButton", {
		Size = UDim2.fromOffset(34, 18),
		Position = UDim2.new(1, -44, 0.5, -9),
		BackgroundColor3 = COL.off, Text = "", AutoButtonColor = false,
	}, row)
	corner(b, 9)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(12, 12),
		Position = UDim2.new(0, 3, 0.5, -6),
		BackgroundColor3 = COL.text,
	}, b)
	corner(knob, 6)
	local on = false
	local function render()
		Tween:Create(b, TweenInfo.new(0.14), { BackgroundColor3 = on and COL.accent or COL.off }):Play()
		Tween:Create(knob, TweenInfo.new(0.14), {
			Position = on and UDim2.new(1, -15, 0.5, -6) or UDim2.new(0, 3, 0.5, -6),
		}):Play()
	end
	render()
	step_toggle = function(v)
		on = v and true or false
		tp.step = on
		render()
	end
	b.MouseButton1Click:Connect(function() step_toggle(not on) end)
end

local tp_list = new("ScrollingFrame", {
	Size = UDim2.new(1, 0, 1, -80),
	Position = UDim2.new(0, 0, 0, 80),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 3,
	ScrollBarImageColor3 = COL.stroke,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	AutomaticCanvasSize = Enum.AutomaticSize.Y,
	ScrollingDirection = Enum.ScrollingDirection.Y,
	LayoutOrder = 3,
}, tp_body)
new("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, tp_list)

local empty_lbl = new("TextLabel", {
	Size = UDim2.new(1, 0, 0, 40),
	BackgroundTransparency = 1,
	Text = "sem spots salvos",
	Font = Enum.Font.GothamMedium, TextSize = 11,
	TextColor3 = COL.dim,
}, tp_list)

local function tp_go(entry)
	local char = LP.Character
	local hrp = char and char:FindFirstChild("HumanoidRootPart")
	if not hrp or type(entry.cf) ~= "table" or #entry.cf < 3 then return end
	local goal = CFrame.new(unpack(entry.cf))
	if not tp.step then
		pcall(function() hrp.CFrame = goal end)
		return
	end
	local start = hrp.CFrame.Position
	local finish = goal.Position
	local total = (finish - start).Magnitude
	local steps = math.clamp(math.ceil(total / 35), 1, 80)
	local rot = goal - goal.Position
	task.spawn(function()
		for i = 1, steps do
			local c = LP.Character
			local r = c and c:FindFirstChild("HumanoidRootPart")
			if not r then return end
			local a = i / steps
			local pos = (i == steps) and finish or start:Lerp(finish, a)
			pcall(function() r.CFrame = CFrame.new(pos) * rot end)
			task.wait(0.015)
		end
	end)
end

local function tp_refresh()
	for _, c in ipairs(tp_list:GetChildren()) do
		if c ~= empty_lbl and not c:IsA("UIListLayout") then
			c:Destroy()
		end
	end
	empty_lbl.Visible = (#tp.list == 0)
	empty_lbl.LayoutOrder = (#tp.list == 0) and 1 or 0
	for i, entry in ipairs(tp.list) do
		local row = new("Frame", {
			Size = UDim2.new(1, -4, 0, 32),
			BackgroundColor3 = COL.panel,
			LayoutOrder = i,
		}, tp_list)
		corner(row, 8)
		new("TextLabel", {
			Size = UDim2.new(1, -86, 1, 0), Position = UDim2.new(0, 10, 0, 0),
			BackgroundTransparency = 1, Text = entry.name or "spot",
			Font = Enum.Font.GothamMedium, TextSize = 12,
			TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd,
		}, row)
		local go = new("TextButton", {
			Size = UDim2.fromOffset(40, 24),
			Position = UDim2.new(1, -74, 0.5, -12),
			BackgroundColor3 = COL.accent,
			Text = "GO",
			Font = Enum.Font.GothamBold, TextSize = 10,
			TextColor3 = Color3.new(1, 1, 1), AutoButtonColor = false,
		}, row)
		corner(go, 6)
		go.MouseButton1Click:Connect(function() tp_go(entry) end)
		local del = new("TextButton", {
			Size = UDim2.fromOffset(24, 24),
			Position = UDim2.new(1, -30, 0.5, -12),
			BackgroundColor3 = COL.panel2,
			Text = "×",
			Font = Enum.Font.GothamBold, TextSize = 13,
			TextColor3 = COL.danger, AutoButtonColor = false,
		}, row)
		corner(del, 6)
		del.MouseButton1Click:Connect(function()
			table.remove(tp.list, i)
			tp_save()
			tp_refresh()
		end)
	end
end

save_btn.MouseButton1Click:Connect(function()
	local char = LP.Character
	local hrp = char and char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	local name = name_box.Text
	if name == "" then name = "spot " .. (#tp.list + 1) end
	local comps = { hrp.CFrame:GetComponents() }
	tp.list[#tp.list + 1] = { name = name, cf = comps }
	name_box.Text = ""
	tp_save()
	tp_refresh()
end)

local function place_tp()
	local vp = viewport()
	local s = compute_scale()
	local x = (vp.X - 304 * s) / 2 + 300 * s
	if x + 304 * s > vp.X - 8 then x = vp.X - 304 * s - 8 end
	if x < 8 then x = 8 end
	tpw.Position = UDim2.fromOffset(x, vp.Y / 2)
end

local tp_tab = new("TextButton", {
	Size = UDim2.new(1, 0, 0, 36),
	BackgroundColor3 = COL.panel,
	Text = "", AutoButtonColor = false,
	LayoutOrder = 5,
}, side)
corner(tp_tab, 8)
new("Frame", {
	Size = UDim2.new(0, 3, 0, 16),
	Position = UDim2.new(0, 0, 0.5, -8),
	BackgroundColor3 = COL.accent2,
}, tp_tab)
new("TextLabel", {
	Size = UDim2.new(1, -22, 1, 0), Position = UDim2.new(0, 14, 0, 0),
	BackgroundTransparency = 1, Text = "TELEPORTS",
	Font = Enum.Font.GothamMedium, TextSize = 13,
	TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left,
}, tp_tab)
tp_tab.MouseButton1Click:Connect(function()
	tpw.Visible = not tpw.Visible
	if tpw.Visible then
		place_tp()
		tp_scale.Scale = compute_scale()
	end
end)

tp_refresh()

--========================================================================
-- SCALE WATCH
--========================================================================
local last_scale = 0
bind(RunService.Heartbeat, function()
	local s = compute_scale()
	if math.abs(s - last_scale) > 0.01 then
		last_scale = s
		win_scale.Scale = s
		tp_scale.Scale = s
		place_tp()
	end
end)

LP.CharacterAdded:Connect(function()
	fly.vert = 0
end)

--========================================================================
-- UNLOAD
--========================================================================
function unload_dummy()
	for _, c in ipairs(CONN) do pcall(function() c:Disconnect() end) end
	for i = #CONN, 1, -1 do CONN[i] = nil end
	for _, d in ipairs(DRAW) do pcall(function() d:Remove() end) end
	for i = #DRAW, 1, -1 do DRAW[i] = nil end
	for p, e in pairs(esp_cache) do esp_drop(e); esp_cache[p] = nil end
	pcall(function() set_kick_guard(false) end)
	pcall(function() gui:Destroy() end)
	if type(getgenv) == "function" then getgenv().ZERO_CLEANUP = nil end
end

if type(getgenv) == "function" then
	getgenv().ZERO_CLEANUP = unload_dummy
end

select_page(aim_page)
print("[ZERO] hub carregado")
