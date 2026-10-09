--[[
==============================================================================
  SHARK HUB  |  Zombie Attack Survival — Admin / Test Panel
  ScreenGui Name : ZombieAttackProjectUI
  PlaceId ref    : 87413622775071
==============================================================================
  Modular client-side admin/test UI.
  Structure verified against game dump + decompiled client scripts.
==============================================================================
]]

-- ============================================================================
-- CONFIG
-- ============================================================================
local Config = {
	ToggleKey       = Enum.KeyCode.LeftControl,
	WalkSpeed       = 16,
	FlySpeed        = 50,
	NoClip          = false,
	Fly             = false,
	Godmode         = false,
	AimBot          = false,
	SilentAim       = false,
	NoCooldown      = false,
	NoRecoil        = false,
	AutoKill        = false,
	BringMobs       = false,
	AutoHeadshot    = false,
	KillAura        = false,
	KillAuraRange   = 12,
	Theme           = "Dark",
	UIName          = "ZombieAttackProjectUI",
	WindowSize      = UDim2.new(0, 680, 0, 480),
	WindowMinSize   = Vector2.new(440, 320),
}

-- ============================================================================
-- SERVICES
-- ============================================================================
local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")
local TweenService      = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace         = game:GetService("Workspace")
local StarterGui        = game:GetService("StarterGui")

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")

-- ============================================================================
-- CLEANUP (prevent duplicate UI / connections)
-- ============================================================================
do
	local old = PlayerGui:FindFirstChild(Config.UIName)
	if old then
		old:Destroy()
	end
end

-- ============================================================================
-- GAME STRUCTURE (paths confirmed from dump)
-- ============================================================================
local GameStructure = {
	RoundInfoFolder = ReplicatedStorage:FindFirstChild("RoundInfo"),
	EventsFolder    = ReplicatedStorage:FindFirstChild("Events"),
	ActionsFolder   = nil, -- filled below
	SharedFolder    = ReplicatedStorage:FindFirstChild("Shared"),
	ZombiesModule   = nil, -- Shared/Zombies
	ActiveZombies   = Workspace:FindFirstChild("ActiveZombies"),
	ZombieClientVisuals = Workspace:FindFirstChild("Zombie_ClientVisuals"),
	ZombiesFolder   = Workspace:FindFirstChild("Zombies"),
	CharactersFolder= Workspace:FindFirstChild("Characters"),
	CorrectWeapons  = ReplicatedStorage:FindFirstChild("CorrectToolgripWeapons"),
}

if GameStructure.EventsFolder then
	GameStructure.ActionsFolder = GameStructure.EventsFolder:FindFirstChild("Actions")
end
if GameStructure.SharedFolder then
	GameStructure.ZombiesModule = GameStructure.SharedFolder:FindFirstChild("Zombies")
end

-- ============================================================================
-- ROUND INFO (ReplicatedStorage/RoundInfo)
-- Confirmed attributes + Value objects from dump
-- ============================================================================
local RoundInfo = {}

local function safeGetAttr(folder, name)
	if not folder then return nil end
	local ok, val = pcall(function()
		return folder:GetAttribute(name)
	end)
	if ok then return val end
	return nil
end

local function safeGetValue(folder, name)
	if not folder then return nil end
	local obj = folder:FindFirstChild(name)
	if obj and (obj:IsA("IntValue") or obj:IsA("NumberValue") or obj:IsA("BoolValue") or obj:IsA("StringValue")) then
		return obj.Value
	end
	return nil
end

function RoundInfo.GetWave()
	return safeGetValue(GameStructure.RoundInfoFolder, "Wave")
		or safeGetAttr(GameStructure.RoundInfoFolder, "Wave")
		or safeGetValue(GameStructure.RoundInfoFolder, "CurrentWave")
		or safeGetAttr(GameStructure.RoundInfoFolder, "CurrentWave")
end

function RoundInfo.GetAliveZombies()
	return safeGetValue(GameStructure.RoundInfoFolder, "AliveZombies")
		or safeGetAttr(GameStructure.RoundInfoFolder, "AliveZombies")
end

function RoundInfo.GetGlobalAliveZombies()
	return safeGetValue(GameStructure.RoundInfoFolder, "GlobalAliveZombies")
		or safeGetAttr(GameStructure.RoundInfoFolder, "GlobalAliveZombies")
end

function RoundInfo.GetPlayerCount()
	return safeGetValue(GameStructure.RoundInfoFolder, "PlayerCount")
		or safeGetAttr(GameStructure.RoundInfoFolder, "PlayerCount")
end

function RoundInfo.GetState()
	return safeGetValue(GameStructure.RoundInfoFolder, "State")
		or safeGetAttr(GameStructure.RoundInfoFolder, "State")
end

function RoundInfo.GetRoundState()
	return safeGetValue(GameStructure.RoundInfoFolder, "RoundState")
		or safeGetAttr(GameStructure.RoundInfoFolder, "RoundState")
end

function RoundInfo.GetZombiesRemaining()
	return safeGetValue(GameStructure.RoundInfoFolder, "ZombiesRemaining")
		or safeGetAttr(GameStructure.RoundInfoFolder, "ZombiesRemaining")
		or RoundInfo.GetAliveZombies()
end

function RoundInfo.GetDifficulty()
	return safeGetValue(GameStructure.RoundInfoFolder, "Difficulty")
		or safeGetAttr(GameStructure.RoundInfoFolder, "Difficulty")
end

function RoundInfo.GetWaitingForWaveEnd()
	return safeGetValue(GameStructure.RoundInfoFolder, "WaitingForWaveEnd")
		or safeGetAttr(GameStructure.RoundInfoFolder, "WaitingForWaveEnd")
end

function RoundInfo.GetWaveStartedAt()
	return safeGetValue(GameStructure.RoundInfoFolder, "WaveStartedAt")
		or safeGetAttr(GameStructure.RoundInfoFolder, "WaveStartedAt")
end

-- ============================================================================
-- CHARACTER CONTROLLER
-- ============================================================================
local CharacterController = {
	_character = nil,
	_humanoid  = nil,
	_root      = nil,
	_connections = {},
	_noClipConn  = nil,
	_flyConn     = nil,
	_flyBV       = nil,
	_godConn     = nil,
	_desiredWalkSpeed = Config.WalkSpeed,
	_keys = {
		W = false, A = false, S = false, D = false,
		Space = false, LeftShift = false,
	},
}

function CharacterController.GetCharacter()
	return LocalPlayer.Character
end

function CharacterController.GetHumanoid()
	local char = CharacterController.GetCharacter()
	if char then
		return char:FindFirstChildOfClass("Humanoid")
	end
	return nil
end

function CharacterController.GetRoot()
	local char = CharacterController.GetCharacter()
	if char then
		return char:FindFirstChild("HumanoidRootPart")
	end
	return nil
end

function CharacterController.SetWalkSpeed(speed)
	CharacterController._desiredWalkSpeed = speed
	Config.WalkSpeed = speed
	local hum = CharacterController.GetHumanoid()
	if hum and not Config.Fly then
		hum.WalkSpeed = speed
	end
end

function CharacterController.ApplyNoClip(enabled)
	Config.NoClip = enabled
	if CharacterController._noClipConn then
		CharacterController._noClipConn:Disconnect()
		CharacterController._noClipConn = nil
	end

	-- When disabling: stop forcing CanCollide; do NOT force true on every part
	-- (Humanoid/rig manage collision; forcing true breaks accessories and thin parts)
	if not enabled then
		return
	end

	CharacterController._noClipConn = RunService.Stepped:Connect(function()
		if not Config.NoClip then return end
		local char = CharacterController.GetCharacter()
		if not char then return end
		for _, part in ipairs(char:GetDescendants()) do
			if part:IsA("BasePart") then
				part.CanCollide = false
			end
		end
	end)
end

function CharacterController.ApplyFly(enabled)
	Config.Fly = enabled

	if CharacterController._flyConn then
		CharacterController._flyConn:Disconnect()
		CharacterController._flyConn = nil
	end
	if CharacterController._flyBV then
		CharacterController._flyBV:Destroy()
		CharacterController._flyBV = nil
	end

	local hum = CharacterController.GetHumanoid()
	local root = CharacterController.GetRoot()

	if not enabled then
		if hum then
			hum.PlatformStand = false
			hum.WalkSpeed = CharacterController._desiredWalkSpeed or Config.WalkSpeed
		end
		return
	end

	if not root or not hum then return end

	hum.PlatformStand = true

	local bv = Instance.new("BodyVelocity")
	bv.Name = "ZAP_FlyVelocity"
	bv.MaxForce = Vector3.new(1e5, 1e5, 1e5)
	bv.Velocity = Vector3.zero
	bv.Parent = root
	CharacterController._flyBV = bv

	CharacterController._flyConn = RunService.RenderStepped:Connect(function()
		if not Config.Fly then return end
		local r = CharacterController.GetRoot()
		local h = CharacterController.GetHumanoid()
		local bodyVel = CharacterController._flyBV
		if not r or not h or not bodyVel or not bodyVel.Parent then return end

		h.PlatformStand = true
		local cam = Workspace.CurrentCamera
		if not cam then return end

		local move = Vector3.zero
		local cf = cam.CFrame
		local keys = CharacterController._keys

		if keys.W then move = move + cf.LookVector end
		if keys.S then move = move - cf.LookVector end
		if keys.A then move = move - cf.RightVector end
		if keys.D then move = move + cf.RightVector end
		if keys.Space then move = move + Vector3.new(0, 1, 0) end
		if keys.LeftShift then move = move - Vector3.new(0, 1, 0) end

		if move.Magnitude > 0 then
			bodyVel.Velocity = move.Unit * (Config.FlySpeed or 50)
		else
			bodyVel.Velocity = Vector3.zero
		end
	end)
end

--[[
  Godmode (client-side attempt):
  Zombie damage is processed server-side in this game. Restoring Health on the
  client may be overwritten by the server. This is a best-effort local shield
  for testing; true invulnerability requires a server-side adapter.
]]
function CharacterController.ApplyGodmode(enabled)
	Config.Godmode = enabled
	if CharacterController._godConn then
		CharacterController._godConn:Disconnect()
		CharacterController._godConn = nil
	end
	if not enabled then return end

	local function bind(hum)
		if not hum then return end
		CharacterController._godConn = hum.HealthChanged:Connect(function(hp)
			if not Config.Godmode then return end
			if hp < hum.MaxHealth then
				hum.Health = hum.MaxHealth
			end
		end)
		hum.Health = hum.MaxHealth
	end

	bind(CharacterController.GetHumanoid())
end

function CharacterController.OnCharacterAdded(char)
	CharacterController._character = char
	local hum = char:WaitForChild("Humanoid", 5)
	local root = char:WaitForChild("HumanoidRootPart", 5)
	CharacterController._humanoid = hum
	CharacterController._root = root

	-- Clean leftover fly instance from previous life
	if CharacterController._flyBV then
		pcall(function() CharacterController._flyBV:Destroy() end)
		CharacterController._flyBV = nil
	end

	if hum and CharacterController._desiredWalkSpeed then
		hum.WalkSpeed = CharacterController._desiredWalkSpeed
	end

	if Config.NoClip then
		CharacterController.ApplyNoClip(true)
	end
	if Config.Fly then
		task.defer(function()
			CharacterController.ApplyFly(true)
		end)
	end
	if Config.Godmode then
		task.defer(function()
			CharacterController.ApplyGodmode(true)
		end)
	end
end

function CharacterController.Init()
	if LocalPlayer.Character then
		CharacterController.OnCharacterAdded(LocalPlayer.Character)
	end
	LocalPlayer.CharacterAdded:Connect(CharacterController.OnCharacterAdded)

	-- Fly movement keys (single connections)
	UserInputService.InputBegan:Connect(function(input, gpe)
		if gpe then return end
		local k = input.KeyCode
		if k == Enum.KeyCode.W then CharacterController._keys.W = true
		elseif k == Enum.KeyCode.A then CharacterController._keys.A = true
		elseif k == Enum.KeyCode.S then CharacterController._keys.S = true
		elseif k == Enum.KeyCode.D then CharacterController._keys.D = true
		elseif k == Enum.KeyCode.Space then CharacterController._keys.Space = true
		elseif k == Enum.KeyCode.LeftShift then CharacterController._keys.LeftShift = true
		end
	end)
	UserInputService.InputEnded:Connect(function(input)
		local k = input.KeyCode
		if k == Enum.KeyCode.W then CharacterController._keys.W = false
		elseif k == Enum.KeyCode.A then CharacterController._keys.A = false
		elseif k == Enum.KeyCode.S then CharacterController._keys.S = false
		elseif k == Enum.KeyCode.D then CharacterController._keys.D = false
		elseif k == Enum.KeyCode.Space then CharacterController._keys.Space = false
		elseif k == Enum.KeyCode.LeftShift then CharacterController._keys.LeftShift = false
		end
	end)
end

-- ============================================================================
-- SAFE ZONE FINDER
-- Dump shows: Workspace/ZombieTestMap/PlayerSpawn (SpawnLocation)
-- No explicit "SafeZone" name found. Priority list based on dump.
-- ============================================================================
local function FindSafeZone()
	-- Priority 1: exact path from dump
	local zt = Workspace:FindFirstChild("ZombieTestMap")
	if zt then
		local ps = zt:FindFirstChild("PlayerSpawn")
		if ps and ps:IsA("BasePart") then
			return ps
		end
	end

	-- Priority 2: any SpawnLocation named PlayerSpawn
	for _, obj in ipairs(Workspace:GetDescendants()) do
		if obj:IsA("SpawnLocation") and (obj.Name == "PlayerSpawn" or obj.Name == "SpawnLocation") then
			return obj
		end
	end

	-- Priority 3: common fallback names (only if they exist)
	local candidates = {"SafeZone", "Safe Zone", "Safezone", "Lobby", "LobbySpawn", "Spawn"}
	for _, name in ipairs(candidates) do
		local found = Workspace:FindFirstChild(name, true)
		if found and found:IsA("BasePart") then
			return found
		end
	end

	return nil
end

local function TeleportToSafeZone()
	local zone = FindSafeZone()
	if not zone then
		warn("[ZombieAttackProjectUI] SafeZone / PlayerSpawn not found.")
		return false, "SafeZone não encontrada"
	end
	local root = CharacterController.GetRoot()
	if not root then
		warn("[ZombieAttackProjectUI] HumanoidRootPart not found.")
		return false, "Personagem não encontrado"
	end
	root.CFrame = zone.CFrame + Vector3.new(0, 3, 0)
	return true, "Teleportado"
end

-- ============================================================================
-- PROJECT API (combat adapters - UI only calls these)
-- Signatures extracted from decompiled client scripts (.rbxlx)
-- ============================================================================
--[[
  Confirmed from Scripts_87413622775071.rbxlx:

  MeleeRequest:FireServer({
    weaponId = <tool name string>,
    swingId  = <number>,
    predictedZombieId = <number|nil>
  })

  Workspace.Zombie_ClientHooks.HitPrediction : BindableEvent
  :Fire(
    zombieId,          -- number
    hitboxRole,        -- "Body" | "Head"
    hitPosition,       -- Vector3
    damage,            -- number
    weaponName,        -- string
    swingId,           -- number
    highRate,          -- boolean
    origin,            -- Vector3
    direction,         -- Vector3 (unit, horizontal)
    range,             -- number
    hitboxRole2,       -- "Body" | "Head"
    uniqueId           -- string e.g. "melee:"..swingId
  )

  Fire:FireServer(weaponName, RaycastData, BulletData, firstClientShotId, clientClip, clientAmmoActionId)
  FireProjectileWeapon:FireServer(weaponName, aimTargetPos, firstClientShotId, clientClip, clientAmmoActionId, seed)
  -> RaycastData/BulletData builders are complex; gun auto-fire left as TODO for full Protocol integration.
]]

local ProjectAPI = {
	_connections = {},
	_aimbotConn   = nil,
	_autoKillConn = nil,
	_killAuraConn = nil,
	_bringConn    = nil,
	_noCdConn     = nil,
	_headshotConn = nil,
	_silentConn   = nil,
	_noRecoilConn = nil,
	_swingId      = 0,
	_shotId       = 0,
	_lastAuraAt   = 0,
	_lastAutoAt   = 0,
	_silentTarget = nil,
	_gunDataCache = nil,
	_shakeMod     = nil,
	_fpsViewport  = nil,
}

-- ---------------------------------------------------------------------------
-- GunData / ClientGunDependencies (paths from decompile)
-- ---------------------------------------------------------------------------
local function getGunDataModule()
	if ProjectAPI._gunDataCache then
		return ProjectAPI._gunDataCache
	end
	local shared = GameStructure.SharedFolder or ReplicatedStorage:FindFirstChild("Shared")
	if not shared then return nil end
	local modules = shared:FindFirstChild("Modules")
	if not modules then return nil end
	local mod = modules:FindFirstChild("GunData")
	if mod then
		local ok, data = pcall(require, mod)
		if ok then
			ProjectAPI._gunDataCache = data
			return data
		end
	end
	return nil
end

local function getWeaponGunInfo(weaponName)
	local gd = getGunDataModule()
	if not gd or not weaponName then return nil end
	local info = gd[weaponName]
	if typeof(info) == "table" then
		return info
	end
	return nil
end

local function getClientGunDep(name)
	local shared = GameStructure.SharedFolder or ReplicatedStorage:FindFirstChild("Shared")
	if not shared then return nil end
	local deps = shared:FindFirstChild("ClientGunDependencies")
	if not deps then return nil end
	return deps:FindFirstChild(name)
end

local function getHumanoidCameraShake()
	if ProjectAPI._shakeMod ~= nil then
		return ProjectAPI._shakeMod ~= false and ProjectAPI._shakeMod or nil
	end
	local mod = getClientGunDep("HumanoidCameraShake")
	if mod then
		local ok, req = pcall(require, mod)
		if ok and typeof(req) == "table" then
			ProjectAPI._shakeMod = req
			return req
		end
	end
	ProjectAPI._shakeMod = false
	return nil
end

local function getFPSViewport()
	if ProjectAPI._fpsViewport ~= nil then
		return ProjectAPI._fpsViewport ~= false and ProjectAPI._fpsViewport or nil
	end
	local mod = getClientGunDep("FPSViewport")
	if not mod then
		mod = getClientGunDep("DisabledFPSViewport")
	end
	if mod then
		local ok, req = pcall(require, mod)
		if ok and typeof(req) == "table" then
			ProjectAPI._fpsViewport = req
			return req
		end
	end
	ProjectAPI._fpsViewport = false
	return nil
end

local function nextShotId()
	ProjectAPI._shotId = ProjectAPI._shotId + 1
	if ProjectAPI._shotId > 4294967295 then
		ProjectAPI._shotId = 1
	end
	return ProjectAPI._shotId
end

-- Resolve remotes (re-resolve if destroyed)
local MeleeRequestRemote = nil
local function getMeleeRequest()
	if MeleeRequestRemote and MeleeRequestRemote.Parent then
		return MeleeRequestRemote
	end
	local actions = GameStructure.ActionsFolder
		or (ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("Actions"))
	if actions then
		MeleeRequestRemote = actions:FindFirstChild("MeleeRequest")
	end
	return MeleeRequestRemote
end

local function getHitPredictionEvent()
	local hooks = Workspace:FindFirstChild("Zombie_ClientHooks")
	if not hooks then return nil end
	local ev = hooks:FindFirstChild("HitPrediction")
	if ev and ev:IsA("BindableEvent") then
		return ev
	end
	return nil
end

-- Always re-query folders (they may be empty at script load)
local function getZombieFolder()
	local f = Workspace:FindFirstChild("Zombie_ClientVisuals")
	if f then return f end
	f = Workspace:FindFirstChild("ActiveZombies")
	if f then return f end
	return Workspace:FindFirstChild("Zombies")
end

local function getZombieModels()
	local list = {}
	local folder = getZombieFolder()
	if not folder then return list end
	for _, child in ipairs(folder:GetChildren()) do
		if child:IsA("Model") then
			local root = child:FindFirstChild("HumanoidRootPart")
			if root and root:IsA("BasePart") then
				table.insert(list, child)
			end
		end
	end
	return list
end

local function getZombieId(model)
	local attr = model:GetAttribute("ZombieId")
	if typeof(attr) == "number" then
		return attr
	end
	return tonumber(string.match(model.Name, "Zombie_(%d+)"))
end

local function getHeadPart(model)
	local head = model:FindFirstChild("Head")
	if head and head:IsA("BasePart") then
		return head
	end
	local fake = model:FindFirstChild("FakeHead")
	if fake and fake:IsA("BasePart") then
		return fake
	end
	return model:FindFirstChild("HumanoidRootPart")
end

local function getEquippedWeaponName()
	local char = CharacterController.GetCharacter()
	if not char then return nil end
	for _, child in ipairs(char:GetChildren()) do
		if child:IsA("Tool") then
			local real = child:GetAttribute("RealName")
			if typeof(real) == "string" and real ~= "" then
				return real
			end
			return child.Name
		end
	end
	return nil
end

local function nextSwingId()
	ProjectAPI._swingId = ProjectAPI._swingId + 1
	if ProjectAPI._swingId > 4294967295 then
		ProjectAPI._swingId = 1
	end
	return ProjectAPI._swingId
end

local function getMeleeRange()
	return Config.KillAuraRange or 12
end

local function getMeleeDamage()
	return 100
end

local function rootLookFlat(root)
	local lv = root.CFrame.LookVector
	local flat = Vector3.new(lv.X, 0, lv.Z)
	if flat.Magnitude < 0.001 then
		return Vector3.new(0, 0, -1)
	end
	return flat.Unit
end

-- Direction toward target (used when Silent Aim is on so hit is not look-dependent)
local function directionToTarget(root, targetPos)
	local delta = targetPos - root.Position
	local flat = Vector3.new(delta.X, 0, delta.Z)
	if flat.Magnitude < 0.001 then
		return rootLookFlat(root)
	end
	return flat.Unit
end

local function preferHeadHit()
	return Config.AutoHeadshot or Config.SilentAim
end

-- Core melee hit – confirmed payload from Bat Handler decompile
local function performMeleeHit(zombieModel, forceHead)
	local remote = getMeleeRequest()
	if not remote then return false end

	local root = CharacterController.GetRoot()
	local hum = CharacterController.GetHumanoid()
	if not root or not hum or hum.Health <= 0 then return false end

	local zombieId = getZombieId(zombieModel)
	if not zombieId then return false end

	local useHead = forceHead or preferHeadHit()
	local hitPart
	local role
	if useHead then
		hitPart = getHeadPart(zombieModel)
		role = "Head"
	else
		hitPart = zombieModel:FindFirstChild("Torso") or zombieModel:FindFirstChild("HumanoidRootPart")
		role = "Body"
	end
	if not hitPart then return false end

	local hitPos = hitPart.Position
	local weaponName = getEquippedWeaponName() or "Bat"
	local swingId = nextSwingId()
	local range = getMeleeRange()
	local damage = getMeleeDamage()
	local origin = root.Position + Vector3.new(0, 1.35, 0)

	-- Silent Aim / AutoHeadshot: aim direction at the target, not player look
	local dir
	if Config.SilentAim or Config.AutoHeadshot then
		dir = directionToTarget(root, hitPos)
	else
		dir = rootLookFlat(root)
	end

	remote:FireServer({
		weaponId = weaponName,
		swingId = swingId,
		predictedZombieId = zombieId,
	})

	local pred = getHitPredictionEvent()
	if pred then
		pred:Fire(
			zombieId,
			role,
			hitPos,
			damage,
			weaponName,
			swingId,
			false,
			origin,
			dir,
			range,
			role,
			"melee:" .. tostring(swingId)
		)
	end
	return true
end

local function findClosestZombie(maxDist)
	local root = CharacterController.GetRoot()
	if not root then return nil, nil end
	local bestPart, bestDist, bestModel = nil, maxDist or 200, nil
	for _, z in ipairs(getZombieModels()) do
		local zr = z:FindFirstChild("HumanoidRootPart")
		if zr then
			local d = (zr.Position - root.Position).Magnitude
			if d < bestDist then
				bestDist = d
				bestPart = zr
				bestModel = z
			end
		end
	end
	return bestModel, bestPart
end

local function findClosestInRange(range)
	local root = CharacterController.GetRoot()
	if not root then return nil end
	local bestModel, bestDist = nil, range
	for _, z in ipairs(getZombieModels()) do
		local zr = z:FindFirstChild("HumanoidRootPart")
		if zr then
			local d = (zr.Position - root.Position).Magnitude
			if d <= bestDist then
				bestDist = d
				bestModel = z
			end
		end
	end
	return bestModel
end

function ProjectAPI.AimBot(enabled)
	Config.AimBot = enabled
	if ProjectAPI._aimbotConn then
		ProjectAPI._aimbotConn:Disconnect()
		ProjectAPI._aimbotConn = nil
	end
	if not enabled then return end

	-- Visual camera assist only. Independent from Silent Aim.
	ProjectAPI._aimbotConn = RunService.RenderStepped:Connect(function()
		if not Config.AimBot then return end
		local cam = Workspace.CurrentCamera
		if not cam then return end

		local model = select(1, findClosestZombie(250))
		if not model then return end

		local targetPart
		if Config.AutoHeadshot or Config.SilentAim then
			targetPart = getHeadPart(model)
		else
			targetPart = model:FindFirstChild("HumanoidRootPart") or getHeadPart(model)
		end
		if not targetPart then return end

		local look = CFrame.lookAt(cam.CFrame.Position, targetPart.Position)
		cam.CFrame = cam.CFrame:Lerp(look, 0.22)
	end)
end

function ProjectAPI.SilentAim(enabled)
	Config.SilentAim = enabled
	if ProjectAPI._silentConn then
		ProjectAPI._silentConn:Disconnect()
		ProjectAPI._silentConn = nil
	end
	ProjectAPI._silentTarget = nil

	if not enabled then return end

	--[[
	  Silent Aim tracks closest zombie for:
	  - Melee: HitPrediction direction override (no camera)
	  - Guns: AutoKill uses ZombieShotClient-style Protocol FireShot with
	    origin/direction toward target (no camera). See performGunShot.
	]]
	ProjectAPI._silentConn = RunService.Heartbeat:Connect(function()
		if not Config.SilentAim then return end
		local model = select(1, findClosestZombie(500))
		ProjectAPI._silentTarget = model
	end)
end

function ProjectAPI.NoCooldown(enabled)
	Config.NoCooldown = enabled
	if ProjectAPI._noCdConn then
		ProjectAPI._noCdConn:Disconnect()
		ProjectAPI._noCdConn = nil
	end
	if not enabled then return end

	ProjectAPI._noCdConn = RunService.Heartbeat:Connect(function()
		if not Config.NoCooldown then return end
		local char = CharacterController.GetCharacter()
		if not char then return end
		for _, tool in ipairs(char:GetChildren()) do
			if tool:IsA("Tool") then
				local info = tool:FindFirstChild("Info")
				if info then
					local clip = info:FindFirstChild("Clip")
					local clipSize = info:FindFirstChild("ClipSize")
					local reserve = info:FindFirstChild("Reserve")
					if clip and clip:IsA("IntValue") and clipSize and clipSize:IsA("IntValue") then
						if clip.Value < clipSize.Value then
							clip.Value = clipSize.Value
						end
					end
					if reserve and reserve:IsA("IntValue") and reserve.Value < 9999 then
						reserve.Value = math.max(reserve.Value, 9999)
					end
				end
			end
		end
	end)
end

local function getEquippedTool()
	local char = CharacterController.GetCharacter()
	if not char then return nil end
	for _, child in ipairs(char:GetChildren()) do
		if child:IsA("Tool") then
			return child
		end
	end
	return nil
end

local function isMeleeTool(tool)
	if not tool then return false end
	local real = tool:GetAttribute("RealName") or tool.Name
	local info = getWeaponGunInfo(real)
	if info and info.Type == "Melee" then
		return true
	end
	local meleeNames = {
		Bat = true, Crowbar = true, Cleaver = true, Machete = true,
		Axe = true, Katana = true, Scythe = true, Shovel = true,
		["War Axe"] = true, ["War Hammer"] = true, ["Golf Club"] = true,
		["Mace Ball"] = true, Knife = true,
	}
	if meleeNames[real] then return true end
	local toolInfo = tool:FindFirstChild("Info")
	if toolInfo and toolInfo:FindFirstChild("Clip") then
		return false
	end
	return true
end

local function getFireInterval(tool)
	if Config.NoCooldown then
		return 0.06
	end
	if not tool then return 0.2 end
	local name = tool:GetAttribute("RealName") or tool.Name
	local info = getWeaponGunInfo(name)
	if info then
		local rate = info.RateOfFire or info.FireRate
		if typeof(rate) == "number" and rate > 0 and rate < 10 then
			return math.clamp(rate, 0.05, 1.5)
		end
		if typeof(rate) == "number" and rate >= 10 then
			-- FireRate in RPM style (e.g. 1600) → seconds between shots
			return math.clamp(60 / rate, 0.05, 1.5)
		end
		if info.Type == "Melee" and typeof(info.Debounce) == "number" then
			return math.clamp(info.Debounce, 0.1, 2)
		end
	end
	return 0.18
end

-- ZombieProtocol client shot (from ZombieShotClient.FireShot)
local function getZombieProtocol()
	local shared = GameStructure.SharedFolder or ReplicatedStorage:FindFirstChild("Shared")
	if not shared then return nil end
	local zombies = shared:FindFirstChild("Zombies")
	if not zombies then return nil end
	local proto = zombies:FindFirstChild("ZombieProtocol")
	if not proto then return nil end
	local ok, mod = pcall(require, proto)
	if ok then return mod end
	return nil
end

local function getZombieClientEvent(protocol)
	if not protocol then return nil end
	local folderName = protocol.RemoteFolderName
	local eventName = protocol.ClientEventName or protocol.EventName
	if folderName then
		local folder = ReplicatedStorage:FindFirstChild(folderName)
		if folder and eventName then
			local ev = folder:FindFirstChild(eventName)
			if ev and ev:IsA("RemoteEvent") then
				return ev
			end
		end
	end
	if typeof(protocol.GetClientEvent) == "function" then
		local ok, ev = pcall(protocol.GetClientEvent)
		if ok and ev and ev:IsA("RemoteEvent") then
			return ev
		end
	end
	if typeof(protocol.GetEvent) == "function" then
		local ok, ev = pcall(protocol.GetEvent)
		if ok and ev and ev:IsA("RemoteEvent") then
			return ev
		end
	end
	return nil
end

--[[
  Gun silent shot using Protocol.MakeClientShotPacket (ZombieShotClient.FireShot).
  origin/direction chosen by us → works with Silent Aim without moving camera.
  Server may still validate; in-game test required.
]]
local function performGunShot(model, preferHead)
	local root = CharacterController.GetRoot()
	local tool = getEquippedTool()
	if not root or not tool or not model then return false end

	local weaponName = tool:GetAttribute("RealName") or tool.Name
	local gunInfo = getWeaponGunInfo(weaponName) or {}
	local range = gunInfo.MaxBulletDistance or gunInfo.Range or 500

	local targetPart
	if preferHead then
		targetPart = getHeadPart(model)
	else
		targetPart = model:FindFirstChild("HumanoidRootPart") or getHeadPart(model)
	end
	if not targetPart then return false end

	local origin = root.Position + Vector3.new(0, 1.4, 0)
	local cam = Workspace.CurrentCamera
	if cam then
		origin = cam.CFrame.Position
	end

	local aimPos = targetPart.Position
	local delta = aimPos - origin
	if delta.Magnitude < 0.05 then return false end
	local direction = delta.Unit

	local zombieId = getZombieId(model)
	local hitRole = preferHead and "Head" or "Body"
	local shotId = nextShotId()

	local clipVal = 30
	local infoFolder = tool:FindFirstChild("Info")
	if infoFolder then
		local clip = infoFolder:FindFirstChild("Clip")
		if clip and clip:IsA("IntValue") then
			clipVal = clip.Value
		end
	end

	-- 1) Preferred: ZombieProtocol ClientShot packet (Silent Aim path)
	local protocol = getZombieProtocol()
	local event = getZombieClientEvent(protocol)
	if protocol and event and typeof(protocol.MakeClientShotPacket) == "function" then
		local packetData = {
			weaponId = weaponName,
			origin = origin,
			direction = direction,
			clientShotTime = Workspace:GetServerTimeNow() - 0.08,
			clientShotId = shotId,
			range = range,
			candidateZombieId = zombieId,
			candidatePart = hitRole,
			candidateHitboxName = hitRole,
			candidatePosition = aimPos,
			highRateWeapon = gunInfo.Automatic == true,
			clientClip = clipVal,
			clientAmmoActionId = shotId,
		}
		local ok = pcall(function()
			event:FireServer(protocol.MakeClientShotPacket(packetData))
		end)
		if ok then
			-- Local prediction (same BindableEvent as melee)
			local hp = getHitPredictionEvent()
			if hp and zombieId then
				local dmg = gunInfo.HeadshotDamage or gunInfo.Damage or 25
				if preferHead and gunInfo.HeadshotDamage then
					dmg = gunInfo.HeadshotDamage
				elseif preferHead then
					dmg = (gunInfo.Damage or 25) * 1.6
				end
				pcall(function()
					hp:Fire(
						zombieId, hitRole, aimPos, dmg, weaponName, shotId,
						gunInfo.Automatic == true, origin, direction, range, hitRole, "gun:" .. shotId
					)
				end)
			end
			return true
		end
	end

	-- 2) Fallback: FireProjectileWeapon with aim position (projectile weapons)
	local actions = GameStructure.ActionsFolder
		or (ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("Actions"))
	if actions then
		local fpw = actions:FindFirstChild("FireProjectileWeapon")
		if fpw and fpw:IsA("RemoteEvent") then
			local ok = pcall(function()
				fpw:FireServer(weaponName, aimPos, shotId, clipVal, shotId, math.random(1, 999999))
			end)
			if ok then return true end
		end
	end

	-- 3) Last resort: Tool:Activate() (uses camera aim — pair with AimBot)
	local ok2 = pcall(function()
		tool:Activate()
	end)
	return ok2
end

function ProjectAPI.AutoKill(enabled)
	Config.AutoKill = enabled
	if ProjectAPI._autoKillConn then
		ProjectAPI._autoKillConn:Disconnect()
		ProjectAPI._autoKillConn = nil
	end
	if not enabled then return end

	--[[
	  Auto Kill:
	  - Melee: MeleeRequest + HitPrediction
	  - Guns: Protocol FireShot toward Silent Aim target (no camera), else Activate
	]]
	ProjectAPI._autoKillConn = RunService.Heartbeat:Connect(function()
		if not Config.AutoKill then return end

		local tool = getEquippedTool()
		local interval = getFireInterval(tool)
		local now = tick()
		if now - ProjectAPI._lastAutoAt < interval then return end

		local hum = CharacterController.GetHumanoid()
		if not hum or hum.Health <= 0 then return end

		local model = ProjectAPI._silentTarget
		if not model or not model.Parent then
			model = select(1, findClosestZombie(500))
		end
		if not model then return end

		local preferHead = Config.AutoHeadshot or Config.SilentAim

		if not tool or isMeleeTool(tool) then
			local inRange = findClosestInRange(getMeleeRange() + 6)
			if inRange and performMeleeHit(inRange, preferHead) then
				ProjectAPI._lastAutoAt = now
			end
		else
			if performGunShot(model, preferHead) then
				ProjectAPI._lastAutoAt = now
			end
		end
	end)
end

function ProjectAPI.BringMobs(enabled)
	Config.BringMobs = enabled
	if ProjectAPI._bringConn then
		ProjectAPI._bringConn:Disconnect()
		ProjectAPI._bringConn = nil
	end
	if not enabled then return end

	-- Stable offsets (avoid random jitter every frame)
	local offsets = {}
	ProjectAPI._bringConn = RunService.Heartbeat:Connect(function()
		if not Config.BringMobs then return end
		local root = CharacterController.GetRoot()
		if not root then return end
		local i = 0
		for _, z in ipairs(getZombieModels()) do
			local zr = z:FindFirstChild("HumanoidRootPart")
			if zr then
				i = i + 1
				if not offsets[z] then
					local angle = (i * 0.7)
					offsets[z] = Vector3.new(math.cos(angle) * 4, 0, math.sin(angle) * 4)
				end
				zr.CFrame = CFrame.new(root.Position + offsets[z])
			end
		end
	end)
end

function ProjectAPI.AutoHeadshot(enabled)
	Config.AutoHeadshot = enabled
	-- Flag consumed by AimBot, KillAura, AutoKill, performMeleeHit
	if ProjectAPI._headshotConn then
		ProjectAPI._headshotConn:Disconnect()
		ProjectAPI._headshotConn = nil
	end
end

function ProjectAPI.KillAura(enabled)
	Config.KillAura = enabled
	if ProjectAPI._killAuraConn then
		ProjectAPI._killAuraConn:Disconnect()
		ProjectAPI._killAuraConn = nil
	end
	if not enabled then return end

	--[[
	  Kill Aura fixes:
	  - Separate timer (_lastAuraAt) so AutoKill does not block it
	  - Hits ONE closest target per cycle (server rejects multi-swing spam)
	  - Live folder lookup for zombies
	  - Direction toward target when SilentAim/AutoHeadshot enabled
	  - Configurable range (Config.KillAuraRange)
	]]
	ProjectAPI._killAuraConn = RunService.Heartbeat:Connect(function()
		if not Config.KillAura then return end

		local interval = Config.NoCooldown and 0.1 or 0.35
		local now = tick()
		if now - ProjectAPI._lastAuraAt < interval then return end

		local root = CharacterController.GetRoot()
		local hum = CharacterController.GetHumanoid()
		if not root or not hum or hum.Health <= 0 then return end

		local range = getMeleeRange()
		local target = findClosestInRange(range)
		if not target then return end

		if performMeleeHit(target, preferHeadHit()) then
			ProjectAPI._lastAuraAt = now
		end
	end)
end

--[[
  No Recoil (client):
  1) Humanoid.CameraOffset = 0
  2) HumanoidCameraShake.Reset() when available
  3) FPSViewport RecoilOffset / RecoilAngle / velocities zeroed
  Does NOT remove raycast Spread in the shot packet (server-side geometry).
]]
function ProjectAPI.NoRecoil(enabled)
	Config.NoRecoil = enabled
	if ProjectAPI._noRecoilConn then
		ProjectAPI._noRecoilConn:Disconnect()
		ProjectAPI._noRecoilConn = nil
	end
	if not enabled then return end

	ProjectAPI._noRecoilConn = RunService.RenderStepped:Connect(function()
		if not Config.NoRecoil then return end

		local hum = CharacterController.GetHumanoid()
		if hum then
			hum.CameraOffset = Vector3.zero
		end

		local shake = getHumanoidCameraShake()
		if shake then
			if typeof(shake.Reset) == "function" then
				pcall(function()
					shake.Reset()
				end)
			end
		end

		local fps = getFPSViewport()
		if fps then
			if fps.RecoilOffset ~= nil then fps.RecoilOffset = Vector3.zero end
			if fps.RecoilAngle ~= nil then fps.RecoilAngle = Vector3.zero end
			if fps.RecoilVelocity ~= nil then fps.RecoilVelocity = Vector3.zero end
			if fps.RecoilAngularVelocity ~= nil then fps.RecoilAngularVelocity = Vector3.zero end
			if fps.FireStack ~= nil then fps.FireStack = 0 end
			if typeof(fps.ResetTransientMotion) == "function" then
				pcall(function()
					fps.ResetTransientMotion()
				end)
			end
		end
	end)
end

function ProjectAPI.Cleanup()
	for _, key in ipairs({
		"_aimbotConn", "_autoKillConn", "_killAuraConn",
		"_bringConn", "_noCdConn", "_headshotConn", "_silentConn", "_noRecoilConn"
	}) do
		local c = ProjectAPI[key]
		if c then
			c:Disconnect()
			ProjectAPI[key] = nil
		end
	end
	if Config.Fly then
		CharacterController.ApplyFly(false)
	end
	if Config.Godmode then
		CharacterController.ApplyGodmode(false)
	end
end

-- ============================================================================
-- THEMES
-- ============================================================================
local Themes = {
	Dark = {
		Background   = Color3.fromRGB(13, 14, 20),
		Sidebar      = Color3.fromRGB(10, 11, 16),
		Panel        = Color3.fromRGB(20, 22, 31),
		Button       = Color3.fromRGB(28, 31, 43),
		ButtonHover  = Color3.fromRGB(38, 42, 58),
		Text         = Color3.fromRGB(230, 233, 245),
		SecondaryText= Color3.fromRGB(125, 132, 155),
		Border       = Color3.fromRGB(36, 40, 56),
		Accent       = Color3.fromRGB(124, 106, 255),
		ToggleOn     = Color3.fromRGB(124, 106, 255),
		ToggleOff    = Color3.fromRGB(44, 48, 66),
		TitleBar     = Color3.fromRGB(10, 11, 16),
		OnAccent     = Color3.fromRGB(255, 255, 255),
		Subtle       = Color3.fromRGB(32, 30, 62),
		Danger       = Color3.fromRGB(235, 70, 85),
	},
	White = {
		Background   = Color3.fromRGB(244, 245, 249),
		Sidebar      = Color3.fromRGB(235, 237, 244),
		Panel        = Color3.fromRGB(255, 255, 255),
		Button       = Color3.fromRGB(238, 240, 246),
		ButtonHover  = Color3.fromRGB(226, 229, 238),
		Text         = Color3.fromRGB(24, 26, 38),
		SecondaryText= Color3.fromRGB(105, 110, 130),
		Border       = Color3.fromRGB(216, 220, 232),
		Accent       = Color3.fromRGB(88, 80, 236),
		ToggleOn     = Color3.fromRGB(88, 80, 236),
		ToggleOff    = Color3.fromRGB(190, 195, 210),
		TitleBar     = Color3.fromRGB(250, 251, 253),
		OnAccent     = Color3.fromRGB(255, 255, 255),
		Subtle       = Color3.fromRGB(225, 226, 250),
		Danger       = Color3.fromRGB(225, 60, 75),
	},
	Shark = {
		Background   = Color3.fromRGB(9, 20, 33),
		Sidebar      = Color3.fromRGB(7, 15, 26),
		Panel        = Color3.fromRGB(14, 30, 47),
		Button       = Color3.fromRGB(19, 40, 62),
		ButtonHover  = Color3.fromRGB(27, 55, 84),
		Text         = Color3.fromRGB(215, 238, 250),
		SecondaryText= Color3.fromRGB(115, 155, 180),
		Border       = Color3.fromRGB(26, 54, 80),
		Accent       = Color3.fromRGB(0, 208, 230),
		ToggleOn     = Color3.fromRGB(0, 208, 230),
		ToggleOff    = Color3.fromRGB(32, 56, 80),
		TitleBar     = Color3.fromRGB(7, 14, 24),
		OnAccent     = Color3.fromRGB(6, 16, 26),
		Subtle       = Color3.fromRGB(10, 52, 72),
		Danger       = Color3.fromRGB(235, 70, 85),
	},
}

-- ============================================================================
-- UI STATE / REFERENCES
-- ============================================================================
local UI = {
	ScreenGui   = nil,
	MainFrame   = nil,
	TitleBar    = nil,
	Sidebar     = nil,
	Content     = nil,
	Pages       = {},
	NavButtons  = {},
	Themeable   = {}, -- {instance, property, themeKey}
	Toggles     = {}, -- name -> {button, label, state}
	CurrentPage = "MAIN",
	Visible     = true,
	Dragging    = false,
	DragStart   = nil,
	StartPos    = nil,
	ListeningKey= false,
	KeyLabel    = nil,
	StatusLabel = nil,
	Connections = {},
}

local function registerThemeable(inst, prop, themeKey)
	table.insert(UI.Themeable, {inst = inst, prop = prop, key = themeKey})
end

local function tween(obj, props, time, style, dir)
	local ti = TweenInfo.new(time or 0.18, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out)
	local tw = TweenService:Create(obj, ti, props)
	tw:Play()
	return tw
end

-- Visual state of the sidebar navigation (active / inactive)
local function refreshNav(animate)
	local t = Themes[Config.Theme]
	local dur = animate and 0.18 or 0
	for n, btn in pairs(UI.NavButtons) do
		local active = (n == UI.CurrentPage)
		local indicator = btn:FindFirstChild("Indicator")
		local title = btn:FindFirstChild("Title")
		local index = btn:FindFirstChild("Index")

		tween(btn, {
			BackgroundColor3 = t.Subtle,
			BackgroundTransparency = active and 0 or 1,
		}, dur)
		if indicator then
			tween(indicator, {
				BackgroundColor3 = t.Accent,
				Size = UDim2.new(0, 3, active and 0.55 or 0, 0),
			}, dur, Enum.EasingStyle.Quint)
		end
		if title then
			tween(title, {TextColor3 = active and t.Text or t.SecondaryText}, dur)
		end
		if index then
			tween(index, {TextColor3 = active and t.Accent or t.SecondaryText}, dur)
		end
	end
end

local function applyTheme(themeName)
	local t = Themes[themeName]
	if not t then return end
	Config.Theme = themeName
	for _, entry in ipairs(UI.Themeable) do
		if entry.inst and entry.inst.Parent then
			local ok = pcall(function()
				entry.inst[entry.prop] = t[entry.key]
			end)
		end
	end
	-- Refresh toggle visuals
	for name, data in pairs(UI.Toggles) do
		if data.button and data.button.Parent and data.apply then
			data.apply(false)
		end
	end
	refreshNav(false)
end

-- ============================================================================
-- UI FACTORY HELPERS
-- ============================================================================
local function createCorner(parent, radius)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius or 8)
	c.Parent = parent
	return c
end

local function createStroke(parent, color, thickness)
	local s = Instance.new("UIStroke")
	s.Color = color or Themes[Config.Theme].Border
	s.Thickness = thickness or 1
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = parent
	registerThemeable(s, "Color", "Border")
	return s
end

local function createPadding(parent, t, b, l, r)
	local p = Instance.new("UIPadding")
	p.PaddingTop = UDim.new(0, t or 0)
	p.PaddingBottom = UDim.new(0, b or 0)
	p.PaddingLeft = UDim.new(0, l or 0)
	p.PaddingRight = UDim.new(0, r or 0)
	p.Parent = parent
	return p
end

local function createListLayout(parent, padding, fillDir, hAlign, vAlign)
	local l = Instance.new("UIListLayout")
	l.Padding = UDim.new(0, padding or 6)
	l.FillDirection = fillDir or Enum.FillDirection.Vertical
	l.HorizontalAlignment = hAlign or Enum.HorizontalAlignment.Left
	l.VerticalAlignment = vAlign or Enum.VerticalAlignment.Top
	l.SortOrder = Enum.SortOrder.LayoutOrder
	l.Parent = parent
	return l
end

-- Small helper: thin solid rectangle (dividers, accent bars)
local function createRect(parent, name, size, pos, themeKey)
	local f = Instance.new("Frame")
	f.Name = name
	f.Size = size
	f.Position = pos or UDim2.new(0, 0, 0, 0)
	f.BackgroundColor3 = Themes[Config.Theme][themeKey]
	f.BorderSizePixel = 0
	f.Parent = parent
	registerThemeable(f, "BackgroundColor3", themeKey)
	return f
end

local function CreatePage(name)
	local page = Instance.new("ScrollingFrame")
	page.Name = "Page_" .. name
	page.Size = UDim2.new(1, 0, 1, 0)
	page.BackgroundTransparency = 1
	page.BorderSizePixel = 0
	page.ScrollBarThickness = 3
	page.ScrollBarImageColor3 = Themes[Config.Theme].Accent
	page.ScrollBarImageTransparency = 0.35
	page.ScrollingDirection = Enum.ScrollingDirection.Y
	page.CanvasSize = UDim2.new(0, 0, 0, 0)
	page.AutomaticCanvasSize = Enum.AutomaticSize.Y
	page.Visible = false
	page.Parent = UI.Content
	registerThemeable(page, "ScrollBarImageColor3", "Accent")
	createPadding(page, 14, 14, 16, 16)
	createListLayout(page, 8)
	UI.Pages[name] = page
	return page
end

local function CreateButton(parent, text, callback)
	local btn = Instance.new("TextButton")
	btn.Name = "Btn_" .. text:gsub("%s+", "")
	btn.Size = UDim2.new(1, 0, 0, 38)
	btn.BackgroundColor3 = Themes[Config.Theme].Button
	btn.Text = text
	btn.Font = Enum.Font.GothamMedium
	btn.TextSize = 13
	btn.TextXAlignment = Enum.TextXAlignment.Left
	btn.TextColor3 = Themes[Config.Theme].Text
	btn.AutoButtonColor = false
	btn.Parent = parent
	createCorner(btn, 8)
	local stroke = createStroke(btn)
	createPadding(btn, 0, 0, 16, 16)
	registerThemeable(btn, "BackgroundColor3", "Button")
	registerThemeable(btn, "TextColor3", "Text")

	local scale = Instance.new("UIScale")
	scale.Scale = 1
	scale.Parent = btn

	local chevron = Instance.new("TextLabel")
	chevron.Name = "Chevron"
	chevron.AnchorPoint = Vector2.new(1, 0.5)
	chevron.Position = UDim2.new(1, 0, 0.5, 0)
	chevron.Size = UDim2.new(0, 16, 0, 16)
	chevron.BackgroundTransparency = 1
	chevron.Text = "›"
	chevron.Font = Enum.Font.GothamBold
	chevron.TextSize = 20
	chevron.TextColor3 = Themes[Config.Theme].SecondaryText
	chevron.Parent = btn
	registerThemeable(chevron, "TextColor3", "SecondaryText")

	btn.MouseEnter:Connect(function()
		tween(btn, {BackgroundColor3 = Themes[Config.Theme].ButtonHover}, 0.12)
		tween(stroke, {Color = Themes[Config.Theme].Accent}, 0.12)
		tween(chevron, {TextColor3 = Themes[Config.Theme].Accent, Position = UDim2.new(1, 3, 0.5, 0)}, 0.12)
	end)
	btn.MouseLeave:Connect(function()
		tween(btn, {BackgroundColor3 = Themes[Config.Theme].Button}, 0.12)
		tween(stroke, {Color = Themes[Config.Theme].Border}, 0.12)
		tween(chevron, {TextColor3 = Themes[Config.Theme].SecondaryText, Position = UDim2.new(1, 0, 0.5, 0)}, 0.12)
	end)
	btn.MouseButton1Click:Connect(function()
		tween(scale, {Scale = 0.97}, 0.06)
		tween(btn, {BackgroundColor3 = Themes[Config.Theme].Accent, TextColor3 = Themes[Config.Theme].OnAccent}, 0.08)
		task.delay(0.1, function()
			if btn.Parent then
				tween(scale, {Scale = 1}, 0.14, Enum.EasingStyle.Back)
				tween(btn, {BackgroundColor3 = Themes[Config.Theme].Button, TextColor3 = Themes[Config.Theme].Text}, 0.14)
			end
		end)
		if callback then callback() end
	end)
	return btn
end

local function CreateToggle(parent, name, initial, onChange, description)
	local hasDesc = typeof(description) == "string" and description ~= ""
	local rowH = hasDesc and 56 or 40

	local row = Instance.new("Frame")
	row.Name = "ToggleRow_" .. name:gsub("%s+", "")
	row.Size = UDim2.new(1, 0, 0, rowH)
	row.BackgroundColor3 = Themes[Config.Theme].Panel
	row.BorderSizePixel = 0
	row.ClipsDescendants = true
	row.Parent = parent
	createCorner(row, 8)
	createStroke(row)
	registerThemeable(row, "BackgroundColor3", "Panel")

	-- Left status bar (lights up when the toggle is ON)
	local bar = Instance.new("Frame")
	bar.Name = "StatusBar"
	bar.Size = UDim2.new(0, 3, 1, 0)
	bar.Position = UDim2.new(0, 0, 0, 0)
	bar.BackgroundColor3 = Themes[Config.Theme].Accent
	bar.BackgroundTransparency = 1
	bar.BorderSizePixel = 0
	bar.Parent = row

	local label = Instance.new("TextLabel")
	label.Name = "Label"
	label.Size = UDim2.new(1, -120, 0, hasDesc and 18 or rowH)
	label.Position = UDim2.new(0, 16, 0, hasDesc and 10 or 0)
	label.BackgroundTransparency = 1
	label.Text = name
	label.Font = Enum.Font.GothamMedium
	label.TextSize = 13
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.TextYAlignment = Enum.TextYAlignment.Center
	label.TextColor3 = Themes[Config.Theme].Text
	label.Parent = row
	registerThemeable(label, "TextColor3", "Text")

	if hasDesc then
		local desc = Instance.new("TextLabel")
		desc.Name = "Description"
		desc.Size = UDim2.new(1, -120, 0, 16)
		desc.Position = UDim2.new(0, 16, 0, 30)
		desc.BackgroundTransparency = 1
		desc.Text = description
		desc.Font = Enum.Font.Gotham
		desc.TextSize = 11
		desc.TextXAlignment = Enum.TextXAlignment.Left
		desc.TextYAlignment = Enum.TextYAlignment.Top
		desc.TextColor3 = Themes[Config.Theme].SecondaryText
		desc.TextTruncate = Enum.TextTruncate.AtEnd
		desc.Parent = row
		registerThemeable(desc, "TextColor3", "SecondaryText")
	end

	-- ON / OFF mono text
	local stateText = Instance.new("TextLabel")
	stateText.Name = "State"
	stateText.Size = UDim2.new(0, 30, 0, 14)
	stateText.AnchorPoint = Vector2.new(1, 0.5)
	stateText.Position = UDim2.new(1, -62, 0.5, 0)
	stateText.BackgroundTransparency = 1
	stateText.Font = Enum.Font.Code
	stateText.TextSize = 11
	stateText.TextXAlignment = Enum.TextXAlignment.Right
	stateText.Text = "OFF"
	stateText.TextColor3 = Themes[Config.Theme].SecondaryText
	stateText.Parent = row

	-- Switch (track + knob)
	local toggle = Instance.new("TextButton")
	toggle.Name = "Toggle"
	toggle.Size = UDim2.new(0, 42, 0, 22)
	toggle.Position = UDim2.new(1, -54, 0.5, -11)
	toggle.BackgroundColor3 = Themes[Config.Theme].ToggleOff
	toggle.Text = ""
	toggle.AutoButtonColor = false
	toggle.Parent = row
	createCorner(toggle, 11)

	local knob = Instance.new("Frame")
	knob.Name = "Knob"
	knob.Size = UDim2.new(0, 16, 0, 16)
	knob.Position = UDim2.new(0, 3, 0.5, -8)
	knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
	knob.BorderSizePixel = 0
	knob.Parent = toggle
	createCorner(knob, 8)

	local offPos = UDim2.new(0, 3, 0.5, -8)
	local onPos  = UDim2.new(1, -19, 0.5, -8)

	local state = not not initial

	local function apply(animated)
		local t = Themes[Config.Theme]
		local dur = animated and 0.16 or 0
		tween(toggle, {BackgroundColor3 = state and t.ToggleOn or t.ToggleOff}, dur)
		tween(knob, {Position = state and onPos or offPos}, dur, Enum.EasingStyle.Quint)
		tween(bar, {
			BackgroundColor3 = t.Accent,
			BackgroundTransparency = state and 0 or 1,
		}, dur)
		stateText.Text = state and "ON" or "OFF"
		tween(stateText, {TextColor3 = state and t.Accent or t.SecondaryText}, dur)
	end

	UI.Toggles[name] = {button = toggle, label = label, state = state, row = row, apply = apply}
	apply(false)

	toggle.MouseButton1Click:Connect(function()
		state = not state
		UI.Toggles[name].state = state
		apply(true)
		if onChange then onChange(state) end
	end)

	row.MouseEnter:Connect(function()
		tween(row, {BackgroundColor3 = Themes[Config.Theme].Button}, 0.12)
	end)
	row.MouseLeave:Connect(function()
		tween(row, {BackgroundColor3 = Themes[Config.Theme].Panel}, 0.12)
	end)

	return row, toggle
end

local function CreateNumberBox(parent, name, initial, onChange)
	local row = Instance.new("Frame")
	row.Name = "NumRow_" .. name
	row.Size = UDim2.new(1, 0, 0, 44)
	row.BackgroundColor3 = Themes[Config.Theme].Panel
	row.BorderSizePixel = 0
	row.Parent = parent
	createCorner(row, 8)
	createStroke(row)
	registerThemeable(row, "BackgroundColor3", "Panel")

	local label = Instance.new("TextLabel")
	label.Size = UDim2.new(0.5, -8, 1, 0)
	label.Position = UDim2.new(0, 16, 0, 0)
	label.BackgroundTransparency = 1
	label.Text = name
	label.Font = Enum.Font.GothamMedium
	label.TextSize = 13
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.TextColor3 = Themes[Config.Theme].Text
	label.Parent = row
	registerThemeable(label, "TextColor3", "Text")

	local box = Instance.new("TextBox")
	box.Name = "ValueBox"
	box.Size = UDim2.new(0, 84, 0, 28)
	box.Position = UDim2.new(1, -98, 0.5, -14)
	box.BackgroundColor3 = Themes[Config.Theme].Background
	box.Text = tostring(initial)
	box.Font = Enum.Font.Code
	box.TextSize = 13
	box.TextColor3 = Themes[Config.Theme].Text
	box.PlaceholderColor3 = Themes[Config.Theme].SecondaryText
	box.ClearTextOnFocus = false
	box.Parent = row
	createCorner(box, 6)
	local boxStroke = createStroke(box)
	registerThemeable(box, "BackgroundColor3", "Background")
	registerThemeable(box, "TextColor3", "Text")

	box.Focused:Connect(function()
		tween(boxStroke, {Color = Themes[Config.Theme].Accent}, 0.12)
	end)

	local lastValid = tonumber(initial) or 0
	box.FocusLost:Connect(function()
		tween(boxStroke, {Color = Themes[Config.Theme].Border}, 0.12)
		local n = tonumber(box.Text)
		if n and n >= 0 and n <= 500 then
			lastValid = math.floor(n)
			box.Text = tostring(lastValid)
			if onChange then onChange(lastValid) end
		else
			box.Text = tostring(lastValid)
		end
	end)

	return row, box
end

local function CreateSectionHeader(parent, text)
	local holder = Instance.new("Frame")
	holder.Name = "Section_" .. text
	holder.Size = UDim2.new(1, 0, 0, 26)
	holder.BackgroundTransparency = 1
	holder.Parent = parent

	local slashes = Instance.new("TextLabel")
	slashes.Name = "Slashes"
	slashes.Size = UDim2.new(0, 16, 1, 0)
	slashes.BackgroundTransparency = 1
	slashes.Text = "//"
	slashes.Font = Enum.Font.Code
	slashes.TextSize = 12
	slashes.TextXAlignment = Enum.TextXAlignment.Left
	slashes.TextColor3 = Themes[Config.Theme].Accent
	slashes.Parent = holder
	registerThemeable(slashes, "TextColor3", "Accent")

	local lbl = Instance.new("TextLabel")
	lbl.Name = "Title"
	lbl.Size = UDim2.new(1, -20, 1, 0)
	lbl.Position = UDim2.new(0, 20, 0, 0)
	lbl.BackgroundTransparency = 1
	lbl.Text = text
	lbl.Font = Enum.Font.GothamBold
	lbl.TextSize = 11
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.TextColor3 = Themes[Config.Theme].SecondaryText
	lbl.Parent = holder
	registerThemeable(lbl, "TextColor3", "SecondaryText")

	local line = Instance.new("Frame")
	line.Name = "Line"
	line.BackgroundColor3 = Themes[Config.Theme].Border
	line.BorderSizePixel = 0
	line.Parent = holder
	registerThemeable(line, "BackgroundColor3", "Border")

	local function placeLine()
		local startX = 20 + lbl.TextBounds.X + 10
		line.Position = UDim2.new(0, startX, 0.5, 0)
		line.Size = UDim2.new(1, -startX, 0, 1)
	end
	placeLine()
	lbl:GetPropertyChangedSignal("TextBounds"):Connect(placeLine)

	return holder
end

-- ============================================================================
-- PAGE SWITCH
-- ============================================================================
function switchPage(name)
	local changed = UI.CurrentPage ~= name
	UI.CurrentPage = name
	for n, page in pairs(UI.Pages) do
		local show = (n == name)
		page.Visible = show
		if show and changed then
			page.Position = UDim2.new(0, 0, 0, 10)
			tween(page, {Position = UDim2.new(0, 0, 0, 0)}, 0.22, Enum.EasingStyle.Quint)
		end
	end
	refreshNav(true)
end

-- ============================================================================
-- BUILD UI
-- ============================================================================
local function buildUI()
	local theme = Themes[Config.Theme]

	local sg = Instance.new("ScreenGui")
	sg.Name = Config.UIName
	sg.ResetOnSpawn = false
	sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	sg.IgnoreGuiInset = true
	sg.Parent = PlayerGui
	UI.ScreenGui = sg

	local main = Instance.new("Frame")
	main.Name = "MainWindow"
	main.Size = Config.WindowSize
	main.Position = UDim2.new(0.5, -Config.WindowSize.X.Offset/2, 0.5, -Config.WindowSize.Y.Offset/2)
	main.BackgroundColor3 = theme.Background
	main.BorderSizePixel = 0
	main.ClipsDescendants = true
	main.Parent = sg
	UI.MainFrame = main
	createCorner(main, 12)
	createStroke(main)
	registerThemeable(main, "BackgroundColor3", "Background")

	-- Title bar
	local titleBar = Instance.new("Frame")
	titleBar.Name = "TitleBar"
	titleBar.Size = UDim2.new(1, 0, 0, 46)
	titleBar.BackgroundColor3 = theme.TitleBar
	titleBar.BorderSizePixel = 0
	titleBar.Parent = main
	UI.TitleBar = titleBar
	registerThemeable(titleBar, "BackgroundColor3", "TitleBar")

	createRect(titleBar, "Divider", UDim2.new(1, 0, 0, 1), UDim2.new(0, 0, 1, -1), "Border")

	-- Accent line (fades to the right)
	local accentLine = createRect(main, "AccentLine", UDim2.new(1, 0, 0, 2), UDim2.new(0, 0, 0, 0), "Accent")
	accentLine.ZIndex = 5
	local accentFade = Instance.new("UIGradient")
	accentFade.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0),
		NumberSequenceKeypoint.new(0.55, 0.35),
		NumberSequenceKeypoint.new(1, 1),
	})
	accentFade.Parent = accentLine

	-- Logo badge
	local logo = Instance.new("Frame")
	logo.Name = "Logo"
	logo.Size = UDim2.new(0, 28, 0, 28)
	logo.Position = UDim2.new(0, 14, 0.5, -14)
	logo.BackgroundColor3 = theme.Accent
	logo.BorderSizePixel = 0
	logo.Parent = titleBar
	createCorner(logo, 8)
	registerThemeable(logo, "BackgroundColor3", "Accent")
	local logoGrad = Instance.new("UIGradient")
	logoGrad.Rotation = 45
	logoGrad.Color = ColorSequence.new(Color3.fromRGB(255, 255, 255), Color3.fromRGB(165, 165, 180))
	logoGrad.Parent = logo

	local logoText = Instance.new("TextLabel")
	logoText.Size = UDim2.new(1, 0, 1, 0)
	logoText.BackgroundTransparency = 1
	logoText.Text = "S"
	logoText.Font = Enum.Font.GothamBlack
	logoText.TextSize = 15
	logoText.TextColor3 = theme.OnAccent
	logoText.Parent = logo
	registerThemeable(logoText, "TextColor3", "OnAccent")

	local titleLabel = Instance.new("TextLabel")
	titleLabel.Size = UDim2.new(0, 170, 0, 18)
	titleLabel.Position = UDim2.new(0, 54, 0, 6)
	titleLabel.BackgroundTransparency = 1
	titleLabel.Text = "SHARK HUB"
	titleLabel.Font = Enum.Font.GothamBold
	titleLabel.TextSize = 14
	titleLabel.TextXAlignment = Enum.TextXAlignment.Left
	titleLabel.TextColor3 = theme.Text
	titleLabel.Parent = titleBar
	registerThemeable(titleLabel, "TextColor3", "Text")

	-- Version pill (follows the title width)
	local pill = Instance.new("Frame")
	pill.Name = "VersionPill"
	pill.Size = UDim2.new(0, 26, 0, 14)
	pill.Position = UDim2.new(0, 54 + 80, 0, 8)
	pill.BackgroundColor3 = theme.Subtle
	pill.BorderSizePixel = 0
	pill.Parent = titleBar
	createCorner(pill, 7)
	registerThemeable(pill, "BackgroundColor3", "Subtle")

	local pillText = Instance.new("TextLabel")
	pillText.Size = UDim2.new(1, 0, 1, 0)
	pillText.BackgroundTransparency = 1
	pillText.Text = "v2"
	pillText.Font = Enum.Font.Code
	pillText.TextSize = 10
	pillText.TextColor3 = theme.Accent
	pillText.Parent = pill
	registerThemeable(pillText, "TextColor3", "Accent")

	local function placePill()
		pill.Position = UDim2.new(0, 54 + titleLabel.TextBounds.X + 8, 0, 8)
	end
	placePill()
	titleLabel:GetPropertyChangedSignal("TextBounds"):Connect(placePill)

	local subLabel = Instance.new("TextLabel")
	subLabel.Size = UDim2.new(1, -120, 0, 12)
	subLabel.Position = UDim2.new(0, 54, 0, 25)
	subLabel.BackgroundTransparency = 1
	subLabel.Text = "Zombie Attack  ·  Admin / Test"
	subLabel.Font = Enum.Font.Code
	subLabel.TextSize = 10
	subLabel.TextXAlignment = Enum.TextXAlignment.Left
	subLabel.TextColor3 = theme.SecondaryText
	subLabel.Parent = titleBar
	registerThemeable(subLabel, "TextColor3", "SecondaryText")

	local closeBtn = Instance.new("TextButton")
	closeBtn.Name = "Close"
	closeBtn.Size = UDim2.new(0, 28, 0, 28)
	closeBtn.Position = UDim2.new(1, -40, 0.5, -14)
	closeBtn.BackgroundColor3 = theme.Button
	closeBtn.BackgroundTransparency = 1
	closeBtn.Text = "✕"
	closeBtn.Font = Enum.Font.GothamBold
	closeBtn.TextSize = 13
	closeBtn.TextColor3 = theme.SecondaryText
	closeBtn.AutoButtonColor = false
	closeBtn.Parent = titleBar
	createCorner(closeBtn, 8)
	registerThemeable(closeBtn, "BackgroundColor3", "Button")
	registerThemeable(closeBtn, "TextColor3", "SecondaryText")
	closeBtn.MouseEnter:Connect(function()
		tween(closeBtn, {
			BackgroundColor3 = Themes[Config.Theme].Danger,
			BackgroundTransparency = 0,
			TextColor3 = Color3.fromRGB(255, 255, 255),
		}, 0.12)
	end)
	closeBtn.MouseLeave:Connect(function()
		tween(closeBtn, {
			BackgroundColor3 = Themes[Config.Theme].Button,
			BackgroundTransparency = 1,
			TextColor3 = Themes[Config.Theme].SecondaryText,
		}, 0.12)
	end)
	closeBtn.MouseButton1Click:Connect(function()
		UI.Visible = false
		tween(main, {Size = UDim2.new(0, 0, 0, 0)}, 0.2)
		task.delay(0.2, function()
			if main then main.Visible = false end
		end)
	end)

	-- Dragging
	titleBar.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			UI.Dragging = true
			UI.DragStart = input.Position
			UI.StartPos = main.Position
		end
	end)
	titleBar.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			UI.Dragging = false
		end
	end)
	table.insert(UI.Connections, UserInputService.InputChanged:Connect(function(input)
		if UI.Dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			local delta = input.Position - UI.DragStart
			main.Position = UDim2.new(
				UI.StartPos.X.Scale, UI.StartPos.X.Offset + delta.X,
				UI.StartPos.Y.Scale, UI.StartPos.Y.Offset + delta.Y
			)
		end
	end))

	-- Body
	local body = Instance.new("Frame")
	body.Name = "Body"
	body.Size = UDim2.new(1, 0, 1, -46)
	body.Position = UDim2.new(0, 0, 0, 46)
	body.BackgroundTransparency = 1
	body.Parent = main

	-- Sidebar
	local sidebar = Instance.new("Frame")
	sidebar.Name = "Sidebar"
	sidebar.Size = UDim2.new(0, 150, 1, 0)
	sidebar.BackgroundColor3 = theme.Sidebar
	sidebar.BorderSizePixel = 0
	sidebar.Parent = body
	UI.Sidebar = sidebar
	registerThemeable(sidebar, "BackgroundColor3", "Sidebar")
	createRect(sidebar, "Divider", UDim2.new(0, 1, 1, 0), UDim2.new(1, -1, 0, 0), "Border")

	local navCaption = Instance.new("TextLabel")
	navCaption.Name = "NavCaption"
	navCaption.Size = UDim2.new(1, -16, 0, 12)
	navCaption.Position = UDim2.new(0, 16, 0, 16)
	navCaption.BackgroundTransparency = 1
	navCaption.Text = "NAVIGATION"
	navCaption.Font = Enum.Font.Code
	navCaption.TextSize = 10
	navCaption.TextXAlignment = Enum.TextXAlignment.Left
	navCaption.TextColor3 = theme.SecondaryText
	navCaption.Parent = sidebar
	registerThemeable(navCaption, "TextColor3", "SecondaryText")

	local navHolder = Instance.new("Frame")
	navHolder.Name = "NavHolder"
	navHolder.Size = UDim2.new(1, -20, 1, -110)
	navHolder.Position = UDim2.new(0, 10, 0, 36)
	navHolder.BackgroundTransparency = 1
	navHolder.Parent = sidebar
	createListLayout(navHolder, 4)

	-- Content area
	local content = Instance.new("Frame")
	content.Name = "Content"
	content.Size = UDim2.new(1, -150, 1, 0)
	content.Position = UDim2.new(0, 150, 0, 0)
	content.BackgroundTransparency = 1
	content.ClipsDescendants = true
	content.Parent = body
	UI.Content = content

	-- Pages
	CreatePage("MAIN")
	CreatePage("PLAYER")
	CreatePage("MISC")

	-- Nav buttons
	local function makeNav(name, order)
		local btn = Instance.new("TextButton")
		btn.Name = "Nav_" .. name
		btn.Size = UDim2.new(1, 0, 0, 36)
		btn.BackgroundColor3 = theme.Subtle
		btn.BackgroundTransparency = 1
		btn.Text = ""
		btn.LayoutOrder = order
		btn.AutoButtonColor = false
		btn.Parent = navHolder
		createCorner(btn, 8)
		UI.NavButtons[name] = btn

		local indicator = Instance.new("Frame")
		indicator.Name = "Indicator"
		indicator.AnchorPoint = Vector2.new(0, 0.5)
		indicator.Size = UDim2.new(0, 3, 0, 0)
		indicator.Position = UDim2.new(0, 0, 0.5, 0)
		indicator.BackgroundColor3 = theme.Accent
		indicator.BorderSizePixel = 0
		indicator.Parent = btn
		createCorner(indicator, 2)

		local title = Instance.new("TextLabel")
		title.Name = "Title"
		title.Size = UDim2.new(1, -50, 1, 0)
		title.Position = UDim2.new(0, 14, 0, 0)
		title.BackgroundTransparency = 1
		title.Text = name
		title.Font = Enum.Font.GothamMedium
		title.TextSize = 12
		title.TextXAlignment = Enum.TextXAlignment.Left
		title.TextColor3 = theme.SecondaryText
		title.Parent = btn

		local index = Instance.new("TextLabel")
		index.Name = "Index"
		index.Size = UDim2.new(0, 24, 1, 0)
		index.Position = UDim2.new(1, -32, 0, 0)
		index.BackgroundTransparency = 1
		index.Text = string.format("%02d", order)
		index.Font = Enum.Font.Code
		index.TextSize = 10
		index.TextXAlignment = Enum.TextXAlignment.Right
		index.TextColor3 = theme.SecondaryText
		index.Parent = btn

		btn.MouseEnter:Connect(function()
			if UI.CurrentPage ~= name then
				tween(btn, {BackgroundColor3 = Themes[Config.Theme].Subtle, BackgroundTransparency = 0.6}, 0.12)
				tween(title, {TextColor3 = Themes[Config.Theme].Text}, 0.12)
			end
		end)
		btn.MouseLeave:Connect(function()
			refreshNav(true)
		end)

		btn.MouseButton1Click:Connect(function()
			switchPage(name)
		end)
		return btn
	end

	makeNav("MAIN", 1)
	makeNav("PLAYER", 2)
	makeNav("MISC", 3)

	-- Status / key hint at bottom of sidebar
	local keyCaption = Instance.new("TextLabel")
	keyCaption.Name = "KeyCaption"
	keyCaption.AnchorPoint = Vector2.new(0, 1)
	keyCaption.Size = UDim2.new(1, -16, 0, 12)
	keyCaption.Position = UDim2.new(0, 16, 1, -54)
	keyCaption.BackgroundTransparency = 1
	keyCaption.Text = "TOGGLE KEY"
	keyCaption.Font = Enum.Font.Code
	keyCaption.TextSize = 10
	keyCaption.TextXAlignment = Enum.TextXAlignment.Left
	keyCaption.TextColor3 = theme.SecondaryText
	keyCaption.Parent = sidebar
	registerThemeable(keyCaption, "TextColor3", "SecondaryText")

	local keyHint = Instance.new("TextButton")
	keyHint.Name = "ToggleKeyBtn"
	keyHint.AnchorPoint = Vector2.new(0, 1)
	keyHint.Size = UDim2.new(1, -20, 0, 34)
	keyHint.Position = UDim2.new(0, 10, 1, -12)
	keyHint.BackgroundColor3 = theme.Panel
	keyHint.Text = "Key: CTRL"
	keyHint.Font = Enum.Font.Code
	keyHint.TextSize = 12
	keyHint.TextColor3 = theme.Text
	keyHint.LayoutOrder = 99
	keyHint.AutoButtonColor = false
	keyHint.Parent = sidebar
	createCorner(keyHint, 8)
	local keyStroke = createStroke(keyHint)
	registerThemeable(keyHint, "BackgroundColor3", "Panel")
	registerThemeable(keyHint, "TextColor3", "Text")
	UI.KeyLabel = keyHint

	keyHint.MouseEnter:Connect(function()
		tween(keyStroke, {Color = Themes[Config.Theme].Accent}, 0.12)
	end)
	keyHint.MouseLeave:Connect(function()
		tween(keyStroke, {Color = Themes[Config.Theme].Border}, 0.12)
	end)

	keyHint.MouseButton1Click:Connect(function()
		if UI.ListeningKey then return end
		UI.ListeningKey = true
		keyHint.Text = "Press key..."
		local conn
		conn = UserInputService.InputBegan:Connect(function(input, gpe)
			if input.UserInputType == Enum.UserInputType.Keyboard then
				Config.ToggleKey = input.KeyCode
				local name = input.KeyCode.Name
				if name == "LeftControl" or name == "RightControl" then
					name = "CTRL"
				end
				keyHint.Text = "Key: " .. name
				UI.ListeningKey = false
				conn:Disconnect()
			end
		end)
	end)

	-- ========== MAIN PAGE ==========
	local mainPage = UI.Pages.MAIN
	CreateSectionHeader(mainPage, "COMBAT / TEST TOGGLES")

	CreateToggle(mainPage, "AimBot", Config.AimBot, function(v)
		ProjectAPI.AimBot(v)
	end, "Suaviza a câmera em direção ao zombie mais próximo.")

	CreateToggle(mainPage, "Silent Aim", Config.SilentAim, function(v)
		ProjectAPI.SilentAim(v)
	end, "Melee + armas: mira no alvo sem mover a câmera (Protocol FireShot).")

	CreateToggle(mainPage, "No Cooldown", Config.NoCooldown, function(v)
		ProjectAPI.NoCooldown(v)
	end, "Reduz intervalos e reabastece Clip/Reserve no client.")

	CreateToggle(mainPage, "No Recoil", Config.NoRecoil, function(v)
		ProjectAPI.NoRecoil(v)
	end, "Zera shake de câmera e recoil do viewmodel (FPSViewport).")

	CreateToggle(mainPage, "Auto Kill Zombies", Config.AutoKill, function(v)
		ProjectAPI.AutoKill(v)
	end, "Melee e armas: atira no alvo automaticamente (GunData + FireShot).")

	CreateToggle(mainPage, "Bring All Mobs", Config.BringMobs, function(v)
		ProjectAPI.BringMobs(v)
	end, "Aproxima visuais client-side dos zombies.")

	CreateToggle(mainPage, "Auto Headshot", Config.AutoHeadshot, function(v)
		ProjectAPI.AutoHeadshot(v)
	end, "Prioriza hitbox Head em melee, Silent Aim e Auto Kill.")

	CreateToggle(mainPage, "Kill Aura", Config.KillAura, function(v)
		ProjectAPI.KillAura(v)
	end, "MeleeRequest no zombie mais próximo dentro do alcance.")

	CreateSectionHeader(mainPage, "ROUND INFO (read-only)")

	-- Telemetry card (terminal style)
	local infoCard = Instance.new("Frame")
	infoCard.Name = "RoundInfoCard"
	infoCard.Size = UDim2.new(1, 0, 0, 104)
	infoCard.BackgroundColor3 = theme.Panel
	infoCard.BorderSizePixel = 0
	infoCard.ClipsDescendants = true
	infoCard.Parent = mainPage
	createCorner(infoCard, 8)
	createStroke(infoCard)
	registerThemeable(infoCard, "BackgroundColor3", "Panel")

	local infoHeader = Instance.new("Frame")
	infoHeader.Name = "Header"
	infoHeader.Size = UDim2.new(1, 0, 0, 26)
	infoHeader.BackgroundColor3 = theme.Sidebar
	infoHeader.BorderSizePixel = 0
	infoHeader.Parent = infoCard
	registerThemeable(infoHeader, "BackgroundColor3", "Sidebar")
	createRect(infoHeader, "Divider", UDim2.new(1, 0, 0, 1), UDim2.new(0, 0, 1, -1), "Border")

	local infoTitle = Instance.new("TextLabel")
	infoTitle.Size = UDim2.new(0.6, 0, 1, 0)
	infoTitle.Position = UDim2.new(0, 12, 0, 0)
	infoTitle.BackgroundTransparency = 1
	infoTitle.Text = "round.telemetry"
	infoTitle.Font = Enum.Font.Code
	infoTitle.TextSize = 11
	infoTitle.TextXAlignment = Enum.TextXAlignment.Left
	infoTitle.TextColor3 = theme.SecondaryText
	infoTitle.Parent = infoHeader
	registerThemeable(infoTitle, "TextColor3", "SecondaryText")

	local liveDot = Instance.new("Frame")
	liveDot.Name = "LiveDot"
	liveDot.Size = UDim2.new(0, 6, 0, 6)
	liveDot.Position = UDim2.new(1, -52, 0.5, -3)
	liveDot.BackgroundColor3 = theme.ToggleOn
	liveDot.BorderSizePixel = 0
	liveDot.Parent = infoHeader
	createCorner(liveDot, 3)
	registerThemeable(liveDot, "BackgroundColor3", "Accent")
	TweenService:Create(
		liveDot,
		TweenInfo.new(0.9, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true),
		{BackgroundTransparency = 0.75}
	):Play()

	local liveText = Instance.new("TextLabel")
	liveText.Size = UDim2.new(0, 34, 1, 0)
	liveText.Position = UDim2.new(1, -42, 0, 0)
	liveText.BackgroundTransparency = 1
	liveText.Text = "LIVE"
	liveText.Font = Enum.Font.Code
	liveText.TextSize = 10
	liveText.TextXAlignment = Enum.TextXAlignment.Left
	liveText.TextColor3 = theme.SecondaryText
	liveText.Parent = infoHeader
	registerThemeable(liveText, "TextColor3", "SecondaryText")

	local infoLabel = Instance.new("TextLabel")
	infoLabel.Name = "RoundInfoLabel"
	infoLabel.Size = UDim2.new(1, 0, 1, -26)
	infoLabel.Position = UDim2.new(0, 0, 0, 26)
	infoLabel.BackgroundTransparency = 1
	infoLabel.TextColor3 = theme.Text
	infoLabel.Font = Enum.Font.Code
	infoLabel.TextSize = 12
	infoLabel.LineHeight = 1.3
	infoLabel.TextXAlignment = Enum.TextXAlignment.Left
	infoLabel.TextYAlignment = Enum.TextYAlignment.Top
	infoLabel.Text = "Loading..."
	infoLabel.Parent = infoCard
	createPadding(infoLabel, 8, 8, 12, 12)
	registerThemeable(infoLabel, "TextColor3", "Text")
	UI.StatusLabel = infoLabel

	-- ========== PLAYER PAGE ==========
	local playerPage = UI.Pages.PLAYER
	CreateSectionHeader(playerPage, "MOVEMENT")

	CreateNumberBox(playerPage, "Walk Speed", Config.WalkSpeed, function(v)
		CharacterController.SetWalkSpeed(v)
	end)

	CreateToggle(playerPage, "NoClip", Config.NoClip, function(v)
		CharacterController.ApplyNoClip(v)
	end, "Desativa colisão das parts do personagem enquanto ativo.")

	CreateToggle(playerPage, "Fly", Config.Fly, function(v)
		CharacterController.ApplyFly(v)
	end, "Voo livre: WASD + Space sobe + Shift desce.")

	CreateNumberBox(playerPage, "Fly Speed", Config.FlySpeed, function(v)
		Config.FlySpeed = math.clamp(v, 1, 300)
	end)

	CreateToggle(playerPage, "Godmode", Config.Godmode, function(v)
		CharacterController.ApplyGodmode(v)
	end, "Tenta restaurar vida no cliente. Dano de zombie é validado no servidor.")

	CreateSectionHeader(playerPage, "TELEPORT")
	CreateButton(playerPage, "TELEPORT TO SAFEZONE", function()
		local ok, msg = TeleportToSafeZone()
		if UI.StatusLabel then
			-- brief feedback via console; optional UI toast could be added
			print("[ZombieAttackProjectUI]", msg)
		end
	end)

	-- ========== MISC PAGE ==========
	local miscPage = UI.Pages.MISC
	CreateSectionHeader(miscPage, "THEMES")

	local function themeBtn(name)
		local b = CreateButton(miscPage, "Theme: " .. name, function()
			applyTheme(name)
		end)
		-- Color swatch preview replaces the chevron
		local chev = b:FindFirstChild("Chevron")
		if chev then chev.Visible = false end
		local swatch = Instance.new("Frame")
		swatch.Name = "Swatch"
		swatch.AnchorPoint = Vector2.new(1, 0.5)
		swatch.Size = UDim2.new(0, 14, 0, 14)
		swatch.Position = UDim2.new(1, 0, 0.5, 0)
		swatch.BackgroundColor3 = Themes[name].Accent
		swatch.BorderSizePixel = 0
		swatch.Parent = b
		createCorner(swatch, 7)
		local ring = Instance.new("UIStroke")
		ring.Color = Themes[name].Border
		ring.Thickness = 2
		ring.Parent = swatch
	end
	themeBtn("Dark")
	themeBtn("White")
	themeBtn("Shark")

	CreateSectionHeader(miscPage, "INFO")

	local aboutCard = Instance.new("Frame")
	aboutCard.Name = "AboutCard"
	aboutCard.Size = UDim2.new(1, 0, 0, 0)
	aboutCard.AutomaticSize = Enum.AutomaticSize.Y
	aboutCard.BackgroundColor3 = theme.Panel
	aboutCard.BorderSizePixel = 0
	aboutCard.ClipsDescendants = true
	aboutCard.Parent = miscPage
	createCorner(aboutCard, 8)
	createStroke(aboutCard)
	registerThemeable(aboutCard, "BackgroundColor3", "Panel")
	createRect(aboutCard, "AccentBar", UDim2.new(0, 3, 1, 0), UDim2.new(0, 0, 0, 0), "Accent")
	createPadding(aboutCard, 10, 10, 16, 12)

	local about = Instance.new("TextLabel")
	about.Size = UDim2.new(1, 0, 0, 0)
	about.AutomaticSize = Enum.AutomaticSize.Y
	about.BackgroundTransparency = 1
	about.TextColor3 = theme.SecondaryText
	about.Font = Enum.Font.Gotham
	about.TextSize = 11
	about.LineHeight = 1.25
	about.TextWrapped = true
	about.TextXAlignment = Enum.TextXAlignment.Left
	about.TextYAlignment = Enum.TextYAlignment.Top
	about.Text = "Shark Hub v2 — admin/teste.\nSilent Aim + Auto Kill: melee (HitPrediction) e armas (ZombieProtocol FireShot).\nNo Recoil: HumanoidCameraShake + FPSViewport.\nGodmode: restore local (server ainda valida dano)."
	about.Parent = aboutCard
	registerThemeable(about, "TextColor3", "SecondaryText")

	-- Initial page
	switchPage("MAIN")
	applyTheme(Config.Theme)
end

-- ============================================================================
-- INPUT SYSTEM (single connection, rebindable key)
-- ============================================================================
local function setupInput()
	table.insert(UI.Connections, UserInputService.InputBegan:Connect(function(input, gameProcessed)
		if UI.ListeningKey then return end
		if gameProcessed then return end
		if input.KeyCode == Config.ToggleKey then
			UI.Visible = not UI.Visible
			local main = UI.MainFrame
			if not main then return end
			if UI.Visible then
				main.Visible = true
				main.Size = UDim2.new(0, Config.WindowSize.X.Offset * 0.92, 0, Config.WindowSize.Y.Offset * 0.92)
				main.BackgroundTransparency = 0.35
				tween(main, {
					Size = Config.WindowSize,
					BackgroundTransparency = 0,
				}, 0.28, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
			else
				tween(main, {
					Size = UDim2.new(0, Config.WindowSize.X.Offset * 0.92, 0, Config.WindowSize.Y.Offset * 0.92),
					BackgroundTransparency = 0.4,
				}, 0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.In)
				task.delay(0.16, function()
					if main and not UI.Visible then
						main.Visible = false
						main.BackgroundTransparency = 0
						main.Size = Config.WindowSize
					end
				end)
			end
		end
	end))
end

-- ============================================================================
-- ROUND INFO LIVE UPDATE
-- ============================================================================
local function setupRoundInfoUpdater()
	task.spawn(function()
		while UI.ScreenGui and UI.ScreenGui.Parent do
			if UI.StatusLabel and UI.StatusLabel.Parent then
				local wave   = RoundInfo.GetWave() or "?"
				local alive  = RoundInfo.GetAliveZombies() or "?"
				local global = RoundInfo.GetGlobalAliveZombies() or "?"
				local pcs    = RoundInfo.GetPlayerCount() or "?"
				local state  = RoundInfo.GetRoundState() or RoundInfo.GetState() or "?"
				local diff   = RoundInfo.GetDifficulty() or "?"
				local remain = RoundInfo.GetZombiesRemaining() or alive
				UI.StatusLabel.Text = string.format(
					"Wave: %s  |  State: %s\nAlive: %s  |  Global: %s  |  Remaining: %s\nPlayers: %s  |  Diff: %s",
					tostring(wave), tostring(state),
					tostring(alive), tostring(global), tostring(remain),
					tostring(pcs), tostring(diff)
				)
			end
			task.wait(1)
		end
	end)
end

-- ============================================================================
-- INIT
-- ============================================================================
CharacterController.Init()
buildUI()
setupInput()
setupRoundInfoUpdater()

print("[ZombieAttackProjectUI] Loaded. Toggle key:", Config.ToggleKey.Name)
print("[ZombieAttackProjectUI] RoundInfo folder:", GameStructure.RoundInfoFolder and "OK" or "MISSING")
print("[ZombieAttackProjectUI] Actions folder:", GameStructure.ActionsFolder and "OK" or "MISSING")
print("[ZombieAttackProjectUI] PlayerSpawn/SafeZone:", FindSafeZone() and "FOUND" or "NOT FOUND")
