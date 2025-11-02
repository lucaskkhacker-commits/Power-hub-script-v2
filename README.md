-- ⚡ Power Hub V3.6 Clean Compact +13% (Full Systems + Scroll Edition) — Atualizado (View Player: LISTA SCROLLABLE DE JOGADORES + botões vermelho escuro)
-- Integra: Intro, Fade, Música, Menu, Scroll (X+Y touch+mouse), AutoFarm, TP Tool, Boombox,
-- Speed/Jump, Aim Sist Apenas Bots, Precision Overlay, ESP (Highlights + Rings), TargetBox,
-- Shift Lock Real (Scriptable, câmera fixa), HyperLaserGun, View Player (LISTA SCROLLABLE), mini toggle, cleanup.

-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = workspace
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local humanoid = char:WaitForChild("Humanoid")

-- Escala +13% aplicada (valores arredondados)
local MENU_WIDTH, MENU_HEIGHT = 356, 356 -- 315 -> 356 (x1.13)
local BTN_W, BTN_H = 136, 36             -- 120x32 -> 136x36
local SPACING_X, SPACING_Y = 162, 46
local OFFSET_X, OFFSET_Y = 18, 18

-- Botão padrão vermelho escuro (#8B0000)
local BUTTON_RED = Color3.fromRGB(139, 0, 0)

-- Variáveis principais
local walkSpeed, jumpPower = 16, 26
local prevWalkSpeed = walkSpeed
local farming = false
local farmLoop
local safePos = nil
local currentSound, musicPlaying = nil
local introSound = nil
local aimLoop
local precisionOn = false
local aimAssist = false
local espOn = false
local shiftLocked = false

-- View Player state
local viewingPlayer = nil
local viewLoop = nil
local viewModeActive = false
local backViewBtn = nil
local playersLineFrame = nil -- Frame para a linha de jogadores
local playersLineVisible = false

-- camera original vars (already used by shift lock; reuse safely)
local originalMouseBehavior = UIS.MouseBehavior
local originalMouseIconEnabled = UIS.MouseIconEnabled
local originalCameraType = nil
local originalCameraSubject = nil

--------------------------------------------------------------------
-- Intro (6s screen + music ~17s)
local introGui = Instance.new("ScreenGui")
introGui.Name = "PowerHubIntro"
introGui.ResetOnSpawn = false
introGui.IgnoreGuiInset = true
introGui.Parent = player:WaitForChild("PlayerGui")

local introFrame = Instance.new("Frame")
introFrame.Size = UDim2.new(1,0,1,0)
introFrame.BackgroundColor3 = Color3.fromRGB(20,20,20)
introFrame.BackgroundTransparency = 1
introFrame.Parent = introGui

local introText = Instance.new("TextLabel")
introText.Size = UDim2.new(1,0,1,0)
introText.BackgroundTransparency = 1
introText.Text = "⚡ Power Hub V3.6 Clean Compact +13% ⚡"
introText.TextColor3 = Color3.new(1,1,1)
introText.Font = Enum.Font.SourceSansBold
introText.TextSize = 40
introText.TextTransparency = 1
introText.Parent = introFrame

introSound = Instance.new("Sound")
introSound.SoundId = "rbxassetid://142376088"
introSound.Volume = 1
introSound.Looped = false
introSound.Parent = Workspace
pcall(function() introSound:Play() end)

for i = 1, 10 do
	introFrame.BackgroundTransparency = 1 - (i/10)
	introText.TextTransparency = 1 - (i/10)
	task.wait(0.02)
end

task.spawn(function()
	task.wait(6)
	for i = 0, 1, 0.05 do
		introFrame.BackgroundTransparency = i
		introText.TextTransparency = i
		task.wait(0.04)
	end
	introGui:Destroy()
end)

task.spawn(function()
	task.wait(17)
	if introSound then
		pcall(function() introSound:Stop() end)
		introSound:Destroy()
		introSound = nil
	end
end)

--------------------------------------------------------------------
-- Main GUI (single ScreenGui used for the hub)
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "PowerHubGUI"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.Parent = player:WaitForChild("PlayerGui")

-- create menu frame (fixo, aumentado +13%)
local menuFrame = Instance.new("Frame")
menuFrame.Size = UDim2.new(0, MENU_WIDTH, 0, MENU_HEIGHT)
menuFrame.Position = UDim2.new(0, 50, 0, 80)
menuFrame.BackgroundColor3 = Color3.fromRGB(180,20,20) -- classic red (menu background kept)
menuFrame.Active = true
menuFrame.Selectable = true
menuFrame.Visible = true -- start visible
menuFrame.Parent = screenGui
Instance.new("UICorner", menuFrame).CornerRadius = UDim.new(0,12)

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(0, MENU_WIDTH, 0, 26)
titleLabel.Position = UDim2.new(0, 0, 0, -28)
titleLabel.BackgroundColor3 = Color3.fromRGB(140,10,10)
titleLabel.Text = "⚡ Power Hub V3.6 Clean Compact +13%"
titleLabel.TextColor3 = Color3.new(1,1,1)
titleLabel.Font = Enum.Font.SourceSansBold
titleLabel.TextSize = 16
titleLabel.Parent = menuFrame

--------------------------------------------------------------------
-- NOVO E MELHORADO SISTEMA DE DRAGGING (PC + MOBILE)
--------------------------------------------------------------------
local draggingMenu, dragStartMenu, startPosMenu
local function startDrag(input)
	-- Verifica se a entrada é MouseButton1 ou Touch
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		draggingMenu = true
		dragStartMenu = input.Position
		startPosMenu = menuFrame.Position
		
		-- Captura a entrada para garantir que o movimento não seja interrompido (melhora o Mobile/Touch)
		if input.UserInputType == Enum.UserInputType.Touch then
			UIS:CaptureInput(input)
		end
	end
end

local function updateDrag(input)
	if draggingMenu and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
		local delta = input.Position - dragStartMenu
		local newX = startPosMenu.X.Offset + delta.X
		local newY = startPosMenu.Y.Offset + delta.Y
		
		-- Limitar o movimento para evitar que o menu saia da tela
		local screenW = Workspace.CurrentCamera.ViewportSize.X
		local screenH = Workspace.CurrentCamera.ViewportSize.Y
		
		-- Clamping: Mantém o menu dentro dos limites (margem de 50 pixels)
		newX = math.clamp(newX, -MENU_WIDTH + 50, screenW - 50)
		newY = math.clamp(newY, -MENU_HEIGHT + 50, screenH - 50)
		
		menuFrame.Position = UDim2.new(0, newX, 0, newY)
	end
end

local function endDrag(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		if draggingMenu then
			draggingMenu = false
			-- Libera a entrada capturada
			if input.UserInputType == Enum.UserInputType.Touch then
				UIS:ReleaseInput(input)
			end
		end
	end
end

-- Conectar o Frame do Menu (arrastar pelo corpo do menu)
menuFrame.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		startDrag(i)
	end
end)

-- Conectar os eventos globais de Input Changed e Ended
UIS.InputChanged:Connect(updateDrag)
UIS.InputEnded:Connect(endDrag)

--------------------------------------------------------------------
-- 🌀 Scrollable container for buttons (inside menuFrame)
local scrollButtons = Instance.new("ScrollingFrame")
scrollButtons.Name = "ButtonsContainer"
scrollButtons.Parent = menuFrame
scrollButtons.Size = UDim2.new(1, 0, 1, 0)
scrollButtons.Position = UDim2.new(0, 0, 0, 0)
scrollButtons.BackgroundTransparency = 1
scrollButtons.BorderSizePixel = 0
scrollButtons.ScrollBarThickness = 6
scrollButtons.ScrollingDirection = Enum.ScrollingDirection.XY
scrollButtons.AutomaticCanvasSize = Enum.AutomaticSize.XY
scrollButtons.CanvasSize = UDim2.new(0, 0, 0, 0)
scrollButtons.ClipsDescendants = true

local gridLayout = Instance.new("UIGridLayout")
gridLayout.Parent = scrollButtons
gridLayout.CellPadding = UDim2.new(0, 8, 0, 8)
gridLayout.CellSize = UDim2.new(0, BTN_W, 0, BTN_H)
gridLayout.SortOrder = Enum.SortOrder.LayoutOrder

-- helper to create buttons (keeps same look + scaled sizes)
local function createButton(text, width)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(0, width or BTN_W, 0, BTN_H)
	b.Text = text
	b.BackgroundColor3 = BUTTON_RED
	b.TextColor3 = Color3.new(1,1,1)
	b.Font = Enum.Font.SourceSansBold
	b.TextSize = math.clamp(14 * 1.13, 10, 20) -- scaled text a bit
	Instance.new("UICorner", b).CornerRadius = UDim.new(0,8)
	b.Parent = scrollButtons
	return b
end

--------------------------------------------------------------------
-- Status label positioned inside the fixed menu (not part of scroll)
local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(0, MENU_WIDTH - 20, 0, 22)
statusLabel.Position = UDim2.new(0, 10, 1, -36)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = "Auto Farm Desligado"
statusLabel.TextColor3 = Color3.fromRGB(255,220,220)
statusLabel.Font = Enum.Font.SourceSansBold
statusLabel.TextSize = 14
statusLabel.Parent = menuFrame

--------------------------------------------------------------------
-- Create the buttons (all parented to scrollButtons)
local autoFarmBtn = createButton("AutoFarm [✖]")
local tpToolBtn = createButton("TP Tool")
local jumpBtn = createButton("Pulo: 26")
local boomboxBtn = createButton("Boombox")
local toggleMusicBtn = Instance.new("TextButton") -- create separately to make small size
toggleMusicBtn.Size = UDim2.new(0,36,0,36)
toggleMusicBtn.Text = "🔇 OFF"
toggleMusicBtn.BackgroundColor3 = BUTTON_RED
toggleMusicBtn.TextColor3 = Color3.new(1,1,1)
toggleMusicBtn.Font = Enum.Font.SourceSansBold
toggleMusicBtn.TextSize = 14
Instance.new("UICorner", toggleMusicBtn).CornerRadius = UDim.new(0,8)
toggleMusicBtn.Parent = scrollButtons

local speedBtn = createButton("Velocidade: 16")
local tpSafeBtn = createButton("TP Safe Zone")
local modeAllBtn = createButton("All")
local modeAnimalBtn = createButton("Animal Simulator")
local aimBtn = createButton("Aim Sist Apenas Bots [✖]")
local precisionBtn = createButton("Mais Precisão [✖]")
local espBtn = createButton("ESP [✖]")

-- Shift Lock Real button
local shiftLockRealBtn = createButton("Shift Lock Real [✖]")

-- HyperLaserGun Tool button
local hyperBtn = createButton("HyperLaserGun Tool [⚡]")

-- View Player - fica na aba All
local viewPlayerBtn = createButton("View Player") 

-- Initially hide the "other" controls like in original logic
local otherButtons = {tpToolBtn, jumpBtn, boomboxBtn, toggleMusicBtn, speedBtn, tpSafeBtn, aimBtn, precisionBtn, espBtn, shiftLockRealBtn, hyperBtn, viewPlayerBtn}
for _, b in ipairs(otherButtons) do b.Visible = false end
autoFarmBtn.Visible = false

local allModeActive = false
modeAllBtn.MouseButton1Click:Connect(function()
	for _, b in ipairs(otherButtons) do b.Visible = true end
	autoFarmBtn.Visible = false
	modeAllBtn.BackgroundColor3 = BUTTON_RED
	modeAnimalBtn.BackgroundColor3 = BUTTON_RED
	allModeActive = true
end)
modeAnimalBtn.MouseButton1Click:Connect(function()
	for _, b in ipairs(otherButtons) do b.Visible = false end
	autoFarmBtn.Visible = true
	modeAnimalBtn.BackgroundColor3 = BUTTON_RED
	modeAllBtn.BackgroundColor3 = BUTTON_RED
	allModeActive = false
end)

--------------------------------------------------------------------
-- AutoFarm (reused logic)
local function findCoinRemote()
	for _, v in pairs(ReplicatedStorage:GetDescendants()) do
		if v:IsA("RemoteEvent") and v.Name:lower():find("coin") then return v end
	end
	for _, v in pairs(Workspace:GetDescendants()) do
		if v:IsA("RemoteEvent") and v.Name:lower():find("coin") then return v end
	end
	return nil
end

autoFarmBtn.MouseButton1Click:Connect(function()
	farming = not farming
	if farmLoop then farmLoop:Disconnect() farmLoop = nil end
	if farming then
		autoFarmBtn.Text = "AutoFarm [✔]"
		autoFarmBtn.BackgroundColor3 = BUTTON_RED
		local remote = findCoinRemote()
		if remote then
			statusLabel.Text = "Auto Farm Ligado"
			statusLabel.TextColor3 = Color3.fromRGB(0,255,0)
			farmLoop = RunService.RenderStepped:Connect(function()
				pcall(function() remote:FireServer() end)
			end)
		else
			statusLabel.Text = "⚠️ Nenhum Remote encontrado"
			statusLabel.TextColor3 = Color3.fromRGB(255,255,0)
			farming = false
			autoFarmBtn.Text = "AutoFarm [✖]"
			autoFarmBtn.BackgroundColor3 = BUTTON_RED
		end
	else
		statusLabel.Text = "Auto Farm Desligado"
		statusLabel.TextColor3 = Color3.fromRGB(255,220,220)
		autoFarmBtn.Text = "AutoFarm [✖]"
		autoFarmBtn.BackgroundColor3 = BUTTON_RED
	end
end)

--------------------------------------------------------------------
-- Boombox (reused) - now updates toggleMusicBtn with emoji ON/OFF
local function updateToggleMusicBtn()
	if currentSound and musicPlaying then
		toggleMusicBtn.Text = "🎵 ON"
	else
		toggleMusicBtn.Text = "🔇 OFF"
	end
	toggleMusicBtn.BackgroundColor3 = BUTTON_RED
end

boomboxBtn.MouseButton1Click:Connect(function()
	if screenGui:FindFirstChild("BoomboxInput") then return end
	local input = Instance.new("TextBox")
	input.Name = "BoomboxInput"
	input.Size = UDim2.new(0,200,0,32)
	input.Position = UDim2.new(0.5,-100,0.5,-18)
	input.BackgroundColor3 = Color3.fromRGB(40,40,40)
	input.Text = "Digite o código da música"
	input.TextColor3 = Color3.new(1,1,1)
	input.Font = Enum.Font.SourceSans
	input.TextSize = 14
	input.ClearTextOnFocus = true
	input.Parent = screenGui
	Instance.new("UICorner", input).CornerRadius = UDim.new(0,8)
	input.FocusLost:Connect(function(enter)
		if enter then
			local id = tonumber(input.Text)
			if id then
				if currentSound then
					pcall(function() currentSound:Stop() end)
					pcall(function() currentSound:Destroy() end)
					currentSound = nil
					musicPlaying = false
				end
				currentSound = Instance.new("Sound")
				currentSound.SoundId = "rbxassetid://"..id
				currentSound.Volume = 1
				currentSound.Looped = true
				currentSound.Parent = Workspace
				local ok, _ = pcall(function() currentSound:Play() end)
				musicPlaying = ok or true
				updateToggleMusicBtn()
				statusLabel.Text = "Música iniciada"
				statusLabel.TextColor3 = Color3.fromRGB(200,200,255)
				task.delay(1.5, function()
					statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
					statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
				end)
			else
				statusLabel.Text = "ID inválido"
				statusLabel.TextColor3 = Color3.fromRGB(255,100,100)
				task.delay(2, function()
					statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
					statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
				end)
			end
		end
		input:Destroy()
	end)
end)

-- Toggle (pause/resume) for Boombox: shows 🎵 ON when playing, 🔇 OFF when paused/none
toggleMusicBtn.MouseButton1Click:Connect(function()
	if currentSound then
		if musicPlaying then
			pcall(function() currentSound:Pause() end)
			musicPlaying = false
		else
			pcall(function() currentSound:Resume() end)
			musicPlaying = true
		end
		updateToggleMusicBtn()
	else
		-- no sound: inform user
		statusLabel.Text = "Nenhuma música tocando"
		statusLabel.TextColor3 = Color3.fromRGB(255,200,0)
		task.delay(2, function()
			statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
			statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
		end)
	end
end)

-- initial state for toggle
updateToggleMusicBtn()

--------------------------------------------------------------------
-- TP Tool (no-duplicate)
local function giveTPTool()
	if player.Backpack then
		for _, tool in pairs(player.Backpack:GetChildren()) do
			if tool.Name == "TP Tool" then tool:Destroy() end
		end
	end
	if player.Character then
		for _, tool in pairs(player.Character:GetChildren()) do
			if tool:IsA("Tool") and tool.Name == "TP Tool" then tool:Destroy() end
		end
	end
	local tool = Instance.new("Tool")
	tool.Name, tool.RequiresHandle, tool.CanBeDropped = "TP Tool", false, false
	tool.Parent = player.Backpack
	local mouse = player:GetMouse()
	tool.Activated:Connect(function()
		if mouse and mouse.Hit then
			local ok, pos = pcall(function() return mouse.Hit.Position end)
			if ok and pos then
				local tpPos = pos + Vector3.new(0,3,0)
				if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
					pcall(function() player.Character:MoveTo(tpPos) end)
				end
			end
		end
	end)
end

tpToolBtn.MouseButton1Click:Connect(function()
	giveTPTool()
	statusLabel.Text = "TP Tool dada"
	statusLabel.TextColor3 = Color3.fromRGB(200,200,255)
	task.delay(2, function()
		statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
		statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
	end)
end)

--------------------------------------------------------------------
-- TP Safe Zone
tpSafeBtn.MouseButton1Click:Connect(function()
	if not safePos then
		if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
			safePos = player.Character.HumanoidRootPart.Position
			statusLabel.Text = "Safe Zone definida!"
			statusLabel.TextColor3 = Color3.fromRGB(0,200,255)
			task.delay(2, function()
				statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
				statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
			end)
		end
	else
		if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
			pcall(function() player.Character:MoveTo(safePos) end)
		end
	end
end)

--------------------------------------------------------------------
-- Speed / Jump
speedBtn.MouseButton1Click:Connect(function()
	walkSpeed = walkSpeed + 8 -- scaled increments for mobile feel
	if walkSpeed > 135 then walkSpeed = 16 end
	speedBtn.Text = "Velocidade: " .. walkSpeed
	if humanoid then pcall(function() humanoid.WalkSpeed = walkSpeed end) end
end)
jumpBtn.MouseButton1Click:Connect(function()
	jumpPower = jumpPower + 10 -- scaled increments
	if jumpPower > 215 then jumpPower = 26 end
	jumpBtn.Text = "Pulo: " .. jumpPower
	if humanoid then
		pcall(function()
			humanoid.UseJumpPower = true
			humanoid.JumpPower = jumpPower
		end)
	end
end)

--------------------------------------------------------------------
-- Respawn handling (reaplica TP tool e shift lock)
player.CharacterAdded:Connect(function(c)
	char = c
	humanoid = c:WaitForChild("Humanoid")
	task.wait(0.5)
	pcall(function()
		humanoid.WalkSpeed = walkSpeed
		humanoid.UseJumpPower = true
		humanoid.JumpPower = jumpPower
	end)
	task.delay(0.2, function()
		if not player.Backpack:FindFirstChild("TP Tool") then giveTPTool() end
	end)
	-- reapply shift lock if it was enabled
	if shiftLocked then
		task.delay(0.1, function()
			pcall(function()
				if Workspace.CurrentCamera and char and char:FindFirstChild("Humanoid") then
					Workspace.CurrentCamera.CameraSubject = char:FindFirstChild("Humanoid")
				end
				pcall(function() UIS.MouseIconEnabled = false end)
			end)
		end)
	end
	-- if viewing another player, stop view (safety)
	if viewingPlayer then
		-- restore camera on respawn to avoid camera stuck
		if backViewBtn then backViewBtn:Destroy(); backViewBtn = nil end
		viewingPlayer = nil
		if viewLoop then viewLoop:Disconnect(); viewLoop = nil end
		viewModeActive = false
		pcall(function()
			if Workspace.CurrentCamera then
				if originalCameraType then Workspace.CurrentCamera.CameraType = originalCameraType end
				if originalCameraSubject then Workspace.CurrentCamera.CameraSubject = originalCameraSubject end
			end
		end)
	end
end)

if player and player.Backpack and not player.Backpack:FindFirstChild("TP Tool") then
	giveTPTool()
end

if char and char:FindFirstChild("HumanoidRootPart") then
	safePos = char.HumanoidRootPart.Position
end

--------------------------------------------------------------------
print("[✅ Power Hub V3.6 Clean Compact +13% carregado]")

--------------------------------------------------------------------
-- Aim Sist Apenas Bots (215 studs) - lógica reutilizada
local function getClosestNPC()
	local closest, minDist = nil, 215
	if not (char and char:FindFirstChild("HumanoidRootPart")) then return nil end
	for _, obj in pairs(Workspace:GetDescendants()) do
		if obj:IsA("Model") and obj ~= char and obj:FindFirstChild("Humanoid") and obj:FindFirstChild("HumanoidRootPart") then
			local maybePlayer = Players:GetPlayerFromCharacter(obj)
			if maybePlayer == nil then
				local root = obj:FindFirstChild("HumanoidRootPart")
				local dist = (root.Position - char.HumanoidRootPart.Position).Magnitude
				if dist < minDist then minDist = dist; closest = obj end
			end
		end
	end
	return closest
end

aimBtn.MouseButton1Click:Connect(function()
	aimAssist = not aimAssist
	if aimAssist then
		aimBtn.Text = "Aim Sist Apenas Bots [✔]"
		aimBtn.BackgroundColor3 = BUTTON_RED
		if aimLoop then aimLoop:Disconnect() aimLoop = nil end
		aimLoop = RunService.RenderStepped:Connect(function()
			if not (char and char.Parent and char:FindFirstChild("HumanoidRootPart")) then return end
			local target = getClosestNPC()
			if target and target:FindFirstChild("HumanoidRootPart") then
				local hrp = char:FindFirstChild("HumanoidRootPart")
				local cam = Workspace.CurrentCamera
				local targetPos = target.HumanoidRootPart.Position
				local lookVec = Vector3.new(targetPos.X, hrp.Position.Y, targetPos.Z)
				hrp.CFrame = CFrame.new(hrp.Position, lookVec)
				if cam and cam:IsA("Camera") then
					cam.CFrame = cam.CFrame:Lerp(CFrame.new(cam.CFrame.Position, targetPos), 0.15)
				end
			end
		end)
	else
		aimBtn.Text = "Aim Sist Apenas Bots [✖]"
		aimBtn.BackgroundColor3 = BUTTON_RED
		if aimLoop then aimLoop:Disconnect() aimLoop = nil end
	end
end)

--------------------------------------------------------------------
-- Mais Precisão overlay (transparent center + dot)
local precisionOverlay = Instance.new("Frame")
precisionOverlay.Name = "PrecisionOverlay"
precisionOverlay.Size = UDim2.new(1,0,1,0)
precisionOverlay.Position = UDim2.new(0,0,0,0)
precisionOverlay.BackgroundTransparency = 1
precisionOverlay.Visible = false
precisionOverlay.Parent = screenGui
precisionOverlay.ZIndex = 50

local precisionContainer = Instance.new("Frame")
precisionContainer.Size = UDim2.new(0, 185, 0, 185)
precisionContainer.Position = UDim2.new(0.5, -92, 0.5, -92)
precisionContainer.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
precisionContainer.BackgroundTransparency = 0.45
precisionContainer.BorderSizePixel = 0
precisionContainer.Parent = precisionOverlay
precisionContainer.ZIndex = 51
Instance.new("UICorner", precisionContainer).CornerRadius = UDim.new(1, 0)

local innerCircle = Instance.new("Frame")
innerCircle.Size = UDim2.new(0, 100, 0, 100)
innerCircle.Position = UDim2.new(0.5, -50, 0.5, -50)
innerCircle.BackgroundColor3 = Color3.fromRGB(140, 0, 200)
innerCircle.BackgroundTransparency = 0.5
innerCircle.BorderSizePixel = 0
innerCircle.Parent = precisionContainer
innerCircle.ZIndex = 52
Instance.new("UICorner", innerCircle).CornerRadius = UDim.new(1, 0)

local centerDot = Instance.new("Frame")
centerDot.Size = UDim2.new(0, 12, 0, 12)
centerDot.Position = UDim2.new(0.5, -6, 0.5, -6)
centerDot.BackgroundColor3 = Color3.fromRGB(255,255,255)
centerDot.BackgroundTransparency = 0.25
centerDot.BorderSizePixel = 0
centerDot.Parent = precisionContainer
centerDot.ZIndex = 53
Instance.new("UICorner", centerDot).CornerRadius = UDim.new(1,0)

local function togglePrecisionOverlay()
	precisionOn = not precisionOn
	precisionOverlay.Visible = precisionOn
	if precisionOn then
		precisionBtn.Text = "Mais Precisão [✔]"
		precisionBtn.BackgroundColor3 = BUTTON_RED
	else
		precisionBtn.Text = "Mais Precisão [✖]"
		precisionBtn.BackgroundColor3 = BUTTON_RED
	end
end
precisionBtn.MouseButton1Click:Connect(togglePrecisionOverlay)
precisionOverlay.Visible = false

--------------------------------------------------------------------
-- ESP (Highlight) + per-character circular outline (BillboardGui + UIStroke)
local highlights = {}
local rings = {} -- map character -> BillboardGui ring
local hostilePlayers = {}
local friendlyPlayers = {}

local function determineColorForCharacter(ch)
	local possiblePlayer = Players:GetPlayerFromCharacter(ch)
	if not possiblePlayer then
		return Color3.fromRGB(255,165,0), Color3.fromRGB(200,120,0) -- NPC orange
	end
	if hostilePlayers[possiblePlayer] then
		return Color3.fromRGB(255,0,0), Color3.fromRGB(160,20,20)
	end
	local ok, localTeam = pcall(function() return player.Team end)
	local ok2, otherTeam = pcall(function() return possiblePlayer.Team end)
	if ok and ok2 and localTeam and otherTeam and localTeam ~= otherTeam then
		return Color3.fromRGB(255,0,0), Color3.fromRGB(160,20,20)
	elseif ok and ok2 and localTeam and otherTeam and localTeam == otherTeam then
		return Color3.fromRGB(0,130,255), Color3.fromRGB(20,90,160)
	end
	for _, obj in pairs(ch:GetDescendants()) do
		if obj:IsA("Tool") then
			local n = obj.Name:lower()
			if n:find("sword") or n:find("gun") or n:find("blade") or n:find("knife") or n:find("pistol") or n:find("rifle") or n:find("weapon") then
				return Color3.fromRGB(255,0,0), Color3.fromRGB(160,20,20)
			end
		end
	end
	if friendlyPlayers[possiblePlayer] then
		return Color3.fromRGB(0,130,255), Color3.fromRGB(20,90,160)
	end
	return Color3.fromRGB(255,255,255), Color3.fromRGB(200,200,200)
end

local function ensureHighlight(ch)
	if not ch or not ch.Parent then return end
	if highlights[ch] and highlights[ch].Parent then
		local f,o = determineColorForCharacter(ch)
		highlights[ch].FillColor = f
		highlights[ch].OutlineColor = o
		highlights[ch].Enabled = espOn
	else
		local hl = Instance.new("Highlight")
		hl.Name = "PH_ESP"
		hl.Adornee = ch
		local f,o = determineColorForCharacter(ch)
		hl.FillColor = f
		hl.OutlineColor = o
		hl.FillTransparency = 0.65
		hl.OutlineTransparency = 0.4
		hl.Enabled = espOn
		hl.Parent = ch
		highlights[ch] = hl
	end
	-- create or update ring too
	if espOn then
		if rings[ch] == nil then
			local hrp = ch:FindFirstChild("HumanoidRootPart")
			if hrp then
				local bg = Instance.new("BillboardGui")
				bg.Name = "PH_Ring"
				bg.Adornee = hrp
				bg.AlwaysOnTop = true
				bg.Size = UDim2.new(0, 160, 0, 160)
				bg.StudsOffset = Vector3.new(0, 2.4, 0)
				bg.MaxDistance = 2000
				bg.Parent = ch

				local ringFrame = Instance.new("Frame")
				ringFrame.Name = "RingFrame"
				ringFrame.Size = UDim2.new(1, 0, 1, 0)
				ringFrame.Position = UDim2.new(0, 0, 0, 0)
				ringFrame.BackgroundTransparency = 1
				ringFrame.BorderSizePixel = 0
				ringFrame.Parent = bg

				local circ = Instance.new("Frame")
				circ.Name = "Circ"
				circ.Size = UDim2.new(0, 160, 0, 160)
				circ.Position = UDim2.new(0, 0, 0, 0)
				circ.AnchorPoint = Vector2.new(0,0)
				circ.BackgroundColor3 = Color3.fromRGB(255,255,255)
				circ.BackgroundTransparency = 1
				circ.BorderSizePixel = 0
				circ.Parent = ringFrame

				local uic = Instance.new("UICorner")
				uic.CornerRadius = UDim.new(1, 0)
				uic.Parent = circ

				local stroke = Instance.new("UIStroke")
				stroke.Name = "RingStroke"
				stroke.LineJoinMode = Enum.LineJoinMode.Round
				stroke.Thickness = 2
				stroke.Color = Color3.fromRGB(255,255,255)
				stroke.Transparency = 0.7
				stroke.Parent = circ

				rings[ch] = bg
			end
		else
			local bg = rings[ch]
			if bg and bg:FindFirstChild("RingFrame") and bg.RingFrame:FindFirstChild("Circ") and bg.RingFrame.Circ:FindFirstChild("RingStroke") then
				local stroke = bg.RingFrame.Circ.RingStroke
				stroke.Transparency = 0.7
			end
		end
	end
end

local function removeHighlight(ch)
	if highlights[ch] then
		pcall(function() highlights[ch]:Destroy() end)
		highlights[ch] = nil
	end
	if rings[ch] then
		pcall(function() rings[ch]:Destroy() end)
		rings[ch] = nil
	end
end

local function scanAndApplyESP()
	for _, obj in pairs(Workspace:GetDescendants()) do
		if obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj:FindFirstChild("HumanoidRootPart") then
			ensureHighlight(obj)
		end
	end
	for ch,hl in pairs(highlights) do
		if not ch.Parent then removeHighlight(ch) end
	end
end

local espUpdateConn = nil
local function startESPLoop()
	if espUpdateConn then espUpdateConn:Disconnect() espUpdateConn = nil end
	espUpdateConn = RunService.Heartbeat:Connect(function()
		for ch,hl in pairs(highlights) do
			if ch and ch.Parent then
				local f,o = determineColorForCharacter(ch)
				hl.FillColor = f
				hl.OutlineColor = o
				hl.Enabled = espOn
				if espOn then
					if rings[ch] == nil then
						pcall(function() ensureHighlight(ch) end)
					else
						local bg = rings[ch]
						if bg and bg.Adornee and bg.Adornee:IsA("BasePart") then
							local cam = Workspace.CurrentCamera
							local hrp = ch:FindFirstChild("HumanoidRootPart")
							if cam and hrp then
								local dist = (cam.CFrame.Position - hrp.Position).Magnitude
								local newSize = math.clamp(160 * (1 - (dist/1000)), 40, 220)
								bg.Size = UDim2.new(0, newSize, 0, newSize)
								bg.Enabled = true
							end
						end
					end
				else
					if rings[ch] and rings[ch].Parent then
						rings[ch].Enabled = false
					end
				end
			else
				removeHighlight(ch)
			end
		end
	end)
end

local function stopESPLoop()
	if espUpdateConn then espUpdateConn:Disconnect() espUpdateConn = nil end
	for ch,hl in pairs(highlights) do removeHighlight(ch) end
end

-- watch humanoids for creator tags to determine hostiles
local function watchHumanoidForCreator(hum)
	if not hum then return end
	hum.ChildAdded:Connect(function(child)
		if child:IsA("ObjectValue") and (child.Name:lower():find("creator") or child.Name:lower():find("killer")) then
			local val = child.Value
			if val and typeof(val) == "Instance" then
				local pl = Players:GetPlayerFromCharacter(val)
				if pl then
					hostilePlayers[pl] = true
					task.delay(20, function() hostilePlayers[pl] = nil end)
				end
			end
		end
	end)
end

if humanoid then
	watchHumanoidForCreator(humanoid)
	humanoid.ChildAdded:Connect(function(child)
		if child:IsA("ObjectValue") and (child.Name:lower():find("creator") or child.Name:lower():find("killer")) then
			local val = child.Value
			if val and typeof(val) == "Instance" then
				local pl = Players:GetPlayerFromCharacter(val)
				if pl then
					hostilePlayers[pl] = true
					task.delay(20, function() hostilePlayers[pl] = nil end)
				end
			end
		end
	end)
end

for _, m in pairs(Workspace:GetDescendants()) do
	if m:IsA("Model") and m:FindFirstChild("Humanoid") then
		pcall(function() watchHumanoidForCreator(m:FindFirstChild("Humanoid")) end)
	end
end
Workspace.DescendantAdded:Connect(function(d)
	if d:IsA("Model") and d:FindFirstChild("Humanoid") then
		pcall(function() watchHumanoidForCreator(d:FindFirstChild("Humanoid")) end)
		if espOn then ensureHighlight(d) end
	end
end)

Players.PlayerAdded:Connect(function(pl)
	task.delay(1, function()
		for _, ch in pairs(Workspace:GetDescendants()) do
			if ch:IsA("Model") and Players:GetPlayerFromCharacter(ch) == pl then
				if espOn then ensureHighlight(ch) end
			end
		end
	end)
end)

--------------------------------------------------------------------
-- Target Box 2D
local targetBox = Instance.new("Frame")
targetBox.Name = "PH_TargetBox"
targetBox.Size = UDim2.new(0, 60, 0, 60)
targetBox.AnchorPoint = Vector2.new(0.5, 0.5)
targetBox.Position = UDim2.new(0.5, 0.5)
targetBox.BackgroundColor3 = Color3.fromRGB(255,0,0)
targetBox.BackgroundTransparency = 0
targetBox.BorderSizePixel = 0
targetBox.ZIndex = 1000
targetBox.Visible = false
targetBox.Parent = screenGui

local function getBestTargetForBox()
	local cam = Workspace.CurrentCamera
	if not cam then return nil end
	local cx, cy = cam.ViewportSize.X/2, cam.ViewportSize.Y/2
	local best, bestDist = nil, math.huge
	for _, obj in pairs(Workspace:GetDescendants()) do
		if obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj:FindFirstChild("HumanoidRootPart") and obj ~= char then
			local ok, screenPos = pcall(function() return cam:WorldToViewportPoint(obj.HumanoidRootPart.Position) end)
			if ok and screenPos then
				local sx, sy, sz = screenPos.X, screenPos.Y, screenPos.Z
				if sz > 0 then
					local dist2D = (Vector2.new(sx, sy) - Vector2.new(cx, cy)).Magnitude
					if dist2D < bestDist and dist2D < 600 then
						bestDist = dist2D
						best = obj
					end
				end
			end
		end
	end
	return best
end

local targetConn = nil
function startTargetBoxLoop()
	if targetConn then targetConn:Disconnect() targetConn = nil end
	targetConn = RunService.RenderStepped:Connect(function()
		if not espOn then
			if targetBox.Visible then targetBox.Visible = false end
			return
		end
		local cam = Workspace.CurrentCamera
		if not cam then targetBox.Visible = false return end
		local best = getBestTargetForBox()
		if not best or not best.Parent or not best:FindFirstChild("HumanoidRootPart") then targetBox.Visible = false return end
		local ok, screenPos = pcall(function() return cam:WorldToViewportPoint(best.HumanoidRootPart.Position) end)
		if not ok or not screenPos then targetBox.Visible = false return end
		local sx, sy, sz = screenPos.X, screenPos.Y, screenPos.Z
		if sz <= 0 then targetBox.Visible = false return end
		targetBox.Position = UDim2.new(0, sx, 0, sy)
		targetBox.AnchorPoint = Vector2.new(0.5, 0.5)
		targetBox.Visible = true
	end)
end
function stopTargetBoxLoop()
	if targetConn then targetConn:Disconnect() targetConn = nil end
	targetBox.Visible = false
end

if espOn then startTargetBoxLoop() else stopTargetBoxLoop() end

--------------------------------------------------------------------
-- ESP toggle: start/stop rings/target box
espBtn.MouseButton1Click:Connect(function()
	espOn = not espOn
	if espOn then
		espBtn.Text = "ESP [✔]"
		espBtn.BackgroundColor3 = BUTTON_RED
		scanAndApplyESP()
		startESPLoop()
		startTargetBoxLoop()
	else
		espBtn.Text = "ESP [✖]"
		espBtn.BackgroundColor3 = BUTTON_RED
		stopESPLoop()
		stopTargetBoxLoop()
	end
end)

--------------------------------------------------------------------
-- Shift Lock Real (Scriptable, câmera fixa) - behavior preserved
local originalCameraTypeForShift = nil
local originalCameraSubjectForShift = nil
local originalWalkSpeed = nil
local shiftConn = nil
local shiftCameraCFrame = nil
local allowMoveThreshold = 0.12
local lastWalkSpeedRestore = 0

local SHIFT_OFFSET_RIGHT = 1.2
local SHIFT_OFFSET_BACK = 2.6
local SHIFT_HEIGHT = 1.6

local function getShiftCameraCFrame(hrp)
	local rootCFrame = hrp.CFrame
	local lookVector = rootCFrame.LookVector
	local rightVector = rootCFrame.RightVector
	local targetPos = hrp.Position - (lookVector * SHIFT_OFFSET_BACK) + (rightVector * SHIFT_OFFSET_RIGHT) + Vector3.new(0, SHIFT_HEIGHT, 0)
	local lookAt = hrp.Position + Vector3.new(0, 1.2, 0)
	return CFrame.new(targetPos, lookAt)
end

local function startShiftLoop()
	if shiftConn then shiftConn:Disconnect() shiftConn = nil end
	shiftConn = RunService.RenderStepped:Connect(function()
		if not shiftLocked then return end
		if humanoid then
			local md = humanoid.MoveDirection
			local mag = (md and md.Magnitude) or 0
			if mag > allowMoveThreshold then
				if humanoid.WalkSpeed ~= (originalWalkSpeed or walkSpeed) then
					pcall(function() humanoid.WalkSpeed = (originalWalkSpeed or walkSpeed) end)
					lastWalkSpeedRestore = tick()
				end
			else
				if tick() - lastWalkSpeedRestore > 0.03 then
					if humanoid.WalkSpeed ~= 0 then
						pcall(function() humanoid.WalkSpeed = 0 end)
					end
				end
			end
		end
		if shiftCameraCFrame and Workspace.CurrentCamera then
			pcall(function() Workspace.CurrentCamera.CFrame = shiftCameraCFrame end)
		end
	end)
end

local function stopShiftLoop()
	if shiftConn then shiftConn:Disconnect() shiftConn = nil end
end

local function enableShiftLock()
	if shiftLocked then return end
	if not char or not char.Parent then return end
	shiftLocked = true
	shiftLockRealBtn.Text = "Shift Lock Real [✔]"
	shiftLockRealBtn.BackgroundColor3 = BUTTON_RED

	originalMouseBehavior = UIS.MouseBehavior
	originalMouseIconEnabled = UIS.MouseIconEnabled
	if Workspace.CurrentCamera then
		originalCameraType = Workspace.CurrentCamera.CameraType
		originalCameraSubject = Workspace.CurrentCamera.CameraSubject
	end
	originalWalkSpeed = humanoid and humanoid.WalkSpeed or walkSpeed

	pcall(function() UIS.MouseIconEnabled = false end)
	pcall(function() UIS.MouseBehavior = Enum.MouseBehavior.LockCenter end)

	pcall(function()
		if Workspace.CurrentCamera and humanoid and char:FindFirstChild("HumanoidRootPart") then
			Workspace.CurrentCamera.CameraType = Enum.CameraType.Scriptable
			Workspace.CurrentCamera.CameraSubject = humanoid
			shiftCameraCFrame = getShiftCameraCFrame(char:FindFirstChild("HumanoidRootPart"))
			Workspace.CurrentCamera.CFrame = shiftCameraCFrame
		end
	end)

	pcall(function()
		if humanoid then humanoid.WalkSpeed = 0 end
	end)

	startShiftLoop()
end

local function disableShiftLock()
	if not shiftLocked then return end
	shiftLocked = false
	shiftLockRealBtn.Text = "Shift Lock Real [✖]"
	shiftLockRealBtn.BackgroundColor3 = BUTTON_RED

	pcall(function()
		if humanoid and originalWalkSpeed then humanoid.WalkSpeed = originalWalkSpeed end
	end)

	pcall(function() UIS.MouseBehavior = originalMouseBehavior or Enum.MouseBehavior.Default end)
	pcall(function() UIS.MouseIconEnabled = originalMouseIconEnabled ~= nil and originalMouseIconEnabled or true end)

	stopShiftLoop()

	pcall(function()
		if Workspace.CurrentCamera then
			if originalCameraType then Workspace.CurrentCamera.CameraType = originalCameraType end
			if originalCameraSubject then Workspace.CurrentCamera.CameraSubject = originalCameraSubject end
			shiftCameraCFrame = nil
		end
	end)
end

shiftLockRealBtn.MouseButton1Click:Connect(function()
	if shiftLocked then disableShiftLock() else enableShiftLock() end
end)

--------------------------------------------------------------------
-- HyperLaserGun Tool implementation (try load assets, else fallback)
local HYPER_ASSET_IDS = {
	"rbxassetid://130113146",
	"rbxassetid://130113157",
}

local function giveHyperLaser()
	-- prevent duplicates
	if player.Backpack then
		for _, t in pairs(player.Backpack:GetChildren()) do
			if t.Name == "HyperLaserGun" then
				statusLabel.Text = "Já possui HyperLaserGun"
				statusLabel.TextColor3 = Color3.fromRGB(200,200,255)
				task.delay(1.5, function()
					statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
					statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
				end)
				return
			end
		end
	end
	for _, id in ipairs(HYPER_ASSET_IDS) do
		local ok, model = pcall(function() return game:GetObjects(id)[1] end)
		if ok and model then
			local tool = nil
			if model:IsA("Tool") then
				tool = model
			else
				tool = model:FindFirstChildOfClass("Tool") or model:FindFirstChild("HyperLaserGun") or model:FindFirstChildWhichIsA("Tool")
			end
			if tool then
				tool.Name = "HyperLaserGun"
				tool.Parent = player.Backpack
				statusLabel.Text = "HyperLaserGun dada!"
				statusLabel.TextColor3 = Color3.fromRGB(200,200,255)
				task.delay(1.5, function()
					statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
					statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
				end)
				if model ~= tool then
					pcall(function() model:Destroy() end)
				end
				return
			else
				pcall(function() model.Parent = player.Backpack end)
				statusLabel.Text = "HyperLaserGun dada!"
				statusLabel.TextColor3 = Color3.fromRGB(200,200,255)
				task.delay(1.5, function()
					statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
					statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
				end)
				return
			end
		end
	end

	-- fallback simple tool (non-damaging visual)
	local fallback = Instance.new("Tool")
	fallback.Name = "HyperLaserGun"
	fallback.RequiresHandle = true
	fallback.CanBeDropped = true
	local handle = Instance.new("Part")
	handle.Name = "Handle"
	handle.Size = Vector3.new(1,1,3)
	handle.Color = Color3.fromRGB(60,130,255)
	handle.Material = Enum.Material.Neon
	handle.CanCollide = false
	handle.Parent = fallback
	fallback.Parent = player.Backpack

	local ok, ls = pcall(function()
		local s = Instance.new("LocalScript")
		s.Name = "PH_HyperFallbackScript"
		s.Source = [[
local tool = script.Parent
tool.Activated:Connect(function()
	local h = tool:FindFirstChild("Handle")
	if h then
		local orig = h.Transparency
		h.Transparency = math.clamp(orig - 0.2, 0, 1)
		wait(0.06)
		h.Transparency = orig
	end
end)
]]
		s.Parent = fallback
		return s
	end)
	pcall(function() if ls then ls.Parent = fallback end end)

	statusLabel.Text = "HyperLaserGun (fallback) dada!"
	statusLabel.TextColor3 = Color3.fromRGB(200,200,255)
	task.delay(1.5, function()
		statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
		statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
	end)
end

hyperBtn.MouseButton1Click:Connect(function()
	pcall(function() giveHyperLaser() end)
end)

--------------------------------------------------------------------
-- View Player system (LISTA SCROLLABLE DE JOGADORES)
local PLAYER_BTN_W, PLAYER_BTN_H = 100, 24
local PLAYER_BTN_SPACING = 4
local LINE_HEIGHT = PLAYER_BTN_H + PLAYER_BTN_SPACING * 2

-- Function to start spectating
local function startViewMode(targetPlayer)
    if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
        statusLabel.Text = "Jogador sem personagem"
        statusLabel.TextColor3 = Color3.fromRGB(255,150,100)
        task.delay(1.5, function()
            statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
            statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
        end)
        return
    end

    if targetPlayer == player then
        statusLabel.Text = "Não pode visualizar a si mesmo."
        statusLabel.TextColor3 = Color3.fromRGB(255,100,100)
        task.delay(1.5, function()
            statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
            statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
        end)
        return
    end

    -- Esconder a linha de jogadores
    if playersLineFrame and playersLineFrame.Parent then hidePlayersLine() end

    -- Desabilitar shift lock se estiver ligado
    if shiftLocked then
        disableShiftLock()
    end

    viewingPlayer = targetPlayer
    viewModeActive = true

    -- Armazenar o estado original da câmera
    if Workspace.CurrentCamera then
        originalCameraType = Workspace.CurrentCamera.CameraType
        originalCameraSubject = Workspace.CurrentCamera.CameraSubject
    end
    
    statusLabel.Text = "Visualizando: "..targetPlayer.Name
    statusLabel.TextColor3 = Color3.fromRGB(200, 255, 200)

    -- Criar pequeno botão para voltar
    if not backViewBtn or not backViewBtn.Parent then
        backViewBtn = Instance.new("TextButton")
        backViewBtn.Name = "PH_ViewBackBtn"
        backViewBtn.Size = UDim2.new(0, 44, 0, 28)
        backViewBtn.Position = UDim2.new(1, -54, 1, -38) -- canto inferior-direito
        backViewBtn.AnchorPoint = Vector2.new(0,0)
        backViewBtn.Text = "🔙"
        backViewBtn.Font = Enum.Font.SourceSansBold
        backViewBtn.TextSize = 14
        backViewBtn.BackgroundColor3 = BUTTON_RED
        backViewBtn.TextColor3 = Color3.new(1,1,1)
        backViewBtn.Parent = screenGui
        backViewBtn.ZIndex = 3000
        Instance.new("UICorner", backViewBtn).CornerRadius = UDim.new(0,6)

        backViewBtn.MouseButton1Click:Connect(function()
            -- Parar modo de visualização
            if viewLoop then viewLoop:Disconnect(); viewLoop = nil end
            viewingPlayer = nil
            viewModeActive = false
            if backViewBtn and backViewBtn.Parent then backViewBtn:Destroy(); backViewBtn = nil end
            
            -- Restaurar a câmera
            pcall(function()
                if Workspace.CurrentCamera then
                    if originalCameraType then Workspace.CurrentCamera.CameraType = originalCameraType end
                    if originalCameraSubject then Workspace.CurrentCamera.CameraSubject = originalCameraSubject end
                end
            end)
            statusLabel.Text = farming and "Auto Farm Ligado" or "Auto Farm Desligado"
            statusLabel.TextColor3 = farming and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,220,220)
        end)
    end

    -- Iniciar loop de visualização (SPECTATE)
    if viewLoop then viewLoop:Disconnect(); viewLoop = nil end
    viewLoop = RunService.RenderStepped:Connect(function()
        if not viewingPlayer or not viewingPlayer.Character or not viewingPlayer.Character:FindFirstChild("HumanoidRootPart") then
            -- Alvo perdido -> parar
            if viewLoop then viewLoop:Disconnect(); viewLoop = nil end
            viewingPlayer = nil
            viewModeActive = false
            if backViewBtn and backViewBtn.Parent then backViewBtn:Destroy(); backViewBtn = nil end
            pcall(function()
                if Workspace.CurrentCamera then
                    if originalCameraType then Workspace.CurrentCamera.CameraType = originalCameraType end
                    if originalCameraSubject then Workspace.CurrentCamera.CameraSubject = originalCameraSubject end
                end
            end)
            statusLabel.Text = "Visualização Encerrada (alvo perdido)"
            statusLabel.TextColor3 = Color3.fromRGB(255,100,100)
            return
        end

        local cam = Workspace.CurrentCamera
        if not cam then return end

        -- Computar a posição desejada da câmera atrás do alvo (Lógica de spectate)
        local targetHRP = viewingPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not targetHRP then return end
        local offsetBack = 3.5
        local offsetRight = 0.6
        local height = 1.6
        local rootCFrame = targetHRP.CFrame
        local lookVector = rootCFrame.LookVector
        local rightVector = rootCFrame.RightVector
        local targetPos = targetHRP.Position
        local camPos = targetPos - (lookVector * offsetBack) + (rightVector * offsetRight) + Vector3.new(0, height, 0)
        local lookAt = targetPos + Vector3.new(0, 1.2, 0)

        pcall(function()
            cam.CameraType = Enum.CameraType.Scriptable
            cam.CFrame = cam.CFrame:Lerp(CFrame.new(camPos, lookAt), 0.4)
        end)
    end)
end

-- helper to create horizontal player list UI
local function createPlayersLine()
    -- remove previous line if exists
    if playersLineFrame and playersLineFrame.Parent then playersLineFrame:Destroy() end

    playersLineFrame = Instance.new("ScrollingFrame")
    playersLineFrame.Name = "PlayersLine"
    playersLineFrame.Size = UDim2.new(1, 0, 0, LINE_HEIGHT)
    playersLineFrame.Position = UDim2.new(0, 0, 0, -LINE_HEIGHT - 10) -- Top-center, start position (off-screen)
    playersLineFrame.BackgroundTransparency = 1
    playersLineFrame.BorderSizePixel = 0
    playersLineFrame.ScrollingDirection = Enum.ScrollingDirection.X
    playersLineFrame.CanvasSize = UDim2.new(0, 0, 0, 0) -- Adjust automatically
    playersLineFrame.Parent = screenGui
    playersLineFrame.ZIndex = 2000
	-- Conexão de Scroll para a lista de jogadores (mantida)
	local draggingList, dragStartList, startOffsetList
	playersLineFrame.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
			draggingList = true
			dragStartList = input.Position
			startOffsetList = Vector2.new(playersLineFrame.CanvasPosition.X, playersLineFrame.CanvasPosition.Y)
		end
	end)
	playersLineFrame.InputChanged:Connect(function(input)
		if draggingList and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
			local delta = input.Position - dragStartList
			playersLineFrame.CanvasPosition = Vector2.new(startOffsetList.X - delta.X, startOffsetList.Y) -- Only allow X scrolling
		end
	end)
	playersLineFrame.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
			draggingList = false
		end
	end)


    local uiList = Instance.new("UIListLayout")
    uiList.Parent = playersLineFrame
    uiList.Padding = UDim2.new(0, PLAYER_BTN_SPACING, 0, 0)
    uiList.FillDirection = Enum.FillDirection.Horizontal
    uiList.VerticalAlignment = Enum.VerticalAlignment.Center

    local buttonCount = 0
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player then
            buttonCount = buttonCount + 1
            local b = Instance.new("TextButton")
            b.Size = UDim2.new(0, PLAYER_BTN_W, 0, PLAYER_BTN_H)
            b.Text = p.Name
            b.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
            b.TextColor3 = Color3.new(1,1,1)
            b.Font = Enum.Font.SourceSans
            b.TextSize = 14
            b.Parent = playersLineFrame
            Instance.new("UICorner", b).CornerRadius = UDim.new(0,4)

            -- Connect the button to the spectate function
            local playerRef = p
            b.MouseButton1Click:Connect(function()
                if viewModeActive then
                    statusLabel.Text = "Já em visualização. Clique em 🔙 para parar."
                    statusLabel.TextColor3 = Color3.fromRGB(255,200,0)
                else
                    startViewMode(playerRef)
                end
            end)
        end
    end

    if buttonCount > 0 then
        -- Set CanvasSize to fit all buttons horizontally + some padding
        local canvasWidth = (PLAYER_BTN_W + PLAYER_BTN_SPACING) * buttonCount + PLAYER_BTN_SPACING
        playersLineFrame.CanvasSize = UDim2.new(0, canvasWidth, 0, 0)
    end

    -- Final position: slightly below screen top
    local targetY = 10 
    local targetPos = UDim2.new(0, 0, 0, targetY)

    -- Animate it in
    local tween = TweenService:Create(playersLineFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position = targetPos})
    tween:Play()
end

-- Function to hide the player list
local function hidePlayersLine()
    if playersLineFrame and playersLineFrame.Parent then
        local targetY = -LINE_HEIGHT - 10
        local targetPos = UDim2.new(0, 0, 0, targetY)
        local tween = TweenService:Create(playersLineFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Position = targetPos})
        tween:Play()
        tween.Completed:Connect(function()
            playersLineFrame:Destroy()
            playersLineFrame = nil
        end)
    end
    playersLineVisible = false
end

-- View Player button action (AGORA ALTERNA A LINHA)
viewPlayerBtn.MouseButton1Click:Connect(function()
    if viewModeActive then
        statusLabel.Text = "Já em visualização. Clique em 🔙 para parar."
        statusLabel.TextColor3 = Color3.fromRGB(255,200,0)
        return
    end

    if playersLineVisible then
        hidePlayersLine()
    else
        createPlayersLine()
        playersLineVisible = true
    end
end)

-- Update players list when players join/leave (automatic refresh)
Players.PlayerAdded:Connect(function()
    if playersLineVisible then
        task.delay(0.5, createPlayersLine) -- Refresh with a small delay
    end
end)
Players.PlayerRemoving:Connect(function()
    if playersLineVisible then
        task.delay(0.5, createPlayersLine) -- Refresh with a small delay
    end
end)

--------------------------------------------------------------------
-- INTEGRATED MINI TOGGLE (colado com o menu)
local miniToggleBtn = Instance.new("TextButton")
miniToggleBtn.Name = "PH_MiniToggle"
miniToggleBtn.Size = UDim2.new(0, 28, 0, 28)
miniToggleBtn.AnchorPoint = Vector2.new(0, 0)
miniToggleBtn.BackgroundColor3 = Color3.fromRGB(0,0,0)
miniToggleBtn.BackgroundTransparency = 0.45
miniToggleBtn.Text = "≡"
miniToggleBtn.TextColor3 = Color3.new(1,1,1)
miniToggleBtn.Font = Enum.Font.SourceSansBold
miniToggleBtn.TextSize = 16
miniToggleBtn.Parent = screenGui
miniToggleBtn.ZIndex = 1001
Instance.new("UICorner", miniToggleBtn).CornerRadius = UDim.new(0,6)

-- NOVO: O miniToggleBtn é arrastável e move o menuFrame
miniToggleBtn.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		startDrag(i) -- Usa a mesma função startDrag para mover o menu
	end
end)

-- helper to position mini relative to menu position
local function updateMiniPos()
	local x = menuFrame.Position.X.Offset + MENU_WIDTH - 28
	local y = menuFrame.Position.Y.Offset + 8
	miniToggleBtn.Position = UDim2.new(menuFrame.Position.X.Scale, x, menuFrame.Position.Y.Scale, y)
end
updateMiniPos()

-- O Heartbeat garante que o mini toggle se mova suavemente com o menu, mesmo durante o arrasto.
local miniUpdateConn = RunService.Heartbeat:Connect(function() 
	-- Apenas atualiza quando o menu está visível, mas pode ser otimizado para rodar sempre
	-- para o caso de o menu ser reativado
	updateMiniPos() 
end)

-- animation helpers (fade show/hide)
local tweenInfoSlow = TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)

local menuVisible = true
local function animateHideMenu()
	if not menuVisible then return end
	menuVisible = false
	local t1 = TweenService:Create(menuFrame, tweenInfoSlow, {BackgroundTransparency = 1})
	local t2 = TweenService:Create(titleLabel, tweenInfoSlow, {TextTransparency = 1})
	t1:Play(); t2:Play()
	for _, obj in pairs(menuFrame:GetChildren()) do
		if obj:IsA("TextButton") or obj:IsA("TextLabel") then
			pcall(function() TweenService:Create(obj, tweenInfoSlow, {TextTransparency = 1}):Play() end)
		end
	end
	task.delay(0.18, function()
		menuFrame.Visible = false
	end)
end

local function animateShowMenu()
	if menuVisible then return end
	menuVisible = true
	menuFrame.BackgroundTransparency = 1
	titleLabel.TextTransparency = 1
	for _, obj in pairs(menuFrame:GetChildren()) do
		if obj:IsA("TextButton") or obj:IsA("TextLabel") then
			obj.TextTransparency = 1
		end
	end
	menuFrame.Visible = true
	local t1 = TweenService:Create(menuFrame, tweenInfoSlow, {BackgroundTransparency = 0})
	local t2 = TweenService:Create(titleLabel, tweenInfoSlow, {TextTransparency = 0})
	t1:Play(); t2:Play()
	for _, obj in pairs(menuFrame:GetChildren()) do
		if obj:IsA("TextButton") or obj:IsA("TextLabel") then
			pcall(function() TweenService:Create(obj, tweenInfoSlow, {TextTransparency = 0}):Play() end)
		end
	end
end

miniToggleBtn.MouseButton1Click:Connect(function()
	if menuVisible then
		animateHideMenu()
	else
		animateShowMenu()
	end
end)

-- set initial text transparencies
for _, obj in pairs(menuFrame:GetChildren()) do
	if obj:IsA("TextButton") or obj:IsA("TextLabel") then
		obj.TextTransparency = 0
	end
end

--------------------------------------------------------------------
-- Scroll input: touch drag + mouse drag + mouse wheel
local dragging = false
local dragStart, startOffset

scrollButtons.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragStart = input.Position
		startOffset = Vector2.new(scrollButtons.CanvasPosition.X, scrollButtons.CanvasPosition.Y)
	end
end)

scrollButtons.InputChanged:Connect(function(input)
	if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
		local delta = input.Position - dragStart
		scrollButtons.CanvasPosition = startOffset - delta
	end
end)

scrollButtons.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = false
	end
end)

scrollButtons.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseWheel then
		scrollButtons.CanvasPosition = scrollButtons.CanvasPosition - Vector2.new(0, input.Position.Z * 20)
	end
end)

-- Scroll input for player list (horizontal drag) - OBS: JÁ ADICIONADO NA FUNÇÃO createPlayersLine()
-- Não é necessário duplicar aqui.

--------------------------------------------------------------------
-- ensure consistent state on start
if not menuVisible then
	menuFrame.Visible = false
else
	menuFrame.Visible = true
end

-- cleanup on destroy/unload
screenGui.Destroying:Connect(function()
	if miniUpdateConn then miniUpdateConn:Disconnect(); miniUpdateConn = nil end
	if farmLoop then farmLoop:Disconnect(); farmLoop = nil end
	if espUpdateConn then espUpdateConn:Disconnect(); espUpdateConn = nil end
	if targetConn then targetConn:Disconnect(); targetConn = nil end
	if aimLoop then aimLoop:Disconnect(); aimLoop = nil end
	if shiftConn then shiftConn:Disconnect(); shiftConn = nil end
	if viewLoop then viewLoop:Disconnect(); viewLoop = nil end
	if backViewBtn and backViewBtn.Parent then backViewBtn:Destroy(); backViewBtn = nil end
	if playersLineFrame and playersLineFrame.Parent then playersLineFrame:Destroy(); playersLineFrame = nil end
	if currentSound then
		pcall(function() currentSound:Stop() end)
		pcall(function() currentSound:Destroy() end)
		currentSound = nil
	end
	-- Limpa as conexões de arrasto globais para evitar vazamento de memória (melhor prática)
	UIS.InputChanged:Disconnect(updateDrag)
	UIS.InputEnded:Disconnect(endDrag)
	
end)

print("[✅ Power Hub V3.6 Clean Compact +13% — Full Systems + Scroll + View Player (Lista Scrollable) [DRAG APRIMORADO] carregado]")
