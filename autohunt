local mq = require('mq')
local ImGui = require('ImGui')
local okTheme, themeBridge = pcall(require, 'lib.maui_theme_bridge')
if not okTheme then
    themeBridge = { push = function() return nil end, pop = function() end }
end

--[[
AutoHunt v2 - Weighted Hunter / Puller for MQ Lua
Use only where automation is allowed on your server.
]]

local SCRIPT = 'AutoHunt'
local DATA_VERSION = 2
local SETTINGS_FILE = mq.configDir .. '/autohunt_settings.lua'
local LISTS_FILE = mq.configDir .. '/autohunt_lists.lua'
local PROFILES_FILE = mq.configDir .. '/autohunt_profiles.lua'
local ERROR_FILE = mq.configDir .. '/autohunt_error.log'

local function now_ms()
    local ok, v = pcall(function() return mq.gettime() end)
    if ok and type(v) == 'number' then return v end
    return math.floor(os.clock() * 1000)
end

local function ts_stamp()
    return os.date('%Y%m%d_%H%M%S')
end

local function sc(fn, fallback)
    local ok, v = pcall(fn)
    if ok then return v end
    return fallback
end

local function trim(s)
    s = tostring(s or '')
    return (s:gsub('^%s+', ''):gsub('%s+$', ''))
end

local function lower(s)
    return trim(s):lower()
end

local function clamp(v, lo, hi)
    if v < lo then return lo end
    if v > hi then return hi end
    return v
end

local function deepcopy(v)
    if type(v) ~= 'table' then return v end
    local out = {}
    for k, vv in pairs(v) do out[k] = deepcopy(vv) end
    return out
end

local function merge_defaults(dst, src)
    if type(src) ~= 'table' then return end
    for k, v in pairs(src) do
        if type(v) == 'table' then
            if type(dst[k]) ~= 'table' then dst[k] = {} end
            merge_defaults(dst[k], v)
        elseif dst[k] == nil then
            dst[k] = v
        end
    end
end

local defaults = {
    version = DATA_VERSION,
    enabled = true,
    show_ui = true,
    scan_radius = 180,
    scan_zradius = 70,
    rescan_delay = 1.2,
    min_hp_pct = 15,
    max_hp_pct = 100,
    allow_out_of_combat_scanning = true,
    check_collision = false,
    stick_to_current_target = true,
    switch_if_higher_priority_found = true,
    switch_score_margin = 10,
    do_face = false,
    do_stick = false,
    stick_command = '/stick hold 10',
    do_attack_on = false,
    humanize_actions = false,
    humanize_min_ms = 300,
    humanize_max_ms = 1200,
    pull_mode = false,
    pull_radius = 500,
    pull_approach_distance = 240,
    pull_command = '/ranged',
    pull_timeout_ms = 12000,
    pull_nav_recheck_ms = 300,
    pull_fallback_stick = true,
    assist_group_ma = true,
    only_aggro_on_group = false,
    prioritize_attacker_on_lowest_hp = true,
    defensive_mode = true,
    defensive_hp_threshold = 45,
    squishy_classes = { CLR = true, DRU = true, SHM = true, ENC = true, WIZ = true, MAG = true, NEC = true },
    safety_suspend_on_pc = false,
    safety_suspend_on_gm = true,
    safety_pc_radius = 120,
    safety_avoid_adds = true,
    safety_adds_radius = 65,
    safety_adds_threshold = 3,
    safety_max_combat_time_sec = 600,
    safety_no_target_timeout_sec = 180,
    safety_no_target_command = '/sit',
    safety_mana_suspend_enabled = false,
    safety_mana_suspend_pct = 20,
    safety_mana_suspend_while_standing = true,
    active_list_mode = 'zone',
    active_named_list = 'default',
    priority_actors = {},
    avoid_actors = {},
    scoring = {
        enabled = true,
        weights = {
            priority_pattern = 8.0,
            named_tier = 5.0,
            distance = 3.0,
            level_delta = 1.5,
            hp_pct = 2.5,
            difficulty = 1.2,
            aggro_group = 6.0,
            bodytype = 1.0,
            recent_kill = 2.0,
            loot = 1.0,
            group_defense = 5.0,
            custom = 1.0,
        },
        toggles = {
            distance = true,
            named_tier = true,
            level_delta = true,
            hp_pct = true,
            difficulty = false,
            aggro_group = true,
            bodytype = false,
            recent_kill = true,
            loot = false,
            group_defense = true,
            custom = true,
        },
        named_rare_patterns = { 'ancient', 'overlord', 'warlord', 'lord', 'queen', 'king' },
        bodytype_bonus = { undead = 15, giant = 5, dragon = 30 },
    },
    custom_score_rules = {},
    button_width = 120,
    button_height = 26,
    small_button_width = 44,
    priority_filter = '',
}

local settings = deepcopy(defaults)
local state = {
    running = true,
    status = 'Idle',
    suspended = false,
    suspend_reason = '',
    last_scan_ms = 0,
    last_zone_id = 0,
    candidates_seen = 0,
    best_live_line = 'None',
    last_choice_reason = 'n/a',
    last_target_id = 0,
    last_target_name = 'None',
    last_target_score = 0,
    no_target_since_ms = 0,
    combat_start_ms = 0,
    fast_switches = {},
    kill_cache = {},
    recent_damage_by_spawn = {},
    last_damage_cache_ms = 0,
    pull = { active = false, phase = 'idle', target_id = 0, target_name = '', started_ms = 0, last_nav_ms = 0 },
    add_pattern = '',
    add_minhp = '',
    add_required = false,
    add_avoid = '',
    profile_name = 'default',
    lists = { global = {}, zones = {}, named = { default = {} } },
    profiles = { version = DATA_VERSION, profiles = {} },
}

local AutoHunt = {}
_G.AutoHunt = AutoHunt

local function write_error_file(text)
    local ok, fh = pcall(io.open, ERROR_FILE, 'w')
    if ok and fh then fh:write(text or ''); fh:close() end
end

local function log(msg, color)
    local text = ('%s[%s]\\ax %s'):format(color or '\\at', SCRIPT, tostring(msg))
    if type(mq.msg) == 'function' and pcall(mq.msg, text, true) then return end
    if type(mq.cmd) == 'function' and pcall(mq.cmd, '/echo ' .. text) then return end
    print(text)
end
local function info(msg) log(msg, '\\at') end
local function warn(msg) log(msg, '\\ay') end
local function err(msg) log(msg, '\\ar') end

local function backup_file(path, stem)
    local f = io.open(path, 'rb')
    if not f then return end
    local data = f:read('*a')
    f:close()
    local backup = string.format('%s/%s_%s.bak.lua', mq.configDir, stem, ts_stamp())
    local b = io.open(backup, 'wb')
    if b then b:write(data); b:close() end
end

local function zone_short()
    return sc(function() return mq.TLO.Zone.ShortName() end, 'unknownzone')
end

local function zone_id()
    return sc(function() return tonumber(mq.TLO.Zone.ID()) or 0 end, 0)
end

local function me_id()
    return sc(function() return tonumber(mq.TLO.Me.ID()) or 0 end, 0)
end

local function me_level()
    return sc(function() return tonumber(mq.TLO.Me.Level()) or 1 end, 1)
end

local function in_combat()
    return sc(function() return mq.TLO.Me.Combat() == true end, false)
end

local function current_target_id()
    return sc(function() return tonumber(mq.TLO.Target.ID()) or 0 end, 0)
end

local function spawn_by_id(id)
    if not id or id == 0 then return nil end
    local s = mq.TLO.Spawn('id ' .. tostring(id))
    if not s or not s() then return nil end
    return s
end

local function normalize_entry(row)
    if type(row) ~= 'table' then return nil end
    local pattern = trim(row.pattern)
    if pattern == '' then return nil end
    local min_hp = nil
    if row.min_hp ~= nil and trim(row.min_hp) ~= '' then min_hp = tonumber(row.min_hp) end
    return { pattern = pattern, min_hp = min_hp, required = row.required == true }
end

local function normalize_list(list)
    local out = {}
    if type(list) ~= 'table' then return out end
    for i = 1, #list do
        local e = normalize_entry(list[i])
        if e then out[#out + 1] = e end
    end
    return out
end

local function normalize_avoid_list(list)
    local out = {}
    if type(list) ~= 'table' then return out end
    for i = 1, #list do
        local p = trim(list[i])
        if p ~= '' then out[#out + 1] = p end
    end
    return out
end

local function active_list_ref()
    if settings.active_list_mode == 'global' then return state.lists.global end
    if settings.active_list_mode == 'named' then
        local key = trim(settings.active_named_list)
        if key == '' then key = 'default' end
        settings.active_named_list = key
        if not state.lists.named[key] then state.lists.named[key] = {} end
        return state.lists.named[key]
    end
    local z = zone_short()
    if not state.lists.zones[z] then state.lists.zones[z] = {} end
    return state.lists.zones[z]
end

local function sync_settings_from_active_list()
    settings.priority_actors = deepcopy(active_list_ref())
end

local function sync_active_list_from_settings()
    local ref = active_list_ref()
    for k in pairs(ref) do ref[k] = nil end
    for i = 1, #settings.priority_actors do ref[i] = deepcopy(settings.priority_actors[i]) end
end

local function migrate_settings(data)
    if type(data) ~= 'table' then return deepcopy(defaults) end
    merge_defaults(data, defaults)
    data.version = DATA_VERSION
    data.priority_actors = normalize_list(data.priority_actors or {})
    data.avoid_actors = normalize_avoid_list(data.avoid_actors or {})
    data.scan_radius = clamp(tonumber(data.scan_radius) or defaults.scan_radius, 1, 700)
    data.scan_zradius = clamp(tonumber(data.scan_zradius) or defaults.scan_zradius, 1, 300)
    data.rescan_delay = clamp(tonumber(data.rescan_delay) or defaults.rescan_delay, 0.1, 10)
    data.min_hp_pct = clamp(tonumber(data.min_hp_pct) or defaults.min_hp_pct, 0, 100)
    data.max_hp_pct = clamp(tonumber(data.max_hp_pct) or defaults.max_hp_pct, 0, 100)
    data.pull_radius = clamp(tonumber(data.pull_radius) or defaults.pull_radius, 120, 1200)
    data.pull_approach_distance = clamp(tonumber(data.pull_approach_distance) or defaults.pull_approach_distance, 40, 500)
    data.button_width = clamp(tonumber(data.button_width) or defaults.button_width, 70, 260)
    data.button_height = clamp(tonumber(data.button_height) or defaults.button_height, 20, 44)
    data.small_button_width = clamp(tonumber(data.small_button_width) or defaults.small_button_width, 34, 120)
    return data
end

local function save_settings()
    backup_file(SETTINGS_FILE, 'autohunt_settings')
    local payload = deepcopy(settings)
    payload.priority_actors = normalize_list(payload.priority_actors or {})
    payload.avoid_actors = normalize_avoid_list(payload.avoid_actors or {})
    payload.version = DATA_VERSION
    mq.pickle(SETTINGS_FILE, payload)
end

local function load_settings()
    local fn = loadfile(SETTINGS_FILE)
    if not fn then settings = deepcopy(defaults); return end
    local ok, data = pcall(fn)
    if not ok then settings = deepcopy(defaults); return end
    settings = migrate_settings(data)
end

local function save_lists()
    backup_file(LISTS_FILE, 'autohunt_lists')
    local payload = { global = normalize_list(state.lists.global), zones = {}, named = {} }
    for k, v in pairs(state.lists.zones) do payload.zones[k] = normalize_list(v) end
    for k, v in pairs(state.lists.named) do payload.named[k] = normalize_list(v) end
    mq.pickle(LISTS_FILE, payload)
end

local function load_lists()
    local fn = loadfile(LISTS_FILE)
    if not fn then return end
    local ok, data = pcall(fn)
    if not ok or type(data) ~= 'table' then return end
    state.lists.global = normalize_list(data.global or {})
    state.lists.zones = {}
    state.lists.named = {}
    if type(data.zones) == 'table' then for k, v in pairs(data.zones) do state.lists.zones[k] = normalize_list(v) end end
    if type(data.named) == 'table' then for k, v in pairs(data.named) do state.lists.named[k] = normalize_list(v) end end
    if not state.lists.named.default then state.lists.named.default = {} end
end

local function save_profiles()
    backup_file(PROFILES_FILE, 'autohunt_profiles')
    local p = deepcopy(state.profiles)
    p.version = DATA_VERSION
    mq.pickle(PROFILES_FILE, p)
end

local function load_profiles()
    local fn = loadfile(PROFILES_FILE)
    if not fn then state.profiles = { version = DATA_VERSION, profiles = {} }; return end
    local ok, data = pcall(fn)
    if not ok or type(data) ~= 'table' then state.profiles = { version = DATA_VERSION, profiles = {} }; return end
    if type(data.profiles) ~= 'table' then data.profiles = {} end
    data.version = DATA_VERSION
    state.profiles = data
end

local function persist_all()
    sync_active_list_from_settings()
    save_lists()
    save_settings()
    save_profiles()
end

local function me_ready()
    if not sc(function() return mq.TLO.Me() ~= nil end, false) then return false, 'Me unavailable' end
    if sc(function() return mq.TLO.Me.Dead() == true end, false) then return false, 'Dead' end
    if sc(function() return mq.TLO.Me.Hovering() == true end, false) then return false, 'Hovering' end
    if sc(function() return mq.TLO.Me.Charmed() == true end, false) then return false, 'Charmed' end
    if sc(function() return mq.TLO.Me.Mezzed() == true end, false) then return false, 'Mezzed' end
    if settings.safety_mana_suspend_enabled then
        local mana = sc(function() return tonumber(mq.TLO.Me.PctMana()) or 100 end, 100)
        local sitting = sc(function() return mq.TLO.Me.Sitting() == true end, false)
        if mana < tonumber(settings.safety_mana_suspend_pct) and (not sitting or settings.safety_mana_suspend_while_standing) then
            return false, ('Mana low (%d%%)'):format(mana)
        end
    end
    return true, 'OK'
end

local function build_search(radius)
    local x = sc(function() return tonumber(mq.TLO.Me.X()) or 0 end, 0)
    local y = sc(function() return tonumber(mq.TLO.Me.Y()) or 0 end, 0)
    local z = sc(function() return tonumber(mq.TLO.Me.Z()) or 0 end, 0)
    local r = tonumber(radius) or tonumber(settings.scan_radius) or defaults.scan_radius
    return ('npc radius %d zradius %d loc %d %d %d'):format(r, tonumber(settings.scan_zradius) or defaults.scan_zradius, x, y, z)
end

local function is_on_xtarget(id)
    local xt = sc(function() return tonumber(mq.TLO.Me.XTarget()) or 0 end, 0)
    for i = 1, xt do
        local sid = sc(function() return tonumber(mq.TLO.Me.XTarget(i).ID()) or 0 end, 0)
        if sid == id then return true end
    end
    return false
end

local function update_recent_damage_cache()
    local n = now_ms()
    if (n - state.last_damage_cache_ms) < 300 then return end
    state.last_damage_cache_ms = n
    local xt = sc(function() return tonumber(mq.TLO.Me.XTarget()) or 0 end, 0)
    for i = 1, xt do
        local sid = sc(function() return tonumber(mq.TLO.Me.XTarget(i).ID()) or 0 end, 0)
        if sid > 0 then state.recent_damage_by_spawn[sid] = n end
    end
    for sid, t in pairs(state.recent_damage_by_spawn) do
        if (n - t) > 20000 then state.recent_damage_by_spawn[sid] = nil end
    end
end

local function nearby_pc_or_gm()
    local r = tonumber(settings.safety_pc_radius) or 120
    local meId = me_id()
    local pcCount = sc(function() return tonumber(mq.TLO.SpawnCount(('pc radius %d'):format(r))()) or 0 end, 0)
    if pcCount > 0 then
        local pcs = {}
        for i = 1, pcCount do
            local s = mq.TLO.NearestSpawn(i, ('pc radius %d'):format(r))
            if s and s() then
                local sid = sc(function() return tonumber(s.ID()) or 0 end, 0)
                if sid > 0 and sid ~= meId then
                    local gm = sc(function() return s.GM() == true end, false)
                    if settings.safety_suspend_on_gm and gm then return true, 'GM nearby' end
                    if settings.safety_suspend_on_pc then pcs[#pcs + 1] = sid end
                end
            end
        end
        if settings.safety_suspend_on_pc and #pcs > 0 then return true, ('PC nearby (%d)'):format(#pcs) end
    end
    return false, ''
end

local function too_many_adds()
    if not settings.safety_avoid_adds then return false end
    local c = sc(function() return tonumber(mq.TLO.SpawnCount(('npc radius %d'):format(tonumber(settings.safety_adds_radius) or 65))()) or 0 end, 0)
    return c > (tonumber(settings.safety_adds_threshold) or 3), c
end

local function priority_index(name, hp)
    local lname = lower(name)
    for i = 1, #settings.priority_actors do
        local row = settings.priority_actors[i]
        local pat = lower(row.pattern)
        if pat ~= '' and lname:find(pat, 1, true) then
            if row.min_hp ~= nil and hp < tonumber(row.min_hp) then
                if row.required then return nil end
            else
                return i
            end
        end
    end
    return nil
end

local function is_avoid_listed(name)
    local lname = lower(name)
    for i = 1, #settings.avoid_actors do
        local p = lower(settings.avoid_actors[i])
        if p ~= '' and lname:find(p, 1, true) then return true end
    end
    return false
end

local function is_valid_spawn(spawn)
    if not spawn or not spawn() then return false end
    if lower(sc(function() return spawn.Type() end, '')) ~= 'npc' then return false end
    if sc(function() return spawn.Dead() == true end, false) then return false end
    if sc(function() return spawn.Pet() == true end, false) then return false end
    if sc(function() return spawn.Charmed() == true end, false) then return false end
    if sc(function() return spawn.Mezzed() == true end, false) then return false end
    if sc(function() return spawn.Fleeing() == true end, false) then return false end
    local hp = sc(function() return tonumber(spawn.PctHPs()) or 0 end, 0)
    if hp <= 0 then return false end
    if hp < tonumber(settings.min_hp_pct) then return false end
    if hp > tonumber(settings.max_hp_pct) then return false end
    local owner = trim(sc(function() return spawn.Master() end, ''))
    if owner ~= '' and owner ~= 'NULL' then return false end
    if settings.check_collision then
        local los = sc(function() return spawn.LineOfSight() end, nil)
        if los ~= nil and los == false then return false end
    end
    return true
end

local function group_members_snapshot()
    local out = {}
    local n = sc(function() return tonumber(mq.TLO.Group()) or 0 end, 0)
    for i = 1, n do
        local m = mq.TLO.Group.Member(i)
        if m and m() then
            out[#out + 1] = {
                id = sc(function() return tonumber(m.ID()) or 0 end, 0),
                name = trim(sc(function() return m.CleanName() end, '')),
                hp = sc(function() return tonumber(m.PctHPs()) or 100 end, 100),
                class = trim(sc(function() return m.Class.ShortName() end, '')),
            }
        end
    end
    return out
end

local function lowest_hp_member(members)
    local low = nil
    for i = 1, #members do if not low or members[i].hp < low.hp then low = members[i] end end
    return low
end

local function get_ma_target_id()
    local maId = sc(function() return tonumber(mq.TLO.Group.MainAssist.ID()) or 0 end, 0)
    if maId > 0 then return sc(function() return tonumber(mq.TLO.Spawn('id ' .. tostring(maId)).Target.ID()) or 0 end, 0) end
    return sc(function() return tonumber(mq.TLO.Group.MainAssist.Target.ID()) or 0 end, 0)
end

local function named_tier(c)
    if not c.is_named then return 0 end
    local lname = lower(c.name)
    for i = 1, #(settings.scoring.named_rare_patterns or {}) do
        local p = lower(settings.scoring.named_rare_patterns[i])
        if p ~= '' and lname:find(p, 1, true) then return 1.0 end
    end
    return 0.6
end

local function dist_factor(d, maxr)
    local x = math.min((d / maxr), 1.0)
    return 1.0 - (x * x)
end

local function hp_factor(hp) return clamp((100 - hp) / 100, 0, 1) end

local function level_factor(c)
    local meLv = me_level()
    local delta = tonumber(c.level or meLv) - meLv
    if delta <= -5 then return 0.3 end
    if delta <= 2 then return 1.0 end
    if delta <= 5 then return 0.6 end
    return -0.4
end

local function difficulty_factor(c)
    if c.maxhp <= 0 then return 0 end
    local r = c.maxhp / 100000
    if r > 1 then return -clamp(r, 0, 3) / 3 end
    return 0.5 - r
end

local function bodytype_factor(c)
    local b = lower(c.body)
    if b == '' then return 0 end
    for k, v in pairs(settings.scoring.bodytype_bonus or {}) do
        if b:find(lower(k), 1, true) then return (tonumber(v) or 0) / 100 end
    end
    return 0
end

local function recent_kill_factor(c)
    local t = state.kill_cache[lower(c.name)]
    if not t then return 0 end
    local age = now_ms() - t
    if age > 90000 then return 0 end
    return -1.0 + (age / 90000)
end

local function loot_factor(c)
    local f = 0
    if c.is_named then f = f + 0.8 end
    if c.level >= me_level() then f = f + 0.2 end
    return clamp(f, 0, 1)
end

local function compile_rule(expr)
    local fn, loadErr = loadstring(expr or '')
    if not fn then return nil, loadErr end
    return fn
end

local function eval_custom_rules(c)
    if not settings.scoring.toggles.custom then return 0 end
    local total = 0
    for i = 1, #settings.custom_score_rules do
        local rule = settings.custom_score_rules[i]
        if rule.enabled then
            local fn = rule._fn
            if not fn then fn = compile_rule(rule.expr); rule._fn = fn end
            if fn then
                local env = { spawn = c.spawn, candidate = c, mq = mq, T = mq.TLO, state = state, settings = settings, math = math, string = string, tonumber = tonumber, tostring = tostring }
                setfenv(fn, env)
                local ok, v = pcall(fn)
                if ok and type(v) == 'number' then total = total + v end
            end
        end
    end
    return total / 100
end

local function gather_candidates(radius)
    local members = group_members_snapshot()
    local lowMember = lowest_hp_member(members)
    local maTarget = get_ma_target_id()
    local search = build_search(radius)
    local count = sc(function() return tonumber(mq.TLO.SpawnCount(search)()) or 0 end, 0)
    local list = {}
    for i = 1, count do
        local s = mq.TLO.NearestSpawn(i, search)
        if is_valid_spawn(s) then
            local id = sc(function() return tonumber(s.ID()) or 0 end, 0)
            local hp = sc(function() return tonumber(s.PctHPs()) or 0 end, 0)
            local name = trim(sc(function() return s.CleanName() end, ''))
            local c = {
                spawn = s,
                id = id,
                name = name,
                hp = hp,
                maxhp = sc(function() return tonumber(s.MaxHPs()) or 0 end, 0),
                dist = sc(function() return tonumber(s.Distance3D()) or tonumber(s.Distance()) or 99999 end, 99999),
                level = sc(function() return tonumber(s.Level()) or 1 end, 1),
                con = lower(sc(function() return s.ConColor() end, '')),
                body = trim(sc(function() return s.Body.Name() end, '')),
                race = trim(sc(function() return s.Race() end, '')),
                is_named = sc(function() return s.Named() == true end, false),
                aggro = sc(function() return s.Aggressive() == true end, false),
                on_xtarget = is_on_xtarget(id),
                target_of_id = sc(function() return tonumber(s.Target.ID()) or 0 end, 0),
                is_recent_dmg = state.recent_damage_by_spawn[id] and ((now_ms() - state.recent_damage_by_spawn[id]) <= 20000) or false,
                prio_idx = priority_index(name, hp),
                avoid = is_avoid_listed(name),
                group_low = lowMember,
                ma_target = maTarget,
            }
            if settings.only_aggro_on_group and not (c.on_xtarget or c.aggro or c.is_recent_dmg) then
                -- skip
            else
                list[#list + 1] = c
            end
        end
    end
    return list
end

local function score_candidate(c)
    if c.avoid then return -100000, 'avoid-list' end
    local w, t = settings.scoring.weights, settings.scoring.toggles
    local score, reasons = 0, {}
    local function add(name, v, weight)
        local s = (v or 0) * (weight or 0)
        score = score + s
        if math.abs(s) >= 1 then reasons[#reasons + 1] = string.format('%s:%+.1f', name, s) end
    end
    if c.prio_idx then add('priority', (1.0 - (c.prio_idx - 1) / math.max(#settings.priority_actors, 1)), w.priority_pattern) end
    if t.named_tier then add('named', named_tier(c), w.named_tier) end
    if t.distance then add('dist', dist_factor(c.dist, tonumber(settings.scan_radius) or 180), w.distance) end
    if t.level_delta then add('level', level_factor(c), w.level_delta) end
    if t.hp_pct then add('hp', hp_factor(c.hp), w.hp_pct) end
    if t.difficulty then add('difficulty', difficulty_factor(c), w.difficulty) end
    if t.aggro_group then add('aggro', (c.on_xtarget or c.aggro or c.is_recent_dmg) and 1 or 0, w.aggro_group) end
    if t.bodytype then add('body', bodytype_factor(c), w.bodytype) end
    if t.recent_kill then add('recent', recent_kill_factor(c), w.recent_kill) end
    if t.loot then add('loot', loot_factor(c), w.loot) end
    if t.group_defense then
        local gb = 0
        if settings.assist_group_ma and c.ma_target > 0 and c.id == c.ma_target then gb = gb + 1.2 end
        if settings.prioritize_attacker_on_lowest_hp and c.group_low and c.target_of_id == c.group_low.id then gb = gb + 1.0 end
        if settings.defensive_mode and c.group_low and c.group_low.hp <= tonumber(settings.defensive_hp_threshold) then
            if settings.squishy_classes[c.group_low.class] and c.target_of_id == c.group_low.id then gb = gb + 1.4 end
        end
        add('group', gb, w.group_defense)
    end
    local custom = eval_custom_rules(c)
    if custom ~= 0 then add('custom', custom, w.custom) end
    if #reasons == 0 then reasons[1] = 'fallback' end
    return score, table.concat(reasons, ', ')
end

local function best_candidate(radius)
    local list = gather_candidates(radius)
    state.candidates_seen = #list
    local best, bestScore, bestReason = nil, -999999, 'none'
    for i = 1, #list do
        local s, r = score_candidate(list[i])
        list[i].score, list[i].reason = s, r
        if s > bestScore then best, bestScore, bestReason = list[i], s, r end
    end
    if best then
        state.best_live_line = string.format('%s (%.1f) hp:%d dist:%.1f', best.name, best.score, best.hp, best.dist)
        state.last_choice_reason = bestReason
    else
        state.best_live_line = 'None'
    end
    return best
end

local function sleep_humanized()
    if not settings.humanize_actions then return end
    local lo = tonumber(settings.humanize_min_ms) or 300
    local hi = tonumber(settings.humanize_max_ms) or 1200
    if hi < lo then hi = lo end
    mq.delay(math.random(lo, hi))
end

local function target_spawn(id)
    if not id or id == 0 then return false end
    mq.cmd('/squelch /target id ' .. tostring(id))
    mq.delay(50)
    sleep_humanized()
    if settings.do_face then mq.cmd('/squelch /face fast') end
    sleep_humanized()
    if settings.do_stick and trim(settings.stick_command) ~= '' then mq.cmd('/squelch ' .. settings.stick_command) end
    sleep_humanized()
    if settings.do_attack_on then mq.cmd('/attack on') end
    return true
end

local function pull_reset(reason)
    if state.pull.active and sc(function() return mq.TLO.Navigation.Active() == true end, false) then mq.cmd('/nav stop') end
    state.pull.active = false
    state.pull.phase = 'idle'
    state.pull.target_id = 0
    state.pull.target_name = ''
    state.pull.started_ms = 0
    if reason and reason ~= '' then state.status = reason end
end

local function start_pull(c)
    if not c then return end
    state.pull.active = true
    state.pull.phase = 'approach'
    state.pull.target_id = c.id
    state.pull.target_name = c.name
    state.pull.started_ms = now_ms()
    state.pull.last_nav_ms = 0
end

local function nav_loaded()
    return sc(function() return mq.TLO.Plugin('mq2nav').IsLoaded() == true end, false)
end

local function update_pull_mode()
    if not settings.pull_mode then return false end
    if in_combat() then pull_reset('Combat active'); return false end
    if current_target_id() > 0 then return false end
    if not state.pull.active then
        local cand = best_candidate(tonumber(settings.pull_radius) or 500)
        if cand then start_pull(cand) else return false end
    end
    local s = spawn_by_id(state.pull.target_id)
    if not s or not s() then pull_reset('Pull target gone'); return false end
    if sc(function() return s.Dead() == true end, false) then pull_reset('Pull target dead'); return false end
    if now_ms() - state.pull.started_ms > (tonumber(settings.pull_timeout_ms) or 12000) then pull_reset('Pull timeout'); return false end

    local dist = sc(function() return tonumber(s.Distance3D()) or tonumber(s.Distance()) or 99999 end, 99999)
    local desired = tonumber(settings.pull_approach_distance) or 240
    mq.cmd('/squelch /target id ' .. tostring(state.pull.target_id))
    mq.delay(40)

    if dist > desired then
        if nav_loaded() then
            if (now_ms() - state.pull.last_nav_ms) >= (tonumber(settings.pull_nav_recheck_ms) or 300) then
                mq.cmd(('/nav id %d dist=%d'):format(state.pull.target_id, desired))
                state.pull.last_nav_ms = now_ms()
            end
            state.status = ('Pull nav: %s %.1f'):format(state.pull.target_name, dist)
            return true
        end
        if settings.pull_fallback_stick then
            mq.cmd('/squelch /face fast')
            mq.cmd('/squelch /stick id ' .. tostring(state.pull.target_id) .. ' loose 50')
            state.status = ('Pull stick fallback: %s'):format(state.pull.target_name)
            return true
        end
    end
    if nav_loaded() and sc(function() return mq.TLO.Navigation.Active() == true end, false) then mq.cmd('/nav stop') end
    local cmd = trim(settings.pull_command)
    if cmd ~= '' then mq.cmd(cmd) end
    pull_reset('Pull command complete')
    return true
end

local function current_target_snapshot()
    local id = current_target_id()
    if id == 0 then return nil end
    local s = spawn_by_id(id)
    if not is_valid_spawn(s) then return nil end
    local hp = sc(function() return tonumber(s.PctHPs()) or 0 end, 0)
    local c = {
        spawn = s,
        id = id,
        name = trim(sc(function() return s.CleanName() end, '')),
        hp = hp,
        maxhp = sc(function() return tonumber(s.MaxHPs()) or 0 end, 0),
        dist = sc(function() return tonumber(s.Distance3D()) or tonumber(s.Distance()) or 99999 end, 99999),
        level = sc(function() return tonumber(s.Level()) or 1 end, 1),
        con = lower(sc(function() return s.ConColor() end, '')),
        body = trim(sc(function() return s.Body.Name() end, '')),
        race = trim(sc(function() return s.Race() end, '')),
        is_named = sc(function() return s.Named() == true end, false),
        aggro = sc(function() return s.Aggressive() == true end, false),
        on_xtarget = is_on_xtarget(id),
        target_of_id = sc(function() return tonumber(s.Target.ID()) or 0 end, 0),
        is_recent_dmg = state.recent_damage_by_spawn[id] and ((now_ms() - state.recent_damage_by_spawn[id]) <= 20000) or false,
        prio_idx = priority_index(trim(sc(function() return s.CleanName() end, '')), hp),
        avoid = false,
        group_low = lowest_hp_member(group_members_snapshot()),
        ma_target = get_ma_target_id(),
    }
    c.score, c.reason = score_candidate(c)
    return c
end

local function track_target_switch()
    local t = now_ms()
    state.fast_switches[#state.fast_switches + 1] = t
    local kept = {}
    for i = 1, #state.fast_switches do
        if (t - state.fast_switches[i]) <= 8000 then kept[#kept + 1] = state.fast_switches[i] end
    end
    state.fast_switches = kept
    if #kept >= 6 then warn('Suspicious fast target switching detected.') end
end

local function do_scan()
    update_recent_damage_cache()
    local okMe, why = me_ready()
    if not okMe then state.status = why; return end
    local suspend, reason = nearby_pc_or_gm()
    if suspend then state.suspended = true; state.suspend_reason = reason; state.status = 'Suspended: ' .. reason; return end
    state.suspended = false
    state.suspend_reason = ''
    if not settings.allow_out_of_combat_scanning and not in_combat() then state.status = 'Waiting for combat'; return end
    if update_pull_mode() then return end

    if in_combat() then
        if state.combat_start_ms == 0 then state.combat_start_ms = now_ms() end
        if ((now_ms() - state.combat_start_ms) / 1000) > tonumber(settings.safety_max_combat_time_sec) then
            state.status = 'Max combat time reached'
            return
        end
    else
        state.combat_start_ms = 0
    end

    local best = best_candidate(tonumber(settings.scan_radius) or 180)
    if not best then
        if state.no_target_since_ms == 0 then state.no_target_since_ms = now_ms() end
        if ((now_ms() - state.no_target_since_ms) / 1000) > tonumber(settings.safety_no_target_timeout_sec) then
            local cmd = trim(settings.safety_no_target_command)
            if cmd ~= '' then mq.cmd(cmd) end
            state.no_target_since_ms = now_ms()
            warn('No target timeout action fired.')
        end
        state.status = 'No valid target'
        return
    end
    state.no_target_since_ms = 0

    local addWarn = too_many_adds()
    if addWarn then state.status = 'Add check blocked targeting'; return end

    local current = current_target_snapshot()
    if current and settings.stick_to_current_target and in_combat() then
        if not settings.switch_if_higher_priority_found then state.status = 'Sticking current'; return end
        if best.score <= (current.score + tonumber(settings.switch_score_margin)) then state.status = 'Current remains preferred'; return end
    end
    if current and current.id == best.id then state.status = 'Already on best target'; return end

    if target_spawn(best.id) then
        track_target_switch()
        state.last_target_id = best.id
        state.last_target_name = best.name
        state.last_target_score = best.score
        state.last_choice_reason = best.reason
        state.status = ('Targeted: %s'):format(best.name)
        info(('Targeting: %s [Score %.1f HP:%d Dist:%.1f] reason=%s'):format(best.name, best.score, best.hp, best.dist, best.reason))
    end
end

local function add_actor(pattern, min_hp, required)
    local p = trim(pattern)
    if p == '' then return false, 'Pattern is empty.' end
    settings.priority_actors[#settings.priority_actors + 1] = { pattern = p, min_hp = min_hp, required = required == true }
    sync_active_list_from_settings()
    return true, ('Added actor: %s'):format(p)
end

local function add_avoid(pattern)
    local p = trim(pattern)
    if p == '' then return false, 'Avoid pattern empty.' end
    settings.avoid_actors[#settings.avoid_actors + 1] = p
    return true, ('Added avoid: %s'):format(p)
end

local function add_current_target()
    local tid = current_target_id()
    if tid == 0 then return false, 'No current target.' end
    local s = spawn_by_id(tid)
    if not s then return false, 'Target spawn unavailable.' end
    return add_actor(trim(sc(function() return s.CleanName() end, '')), nil, false)
end

local function remove_actor(index)
    index = tonumber(index)
    if not index or index < 1 or index > #settings.priority_actors then return false, 'Invalid index.' end
    local r = settings.priority_actors[index]
    table.remove(settings.priority_actors, index)
    sync_active_list_from_settings()
    return true, ('Removed: %s'):format(r.pattern)
end

local function move_actor(index, delta)
    index = tonumber(index)
    local ni = index and (index + delta) or nil
    if not index or not ni then return false, 'Invalid move.' end
    if index < 1 or index > #settings.priority_actors then return false, 'Index out of range.' end
    if ni < 1 or ni > #settings.priority_actors then return false, 'Cannot move.' end
    settings.priority_actors[index], settings.priority_actors[ni] = settings.priority_actors[ni], settings.priority_actors[index]
    sync_active_list_from_settings()
    return true, ('Moved %d -> %d'):format(index, ni)
end

local function set_mode(mode, key)
    mode = lower(mode)
    if mode ~= 'zone' and mode ~= 'global' and mode ~= 'named' then return false, 'Mode must be zone/global/named.' end
    settings.active_list_mode = mode
    if mode == 'named' then
        key = trim(key)
        if key == '' then key = 'default' end
        settings.active_named_list = key
    end
    sync_settings_from_active_list()
    return true, 'Mode set to ' .. mode
end

local function profile_save(name)
    name = trim(name)
    if name == '' then return false, 'Profile name required.' end
    sync_active_list_from_settings()
    state.profiles.profiles[name] = { version = DATA_VERSION, settings = deepcopy(settings), lists = deepcopy(state.lists) }
    save_profiles()
    return true, 'Profile saved: ' .. name
end

local function profile_load(name)
    name = trim(name)
    local p = state.profiles.profiles[name]
    if not p then return false, 'Profile not found: ' .. name end
    settings = migrate_settings(deepcopy(p.settings or {}))
    if type(p.lists) == 'table' then state.lists = deepcopy(p.lists) end
    if not state.lists.named then state.lists.named = { default = {} } end
    if not state.lists.named.default then state.lists.named.default = {} end
    sync_settings_from_active_list()
    state.profile_name = name
    return true, 'Profile loaded: ' .. name
end

local function profile_delete(name)
    name = trim(name)
    if state.profiles.profiles[name] then
        state.profiles.profiles[name] = nil
        save_profiles()
        return true, 'Profile deleted: ' .. name
    end
    return false, 'Profile not found.'
end

local function profile_list()
    local names = {}
    for k in pairs(state.profiles.profiles) do names[#names + 1] = k end
    table.sort(names)
    if #names == 0 then info('Profiles: none') else info('Profiles: ' .. table.concat(names, ', ')) end
end

local function export_list(path)
    path = trim(path)
    if path == '' then path = mq.configDir .. '/autohunt_export.txt' end
    local f = io.open(path, 'wb')
    if not f then return false, 'Cannot open file: ' .. path end
    for i = 1, #settings.priority_actors do
        local r = settings.priority_actors[i]
        f:write(('%s|%s|%s\n'):format(r.pattern, r.min_hp or '', r.required and '1' or '0'))
    end
    f:close()
    return true, 'Exported list to ' .. path
end

local function import_list(path)
    path = trim(path)
    if path == '' then path = mq.configDir .. '/autohunt_export.txt' end
    local f = io.open(path, 'rb')
    if not f then return false, 'Cannot open file: ' .. path end
    local out = {}
    for line in f:lines() do
        local p, m, r = line:match('^(.-)%|(.-)%|(.-)$')
        if p and trim(p) ~= '' then out[#out + 1] = { pattern = trim(p), min_hp = tonumber(trim(m)), required = trim(r) == '1' } end
    end
    f:close()
    settings.priority_actors = normalize_list(out)
    sync_active_list_from_settings()
    return true, ('Imported %d rows'):format(#settings.priority_actors)
end

local function help()
    info('Commands:')
    info('/autohunt on|off|toggle|ui|save|load|list|add <pattern>|addtarget|del <i>|up <i>|down <i>')
    info('/autohunt avoid add <pattern>|avoid del <i>|avoid list')
    info('/autohunt mode zone|global|named [key] | radius <n> | zradius <n> | delay <sec> | minhp <n> | maxhp <n>')
    info('/autohunt pull on|off|toggle')
    info('/autohunt profile save|load|list|delete <name>')
    info('/autohunt exportlist [path] | importlist [path]')
end

local function cmd(line)
    local args = trim(line)
    if args == '' or lower(args) == 'help' then help(); return end
    local l = lower(args)
    if l == 'on' then settings.enabled = true; return end
    if l == 'off' then settings.enabled = false; return end
    if l == 'toggle' then settings.enabled = not settings.enabled; return end
    if l == 'ui' then settings.show_ui = not settings.show_ui; return end
    if l == 'save' then persist_all(); info('Saved.'); return end
    if l == 'load' then load_lists(); load_settings(); load_profiles(); sync_settings_from_active_list(); info('Reloaded.'); return end
    if l == 'list' then for i = 1, #settings.priority_actors do local r=settings.priority_actors[i]; info(('%d) %s'):format(i, r.pattern)) end return end
    if l == 'addtarget' then local ok, m = add_current_target(); if ok then info(m) else warn(m) end; return end

    local p = args:match('^add%s+(.+)$')
    if p then local ok, m = add_actor(p, nil, false); if ok then info(m) else warn(m) end; return end
    local idx = args:match('^del%s+(%d+)$')
    if idx then local ok, m = remove_actor(idx); if ok then info(m) else warn(m) end; return end
    idx = args:match('^up%s+(%d+)$')
    if idx then local ok, m = move_actor(idx, -1); if ok then info(m) else warn(m) end; return end
    idx = args:match('^down%s+(%d+)$')
    if idx then local ok, m = move_actor(idx, 1); if ok then info(m) else warn(m) end; return end

    local amode, aarg = args:match('^mode%s+(%S+)%s*(.-)$')
    if amode then local ok, m = set_mode(amode, aarg); if ok then info(m) else warn(m) end; return end
    local n = args:match('^radius%s+(%d+)$'); if n then settings.scan_radius = clamp(tonumber(n), 1, 700); return end
    n = args:match('^zradius%s+(%d+)$'); if n then settings.scan_zradius = clamp(tonumber(n), 1, 300); return end
    local f = args:match('^delay%s+([%d%.]+)$'); if f then settings.rescan_delay = clamp(tonumber(f) or defaults.rescan_delay, 0.1, 10); return end
    n = args:match('^minhp%s+(%d+)$'); if n then settings.min_hp_pct = clamp(tonumber(n), 0, 100); return end
    n = args:match('^maxhp%s+(%d+)$'); if n then settings.max_hp_pct = clamp(tonumber(n), 0, 100); return end

    local pullarg = args:match('^pull%s+(%S+)$')
    if pullarg then pullarg = lower(pullarg); if pullarg == 'on' then settings.pull_mode = true elseif pullarg == 'off' then settings.pull_mode = false else settings.pull_mode = not settings.pull_mode end; return end
    local what, rest = args:match('^avoid%s+(%S+)%s*(.-)$')
    if what then
        what = lower(what)
        if what == 'add' then local ok, m = add_avoid(rest); if ok then info(m) else warn(m) end; return end
        if what == 'del' then local i = tonumber(rest); if i and i >= 1 and i <= #settings.avoid_actors then table.remove(settings.avoid_actors, i); info('Avoid removed.') else warn('Invalid avoid index.') end; return end
        if what == 'list' then for i = 1, #settings.avoid_actors do info(('%d) %s'):format(i, settings.avoid_actors[i])) end; return end
    end

    local pa, pb = args:match('^profile%s+(%S+)%s*(.-)$')
    if pa then pa = lower(pa); if pa == 'save' then local ok,m=profile_save(pb); if ok then info(m) else warn(m) end; return end
        if pa == 'load' then local ok,m=profile_load(pb); if ok then info(m) else warn(m) end; return end
        if pa == 'delete' then local ok,m=profile_delete(pb); if ok then info(m) else warn(m) end; return end
        if pa == 'list' then profile_list(); return end
    end
    local ep = args:match('^exportlist%s*(.-)$'); if ep ~= nil then local ok,m=export_list(ep); if ok then info(m) else warn(m) end; return end
    local ip = args:match('^importlist%s*(.-)$'); if ip ~= nil then local ok,m=import_list(ip); if ok then info(m) else warn(m) end; return end
    help()
end

local function btn(label, width)
    local w = tonumber(width) or tonumber(settings.button_width) or defaults.button_width
    local h = tonumber(settings.button_height) or defaults.button_height
    return ImGui.Button(label, w, h)
end

local function sbtn(label, width)
    local w = tonumber(width) or tonumber(settings.small_button_width) or defaults.small_button_width
    local h = tonumber(settings.button_height) or defaults.button_height
    return ImGui.Button(label, w, h)
end

local function ui_settings_tab()
    if ImGui.CollapsingHeader('Core', ImGuiTreeNodeFlags.DefaultOpen) then
        settings.enabled = ImGui.Checkbox('Enable Auto Hunt', settings.enabled)
        settings.allow_out_of_combat_scanning = ImGui.Checkbox('Allow Out of Combat Scanning', settings.allow_out_of_combat_scanning)
        settings.scan_radius = ImGui.SliderInt('Scan Radius', tonumber(settings.scan_radius) or defaults.scan_radius, 1, 700)
        settings.scan_zradius = ImGui.SliderInt('Scan Height Radius', tonumber(settings.scan_zradius) or defaults.scan_zradius, 1, 300)
        settings.rescan_delay = ImGui.SliderFloat('Rescan Delay (sec)', tonumber(settings.rescan_delay) or defaults.rescan_delay, 0.1, 5.0, '%.2f')
        settings.min_hp_pct = ImGui.SliderInt('Min HP %', tonumber(settings.min_hp_pct) or defaults.min_hp_pct, 0, 100)
        settings.max_hp_pct = ImGui.SliderInt('Max HP %', tonumber(settings.max_hp_pct) or defaults.max_hp_pct, 0, 100)
    end
    if ImGui.CollapsingHeader('Scoring', ImGuiTreeNodeFlags.DefaultOpen) then
        settings.scoring.enabled = ImGui.Checkbox('Use Weighted Scoring', settings.scoring.enabled)
        settings.scoring.weights.priority_pattern = ImGui.SliderFloat('W Priority', settings.scoring.weights.priority_pattern, 0.0, 15.0, '%.1f')
        settings.scoring.weights.named_tier = ImGui.SliderFloat('W Named', settings.scoring.weights.named_tier, 0.0, 15.0, '%.1f')
        settings.scoring.weights.distance = ImGui.SliderFloat('W Distance', settings.scoring.weights.distance, 0.0, 15.0, '%.1f')
        settings.scoring.weights.hp_pct = ImGui.SliderFloat('W HP%', settings.scoring.weights.hp_pct, 0.0, 15.0, '%.1f')
        settings.scoring.weights.aggro_group = ImGui.SliderFloat('W Aggro', settings.scoring.weights.aggro_group, 0.0, 15.0, '%.1f')
        settings.scoring.weights.group_defense = ImGui.SliderFloat('W Group Def', settings.scoring.weights.group_defense, 0.0, 15.0, '%.1f')
        settings.switch_score_margin = ImGui.SliderInt('Switch Margin', tonumber(settings.switch_score_margin) or 10, 0, 100)
    end
    if ImGui.CollapsingHeader('Puller', ImGuiTreeNodeFlags.DefaultOpen) then
        settings.pull_mode = ImGui.Checkbox('Pull Mode', settings.pull_mode)
        settings.pull_radius = ImGui.SliderInt('Pull Radius', tonumber(settings.pull_radius) or defaults.pull_radius, 120, 1200)
        settings.pull_approach_distance = ImGui.SliderInt('Pull Approach Dist', tonumber(settings.pull_approach_distance) or defaults.pull_approach_distance, 40, 500)
        settings.pull_command = ImGui.InputText('Pull Command', settings.pull_command, 256)
    end
    if ImGui.CollapsingHeader('Group', ImGuiTreeNodeFlags.DefaultOpen) then
        settings.assist_group_ma = ImGui.Checkbox('Assist Group MA', settings.assist_group_ma)
        settings.only_aggro_on_group = ImGui.Checkbox('Only Aggro on Group', settings.only_aggro_on_group)
        settings.prioritize_attacker_on_lowest_hp = ImGui.Checkbox('Prioritize on Lowest HP Member', settings.prioritize_attacker_on_lowest_hp)
        settings.defensive_mode = ImGui.Checkbox('Defensive Mode', settings.defensive_mode)
        settings.defensive_hp_threshold = ImGui.SliderInt('Defensive HP Threshold', tonumber(settings.defensive_hp_threshold) or 45, 1, 100)
    end
    if ImGui.CollapsingHeader('Safety', ImGuiTreeNodeFlags.DefaultOpen) then
        settings.safety_suspend_on_pc = ImGui.Checkbox('Suspend on PC', settings.safety_suspend_on_pc)
        settings.safety_suspend_on_gm = ImGui.Checkbox('Suspend on GM', settings.safety_suspend_on_gm)
        settings.safety_pc_radius = ImGui.SliderInt('PC/GM Radius', tonumber(settings.safety_pc_radius) or 120, 20, 400)
        settings.safety_avoid_adds = ImGui.Checkbox('Avoid Adds', settings.safety_avoid_adds)
        settings.safety_adds_radius = ImGui.SliderInt('Adds Radius', tonumber(settings.safety_adds_radius) or 65, 20, 150)
        settings.safety_adds_threshold = ImGui.SliderInt('Adds Threshold', tonumber(settings.safety_adds_threshold) or 3, 1, 10)
        settings.safety_max_combat_time_sec = ImGui.SliderInt('Max Combat Sec', tonumber(settings.safety_max_combat_time_sec) or 600, 30, 1800)
        settings.safety_no_target_timeout_sec = ImGui.SliderInt('No Target Timeout Sec', tonumber(settings.safety_no_target_timeout_sec) or 180, 30, 1200)
        settings.safety_no_target_command = ImGui.InputText('No Target Command', settings.safety_no_target_command, 256)
    end
    if ImGui.CollapsingHeader('Visuals', ImGuiTreeNodeFlags.DefaultOpen) then
        settings.button_width = ImGui.SliderInt('Button Width', tonumber(settings.button_width) or defaults.button_width, 70, 260)
        settings.button_height = ImGui.SliderInt('Button Height', tonumber(settings.button_height) or defaults.button_height, 20, 44)
        settings.small_button_width = ImGui.SliderInt('Small Button Width', tonumber(settings.small_button_width) or defaults.small_button_width, 34, 120)
    end
end

local function ui_priority_tab()
    settings.priority_filter = ImGui.InputText('Search', settings.priority_filter or '', 128)
    state.add_pattern = ImGui.InputText('Actor Name/Pattern', state.add_pattern, 256)
    state.add_minhp = ImGui.InputText('Min HP (optional)', state.add_minhp, 32)
    state.add_required = ImGui.Checkbox('Required', state.add_required)
    if btn('Add Actor') then local mh=tonumber(trim(state.add_minhp)); if trim(state.add_minhp)=='' then mh=nil end; local ok,m=add_actor(state.add_pattern,mh,state.add_required); if ok then state.add_pattern=''; state.add_minhp=''; state.add_required=false; info(m) else warn(m) end end
    ImGui.SameLine()
    if btn('Add Current Target', 170) then local ok,m=add_current_target(); if ok then info(m) else warn(m) end end
    ImGui.Separator()
    local flags = ImGuiTableFlags.Borders + ImGuiTableFlags.RowBg + ImGuiTableFlags.Resizable
    if ImGui.BeginTable('##PriorityTable', 6, flags) then
        ImGui.TableSetupColumn('#'); ImGui.TableSetupColumn('Pattern'); ImGui.TableSetupColumn('MinHP'); ImGui.TableSetupColumn('Req')
        ImGui.TableSetupColumn('Order', ImGuiTableColumnFlags.WidthFixed, 96); ImGui.TableSetupColumn('Action', ImGuiTableColumnFlags.WidthFixed, 48); ImGui.TableHeadersRow()
        local filter = lower(settings.priority_filter or '')
        for i=1,#settings.priority_actors do
            local r = settings.priority_actors[i]
            if filter=='' or lower(r.pattern):find(filter,1,true) then
                ImGui.TableNextRow()
                ImGui.TableSetColumnIndex(0); ImGui.Text(tostring(i))
                ImGui.TableSetColumnIndex(1); if is_avoid_listed(r.pattern) then ImGui.TextColored(1.0,0.35,0.35,1.0,r.pattern) else ImGui.TextColored(0.35,1.0,0.35,1.0,r.pattern) end
                ImGui.TableSetColumnIndex(2); ImGui.Text(r.min_hp and tostring(r.min_hp) or '-')
                ImGui.TableSetColumnIndex(3); ImGui.Text(r.required and 'Y' or 'N')
                ImGui.TableSetColumnIndex(4); if sbtn('Up##'..i,44) then move_actor(i,-1) end; ImGui.SameLine(); if sbtn('Dn##'..i,44) then move_actor(i,1) end
                ImGui.TableSetColumnIndex(5); if sbtn('Del##'..i,44) then remove_actor(i); break end
            end
        end
        ImGui.EndTable()
    end
    ImGui.Separator()
    state.add_avoid = ImGui.InputText('Avoid Pattern', state.add_avoid, 256)
    if btn('Add Avoid', 140) then local ok,m=add_avoid(state.add_avoid); if ok then state.add_avoid=''; info(m) else warn(m) end end
    ImGui.SameLine(); ImGui.Text(('Avoid entries: %d'):format(#settings.avoid_actors))
end

local function ui_profiles_tab()
    state.profile_name = ImGui.InputText('Profile Name', state.profile_name, 128)
    if btn('Profile Save',130) then local ok,m=profile_save(state.profile_name); if ok then info(m) else warn(m) end end
    ImGui.SameLine(); if btn('Profile Load',130) then local ok,m=profile_load(state.profile_name); if ok then info(m) else warn(m) end end
    ImGui.SameLine(); if btn('Profile Delete',130) then local ok,m=profile_delete(state.profile_name); if ok then info(m) else warn(m) end end
    ImGui.SameLine(); if btn('Profile List',130) then profile_list() end
    ImGui.Separator()
    if btn('Export List',130) then local ok,m=export_list(''); if ok then info(m) else warn(m) end end
    ImGui.SameLine(); if btn('Import List',130) then local ok,m=import_list(''); if ok then info(m) else warn(m) end end
end

local function ui_status_tab()
    ImGui.Text(('Enabled: %s'):format(settings.enabled and 'YES' or 'NO'))
    ImGui.Text(('Status: %s'):format(state.status))
    if state.suspended then ImGui.TextColored(1.0,0.4,0.4,1.0,'Suspended: '..state.suspend_reason) end
    ImGui.Text(('Zone: %s'):format(zone_short()))
    ImGui.Text(('Candidates: %d'):format(state.candidates_seen))
    ImGui.Text(('Best: %s'):format(state.best_live_line))
    ImGui.Text(('Reason: %s'):format(state.last_choice_reason))
    ImGui.Text(('Last Target: %s (%d) score=%.1f'):format(state.last_target_name, state.last_target_id, state.last_target_score))
    ImGui.Text(('Pull state: %s'):format(state.pull.phase))
end

local function draw_action_bar()
    if btn(settings.enabled and 'Disable' or 'Enable') then settings.enabled = not settings.enabled end
    ImGui.SameLine(); if btn('Force Scan') then state.last_scan_ms = 0; do_scan() end
    ImGui.SameLine(); if btn('Add Current Target',170) then local ok,m=add_current_target(); if ok then info(m) else warn(m) end end
    ImGui.SameLine(); if btn('Save') then persist_all(); info('Saved settings/lists/profiles.') end
    ImGui.SameLine(); if btn('Reload') then load_lists(); load_settings(); load_profiles(); sync_settings_from_active_list(); info('Reloaded settings/lists/profiles.') end
    ImGui.SameLine(); if btn('Close') then settings.show_ui = false end
    settings.pull_mode = ImGui.Checkbox('Pull Mode', settings.pull_mode)
    ImGui.SameLine(); settings.assist_group_ma = ImGui.Checkbox('Assist MA', settings.assist_group_ma)
    ImGui.SameLine(); settings.only_aggro_on_group = ImGui.Checkbox('Aggro Group Only', settings.only_aggro_on_group)
    ImGui.SameLine(); ImGui.TextDisabled(('Best: %s'):format(state.best_live_line))
    ImGui.Separator()
end

local function draw_ui()
    if not settings.show_ui then return end
    local token = themeBridge.push()
    ImGui.SetNextWindowSize(1080, 760, ImGuiCond.FirstUseEver)
    settings.show_ui = ImGui.Begin('AutoHunt##Main', settings.show_ui, ImGuiWindowFlags.NoCollapse)
    if settings.show_ui then
        ImGui.TextColored(0.6, 0.95, 1.0, 1.0, 'AutoHunt v2 - Weighted Hunter / Puller')
        ImGui.SameLine(); ImGui.TextDisabled('Reason-aware target selection for MQ boxing/hunting')
        draw_action_bar()
        if ImGui.BeginTabBar('##AHTabs') then
            if ImGui.BeginTabItem('Settings') then ui_settings_tab(); ImGui.EndTabItem() end
            if ImGui.BeginTabItem('Priority') then ui_priority_tab(); ImGui.EndTabItem() end
            if ImGui.BeginTabItem('Profiles') then ui_profiles_tab(); ImGui.EndTabItem() end
            if ImGui.BeginTabItem('Status') then ui_status_tab(); ImGui.EndTabItem() end
            ImGui.EndTabBar()
        end
    end
    ImGui.End()
    themeBridge.pop(token)
end

local function on_zone_change()
    local zid = zone_id()
    if zid ~= state.last_zone_id then
        state.last_zone_id = zid
        if settings.active_list_mode == 'zone' then sync_settings_from_active_list() end
        pull_reset('Zone changed')
    end
end

local function monitor_target_kill()
    local tid = state.last_target_id
    if tid <= 0 then return end
    local s = spawn_by_id(tid)
    if s and s() and sc(function() return s.Dead() == true end, false) then
        local n = lower(trim(sc(function() return s.CleanName() end, '')))
        if n ~= '' then state.kill_cache[n] = now_ms() end
        state.last_target_id = 0
    end
end

local function init()
    load_lists(); load_settings(); load_profiles()
    if not state.lists.named.default then state.lists.named.default = {} end
    sync_settings_from_active_list()
    state.last_zone_id = zone_id()
    AutoHunt.settings = settings
    AutoHunt.state = state
    AutoHunt.enabled = settings.enabled
    AutoHunt.show_ui = settings.show_ui
    AutoHunt.persist = persist_all
    mq.bind('/autohunt', cmd)
    mq.imgui.init('AutoHunt', draw_ui)
    info('Loaded AutoHunt v2.')
end

local function cleanup()
    persist_all()
    pull_reset('')
    mq.unbind('/autohunt')
    info('Shutdown complete.')
end

local function main()
    init()
    while state.running do
        if AutoHunt.enabled ~= settings.enabled then settings.enabled = AutoHunt.enabled == true end
        if AutoHunt.show_ui ~= settings.show_ui then settings.show_ui = AutoHunt.show_ui == true end
        on_zone_change()
        monitor_target_kill()
        if settings.enabled then
            local delay_ms = math.floor((tonumber(settings.rescan_delay) or 1.2) * 1000)
            if now_ms() - state.last_scan_ms >= delay_ms then
                state.last_scan_ms = now_ms()
                local ok, scan_err = xpcall(do_scan, function(e) return debug.traceback(e, 2) end)
                if not ok then state.status = 'Scan error'; err(scan_err); write_error_file(scan_err) end
            end
        else
            state.status = 'Disabled'
        end
        mq.delay(50)
    end
    cleanup()
end

local ok, boot_err = xpcall(main, function(e) return debug.traceback(e, 2) end)
if not ok then
    write_error_file(boot_err)
    err('Fatal startup/runtime error. Full traceback written to: ' .. ERROR_FILE)
    err(boot_err)
end
