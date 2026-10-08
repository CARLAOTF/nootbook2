local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local LOG_PREFIX = "[Auto Laptop]"
print(LOG_PREFIX, "Carregandoâ€¦")

assert(
	type(getgenv) == "function"
		and type(getconnections) == "function"
		and type(firesignal) == "function"
		and type(debug.getupvalues) == "function",
	"Executor sem suporte a introspecÃ§Ã£o de conexÃµes"
)

local playerGui = Players.LocalPlayer:WaitForChild("PlayerGui", 10)
assert(playerGui, "PlayerGui indisponÃ­vel")

local featuresFolder = ReplicatedStorage:WaitForChild("Features", 10)
local feature = featuresFolder and featuresFolder:FindFirstChild("CoinClicker")
assert(feature, "Coin Clicker nÃ£o encontrado")

local vide = require(ReplicatedStorage.Packages.vide)
local Catalog = require(feature.Catalog)
local Upgrades = require(feature.Upgrades)
local Fortunes = require(feature.Fortunes)
local Rules = require(feature.Rules)
local Achievements = require(feature.Achievements)
local Engine = require(feature.Engine)
local SettingsFactory = require(feature.Settings)
local ServerBackend = require(feature.Drivers.ServerBackend)
local remotes = require(feature.Remotes).get()

-- Constants

local SESSION_KEY = "AutoLaptop"
local GUI_NAME = "AutoLaptop"
local GAME_GUI_NAME = "CoinClickerGui"

local MAX_FAILURES = 3
local MAX_LOGS = 24
local VISIBLE_LOGS = 9
local CHART_SAMPLES = 24
local SAMPLE_WINDOW = 8
local PENDING_TIMEOUT = 8
local COLLECT_TICK = 0.05
local REACTION_DELAY = NumberRange.new(0.45, 1.3)
local CLICK_GAP = NumberRange.new(0.22, 0.6)
local HESITATION = NumberRange.new(0.4, 1.2)
local HESITATION_CHANCE = 0.12
local WRINKLER_JITTER = 0.15
local WRINKLER_WINDOW = 15
local PLAN_INTERVAL = 0.45
local SAVE_ADVANTAGE = 1.08
local UPGRADE_SLOTS = 12
local MAX_SPAM_PER_FRAME = 250
local PANEL_TRANSPARENCY = 0.15
local SCREEN_COVERAGE = 0.9
local COMPACT_SCALE_NAME = "AutoLaptopScale"
local DRAG_THRESHOLD = 6

local WINDOW_SIZE = Vector2.new(780, 540)
local SIDEBAR_WIDTH = 176

local STRATEGIES = { "Lucro", "Mais barato", "Desbloquear" }
local STRATEGY_HINTS = {
	Lucro = "Compara o ganho passivo e o ganho dos cliques.",
	["Mais barato"] = "Prioriza a opÃ§Ã£o com menor preÃ§o.",
	Desbloquear = "Prioriza tipos ainda nÃ£o comprados; depois, retorno.",
}

local AUTO_SPELL_ORDER = { "handOfFate", "sleightOfHand", "packedHouse", "conjure" }
local SPELL_NAMES = {
	auto = "Combo inteligente",
	conjure = "Conjurar moedas",
	stretchTime = "Estender buffs",
	resurrect = "Invocar wrinkler",
	handOfFate = "Moeda dourada",
	sleightOfHand = "Clique x25",
	packedHouse = "ProduÃ§Ã£o x5",
	edifice = "Criar edifÃ­cio",
}

local GENERATOR_NAMES = {
	Finger = "Dedo",
	StickyNote = "Nota pegajosa",
	Wallet = "Carteira",
	Coffee = "CafÃ© gelado",
	PlantPot = "Vaso de plantas",
	SignBoard = "Placa",
	Pizza = "Festa da pizza",
	Boombox = "Boombox",
	Dumbbell = "GinÃ¡sio",
	Vacuum = "Aspirador",
	Megaphone = "Microfone",
	CrystalBall = "Bola de cristal",
	Compass = "BÃºssola",
	Camera = "Cabine de fotos",
}

local REDUCED_EFFECTS = { particles = 0, backgroundCoins = 0, floatingNumbers = false, clickSound = false }

local AUTOMATION_KEYS = {
	"UISpam", "Remote12", "AutoExtraUI", "AutoBuy", "AutoUpgrade", "AutoGolden",
	"AutoLump", "AutoMarket", "AutoWrinkler", "AutoReward", "AutoSpell",
}

local NUMBER_SUFFIXES = { "", "K", "M", "B", "T", "Qa", "Qi", "Sx" }

-- Janitor

local Janitor = {}
Janitor.__index = Janitor

function Janitor.new()
	return setmetatable({ tasks = {} }, Janitor)
end

function Janitor:add(task)
	table.insert(self.tasks, task)
	return task
end

function Janitor:connect(signal, callback)
	return self:add(signal:Connect(callback))
end

function Janitor:clean()
	local tasks = self.tasks
	self.tasks = {}
	for index = #tasks, 1, -1 do
		local task = tasks[index]
		local kind = typeof(task)
		if kind == "RBXScriptConnection" then
			task:Disconnect()
		elseif kind == "Instance" then
			pcall(task.Destroy, task)
		elseif kind == "function" then
			pcall(task)
		elseif kind == "table" then
			pcall(task.clean, task)
		end
	end
end

-- Session

local env = getgenv()
local previousSession = env[SESSION_KEY]
if type(previousSession) == "table" and type(previousSession.Stop) == "function" then
	pcall(previousSession.Stop)
end

local staleGui = playerGui:FindFirstChild(GUI_NAME)
if staleGui then
	staleGui:Destroy()
end

local janitor = Janitor.new()
local running = true
local session = {}

function session.Stop()
	if not running then
		return
	end
	running = false
	janitor:clean()
	if env[SESSION_KEY] == session then
		env[SESSION_KEY] = nil
	end
	print(LOG_PREFIX, "Descarregado: interface, conexÃµes e backend removidos.")
end

env[SESSION_KEY] = session

-- State

local Config = {
	UISpam = true,
	SpamPerFrame = 50,
	Remote12 = true,
	StaminaGuard = true,
	MinStamina = 15,
	ResumeStamina = 60,
	Performance = true,
	CompactView = false,
	CompactScale = 0.75,
	AutoBuy = true,
	AutoUpgrade = true,
	SaveMode = true,
	MaxSaveSeconds = 900,
	Strategy = "Lucro",
	AutoExtraUI = true,
	AutoGolden = true,
	AutoLump = true,
	AutoMarket = true,
	AutoReward = true,
	AutoWrinkler = true,
	WrinklerInterval = 30,
	AutoSpell = true,
	Spell = "auto",
	Paused = true,
	Verbose = false,
}

local counters = {
	uiSignals = 0,
	remoteClicks = 0,
	extraClicks = 0,
	buildings = 0,
	upgrades = 0,
	rares = 0,
	goldens = 0,
	harvests = 0,
	wrinklers = 0,
	spells = 0,
}

local server = {
	rate = 0,
	income = 0,
	coins = nil,
	syncedAt = nil,
	syncCount = 0,
	confirmed = 0,
	rejected = 0,
	chartedAt = 0,
	samples = {},
	history = {},
}

local pending = {}
local logs = {}
local failures = {}
local optionListeners = {}
local startedAt = os.clock()

-- Utilities

local function formatNumber(value)
	value = tonumber(value) or 0
	if value ~= value then
		return "0"
	end
	if math.abs(value) == math.huge then
		return if value > 0 then "âˆž" else "-âˆž"
	end
	local magnitude, tier = math.abs(value), 1
	while magnitude >= 1000 and tier < #NUMBER_SUFFIXES do
		magnitude /= 1000
		tier += 1
	end
	if magnitude >= 1000 then
		return string.format("%.2e", value)
	end
	local signed = if value < 0 then -magnitude else magnitude
	if tier == 1 then
		return (string.format("%.1f", signed):gsub("%.0$", ""))
	end
	return string.format("%.2f%s", signed, NUMBER_SUFFIXES[tier])
end

local function formatDuration(seconds)
	return string.format("%d min %d s", math.floor(seconds / 60), math.floor(seconds % 60))
end

local function nextInList(list, current)
	local index = table.find(list, current) or 0
	return list[index % #list + 1]
end

local function generatorName(generator)
	return GENERATOR_NAMES[generator.id] or generator.name
end

local function log(message, isError)
	local elapsed = os.clock() - startedAt
	table.insert(logs, 1, string.format("%02d:%02d  %s", math.floor(elapsed / 60), math.floor(elapsed % 60), message))
	if #logs > MAX_LOGS then
		table.remove(logs)
	end
	if isError then
		warn(LOG_PREFIX, message)
	elseif Config.Verbose then
		print(LOG_PREFIX, message)
	end
end

local function attempt(name, callback, ...)
	local ok, err = pcall(callback, ...)
	if ok then
		failures[name] = 0
		return true
	end
	failures[name] = (failures[name] or 0) + 1
	if failures[name] == 1 then
		log(name .. ": " .. tostring(err):sub(1, 170), true)
	end
	return false
end

local function setOption(key, value)
	Config[key] = value
	for _, listener in optionListeners[key] or {} do
		listener(value)
	end
end

local function onOption(key, listener)
	optionListeners[key] = optionListeners[key] or {}
	table.insert(optionListeners[key], listener)
	listener(Config[key])
end

local function togglePause()
	setOption("Paused", not Config.Paused)
	log(if Config.Paused then "AutomaÃ§Ãµes pausadas" else "AutomaÃ§Ãµes retomadas")
end

-- Backend

local backendJanitor = janitor:add(Janitor.new())
local driver, gameSettings, defaultEffects, viewState, appliedPerformance

local function ready()
	return driver ~= nil and driver.ready()
end

local function findViewState(backend)
	for _, upvalue in debug.getupvalues(backend.buy) do
		if type(upvalue) == "table" and type(upvalue.hydrate) == "function" and type(upvalue.buy) == "function" then
			for _, candidate in debug.getupvalues(upvalue.buy) do
				if type(candidate) == "function" and debug.info(candidate, "n") == "view" then
					return candidate
				end
			end
		end
	end
end

local function applyEffects(reduced)
	local snapshot = gameSettings.snapshot()
	for key, value in REDUCED_EFFECTS do
		snapshot[key] = if reduced then value else defaultEffects[key]
	end
	gameSettings.apply(snapshot)
end

local function restoreEffects()
	if appliedPerformance == nil then
		return
	end
	applyEffects(false)
	driver.flushSettings()
end

local function syncPerformance()
	local reduced = Config.Performance and not Config.Paused
	if (appliedPerformance == true) == reduced or not ready() then
		return
	end
	applyEffects(reduced)
	appliedPerformance = reduced
	log(if reduced then "Desempenho: partÃ­culas, nÃºmeros e som desligados" else "Efeitos do jogo restaurados")
end

local function startBackend()
	backendJanitor:clean()
	local settings = SettingsFactory.new()
	local destroy, backend = vide.root(function()
		return ServerBackend({
			settings = settings,
			coinsPerClick = vide.source(1),
			timeScale = vide.source(1),
			particlesPerClick = settings.particles,
			goldenInterval = vide.source(20),
			wrinklerInterval = vide.source(25),
			achievementToasts = settings.achievementToasts,
			floatingNumbers = settings.floatingNumbers,
			clickSound = settings.clickSound,
			backgroundCoins = settings.backgroundCoins,
		})
	end)
	assert(type(destroy) == "function" and type(backend) == "table", "Falha ao iniciar o backend do jogo")

	driver, gameSettings, viewState = backend, settings, findViewState(backend)
	defaultEffects, appliedPerformance = settings.snapshot(), nil
	backendJanitor:add(destroy)
	backendJanitor:add(restoreEffects)
end

startBackend()

-- Purchases

local indexById = {}
for index, generator in Catalog.all do
	indexById[generator.id] = index
end

local planCache, plannedAt = {}, 0

local function invalidatePlan()
	plannedAt = 0
end

local function trackPending(kind, id, owned)
	table.insert(pending, { kind = kind, id = id, owned = owned, at = os.clock(), sync = server.syncCount })
end

local function marketStock()
	local stock = {}
	for _, entry in driver.marketStock() do
		stock[entry.id] = entry.left
	end
	return stock
end

local function canBuyUpgrade(upgrade)
	return not driver.upgradePurchased(upgrade.id) and driver.upgradeUnlocked(upgrade) and driver.coins() >= upgrade.price
end

local function availableUpgrades()
	local available = {}
	for _, upgrade in Upgrades.list do
		if not driver.upgradePurchased(upgrade.id) and driver.upgradeUnlocked(upgrade) then
			table.insert(available, upgrade)
		end
	end
	table.sort(available, function(a, b)
		return a.price < b.price
	end)
	return available
end

local function buyGenerator(id, rare)
	local index = indexById[id]
	if not ready() or not index or driver.coins() < driver.priceAt(index) then
		return false
	end

	local target = driver.ownedAt(index) + 1
	local bought
	if rare then
		bought = driver.marketBuy(id) and true or false
	else
		local amount, selling = driver.buyAmount(), driver.sellMode()
		driver.buyAmount(1)
		driver.sellMode(false)
		local ok, result = pcall(driver.buy, id)
		driver.buyAmount(amount)
		driver.sellMode(selling)
		if not ok then
			error(result, 0)
		end
		bought = result == true
	end

	if bought then
		trackPending("building", id, target)
		if rare then
			counters.rares += 1
		else
			counters.buildings += 1
		end
		invalidatePlan()
	end
	return bought
end

local function buyUpgrade(upgrade)
	if not (ready() and upgrade and canBuyUpgrade(upgrade) and driver.buyUpgrade(upgrade.id)) then
		return false
	end
	trackPending("upgrade", upgrade.id)
	counters.upgrades += 1
	invalidatePlan()
	log("Melhoria enviada: " .. upgrade.name)
	return true
end

local function comparePlans(a, b)
	if Config.Strategy == "Mais barato" and a.cost ~= b.cost then
		return a.cost < b.cost
	end
	if Config.Strategy == "Desbloquear" and a.unowned ~= b.unowned then
		return a.unowned
	end
	if a.roi ~= b.roi then
		return a.roi < b.roi
	end
	return a.cost < b.cost
end

local function planPurchases(force)
	if not ready() or not viewState then
		return {}
	end
	local now = os.clock()
	if not force and now - plannedAt < PLAN_INTERVAL then
		return planCache
	end
	plannedAt = now

	local state = viewState()
	local clickRate = if Config.UISpam or Config.Remote12 then math.max(server.rate, Rules.steadyClicksPerSecond) else 0
	local _, clickMultiplier = Engine.multipliers(state)
	local summary = Engine.summary(state)
	local baseClick = math.max(
		0,
		(driver.clickValue() / math.max(clickMultiplier, 1e-6) - Engine.baseCps(state) * summary.clickCpsShare)
			/ 2 ^ summary.clickTiers
	)

	local function income(candidate)
		return Engine.coinsPerSecond(candidate) + Engine.clickValue(candidate, baseClick) * clickRate
	end

	local baseIncome = income(state)
	local plans = {}

	local function evaluate(kind, id, name, cost, unowned, mutate)
		local candidate = table.clone(state)
		candidate.owned = table.clone(state.owned)
		candidate.purchased = table.clone(state.purchased)
		mutate(candidate)
		local gain = income(candidate) - baseIncome
		table.insert(plans, {
			kind = kind,
			id = id,
			name = name,
			cost = cost,
			gain = gain,
			roi = if gain > 1e-6 then cost / gain else math.huge,
			unowned = unowned,
		})
	end

	local stock = if Config.AutoMarket then marketStock() else {}
	for index, generator in Catalog.all do
		local enabled = if generator.rare then (stock[generator.id] or 0) > 0 else Config.AutoBuy
		if enabled then
			evaluate(
				if generator.rare then "rare" else "building",
				generator.id,
				generatorName(generator),
				driver.priceAt(index),
				driver.ownedAt(index) == 0,
				function(candidate)
					candidate.owned[generator.id] = (candidate.owned[generator.id] or 0) + 1
				end
			)
		end
	end

	if Config.AutoUpgrade then
		for _, upgrade in availableUpgrades() do
			evaluate("upgrade", upgrade.id, upgrade.name, upgrade.price, false, function(candidate)
				candidate.purchased[upgrade.id] = true
			end)
		end
	end

	table.sort(plans, comparePlans)
	planCache = plans
	return plans
end

local function decidePurchase(force)
	local plans = planPurchases(force)
	local target = plans[1]
	if not target then
		return { action = "idle" }
	end

	local funds = driver.coins()
	if target.cost <= funds then
		return { action = "buy", buy = target, target = target }
	end

	local affordable
	for _, plan in plans do
		if plan.cost <= funds then
			affordable = plan
			break
		end
	end

	local liveIncome = math.max(0.001, server.income, driver.coinsPerSecond())
	local wait = (target.cost - funds) / liveIncome
	local decision = {
		action = "save",
		target = target,
		missing = target.cost - funds,
		wait = wait,
		alternative = affordable,
		advantage = math.huge,
	}
	if not affordable then
		return decision
	end

	local waitAfter = (target.cost - (funds - affordable.cost)) / (liveIncome + math.max(0, affordable.gain))
	decision.advantage = affordable.roi / math.max(1e-6, target.roi)

	local shouldSave = Config.SaveMode
		and Config.Strategy == "Lucro"
		and wait <= Config.MaxSaveSeconds
		and decision.advantage >= SAVE_ADVANTAGE
		and waitAfter > wait + 0.05
		and affordable.roi > wait * 0.9

	if not shouldSave then
		decision.action = "buy"
		decision.buy = affordable
	end
	return decision
end

local function describeDecision(decision)
	if decision.action == "save" then
		local detail = string.format(
			"Faltam %s â€¢ ETA %ds â€¢ +%s/s",
			formatNumber(decision.missing),
			math.ceil(decision.wait),
			formatNumber(decision.target.gain)
		)
		if decision.alternative then
			detail ..= string.format(" â€¢ %s seria %.2fx pior", decision.alternative.name, decision.advantage)
		end
		return "Guardando para " .. decision.target.name, detail
	end
	if decision.action == "buy" then
		local plan = decision.buy
		return "PrÃ³xima compra: " .. plan.name,
			string.format("Custo %s â€¢ +%s/s â€¢ retorno %ss", formatNumber(plan.cost), formatNumber(plan.gain), formatNumber(plan.roi))
	end
	return "Aguardando oportunidades", "Ative geradores, melhorias ou mercado nas abas Farm e Eventos."
end

local savingTargetKey

local function runPurchases()
	local decision = decidePurchase(true)
	local plan = decision.buy
	if not plan then
		local key = decision.target and decision.target.kind .. decision.target.id
		if decision.action == "save" and key ~= savingTargetKey then
			savingTargetKey = key
			local title, detail = describeDecision(decision)
			log(title .. " â€¢ " .. detail)
		end
		return
	end

	savingTargetKey = nil
	if plan.kind == "upgrade" then
		buyUpgrade(Upgrades.byId[plan.id])
	elseif buyGenerator(plan.id, plan.kind == "rare") then
		log(string.format("Compra inteligente: %s â€¢ +%s/s â€¢ retorno %ss", plan.name, formatNumber(plan.gain), formatNumber(plan.roi)))
	end
end

-- Actions

local function canCast(spell)
	return driver.spellReady(spell) and driver.mana() >= driver.spellCost(spell)
end

local function pickSpell()
	if Config.Spell ~= "auto" then
		return Fortunes.byId[Config.Spell]
	end
	for _, id in AUTO_SPELL_ORDER do
		local spell = Fortunes.byId[id]
		if spell and canCast(spell) then
			return spell
		end
	end
end

local function castSpell(spell)
	if not (ready() and spell and canCast(spell) and driver.castSpell(spell.id)) then
		return false
	end
	counters.spells += 1
	log("FeitiÃ§o enviado: " .. spell.name)
	return true
end

local function harvestLump()
	if not (ready() and driver.lumpStage() == "ripe" and driver.harvestLump()) then
		return false
	end
	counters.harvests += 1
	log("Colheita enviada")
	return true
end

local function claimReward()
	if not (ready() and not driver.rewardOwned() and driver.rewardReady() and driver.claimReward()) then
		return false
	end
	log("Recompensa resgatada")
	return true
end

-- Stamina

local staminaResting = false

local function hasStamina()
	if not Config.StaminaGuard then
		staminaResting = false
		return true
	end
	local stamina = math.floor(driver.stamina() * 100)
	local resumeAt = math.max(Config.ResumeStamina, Config.MinStamina)
	if staminaResting and stamina >= resumeAt then
		staminaResting = false
		log(string.format("Stamina em %d%%; cliques retomados", stamina))
	elseif not staminaResting and stamina <= Config.MinStamina then
		staminaResting = true
		log(string.format("Stamina em %d%%; cliques pausados atÃ© %d%%", stamina, resumeAt))
	end
	return not staminaResting
end

local function fireRemoteClick()
	if not hasStamina() then
		return
	end
	remotes.Clicks:FireServer(1)
	counters.remoteClicks += 1
end

-- Game interface

local gameGuiJanitor = janitor:add(Janitor.new())
local gameGui = {
	root = nil,
	bigCoin = nil,
	bigClick = nil,
	searchedAt = 0,
	extras = {},
}

local laptopOverrides = {}
local laptopInstances = {}

local function override(instance, property, value)
	local saved = laptopOverrides[instance]
	if not saved then
		saved = {}
		laptopOverrides[instance] = saved
	end
	if saved[property] == nil then
		saved[property] = instance[property]
	end
	instance[property] = value
end

local function originalValue(instance, property)
	local saved = laptopOverrides[instance]
	local value = saved and saved[property]
	if value == nil then
		return instance[property]
	end
	return value
end

local function restoreLaptop()
	for instance, saved in laptopOverrides do
		for property, value in saved do
			pcall(function()
				instance[property] = value
			end)
		end
	end
	for _, instance in laptopInstances do
		pcall(instance.Destroy, instance)
	end
	table.clear(laptopOverrides)
	table.clear(laptopInstances)
end

janitor:add(restoreLaptop)

local function trackGameButton(item)
	if not item:IsA("GuiButton") then
		return
	end
	if item.Name == "BigCoin" then
		gameGui.bigCoin, gameGui.bigClick, gameGui.searchedAt = item, nil, 0
	elseif item.Name == "GoldenCoin" or item.Name == "Wrinkler" then
		gameGui.extras[item] = true
	end
end

local function untrackGameButton(item)
	gameGui.extras[item] = nil
	if item == gameGui.bigCoin then
		gameGui.bigCoin, gameGui.bigClick = nil, nil
	end
end

local function bindGameGui(root)
	gameGuiJanitor:clean()
	restoreLaptop()
	table.clear(gameGui.extras)
	gameGui.root, gameGui.bigCoin, gameGui.bigClick = root, nil, nil
	if not root then
		return
	end
	for _, item in root:GetDescendants() do
		trackGameButton(item)
	end
	gameGuiJanitor:connect(root.DescendantAdded, trackGameButton)
	gameGuiJanitor:connect(root.DescendantRemoving, untrackGameButton)
end

local function clearFill(item)
	override(item, "BackgroundTransparency", 1)
	override(item, "Active", false)
	if item:IsA("ImageLabel") then
		override(item, "ImageTransparency", 1)
	end
end

local function coversScreen(item, viewport)
	return item:IsA("GuiObject")
		and item.AbsoluteSize.X >= viewport.X * SCREEN_COVERAGE
		and item.AbsoluteSize.Y >= viewport.Y * SCREEN_COVERAGE
end

local function pinToEdge(column, edgeX)
	local anchor = originalValue(column, "AnchorPoint")
	local position = originalValue(column, "Position")
	local size = column.Size
	local shiftX = edgeX - anchor.X
	override(column, "AnchorPoint", Vector2.new(edgeX, 0))
	override(column, "Position", UDim2.new(
		position.X.Scale + size.X.Scale * shiftX,
		position.X.Offset + size.X.Offset * shiftX,
		position.Y.Scale - size.Y.Scale * anchor.Y,
		position.Y.Offset - size.Y.Offset * anchor.Y
	))
end

local function scaleColumn(column)
	local scale = column:FindFirstChildOfClass("UIScale")
	if not scale then
		scale = Instance.new("UIScale")
		scale.Name = COMPACT_SCALE_NAME
		scale.Parent = column
		table.insert(laptopInstances, scale)
	end
	local base = if scale.Name == COMPACT_SCALE_NAME then 1 else originalValue(scale, "Scale")
	override(scale, "Scale", base * Config.CompactScale)
end

local missingLaptopLogged = false

local function compactLaptop()
	local root, coin = gameGui.root, gameGui.bigCoin
	local main = root and root:FindFirstChild("Main")
	local body = main and main:FindFirstChild("Body")
	local camera = workspace.CurrentCamera
	if not (body and coin and camera) then
		if root and not missingLaptopLogged then
			missingLaptopLogged = true
			log("Compact view: Main/Body ou BigCoin nÃ£o encontrados na CoinClickerGui", true)
		end
		return
	end
	missingLaptopLogged = false

	for _, layer in root:GetChildren() do
		if layer ~= main and coversScreen(layer, camera.ViewportSize) then
			clearFill(layer)
		end
	end
	local panelColor = main.BackgroundColor3
	clearFill(main)
	clearFill(body)

	for _, column in body:GetChildren() do
		local isStore, isCoin = column.Name == "RightColumn", coin:IsDescendantOf(column)
		if (isStore or isCoin) and column:IsA("GuiObject") then
			override(column, "BackgroundColor3", panelColor)
			override(column, "BackgroundTransparency", PANEL_TRANSPARENCY)
			pinToEdge(column, if isStore then 1 else 0)
			scaleColumn(column)
		elseif column:IsA("GuiObject") then
			clearFill(column)
			for _, child in column:GetChildren() do
				if child:IsA("GuiObject") then
					override(child, "Visible", false)
				end
			end
		end
	end
end

onOption("CompactView", function(enabled)
	if enabled then
		attempt("Laptop compacto", compactLaptop)
	else
		restoreLaptop()
	end
end)

local function findBigCoinClick(coin)
	for _, connection in getconnections(coin.Activated) do
		local handler = connection.Function
		if type(handler) == "function" then
			for _, upvalue in debug.getupvalues(handler) do
				if
					type(upvalue) == "function"
					and debug.info(upvalue, "n") == "click"
					and debug.info(upvalue, "s"):find("ServerBackend")
				then
					return upvalue
				end
			end
		end
	end
end

local function restoreBigCoin(coin)
	coin.Visible = true
	coin.BackgroundTransparency = 0
	local coinScale = coin:FindFirstChildOfClass("UIScale")
	if coinScale then
		coinScale.Scale = 1
	end
	for _, child in coin:GetChildren() do
		if child:IsA("ImageLabel") then
			child.Visible = true
			child.ImageTransparency = 0
		end
	end
end

local function spamBigCoin(now)
	local coin = gameGui.bigCoin
	if not coin or not hasStamina() then
		return
	end
	if not gameGui.bigClick and now - gameGui.searchedAt >= 1 then
		gameGui.searchedAt = now
		gameGui.bigClick = findBigCoinClick(coin)
	end
	local click = gameGui.bigClick
	if not click then
		return
	end
	restoreBigCoin(coin)
	local amount = math.clamp(math.floor(Config.SpamPerFrame), 1, MAX_SPAM_PER_FRAME)
	for _ = 1, amount do
		click()
	end
	counters.uiSignals += amount
end

-- Collector

local random = Random.new()
local collector = {
	generation = 0,
	nextClickAt = 0,
	wrinklersUntil = 0,
	wrinklersDueAt = os.clock() + Config.WrinklerInterval,
	targets = { extra = {}, golden = {}, wrinkler = {} },
}

local function humanDelay(range)
	return range.Min + (range.Max - range.Min) * (random:NextNumber() + random:NextNumber()) / 2
end

local function nextClickGap()
	local gap = humanDelay(CLICK_GAP)
	if random:NextNumber() < HESITATION_CHANCE then
		gap += humanDelay(HESITATION)
	end
	return gap
end

local function markTarget(kind, key, now)
	local pool = collector.targets[kind]
	local target = pool[key]
	if not target then
		target = { readyAt = now + humanDelay(REACTION_DELAY) }
		pool[key] = target
	end
	target.generation = collector.generation
end

local function scanTargets(now, automatic)
	collector.generation += 1
	if automatic and Config.AutoExtraUI then
		for extra in gameGui.extras do
			if extra.Visible and extra.Parent then
				markTarget("extra", extra, now)
			end
		end
	end
	if automatic and Config.AutoGolden then
		for _, golden in driver.goldens() do
			markTarget("golden", golden.id, now)
		end
	end
	if now < collector.wrinklersUntil then
		local wrinklers = driver.wrinklers()
		if #wrinklers == 0 then
			collector.wrinklersUntil = 0
		end
		for _, wrinkler in wrinklers do
			markTarget("wrinkler", wrinkler.id, now)
		end
	end
end

local function pickTarget(now)
	local bestKind, bestKey, bestAt
	for kind, pool in collector.targets do
		for key, target in pool do
			if target.generation ~= collector.generation then
				pool[key] = nil
			elseif target.readyAt <= now and (not bestAt or target.readyAt < bestAt) then
				bestKind, bestKey, bestAt = kind, key, target.readyAt
			end
		end
	end
	return bestKind, bestKey
end

local collectActions = {
	extra = function(extra)
		if extra.Parent and extra.Visible then
			firesignal(extra.Activated)
			counters.extraClicks += 1
		end
	end,
	golden = function(id)
		if driver.clickGolden(id) then
			counters.goldens += 1
			log("Moeda dourada coletada")
		end
	end,
	wrinkler = function(id)
		if driver.popWrinkler(id) then
			counters.wrinklers += 1
			log("Wrinkler coletado")
		end
	end,
}

local function requestWrinklers(now)
	collector.wrinklersUntil = now + WRINKLER_WINDOW
end

local function runCollector(now)
	local automatic = not Config.Paused
	if automatic and Config.AutoWrinkler and now >= collector.wrinklersDueAt then
		collector.wrinklersDueAt = now + Config.WrinklerInterval * random:NextNumber(1 - WRINKLER_JITTER, 1 + WRINKLER_JITTER)
		requestWrinklers(now)
	end
	scanTargets(now, automatic)
	if now < collector.nextClickAt then
		return
	end
	local kind, key = pickTarget(now)
	if not kind then
		return
	end
	collector.targets[kind][key] = nil
	collector.nextClickAt = now + nextClickGap()
	collectActions[kind](key)
end

local function openGamePanel(panelName)
	local buttons = gameGui.root and gameGui.root:FindFirstChild("Buttons", true)
	if buttons then
		for _, gameButton in buttons:GetChildren() do
			if gameButton:IsA("TextButton") and gameButton.Text == panelName then
				firesignal(gameButton.Activated)
				return true
			end
		end
	end
	log("Painel do jogo indisponÃ­vel: " .. panelName)
	return false
end

-- Server sync

local function recordSample(snapshot, now)
	local samples = server.samples
	table.insert(samples, { at = now, clicks = snapshot.TotalClicks or 0, earned = snapshot.TotalEarned or 0 })
	while #samples > 2 and now - samples[2].at > SAMPLE_WINDOW do
		table.remove(samples, 1)
	end

	local first, latest = samples[1], samples[#samples]
	local span = now - first.at
	if span >= 1 then
		server.rate = math.max(0, (latest.clicks - first.clicks) / span)
		server.income = math.max(0, (latest.earned - first.earned) / span)
	end

	if now - server.chartedAt >= 1 then
		server.chartedAt = now
		table.insert(server.history, server.income)
		if #server.history > CHART_SAMPLES then
			table.remove(server.history, 1)
		end
	end
end

local function isConfirmed(entry, snapshot)
	if entry.kind == "upgrade" then
		return (snapshot.Purchased or {})[entry.id] == true
	end
	return ((snapshot.Owned or {})[entry.id] or 0) >= entry.owned
end

local function onSync(snapshot)
	if type(snapshot) ~= "table" then
		return
	end
	local now = os.clock()
	recordSample(snapshot, now)
	server.coins, server.syncedAt = snapshot.Coins, now
	server.syncCount += 1

	for index = #pending, 1, -1 do
		local entry = pending[index]
		if isConfirmed(entry, snapshot) then
			server.confirmed += 1
			table.remove(pending, index)
		elseif now - entry.at > PENDING_TIMEOUT and server.syncCount > entry.sync + 1 then
			server.rejected += 1
			table.remove(pending, index)
		end
	end
	invalidatePlan()
end

local function reconnect()
	startBackend()
	server.rate, server.income, server.coins, server.syncedAt = 0, 0, nil, nil
	table.clear(server.samples)
	table.clear(pending)
	invalidatePlan()
	log("Backend reiniciado; aguardando o servidor")
end

local function buildInterface()
	-- Theme

	local Theme = {
		Background = Color3.fromRGB(11, 13, 18),
		Sidebar = Color3.fromRGB(15, 18, 25),
		Surface = Color3.fromRGB(20, 24, 33),
		Field = Color3.fromRGB(29, 34, 46),
		Stroke = Color3.fromRGB(36, 42, 56),
		Muted = Color3.fromRGB(52, 59, 76),
		Text = Color3.fromRGB(236, 239, 245),
		Subtext = Color3.fromRGB(136, 146, 166),
		Accent = Color3.fromRGB(91, 104, 245),
		Success = Color3.fromRGB(52, 211, 153),
		Warning = Color3.fromRGB(251, 191, 36),
		Danger = Color3.fromRGB(239, 83, 104),
	}

	local TWEEN_INFO = TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
	local RIGHT_CENTER = Vector2.new(1, 0.5)
	local CONTROL_POSITION = UDim2.new(1, -16, 0.5, 0)
	local KNOB_OFF = UDim2.new(0, 3, 0.5, 0)
	local KNOB_ON = UDim2.new(1, -21, 0.5, 0)

	local LABEL_DEFAULTS = {
		BackgroundTransparency = 1,
		Font = Enum.Font.Gotham,
		TextSize = 13,
		TextColor3 = Theme.Text,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
	}

	local BUTTON_DEFAULTS = {
		BackgroundColor3 = Theme.Field,
		BorderSizePixel = 0,
		AutoButtonColor = true,
		Font = Enum.Font.GothamBold,
		TextSize = 12,
		TextColor3 = Theme.Text,
	}

	-- Components

	local function merge(base, overrides)
		local result = table.clone(base)
		for key, value in overrides do
			result[key] = value
		end
		return result
	end

	local function create(className, properties)
		local instance = Instance.new(className)
		for key, value in properties do
			if key ~= "Parent" then
				instance[key] = value
			end
		end
		instance.Parent = properties.Parent
		return instance
	end

	local function tween(instance, goal)
		TweenService:Create(instance, TWEEN_INFO, goal):Play()
	end

	local function round(parent, radius)
		create("UICorner", { Parent = parent, CornerRadius = UDim.new(0, radius) })
	end

	local function outline(parent)
		create("UIStroke", { Parent = parent, Color = Theme.Stroke, ApplyStrokeMode = Enum.ApplyStrokeMode.Border })
	end

	local function stack(parent, gap, properties)
		return create("UIListLayout", merge({
			Parent = parent,
			Padding = UDim.new(0, gap),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, properties or {}))
	end

	local function frame(properties)
		return create("Frame", merge({ BackgroundColor3 = Theme.Surface, BorderSizePixel = 0 }, properties))
	end

	local function label(properties)
		return create("TextLabel", merge(LABEL_DEFAULTS, properties))
	end

	local function button(properties, onClick)
		local instance = create("TextButton", merge(BUTTON_DEFAULTS, properties))
		round(instance, 8)
		janitor:connect(instance.Activated, function()
			attempt(instance.Name, onClick)
		end)
		return instance
	end

	local function paint(target, enabled)
		target.BackgroundColor3 = if enabled then Theme.Accent else Theme.Field
	end

	local function nextOrder(parent)
		return #parent:GetChildren()
	end

	local function section(page, title, description)
		local holder = frame({
			Parent = page,
			Size = UDim2.new(1, 0, 0, if description then 46 else 28),
			BackgroundTransparency = 1,
			LayoutOrder = nextOrder(page),
		})
		label({
			Parent = holder,
			Text = title,
			Position = UDim2.fromOffset(2, 4),
			Size = UDim2.new(1, -4, 0, 20),
			Font = Enum.Font.GothamBold,
			TextSize = 15,
		})
		if description then
			label({
				Parent = holder,
				Text = description,
				Position = UDim2.fromOffset(2, 25),
				Size = UDim2.new(1, -4, 0, 16),
				TextSize = 11,
				TextColor3 = Theme.Subtext,
			})
		end
	end

	local function card(page, height)
		local instance = frame({ Parent = page, Size = UDim2.new(1, 0, 0, height), LayoutOrder = nextOrder(page) })
		round(instance, 10)
		outline(instance)
		return instance
	end

	local function textCard(page, height, textSize)
		local holder = card(page, height)
		return label({
			Parent = holder,
			Position = UDim2.fromOffset(16, 12),
			Size = UDim2.new(1, -32, 1, -24),
			TextSize = textSize,
			TextWrapped = true,
			TextTruncate = Enum.TextTruncate.None,
			TextYAlignment = Enum.TextYAlignment.Top,
			LineHeight = 1.15,
		})
	end

	local function settingRow(page, title, description, controlWidth, icon)
		local row = card(page, 58)
		local textLeft = if icon then 62 else 16
		local textWidth = UDim2.new(1, -(textLeft + controlWidth + 28), 0, 18)
		if icon then
			create("ImageLabel", {
				Parent = row,
				Image = icon,
				BackgroundTransparency = 1,
				Position = UDim2.fromOffset(14, 10),
				Size = UDim2.fromOffset(38, 38),
			})
		end
		local heading = label({
			Parent = row,
			Text = title,
			Position = UDim2.fromOffset(textLeft, 10),
			Size = textWidth,
			Font = Enum.Font.GothamBold,
		})
		local detail = label({
			Parent = row,
			Text = description or "",
			Position = UDim2.fromOffset(textLeft, 30),
			Size = textWidth,
			TextSize = 11,
			TextColor3 = Theme.Subtext,
		})
		return row, detail, heading
	end

	local function toggleRow(page, key, title, description)
		local row = settingRow(page, title, description, 44)
		local track = create("TextButton", {
			Parent = row,
			Name = key,
			Text = "",
			AutoButtonColor = false,
			BorderSizePixel = 0,
			AnchorPoint = RIGHT_CENTER,
			Position = CONTROL_POSITION,
			Size = UDim2.fromOffset(44, 24),
			BackgroundColor3 = Theme.Muted,
		})
		round(track, 12)
		local knob = frame({
			Parent = track,
			AnchorPoint = Vector2.new(0, 0.5),
			Position = KNOB_OFF,
			Size = UDim2.fromOffset(18, 18),
			BackgroundColor3 = Theme.Text,
		})
		round(knob, 9)

		janitor:connect(track.Activated, function()
			setOption(key, not Config[key])
			log(title .. ": " .. (if Config[key] then "ligado" else "desligado"))
		end)
		onOption(key, function(enabled)
			tween(track, { BackgroundColor3 = if enabled then Theme.Accent else Theme.Muted })
			tween(knob, { Position = if enabled then KNOB_ON else KNOB_OFF })
		end)
	end

	local function numberRow(page, key, title, description, min, max)
		local row = settingRow(page, title, description, 96)
		local box = create("TextBox", {
			Parent = row,
			Name = key,
			AnchorPoint = RIGHT_CENTER,
			Position = CONTROL_POSITION,
			Size = UDim2.fromOffset(96, 30),
			BackgroundColor3 = Theme.Field,
			BorderSizePixel = 0,
			Text = tostring(Config[key]),
			TextColor3 = Theme.Text,
			Font = Enum.Font.GothamBold,
			TextSize = 13,
			ClearTextOnFocus = false,
		})
		round(box, 8)
		outline(box)

		janitor:connect(box.FocusLost, function()
			local value = tonumber((box.Text:gsub("%s", ""):gsub(",", ".")))
			if value and value == value and math.abs(value) ~= math.huge then
				setOption(key, math.clamp(value, min, max))
				log(title .. " = " .. Config[key])
			else
				log("NÃºmero invÃ¡lido: " .. title)
			end
			box.Text = tostring(Config[key])
		end)
	end

	local function actionRow(page, title, description, text, onClick, options)
		options = options or {}
		local row, detail, heading = settingRow(page, title, description, 104, options.icon)
		local action = button({
			Parent = row,
			Name = title,
			Text = text,
			AnchorPoint = RIGHT_CENTER,
			Position = CONTROL_POSITION,
			Size = UDim2.fromOffset(104, 30),
			BackgroundColor3 = options.color or Theme.Field,
		}, onClick)
		return detail, action, heading
	end

	-- Window

	local screen = janitor:add(create("ScreenGui", {
		Parent = playerGui,
		Name = GUI_NAME,
		ResetOnSpawn = false,
		IgnoreGuiInset = true,
		DisplayOrder = 1200,
		ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	}))

	local window = frame({
		Parent = screen,
		Name = "Window",
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.fromScale(0.5, 0.5),
		Size = UDim2.fromOffset(WINDOW_SIZE.X, WINDOW_SIZE.Y),
		BackgroundColor3 = Theme.Background,
		Active = true,
	})
	round(window, 14)
	outline(window)
	local windowScale = create("UIScale", { Parent = window })

	local reopenButton, reopenDragged

	local function setWindowVisible(visible)
		window.Visible = visible
		reopenButton.Visible = not visible
	end

	reopenButton = button({
		Parent = screen,
		Name = "Reopen",
		Text = "AL",
		Visible = false,
		Position = UDim2.fromOffset(20, 160),
		Size = UDim2.fromOffset(58, 38),
		BackgroundColor3 = Theme.Accent,
	}, function()
		if not reopenDragged() then
			setWindowVisible(true)
		end
	end)

	local sidebar = frame({
		Parent = window,
		Name = "Sidebar",
		Size = UDim2.new(0, SIDEBAR_WIDTH, 1, 0),
		BackgroundColor3 = Theme.Sidebar,
		Active = true,
	})
	round(sidebar, 14)
	frame({
		Parent = sidebar,
		Position = UDim2.new(1, -14, 0, 0),
		Size = UDim2.new(0, 14, 1, 0),
		BackgroundColor3 = Theme.Sidebar,
	})
	frame({
		Parent = sidebar,
		Position = UDim2.new(1, -1, 0, 0),
		Size = UDim2.new(0, 1, 1, 0),
		BackgroundColor3 = Theme.Stroke,
	})

	label({
		Parent = sidebar,
		Text = "Auto Laptop",
		Position = UDim2.fromOffset(20, 18),
		Size = UDim2.new(1, -40, 0, 20),
		Font = Enum.Font.GothamBold,
		TextSize = 16,
	})
	label({
		Parent = sidebar,
		Text = "Coin Clicker",
		Position = UDim2.fromOffset(20, 40),
		Size = UDim2.new(1, -40, 0, 14),
		TextSize = 11,
		TextColor3 = Theme.Subtext,
	})

	local tabList = frame({
		Parent = sidebar,
		Name = "Tabs",
		Position = UDim2.fromOffset(12, 76),
		Size = UDim2.new(1, -24, 1, -150),
		BackgroundTransparency = 1,
	})
	stack(tabList, 4)

	local connectionDot = frame({
		Parent = sidebar,
		Position = UDim2.new(0, 20, 1, -70),
		Size = UDim2.fromOffset(8, 8),
		BackgroundColor3 = Theme.Warning,
	})
	round(connectionDot, 4)
	local connectionLabel = label({
		Parent = sidebar,
		Position = UDim2.new(0, 36, 1, -76),
		Size = UDim2.new(1, -52, 0, 20),
		TextSize = 11,
		Font = Enum.Font.GothamBold,
	})
	label({
		Parent = sidebar,
		Text = "F6 pausar  â€¢  F7 compacto\nF8 ocultar painel",
		Position = UDim2.new(0, 20, 1, -48),
		Size = UDim2.new(1, -40, 0, 30),
		TextSize = 10,
		TextColor3 = Theme.Subtext,
		TextTruncate = Enum.TextTruncate.None,
		TextYAlignment = Enum.TextYAlignment.Top,
	})

	local main = frame({
		Parent = window,
		Name = "Main",
		Position = UDim2.fromOffset(SIDEBAR_WIDTH, 0),
		Size = UDim2.new(1, -SIDEBAR_WIDTH, 1, 0),
		BackgroundTransparency = 1,
	})

	local topBar = frame({
		Parent = main,
		Name = "TopBar",
		Size = UDim2.new(1, 0, 0, 60),
		BackgroundTransparency = 1,
		Active = true,
	})
	local pageTitle = label({
		Parent = topBar,
		Position = UDim2.fromOffset(20, 0),
		Size = UDim2.new(1, -300, 1, 0),
		Font = Enum.Font.GothamBold,
		TextSize = 18,
	})

	local controls = frame({
		Parent = topBar,
		AnchorPoint = RIGHT_CENTER,
		Position = UDim2.new(1, -16, 0.5, 0),
		Size = UDim2.fromOffset(280, 30),
		BackgroundTransparency = 1,
	})
	stack(controls, 8, {
		FillDirection = Enum.FillDirection.Horizontal,
		HorizontalAlignment = Enum.HorizontalAlignment.Right,
		VerticalAlignment = Enum.VerticalAlignment.Center,
	})

	local statusPill = frame({ Parent = controls, Size = UDim2.fromOffset(84, 26), LayoutOrder = 1 })
	round(statusPill, 13)
	local statusLabel = label({
		Parent = statusPill,
		Size = UDim2.fromScale(1, 1),
		Font = Enum.Font.GothamBold,
		TextSize = 10,
		TextXAlignment = Enum.TextXAlignment.Center,
	})

	local pauseButton = button({
		Parent = controls,
		Name = "Pause",
		Size = UDim2.fromOffset(88, 30),
		LayoutOrder = 2,
	}, togglePause)
	button({
		Parent = controls,
		Name = "Minimize",
		Text = "â€“",
		TextSize = 16,
		Size = UDim2.fromOffset(30, 30),
		LayoutOrder = 3,
	}, function()
		setWindowVisible(false)
	end)
	button({
		Parent = controls,
		Name = "Close",
		Text = "Ã—",
		TextSize = 18,
		Size = UDim2.fromOffset(30, 30),
		BackgroundColor3 = Theme.Danger,
		LayoutOrder = 4,
	}, session.Stop)

	onOption("Paused", function(paused)
		statusLabel.Text = if paused then "PAUSADO" else "ATIVO"
		statusLabel.TextColor3 = if paused then Theme.Warning else Theme.Success
		statusPill.BackgroundColor3 = if paused then Color3.fromRGB(58, 46, 18) else Color3.fromRGB(18, 52, 43)
		pauseButton.Text = if paused then "RETOMAR" else "PAUSAR"
		paint(pauseButton, paused)
	end)

	local metricRow = frame({
		Parent = main,
		Position = UDim2.fromOffset(20, 60),
		Size = UDim2.new(1, -40, 0, 64),
		BackgroundTransparency = 1,
	})
	stack(metricRow, 10, { FillDirection = Enum.FillDirection.Horizontal })

	local function metric(caption)
		local holder = frame({ Parent = metricRow, Size = UDim2.new(1 / 3, -20 / 3, 1, 0), LayoutOrder = nextOrder(metricRow) })
		round(holder, 10)
		outline(holder)
		label({
			Parent = holder,
			Text = caption,
			Position = UDim2.fromOffset(14, 10),
			Size = UDim2.new(1, -28, 0, 14),
			Font = Enum.Font.GothamBold,
			TextSize = 10,
			TextColor3 = Theme.Subtext,
		})
		return label({
			Parent = holder,
			Text = "â€”",
			Position = UDim2.fromOffset(14, 28),
			Size = UDim2.new(1, -28, 0, 24),
			Font = Enum.Font.GothamBold,
			TextSize = 20,
		})
	end

	local coinsMetric = metric("SALDO CONFIRMADO")
	local productionMetric = metric("PRODUÃ‡ÃƒO / S")
	local staminaMetric = metric("STAMINA")

	local pageHost = frame({
		Parent = main,
		Position = UDim2.fromOffset(20, 136),
		Size = UDim2.new(1, -28, 1, -150),
		BackgroundTransparency = 1,
	})

	-- Navigation

	local pages, tabs, pageUpdaters = {}, {}, {}
	local selectedPage

	local function selectPage(name)
		selectedPage = name
		pageTitle.Text = name
		for pageName, page in pages do
			page.Visible = pageName == name
		end
		for pageName, tab in tabs do
			local active = pageName == name
			tab.button.BackgroundTransparency = if active then 0 else 1
			tab.label.TextColor3 = if active then Theme.Text else Theme.Subtext
			tab.indicator.Visible = active
		end
		local update = pageUpdaters[name]
		if update then
			attempt("Interface", update)
		end
	end

	local function addPage(name)
		local page = create("ScrollingFrame", {
			Parent = pageHost,
			Name = name,
			Size = UDim2.fromScale(1, 1),
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			ScrollBarThickness = 3,
			ScrollBarImageColor3 = Theme.Muted,
			AutomaticCanvasSize = Enum.AutomaticSize.Y,
			CanvasSize = UDim2.new(),
			ScrollingDirection = Enum.ScrollingDirection.Y,
			Visible = false,
		})
		stack(page, 8)
		create("UIPadding", { Parent = page, PaddingRight = UDim.new(0, 12), PaddingBottom = UDim.new(0, 8) })
		pages[name] = page

		local tabButton = button({
			Parent = tabList,
			Name = "Tab" .. name,
			Text = "",
			AutoButtonColor = false,
			BackgroundColor3 = Theme.Surface,
			BackgroundTransparency = 1,
			Size = UDim2.new(1, 0, 0, 36),
			LayoutOrder = nextOrder(tabList),
		}, function()
			selectPage(name)
		end)
		local indicator = frame({
			Parent = tabButton,
			AnchorPoint = Vector2.new(0, 0.5),
			Position = UDim2.fromScale(0, 0.5),
			Size = UDim2.fromOffset(3, 16),
			BackgroundColor3 = Theme.Accent,
			Visible = false,
		})
		round(indicator, 2)
		local tabLabel = label({
			Parent = tabButton,
			Text = name,
			Position = UDim2.fromOffset(16, 0),
			Size = UDim2.new(1, -16, 1, 0),
			Font = Enum.Font.GothamBold,
			TextColor3 = Theme.Subtext,
		})
		tabs[name] = { button = tabButton, label = tabLabel, indicator = indicator }
		return page
	end

	-- Page: Painel

	do
		local page = addPage("Painel")

		section(page, "Servidor ao vivo", "MÃ©dia mÃ³vel de 8 segundos das atualizaÃ§Ãµes do servidor.")
		local live = card(page, 98)
		local function liveStat(position, caption)
			label({
				Parent = live,
				Text = caption,
				Position = position + UDim2.fromOffset(0, 34),
				Size = UDim2.new(0.5, -24, 0, 14),
				TextSize = 10,
				TextColor3 = Theme.Subtext,
			})
			return label({
				Parent = live,
				Position = position,
				Size = UDim2.new(0.5, -24, 0, 30),
				Font = Enum.Font.GothamBold,
				TextSize = 22,
			})
		end
		local rateLabel = liveStat(UDim2.fromOffset(16, 10), "CLIQUES / S")
		local incomeLabel = liveStat(UDim2.new(0.5, 8, 0, 10), "MOEDAS / S")
		rateLabel.TextColor3 = Theme.Success
		local syncLabel = label({
			Parent = live,
			Position = UDim2.fromOffset(16, 68),
			Size = UDim2.new(1, -32, 0, 18),
			TextSize = 11,
			TextColor3 = Theme.Subtext,
		})

		section(page, "Ritmo de ganho", "Uma barra por segundo de atualizaÃ§Ã£o real.")
		local chart = card(page, 96)
		local plot = frame({
			Parent = chart,
			Position = UDim2.fromOffset(12, 12),
			Size = UDim2.new(1, -24, 1, -24),
			BackgroundTransparency = 1,
		})
		local bars = {}
		for index = 1, CHART_SAMPLES do
			local bar = frame({
				Parent = plot,
				AnchorPoint = Vector2.new(0, 1),
				Position = UDim2.new((index - 1) / CHART_SAMPLES, 2, 1, 0),
				Size = UDim2.new(1 / CHART_SAMPLES, -4, 0, 2),
				BackgroundColor3 = Theme.Muted,
			})
			round(bar, 3)
			bars[index] = bar
		end

		section(page, "Motor de compras", "Compara ganho passivo e valor dos cliques por moeda gasta.")
		local plan = card(page, 70)
		local planTitle = label({
			Parent = plan,
			Position = UDim2.fromOffset(16, 12),
			Size = UDim2.new(1, -32, 0, 20),
			Font = Enum.Font.GothamBold,
			TextSize = 14,
		})
		local planDetail = label({
			Parent = plan,
			Position = UDim2.fromOffset(16, 36),
			Size = UDim2.new(1, -32, 0, 18),
			TextSize = 12,
			TextColor3 = Theme.Subtext,
		})

		pageUpdaters.Painel = function()
			rateLabel.Text = string.format("%.1f", server.rate)
			incomeLabel.Text = formatNumber(server.income)
			syncLabel.Text = string.format(
				"%d compras confirmadas â€¢ %d aguardando â€¢ %s",
				server.confirmed,
				#pending,
				if server.syncedAt then string.format("sync hÃ¡ %.1fs", os.clock() - server.syncedAt) else "aguardando servidor"
			)

			local peak = 1
			for _, sample in server.history do
				peak = math.max(peak, sample)
			end
			for index, bar in bars do
				local sample = server.history[index]
				bar.Size = UDim2.new(1 / CHART_SAMPLES, -4, if sample then math.max(0.04, sample / peak) else 0, 2)
				bar.BackgroundColor3 = if sample then Theme.Accent else Theme.Muted
			end

			planTitle.Text, planDetail.Text = describeDecision(decidePurchase())
		end
	end

	-- Page: Farm

	do
		local page = addPage("Farm")

		section(page, "Cliques", "Spam da moeda pela interface e remote em paralelo.")
		toggleRow(page, "UISpam", "Spam na moeda pela UI", "Chama o onClick do BigCoin sem acumular a animaÃ§Ã£o.")
		numberRow(page, "SpamPerFrame", "Cliques UI por frame", "Sinais enviados Ã  moeda por frame. PadrÃ£o: 50.", 1, MAX_SPAM_PER_FRAME)
		toggleRow(page, "Remote12", "Remote de 12/s", "Envia uma chamada unitÃ¡ria a cada 1/12 de segundo.")

		section(page, "Stamina", "Para os cliques na moeda quando a stamina acaba e espera recuperar.")
		toggleRow(page, "StaminaGuard", "Respeitar stamina", "Pausa spam e remote abaixo do limite mÃ­nimo.")
		numberRow(page, "MinStamina", "Pausar abaixo de (%)", "Cliques param quando a stamina chega neste valor.", 0, 90)
		numberRow(page, "ResumeStamina", "Retomar a partir de (%)", "Cliques voltam quando a stamina recupera atÃ© aqui.", 10, 100)

		section(page, "Investimento", "Planejador compara custo, ganho e tempo atÃ© a prÃ³xima compra.")
		toggleRow(page, "AutoBuy", "Comprar geradores", "Reinveste o saldo nas melhores oportunidades.")
		toggleRow(page, "AutoUpgrade", "Comprar melhorias", "Compara melhorias e geradores pelo ganho de renda.")
		toggleRow(page, "SaveMode", "Guardar para compra melhor", "Evita compras baratas que atrasariam opÃ§Ã£o superior.")
		numberRow(page, "MaxSaveSeconds", "Limite para guardar", "Espera mÃ¡xima, em segundos, por compra superior.", 30, 3600)

		local strategyDetail, strategyButton = actionRow(page, "EstratÃ©gia de compra", "", "", function()
			setOption("Strategy", nextInList(STRATEGIES, Config.Strategy))
			invalidatePlan()
			log("EstratÃ©gia: " .. Config.Strategy)
		end)
		onOption("Strategy", function(strategy)
			strategyButton.Text = strategy
			strategyDetail.Text = STRATEGY_HINTS[strategy]
		end)
	end

	-- Page: Loja

	do
		local page = addPage("Loja")

		section(page, "Geradores", "Compra 1 unidade preservando a seleÃ§Ã£o da loja do jogo.")
		local generatorRows = {}
		for index, generator in Catalog.generators do
			local detail, buyButton = actionRow(page, generatorName(generator), "", "Comprar 1", function()
				if buyGenerator(generator.id, false) then
					log("Comprou " .. generatorName(generator))
				else
					log("Saldo insuficiente ou gerador indisponÃ­vel")
				end
			end, { icon = generator.icon })
			table.insert(generatorRows, { index = index, detail = detail, button = buyButton })
		end

		section(page, "Melhorias disponÃ­veis", "As 12 melhorias liberadas mais baratas.")
		local upgradeRows = {}
		for slot = 1, UPGRADE_SLOTS do
			local entry = {}
			entry.detail, entry.button, entry.title = actionRow(page, "Melhoria " .. slot, "", "Comprar", function()
				if not buyUpgrade(entry.upgrade) then
					log("Melhoria indisponÃ­vel ou saldo insuficiente")
				end
			end)
			entry.row = entry.title.Parent
			upgradeRows[slot] = entry
		end
		local emptyLabel = textCard(page, 44, 12)
		emptyLabel.Text = "Nenhuma melhoria liberada no momento."
		emptyLabel.TextColor3 = Theme.Subtext

		pageUpdaters.Loja = function()
			if not ready() then
				return
			end
			local coins = driver.coins()
			for _, entry in generatorRows do
				local cost = driver.priceAt(entry.index)
				entry.detail.Text = string.format(
					"Possui %d â€¢ %s moedas â€¢ +%s/s â€¢ retorno %ss",
					driver.ownedAt(entry.index),
					formatNumber(cost),
					formatNumber(driver.marginalCpsAt(entry.index)),
					formatNumber(driver.paybackAt(entry.index))
				)
				paint(entry.button, coins >= cost)
			end

			local available = availableUpgrades()
			emptyLabel.Parent.Visible = #available == 0
			for slot, entry in upgradeRows do
				local upgrade = available[slot]
				entry.upgrade = upgrade
				entry.row.Visible = upgrade ~= nil
				if upgrade then
					entry.title.Text = upgrade.name
					entry.detail.Text = formatNumber(upgrade.price) .. " moedas â€¢ " .. upgrade.kind
					paint(entry.button, coins >= upgrade.price)
				end
			end
		end
	end

	-- Page: Eventos

	do
		local page = addPage("Eventos")

		section(page, "Coleta automÃ¡tica", "Recursos aguardam desbloqueio e disponibilidade no servidor.")
		toggleRow(page, "AutoExtraUI", "Clicar extras da interface", "Clica GoldenCoin e Wrinklers com reaÃ§Ã£o humana.")
		toggleRow(page, "AutoGolden", "Coletar moedas douradas", "Inclui as moedas de tempestades.")
		toggleRow(page, "AutoLump", "Colher lumps maduros", "Colhe apenas no estÃ¡gio ripe, com chance garantida.")
		toggleRow(page, "AutoMarket", "Comprar raros do mercado", "Inclui raros em estoque no planejador.")
		toggleRow(page, "AutoReward", "Resgatar recompensa", "Resgata apÃ³s cumprir os requisitos.")
		toggleRow(page, "AutoWrinkler", "Coletar wrinklers", "Estoura um por vez, perto do intervalo configurado.")
		numberRow(page, "WrinklerInterval", "Intervalo de wrinklers", "Segundos entre coletas automÃ¡ticas.", 10, 3600)

		section(page, "AÃ§Ãµes manuais")
		local lumpDetail = actionRow(page, "Colher lump agora", "", "Colher", function()
			if not harvestLump() then
				log("Colheita garantida ainda indisponÃ­vel")
			end
		end)
		local wrinklerDetail = actionRow(page, "Coletar wrinklers agora", "", "Coletar", function()
			if not ready() or #driver.wrinklers() == 0 then
				log("Nenhum wrinkler disponÃ­vel")
				return
			end
			requestWrinklers(os.clock())
			log("Coletando wrinklers")
		end)
		local rewardDetail = actionRow(page, "Recompensa de conquistas", "", "Resgatar", function()
			if not claimReward() then
				log("Recompensa jÃ¡ recebida ou requisitos incompletos")
			end
		end)

		section(page, "FeitiÃ§os", "Liberados pela Bola de Cristal; podem falhar e aplicar penalidades.")
		toggleRow(page, "AutoSpell", "ConjuraÃ§Ã£o automÃ¡tica", "Combo inteligente ou feitiÃ§o escolhido; aguarda mana.")
		local spellIds = { "auto" }
		for _, spell in Fortunes.list do
			table.insert(spellIds, spell.id)
		end
		local spellChoiceDetail = actionRow(page, "FeitiÃ§o automÃ¡tico", "", "Trocar", function()
			setOption("Spell", nextInList(spellIds, Config.Spell))
			log("FeitiÃ§o escolhido: " .. (SPELL_NAMES[Config.Spell] or Config.Spell))
		end)
		onOption("Spell", function(spellId)
			spellChoiceDetail.Text = SPELL_NAMES[spellId] or spellId
		end)

		local spellRows = {}
		for _, spell in Fortunes.list do
			local detail, castButton = actionRow(page, SPELL_NAMES[spell.id] or spell.name, "", "Conjurar", function()
				if not castSpell(spell) then
					log("FeitiÃ§o bloqueado ou mana insuficiente")
				end
			end)
			table.insert(spellRows, { spell = spell, detail = detail, button = castButton })
		end

		section(page, "Mercado de raros", "Estoque aparece apenas durante o evento de mercado.")
		local rareRows = {}
		for _, generator in Catalog.rares do
			local detail, buyButton = actionRow(page, generator.name, "", "Comprar 1", function()
				if ready() and (marketStock()[generator.id] or 0) > 0 and buyGenerator(generator.id, true) then
					log("Raro comprado: " .. generator.name)
				else
					log("Raro sem estoque ou saldo insuficiente")
				end
			end, { icon = generator.icon })
			table.insert(rareRows, { generator = generator, detail = detail, button = buyButton })
		end

		pageUpdaters.Eventos = function()
			if not ready() then
				return
			end
			local stage = driver.lumpStage()
			lumpDetail.Text = if stage == "locked"
				then "Desbloqueia com progresso no jogo."
				else string.format("EstÃ¡gio %s â€¢ possui %s â€¢ %d min", stage, formatNumber(driver.lumps()), math.ceil(driver.lumpRipeIn() / 60))
			wrinklerDetail.Text = #driver.wrinklers() .. " wrinklers disponÃ­veis."
			rewardDetail.Text = if driver.rewardOwned()
				then "Recompensa recebida."
				elseif driver.rewardReady() then "Recompensa pronta para resgatar."
				else driver.achievementCount() .. "/" .. #Achievements.list .. " conquistas."

			local mana, manaCap, coins = driver.mana(), driver.manaCap(), driver.coins()
			for _, entry in spellRows do
				local unlocked = driver.spellReady(entry.spell)
				local cost = driver.spellCost(entry.spell)
				entry.detail.Text = if unlocked
					then string.format("Mana %s/%s â€¢ custo %s", formatNumber(mana), formatNumber(manaCap), formatNumber(cost))
					else "Bloqueado: aumente a capacidade de mana."
				paint(entry.button, unlocked and mana >= cost)
			end

			local stock = marketStock()
			for _, entry in rareRows do
				local index = indexById[entry.generator.id]
				local price = driver.priceAt(index)
				local left = stock[entry.generator.id] or 0
				entry.detail.Text = string.format("Estoque %d â€¢ %s moedas â€¢ possui %d", left, formatNumber(price), driver.ownedAt(index))
				paint(entry.button, left > 0 and coins >= price)
			end
		end
	end

	-- Page: EstatÃ­sticas

	do
		local page = addPage("EstatÃ­sticas")

		section(page, "SessÃ£o e progresso", "Contadores da sessÃ£o e dados atuais do jogo.")
		local statsLabel = textCard(page, 300, 12)

		section(page, "Atalhos do jogo")
		actionRow(page, "EstatÃ­sticas completas", "Abre o painel de estatÃ­sticas do jogo.", "Abrir", function()
			if openGamePanel("Stats") then
				setWindowVisible(false)
			end
		end)
		actionRow(page, "Legado e prestÃ­gio", "Abre o painel de ascensÃ£o do jogo.", "Abrir", function()
			if openGamePanel("Legacy") then
				setWindowVisible(false)
			end
		end)

		section(page, "Registro de aÃ§Ãµes")
		local logsLabel = textCard(page, 190, 11)
		logsLabel.TextColor3 = Theme.Subtext

		pageUpdaters["EstatÃ­sticas"] = function()
			logsLabel.Text = table.concat(logs, "\n", 1, math.min(VISIBLE_LOGS, #logs))
			if not ready() then
				statsLabel.Text = "Aguardando o Coin Clickerâ€¦"
				return
			end

			local buffs = {}
			for _, buff in driver.buffs() do
				table.insert(buffs, buff.name or buff.kind or tostring(buff.id))
			end

			statsLabel.Text = table.concat({
				"SessÃ£o: " .. formatDuration(os.clock() - startedAt),
				"Sinais UI na moeda: " .. counters.uiSignals .. "   |   Remote 12/s: " .. counters.remoteClicks,
				"Extras clicados na interface: " .. counters.extraClicks,
				string.format("Pedidos: %d geradores â€¢ %d melhorias â€¢ %d raros", counters.buildings, counters.upgrades, counters.rares),
				string.format(
					"Douradas: %d   |   Lumps: %d   |   Wrinklers: %d   |   FeitiÃ§os: %d",
					counters.goldens,
					counters.harvests,
					counters.wrinklers,
					counters.spells
				),
				"",
				"Valor do clique: " .. formatNumber(driver.clickValue()) .. "   |   Geradores: " .. driver.totalOwned(),
				"Ganho total: " .. formatNumber(driver.totalEarned()) .. "   |   Gasto total: " .. formatNumber(driver.totalSpent()),
				"Conquistas: " .. driver.achievementCount() .. "/" .. #Achievements.list .. "   |   Cliques: " .. formatNumber(driver.totalClicks()),
				"PrestÃ­gio: " .. formatNumber(driver.prestigeLevel()) .. "   |   Ganho de ascensÃ£o: " .. formatNumber(driver.prestigeGain()),
				"Mana: " .. formatNumber(driver.mana()) .. "/" .. formatNumber(driver.manaCap()) .. "   |   Lumps: " .. formatNumber(driver.lumps()),
				"Buffs ativos: " .. (if #buffs > 0 then table.concat(buffs, ", ") else "nenhum"),
				string.format("Servidor: %.1f cliques/s â€¢ %s moedas/s", server.rate, formatNumber(server.income)),
				string.format("Compras confirmadas: %d â€¢ pendentes: %d â€¢ rejeitadas: %d", server.confirmed, #pending, server.rejected),
			}, "\n")
		end
	end

	-- Page: ConfiguraÃ§Ãµes

	do
		local page = addPage("ConfiguraÃ§Ãµes")

		section(page, "Interface", "Ajustes visuais do laptop e do jogo.")
		toggleRow(page, "CompactView", "Compact view", "Remove o fundo e a coluna central do laptop.")
		numberRow(page, "CompactScale", "Escala do compact view", "Tamanho das colunas no modo compacto, de 0.4 a 1.", 0.4, 1)
		toggleRow(page, "Performance", "Modo desempenho", "Remove partÃ­culas, nÃºmeros e som de clique.")

		section(page, "Controles", "ConfiguraÃ§Ã£o mantida enquanto a sessÃ£o estiver ativa.")
		toggleRow(page, "Paused", "Pausar automaÃ§Ãµes", "AÃ§Ãµes manuais continuam disponÃ­veis.")
		toggleRow(page, "Verbose", "DiagnÃ³stico no console", "Exibe registros e contadores no console.")
		actionRow(page, "Desligar automaÃ§Ãµes", "Desativa todos os recursos automÃ¡ticos.", "Desligar", function()
			for _, key in AUTOMATION_KEYS do
				setOption(key, false)
			end
			log("Todos os recursos automÃ¡ticos desligados")
		end)
		actionRow(page, "Zerar contadores", "Zera os contadores da sessÃ£o.", "Zerar", function()
			for key in counters do
				counters[key] = 0
			end
			log("Contadores da sessÃ£o zerados")
		end)
		actionRow(page, "Reconectar ao jogo", "Recria o backend privado do Coin Clicker.", "Reconectar", reconnect)
		actionRow(page, "Minimizar interface", "F8 ou botÃ£o AL reabre; o farm continua.", "Minimizar", function()
			setWindowVisible(false)
		end)
		actionRow(page, "Descarregar Auto Laptop", "Remove interface, conexÃµes, backend e restaura efeitos.", "Descarregar", session.Stop, {
			color = Theme.Danger,
		})
	end

	-- Interaction

	local function viewportSize()
		local camera = workspace.CurrentCamera
		return if camera then camera.ViewportSize else Vector2.new(1920, 1080)
	end

	local function isPointer(input)
		return input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch
	end

	local function makeDraggable(target, handles)
		local dragStart, dragOrigin
		local moved = false
		for _, handle in handles do
			janitor:connect(handle.InputBegan, function(input)
				if isPointer(input) then
					dragStart = input.Position
					dragOrigin = target.AbsolutePosition + target.AbsoluteSize * target.AnchorPoint
					moved = false
				end
			end)
		end
		janitor:connect(UserInputService.InputChanged, function(input)
			local inputType = input.UserInputType
			if not dragStart or (inputType ~= Enum.UserInputType.MouseMovement and inputType ~= Enum.UserInputType.Touch) then
				return
			end
			local delta = Vector2.new(input.Position.X - dragStart.X, input.Position.Y - dragStart.Y)
			if not moved and delta.Magnitude < DRAG_THRESHOLD then
				return
			end
			moved = true
			local viewport, size = viewportSize(), target.AbsoluteSize
			local low, high = size * target.AnchorPoint, viewport - size * (Vector2.one - target.AnchorPoint)
			local point = dragOrigin + delta
			target.Position = UDim2.fromOffset(
				math.clamp(point.X, low.X, math.max(low.X, high.X)),
				math.clamp(point.Y, low.Y, math.max(low.Y, high.Y))
			)
		end)
		janitor:connect(UserInputService.InputEnded, function(input)
			if isPointer(input) then
				dragStart = nil
			end
		end)
		return function()
			return moved
		end
	end

	makeDraggable(window, { topBar, sidebar })
	reopenDragged = makeDraggable(reopenButton, { reopenButton })

	janitor:connect(UserInputService.InputBegan, function(input, processed)
		if processed then
			return
		end
		if input.KeyCode == Enum.KeyCode.F6 then
			togglePause()
		elseif input.KeyCode == Enum.KeyCode.F7 then
			setOption("CompactView", not Config.CompactView)
		elseif input.KeyCode == Enum.KeyCode.F8 then
			setWindowVisible(not window.Visible)
		end
	end)

	-- Refresh

	local function refreshInterface()
		local viewport = viewportSize()
		windowScale.Scale = math.clamp(math.min((viewport.X - 24) / WINDOW_SIZE.X, (viewport.Y - 24) / WINDOW_SIZE.Y), 0.4, 1)

		local connected = ready()
		connectionDot.BackgroundColor3 = if connected then Theme.Success else Theme.Warning
		connectionLabel.Text = if connected then "Conectado" else "Aguardando jogo"

		if connected then
			local stamina = driver.stamina()
			coinsMetric.Text = formatNumber(server.coins or driver.coins())
			productionMetric.Text = formatNumber(driver.coinsPerSecond())
			staminaMetric.Text = string.format(if staminaResting then "%d%%  â€¢  recuperando" else "%d%%", math.floor(stamina * 100))
			staminaMetric.TextColor3 = if staminaResting then Theme.Warning elseif stamina < 0.2 then Theme.Danger else Theme.Success
		else
			coinsMetric.Text, productionMetric.Text, staminaMetric.Text = "â€”", "â€”", "â€”"
		end

		local update = window.Visible and pageUpdaters[selectedPage]
		if update then
			update()
		end
	end

	return refreshInterface, selectPage
end

local refreshInterface, selectPage = buildInterface()

-- Scheduler

local jobs = {}

local function every(interval, callback)
	table.insert(jobs, { interval = interval, callback = callback, last = os.clock() })
end

local function anyEnabled(keys)
	for _, key in keys do
		if Config[key] then
			return true
		end
	end
	return false
end

local function automate(keys, interval, action)
	local name = table.concat(keys, "/")
	every(interval, function(now)
		if Config.Paused or not ready() or not anyEnabled(keys) then
			return
		end
		if attempt(name, action, now) or failures[name] < MAX_FAILURES then
			return
		end
		for _, key in keys do
			setOption(key, false)
		end
		log("Desativado apÃ³s " .. MAX_FAILURES .. " falhas: " .. name, true)
	end)
end

automate({ "UISpam" }, 0, spamBigCoin)
automate({ "Remote12" }, 1 / 12, fireRemoteClick)
automate({ "AutoBuy", "AutoUpgrade", "AutoMarket" }, 0.3, runPurchases)
automate({ "AutoLump" }, 2, harvestLump)
automate({ "AutoReward" }, 2, claimReward)
automate({ "AutoSpell" }, 1, function()
	castSpell(pickSpell())
end)
every(COLLECT_TICK, function(now)
	if ready() then
		attempt("Coleta", runCollector, now)
	end
end)
every(1, function()
	attempt("Desempenho", syncPerformance)
end)
every(0.5, function()
	if Config.CompactView then
		attempt("Laptop compacto", compactLaptop)
	end
end)
every(0.35, function()
	attempt("Interface", refreshInterface)
end)
every(60, function()
	if Config.Verbose then
		print(
			LOG_PREFIX,
			string.format(
				"Servidor: %.1f cliques/s, %s moedas/s; confirmadas %d, rejeitadas %d",
				server.rate,
				formatNumber(server.income),
				server.confirmed,
				server.rejected
			)
		)
	end
end)

-- Start

bindGameGui(playerGui:FindFirstChild(GAME_GUI_NAME))

janitor:connect(playerGui.ChildAdded, function(child)
	if child.Name == GAME_GUI_NAME then
		bindGameGui(child)
	end
end)
janitor:connect(playerGui.ChildRemoved, function(child)
	if child == gameGui.root then
		bindGameGui(playerGui:FindFirstChild(GAME_GUI_NAME))
	end
end)
janitor:connect(remotes.Sync.OnClientEvent, onSync)
janitor:connect(RunService.Heartbeat, function()
	local now = os.clock()
	for _, job in jobs do
		local interval = if type(job.interval) == "function" then job.interval() else job.interval
		if now - job.last >= interval then
			job.last = now
			job.callback(now)
		end
	end
end)

selectPage("Painel")
attempt("Desempenho", syncPerformance)
attempt("Interface", refreshInterface)
log("Auto Laptop iniciado pausado; pressione F6 ou RETOMAR para comeÃ§ar.")
print(LOG_PREFIX, "Pronto. Pausado; pressione F6 ou RETOMAR para comeÃ§ar.")
