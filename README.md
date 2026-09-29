--// ============================================================
--//  NEXUS PANEL v3 - UI Library futurista + Painel Admin (COMPLETO)
--//  Arraste o painel pela barra do topo
--//  Arraste o quadrado flutuante (com imagem) e clique para abrir/fechar
--//  RightShift também abre/fecha (configurável na aba Config)
--// ============================================================

-- IMAGEM DO BOTÃO: ID da sua imagem (ex: "rbxassetid://123456789" ou só os números)
-- Deixe vazio ("") para usar a foto do seu avatar automaticamente.
local LOGO_ID = ""
local CONFIG_FILE = "NexusPanel_config.json"

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local GuiService = game:GetService("GuiService")
local ContextActionService = game:GetService("ContextActionService")
local VirtualInputManager
pcall(function()
	VirtualInputManager = game:GetService("VirtualInputManager")
end)

local player = Players.LocalPlayer

--// ============================================================
--//  CONFIGURAÇÃO SALVA (carrega ao abrir)
--// ============================================================
local Saved = { toggles = {}, sliders = {}, keys = {}, favorites = {}, waypoints = {}, misc = {} }

local function loadConfig()
	if not (isfile and readfile) then
		return
	end
	local ok, data = pcall(function()
		if isfile(CONFIG_FILE) then
			return HttpService:JSONDecode(readfile(CONFIG_FILE))
		end
	end)
	if ok and type(data) == "table" then
		for k in pairs(Saved) do
			if type(data[k]) == "table" then
				Saved[k] = data[k]
			end
		end
	end
end
loadConfig()

local Favorites = Saved.favorites
local Waypoints = Saved.waypoints

--// ============================================================
--//  PARENT DA GUI
--// ============================================================
local function getGuiParent()
	local ok, result = pcall(function()
		return (gethui and gethui()) or game:GetService("CoreGui")
	end)
	if ok and result then
		return result
	end
	return player:WaitForChild("PlayerGui")
end

local guiParent = getGuiParent()
local old = guiParent:FindFirstChild("NexusPanel")
if old then
	old:Destroy()
end

--// ============================================================
--//  TEMAS
--// ============================================================
local ThemePresets = {
	["Ciano / Roxo"] = { Color3.fromRGB(0, 240, 255), Color3.fromRGB(150, 60, 255) },
	["Vermelho"] = { Color3.fromRGB(255, 60, 80), Color3.fromRGB(150, 0, 30) },
	["Verde"] = { Color3.fromRGB(60, 255, 120), Color3.fromRGB(0, 140, 70) },
	["Rosa Neon"] = { Color3.fromRGB(255, 60, 200), Color3.fromRGB(140, 40, 255) },
}
local ThemeOrder = { "Ciano / Roxo", "Vermelho", "Verde", "Rosa Neon" }

local Theme = {
	Background = Color3.fromRGB(10, 10, 20),
	Panel = Color3.fromRGB(16, 16, 30),
	Element = Color3.fromRGB(24, 24, 44),
	Accent = ThemePresets["Ciano / Roxo"][1],
	Accent2 = ThemePresets["Ciano / Roxo"][2],
	Text = Color3.fromRGB(235, 240, 255),
	SubText = Color3.fromRGB(140, 150, 180),
}
do
	local p = ThemePresets[Saved.misc.theme or ""]
	if p then
		Theme.Accent, Theme.Accent2 = p[1], p[2]
	end
end

--// ============================================================
--//  BASE: conexões, tween, helpers de UI
--// ============================================================
local connections = {}
local function connect(signal, fn)
	local c = signal:Connect(fn)
	table.insert(connections, c)
	return c
end

local function tween(obj, time, props)
	TweenService:Create(obj, TweenInfo.new(time, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), props):Play()
end

local function corner(parent, radius)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius or 8)
	c.Parent = parent
	return c
end

local function stroke(parent, color, thickness, transparency)
	local s = Instance.new("UIStroke")
	s.Color = color or Theme.Accent
	s.Thickness = thickness or 1
	s.Transparency = transparency or 0.5
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = parent
	return s
end

local function gradient(parent, c1, c2, rotation)
	local g = Instance.new("UIGradient")
	g.Color = ColorSequence.new(c1, c2)
	g.Rotation = rotation or 0
	g.Parent = parent
	return g
end

--// ============================================================
--//  ATALHOS CONFIGURÁVEIS
--// ============================================================
local Binds = {}
local BindOrder = {}
local listeningBind = false

local function registerBind(id, label, defaultKey, fn)
	local key = defaultKey
	local saved = Saved.keys[id]
	if saved then
		local ok, k = pcall(function()
			return Enum.KeyCode[saved]
		end)
		if ok and k then
			key = k
		end
	end
	Binds[id] = { label = label, key = key, fn = fn }
	table.insert(BindOrder, id)
end

connect(UserInputService.InputBegan, function(input, processed)
	if processed or listeningBind then
		return
	end
	if input.UserInputType ~= Enum.UserInputType.Keyboard then
		return
	end
	for _, b in pairs(Binds) do
		if b.key ~= Enum.KeyCode.Unknown and b.key == input.KeyCode then
			task.spawn(b.fn)
		end
	end
end)

--// ============================================================
--//  UI LIBRARY
--// ============================================================
local Library = { Toggles = {}, Sliders = {} }

--// Sistema de arrastar (mouse e touch). onClick dispara se não houve arrasto.
local function makeDraggable(handle, target, onClick)
	local dragging = false
	local moved = false
	local dragStart, startPos

	connect(handle.InputBegan, function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			moved = false
			dragStart = input.Position
			startPos = target.Position

			local endedConn
			endedConn = input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
					endedConn:Disconnect()
					if not moved and onClick then
						onClick()
					end
				end
			end)
		end
	end)

	connect(UserInputService.InputChanged, function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
			or input.UserInputType == Enum.UserInputType.Touch) then
			local delta = input.Position - dragStart
			if delta.Magnitude > 4 then
				moved = true
			end
			if moved then
				target.Position = UDim2.new(
					startPos.X.Scale, startPos.X.Offset + delta.X,
					startPos.Y.Scale, startPos.Y.Offset + delta.Y
				)
			end
		end
	end)
end

function Library:CreateWindow(config)
	config = config or {}
	local title = config.Title or "NEXUS"

	local screenGui = Instance.new("ScreenGui")
	screenGui.Name = "NexusPanel"
	screenGui.ResetOnSpawn = false
	screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	screenGui.DisplayOrder = 999
	screenGui.Parent = guiParent

	--// Janela principal
	local main = Instance.new("Frame")
	main.Name = "Main"
	main.Size = UDim2.new(0, 580, 0, 380)
	main.Position = UDim2.new(0.5, -290, 0.5, -190)
	main.BackgroundColor3 = Theme.Background
	main.BorderSizePixel = 0
	main.Parent = screenGui
	corner(main, 12)
	local mainStroke = stroke(main, Theme.Accent, 1.5, 0.3)
	gradient(mainStroke, Theme.Accent, Theme.Accent2, 45)

	local scale = Instance.new("UIScale")
	scale.Scale = 1
	scale.Parent = main

	--// Barra do topo
	local topbar = Instance.new("Frame")
	topbar.Name = "Topbar"
	topbar.Size = UDim2.new(1, 0, 0, 42)
	topbar.BackgroundColor3 = Theme.Panel
	topbar.BorderSizePixel = 0
	topbar.Parent = main
	corner(topbar, 12)

	local topFix = Instance.new("Frame")
	topFix.Size = UDim2.new(1, 0, 0, 12)
	topFix.Position = UDim2.new(0, 0, 1, -12)
	topFix.BackgroundColor3 = Theme.Panel
	topFix.BorderSizePixel = 0
	topFix.Parent = topbar

	local topLine = Instance.new("Frame")
	topLine.Size = UDim2.new(1, 0, 0, 2)
	topLine.Position = UDim2.new(0, 0, 1, 0)
	topLine.BorderSizePixel = 0
	topLine.BackgroundColor3 = Theme.Accent
	topLine.Parent = topbar
	gradient(topLine, Theme.Accent, Theme.Accent2, 0)

	local logoImg = Instance.new("ImageLabel")
	logoImg.Size = UDim2.new(0, 28, 0, 28)
	logoImg.Position = UDim2.new(0, 10, 0.5, -14)
	logoImg.BackgroundColor3 = Theme.Element
	logoImg.Image = ""
	logoImg.ScaleType = Enum.ScaleType.Crop
	logoImg.Parent = topbar
	corner(logoImg, 8)

	local logoFallback = Instance.new("TextLabel")
	logoFallback.Size = UDim2.new(1, 0, 1, 0)
	logoFallback.BackgroundTransparency = 1
	logoFallback.Text = "N"
	logoFallback.Font = Enum.Font.GothamBlack
	logoFallback.TextSize = 16
	logoFallback.TextColor3 = Theme.Accent
	logoFallback.Parent = logoImg

	local titleLabel = Instance.new("TextLabel")
	titleLabel.Size = UDim2.new(1, -120, 1, 0)
	titleLabel.Position = UDim2.new(0, 46, 0, 0)
	titleLabel.BackgroundTransparency = 1
	titleLabel.Text = title
	titleLabel.Font = Enum.Font.GothamBold
	titleLabel.TextSize = 16
	titleLabel.TextColor3 = Theme.Text
	titleLabel.TextXAlignment = Enum.TextXAlignment.Left
	titleLabel.Parent = topbar
	gradient(titleLabel, Theme.Accent, Theme.Accent2, 0)

	local closeBtn = Instance.new("TextButton")
	closeBtn.Size = UDim2.new(0, 28, 0, 28)
	closeBtn.Position = UDim2.new(1, -38, 0.5, -14)
	closeBtn.BackgroundColor3 = Theme.Element
	closeBtn.Text = "X"
	closeBtn.Font = Enum.Font.GothamBold
	closeBtn.TextSize = 14
	closeBtn.TextColor3 = Theme.Text
	closeBtn.AutoButtonColor = false
	closeBtn.Parent = topbar
	corner(closeBtn, 8)

	--// Sidebar (com rolagem, pois tem várias abas)
	local sidebar = Instance.new("ScrollingFrame")
	sidebar.Size = UDim2.new(0, 130, 1, -54)
	sidebar.Position = UDim2.new(0, 8, 0, 48)
	sidebar.BackgroundColor3 = Theme.Panel
	sidebar.BorderSizePixel = 0
	sidebar.ScrollBarThickness = 2
	sidebar.ScrollBarImageColor3 = Theme.Accent
	sidebar.AutomaticCanvasSize = Enum.AutomaticSize.Y
	sidebar.CanvasSize = UDim2.new(0, 0, 0, 0)
	sidebar.Parent = main
	corner(sidebar, 10)

	local sideLayout = Instance.new("UIListLayout")
	sideLayout.Padding = UDim.new(0, 6)
	sideLayout.Parent = sidebar
	local sidePad = Instance.new("UIPadding")
	sidePad.PaddingTop = UDim.new(0, 8)
	sidePad.PaddingBottom = UDim.new(0, 8)
	sidePad.PaddingLeft = UDim.new(0, 8)
	sidePad.PaddingRight = UDim.new(0, 8)
	sidePad.Parent = sidebar

	--// Conteúdo
	local content = Instance.new("Frame")
	content.Size = UDim2.new(1, -154, 1, -54)
	content.Position = UDim2.new(0, 146, 0, 48)
	content.BackgroundColor3 = Theme.Panel
	content.BorderSizePixel = 0
	content.Parent = main
	corner(content, 10)

	--// Botão flutuante (quadrado arrastável COM IMAGEM)
	local floatBtn = Instance.new("ImageButton")
	floatBtn.Name = "FloatButton"
	floatBtn.Size = UDim2.new(0, 60, 0, 60)
	floatBtn.Position = UDim2.new(0, 20, 0.5, -30)
	floatBtn.BackgroundColor3 = Theme.Background
	floatBtn.Image = ""
	floatBtn.ScaleType = Enum.ScaleType.Crop
	floatBtn.AutoButtonColor = false
	floatBtn.Parent = screenGui
	corner(floatBtn, 14)
	local floatStroke = stroke(floatBtn, Theme.Accent, 2.5, 0)
	local floatGrad = gradient(floatStroke, Theme.Accent, Theme.Accent2, 45)

	local floatFallback = Instance.new("TextLabel")
	floatFallback.Size = UDim2.new(1, 0, 1, 0)
	floatFallback.BackgroundTransparency = 1
	floatFallback.Text = "N"
	floatFallback.Font = Enum.Font.GothamBlack
	floatFallback.TextSize = 28
	floatFallback.TextColor3 = Theme.Accent
	floatFallback.Parent = floatBtn

	task.spawn(function()
		local rot = 0
		while floatBtn.Parent do
			rot = (rot + 2) % 360
			floatGrad.Rotation = rot
			task.wait(0.03)
		end
	end)

	--// Escala (tamanho do painel + modo mobile)
	local baseScale, mobileMult = 1, 1
	local function currentScale()
		return baseScale * mobileMult
	end

	--// Abrir / fechar
	local opened = true
	local function setOpen(state)
		opened = state
		if state then
			main.Visible = true
			scale.Scale = currentScale() * 0.85
			tween(scale, 0.25, { Scale = currentScale() })
		else
			tween(scale, 0.2, { Scale = currentScale() * 0.85 })
			task.delay(0.2, function()
				if not opened then
					main.Visible = false
				end
			end)
		end
	end

	makeDraggable(topbar, main)
	makeDraggable(floatBtn, floatBtn, function()
		setOpen(not opened)
	end)

	connect(closeBtn.MouseButton1Click, function()
		setOpen(false)
	end)
	connect(closeBtn.MouseEnter, function()
		tween(closeBtn, 0.15, { BackgroundColor3 = Color3.fromRGB(255, 60, 90) })
	end)
	connect(closeBtn.MouseLeave, function()
		tween(closeBtn, 0.15, { BackgroundColor3 = Theme.Element })
	end)

	--// Notificações
	local toasts = Instance.new("Frame")
	toasts.Size = UDim2.new(0, 250, 1, -20)
	toasts.Position = UDim2.new(1, -260, 0, 10)
	toasts.BackgroundTransparency = 1
	toasts.Parent = screenGui
	local toastLayout = Instance.new("UIListLayout")
	toastLayout.VerticalAlignment = Enum.VerticalAlignment.Bottom
	toastLayout.Padding = UDim.new(0, 6)
	toastLayout.Parent = toasts

	--// Log de ações na tela (canto inferior esquerdo)
	local logFrame = Instance.new("Frame")
	logFrame.Size = UDim2.new(0, 300, 0, 200)
	logFrame.Position = UDim2.new(0, 10, 1, -210)
	logFrame.BackgroundTransparency = 1
	logFrame.Parent = screenGui
	local logLayout = Instance.new("UIListLayout")
	logLayout.VerticalAlignment = Enum.VerticalAlignment.Bottom
	logLayout.Padding = UDim.new(0, 3)
	logLayout.Parent = logFrame

	--// Botões de subir/descer para celular (voo e freecam)
	local mobileBar = Instance.new("Frame")
	mobileBar.Size = UDim2.new(0, 70, 0, 150)
	mobileBar.Position = UDim2.new(1, -90, 1, -260)
	mobileBar.BackgroundTransparency = 1
	mobileBar.Visible = false
	mobileBar.Parent = screenGui

	--// Objeto Window
	local Window = {
		Tabs = {},
		ScreenGui = screenGui,
		Main = main,
		FloatBtn = floatBtn,
		MobileBar = mobileBar,
		MobileInput = { up = false, down = false },
		Mobile = false,
		LogEnabled = true,
	}
	local currentTab
	local elementFrames = {}
	local currentT = 0

	local function makeMobileBtn(text, y, key)
		local b = Instance.new("TextButton")
		b.Size = UDim2.new(0, 70, 0, 70)
		b.Position = UDim2.new(0, 0, 0, y)
		b.BackgroundColor3 = Theme.Panel
		b.Text = text
		b.Font = Enum.Font.GothamBold
		b.TextSize = 28
		b.TextColor3 = Theme.Accent
		b.AutoButtonColor = false
		b.Parent = mobileBar
		corner(b, 14)
		stroke(b, Theme.Accent, 2, 0.2)
		connect(b.InputBegan, function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1
				or input.UserInputType == Enum.UserInputType.Touch then
				Window.MobileInput[key] = true
			end
		end)
		connect(b.InputEnded, function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1
				or input.UserInputType == Enum.UserInputType.Touch then
				Window.MobileInput[key] = false
			end
		end)
	end
	makeMobileBtn("▲", 0, "up")
	makeMobileBtn("▼", 80, "down")

	function Window:Toggle()
		setOpen(not opened)
	end

	function Window:Notify(text)
		local t = Instance.new("TextLabel")
		t.Size = UDim2.new(1, 0, 0, 34)
		t.BackgroundColor3 = Theme.Panel
		t.BackgroundTransparency = 1
		t.TextTransparency = 1
		t.Text = text
		t.Font = Enum.Font.GothamSemibold
		t.TextSize = 12
		t.TextColor3 = Theme.Text
		t.TextWrapped = true
		t.Parent = toasts
		corner(t, 8)
		local s = stroke(t, Theme.Accent, 1.2, 1)
		tween(t, 0.25, { BackgroundTransparency = 0, TextTransparency = 0 })
		tween(s, 0.25, { Transparency = 0.3 })
		task.delay(3, function()
			if t.Parent then
				tween(t, 0.3, { BackgroundTransparency = 1, TextTransparency = 1 })
				tween(s, 0.3, { Transparency = 1 })
				task.delay(0.35, function()
					t:Destroy()
				end)
			end
		end)
	end

	function Window:Log(text)
		if not Window.LogEnabled then
			return
		end
		local l = Instance.new("TextLabel")
		l.Size = UDim2.new(1, 0, 0, 20)
		l.BackgroundColor3 = Theme.Panel
		l.BackgroundTransparency = 0.25
		l.Text = "  " .. text
		l.Font = Enum.Font.GothamSemibold
		l.TextSize = 12
		l.TextColor3 = Theme.Text
		l.TextXAlignment = Enum.TextXAlignment.Left
		l.Parent = logFrame
		corner(l, 6)
		local lines = {}
		for _, c in ipairs(logFrame:GetChildren()) do
			if c:IsA("TextLabel") then
				table.insert(lines, c)
			end
		end
		if #lines > 8 then
			lines[1]:Destroy()
		end
		task.delay(4, function()
			if l.Parent then
				tween(l, 0.4, { BackgroundTransparency = 1, TextTransparency = 1 })
				task.delay(0.45, function()
					l:Destroy()
				end)
			end
		end)
	end

	--// Tema ao vivo: troca as cores de destaque em tudo que já existe
	function Window:SetTheme(name)
		local preset = ThemePresets[name]
		if not preset then
			return
		end
		local oldA, oldB = Theme.Accent, Theme.Accent2
		local newA, newB = preset[1], preset[2]
		Theme.Accent, Theme.Accent2 = newA, newB
		Saved.misc.theme = name

		local function close(a, b)
			return math.abs(a.R - b.R) < 0.01 and math.abs(a.G - b.G) < 0.01 and math.abs(a.B - b.B) < 0.01
		end
		local function conv(c)
			if close(c, oldA) then
				return newA
			elseif close(c, oldB) then
				return newB
			end
			return c
		end

		for _, d in ipairs(screenGui:GetDescendants()) do
			if d:IsA("GuiObject") then
				d.BackgroundColor3 = conv(d.BackgroundColor3)
				if d:IsA("TextLabel") or d:IsA("TextButton") or d:IsA("TextBox") then
					d.TextColor3 = conv(d.TextColor3)
				end
				if d:IsA("ScrollingFrame") then
					d.ScrollBarImageColor3 = conv(d.ScrollBarImageColor3)
				end
			elseif d:IsA("UIStroke") then
				d.Color = conv(d.Color)
			elseif d:IsA("UIGradient") then
				local kps = d.Color.Keypoints
				local new = {}
				for i, k in ipairs(kps) do
					new[i] = ColorSequenceKeypoint.new(k.Time, conv(k.Value))
				end
				d.Color = ColorSequence.new(new)
			elseif d:IsA("Highlight") then
				d.FillColor = conv(d.FillColor)
				d.OutlineColor = conv(d.OutlineColor)
			end
		end
	end

	function Window:SetOpacity(v)
		currentT = 1 - math.clamp(v, 0.2, 1)
		for _, f in ipairs({ main, topbar, topFix, sidebar, content }) do
			f.BackgroundTransparency = currentT
		end
		for _, f in ipairs(elementFrames) do
			if f.Parent then
				f.BackgroundTransparency = currentT
			end
		end
	end

	function Window:SetScale(v)
		baseScale = v
		if opened then
			scale.Scale = currentScale()
		end
	end

	function Window:SetMobile(on)
		Window.Mobile = on
		mobileMult = on and 1.2 or 1
		local s = on and 84 or 60
		floatBtn.Size = UDim2.new(0, s, 0, s)
		if opened then
			scale.Scale = currentScale()
		end
	end

	function Window:IsOverPanel(loc)
		local p = loc - GuiService:GetGuiInset()
		local function inside(g)
			local a, s = g.AbsolutePosition, g.AbsoluteSize
			return p.X >= a.X and p.X <= a.X + s.X and p.Y >= a.Y and p.Y <= a.Y + s.Y
		end
		return (main.Visible and inside(main)) or inside(floatBtn)
	end

	--// Trocar imagem do botão / logo
	local function applyImage(img)
		logoImg.Image = img
		floatBtn.Image = img
		local has = img ~= ""
		logoFallback.Visible = not has
		floatFallback.Visible = not has
	end

	function Window:SetLogo(value)
		value = (tostring(value or "")):match("^%s*(.-)%s*$")
		if value == "" then
			applyImage("")
			return true
		end
		if value:match("^%d+$") then
			applyImage("rbxassetid://" .. value)
			return true
		end
		if value:match("^rbxassetid://") or value:match("^rbxthumb://") or value:match("^rbxasset://") then
			applyImage(value)
			return true
		end
		if value:match("^https?://") then
			local ok, res = pcall(function()
				local req = (syn and syn.request) or (http and http.request) or http_request or request
				local resp = req({ Url = value, Method = "GET" })
				writefile("nexus_logo.png", resp.Body)
				return getcustomasset("nexus_logo.png")
			end)
			if ok and res then
				applyImage(res)
				return true
			end
			Window:Notify("Não consegui baixar a imagem (seu executor não suporta)")
			return false
		end
		Window:Notify("ID de imagem inválido")
		return false
	end

	local function selectTab(tab)
		if currentTab == tab then
			return
		end
		if currentTab then
			currentTab.Page.Visible = false
			tween(currentTab.Button, 0.2, { BackgroundColor3 = Theme.Element })
			currentTab.Button.TextColor3 = Theme.SubText
		end
		currentTab = tab
		tab.Page.Visible = true
		tween(tab.Button, 0.2, { BackgroundColor3 = Theme.Accent2 })
		tab.Button.TextColor3 = Theme.Text
	end

	function Window:CreateTab(name)
		local tab = {}

		local button = Instance.new("TextButton")
		button.Size = UDim2.new(1, 0, 0, 32)
		button.BackgroundColor3 = Theme.Element
		button.Text = name
		button.Font = Enum.Font.GothamSemibold
		button.TextSize = 13
		button.TextColor3 = Theme.SubText
		button.AutoButtonColor = false
		button.Parent = sidebar
		corner(button, 8)

		local page = Instance.new("ScrollingFrame")
		page.Size = UDim2.new(1, -12, 1, -12)
		page.Position = UDim2.new(0, 6, 0, 6)
		page.BackgroundTransparency = 1
		page.BorderSizePixel = 0
		page.ScrollBarThickness = 3
		page.ScrollBarImageColor3 = Theme.Accent
		page.CanvasSize = UDim2.new(0, 0, 0, 0)
		page.AutomaticCanvasSize = Enum.AutomaticSize.Y
		page.Visible = false
		page.Parent = content

		local layout = Instance.new("UIListLayout")
		layout.Padding = UDim.new(0, 6)
		layout.SortOrder = Enum.SortOrder.LayoutOrder
		layout.Parent = page

		local pad = Instance.new("UIPadding")
		pad.PaddingRight = UDim.new(0, 6)
		pad.Parent = page

		tab.Button = button
		tab.Page = page

		connect(button.MouseButton1Click, function()
			selectTab(tab)
		end)

		local function baseElement(height)
			local f = Instance.new("Frame")
			f.Size = UDim2.new(1, 0, 0, height or 38)
			f.BackgroundColor3 = Theme.Element
			f.BackgroundTransparency = currentT
			f.BorderSizePixel = 0
			f.Parent = page
			corner(f, 8)
			stroke(f, Theme.Accent, 1, 0.85)
			table.insert(elementFrames, f)
			return f
		end

		local function nameLabel(parent, text)
			local l = Instance.new("TextLabel")
			l.Size = UDim2.new(1, -70, 1, 0)
			l.Position = UDim2.new(0, 12, 0, 0)
			l.BackgroundTransparency = 1
			l.Text = text
			l.Font = Enum.Font.Gotham
			l.TextSize = 13
			l.TextColor3 = Theme.Text
			l.TextXAlignment = Enum.TextXAlignment.Left
			l.TextTruncate = Enum.TextTruncate.AtEnd
			l.Parent = parent
			return l
		end

		function tab:AddSection(text)
			local l = Instance.new("TextLabel")
			l.Size = UDim2.new(1, 0, 0, 22)
			l.BackgroundTransparency = 1
			l.Text = string.upper(text)
			l.Font = Enum.Font.GothamBold
			l.TextSize = 11
			l.TextColor3 = Theme.Accent
			l.TextXAlignment = Enum.TextXAlignment.Left
			l.Parent = page
			return l
		end

		function tab:AddLabel(text)
			local f = baseElement(30)
			local l = nameLabel(f, text)
			l.Size = UDim2.new(1, -24, 1, 0)
			l.TextColor3 = Theme.SubText
			l.TextTruncate = Enum.TextTruncate.AtEnd
			return l
		end

		function tab:AddButton(text, callback)
			local f = baseElement(38)
			local b = Instance.new("TextButton")
			b.Size = UDim2.new(1, 0, 1, 0)
			b.BackgroundTransparency = 1
			b.Text = text
			b.Font = Enum.Font.GothamSemibold
			b.TextSize = 13
			b.TextColor3 = Theme.Text
			b.AutoButtonColor = false
			b.Parent = f

			connect(b.MouseEnter, function()
				tween(f, 0.15, { BackgroundColor3 = Theme.Accent2 })
			end)
			connect(b.MouseLeave, function()
				tween(f, 0.15, { BackgroundColor3 = Theme.Element })
			end)
			connect(b.MouseButton1Click, function()
				tween(f, 0.08, { BackgroundColor3 = Theme.Accent })
				task.delay(0.1, function()
					tween(f, 0.15, { BackgroundColor3 = Theme.Element })
				end)
				if callback then
					task.spawn(callback)
				end
			end)
		end

		-- AddToggle(texto, padrão, callback, id)  -> id (opcional) salva o estado na config
		function tab:AddToggle(text, default, callback, id)
			local state = default or false
			local fromSaved = id ~= nil and Saved.toggles[id] ~= nil
			if fromSaved then
				state = Saved.toggles[id] == true
			end
			local api = {}
			local f = baseElement(38)
			nameLabel(f, text)

			local track = Instance.new("Frame")
			track.Size = UDim2.new(0, 42, 0, 22)
			track.Position = UDim2.new(1, -54, 0.5, -11)
			track.BackgroundColor3 = Color3.fromRGB(40, 40, 64)
			track.BorderSizePixel = 0
			track.Parent = f
			corner(track, 11)

			local knob = Instance.new("Frame")
			knob.Size = UDim2.new(0, 16, 0, 16)
			knob.Position = UDim2.new(0, 3, 0.5, -8)
			knob.BackgroundColor3 = Theme.Text
			knob.BorderSizePixel = 0
			knob.Parent = track
			corner(knob, 8)

			local function render()
				if state then
					tween(track, 0.2, { BackgroundColor3 = Theme.Accent })
					tween(knob, 0.2, { Position = UDim2.new(1, -19, 0.5, -8) })
				else
					tween(track, 0.2, { BackgroundColor3 = Color3.fromRGB(40, 40, 64) })
					tween(knob, 0.2, { Position = UDim2.new(0, 3, 0.5, -8) })
				end
			end
			render()

			local click = Instance.new("TextButton")
			click.Size = UDim2.new(1, 0, 1, 0)
			click.BackgroundTransparency = 1
			click.Text = ""
			click.Parent = f

			connect(click.MouseButton1Click, function()
				state = not state
				render()
				Window:Log(text .. (state and "  ●  ON" or "  ○  OFF"))
				if callback then
					task.spawn(callback, state)
				end
			end)

			function api.Set(v, silent)
				state = v
				render()
				if not silent then
					Window:Log(text .. (state and "  ●  ON" or "  ○  OFF"))
				end
				if callback and not silent then
					task.spawn(callback, state)
				end
			end
			function api.Get()
				return state
			end

			if id then
				Library.Toggles[id] = api
			end
			if callback and (state or fromSaved) then
				task.spawn(callback, state)
			end
			return api
		end

		-- AddSlider(texto, min, max, padrão, callback, step, id)
		function tab:AddSlider(text, min, max, default, callback, step, id)
			step = step or 1
			local value = default or min
			local fromSaved = id ~= nil and type(Saved.sliders[id]) == "number"
			if fromSaved then
				value = math.clamp(Saved.sliders[id], min, max)
			end
			local api = {}
			local f = baseElement(52)

			local function fmt(v)
				if step < 0.1 then
					return string.format("%.2f", v)
				elseif step < 1 then
					return string.format("%.1f", v)
				end
				return tostring(v)
			end

			local l = nameLabel(f, text)
			l.Size = UDim2.new(1, -80, 0, 26)

			local valueLabel = Instance.new("TextLabel")
			valueLabel.Size = UDim2.new(0, 60, 0, 26)
			valueLabel.Position = UDim2.new(1, -70, 0, 0)
			valueLabel.BackgroundTransparency = 1
			valueLabel.Text = fmt(value)
			valueLabel.Font = Enum.Font.GothamBold
			valueLabel.TextSize = 13
			valueLabel.TextColor3 = Theme.Accent
			valueLabel.TextXAlignment = Enum.TextXAlignment.Right
			valueLabel.Parent = f

			local track = Instance.new("Frame")
			track.Size = UDim2.new(1, -24, 0, 6)
			track.Position = UDim2.new(0, 12, 0, 34)
			track.BackgroundColor3 = Color3.fromRGB(40, 40, 64)
			track.BorderSizePixel = 0
			track.Parent = f
			corner(track, 3)

			local fill = Instance.new("Frame")
			fill.Size = UDim2.new((value - min) / (max - min), 0, 1, 0)
			fill.BackgroundColor3 = Theme.Accent
			fill.BorderSizePixel = 0
			fill.Parent = track
			corner(fill, 3)
			gradient(fill, Theme.Accent, Theme.Accent2, 0)

			local knob = Instance.new("Frame")
			knob.Size = UDim2.new(0, 14, 0, 14)
			knob.AnchorPoint = Vector2.new(0.5, 0.5)
			knob.Position = UDim2.new(1, 0, 0.5, 0)
			knob.BackgroundColor3 = Theme.Text
			knob.BorderSizePixel = 0
			knob.Parent = fill
			corner(knob, 7)

			local function apply(v, fire)
				value = math.clamp(math.floor(v * 100 + 0.5) / 100, min, max)
				fill.Size = UDim2.new((value - min) / (max - min), 0, 1, 0)
				valueLabel.Text = fmt(value)
				if fire and callback then
					callback(value)
				end
			end

			local dragging = false
			local function setFromX(x)
				local rel = math.clamp((x - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
				local raw = min + (max - min) * rel
				apply(min + math.floor((raw - min) / step + 0.5) * step, true)
			end

			connect(track.InputBegan, function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1
					or input.UserInputType == Enum.UserInputType.Touch then
					dragging = true
					setFromX(input.Position.X)
				end
			end)
			connect(UserInputService.InputChanged, function(input)
				if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
					or input.UserInputType == Enum.UserInputType.Touch) then
					setFromX(input.Position.X)
				end
			end)
			connect(UserInputService.InputEnded, function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1
					or input.UserInputType == Enum.UserInputType.Touch then
					dragging = false
				end
			end)

			function api.Set(v)
				apply(v, true)
			end
			function api.Get()
				return value
			end
			if id then
				Library.Sliders[id] = api
			end
			if fromSaved and callback then
				task.spawn(callback, value)
			end
			return api
		end

		function tab:AddTextbox(text, placeholder, callback)
			local f = baseElement(38)
			local l = nameLabel(f, text)
			l.Size = UDim2.new(0.4, -12, 1, 0)

			local box = Instance.new("TextBox")
			box.Size = UDim2.new(0.6, -12, 0, 26)
			box.Position = UDim2.new(0.4, 0, 0.5, -13)
			box.BackgroundColor3 = Color3.fromRGB(14, 14, 26)
			box.Text = ""
			box.PlaceholderText = placeholder or ""
			box.PlaceholderColor3 = Theme.SubText
			box.ClearTextOnFocus = false
			box.Font = Enum.Font.Gotham
			box.TextSize = 12
			box.TextColor3 = Theme.Text
			box.Parent = f
			corner(box, 6)
			stroke(box, Theme.Accent, 1, 0.7)

			connect(box.FocusLost, function(enter)
				if callback then
					task.spawn(callback, box.Text, enter)
				end
			end)
			return box
		end

		-- Atalho configurável (linha na aba Config)
		function tab:AddKeybind(text, id)
			local b = Binds[id]
			local f = baseElement(38)
			local l = nameLabel(f, text)
			l.Size = UDim2.new(1, -120, 1, 0)

			local btn = Instance.new("TextButton")
			btn.Size = UDim2.new(0, 84, 0, 26)
			btn.Position = UDim2.new(1, -96, 0.5, -13)
			btn.BackgroundColor3 = Color3.fromRGB(14, 14, 26)
			btn.Font = Enum.Font.GothamBold
			btn.TextSize = 12
			btn.TextColor3 = Theme.Accent
			btn.AutoButtonColor = false
			btn.Parent = f
			corner(btn, 6)
			stroke(btn, Theme.Accent, 1, 0.7)

			local function label()
				btn.Text = (b.key == Enum.KeyCode.Unknown) and "—" or b.key.Name
			end
			label()

			connect(btn.MouseButton1Click, function()
				if listeningBind then
					return
				end
				listeningBind = true
				btn.Text = "..."
				local c
				c = UserInputService.InputBegan:Connect(function(input)
					if input.UserInputType == Enum.UserInputType.Keyboard then
						if input.KeyCode == Enum.KeyCode.Escape then
							-- cancela
						elseif input.KeyCode == Enum.KeyCode.Backspace then
							b.key = Enum.KeyCode.Unknown
						else
							b.key = input.KeyCode
						end
						c:Disconnect()
						label()
						task.delay(0.15, function()
							listeningBind = false
						end)
					end
				end)
			end)
		end

		-- Lista genérica com linhas clicáveis
		function tab:AddList(height)
			local holder = baseElement(height or 120)
			local api = {}
			local rows = {}
			local count = 0
			local selectedRow

			local sf = Instance.new("ScrollingFrame")
			sf.Size = UDim2.new(1, -12, 1, -12)
			sf.Position = UDim2.new(0, 6, 0, 6)
			sf.BackgroundTransparency = 1
			sf.BorderSizePixel = 0
			sf.ScrollBarThickness = 3
			sf.ScrollBarImageColor3 = Theme.Accent
			sf.AutomaticCanvasSize = Enum.AutomaticSize.Y
			sf.CanvasSize = UDim2.new(0, 0, 0, 0)
			sf.Parent = holder

			local ll = Instance.new("UIListLayout")
			ll.Padding = UDim.new(0, 3)
			ll.SortOrder = Enum.SortOrder.LayoutOrder
			ll.Parent = sf

			function api.Clear()
				for _, r in ipairs(rows) do
					r:Destroy()
				end
				rows = {}
				selectedRow = nil
				count = 0
			end

			-- Add(texto, onClick, cor, noTopo)
			function api.Add(text, onClick, color, top)
				count += 1
				local r = Instance.new("TextButton")
				r.Size = UDim2.new(1, -6, 0, 24)
				r.BackgroundColor3 = Theme.Panel
				r.Text = text
				r.Font = Enum.Font.Gotham
				r.TextSize = 12
				r.TextColor3 = color or Theme.Text
				r.TextXAlignment = Enum.TextXAlignment.Left
				r.TextTruncate = Enum.TextTruncate.AtEnd
				r.AutoButtonColor = false
				r.LayoutOrder = top and -count or count
				r.Parent = sf
				corner(r, 6)
				local rp = Instance.new("UIPadding")
				rp.PaddingLeft = UDim.new(0, 8)
				rp.Parent = r
				table.insert(rows, r)

				connect(r.MouseButton1Click, function()
					if onClick then
						if selectedRow and selectedRow.Parent then
							selectedRow.BackgroundColor3 = Theme.Panel
						end
						selectedRow = r
						r.BackgroundColor3 = Theme.Accent2
						task.spawn(onClick, r)
					end
				end)

				if top and #rows > 40 then
					local oldest = table.remove(rows, 1)
					oldest:Destroy()
				end
				return r
			end
			return api
		end

		--// Lista de jogadores (atualiza sozinha, favoritos no topo)
		function tab:AddPlayerList(onSelect)
			local holder = baseElement(210)
			local api = {}
			local rows = {}
			local rowLabels = {}
			local selected

			local search = Instance.new("TextBox")
			search.Size = UDim2.new(1, -16, 0, 26)
			search.Position = UDim2.new(0, 8, 0, 6)
			search.BackgroundColor3 = Color3.fromRGB(14, 14, 26)
			search.Text = ""
			search.PlaceholderText = "Buscar jogador..."
			search.PlaceholderColor3 = Theme.SubText
			search.ClearTextOnFocus = false
			search.Font = Enum.Font.Gotham
			search.TextSize = 12
			search.TextColor3 = Theme.Text
			search.Parent = holder
			corner(search, 6)
			stroke(search, Theme.Accent, 1, 0.7)

			local list = Instance.new("ScrollingFrame")
			list.Size = UDim2.new(1, -16, 1, -46)
			list.Position = UDim2.new(0, 8, 0, 38)
			list.BackgroundTransparency = 1
			list.BorderSizePixel = 0
			list.ScrollBarThickness = 3
			list.ScrollBarImageColor3 = Theme.Accent
			list.AutomaticCanvasSize = Enum.AutomaticSize.Y
			list.CanvasSize = UDim2.new(0, 0, 0, 0)
			list.Parent = holder

			local listLayout = Instance.new("UIListLayout")
			listLayout.Padding = UDim.new(0, 4)
			listLayout.SortOrder = Enum.SortOrder.Name
			listLayout.Parent = list

			local function applyFilter()
				local q = string.lower(search.Text)
				for p, row in pairs(rows) do
					row.Visible = q == ""
						or string.find(string.lower(p.Name), q, 1, true) ~= nil
						or string.find(string.lower(p.DisplayName), q, 1, true) ~= nil
				end
			end
			connect(search:GetPropertyChangedSignal("Text"), applyFilter)

			local function selectPlayer(p)
				selected = p
				for pl, row in pairs(rows) do
					tween(row, 0.15, { BackgroundColor3 = (pl == p) and Theme.Accent2 or Theme.Panel })
				end
				if onSelect then
					onSelect(p)
				end
			end

			local function styleRow(p)
				local row, dn = rows[p], rowLabels[p]
				if not row then
					return
				end
				local fav = Favorites[p.Name] == true
				row.Name = (fav and "0_" or "1_") .. p.Name
				dn.Text = (fav and "★ " or "") .. p.DisplayName
			end

			local function createRow(p)
				local row = Instance.new("TextButton")
				row.Size = UDim2.new(1, -6, 0, 38)
				row.BackgroundColor3 = Theme.Panel
				row.Text = ""
				row.AutoButtonColor = false
				row.Parent = list
				corner(row, 8)

				local av = Instance.new("ImageLabel")
				av.Size = UDim2.new(0, 28, 0, 28)
				av.Position = UDim2.new(0, 6, 0.5, -14)
				av.BackgroundColor3 = Theme.Element
				av.Image = "rbxthumb://type=AvatarHeadShot&id=" .. p.UserId .. "&w=48&h=48"
				av.Parent = row
				corner(av, 14)

				local dn = Instance.new("TextLabel")
				dn.Size = UDim2.new(1, -48, 0, 18)
				dn.Position = UDim2.new(0, 42, 0, 3)
				dn.BackgroundTransparency = 1
				dn.Text = p.DisplayName
				dn.Font = Enum.Font.GothamBold
				dn.TextSize = 12
				dn.TextColor3 = Theme.Text
				dn.TextXAlignment = Enum.TextXAlignment.Left
				dn.TextTruncate = Enum.TextTruncate.AtEnd
				dn.Parent = row

				local un = Instance.new("TextLabel")
				un.Size = UDim2.new(1, -48, 0, 14)
				un.Position = UDim2.new(0, 42, 0, 20)
				un.BackgroundTransparency = 1
				un.Text = "@" .. p.Name
				un.Font = Enum.Font.Gotham
				un.TextSize = 11
				un.TextColor3 = Theme.SubText
				un.TextXAlignment = Enum.TextXAlignment.Left
				un.TextTruncate = Enum.TextTruncate.AtEnd
				un.Parent = row

				connect(row.MouseButton1Click, function()
					selectPlayer(p)
				end)
				connect(row.MouseEnter, function()
					if selected ~= p then
						tween(row, 0.15, { BackgroundColor3 = Theme.Element })
					end
				end)
				connect(row.MouseLeave, function()
					if selected ~= p then
						tween(row, 0.15, { BackgroundColor3 = Theme.Panel })
					end
				end)

				rows[p] = row
				rowLabels[p] = dn
				styleRow(p)
			end

			local function removeRow(p)
				local row = rows[p]
				if row then
					row:Destroy()
					rows[p] = nil
					rowLabels[p] = nil
				end
				if selected == p then
					selectPlayer(nil)
				end
			end

			local function refresh(ignore)
				local present = {}
				local count = 0
				for _, p in ipairs(Players:GetPlayers()) do
					if p ~= player and p ~= ignore then
						present[p] = true
						count += 1
						if not rows[p] then
							createRow(p)
						else
							styleRow(p)
						end
					end
				end
				for p in pairs(rows) do
					if not present[p] then
						removeRow(p)
					end
				end
				search.PlaceholderText = "Buscar jogador... (" .. count .. " online)"
				applyFilter()
			end

			connect(Players.PlayerAdded, function()
				task.wait(0.3)
				refresh()
			end)
			connect(Players.PlayerRemoving, function(p)
				refresh(p)
			end)

			task.spawn(function()
				while screenGui.Parent do
					refresh()
					task.wait(3)
				end
			end)

			refresh()

			api.Refresh = refresh
			function api.GetSelected()
				return selected
			end
			return api
		end

		table.insert(Window.Tabs, tab)
		if #Window.Tabs == 1 then
			selectTab(tab)
		end

		return tab
	end

	function Window:Destroy()
		for _, c in ipairs(connections) do
			pcall(function()
				c:Disconnect()
			end)
		end
		screenGui:Destroy()
	end

	-- aplica a imagem inicial
	local initial = config.Logo
	if not initial or initial == "" then
		initial = "rbxthumb://type=AvatarHeadShot&id=" .. player.UserId .. "&w=150&h=150"
	end
	Window:SetLogo(initial)

	return Window
end

--// ============================================================
--//  FUNÇÕES AUXILIARES DO PAINEL
--// ============================================================
local function getCharacter()
	return player.Character
end

local function getHumanoid()
	local char = getCharacter()
	return char and char:FindFirstChildOfClass("Humanoid")
end

local function getRoot()
	local char = getCharacter()
	return char and char:FindFirstChild("HumanoidRootPart")
end

local state = {
	-- movimento
	speedOn = true,
	speed = 70,
	jumpOn = false,
	jumpPower = 100,
	infJump = true,
	noclip = false,
	fly = false,
	flySpeed = 60,
	flyCar = false,
	flyCarSpeed = 80,
	autoJump = false,
	wallJump = false,
	walkWater = false,
	fastSwim = false,
	swimSpeed = 120,
	antiFall = false,
	spin = false,
	spinSpeed = 360,
	-- jogadores
	joinAlert = true,
	proxAlert = false,
	proxRadius = 60,
	-- visual
	esp = false,
	espDist = true,
	espHealth = true,
	freecam = false,
	freecamSpeed = 60,
	noFx = false,
	clockLock = false,
	clockTime = 14,
	firstP = false,
	thirdP = false,
	zoomMax = 128,
	stretch = 0.7,
	-- utilidades
	clickTp = false,
	clickDelete = false,
	autoPrompt = false,
	instantPrompt = false,
	autoClick = false,
	cps = 10,
	pathLoop = false,
	-- outros
	antiAfk = true,
	size = 1,
	animPack = nil,
}

local Window = Library:CreateWindow({
	Title = "NEXUS PANEL",
	Logo = LOGO_ID,
})

-- atalho para abrir/fechar o painel
registerBind("menu", "Abrir / fechar painel", Enum.KeyCode.RightShift, function()
	Window:Toggle()
end)

--// Modo de câmera (1ª / 3ª pessoa forçada)
local function applyCamMode()
	if state.firstP then
		player.CameraMinZoomDistance = 0.5
		player.CameraMaxZoomDistance = 0.5
	elseif state.thirdP then
		player.CameraMinZoomDistance = 12
		player.CameraMaxZoomDistance = math.max(state.zoomMax, 20)
	else
		player.CameraMinZoomDistance = 0.5
		player.CameraMaxZoomDistance = state.zoomMax
	end
end

--// ============================================================
--//  ABA: JOGADORES
--// ============================================================
local playersTab = Window:CreateTab("Jogadores")

local selectedPlayer
local infoLabel
local tpLoopToggle, spectateToggle, followToggle
local tpLoopConn, followConn
local spectating

local function tpToPlayer(p)
	local root = getRoot()
	local ch = p and p.Character
	local tr = ch and ch:FindFirstChild("HumanoidRootPart")
	if root and tr then
		root.CFrame = tr.CFrame * CFrame.new(0, 0, 3)
		return true
	end
	return false
end

local function getNearestPlayer()
	local root = getRoot()
	if not root then
		return nil
	end
	local best, bestDist
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= player then
			local tr = p.Character and p.Character:FindFirstChild("HumanoidRootPart")
			if tr then
				local d = (tr.Position - root.Position).Magnitude
				if not bestDist or d < bestDist then
					best, bestDist = p, d
				end
			end
		end
	end
	return best, bestDist
end

local function stopFollow()
	if followConn then
		followConn:Disconnect()
		followConn = nil
	end
end

playersTab:AddSection("Lista de jogadores (atualiza sozinha)")
local playerList = playersTab:AddPlayerList(function(p)
	selectedPlayer = p
end)

infoLabel = playersTab:AddLabel("Nenhum jogador selecionado")
infoLabel.TextColor3 = Theme.Accent

playersTab:AddSection("Ações no jogador selecionado")

playersTab:AddButton("Teleportar até o jogador", function()
	if not selectedPlayer then
		Window:Notify("Selecione um jogador na lista primeiro")
		return
	end
	if tpToPlayer(selectedPlayer) then
		Window:Notify("Teleportado para " .. selectedPlayer.Name)
	else
		Window:Notify("Não foi possível teleportar (personagem indisponível)")
	end
end)

playersTab:AddButton("Teleportar para o jogador MAIS PRÓXIMO", function()
	local p, d = getNearestPlayer()
	if p and tpToPlayer(p) then
		Window:Notify("Teleportado para " .. p.Name .. " (" .. math.floor(d) .. " studs)")
	else
		Window:Notify("Nenhum jogador disponível")
	end
end)
registerBind("nearestTp", "TP no jogador mais próximo", Enum.KeyCode.T, function()
	local p = getNearestPlayer()
	if p and tpToPlayer(p) then
		Window:Log("TP → " .. p.Name)
	end
end)

tpLoopToggle = playersTab:AddToggle("TP Loop (ficar grudado no jogador)", false, function(v)
	if tpLoopConn then
		tpLoopConn:Disconnect()
		tpLoopConn = nil
	end
	if v then
		if not selectedPlayer then
			Window:Notify("Selecione um jogador na lista primeiro")
			tpLoopToggle.Set(false, true)
			return
		end
		tpLoopConn = RunService.Heartbeat:Connect(function()
			if selectedPlayer then
				tpToPlayer(selectedPlayer)
			end
		end)
		Window:Notify("TP Loop ligado em " .. selectedPlayer.Name)
	end
end)

followToggle = playersTab:AddToggle("Seguir jogador (andar atrás dele)", false, function(v)
	stopFollow()
	if not v then
		return
	end
	local target = selectedPlayer
	if not target then
		Window:Notify("Selecione um jogador na lista primeiro")
		followToggle.Set(false, true)
		return
	end
	Window:Notify("Seguindo " .. target.Name)
	local acc = 0
	followConn = RunService.Heartbeat:Connect(function(dt)
		acc += dt
		if acc < 0.15 then
			return
		end
		acc = 0
		local hum, root = getHumanoid(), getRoot()
		local tr = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
		if not (hum and root and tr) then
			return
		end
		if (tr.Position - root.Position).Magnitude > 5 then
			hum:MoveTo(tr.Position)
			if tr.Position.Y - root.Position.Y > 3 and hum.FloorMaterial ~= Enum.Material.Air then
				hum.Jump = true
			end
		else
			hum:MoveTo(root.Position)
		end
	end)
end)

spectateToggle = playersTab:AddToggle("Espectar jogador", false, function(v)
	local cam = workspace.CurrentCamera
	if v then
		if not selectedPlayer then
			Window:Notify("Selecione um jogador na lista primeiro")
			spectateToggle.Set(false, true)
			return
		end
		spectating = selectedPlayer
		Window:Notify("Espectando " .. selectedPlayer.Name)
	else
		spectating = nil
		local hum = getHumanoid()
		if cam and hum then
			cam.CameraSubject = hum
		end
	end
end)

playersTab:AddButton("Copiar nome do jogador", function()
	if not selectedPlayer then
		Window:Notify("Selecione um jogador na lista primeiro")
		return
	end
	local ok = pcall(function()
		setclipboard(selectedPlayer.Name)
	end)
	Window:Notify(ok and "Nome copiado" or "Seu executor não suporta copiar")
end)

playersTab:AddButton("Atualizar lista agora", function()
	playerList.Refresh()
	Window:Notify("Lista atualizada")
end)

-- FAVORITOS
playersTab:AddSection("Favoritos")
playersTab:AddButton("★ Favoritar / desfavoritar selecionado", function()
	if not selectedPlayer then
		Window:Notify("Selecione um jogador na lista primeiro")
		return
	end
	local n = selectedPlayer.Name
	if Favorites[n] then
		Favorites[n] = nil
		Window:Notify("Removido dos favoritos: " .. n)
	else
		Favorites[n] = true
		Window:Notify("Favoritado: " .. n)
	end
	playerList.Refresh()
end)
playersTab:AddLabel("Favoritos aparecem no topo da lista (★)")

-- PROXIMIDADE
playersTab:AddSection("Perto de você / olhando para você")
playersTab:AddToggle("Alerta de proximidade", false, function(v)
	state.proxAlert = v
end, "proxAlert")
playersTab:AddSlider("Raio de proximidade (studs)", 10, 300, 60, function(v)
	state.proxRadius = v
end, 5, "proxRadius")
local nearbyList = playersTab:AddList(96)
playersTab:AddLabel("Roblox não revela quem te espectando; isto mostra quem está perto")

task.spawn(function()
	local alerted = {}
	local lastSig = ""
	while Window.ScreenGui.Parent do
		task.wait(0.5)
		local root = getRoot()
		local entries = {}
		if root then
			for _, p in ipairs(Players:GetPlayers()) do
				if p ~= player then
					local tr = p.Character and p.Character:FindFirstChild("HumanoidRootPart")
					local inside = false
					if tr then
						local off = root.Position - tr.Position
						local d = off.Magnitude
						if d <= state.proxRadius then
							inside = true
							local facing = d > 0.1 and tr.CFrame.LookVector:Dot(off.Unit) > 0.85
							table.insert(entries, { p = p, d = d, facing = facing })
							if not alerted[p] then
								alerted[p] = true
								if state.proxAlert then
									Window:Notify("⚠ " .. p.DisplayName .. " chegou perto de você (" .. math.floor(d) .. ")")
									Window:Log("Proximidade: " .. p.Name)
								end
							end
						end
					end
					if not inside then
						alerted[p] = nil
					end
				end
			end
		end
		table.sort(entries, function(a, b)
			return a.d < b.d
		end)
		local parts = {}
		for _, e in ipairs(entries) do
			table.insert(parts, e.p.Name .. math.floor(e.d) .. (e.facing and "!" or ""))
		end
		local sig = table.concat(parts, "|")
		if sig ~= lastSig then
			lastSig = sig
			nearbyList.Clear()
			if #entries == 0 then
				nearbyList.Add("Ninguém por perto", nil, Theme.SubText)
			else
				for _, e in ipairs(entries) do
					nearbyList.Add(
						string.format("%s  •  %d studs%s", e.p.DisplayName, math.floor(e.d), e.facing and "  •  olhando p/ você" or ""),
						nil,
						e.facing and Color3.fromRGB(255, 120, 120) or Theme.Text
					)
				end
			end
		end
	end
end)

-- HISTÓRICO
playersTab:AddSection("Histórico de entradas e saídas")
playersTab:AddToggle("Avisar entrada / saída na tela", true, function(v)
	state.joinAlert = v
end, "joinAlert")
local historyList = playersTab:AddList(120)

local function pushHistory(kind, p)
	local t = os.date("%H:%M:%S")
	local joined = kind == "join"
	historyList.Add(
		string.format("[%s] %s %s (@%s)", t, joined and "entrou:" or "saiu:", p.DisplayName, p.Name),
		nil,
		joined and Color3.fromRGB(110, 255, 150) or Color3.fromRGB(255, 120, 120),
		true
	)
	if state.joinAlert then
		local star = Favorites[p.Name] and "★ " or ""
		Window:Notify(star .. p.DisplayName .. (joined and " entrou no servidor" or " saiu do servidor"))
	end
end
connect(Players.PlayerAdded, function(p)
	pushHistory("join", p)
end)
connect(Players.PlayerRemoving, function(p)
	pushHistory("leave", p)
end)

-- mantém a câmera no jogador espectado (mesmo se ele renascer)
connect(RunService.Heartbeat, function()
	if spectating then
		local cam = workspace.CurrentCamera
		local ch = spectating.Character
		local hum = ch and ch:FindFirstChildOfClass("Humanoid")
		if cam and hum and cam.CameraSubject ~= hum then
			cam.CameraSubject = hum
		end
	end
end)

-- se o jogador sair, desliga tudo relacionado a ele
connect(Players.PlayerRemoving, function(p)
	if p == spectating then
		spectateToggle.Set(false)
	end
	if p == selectedPlayer then
		tpLoopToggle.Set(false, true)
		followToggle.Set(false, true)
		stopFollow()
		if tpLoopConn then
			tpLoopConn:Disconnect()
			tpLoopConn = nil
		end
	end
end)

-- informações do jogador selecionado
task.spawn(function()
	while Window.ScreenGui.Parent do
		local txt = "Nenhum jogador selecionado"
		if selectedPlayer then
			local ch = selectedPlayer.Character
			local hum = ch and ch:FindFirstChildOfClass("Humanoid")
			local tr = ch and ch:FindFirstChild("HumanoidRootPart")
			local mr = getRoot()
			local dist = (tr and mr) and math.floor((tr.Position - mr.Position).Magnitude) or "?"
			local hp = hum and math.floor(hum.Health) or "?"
			txt = selectedPlayer.Name .. "  |  Vida: " .. tostring(hp) .. "  |  Dist: " .. tostring(dist)
		end
		infoLabel.Text = txt
		task.wait(0.4)
	end
end)

--// ============================================================
--//  ABA: EU (movimento, tamanho, teleporte próprio)
--// ============================================================
local meTab = Window:CreateTab("Eu")

meTab:AddSection("Movimento")
local speedToggle = meTab:AddToggle("Velocidade (caminhar rápido)", true, function(v)
	state.speedOn = v
	if not v then
		local hum = getHumanoid()
		if hum then
			hum.WalkSpeed = 16
		end
	end
end, "speedOn")
meTab:AddSlider("Valor da velocidade", 16, 200, 70, function(v)
	state.speed = v
end, 1, "speed")
registerBind("speed", "Alternar velocidade normal / 70", Enum.KeyCode.Z, function()
	speedToggle.Set(not speedToggle.Get())
end)

meTab:AddToggle("Pulo alto", false, function(v)
	state.jumpOn = v
	if not v then
		local hum = getHumanoid()
		if hum then
			hum.JumpPower = 50
		end
	end
end, "jumpOn")
meTab:AddSlider("Força do pulo", 50, 300, 100, function(v)
	state.jumpPower = v
end, 1, "jumpPower")
meTab:AddToggle("Pulo infinito", true, function(v)
	state.infJump = v
end, "infJump")
meTab:AddToggle("Auto-pulo (pula sozinho ao andar)", false, function(v)
	state.autoJump = v
end, "autoJump")
meTab:AddToggle("Pulo na parede", false, function(v)
	state.wallJump = v
end, "wallJump")
local noclipToggle = meTab:AddToggle("Noclip", false, function(v)
	state.noclip = v
	if not v then
		local char = getCharacter()
		if char then
			for _, part in ipairs(char:GetChildren()) do
				if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
					part.CanCollide = true
				end
			end
		end
	end
end)
registerBind("noclip", "Noclip (liga/desliga)", Enum.KeyCode.N, function()
	noclipToggle.Set(not noclipToggle.Get())
end)
meTab:AddSlider("Gravidade", 0, 300, math.floor(workspace.Gravity), function(v)
	workspace.Gravity = v
end)

meTab:AddSection("Água")
meTab:AddToggle("Andar na água", false, function(v)
	state.walkWater = v
end, "walkWater")
meTab:AddToggle("Nadar rápido", false, function(v)
	state.fastSwim = v
end, "fastSwim")
meTab:AddSlider("Velocidade de nado", 16, 250, 120, function(v)
	state.swimSpeed = v
end, 1, "swimSpeed")

meTab:AddSection("Proteção e giro")
meTab:AddToggle("Anti-queda (volta à última posição segura)", false, function(v)
	state.antiFall = v
end, "antiFall")
local spinToggle = meTab:AddToggle("Spinbot (girar personagem)", false, function(v)
	state.spin = v
end)
meTab:AddSlider("Velocidade do giro (graus/s)", 60, 2000, 360, function(v)
	state.spinSpeed = v
end, 10, "spinSpeed")
registerBind("spin", "Spinbot (liga/desliga)", Enum.KeyCode.Unknown, function()
	spinToggle.Set(not spinToggle.Get())
end)

meTab:AddSection("Tamanho do personagem (R15)")
local function setSize(mult)
	local hum = getHumanoid()
	if not hum then
		return false
	end
	local found = false
	for _, n in ipairs({ "BodyHeightScale", "BodyWidthScale", "BodyDepthScale", "HeadScale" }) do
		local v = hum:FindFirstChild(n)
		if v and v:IsA("NumberValue") then
			v.Value = mult
			found = true
		end
	end
	return found
end

meTab:AddSlider("Tamanho", 0.3, 5, 1, function(v)
	state.size = v
	if not setSize(v) then
		Window:Notify("Mudar tamanho só funciona em personagem R15")
	end
end, 0.1)
meTab:AddButton("Resetar tamanho", function()
	state.size = 1
	setSize(1)
	Window:Notify("Tamanho resetado")
end)

meTab:AddSection("Teleporte")
meTab:AddToggle("Ctrl + Clique para teleportar", false, function(v)
	state.clickTp = v
end)

local savedPos
meTab:AddButton("Salvar posição atual", function()
	local root = getRoot()
	if root then
		savedPos = root.CFrame
		Window:Notify("Posição salva")
	end
end)
meTab:AddButton("Voltar para a posição salva", function()
	local root = getRoot()
	if root and savedPos then
		root.CFrame = savedPos
	else
		Window:Notify("Nenhuma posição salva ainda")
	end
end)
meTab:AddButton("Sentar / levantar", function()
	local hum = getHumanoid()
	if hum then
		hum.Sit = not hum.Sit
	end
end)

meTab:AddSection("Câmera")
meTab:AddSlider("Zoom máximo da câmera", 20, 2000, 128, function(v)
	state.zoomMax = v
	applyCamMode()
end, 10)

--// ============================================================
--//  ABA: VOAR (voo + fly car)
--// ============================================================
local flyTab = Window:CreateTab("Voar")

local flyBV, flyBG
local function stopFly()
	if flyBV then
		flyBV:Destroy()
		flyBV = nil
	end
	if flyBG then
		flyBG:Destroy()
		flyBG = nil
	end
	local hum = getHumanoid()
	if hum then
		hum.PlatformStand = false
	end
end

local function startFly()
	local root = getRoot()
	local hum = getHumanoid()
	if not root or not hum then
		return
	end
	stopFly()
	flyBV = Instance.new("BodyVelocity")
	flyBV.MaxForce = Vector3.new(1e6, 1e6, 1e6)
	flyBV.Velocity = Vector3.zero
	flyBV.Parent = root

	flyBG = Instance.new("BodyGyro")
	flyBG.MaxTorque = Vector3.new(1e6, 1e6, 1e6)
	flyBG.P = 9e4
	flyBG.CFrame = root.CFrame
	flyBG.Parent = root

	hum.PlatformStand = true
end

flyTab:AddSection("Voar (WASD)")
local flyToggle = flyTab:AddToggle("Voar", false, function(v)
	state.fly = v
	if v then
		startFly()
	else
		stopFly()
	end
end)
flyTab:AddSlider("Velocidade do voo", 10, 250, 60, function(v)
	state.flySpeed = v
end, 1, "flySpeed")
flyTab:AddLabel("WASD move | ESPAÇO sobe | CTRL desce")
registerBind("fly", "Voar (liga/desliga)", Enum.KeyCode.F, function()
	flyToggle.Set(not flyToggle.Get())
end)

--// FLY CAR (voa junto com a peça em que você está sentado)
local carBV, carBG, carRoot, carOffset
local sinkBound = false

local function setSpaceSink(on)
	if on == sinkBound then
		return
	end
	sinkBound = on
	if on then
		ContextActionService:BindActionAtPriority("NexusFlyCarSink", function()
			return Enum.ContextActionResult.Sink
		end, false, Enum.ContextActionPriority.High.Value, Enum.KeyCode.Space)
	else
		ContextActionService:UnbindAction("NexusFlyCarSink")
	end
end

local function stopFlyCar()
	if carBV then
		carBV:Destroy()
		carBV = nil
	end
	if carBG then
		carBG:Destroy()
		carBG = nil
	end
	carRoot = nil
	carOffset = nil
	setSpaceSink(false)
end

local function camYaw()
	local look = workspace.CurrentCamera.CFrame.LookVector
	local flat = Vector3.new(look.X, 0, look.Z)
	if flat.Magnitude < 0.01 then
		flat = Vector3.new(0, 0, -1)
	end
	return CFrame.lookAt(Vector3.zero, flat.Unit)
end

local function attachFlyCar(seat)
	stopFlyCar()
	local root = seat.AssemblyRootPart or seat
	carRoot = root
	carOffset = camYaw():Inverse() * root.CFrame.Rotation

	carBV = Instance.new("BodyVelocity")
	carBV.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
	carBV.Velocity = Vector3.zero
	carBV.Parent = root

	carBG = Instance.new("BodyGyro")
	carBG.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
	carBG.P = 9e4
	carBG.D = 1e3
	carBG.CFrame = root.CFrame
	carBG.Parent = root

	setSpaceSink(true)
end

flyTab:AddSection("Fly Car")
local flyCarToggle = flyTab:AddToggle("Fly Car (voar sentado em um veículo)", false, function(v)
	state.flyCar = v
	if not v then
		stopFlyCar()
	else
		Window:Notify("Fly Car ligado: sente em um veículo e use WASD + ESPAÇO/CTRL")
	end
end)
flyTab:AddSlider("Velocidade do Fly Car", 10, 400, 80, function(v)
	state.flyCarSpeed = v
end, 1, "flyCarSpeed")
flyTab:AddLabel("W/A/S/D = direção da câmera | ESPAÇO sobe | CTRL desce")
registerBind("flycar", "Fly Car (liga/desliga)", Enum.KeyCode.J, function()
	flyCarToggle.Set(not flyCarToggle.Get())
end)

connect(RunService.RenderStepped, function()
	if not state.flyCar then
		return
	end
	local hum = getHumanoid()
	local cam = workspace.CurrentCamera
	local seat = hum and hum.SeatPart
	if not seat or not cam then
		if carBV or carBG then
			stopFlyCar()
		end
		return
	end

	local root = seat.AssemblyRootPart or seat
	if root ~= carRoot or not carBV or not carBV.Parent or not carBG or not carBG.Parent then
		attachFlyCar(seat)
	end

	local look, right = cam.CFrame.LookVector, cam.CFrame.RightVector
	local dir = Vector3.zero
	if UserInputService:IsKeyDown(Enum.KeyCode.W) then dir += look end
	if UserInputService:IsKeyDown(Enum.KeyCode.S) then dir -= look end
	if UserInputService:IsKeyDown(Enum.KeyCode.D) then dir += right end
	if UserInputService:IsKeyDown(Enum.KeyCode.A) then dir -= right end
	if UserInputService:IsKeyDown(Enum.KeyCode.Space) or Window.MobileInput.up then dir += Vector3.yAxis end
	if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) or Window.MobileInput.down then dir -= Vector3.yAxis end

	if dir.Magnitude == 0 and hum.MoveDirection.Magnitude > 0 then
		dir = hum.MoveDirection
	end

	carBV.Velocity = dir.Magnitude > 0 and dir.Unit * state.flyCarSpeed or Vector3.zero
	carBG.CFrame = camYaw() * carOffset
end)

--// ============================================================
--//  ABA: VISUAL
--// ============================================================
local visualTab = Window:CreateTab("Visual")

visualTab:AddSlider("Campo de visão (FOV)", 40, 120, 70, function(v)
	if workspace.CurrentCamera then
		workspace.CurrentCamera.FieldOfView = v
	end
end, 1, "fov")

local originalLighting = {
	Brightness = Lighting.Brightness,
	ClockTime = Lighting.ClockTime,
	FogEnd = Lighting.FogEnd,
	GlobalShadows = Lighting.GlobalShadows,
	Ambient = Lighting.Ambient,
}
visualTab:AddToggle("Iluminação total (Fullbright)", false, function(v)
	if v then
		Lighting.Brightness = 2
		Lighting.ClockTime = 14
		Lighting.FogEnd = 1e6
		Lighting.GlobalShadows = false
		Lighting.Ambient = Color3.fromRGB(200, 200, 200)
	else
		for k, val in pairs(originalLighting) do
			Lighting[k] = val
		end
	end
end)

--// Remover neblina e efeitos (blur, bloom, sun rays...)
local fxBackup = {}
local fogBackup
local function scanFx(root)
	for _, inst in ipairs(root:GetDescendants()) do
		if inst:IsA("BlurEffect") or inst:IsA("BloomEffect") or inst:IsA("SunRaysEffect")
			or inst:IsA("DepthOfFieldEffect") then
			if fxBackup[inst] == nil then
				fxBackup[inst] = { kind = "fx", enabled = inst.Enabled }
			end
			inst.Enabled = false
		elseif inst:IsA("Atmosphere") then
			if fxBackup[inst] == nil then
				fxBackup[inst] = { kind = "atm", density = inst.Density, haze = inst.Haze }
			end
			inst.Density = 0
			inst.Haze = 0
		end
	end
end

local function applyNoFx(on)
	if on then
		if not fogBackup then
			fogBackup = { FogEnd = Lighting.FogEnd, FogStart = Lighting.FogStart }
		end
		Lighting.FogEnd = 1e6
		scanFx(Lighting)
		if workspace.CurrentCamera then
			scanFx(workspace.CurrentCamera)
		end
	else
		if fogBackup then
			Lighting.FogEnd = fogBackup.FogEnd
			Lighting.FogStart = fogBackup.FogStart
			fogBackup = nil
		end
		for inst, b in pairs(fxBackup) do
			if inst.Parent then
				if b.kind == "fx" then
					inst.Enabled = b.enabled
				else
					inst.Density = b.density
					inst.Haze = b.haze
				end
			end
		end
		fxBackup = {}
	end
end

visualTab:AddToggle("Remover neblina e efeitos (blur/bloom/rays)", false, function(v)
	state.noFx = v
	applyNoFx(v)
end, "noFx")

task.spawn(function()
	while Window.ScreenGui.Parent do
		task.wait(2)
		if state.noFx then
			applyNoFx(true)
		end
	end
end)

--// Hora do dia
visualTab:AddSlider("Hora do dia", 0, 24, 14, function(v)
	state.clockTime = v
	Lighting.ClockTime = v
end, 0.5)
visualTab:AddToggle("Travar hora do dia", false, function(v)
	state.clockLock = v
end)
task.spawn(function()
	while Window.ScreenGui.Parent do
		task.wait(0.25)
		if state.clockLock then
			Lighting.ClockTime = state.clockTime
		end
	end
end)

--// 1ª / 3ª pessoa forçada
visualTab:AddSection("Câmera")
local firstToggle, thirdToggle
firstToggle = visualTab:AddToggle("Forçar 1ª pessoa", false, function(v)
	state.firstP = v
	if v then
		state.thirdP = false
		if thirdToggle then
			thirdToggle.Set(false, true)
		end
	end
	applyCamMode()
end)
thirdToggle = visualTab:AddToggle("Forçar 3ª pessoa", false, function(v)
	state.thirdP = v
	if v then
		state.firstP = false
		if firstToggle then
			firstToggle.Set(false, true)
		end
	end
	applyCamMode()
end)

--// Tela esticada
local function setStretch(on)
	pcall(function()
		RunService:UnbindFromRenderStep("NexusStretch")
	end)
	if on then
		RunService:BindToRenderStep("NexusStretch", Enum.RenderPriority.Camera.Value + 2, function()
			local cam = workspace.CurrentCamera
			if cam then
				cam.CFrame = cam.CFrame * CFrame.new(0, 0, 0, 1, 0, 0, 0, state.stretch, 0, 0, 0, 1)
			end
		end)
	end
end
local stretchOn = false
visualTab:AddToggle("Tela esticada", false, function(v)
	stretchOn = v
	setStretch(v)
end)
visualTab:AddSlider("Proporção da tela (esticada)", 0.3, 1, 0.7, function(v)
	state.stretch = v
end, 0.05, "stretch")

--// FREECAM
local freecamData = { pos = Vector3.zero, yaw = 0, pitch = 0, wasAnchored = false }
local freecamToggle

local function freecamStep(dt)
	local cam = workspace.CurrentCamera
	if not cam then
		return
	end
	if cam.CameraType ~= Enum.CameraType.Scriptable then
		cam.CameraType = Enum.CameraType.Scriptable
	end
	UserInputService.MouseBehavior = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
		and Enum.MouseBehavior.LockCurrentPosition or Enum.MouseBehavior.Default

	local d = freecamData
	local rot = CFrame.fromOrientation(d.pitch, d.yaw, 0)
	local dir = Vector3.zero
	local K = Enum.KeyCode
	if UserInputService:IsKeyDown(K.W) then dir += rot.LookVector end
	if UserInputService:IsKeyDown(K.S) then dir -= rot.LookVector end
	if UserInputService:IsKeyDown(K.D) then dir += rot.RightVector end
	if UserInputService:IsKeyDown(K.A) then dir -= rot.RightVector end
	if UserInputService:IsKeyDown(K.E) or UserInputService:IsKeyDown(K.Space) or Window.MobileInput.up then
		dir += Vector3.yAxis
	end
	if UserInputService:IsKeyDown(K.Q) or UserInputService:IsKeyDown(K.LeftControl) or Window.MobileInput.down then
		dir -= Vector3.yAxis
	end
	local hum = getHumanoid()
	if dir.Magnitude == 0 and hum and hum.MoveDirection.Magnitude > 0 then
		dir = hum.MoveDirection
	end
	local spd = state.freecamSpeed * (UserInputService:IsKeyDown(K.LeftShift) and 3 or 1)
	if dir.Magnitude > 0 then
		d.pos += dir.Unit * spd * dt
	end
	cam.CFrame = CFrame.new(d.pos) * rot
end

local function setFreecam(on)
	local cam = workspace.CurrentCamera
	if on then
		if state.freecam or not cam then
			return
		end
		state.freecam = true
		freecamData.pos = cam.CFrame.Position
		local rx, ry = cam.CFrame:ToOrientation()
		freecamData.pitch, freecamData.yaw = rx, ry
		local root = getRoot()
		if root then
			freecamData.wasAnchored = root.Anchored
			root.Anchored = true
		end
		cam.CameraType = Enum.CameraType.Scriptable
		local K = Enum.KeyCode
		ContextActionService:BindActionAtPriority("NexusFreecamSink", function()
			return Enum.ContextActionResult.Sink
		end, false, Enum.ContextActionPriority.High.Value,
			K.W, K.A, K.S, K.D, K.Q, K.E, K.Space, K.LeftShift, K.LeftControl)
		RunService:BindToRenderStep("NexusFreecam", Enum.RenderPriority.Camera.Value + 1, freecamStep)
	else
		if not state.freecam then
			return
		end
		state.freecam = false
		pcall(function()
			RunService:UnbindFromRenderStep("NexusFreecam")
		end)
		pcall(function()
			ContextActionService:UnbindAction("NexusFreecamSink")
		end)
		UserInputService.MouseBehavior = Enum.MouseBehavior.Default
		local root = getRoot()
		if root and not freecamData.wasAnchored then
			root.Anchored = false
		end
		local hum = getHumanoid()
		if cam then
			cam.CameraType = Enum.CameraType.Custom
			if hum then
				cam.CameraSubject = hum
			end
		end
	end
end

connect(UserInputService.InputChanged, function(input, processed)
	if not state.freecam then
		return
	end
	local rotate = false
	if input.UserInputType == Enum.UserInputType.MouseMovement
		and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
		rotate = true
	elseif input.UserInputType == Enum.UserInputType.Touch and not processed then
		rotate = true
	end
	if rotate then
		freecamData.yaw -= input.Delta.X * 0.004
		freecamData.pitch = math.clamp(freecamData.pitch - input.Delta.Y * 0.004, -1.5, 1.5)
	end
end)

visualTab:AddSection("Freecam")
freecamToggle = visualTab:AddToggle("Freecam (câmera livre)", false, function(v)
	setFreecam(v)
	if v then
		Window:Notify("Freecam: WASD + E/Q, botão direito gira, SHIFT acelera")
	end
end)
visualTab:AddSlider("Velocidade da freecam", 10, 300, 60, function(v)
	state.freecamSpeed = v
end, 1, "freecamSpeed")
registerBind("freecam", "Freecam (liga/desliga)", Enum.KeyCode.P, function()
	freecamToggle.Set(not freecamToggle.Get())
end)

--// ESP com distância, barra de vida e cor por time
visualTab:AddSection("ESP")
local espFolder = Instance.new("Folder")
espFolder.Name = "NexusESP"
espFolder.Parent = Window.ScreenGui
local espCache = {}

local function teamColor(p)
	if p.Team and p.TeamColor then
		return p.TeamColor.Color
	end
	return Theme.Accent
end

local function clearESP(p)
	local e = espCache[p]
	if e then
		e.hl:Destroy()
		if e.bb then
			e.bb:Destroy()
		end
		espCache[p] = nil
	end
end

local function refreshESP()
	if not state.esp then
		for p in pairs(espCache) do
			clearESP(p)
		end
		return
	end
	for _, other in ipairs(Players:GetPlayers()) do
		if other ~= player then
			local ch = other.Character
			local e = espCache[other]
			if ch and (not e or e.char ~= ch) then
				clearESP(other)
				local hl = Instance.new("Highlight")
				hl.Adornee = ch
				hl.FillColor = Theme.Accent2
				hl.OutlineColor = Theme.Accent
				hl.FillTransparency = 0.6
				hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
				hl.Parent = espFolder

				local entry = { char = ch, hl = hl }
				local head = ch:FindFirstChild("Head")
				if head then
					local bb = Instance.new("BillboardGui")
					bb.Adornee = head
					bb.Size = UDim2.new(0, 170, 0, 34)
					bb.StudsOffset = Vector3.new(0, 2.8, 0)
					bb.AlwaysOnTop = true
					bb.Parent = espFolder

					local tl = Instance.new("TextLabel")
					tl.Size = UDim2.new(1, 0, 0, 18)
					tl.BackgroundTransparency = 1
					tl.Text = other.DisplayName
					tl.Font = Enum.Font.GothamBold
					tl.TextSize = 13
					tl.TextColor3 = Theme.Accent
					tl.TextStrokeTransparency = 0.4
					tl.Parent = bb

					local barBack = Instance.new("Frame")
					barBack.Size = UDim2.new(0.7, 0, 0, 6)
					barBack.Position = UDim2.new(0.15, 0, 0, 22)
					barBack.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
					barBack.BorderSizePixel = 0
					barBack.Parent = bb
					corner(barBack, 3)

					local barFill = Instance.new("Frame")
					barFill.Size = UDim2.new(1, 0, 1, 0)
					barFill.BackgroundColor3 = Color3.fromRGB(60, 255, 90)
					barFill.BorderSizePixel = 0
					barFill.Parent = barBack
					corner(barFill, 3)

					entry.bb, entry.name, entry.barBack, entry.barFill = bb, tl, barBack, barFill
				end
				espCache[other] = entry
			elseif not ch and e then
				clearESP(other)
			end
		end
	end
	for p in pairs(espCache) do
		if not p.Parent then
			clearESP(p)
		end
	end
end

local function updateESP()
	if not state.esp then
		return
	end
	local root = getRoot()
	for p, e in pairs(espCache) do
		local tc = teamColor(p)
		e.hl.FillColor = tc
		e.hl.OutlineColor = tc
		if e.name then
			local ch = p.Character
			local hum = ch and ch:FindFirstChildOfClass("Humanoid")
			local tr = ch and ch:FindFirstChild("HumanoidRootPart")
			local txt = p.DisplayName
			if state.espDist and root and tr then
				txt ..= string.format(" [%dm]", math.floor((tr.Position - root.Position).Magnitude))
			end
			e.name.Text = txt
			e.name.TextColor3 = tc
			e.barBack.Visible = state.espHealth
			if hum and hum.MaxHealth > 0 then
				local r = math.clamp(hum.Health / hum.MaxHealth, 0, 1)
				e.barFill.Size = UDim2.new(r, 0, 1, 0)
				e.barFill.BackgroundColor3 = Color3.fromRGB(255, 60, 60):Lerp(Color3.fromRGB(60, 255, 90), r)
			end
		end
	end
end

local espToggle = visualTab:AddToggle("ESP de jogadores (contorno + nome)", false, function(v)
	state.esp = v
	refreshESP()
end, "esp")
visualTab:AddToggle("ESP: mostrar distância", true, function(v)
	state.espDist = v
end, "espDist")
visualTab:AddToggle("ESP: mostrar barra de vida", true, function(v)
	state.espHealth = v
end, "espHealth")
registerBind("esp", "ESP (liga/desliga)", Enum.KeyCode.K, function()
	espToggle.Set(not espToggle.Get())
end)

task.spawn(function()
	while Window.ScreenGui.Parent do
		refreshESP()
		task.wait(1)
	end
end)
task.spawn(function()
	while Window.ScreenGui.Parent do
		updateESP()
		task.wait(0.15)
	end
end)

--// ============================================================
--//  ABA: UTILIDADES
--// ============================================================
local utilTab = Window:CreateTab("Utilidades")

--// ---------- Waypoints com nome ----------
utilTab:AddSection("Waypoints")
local wpBox = utilTab:AddTextbox("Nome", "ex: Casa")
local wpList
local wpSelected

local function rebuildWaypoints()
	wpList.Clear()
	local names = {}
	for n in pairs(Waypoints) do
		table.insert(names, n)
	end
	table.sort(names)
	if #names == 0 then
		wpList.Add("Nenhum waypoint salvo", nil, Theme.SubText)
		return
	end
	for _, n in ipairs(names) do
		local d = Waypoints[n]
		wpList.Add(string.format("%s   (%d, %d, %d)", n, d[1], d[2], d[3]), function()
			wpSelected = n
			local root = getRoot()
			if root then
				root.CFrame = CFrame.new(d[1], d[2] + 3, d[3])
				Window:Log("TP → " .. n)
			end
		end)
	end
end

utilTab:AddButton("Salvar waypoint aqui", function()
	local root = getRoot()
	if not root then
		return
	end
	local name = (wpBox.Text or ""):match("^%s*(.-)%s*$")
	if name == "" then
		local n = 1
		while Waypoints["WP " .. n] do
			n += 1
		end
		name = "WP " .. n
	end
	local p = root.Position
	Waypoints[name] = { math.floor(p.X), math.floor(p.Y), math.floor(p.Z) }
	wpBox.Text = ""
	rebuildWaypoints()
	Window:Notify("Waypoint salvo: " .. name)
end)
wpList = utilTab:AddList(130)
utilTab:AddLabel("Clique em um waypoint da lista para teleportar")
utilTab:AddButton("Deletar waypoint selecionado", function()
	if wpSelected and Waypoints[wpSelected] then
		Waypoints[wpSelected] = nil
		Window:Notify("Waypoint removido: " .. wpSelected)
		wpSelected = nil
		rebuildWaypoints()
	else
		Window:Notify("Clique em um waypoint da lista primeiro")
	end
end)
rebuildWaypoints()

--// ---------- Gravar e repetir trajeto ----------
utilTab:AddSection("Gravar e repetir trajeto")
local pathPoints = {}
local recording, playing = false, false

utilTab:AddToggle("Gravar trajeto", false, function(v)
	recording = v
	if v then
		pathPoints = {}
		Window:Notify("Gravando trajeto... ande até o destino")
	else
		Window:Notify("Trajeto gravado: " .. #pathPoints .. " pontos")
	end
end)
utilTab:AddToggle("Repetir em loop", false, function(v)
	state.pathLoop = v
end)
utilTab:AddButton("▶ Repetir trajeto gravado", function()
	if #pathPoints == 0 then
		Window:Notify("Nenhum trajeto gravado")
		return
	end
	if playing then
		return
	end
	playing = true
	Window:Notify("Repetindo trajeto")
	repeat
		local prev
		for _, cf in ipairs(pathPoints) do
			if not playing then
				break
			end
			local root = getRoot()
			if not root then
				playing = false
				break
			end
			for k = 1, 3 do
				if prev then
					root.CFrame = prev:Lerp(cf, k / 3)
				else
					root.CFrame = cf
				end
				root.AssemblyLinearVelocity = Vector3.zero
				task.wait(0.033)
			end
			prev = cf
		end
	until not playing or not state.pathLoop
	playing = false
end)
utilTab:AddButton("■ Parar repetição", function()
	playing = false
end)

task.spawn(function()
	while Window.ScreenGui.Parent do
		task.wait(0.1)
		if recording then
			local root = getRoot()
			if root and #pathPoints < 6000 then
				table.insert(pathPoints, root.CFrame)
			end
		end
	end
end)

--// ---------- Clique para deletar peças (só local) ----------
utilTab:AddSection("Deletar peças do mapa (só do seu lado)")
local deletedStack = {}

local function raycastFromScreen(pos)
	local cam = workspace.CurrentCamera
	local ray = cam:ScreenPointToRay(pos.X, pos.Y)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { player.Character }
	return workspace:Raycast(ray.Origin, ray.Direction * 1000, params)
end

utilTab:AddToggle("Clique para deletar peça", false, function(v)
	state.clickDelete = v
end)
utilTab:AddButton("Desfazer última peça deletada", function()
	local last = table.remove(deletedStack)
	if last and last[1] then
		last[1].Parent = last[2]
		Window:Notify("Peça restaurada")
	else
		Window:Notify("Nada para desfazer")
	end
end)
utilTab:AddButton("Restaurar todas as peças deletadas", function()
	for _, d in ipairs(deletedStack) do
		if d[1] then
			d[1].Parent = d[2]
		end
	end
	deletedStack = {}
	Window:Notify("Todas restauradas")
end)

--// ---------- Explorador de objetos ----------
utilTab:AddSection("Explorador de objetos")
local explorerCur = workspace
local explorerSel
local explorerLabel = utilTab:AddLabel("Workspace")
explorerLabel.TextColor3 = Theme.Accent
local explorerList

local function explorerRefresh()
	explorerList.Clear()
	explorerSel = nil
	explorerLabel.Text = explorerCur:GetFullName()
	local kids = explorerCur:GetChildren()
	for i, inst in ipairs(kids) do
		if i > 250 then
			explorerList.Add("... (mais itens não exibidos)", nil, Theme.SubText)
			break
		end
		local more = #inst:GetChildren() > 0 and "  ›" or ""
		explorerList.Add(inst.Name .. "  [" .. inst.ClassName .. "]" .. more, function()
			explorerSel = inst
		end)
	end
	if #kids == 0 then
		explorerList.Add("(vazio)", nil, Theme.SubText)
	end
end

explorerList = utilTab:AddList(170)
utilTab:AddButton("Entrar no selecionado", function()
	if explorerSel and explorerSel.Parent then
		explorerCur = explorerSel
		explorerRefresh()
	else
		Window:Notify("Selecione um item na lista")
	end
end)
utilTab:AddButton("Voltar (pasta pai)", function()
	if explorerCur.Parent and explorerCur ~= game then
		explorerCur = explorerCur.Parent
		explorerRefresh()
	end
end)
utilTab:AddButton("Ir para o Workspace", function()
	explorerCur = workspace
	explorerRefresh()
end)
utilTab:AddButton("Teleportar até o selecionado", function()
	local root = getRoot()
	local s = explorerSel
	if not (root and s and s.Parent) then
		Window:Notify("Selecione um item na lista")
		return
	end
	local cf
	if s:IsA("BasePart") then
		cf = s.CFrame
	elseif s:IsA("Model") then
		cf = s:GetPivot()
	end
	if cf then
		root.CFrame = cf + Vector3.new(0, 5, 0)
	else
		Window:Notify("Esse item não tem posição")
	end
end)
utilTab:AddButton("Copiar caminho do selecionado", function()
	if not explorerSel then
		Window:Notify("Selecione um item na lista")
		return
	end
	local ok = pcall(function()
		setclipboard(explorerSel:GetFullName())
	end)
	Window:Notify(ok and "Caminho copiado" or "Seu executor não suporta copiar")
end)
utilTab:AddButton("Deletar selecionado (só local)", function()
	local s = explorerSel
	if s and s.Parent then
		table.insert(deletedStack, { s, s.Parent })
		s.Parent = nil
		explorerRefresh()
	end
end)
explorerRefresh()

--// ---------- ProximityPrompts ----------
utilTab:AddSection("ProximityPrompts (portas, itens...)")
local promptCache = {}
local function refreshPrompts()
	promptCache = {}
	for _, d in ipairs(workspace:GetDescendants()) do
		if d:IsA("ProximityPrompt") then
			table.insert(promptCache, d)
		end
	end
end

local function promptPosition(pr)
	local par = pr.Parent
	if par and par:IsA("BasePart") then
		return par.Position
	elseif par and par:IsA("Attachment") then
		return par.WorldPosition
	elseif par and par:IsA("Model") then
		return par:GetPivot().Position
	end
end

local function firePromptsNearby()
	local root = getRoot()
	if not root then
		return 0
	end
	local n = 0
	for _, pr in ipairs(promptCache) do
		if pr.Parent and pr.Enabled then
			local pos = promptPosition(pr)
			if pos and (pos - root.Position).Magnitude <= pr.MaxActivationDistance + 2 then
				if pcall(fireproximityprompt, pr) then
					n += 1
				end
			end
		end
	end
	return n
end

utilTab:AddToggle("Auto-ativar ProximityPrompts próximos", false, function(v)
	state.autoPrompt = v
	if v then
		if not fireproximityprompt then
			Window:Notify("Seu executor não suporta fireproximityprompt")
		end
		refreshPrompts()
	end
end)
utilTab:AddToggle("Prompts instantâneos (sem segurar)", false, function(v)
	state.instantPrompt = v
	if v then
		refreshPrompts()
	end
end)
utilTab:AddButton("Ativar prompts próximos agora", function()
	refreshPrompts()
	local n = firePromptsNearby()
	Window:Notify(n .. " prompt(s) ativado(s)")
end)

task.spawn(function()
	local t = 0
	while Window.ScreenGui.Parent do
		task.wait(0.25)
		if state.autoPrompt or state.instantPrompt then
			t += 0.25
			if t >= 5 then
				refreshPrompts()
				t = 0
			end
			if state.instantPrompt then
				for _, pr in ipairs(promptCache) do
					if pr.Parent and pr.HoldDuration ~= 0 then
						pr.HoldDuration = 0
					end
				end
			end
			if state.autoPrompt then
				firePromptsNearby()
			end
		end
	end
end)

--// ---------- Auto-clicker ----------
utilTab:AddSection("Auto-clicker")
local autoClickToggle = utilTab:AddToggle("Auto-clicker", false, function(v)
	state.autoClick = v
	if v then
		Window:Notify("Deixe o mouse fora do painel para clicar no jogo")
	end
end)
utilTab:AddSlider("Cliques por segundo", 1, 50, 10, function(v)
	state.cps = v
end, 1, "cps")
registerBind("autoclick", "Auto-clicker (liga/desliga)", Enum.KeyCode.M, function()
	autoClickToggle.Set(not autoClickToggle.Get())
end)

task.spawn(function()
	while Window.ScreenGui.Parent do
		if state.autoClick then
			task.wait(1 / state.cps)
			local loc = UserInputService:GetMouseLocation()
			if state.autoClick and not Window:IsOverPanel(loc) then
				if VirtualInputManager then
					pcall(function()
						VirtualInputManager:SendMouseButtonEvent(loc.X, loc.Y, 0, true, game, 0)
						VirtualInputManager:SendMouseButtonEvent(loc.X, loc.Y, 0, false, game, 0)
					end)
				elseif mouse1click then
					pcall(mouse1click)
				end
			end
		else
			task.wait(0.2)
		end
	end
end)

--// ============================================================
--//  ABA: DIVERSÃO (emotes, animações, aparência)
--// ============================================================
local funTab = Window:CreateTab("Diversão")

--// ---------- Emotes e danças ----------
funTab:AddSection("Emotes e danças")
local Emotes = {
	{ "Dança 1", 507771019 },
	{ "Dança 2", 507776043 },
	{ "Dança 3", 507777268 },
	{ "Comemorar", 507770677 },
	{ "Rir", 507770818 },
	{ "Apontar", 507770453 },
	{ "Acenar", 507770239 },
	{ "Continência", 3360689775 },
	{ "Inclinar", 3360692915 },
	{ "Estádio", 3360686498 },
	{ "Dar de ombros", 3576968026 },
}
local currentEmote

local function stopEmote()
	if currentEmote then
		currentEmote:Stop()
		currentEmote = nil
	end
end

local function playEmote(id)
	local hum = getHumanoid()
	if not hum then
		return
	end
	local animator = hum:FindFirstChildOfClass("Animator")
	if not animator then
		Window:Notify("Personagem sem Animator")
		return
	end
	stopEmote()
	local a = Instance.new("Animation")
	a.AnimationId = "rbxassetid://" .. id
	local ok, track = pcall(function()
		return animator:LoadAnimation(a)
	end)
	if ok and track then
		track.Priority = Enum.AnimationPriority.Action
		track.Looped = true
		track:Play()
		currentEmote = track
	else
		Window:Notify("Não foi possível tocar esse emote")
	end
end

local emoteList = funTab:AddList(150)
for _, e in ipairs(Emotes) do
	emoteList.Add(e[1], function()
		playEmote(e[2])
		Window:Log("Emote: " .. e[1])
	end)
end
funTab:AddButton("Parar emote", stopEmote)
funTab:AddLabel("O emote para sozinho quando você anda")

connect(RunService.Heartbeat, function()
	if currentEmote then
		local hum = getHumanoid()
		if hum and hum.MoveDirection.Magnitude > 0.1 then
			stopEmote()
		end
	end
end)

--// ---------- Animação de andar e correr ----------
funTab:AddSection("Animação de andar / correr")
local AnimPacks = {
	{ "Ninja", 656117400, 656121766, 656118852 },
	{ "Zumbi", 616158929, 616168032, 616163682 },
	{ "Robô", 616088211, 616095330, 616091570 },
	{ "Levitação", 616006778, 616013216, 616010382 },
	{ "Astronauta", 891621366, 891667138, 891636393 },
	{ "Cartoon", 742637544, 742640026, 742638842 },
	{ "Estiloso", 616136790, 616146177, 616140816 },
	{ "Mago", 707742142, 707897309, 707861613 },
	{ "Pirata", 750781874, 750785693, 750783738 },
	{ "Cavaleiro", 657595757, 657552124, 657564596 },
	{ "Lobisomem", 1083195517, 1083178339, 1083216690 },
	{ "Idoso", 845397899, 845403856, 845386501 },
	{ "Bubbly", 910004836, 910034870, 910025107 },
	{ "Vampiro", 1083445855, 1083473930, 1083462077 },
	{ "Brinquedo", 782841498, 782843345, 782842708 },
	{ "Super-herói", 616111295, 616122287, 616117076 },
}
local animBackup = setmetatable({}, { __mode = "k" })

local function animSlot(animate, folder, name)
	local f = animate:FindFirstChild(folder)
	return f and f:FindFirstChild(name)
end

local function applyAnimPack(pack)
	local char = getCharacter()
	local animate = char and char:FindFirstChild("Animate")
	if not animate then
		Window:Notify("Sem script Animate (funciona em R15 padrão)")
		return false
	end
	local slots = {
		{ "idle", "Animation1", "idle" },
		{ "idle", "Animation2", "idle" },
		{ "walk", "WalkAnim", "walk" },
		{ "run", "RunAnim", "run" },
	}
	if not animBackup[animate] then
		animBackup[animate] = {}
		for _, s in ipairs(slots) do
			local obj = animSlot(animate, s[1], s[2])
			if obj then
				animBackup[animate][s[2]] = obj.AnimationId
			end
		end
	end
	animate.Disabled = true
	for _, s in ipairs(slots) do
		local obj = animSlot(animate, s[1], s[2])
		if obj then
			if pack then
				obj.AnimationId = "rbxassetid://" .. pack[s[3]]
			elseif animBackup[animate][s[2]] then
				obj.AnimationId = animBackup[animate][s[2]]
			end
		end
	end
	local hum = getHumanoid()
	if hum then
		for _, t in ipairs(hum:GetPlayingAnimationTracks()) do
			t:Stop()
		end
	end
	animate.Disabled = false
	return true
end

local animList = funTab:AddList(150)
animList.Add("Padrão (original)", function()
	if applyAnimPack(nil) then
		state.animPack = nil
		Window:Notify("Animação padrão restaurada")
	end
end)
for _, p in ipairs(AnimPacks) do
	local pack = { idle = p[2], walk = p[3], run = p[4] }
	animList.Add(p[1], function()
		if applyAnimPack(pack) then
			state.animPack = pack
			Window:Notify("Animação: " .. p[1])
		end
	end)
end

--// ---------- Aparência (só você vê) ----------
funTab:AddSection("Aparência (só você vê)")
local originalDesc

local function copyAvatar(userId)
	local hum = getHumanoid()
	if not hum then
		return
	end
	task.spawn(function()
		local ok, desc = pcall(function()
			return Players:GetHumanoidDescriptionFromUserId(userId)
		end)
		if not ok or not desc then
			Window:Notify("Não consegui obter o avatar")
			return
		end
		if not originalDesc then
			pcall(function()
				originalDesc = hum:GetAppliedDescription()
			end)
		end
		local ok2 = pcall(function()
			hum:ApplyDescription(desc)
		end)
		Window:Notify(ok2 and "Avatar aplicado (só você vê)" or "Falha ao aplicar avatar")
	end)
end

funTab:AddButton("Copiar avatar do jogador selecionado", function()
	if not selectedPlayer then
		Window:Notify("Selecione um jogador na aba Jogadores")
		return
	end
	copyAvatar(selectedPlayer.UserId)
end)
funTab:AddTextbox("Por usuário", "nome do usuário", function(text)
	if text == "" then
		return
	end
	local ok, id = pcall(function()
		return Players:GetUserIdFromNameAsync(text)
	end)
	if ok and id then
		copyAvatar(id)
	else
		Window:Notify("Usuário não encontrado")
	end
end)
funTab:AddButton("Restaurar meu avatar", function()
	local hum = getHumanoid()
	if not hum then
		return
	end
	task.spawn(function()
		local desc = originalDesc
		if not desc then
			pcall(function()
				desc = Players:GetHumanoidDescriptionFromUserId(player.UserId)
			end)
		end
		local ok = desc and pcall(function()
			hum:ApplyDescription(desc)
		end)
		Window:Notify(ok and "Avatar restaurado" or "Não foi possível restaurar")
	end)
end)

--// ============================================================
--//  ABA: EXTRAS
--// ============================================================
local miscTab = Window:CreateTab("Extras")

miscTab:AddToggle("Anti-AFK", true, function(v)
	state.antiAfk = v
end)
connect(player.Idled, function()
	if state.antiAfk then
		local vu = game:GetService("VirtualUser")
		vu:CaptureController()
		vu:ClickButton2(Vector2.new())
	end
end)

local statsLabel = miscTab:AddLabel("FPS: ...")
statsLabel.TextColor3 = Theme.Accent
do
	local frames, acc = 0, 0
	connect(RunService.Heartbeat, function(dt)
		frames += 1
		acc += dt
		if acc >= 0.5 then
			local fps = math.floor(frames / acc + 0.5)
			local ping = "?"
			pcall(function()
				ping = math.floor(game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue())
			end)
			statsLabel.Text = "FPS: " .. fps .. "  |  Ping: " .. tostring(ping) .. " ms"
			frames, acc = 0, 0
		end
	end)
end

miscTab:AddButton("Resetar personagem", function()
	local hum = getHumanoid()
	if hum then
		hum.Health = 0
	end
end)

miscTab:AddButton("Reentrar no servidor", function()
	TeleportService:Teleport(game.PlaceId, player)
end)

--// ============================================================
--//  ABA: CONFIG (painel, temas, atalhos, salvar)
--// ============================================================
local configTab = Window:CreateTab("Config")

--// ---------- salvar / carregar ----------
local function saveConfig(silent)
	if not writefile then
		if not silent then
			Window:Notify("Seu executor não suporta salvar arquivos")
		end
		return
	end
	local data = { toggles = {}, sliders = {}, keys = {}, favorites = Favorites, waypoints = Waypoints, misc = Saved.misc }
	for id, api in pairs(Library.Toggles) do
		data.toggles[id] = api.Get()
	end
	for id, api in pairs(Library.Sliders) do
		data.sliders[id] = api.Get()
	end
	for id, b in pairs(Binds) do
		data.keys[id] = b.key.Name
	end
	local m, f = Window.Main.Position, Window.FloatBtn.Position
	data.misc.mainPos = { m.X.Scale, m.X.Offset, m.Y.Scale, m.Y.Offset }
	data.misc.floatPos = { f.X.Scale, f.X.Offset, f.Y.Scale, f.Y.Offset }
	local ok = pcall(function()
		writefile(CONFIG_FILE, HttpService:JSONEncode(data))
	end)
	if not silent then
		Window:Notify(ok and "Configurações salvas" or "Falha ao salvar")
	end
end

-- restaura a posição do painel e do botão flutuante
do
	local mp, fp = Saved.misc.mainPos, Saved.misc.floatPos
	if type(mp) == "table" and #mp == 4 then
		Window.Main.Position = UDim2.new(mp[1], mp[2], mp[3], mp[4])
	end
	if type(fp) == "table" and #fp == 4 then
		Window.FloatBtn.Position = UDim2.new(fp[1], fp[2], fp[3], fp[4])
	end
end

configTab:AddSection("Imagem do botão flutuante")
configTab:AddTextbox("ID ou link da imagem", "rbxassetid://... ou número", function(text, enter)
	if text == "" then
		return
	end
	if Window:SetLogo(text) then
		Window:Notify("Imagem atualizada")
	end
end)
configTab:AddButton("Usar foto do meu avatar", function()
	Window:SetLogo("rbxthumb://type=AvatarHeadShot&id=" .. player.UserId .. "&w=150&h=150")
	Window:Notify("Foto do avatar aplicada")
end)
configTab:AddLabel("Use o ID de uma IMAGEM (não decal) do Roblox")

configTab:AddSection("Tema de cores (troca ao vivo)")
for _, name in ipairs(ThemeOrder) do
	configTab:AddButton(name, function()
		Window:SetTheme(name)
		Window:Notify("Tema: " .. name)
	end)
end

configTab:AddSection("Aparência do painel")
configTab:AddSlider("Transparência (opacidade)", 0.3, 1, 1, function(v)
	Window:SetOpacity(v)
end, 0.05, "opacity")
configTab:AddSlider("Tamanho do painel", 0.6, 1.6, 1, function(v)
	Window:SetScale(v)
end, 0.05, "panelScale")
configTab:AddToggle("Modo mobile (botões maiores)", UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled, function(v)
	Window:SetMobile(v)
end, "mobile")
configTab:AddToggle("Log de ações na tela", true, function(v)
	Window.LogEnabled = v
end, "logOn")

configTab:AddSection("Atalhos (clique e aperte a tecla)")
configTab:AddLabel("ESC cancela | BACKSPACE remove o atalho")
for _, id in ipairs(BindOrder) do
	configTab:AddKeybind(Binds[id].label, id)
end

configTab:AddSection("Configurações salvas")
configTab:AddToggle("Salvar automaticamente", true, function() end, "autosave")
configTab:AddButton("Salvar agora", function()
	saveConfig(false)
end)
configTab:AddButton("Apagar configuração salva", function()
	local ok = pcall(function()
		if isfile and isfile(CONFIG_FILE) then
			delfile(CONFIG_FILE)
		end
	end)
	Window:Notify(ok and "Configuração apagada (vale na próxima abertura)" or "Não foi possível apagar")
end)

configTab:AddSection("Painel")
configTab:AddLabel("Abra/feche pelo atalho ou pelo quadrado flutuante")

local function cleanupAll()
	saveConfig(true)
	stopFollow()
	if tpLoopConn then
		tpLoopConn:Disconnect()
		tpLoopConn = nil
	end
	spectating = nil
	recording = false
	playing = false
	stopEmote()
	local cam = workspace.CurrentCamera
	local hum = getHumanoid()
	if cam and hum then
		cam.CameraSubject = hum
	end
	stopFly()
	stopFlyCar()
	setFreecam(false)
	setStretch(false)
	applyNoFx(false)
	state.esp = false
	refreshESP()
	state.firstP, state.thirdP = false, false
	applyCamMode()
	Window:Destroy()
end

configTab:AddButton("Fechar e remover o painel", cleanupAll)

-- autosave a cada 15s
task.spawn(function()
	while Window.ScreenGui.Parent do
		task.wait(15)
		local t = Library.Toggles["autosave"]
		if t and t.Get() then
			saveConfig(true)
		end
	end
end)

-- mostra os botões de subir/descer no celular quando preciso
task.spawn(function()
	while Window.ScreenGui.Parent do
		Window.MobileBar.Visible = Window.Mobile and (state.fly or state.flyCar or state.freecam)
		task.wait(0.3)
	end
end)

--// ============================================================
--//  LOOPS E EVENTOS GLOBAIS
--// ============================================================
local waterPlat
local waterParams = RaycastParams.new()
waterParams.FilterType = Enum.RaycastFilterType.Exclude
waterParams.IgnoreWater = false
local lastSafe
local safeAcc = 0
local swimApplied = false

connect(RunService.Heartbeat, function(dt)
	local hum = getHumanoid()
	local root = getRoot()
	if not hum then
		return
	end

	-- velocidade / nado rápido
	if not state.freecam then
		local swimming = state.fastSwim and hum:GetState() == Enum.HumanoidStateType.Swimming
		if swimming then
			swimApplied = true
			if hum.WalkSpeed ~= state.swimSpeed then
				hum.WalkSpeed = state.swimSpeed
			end
		else
			if swimApplied then
				swimApplied = false
				hum.WalkSpeed = state.speedOn and state.speed or 16
			end
			if state.speedOn and hum.WalkSpeed ~= state.speed then
				hum.WalkSpeed = state.speed
			end
		end
	end

	-- pulo alto
	if state.jumpOn then
		hum.UseJumpPower = true
		hum.JumpPower = state.jumpPower
	end

	-- auto-pulo
	if state.autoJump and hum.MoveDirection.Magnitude > 0 and hum.FloorMaterial ~= Enum.Material.Air then
		hum.Jump = true
	end

	-- modo de câmera forçado
	if state.firstP then
		if player.CameraMaxZoomDistance ~= 0.5 then
			applyCamMode()
		end
	elseif state.thirdP then
		if player.CameraMinZoomDistance ~= 12 then
			applyCamMode()
		end
	end

	if root then
		-- andar na água
		if state.walkWater then
			waterParams.FilterDescendantsInstances = { getCharacter(), waterPlat }
			local res = workspace:Raycast(root.Position + Vector3.new(0, 3, 0), Vector3.new(0, -12, 0), waterParams)
			if res and res.Material == Enum.Material.Water then
				if not waterPlat then
					waterPlat = Instance.new("Part")
					waterPlat.Name = "NexusWaterPlat"
					waterPlat.Size = Vector3.new(14, 1, 14)
					waterPlat.Anchored = true
					waterPlat.CanCollide = true
					waterPlat.Transparency = 1
				end
				waterPlat.CFrame = CFrame.new(root.Position.X, res.Position.Y - 0.5, root.Position.Z)
				waterPlat.Parent = workspace
			elseif waterPlat then
				waterPlat.Parent = nil
			end
		elseif waterPlat and waterPlat.Parent then
			waterPlat.Parent = nil
		end

		-- anti-queda
		if state.antiFall then
			safeAcc += dt
			if safeAcc > 0.3 and hum.FloorMaterial ~= Enum.Material.Air
				and not state.fly and not state.noclip and not state.freecam then
				lastSafe = root.CFrame
				safeAcc = 0
			end
			if lastSafe then
				local lowY = workspace.FallenPartsDestroyHeight + 120
				local plunging = root.AssemblyLinearVelocity.Y < -150 and root.Position.Y < lastSafe.Y - 200
				if root.Position.Y < lowY or plunging then
					root.AssemblyLinearVelocity = Vector3.zero
					root.CFrame = lastSafe + Vector3.new(0, 3, 0)
					Window:Log("Anti-queda: voltou à posição segura")
				end
			end
		end

		-- spinbot
		if state.spin and not state.fly and not state.freecam then
			root.CFrame = root.CFrame * CFrame.Angles(0, math.rad(state.spinSpeed * dt), 0)
		end
	end
end)

-- Noclip
connect(RunService.Stepped, function()
	if not state.noclip then
		return
	end
	local char = getCharacter()
	if not char then
		return
	end
	for _, part in ipairs(char:GetDescendants()) do
		if part:IsA("BasePart") then
			part.CanCollide = false
		end
	end
end)

-- Pulo infinito
connect(UserInputService.JumpRequest, function()
	if not state.infJump or state.fly then
		return
	end
	local hum = getHumanoid()
	if hum then
		hum:ChangeState(Enum.HumanoidStateType.Jumping)
	end
end)

-- Pulo na parede
connect(UserInputService.JumpRequest, function()
	if not state.wallJump or state.fly then
		return
	end
	local hum, root = getHumanoid(), getRoot()
	if not hum or not root or hum.FloorMaterial ~= Enum.Material.Air then
		return
	end
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { getCharacter() }
	for i = 0, 7 do
		local a = math.rad(i * 45)
		local dir = Vector3.new(math.cos(a), 0, math.sin(a))
		local res = workspace:Raycast(root.Position, dir * 3, params)
		if res and res.Instance.CanCollide then
			root.AssemblyLinearVelocity = res.Normal * 35 + Vector3.new(0, 55, 0)
			return
		end
	end
end)

-- Ctrl + clique para teleportar  /  clique para deletar peça
local mouse = player:GetMouse()
connect(UserInputService.InputBegan, function(input, processed)
	if processed then
		return
	end
	local isClick = input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch

	if state.clickTp and input.UserInputType == Enum.UserInputType.MouseButton1
		and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
		local root = getRoot()
		if root and mouse.Hit then
			root.CFrame = CFrame.new(mouse.Hit.Position + Vector3.new(0, 3, 0))
		end
		return
	end

	if state.clickDelete and isClick then
		local res = raycastFromScreen(input.Position)
		if res and res.Instance and not res.Instance:IsA("Terrain") then
			table.insert(deletedStack, { res.Instance, res.Instance.Parent })
			res.Instance.Parent = nil
			Window:Log("Peça deletada (local)")
		end
	end
end)

-- Voo
connect(RunService.RenderStepped, function()
	if not state.fly or not flyBV or not flyBG then
		return
	end
	local hum = getHumanoid()
	local root = getRoot()
	local cam = workspace.CurrentCamera
	if not hum or not root or not cam then
		return
	end

	local move = cam.CFrame:VectorToObjectSpace(hum.MoveDirection)
	local dir = Vector3.zero
	if move.Magnitude > 0 then
		local unit = move.Unit
		dir = cam.CFrame.LookVector * -unit.Z + cam.CFrame.RightVector * unit.X
	end

	if UserInputService:IsKeyDown(Enum.KeyCode.Space) or Window.MobileInput.up then
		dir += Vector3.new(0, 1, 0)
	end
	if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) or Window.MobileInput.down then
		dir -= Vector3.new(0, 1, 0)
	end

	flyBV.Velocity = dir * state.flySpeed
	flyBG.CFrame = CFrame.lookAt(root.Position, root.Position + cam.CFrame.LookVector)
end)

-- Renascimento: reaplica voo, tamanho e animação
connect(player.CharacterAdded, function(char)
	char:WaitForChild("Humanoid")
	task.wait(0.5)
	if state.fly then
		startFly()
	end
	if state.size ~= 1 then
		setSize(state.size)
	end
	lastSafe = nil
	if state.animPack then
		char:WaitForChild("Animate", 5)
		applyAnimPack(state.animPack)
	end
end)

-- aplica configurações visuais salvas
Window:SetOpacity(Library.Sliders["opacity"] and Library.Sliders["opacity"].Get() or 1)

Window:Notify("NEXUS PANEL carregado")
print("NEXUS PANEL v3 carregado")
