gl.setup(NATIVE_WIDTH, NATIVE_HEIGHT)

node.alias "*"
util.no_globals()
math.randomseed(os.time())
local json = require "json"
local loader = require "loader"
local helper = require "helper"
local placement = require "placement"
local easing = require "easing"
local qrcode_overlay = require "qrcode_overlay"
local qr_code_instances = {}
local device_info = nil
local device_info_page_mode = false
local system_info = nil
local system_info_page_mode = false

local function create_or_update_qr_instance(asset_id, position_config)
    local instance_id = "qr_" .. tostring(asset_id)
    
    local default_config = {
        position = "bottom-right",
        margin = 20,
        custom_x = 0,
        custom_y = 0
    }
    
    local final_config = {}
    for k, v in pairs(default_config) do
        final_config[k] = v
    end
    if position_config then
        for k, v in pairs(position_config) do
            final_config[k] = v
        end
    end
    
    -- Create or update the instance
    local existing = qr_code_instances[instance_id]
    if existing then
        existing.trigger_data = tostring(asset_id)
        existing.position_config = final_config
        existing.draw_details = nil
        existing.is_visible = false
    else
        qr_code_instances[instance_id] = {
            id = instance_id,
            trigger_data = tostring(asset_id),
            position_config = final_config,
            draw_details = nil,
            is_visible = false
        }
    end
    
    -- Auto-save after creating/updating instance
    save_qr_instances()
    
    return instance_id
end

local function remove_qr_instance(asset_id)
    local instance_id = "qr_" .. tostring(asset_id)
    if qr_code_instances[instance_id] then
        qr_code_instances[instance_id] = nil
        save_qr_instances()
        return true
    end
    return false
end

local function list_qr_instances()
    local instances = {}
    for id, instance in pairs(qr_code_instances) do
        instances[id] = {
            asset_id = instance.trigger_data,
            position_config = instance.position_config,
            is_visible = instance.is_visible,
            has_draw_details = instance.draw_details ~= nil
        }
    end
    return instances
end

local function get_qr_instance(asset_id)
    local instance_id = "qr_" .. tostring(asset_id)
    return qr_code_instances[instance_id]
end

local function save_qr_instances()
    local data_to_save = {}
    for id, instance in pairs(qr_code_instances) do
        data_to_save[id] = {
            id = instance.id,
            trigger_data = instance.trigger_data,
            position_config = instance.position_config,
            -- Don't save draw_details or is_visible as they're runtime state
        }
    end
    
    local success = pcall(function()
        local file = io.open("qr_instances.json", "w")
        if file then
            file:write(json.encode(data_to_save))
            file:close()
        end
    end)
    
end

local function load_qr_instances()
    local success, data = pcall(function()
        local file = io.open("qr_instances.json", "r")
        if file then
            local content = file:read("*all")
            file:close()
            return json.decode(content)
        end
        return nil
    end)
    
    if success and data then
        for id, instance_data in pairs(data) do
            qr_code_instances[id] = {
                id = instance_data.id,
                trigger_data = instance_data.trigger_data,
                position_config = instance_data.position_config,
                draw_details = nil,
                is_visible = false
            }
        end
        return true
    end
    return false
end

local min, max, abs, floor, ceil = math.min, math.max, math.abs, math.floor, math.ceil

local font_regl = resource.load_font "default-font.ttf"
local font_bold = resource.load_font "default-font-bold.ttf"
local font_7seg = resource.load_font "7segment.ttf"

local debug_marker = resource.create_colored_texture(1, 0, 0, 1)

local colored = resource.create_shader[[
    uniform sampler2D Texture;
    varying vec2 TexCoord;
    uniform vec4 color;
    void main() {
        gl_FragColor = texture2D(Texture, TexCoord) * color;
    }
]]

local white_pixel = resource.create_colored_texture(1,1,1,1)

local function log(system, format, ...)
    return print(string.format("[%s] " .. format, system, ...))
end

-- Now we can safely use the log function for dimensions
log("INIT", "Screen dimensions after log function defined")
log("INIT", "NATIVE_WIDTH x NATIVE_HEIGHT = %d x %d", NATIVE_WIDTH, NATIVE_HEIGHT)

local function permute(tab)
    for i = 1, #tab do
        local j = math.random(i, #tab)
        tab[i], tab[j] = tab[j], tab[i]
    end
end

local function json_nullify(val)
    if val == json.null then
        return
    else
        return val
    end
end

local function startswith(str, prefix)
    return str:sub(1, #prefix) == prefix
end

local function Music()
    local asset_name = nil
    local music = nil

    node.event("config_updated", function(config)
        log("music", "updating music config settings")
        if config.music.asset_id == "mute.mp4" then
            if music then
                music:dispose()
            end
            asset_name = nil
            music = nil
        elseif config.music.asset_name ~= asset_name then
            if music then
                music:dispose()
            end
            asset_name = config.music.asset_name
            music = resource.load_video{
                file = config.music.asset_name,
                looped = true,
                audio = true,
                raw = true,
            }
        end
    end)
end

local music = Music()

local function TCPClients()
    local clients = {}
    local handlers = {}

    node.event("connect", function(client, path)
        log("tcpclient", "new tcp client for %s", path)
        clients[client] = {
            path = path;
            send = function(...)
                node.client_write(client, ...)
            end
        }
    end)

    node.event("input", function(line, client)
        local cinfo = clients[client]
        if handlers[cinfo.path] then
            handlers[cinfo.path](line)
        end
    end)

    node.event("disconnect", function(client)
        clients[client] = nil
    end)

    local function send(path, ...)
        for client, cinfo in pairs(clients) do
            if cinfo.path == path then
                cinfo.send(...)
            end
        end
    end

    local function add_handler(path, handler)
        handlers[path] = handler
    end

    return {
        send = send;
        add_handler = add_handler;
    }
end

local tcp_clients = TCPClients()

local function Screen()
    local placer
    local rotation
    local frame_time = 1/60

    pcall(function()
        local fps, swap_interval = sys.get_ext("screen").get_display_info()
        frame_time = 1 / fps * swap_interval
        log("screen", "detected frame delay is %f", frame_time)
    end)

    node.event("config_updated", function(config)
        rotation = config.rotation
        local is_portrait = rotation == 90 or rotation == 270
        local width, height = config.resolution[1], config.resolution[2]
        log("screen", "configured content resolution is %dx%d", width, height)
        
        -- Debug: Print all screen-related dimensions using log function
        log("screen", "=== SCREEN CONFIG DEBUG ===")
        log("screen", "Configured Resolution: %dx%d", width, height)
        log("screen", "Rotation: %d degrees", rotation)
        log("screen", "Portrait Mode: %s", tostring(is_portrait))
        log("screen", "GL Native Dimensions: %dx%d", NATIVE_WIDTH, NATIVE_HEIGHT)
        log("screen", "========================")

        local surface = {
            width = width,
            height = height,
        }

        local target = {
            x = 0,
            y = 0,
            width = NATIVE_WIDTH,
            height = NATIVE_HEIGHT,
            rotation = rotation,
        }

        if is_portrait then
            surface.width, surface.height = surface.height, surface.width
        end

        placer = placement.Screen(target, surface)
    end)

    local function setup()
        return placer.setup()
    end

    local function place_video(...)
        return placer.place(...)
    end

    local function set_scissor(...)
        return placer.scissor(...)
    end

    local function get_rotation()
        return rotation
    end

    return {
        setup = setup;
        place_video = place_video;
        set_scissor = set_scissor;
        frame_time = frame_time;
        get_rotation = get_rotation;
    }
end

local screen = Screen()

local function FontCache()
    local fonts = {}

    local function get(filename)
        local font = fonts[filename]
        if not font then
            font = {
                obj = resource.load_font(filename),
            }
            fonts[filename] = font
        end
        font.lru = sys.now()
        return font.obj
    end

    local function tick()
        for filename, font in pairs(fonts) do
            if sys.now() - font.lru > 300 then
                log("fontcache", "purging font %s", filename)
                fonts[filename] = nil
            end
        end
    end

    return {
        get = get;
        tick = tick;
    }
end

local FontCache = FontCache()

local error_img = resource.create_colored_texture(1,0,0,1)
local function ImageCache()
    local images = {}

    local get, register

    function get(asset_name, keep)
        if not images[asset_name] then
            register(asset_name, keep)
        end
        local image = images[asset_name]
        if not image then
            return error_img
        end
        if not image.obj then
            image.obj = resource.load_image{
                file = image.file,
                fastload = true,
            }
        end
        image.lru = max(image.lru, sys.now() + keep)
        return image.obj
    end

    function register(asset_name, keep)
        log("imagecache", "register %s %d", asset_name, keep)
        if not images[asset_name] then
            images[asset_name] = {
                file = resource.open_file(asset_name),
                lru = sys.now() + keep
            }
        end
        return function(keep)
            return get(asset_name, keep or 0)
        end
    end

    local function tick()
        for asset_name, image in pairs(images) do
            local max_age = 0.5
            if not image.obj then
                max_age = 10
            end
            if sys.now() - image.lru > max_age then
              log("imagecache", "purging image %s", asset_name)
              image.file:dispose()
              if image.obj then
                  image.obj:dispose()
              end
              images[asset_name] = nil
            end
        end
    end

    return {
        register = register;
        get = get;
        tick = tick;
    }
end

local ImageCache = ImageCache()

local function Clock()
    local has_time = false
    local time = {diff=0}
    local updated = sys.now()

    local function since_midnight()
        local delta = sys.now() - updated
        local seconds = (time.since_midnight + delta) % 86400
        return seconds
    end

    return {
        update = function(new_time)
            time = new_time
            has_time = true
            updated = sys.now()
        end;
        has_time = function()
            return has_time
        end;
        human = function()
            local t = since_midnight()
            return string.format("%02d:%02d", math.floor(t / 3600), math.floor(t % 3600 / 60))
        end;
        unix = function()
            return os.time() + time.diff
        end;
        week_hour = function()
            return time.week_hour
        end;
        day_of_week = function()
            return time.dow
        end;
        since_midnight = since_midnight;
        today = function()
            return {
                day = time.day;
                month = time.month;
                year = time.year;
            }
        end;
    }
end

local function Clocks()
    local tz_clocks = {}

    local function create_and_get(tz)
        if not tz_clocks[tz] then
            tz_clocks[tz] = Clock()
        end
        return tz_clocks[tz]
    end

    util.data_mapper{
        ["clock/(.*)"] = function(tz, data)
            local time = json.decode(data)
            local clock = create_and_get(tz)
            clock.update(time)
        end;
    }

    return {
        get = create_and_get;
    }
end

local Clocks = Clocks()

local function Countdowns()
    local countdowns = {}

    util.data_mapper{
        ["countdown"] = function(update)
            local update = json.decode(update)
            countdowns[update.target] = update.unix
        end;
    }

    local function delta(target)
        local unix = countdowns[target]
        if not unix then
            return 0
        end
        return unix - os.time()
    end

    return {
        delta = delta;
    }
end

local Countdowns = Countdowns()
-- clock object pointing to the configured schedule timezone
local schedule_clock = setmetatable({
    tz = "UTC",
}, {
    __index = function(config, key)
        return Clocks.get(config.tz)[key]
    end,
})
node.event("config_updated", function(config)
    if config.timezone == "device" then
        schedule_clock.tz = config.__metadata.timezone
    else
        schedule_clock.tz = config.timezone
    end
end)

local function clock_for_tz_or_default(tz)
    if tz then
        return Clocks.get(tz)
    else
        return schedule_clock
    end
end

local SharedData = function()
    -- {
    --    scope: { key: data }
    -- }
    local data = {}

    -- {
    --    key: { scope: listener }
    -- }
    local listeners = {}

    local function call_listener(scope, listener, key, value)
        local ok, err = xpcall(listener, debug.traceback, scope, value)
        if not ok then
            log("shareddata", "while calling listener for key %s: %s", key, err)
        end
    end

    local function call_listeners(scope, key, value)
        local key_listeners = listeners[key]
        if not key_listeners then
            return
        end

        for _, listener in pairs(key_listeners) do
            call_listener(scope, listener, key, value)
        end
    end

    local function update(scope, key, value)
        if not data[scope] then
            data[scope] = {}
        end
        data[scope][key] = value
        if value == nil and not next(data[scope]) then
            data[scope] = nil
        end
        return call_listeners(scope, key, value)
    end

    local function delete(scope, key)
        return update(scope, key, nil)
    end

    local function add_listener(scope, key, listener)
        local key_listeners = listeners[key]
        if not key_listeners then
            listeners[key] = {}
            key_listeners = listeners[key]
        end
        if key_listeners[scope] then
            error "right now only a single listener is supported per scope"
        end
        key_listeners[scope] = listener
        for scope, scoped_data in pairs(data) do
            for key, value in pairs(scoped_data) do
                call_listener(scope, listener, key, value)
            end
        end
    end

    local function del_scope(scope)
        for key, key_listeners in pairs(listeners) do
            key_listeners[scope] = nil
            if not next(key_listeners) then
                listeners[key] = nil
            end
        end

        local scoped_data = data[scope]
        if scoped_data then
            for key, value in pairs(scoped_data) do
                delete(scope, key)
            end
        end
        data[scope] = nil
    end

    return {
        update = update;
        delete = delete;
        add_listener = add_listener;
        del_scope = del_scope;
    }
end

local data = SharedData()

local tile_loader = loader.setup "tile.lua"

tile_loader.before_load = function(tile, exports)
    exports.tcp_clients = tcp_clients
    exports.wait_frame = helper.wait_frame
    exports.wait_t = helper.wait_t
    exports.frame_between = helper.frame_between

    exports.screen = {
        place_video = screen.place_video;
        set_scissor = screen.set_scissor;
        get_rotation = screen.get_rotation;
    }

    exports.clock = schedule_clock;

    exports.update_data = function(key, value)
        data.delete(tile, key)
        data.update(tile, key, value)
    end
    exports.add_listener = function(key, listener)
        data.add_listener(tile, key, listener)
    end
end

tile_loader.unload = function(tile)
    data.del_scope(tile)
end

local function dispatch_to_all_tiles(event, ...)
    for module_name, module in pairs(tile_loader.modules) do
        local fn = module[event]
        if fn then
            local ok, err = xpcall(fn, debug.traceback, ...)
            if not ok then
                log(
                    "dispatch_to_all_tiles", 
                    "cannot dispatch '%s' into '%s': %s",
                    event, module_name, err
                )
            end
        end
    end
end

local kenburns_shader = resource.create_shader[[
    uniform sampler2D Texture;
    varying vec2 TexCoord;
    uniform vec4 Color;
    uniform float x, y, s;
    void main() {
        gl_FragColor = texture2D(Texture, TexCoord * vec2(s, s) + vec2(x, y)) * Color;
    }
]]

local gl_effects = {
    none = function(x1, y1, x2, y2)
        local w, h = x2-x1, y2-y1
        return function(draw, t)
            gl.pushMatrix()
                gl.translate(x1, y1)
                draw(t)
            gl.popMatrix()
        end, w, h
    end,
    rotation = function(x1, y1, x2, y2, config)
        local w, h = x2-x1, y2-y1
        return function(draw, t, starts, ends)
            local effect_easing = config.effect_easing or 'inQuad'
            local effect_rotation = config.effect_rotation or 'y-axis'
            local effect_pivot = config.effect_pivot or 'center'

            local pivot_x, pivot_y = unpack(({
                center = { .5, .5},
                top  =   { .5,  0},
                bottom = { .5,  1},
                left =   {  0, .5},
                right =  {  1, .5},
            })[effect_pivot])

            local enter_t = min(t-starts, 1)
            local exit_t = 1-max(0, 1-(ends-t))
            local effect_value = (
              -easing[effect_easing](1-enter_t, 0, 1, 1) 
              +easing[effect_easing](1-exit_t,  0, 1, 1) 
            )

            gl.pushMatrix()
                gl.translate(x1+w*pivot_x, y1+h*pivot_y)
                if effect_rotation == 'y-axis' then
                    gl.rotate(effect_value*90, 0, 1, 0)
                elseif effect_rotation == 'x-axis' then
                    gl.rotate(effect_value*90, 1, 0, 0)
                end
                gl.translate(-w*pivot_x, -h*pivot_y)
                draw(t)
            gl.popMatrix()
        end, w, h
    end,
    enter_exit_move = function(x1, y1, x2, y2, config)
        local w, h = x2-x1, y2-y1
        return function(draw, t, starts, ends)
            local effect_duration = config.effect_duration or 1
            local effect_easing = config.effect_easing or 'inQuad'
            local effect_direction = config.effect_direction or 'from_left'

            local progress = easing[effect_easing](
                1 - helper.ramp(starts, ends, t, effect_duration),
                0, 1, 1
            )
            local move_x, move_y = unpack(({
                from_left   = {-w,  0},
                from_right  = { w,  0},
                from_bottom = { 0,  h},
                from_top    = { 0, -h},
            })[effect_direction])
            gl.pushMatrix()
                gl.translate(x1+move_x*progress, y1+move_y*progress)
                draw(t)
            gl.popMatrix()
        end, w, h
    end,
}

local function ChildTile(asset, config, x1, y1, x2, y2)
    return function(starts, ends)
        local impl = tile_loader.modules[asset.asset_name]
        return impl.task(starts, ends, config, x1, y1, x2, y2)
    end
end

local function ScrollerTile(asset, config, x1, y1, x2, y2)
    return function(starts, ends)
        log("scroller", "ScrollerTile starting - asset: %s, coords: %d,%d,%d,%d", asset.asset_name, x1, y1, x2, y2)
        log("scroller", "ScrollerTile config: texts=%d, speed=%s", #(config.texts or {}), tostring(config.speed))
        
        local impl = tile_loader.modules["scroller"]
        if impl then
            log("scroller", "scroller module found, initializing...")
            -- Convert color arrays to color objects with .r, .g, .b, .a properties
            local function convert_color(color_array)
                if not color_array then return {r=1, g=1, b=1, a=1} end
                return {
                    r = color_array[1] or 1,
                    g = color_array[2] or 1,
                    b = color_array[3] or 1,
                    a = color_array[4] or 1
                }
            end
            
            -- Prepare scroller config in the format the scroller module expects (simple reception style)
            local scroller_config = {
                font = {
                    asset_name = asset.asset_name  -- Use the font asset from the tile
                },
                color = convert_color(config.color),  -- Convert main color
                speed = config.speed or 100,  -- Default speed
                texts = config.texts or {}  -- Simple text array
            }
            
            log("scroller", "scroller_config prepared with %d texts", #scroller_config.texts)
            
            -- Initialize the scroller with proper config
            if impl.updated_config_json then
                impl.updated_config_json(scroller_config)
                log("scroller", "scroller configured successfully")
            end
            
            return impl.task(starts, ends, scroller_config, x1, y1, x2, y2)
        else
            log("scroller", "ERROR: scroller module not found in tile_loader.modules")
            -- List available modules for debugging
            for name, _ in pairs(tile_loader.modules) do
                log("scroller", "Available module: %s", name)
            end
        end
    end
end

local function ImageTile(asset, config, x1, y1, x2, y2)
    -- config:
    --   kenburns: true/false
    --   fade_time: 0-1
    --   fit: true/false

    local img = ImageCache.register(asset.asset_name, 10)
    local fade_time = config.fade_time or 0

    return function(starts, ends)
        helper.wait_t(starts - 2)

        -- force loading and keep around for 3 seconds minimum
        img(3)

        local effect, width, height = gl_effects[
            config.effect or 'none'
        ](x1, y1, x2, y2, config)

        local function draw(now)
            if config.fit then
                util.draw_correct(img(), 0, 0, width, height, helper.ramp(
                    starts, ends, now, fade_time
                ))
            else
                img():draw(0, 0, width, height, helper.ramp(
                    starts, ends, now, fade_time
                ))
            end
        end

        if config.kenburns then
            local function lerp(s, e, t)
                return s + t * (e-s)
            end

            local paths = {
                {from = {x=0.0,  y=0.0,  s=1.0 }, to = {x=0.08, y=0.08, s=0.9 }},
                {from = {x=0.05, y=0.0,  s=0.93}, to = {x=0.03, y=0.03, s=0.97}},
                {from = {x=0.02, y=0.05, s=0.91}, to = {x=0.01, y=0.05, s=0.95}},
                {from = {x=0.07, y=0.05, s=0.91}, to = {x=0.04, y=0.03, s=0.95}},
            }

            local path = paths[math.random(1, #paths)]

            local to, from = path.to, path.from
            if math.random() >= 0.5 then
                to, from = from, to
            end

            local w, h = img():size()
            local duration = ends - starts

            local function lerp(s, e, t)
                return s + t * (e-s)
            end

            for now in helper.frame_between(starts, ends) do
                local t = (now - starts) / duration
                kenburns_shader:use{
                    x = lerp(from.x, to.x, t);
                    y = lerp(from.y, to.y, t);
                    s = lerp(from.s, to.s, t);
                }
                effect(draw, now, starts, ends)
                kenburns_shader:deactivate()
            end
        else
            for now in helper.frame_between(starts, ends) do
                effect(draw, now, starts, ends)
            end
        end
    end
end

local transparent_shader = resource.create_shader[[
    uniform sampler2D Texture;
    varying vec2 TexCoord;
    uniform vec3 Transparent;
    uniform vec4 Color;

    // These values seem to work reasonably well.
    const float thresholdSensitivity = 0.15;
    const float smoothing = 0.3;

    void main() {
        vec3 col = texture2D(Texture, TexCoord).rgb;
        float maskY = 0.2989 * Transparent.r + 0.5866 * Transparent.g + 0.1145 * Transparent.b;
        float maskCr = 0.7132 * (Transparent.r - maskY);
        float maskCb = 0.5647 * (Transparent.b - maskY);

        float Y = 0.2989 * col.r + 0.5866 * col.g + 0.1145 * col.b;
        float Cr = 0.7132 * (col.r - Y);
        float Cb = 0.5647 * (col.b - Y);

        float blendValue = smoothstep(thresholdSensitivity, thresholdSensitivity + smoothing, distance(vec2(Cr, Cb), vec2(maskCr, maskCb)));

        gl_FragColor = vec4(col * blendValue, 1.0 * blendValue);
    }
]]

local function VideoTile(asset, config, x1, y1, x2, y2)
    -- config:
    --   fade_time: 0-1
    --   looped
    --   transparency: bool
    --   transparent_color: #ffffff

    local file = resource.open_file(asset.asset_name)
    local fade_time = config.fade_time or 0.5
    local looped = config.looped
    local audio = config.audio

    local transparency = config.transparency
    local r, g, b = helper.parse_rgb(config.transparent_color or "#ffffff")
    local shader_args = {
        Transparent = {r, g, b}
    }

    return function(starts, ends)
        helper.wait_t(starts - 2)

        local vid = resource.load_video{
            file = file,
            paused = true,
            looped = looped,
            audio = audio,
        }

        for now in helper.frame_between(starts, ends) do
            if transparency then
                transparent_shader:use(shader_args)
            end
            vid:draw(x1, y1, x2, y2, helper.ramp(
                starts, ends, now, fade_time
            )):start()
            if transparency then
                transparent_shader:deactivate()
            end
        end

        vid:dispose()
    end
end

local function RawVideoTile(asset, config, x1, y1, x2, y2)
   

    local file = resource.open_file(asset.asset_name)
    local fade_time = config.fade_time or 0.5
    local looped = config.looped
    local audio = config.audio
    local layer = config.layer or 5

    return function(starts, ends)
        helper.wait_t(starts - 2)

        local vid = resource.load_video{
            file = file,
            paused = true,
            looped = looped,
            audio = audio,
            raw = false, -- CHANGED: Set raw to false to render video in OpenGL context
        }
        vid:layer(-10) -- This layer will now be within OpenGL, may need adjustment

        for now in helper.frame_between(starts, ends) do
        
            vid:draw(x1, y1, x2, y2, helper.ramp(starts, ends, now, fade_time))
            vid:start() -- Ensure video plays
        end

        vid:dispose()
    end
end

local function Streams()
    local frame = 0
    local streams = {}

    local MIN_LOAD_INTERVAL = 5
    local LOADING_TIMEOUT = 10

    local function stream_key(url, audio)
        return string.format("%s|%s", url, audio)
    end

    local function get_stream(url, audio, no_buffer)
        local key = stream_key(url, audio)
        if not streams[key] then
            -- Determine if this is a UDP stream
            local is_udp_stream = string.match(url, "^udp://")
            
            -- Initialize the stream only once
            local stream_config = {
                file = url,
                audio = audio,
            }
            
            -- Add UDP-specific settings for ultra-low latency
            if is_udp_stream or no_buffer then
                -- Use non-raw mode for UDP streams to allow more control
                stream_config.raw = false            -- Use OpenGL rendering for better control
                stream_config.paused = false         -- Start immediately  
                stream_config.looped = false         -- Live streams don't loop
                
                -- Core low-latency settings that info-beamer definitely supports
                stream_config.buffering = false      -- Disable buffering
                stream_config.stream = true          -- Enable live stream mode
                
                -- Additional low-latency hints
                stream_config.buffer_time = 0.01     -- Minimal buffer time (10ms)
                stream_config.framedrop = true       -- Enable frame dropping
                stream_config.faststart = true       -- Fast stream startup
                
                log("stream", "Creating UDP/no-buffer ULTRA-LOW LATENCY stream: %s", url)
            else
                stream_config.raw = true             -- Use raw mode for regular streams
                stream_config.looped = true          -- Regular streams can loop
                log("stream", "Creating regular stream: %s", url)
            end
            
            streams[key] = {
                vid = resource.load_video(stream_config),
                last_used = frame,
                url = url,  -- Store URL for debugging
                is_udp = is_udp_stream,
            }
            
            -- For UDP streams, start immediately with aggressive settings
            if is_udp_stream then
                -- Start UDP stream immediately at full visibility to minimize startup delay
                streams[key].vid:start()
                log("stream", "UDP stream started immediately: %s", url)
            else
                -- Keep regular streams running but hidden
                streams[key].vid:layer(-10):place(0, 0, 0, 0):alpha(0):start()
            end
        end

        streams[key].last_used = frame
        return streams[key].vid
    end

    -- Add function to immediately dispose all streams
    local function dispose_all_streams()
        for key, stream in pairs(streams) do
            log("stream", "Force disposing stream: %s", stream.url)
            if stream.vid then
                stream.vid:dispose()
            end
            streams[key] = nil
        end
    end

    -- Add function to dispose streams by URL pattern
    local function dispose_streams_matching(pattern)
        for key, stream in pairs(streams) do
            if string.match(stream.url, pattern) then
                log("stream", "Force disposing matching stream: %s", stream.url)
                if stream.vid then
                    stream.vid:dispose()
                end
                streams[key] = nil
            end
        end
    end

    local function tick()
        frame = frame + 1
        if frame % 300 == 0 then
            print "[stream] active streams"
            for key, stream in pairs(streams) do
                print(string.format("[stream] %s (UDP: %s)", stream.url, tostring(stream.is_udp)))
            end
        end

        for key, stream in pairs(streams) do
            local frame_delta = frame - stream.last_used
            -- Ultra-aggressive UDP keep-alive time for lowest latency
            local max_idle = stream.is_udp and 60 or 300   -- 1 second for UDP, 5 seconds for others
            if frame_delta > max_idle then
                print("[stream] disposing stream", stream.url)
                if stream.vid then
                    stream.vid:dispose()
                end
                streams[key] = nil
            end
        end
    end

    return {
        get_stream = get_stream;
        tick = tick;
        dispose_all_streams = dispose_all_streams;
        dispose_streams_matching = dispose_streams_matching;
        _get_all_streams = function() return streams end;
        _remove_stream = function(key) streams[key] = nil end;
    }
end
local streams = Streams()

local function StreamTile(asset, config, x1, y1, x2, y2)
    local layer = config.layer or 5
    local url = config.url or ""
    local audio = config.audio
    local no_buffer = config.no_buffer or false  -- New option for UDP streams

    return function(starts, ends)
        -- For UDP streams, minimize delays and optimize for low latency
        local is_udp = string.match(url, "^udp://") or no_buffer
        local wait_time = is_udp and 0.01 or 0.1  -- Minimal wait for UDP streams
        
        helper.wait_t(starts - wait_time)
        
        -- For UDP streams, be more aggressive about disposing conflicting streams
        if is_udp then
            -- Immediately dispose ALL other streams for UDP to avoid conflicts
            for key, stream in pairs(streams._get_all_streams()) do
                if stream.url ~= url then
                    log("stream", "PRIORITY DISPOSAL for UDP - disposing %s to start %s", stream.url, url)
                    if stream.vid then
                        stream.vid:dispose()
                    end
                    streams._remove_stream(key)
                end
            end
        else
            -- Normal disposal logic for non-UDP streams
            for key, stream in pairs(streams._get_all_streams()) do
                if stream.url ~= url then
                    log("stream", "Disposing old stream %s to start new stream %s", stream.url, url)
                    if stream.vid then
                        stream.vid:dispose()
                    end
                    streams._remove_stream(key)
                end
            end
        end
        
        -- player  
        for now in helper.frame_between(starts, ends) do
            local vid = streams.get_stream(url, audio, no_buffer)
            if vid then
                if is_udp then
                    -- For UDP streams using raw=false, draw directly for minimal latency
                    vid:draw(x1, y1, x2, y2, 1.0):start()
                else
                    -- Use screen.place_video for raw streams
                    screen.place_video(vid, layer, 1, x1, y1, x2, y2)
                end
            end
        end
    end
end

local function FlatTile(asset, config, x1, y1, x2, y2)
    -- config:
    --   color: "#rrggbb"
    --   fade_time: 0-1

    local r, g, b = helper.parse_rgb(config.color or "#ffffff")
    local a = (config.alpha or 255)/255

    local flat = resource.create_colored_texture(r, g, b, a)
    local fade_time = config.fade_time or 0.5

    return function(starts, ends)
        for now in helper.frame_between(starts, ends) do
            flat:draw(x1, y1, x2, y2, helper.ramp(
                starts, ends, now, fade_time
            ))
        end
        flat:dispose()
    end
end

local function CountdownTile(asset, config, x1, y1, x2, y2)
    local target = config.target
    local type = config.type or "hms"
    local r, g, b = helper.parse_rgb(config.color or "#333333")
    local align = config.align or "center"
    local mode = config.mode or "countdown"
    local size = y2 - y1
    local font = ({
        default = font_regl,
        digital = font_7seg,
    })[config.font or "default"]
    local fmt = ({
        german = {
            dh = "%d Tag(e), %d Std.",
            hms = "%d Std %d Min %d Sek",
            ms = "%d Min %d Sek",
            hm = "%d Std %d Min",
        },
        english = {
            dh = "%d day(s), %dh",
            hms = "%dh %dm %ds",
            ms = "%dm %ds",
            hm = "%dh %dm",
        },
        none = {
            hms = "%d:%02d:%02d",
            ms = "%d:%02d",
            hm = "%d:%02d",
        },
    })[config.locale or "english"]

    return function(starts, ends)
        for now in helper.frame_between(starts, ends) do
            local delta = Countdowns.delta(target)

            delta = ceil(delta)

            if mode == "countdown" then
                delta = max(0, delta)
            elseif mode == "countup" then
                delta = min(0, delta)
            end

            delta = abs(delta)

            local text
            if type == "hms" then
                text = string.format(fmt.hms,
                    math.floor(delta / 3600),
                    math.floor(delta % 3600 / 60),
                    math.floor(delta % 60)
                )
            elseif type == "hm" then
                text = string.format(fmt.hm,
                    math.floor(delta / 3600),
                    math.floor(delta % 3600 / 60)
                )
            elseif type == "adaptive_dhm" then
                if abs(delta) > 86400 then
                    text = string.format(fmt.dh,
                        math.floor(delta / 86400),
                        math.floor(delta % 86400 / 3600)
                    )
                elseif abs(delta) > 120 * 60 then
                    text = string.format(fmt.hms,
                        math.floor(delta / 3600),
                        math.floor(delta % 3600 / 60),
                        math.floor(delta % 60)
                    )
                else
                    text = string.format(fmt.ms,
                        math.floor(delta / 60),
                        math.floor(delta % 60)
                    )
                end
            elseif type == "adaptive_hms" then
                if abs(delta) > 120 * 60 then
                    text = string.format(fmt.hms,
                        math.floor(delta / 3600),
                        math.floor(delta % 3600 / 60),
                        math.floor(delta % 60)
                    )
                else
                    text = string.format(fmt.ms,
                        math.floor(delta / 60),
                        math.floor(delta % 60)
                    )
                end
            end

            local w = font:width(text, size)

            local x
            if align == "left" then
                x = x1
            elseif align == "right" then
                x = x2 - w
            elseif align == "center" then
                x = x1 + (x2-x1)/2 - w/2
            end

            font:write(x, y1, text, size, r,g,b,1)
        end
    end
end

local function TimeTile(asset, config, x1, y1, x2, y2)
    local r, g, b = helper.parse_rgb(config.color or "#333333")
    local clock = clock_for_tz_or_default(json_nullify(config.timezone))

    local clock_mode = config.mode or "digital_clock"
    local clock_type = config.type or "hms"
    local clock_style = config.style or 1
    local clock_align = config.align or "center"
    local clock_movement = config.movement or "dynamic"

    if clock_mode == "digital_clock" then
        local size = y2 - y1

        local font
        if clock_style == 1 then
            font = font_7seg
        else
            font = font_regl
        end

        local formatter
        if clock_type == "hm" then
            formatter = function(t)
                return string.format("%02d:%02d",
                    math.floor(t / 3600),
                    math.floor(t % 3600 / 60)
                ), '99:99', ''
            end
        elseif clock_type == "hms" then
            formatter = function(t)
                return string.format("%02d:%02d:%02d",
                    math.floor(t / 3600),
                    math.floor(t % 3600 / 60),
                    math.floor(t % 60)
                ), '99:99:99', ''
            end
        elseif clock_type == "hm_12" then
            formatter = function(t)
                local hours = math.floor(t / 3600)
                local minutes = math.floor(t % 3600 / 60)
                local ampm = "am"
                if hours >= 12 then
                    ampm = "pm"
                    hours = hours - 12
                end
                if hours == 0 then
                    hours = 12
                end
                return string.format("%02d:%02d",
                    hours, minutes
                ), '99:99', ' '..ampm
            end
        elseif clock_type == "hms_12" then
            formatter = function(t)
                local hours = math.floor(t / 3600)
                local minutes = math.floor(t % 3600 / 60)
                local seconds = math.floor(t % 60)
                local ampm = "am"
                if hours >= 12 then
                    ampm = "pm"
                    hours = hours - 12
                end
                if hours == 0 then
                    hours = 12
                end
                return string.format("%02d:%02d:%02d",
                    hours, minutes, seconds
                ), '99:99:99', ' '..ampm
            end
        end

        return function(starts, ends)
            for now in helper.frame_between(starts, ends) do
                local time, tmpl, suffix = formatter(clock.since_midnight())

                local w_time = font:width(tmpl, size)
                local w = w_time + font:width(suffix, size)

                local x
                if clock_align == "left" then
                    x = x1
                elseif clock_align == "right" then
                    x = x2 - w
                elseif clock_align == "center" then
                    x = x1 + (x2-x1)/2 - w/2
                end

                font:write(x, y1, time, size, r,g,b,1)
                x = x + w_time
                font:write(x, y1, suffix, size, r,g,b,1)
            end
        end
    elseif clock_mode == "analog_clock" then
        local cx = x1 + (x2 - x1)/2
        local cy = y1 + (y2 - y1)/2
        local size = math.min(y2-y1, x2-x1)
        local radius = size/2
        local unit = size/30

        local hand_img
        
        if clock_style == 1 then 
            hand_img = resource.load_image{
                file = "hand-1.png",
                mipmap = true,
            }
        else
            hand_img = resource.load_image{
                file = "hand-2.png",
                mipmap = true,
            }
        end

        local show_seconds = clock_type == "hms" or clock_type == "hms_12"

        -- local function dots()
        --     gl.pushMatrix()
        --         gl.rotate(90, 0, 0, 1)
        --         for i = 0, 55,5 do
        --             if i % 15 == 0 then
        --                 pixel:draw(radius-unit, -unit/2, radius, unit/2, 0.8)
        --             else 
        --                 pixel:draw(radius-unit/2, -unit/4, radius, unit/4, 0.8)
        --             end
        --             gl.rotate(360/60*5, 0, 0, 1)
        --         end
        --     gl.popMatrix()
        -- end

        local function hand(len, thick, angle)
            gl.pushMatrix()
                gl.rotate(angle*360-90, 0, 0, 1)
                hand_img:draw(-len*0.1, -thick/2, len*0.9, thick/2)
            gl.popMatrix()
        end

        local function movement(val)
            if clock_movement == "dynamic" then
                return math.floor(val) + math.sin(((val%1)-0.5) * math.pi)/2 + .5
            elseif clock_movement == "smooth" then
                return val
            else
                return math.floor(val)
            end
        end

        return function(starts, ends)
            for now in helper.frame_between(starts, ends) do
                local t = clock.since_midnight()

                colored:use{
                    color = {r,g,b,1}
                }
                gl.pushMatrix()
                    gl.translate(cx, cy)

                    local hour = movement((t / 3600) % 12)
                    hand(radius-7*unit, unit, hour/12)

                    local minute = movement(t % 3600 / 60)
                    hand(radius-3*unit, unit*0.75, minute/60)

                    if show_seconds then
                        local second = movement(t % 60)
                        hand(radius, unit*0.5, second/60)
                    end
                gl.popMatrix()
                colored:deactivate()
            end
            hand_img:dispose()
        end
    end
end

local function MarkupTile(asset, config, x1, y1, x2, y2)
    local fade_time = config.fade_time or 0
    local text = config.text or ""
    local font_size = config.font_size or 35
    local align = config.align or "tl"
    local font = FontCache.get(asset.asset_name)

    local width = x2 - x1
    local height = y2 - y1
    local r, g, b = helper.parse_rgb(config.color or "#ffffff")

    local y = 0
    local max_x = 0
    local writes = {}

    local cell_padding = 40
    local paragraph_split = 40
    local line_height = 1.05

    local default_font_size = font_size
    local h1_font_size = default_font_size * 2
    local h2_font_size = floor(h1_font_size * 0.75)

    local function max_per_line(font, size, width)
        -- try to calculate the max characters/line
        -- number based on the average character width
        -- of the specified font.
        local test_width = font:width("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz", size)
        local avg_width = test_width / 52
        local chars_per_line = width / avg_width
        return floor(chars_per_line)
    end

    local rows = {}
    local function flush_table()
        local max_w = {}
        for ri = 1, #rows do
            local row = rows[ri]
            for ci = 1, #row do
                local col = row[ci]
                max_w[ci] = max(max_w[ci] or 0, col.width)
            end
        end

        local TABLE_SEPARATE = 40

        for ri = 1, #rows do
            local row = rows[ri]
            local x = 0
            for ci = 1, #row do
                local col = row[ci]
                if col.text ~= "" then
                    col.x = floor(x)
                    col.y = floor(y)
                    writes[#writes+1] = col
                end
                x = x + max_w[ci]+cell_padding
            end
            y = y + default_font_size * line_height
            max_x = max(max_x, x-cell_padding)
        end
        rows = {}
    end

    local function add_row()
        local cols = {}
        rows[#rows+1] = cols
        return cols
    end

    local function layout_paragraph(paragraph)
        for line in string.gmatch(paragraph, "[^\n]+") do
            local size = default_font_size -- font size for line
            local maxl = max_per_line(font, size, width)

            if line:find "|" then
                -- table row
                local cols = add_row()
                for field in line:gmatch("[^|]+") do
                    field = helper.trim(field)
                    local width = font:width(field, size)
                    cols[#cols+1] = {
                        font = font,
                        text = field,
                        size = size,
                        width = width,
                    }
                end
            else
                -- plain text, wrapped
                flush_table()

                -- markdown header # and ##
                if line:sub(1,2) == "##" then
                    line = line:sub(3)
                    font = font
                    size = h2_font_size
                    maxl = max_per_line(font, size, width)
                elseif line:sub(1,1) == "#" then
                    line = line:sub(2)
                    font = font
                    size = h1_font_size
                    maxl = max_per_line(font, size, width)
                end

                local chunks = helper.wrap(line, maxl)
                for idx = 1, #chunks do
                    local chunk = chunks[idx]
                    chunk = helper.trim(chunk)
                    writes[#writes+1] = {
                        font = font,
                        x = 0,
                        y = floor(y),
                        text = chunk,
                        size = size,
                    }
                    local width = font:width(chunk, size)
                    y = y + size * line_height
                    max_x = max(max_x, width)
                end
            end
        end

        flush_table()
    end

    local paragraphs = helper.split(text, "\n\n")
    for idx = 1, #paragraphs do
        local paragraph = paragraphs[idx]
        paragraph = paragraph:gsub("\t", " ")
        layout_paragraph(paragraph)
        y = y + paragraph_split
    end

    -- remove one split
    local max_y = y - paragraph_split
    local base_x, base_y

    if align == "tl" then
        base_x = 0
        base_y = 0
    elseif align == "center" then
        base_x = floor((width-max_x) / 2)
        base_y = floor((height-max_y) / 2)
    end

    return function(starts, ends)
        for now in helper.frame_between(starts, ends) do
            local x = x1 + base_x
            local y = y1 + base_y
            for idx = 1, #writes do
                local w = writes[idx]
                w.font:write(x+w.x, y+w.y, w.text, w.size, r,g,b,helper.ramp(
                    starts, ends, now, fade_time
                ))
            end
        end
    end
end

local function JobQueue()
    local jobs = {}

    local function add(fn, starts, ends)
        local co = coroutine.create(fn)
        local ok, again = coroutine.resume(co, starts, ends)
        if not ok then
            log(
                "jobqueue",
                "cannot create task:\n%s\n%s\ninside coroutine started by",
                again, debug.traceback(co)
            )
        elseif not again then
            return
        end

        local job = {
            starts = starts,
            ends = ends,
            co = co,
        }

        jobs[#jobs+1] = job
    end

    local function tick(now)
        for idx, job in ipairs(jobs) do
            local ok, again = coroutine.resume(job.co, now)
            if not ok then
                log(
                    "jobqueue",
                    "cannot run task:\n%s\n%s\ninside coroutine %s resumed by",
                    again, debug.traceback(job.co), job
                )
                job.done = true
            elseif not again then
                job.done = true
            end
        end

        -- iterate backwards so we can remove finished jobs
        for idx = #jobs,1,-1 do
            local job = jobs[idx]
            if job.done then
                table.remove(jobs, idx)
            end
        end
    end

    local function flush()
        log("jobqueue", "Flushing job queue and disposing all streams")
        jobs = {}
        -- Immediately dispose all streams when flushing jobs to prevent delays
        streams.dispose_all_streams()
        node.gc()
    end

    return {
        tick = tick;
        add = add;
        flush = flush;
    }
end

local layouts = {}
local background = {r = 0, g = 0, b = 0, a = 0}
local current_setup_id = "UNKNOWN_SETUP" 

node.event("config_updated", function(config)
    layouts = config.layouts
    for _, layout in ipairs(layouts) do
        for _, tile in ipairs(layout.tiles) do
            local asset = tile.asset
            if asset.type == "image" or asset.type == "video" then
                log("config_updated", "fixing layout asset %s", asset.asset_name)
                asset.asset_name = resource.open_file(asset.asset_name)
            end
        end
    end
    background = config.background
    
    -- Store the setup_id when config is updated
    if config.__metadata and config.__metadata.setup_id then
        current_setup_id = config.__metadata.setup_id
        log("config_updated", "Stored setup_id: %s", current_setup_id)
    else
        log("config_updated", "setup_id not found in config metadata")
    end
end)

local function Page(page)
    local function get_duration(playback_mode)
        local duration = page.duration
        if duration == 0 then
            for _, tile in ipairs(page.tiles) do
                if tile.asset.metadata and tile.asset.metadata.duration then
                    duration = max(duration, tile.asset.metadata.duration)
                elseif tile.type == "child" then
                    local child_name = tile.asset.asset_name
                    local impl = tile_loader.modules[child_name]
                    if impl.auto_duration then
                        duration = max(duration, impl.auto_duration())
                    end
                end
            end
            if duration == 0 then
                duration = 10
            else
                duration = math.max(2, duration)
            end
            print("automatically set auto duration is", duration)
        end
        if playback_mode == "interactive" and page.interaction.duration == "forever" then
            duration = "forever"
        end
        return duration
    end

    local function can_show()
        local can_show = true
        for _, tile in ipairs(page.tiles) do
            if tile.type == "child" then
                local child_name = tile.asset.asset_name
                local impl = tile_loader.modules[child_name]
                if impl.can_show and not impl.can_show(tile.config) then
                    print("probing child tile", child_name, "can_show==false")
                    can_show = false
                end
            end
        end
        return can_show
    end

    local function get_tiles()
        local tiles = {}

        local function append_page_tiles()
            for _, tile in ipairs(page.tiles) do
                tiles[#tiles+1] = tile
            end
        end

        local layout = layouts[page.layout_id+1]
        if not layout then
            append_page_tiles()
        else
            for _, tile in ipairs(layout.tiles) do
                if tile.type == "page" then
                    append_page_tiles()
                else
                    tiles[#tiles+1] = tile
                end
            end
        end
        return tiles
    end

    return {
        get_duration = get_duration;
        get_tiles = get_tiles;
        is_fallback = page.is_fallback;
        can_show = can_show;
    }
end

local function Scheduler(page_source, job_queue)
    local SCHEDULE_LOOKAHEAD = 2

    local scheduled_until = sys.now()

    local showing_fallback = false
    local scheduled_forever = false

    local function enqueue_page(page, playback_mode)
        playback_mode = playback_mode or "default"

        local duration = page.get_duration(playback_mode)

        local starts = scheduled_until
        local ends

        if duration == "forever" then
            -- more than 1000 days, so close enough
            ends = starts + 100000000

            -- If a 'forever' page is scheduled, note that this
            -- is the case, so the config watcher can use this
            -- in its decision on whether or not to reset the
            -- scheduled jobs.
            scheduled_forever = true
        else
            ends = starts + duration
        end

        local next_layer = {
            back = -10,
            front = 1,
        }

        local tiles = page.get_tiles()
        for n, tile in ipairs(tiles) do
            local handler = ({
                image = ImageTile,
                video = VideoTile,
                rawvideo = RawVideoTile,
                stream = StreamTile,
                child = ChildTile,
                scroller = ScrollerTile,
                flat = FlatTile,
                time = TimeTile,
                countdown = CountdownTile,
                markup = MarkupTile,
            })[tile.type]

            -- Reorder layering, so that layers back and front
            -- layers are sorted by their tile order.
            if tile.type == 'rawvideo' or tile.type == 'stream' then
                local old_layer = tile.config.layer or 5
                local next_layer_select = old_layer < 0 and "back" or "front"
                tile.config.layer = next_layer[next_layer_select]
                next_layer[next_layer_select] = next_layer[next_layer_select] + 1
            end

            -- print "adding tile"
            job_queue.add(
                handler(tile.asset, tile.config, tile.x1, tile.y1, tile.x2, tile.y2),
                starts, ends
            )
        end

        job_queue.add(
            function(starts, ends)
                helper.wait_t(starts)
                showing_fallback = page.is_fallback
                tcp_clients.send(
                    "root/__fallback__",
                    page.is_fallback and "1" or "0"
                )
            end,
            starts, ends
        )

        scheduled_until = ends
    end

    local function tick(now)
        if now < scheduled_until - SCHEDULE_LOOKAHEAD then
            return
        end

        enqueue_page(page_source.get_next())
    end

    local function reset_scheduler()
        log("scheduler", "Resetting scheduler and disposing streams for fast transition")
        -- Dispose all streams immediately for faster asset transitions
        streams.dispose_all_streams()
        job_queue.flush()
        scheduled_forever = false
        scheduled_until = sys.now()
    end

    local function enqueue_interactive(pages)
        reset_scheduler()
        for i, page in ipairs(pages) do
            enqueue_page(page, "interactive")
        end
    end

    local function handle_keyboard(event)
        -- if event.action ~= "down" and event.action ~= "hold" then
        if event.action ~= "down" then
            return
        end

        if event.key == "esc" then
            reset_scheduler()
            -- Hide all QR codes on ESC
            for id, instance in pairs(qr_code_instances) do
                instance.is_visible = false
                print("Hiding QR instance " .. id .. " due to ESC")
            end
            return
        end

        -- Hide QR codes on arrow key navigation
        if event.key == "left" or event.key == "right" then
            for id, instance in pairs(qr_code_instances) do
                instance.is_visible = false
                print("Hiding QR instance " .. id .. " due to keyboard navigation")
            end
        end

        if event.key == "left" then
            reset_scheduler()
            enqueue_page(page_source.get_prev())
            return
        end

        if event.key == "right" then
            reset_scheduler()
            enqueue_page(page_source.get_next())
            return
        end

        local pages = page_source.find_by_key(event.key)
        if not pages then
            return
        end
        enqueue_interactive(pages)
    end

    local function handle_gamepad(event)
        local pages = page_source.find_by_key(event.key)
        if not pages then
            return
        end
        enqueue_interactive(pages)
    end

    local function handle_gpio(event)
        local pages = page_source.find_by_gpio(event.pin)
        if not pages then
            return
        end
        enqueue_interactive(pages)
    end

    local function handle_remote_trigger(trigger_data)
        print("Remote trigger received:", trigger_data)
        
        -- Debug: Show all existing QR instances
        for id, instance in pairs(qr_code_instances) do
        end

        local qr_instance_updated = false

        -- Check if this trigger_data corresponds to any predefined QR instance's trigger_data
        for id, instance in pairs(qr_code_instances) do
            if instance.trigger_data == trigger_data then
                print("Trigger " .. trigger_data .. " matches predefined QR instance: " .. id)
                local new_draw_details = qrcode_overlay.handle_remote_trigger(instance.trigger_data, current_setup_id)
                if new_draw_details then
                    instance.draw_details = new_draw_details
                    instance.is_visible = true
                    qr_instance_updated = true
                    print("Activated and updated QR instance: " .. id)
                else
                    print("ERROR: Failed to get draw details for QR instance: " .. id .. " (trigger: " .. trigger_data .. ")")
                    instance.is_visible = false -- Hide if generation fails
                    instance.draw_details = nil
                end
                -- If you want a trigger to only activate ONE specific QR code and hide others,
                -- you might add logic here to set other_instance.is_visible = false
                -- For now, a trigger activates its corresponding QR without affecting others that might be visible.
            end
        end

        -- If no QR instance was specifically updated by this trigger, 
        -- AND this trigger is not one of the known QR generation triggers for predefined instances,
        -- then hide all QR codes. This prevents unrelated triggers from leaving QR codes on screen.
        local is_known_qr_gen_trigger = false
        for _, instance in pairs(qr_code_instances) do
            if instance.trigger_data == trigger_data then
                is_known_qr_gen_trigger = true
                break
            end
        end

        if not qr_instance_updated and not is_known_qr_gen_trigger then
            print("Trigger '" .. trigger_data .. "' is not a specific QR activation trigger. Hiding all QR instances.")
            for id, instance in pairs(qr_code_instances) do
                instance.is_visible = false
            end
        end

        -- Process normal page navigation using the scheduler
        local pages = page_source.find_by_remote(trigger_data)
        if not pages then
            -- If not a page navigation trigger either, and we didn't update a QR, it might be an unhandled trigger.
            -- The 'hiding all' logic above should cover making sure QRs don't persist incorrectly.
            return false -- Indicate trigger was not fully handled for page navigation
        end
        enqueue_interactive(pages)
        return true
    end

    local function handle_cec(cec_key)
        -- Hide QR codes on CEC navigation
        if cec_key == "left" or cec_key == "right" then
            for id, instance in pairs(qr_code_instances) do
                instance.is_visible = false
                print("Hiding QR instance " .. id .. " due to CEC navigation")
            end
        end

        if cec_key == "left" then
            reset_scheduler()
            enqueue_page(page_source.get_prev())
            return
        end

        if cec_key == "right" then
            reset_scheduler()
            enqueue_page(page_source.get_next())
            return
        end
    end

    -- Function to clear all overlay content (QR codes and coke overlay)
    local function clear_all_overlays()
        -- Hide all QR code instances
        for id, instance in pairs(qr_code_instances) do
            instance.is_visible = false
            print("Clearing QR instance " .. id .. " due to setup switch")
        end
        
    end

    local last_setup_id, last_config_hash

    node.event("config_updated", function(config)
        local setup_id = config.__metadata.setup_id
        local config_hash = config.__metadata.config_hash
        local reset_mode = config.reset_mode

        local force_reset

        if reset_mode == "config" then
            force_reset = config_hash ~= last_config_hash
        elseif reset_mode == "setup" then
            force_reset = setup_id ~= last_setup_id
        elseif reset_mode == "in_forever" then
            force_reset = scheduled_forever and config_hash ~= last_config_hash
        else -- "none"
            force_reset = false
        end

        if force_reset then
            print("config updated: forcing scheduler reset")
            reset_scheduler()
        end

        last_setup_id = setup_id
        last_config_hash = config_hash
    end)

    return {
        tick = tick;
        handle_keyboard = handle_keyboard;
        handle_gamepad = handle_gamepad;
        handle_gpio = handle_gpio;
        handle_remote_trigger = handle_remote_trigger;
        handle_cec = handle_cec;
    }
end

local function PageSource()
    local schedules = {}

    local cycle_pages = {}
    local cycle_offset = 0

    local find_offset = 0

    local fallback
    local debug_schedule_id, debug_page_id
    local trigger = "next"

    node.event("config_updated", function(config)
        schedules = config.schedules

        for _, schedule in ipairs(schedules) do
            for _, page in ipairs(schedule.pages) do
                page.is_fallback = false
                if page.duration == -1 then
                    -- disabled page? then remove it
                    table.remove(schedule.pages, page_id)
                else
                    for _, tile in ipairs(page.tiles) do
                        local asset = tile.asset
                        if asset.type == "image" or asset.type == "video" then
                            log("config_updated", "fixing schedule asset %s", asset.asset_name)
                            asset.asset_name = resource.open_file(asset.asset_name)
                        end
                    end
                end
            end
        end

        debug_schedule_id = config.scratch.debug_schedule_id
        debug_page_id = config.scratch.debug_page_id
        fallback = config.fallback
        trigger = config.trigger
    end)

    local function date_within(starts, ends, test)
        local function expand(date)
            return date.year * 600 + date.month * 40 + date.day
        end
        local t = expand(test)
        local s = 0
        if starts then
            s = expand(starts)
        end
        local e = 100000000
        if ends then
            e = expand(ends)
        end
        return t >= s and t <= e
    end

    local function parse_date(d)
        if not d then
            return nil
        end
        local year, month, day = d:match "(%d+)-(%d+)-(%d+)"
        if not year then
            return nil
        end
        return {year=tonumber(year), month=tonumber(month), day=tonumber(day)}
    end

    local function parse_hour(h)
        local hour, minute = h:match "(%d+):(%d+)"
        return {hour=tonumber(hour), minute=tonumber(minute)}
    end

    local function minutes_since_midnight(t)
        return t.hour * 60 + t.minute
    end

    local function is_scheduled(schedule)
        local scheduling = schedule.scheduling
        local mode = scheduling.mode or "span"
        local starts = parse_date(scheduling.starts)
        local ends = parse_date(scheduling.ends)

        if mode == "fallback" then
            -- Do not schedule here. Later if there's nothing scheduled at all, we
            -- fetch the 'fallback' schedules in get_fallback_cycle()
            return false
        elseif mode == "always" then
            return true
        elseif mode == "never" then
            return false
        end

        if not schedule_clock.has_time() then
            if starts or ends then
                log("schedule", "no current time. can't schedule playlist with start/end date")
                return false
            end
            if mode == "hour" then
                local hours = scheduling.hours or {}
                for hour = 1, 24*7+1 do
                    -- see if all hours are unchecked. If any is checked,
                    -- we can't schedule as we need to know the current
                    -- hour.
                    if hours[hour+1] == false then
                        log("schedule", "no current time. can't schedule hour based playlist")
                        return false
                    end
                end
            elseif mode == "span" then
                local spans = scheduling.spans or {}
                -- If a timestamp is set, we can't schedule.
                if #spans > 0 then
                    log("schedule", "no current time. can't schedule span scheduled playlist")
                    return false
                end
            elseif mode == "interval" then
                log("schedule", "no current time. can't schedule playlist with time interval")
                return false
            end
            log("schedule", "scheduling although we don't have a correct time as schedule is always active")
            return true
        end

        local today = schedule_clock.today()

        if not date_within(starts, ends, today) then
            log("schedule", "outside of scheduled dates. can't schedule")
            return false
        end

        if mode == "hour" then
            local current_hour = schedule_clock.week_hour()
            local hours = scheduling.hours or {}
            if hours[current_hour+1] == false then
                -- only refuse to schedule if it's actually set to 'false'.
                -- nil means that the hours list is empty which implicitly
                -- means: "always show".
                log("schedule", "not within the current week hour. can't schedule")
                return false
            else
                return true
            end
        elseif mode == "span" then
            local spans = scheduling.spans or {}

            if #spans == 0 then
                log("schedule", "no spans. always schedule")
                return true
            end

            local since_midnight = schedule_clock.since_midnight()
            for span_id, span in ipairs(spans) do
                local dow = schedule_clock.day_of_week()
                if span.days[dow+1] then
                    local start_sec = minutes_since_midnight(parse_hour(span.starts)) * 60
                    local end_sec = minutes_since_midnight(parse_hour(span.ends)) * 60 + 60
                    if since_midnight >= start_sec and since_midnight < end_sec then
                        log("schedule", "span %s matches", span_id)
                        return true
                    end
                end
            end
            return false
        elseif mode == "interval" then
            local interval = scheduling.interval or {}
            local interval_starts = interval.starts or "00:00"
            local interval_ends = interval.ends or "23:59"
            local since_midnight = schedule_clock.since_midnight()

            if date_within(starts, starts, today) and
               since_midnight < minutes_since_midnight(parse_hour(interval_starts)) * 60
            then
                return false
            end

            if date_within(ends, ends, today) and
               since_midnight > minutes_since_midnight(parse_hour(interval_ends)) * 60 + 59
            then
                return false
            end

            log("schedule", "interval matches")
            return true
        end
    end

    local function select_next_find(pages)
        find_offset = find_offset + 1
        if find_offset > #pages then
            find_offset = 1
        end
        return pages[find_offset]
    end

    local function triggered_pages(pages)
        if #pages == 0 then
            return
        end
        if trigger == "next" then
            return {select_next_find(pages)}
        else
            return pages
        end
    end

    local function find_by_key(key)
        local pages = {}
        for schedule_id, schedule in ipairs(schedules) do
            for page_id, page in ipairs(schedule.pages) do
                if page.interaction.key == key then
                    pages[#pages+1] = Page(page)
                end
            end
        end
        return triggered_pages(pages)
    end

    local function find_by_gpio(pin)
        local pages = {}
        local key = string.format("gpio_%d", pin)
        for schedule_id, schedule in ipairs(schedules) do
            for page_id, page in ipairs(schedule.pages) do
                if page.interaction.key == key then
                    pages[#pages+1] = Page(page)
                end
            end
        end
        return triggered_pages(pages)
    end

    local function find_by_remote(remote)
        local pages = {}
        for schedule_id, schedule in ipairs(schedules) do
            for page_id, page in ipairs(schedule.pages) do
                if page.interaction.key == 'remote' and page.interaction.remote == remote then
                    pages[#pages+1] = Page(page)
                end
            end
        end
        return triggered_pages(pages)
    end

    local function get_page(schedule_id, page_id)
        local schedule = schedules[schedule_id]
        if not schedule then
            return
        end

        local page = schedule.pages[page_id]
        if not page then
            return
        end

        return Page(page)
    end

    local function get_debug_page()
        if debug_schedule_id and debug_page_id then
            return get_page(debug_schedule_id+1, debug_page_id+1)
        end
    end

    local function fill_pages_from_schedule(pages, schedule)
        local filtered_pages = {}
        for i, page in ipairs(schedule.pages) do
            page = Page(page)
            if page.can_show() then
                filtered_pages[#filtered_pages+1] = page
            end
        end

        print("filtered pages:", #filtered_pages)

        if #filtered_pages == 0 then
            return
        end

        local display_mode = schedule.display_mode or "all"

        -- take all pages
        if display_mode == "all" then
            log("schedule", "adding all pages")
            for p = 1, #filtered_pages do
                pages[#pages+1] = filtered_pages[p]
            end
            return
        end

        -- randomly select some/all from filtered pages
        local sample = ({
            ["random-1"] = 1,
            ["random-2"] = 2,
            ["random-3"] = 3,
            ["random-4"] = 4,
            ["random-5"] = 5,
            ["random-6"] = 6,
            ["random-all"] = 100000,
        })[display_mode]
        sample = math.min(sample, #filtered_pages)
        log("schedule", "selecting %d random pages", sample)
        permute(filtered_pages)
        for p = 1, sample do
            pages[#pages+1] = filtered_pages[p]
        end
    end

    local function get_scheduled_pages()
        local pages = {}
        for schedule_id, schedule in ipairs(schedules) do
            log("schedule", "checking schedule %s (%d)", schedule.name, schedule_id)
            if is_scheduled(schedule) then
                fill_pages_from_schedule(pages, schedule)
            end
        end
        return pages
    end

    local function get_fallback_cycle()
        local pages = {}
        for schedule_id, schedule in ipairs(schedules) do
            if schedule.scheduling.mode == "fallback" then
                fill_pages_from_schedule(pages, schedule)
            end
        end

        if #pages > 0 then
            log("get_fallback_cycle", "added %d pages scheduled as fallback", #pages)
        else
            if fallback.type == "image" then
                pages[#pages+1] = Page{
                    is_fallback = true,
                    duration = 5,
                    auto_duration = 5,
                    layout_id = -1,
                    overlap = 0,
                    tiles = {{
                        x1 = 0,
                        y1 = 0,
                        x2 = WIDTH,
                        y2 = HEIGHT,
                        asset = fallback,
                        type = 'image',
                        config = {
                            fit = true,
                        }
                    }}
                }
            else
                local duration = 5
                if fallback.metadata and fallback.metadata.duration then
                    duration = fallback.metadata.duration
                end
                pages[#pages+1] = Page{
                    is_fallback = true,
                    duration = duration,
                    auto_duration = duration,
                    layout_id = -1,
                    overlap = 0,
                    tiles = {{
                        x1 = 0,
                        y1 = 0,
                        x2 = WIDTH,
                        y2 = HEIGHT,
                        asset = fallback,
                        type = 'rawvideo',
                        config = {
                            audio = true,
                        }
                    }}
                }
            end
        end
        log("schedule", "fallback cycle len is %d", #pages)
        return pages
    end

    local function generate_cycle()
        local debug_page = get_debug_page()

        if debug_page then
            cycle_pages = {debug_page}
        else
            cycle_pages = get_scheduled_pages()
        end

        if #cycle_pages == 0 then
            log("generate_cycle", "no scheduled pages. using fallback")
            cycle_pages = get_fallback_cycle()
        end

        -- Ensure we always have at least one page to display
        if #cycle_pages == 0 then
            log("generate_cycle", "no fallback pages either. creating emergency fallback")
            cycle_pages = {{
                is_fallback = true,
                duration = 10,
                auto_duration = 10,
                layout_id = -1,
                tiles = {{
                    x1 = 0,
                    y1 = 0,
                    x2 = WIDTH,
                    y2 = HEIGHT,
                    asset = "no-content-info-beamer.jpg",
                    type = 'image',
                    config = {
                        fit = true,
                        fade_time = 0.5
                    }
                }}
            }}
        end

        log("generate_cycle", "generated cycle with %d pages", #cycle_pages)
    end

    local function get_prev()
        cycle_offset = cycle_offset - 1
        if cycle_offset < 1 then
            generate_cycle()
            cycle_offset = #cycle_pages
        end
        return cycle_pages[cycle_offset]
    end

    local function get_next()
        cycle_offset = cycle_offset + 1
        if cycle_offset > #cycle_pages then
            generate_cycle()
            cycle_offset = 1
        end
        return cycle_pages[cycle_offset]
    end

    return {
        get_prev = get_prev;
        get_next = get_next;
        find_by_key = find_by_key;
        find_by_gpio = find_by_gpio;
        find_by_remote = find_by_remote;
    }
end

local page_source = PageSource()
local job_queue = JobQueue()
local scheduler = Scheduler(page_source, job_queue)

local function init_streams(config)
    -- Initialize all configured streams
    for _, schedule in ipairs(config.schedules) do
        for _, page in ipairs(schedule.pages) do
            for _, tile in ipairs(page.tiles) do
                if tile.type == "stream" and tile.config.url then
                    -- Pre-initialize the stream
                    streams.get_stream(tile.config.url, tile.config.audio)
                end
            end
        end
    end
end

-- Add to your config update handler
util.json_watch("config.json", function(config)
    init_streams(config)  -- Initialize streams when config is loaded
    node.dispatch("config_updated", config)
    node.gc()
end)

local function update_qr_position(instance_id, settings)
    print("Attempting to update QR positioning for instance: " .. tostring(instance_id))

    local instance = qr_code_instances[instance_id]
    if not instance then
        print("ERROR: Cannot update position for non-existent QR instance ID: " .. tostring(instance_id))
        return false
    end

    if type(settings) ~= "table" then
        print("ERROR: Settings must be a table")
        return false
    end

    -- Ensure the instance has a position_config table
    if not instance.position_config then
        instance.position_config = { position = "bottom-right", margin = 20, custom_x = 0, custom_y = 0 } -- Initialize if missing
    end

    local config_updated = false
    -- Update individual settings if provided
    if settings.position then
        -- Validate position value
        local valid_positions = {
            ["top-left"] = true,
            ["top-right"] = true,
            ["bottom-left"] = true,
            ["bottom-right"] = true,
            ["custom"] = true
        }

        if valid_positions[settings.position] then
            instance.position_config.position = settings.position
            print("Instance " .. instance_id .. ": Updated position to " .. settings.position)
            config_updated = true
        else
            print("ERROR: Invalid position value: " .. settings.position)
        end
    end

    if settings.margin then
        instance.position_config.margin = settings.margin
        print("Instance " .. instance_id .. ": Updated margin to " .. settings.margin)
        config_updated = true
    end

    -- Handle custom_x and custom_y as percentages
    if settings.custom_x ~= nil then
        instance.position_config.custom_x = settings.custom_x
        print("Instance " .. instance_id .. ": Updated custom_x to " .. settings.custom_x .. "%")
        config_updated = true
    end

    if settings.custom_y ~= nil then
        instance.position_config.custom_y = settings.custom_y
        print("Instance " .. instance_id .. ": Updated custom_y to " .. settings.custom_y .. "%")
        config_updated = true
    end

    return config_updated
end

-- Helper function to validate and explain QR positioning relative to gl.setup dimensions
local function validate_qr_positioning(instance_id)
    local instance = qr_code_instances[instance_id]
    if not instance or not instance.position_config then
        print("ERROR: Invalid QR instance for positioning validation: " .. tostring(instance_id))
        return false
    end
    
    local pos_config = instance.position_config
    
    print("\n=== QR POSITIONING VALIDATION for " .. instance_id .. " ===")
    print("GL Setup Dimensions: " .. NATIVE_WIDTH .. " x " .. NATIVE_HEIGHT .. " pixels")
    print("Position Mode: " .. (pos_config.position or "unknown"))
    
    if pos_config.position == "custom" then
        local x_percent = pos_config.custom_x or 0
        local y_percent = pos_config.custom_y or 0
        local pixel_x = NATIVE_WIDTH * x_percent / 100
        local pixel_y = NATIVE_HEIGHT * y_percent / 100
        
        print("Custom Position:")
        print("  - X: " .. x_percent .. "% = " .. math.floor(pixel_x) .. " pixels from left")
        print("  - Y: " .. y_percent .. "% = " .. math.floor(pixel_y) .. " pixels from top")
        print("  - Coordinate System: (0,0) = top-left, (100,100) = bottom-right")
        
        -- Validate ranges
        if x_percent < 0 or x_percent > 100 then
            print("WARNING: custom_x (" .. x_percent .. "%) is outside 0-100% range")
        end
        if y_percent < 0 or y_percent > 100 then
            print("WARNING: custom_y (" .. y_percent .. "%) is outside 0-100% range")
        end
    else
        print("Preset Position: " .. (pos_config.position or "unknown"))
        print("Margin: " .. (pos_config.margin or 20) .. " pixels")
    end
    
    print("=== END VALIDATION ===\n")
    return true
end

-- === OVERLAY SYSTEM DEFINITIONS (moved here to be available for data_mapper handlers) ===

-- Coke Zero Overlay System
local coke_overlay = {
    enabled = true,  -- Enable by default
    image = nil,
    current_asset = "Courtside_logo.png",  -- Track current asset - default to Courtside
    position = "top-right",  -- top-left, top-right, bottom-left, bottom-right, center, custom
    margin = 20,
    scale = 0.2,  -- Make it bigger so it's more visible
    alpha = 0.9,  -- 90% opacity
    custom_x = 85,  -- 85% from left (only used if position = "custom")
    custom_y = 5,   -- 5% from top (only used if position = "custom")
    
    -- NEW: Preloaded images for instant switching
    preloaded_images = {},
    logos = {
        ["1"] = "Courtside_logo.png",
        ["2"] = "Coke_Zero_Revised_1_lowres.png"
    }
}

local function preload_logo_assets()
    log("coke_overlay", "Preloading logo assets for instant switching...")
    
    for key, asset_name in pairs(coke_overlay.logos) do
        if not coke_overlay.preloaded_images[asset_name] then
            local success, image = pcall(function()
                return resource.load_image{
                    file = asset_name,
                    mipmap = true,
                }
            end)
            
            if success then
                coke_overlay.preloaded_images[asset_name] = image
                log("coke_overlay", "Preloaded: %s", asset_name)
            else
                log("coke_overlay", "Failed to preload: %s - Error: %s", asset_name, tostring(image))
            end
        end
    end
    
    -- Set the default active image
    coke_overlay.image = coke_overlay.preloaded_images[coke_overlay.current_asset]
    log("coke_overlay", "Logo preloading complete. Ready for instant switching.")
end

-- Optimized function for instant logo switching
local function load_coke_overlay(asset_name)
    local actual_asset = asset_name or "Courtside_logo.png"
    
    -- Check if we have this asset preloaded
    if coke_overlay.preloaded_images[actual_asset] then
        -- INSTANT SWITCH - no loading delay!
        coke_overlay.image = coke_overlay.preloaded_images[actual_asset]
        coke_overlay.current_asset = actual_asset
        coke_overlay.enabled = true
        log("coke_overlay", "INSTANT SWITCH to: %s", actual_asset)
        return
    end
    
    -- Fallback: load on demand if not preloaded (shouldn't happen for our main logos)
    log("coke_overlay", "Asset not preloaded, loading on demand: %s", actual_asset)
    
    local success, image = pcall(function()
        return resource.load_image{
            file = actual_asset,
            mipmap = true,
        }
    end)
    
    if success then
        -- Cache it for next time
        coke_overlay.preloaded_images[actual_asset] = image
        coke_overlay.image = image
        coke_overlay.current_asset = actual_asset
        coke_overlay.enabled = true
        log("coke_overlay", "Loaded and cached: %s", actual_asset)
    else
        log("coke_overlay", "Failed to load overlay: %s - Error: %s", actual_asset, tostring(image))
        coke_overlay.enabled = false
        coke_overlay.current_asset = nil
    end
end

local function cleanup_preloaded_logos()
    for asset_name, image in pairs(coke_overlay.preloaded_images) do
        if image then
            image:dispose()
            log("coke_overlay", "Disposed preloaded image: %s", asset_name)
        end
    end
    coke_overlay.preloaded_images = {}
end

local function draw_coke_overlay()
    if not coke_overlay.enabled or not coke_overlay.image then
        return
    end
    
    local img_width, img_height = coke_overlay.image:size()
    local scaled_width = img_width * coke_overlay.scale
    local scaled_height = img_height * coke_overlay.scale
    
    local draw_x, draw_y
    
    if coke_overlay.position == "top-left" then
        draw_x = coke_overlay.margin
        draw_y = coke_overlay.margin
    elseif coke_overlay.position == "top-right" then
        draw_x = NATIVE_WIDTH - scaled_width - coke_overlay.margin
        draw_y = coke_overlay.margin
    elseif coke_overlay.position == "bottom-left" then
        draw_x = coke_overlay.margin
        draw_y = NATIVE_HEIGHT - scaled_height - coke_overlay.margin
    elseif coke_overlay.position == "bottom-right" then
        draw_x = NATIVE_WIDTH - scaled_width - coke_overlay.margin
        draw_y = NATIVE_HEIGHT - scaled_height - coke_overlay.margin
    elseif coke_overlay.position == "center" then
        draw_x = NATIVE_WIDTH / 2 - scaled_width / 2
        draw_y = NATIVE_HEIGHT / 2 - scaled_height / 2
    elseif coke_overlay.position == "custom" then
        draw_x = NATIVE_WIDTH * coke_overlay.custom_x / 100
        draw_y = NATIVE_HEIGHT * coke_overlay.custom_y / 100
    else
        draw_x = NATIVE_WIDTH - scaled_width - coke_overlay.margin
        draw_y = coke_overlay.margin
    end
    
    coke_overlay.image:draw(draw_x, draw_y, draw_x + scaled_width, draw_y + scaled_height, coke_overlay.alpha)
end

-- Scheduled Asset Overlay System
local scheduled_overlay = {
    enabled = false,
    image = nil,
    current_asset = nil,
    source_trigger = nil,
    position = "bottom-right",  -- top-left, top-right, bottom-left, bottom-right, center, custom
    margin = 20,
    scale = 0.3,
    alpha = 0.95,
    custom_x = 70,
    custom_y = 70,
}

local function resolve_page_asset_name(page)
    if not page or type(page.tiles) ~= "table" then
        return nil
    end

    for _, tile in ipairs(page.tiles) do
        local asset = tile and tile.asset
        if type(asset) == "table" then
            if asset.type == "image" and asset.asset_name then
                return asset.asset_name
            end
        elseif type(asset) == "string" then
            return asset
        end
    end
    return nil
end

local function load_scheduled_overlay_from_trigger(trigger_data)
    local trigger = trigger_data and tostring(trigger_data) or ""
    if trigger == "" then
        log("scheduled_overlay", "Missing trigger data")
        return false
    end

    local pages = page_source.find_by_remote(trigger)
    if not pages or #pages == 0 then
        log("scheduled_overlay", "No scheduled pages found for trigger: %s", trigger)
        return false
    end

    local asset_name = resolve_page_asset_name(pages[1])
    if not asset_name then
        log("scheduled_overlay", "No image asset found in first matching page for trigger: %s", trigger)
        return false
    end

    local success, image = pcall(function()
        return resource.load_image{
            file = asset_name,
            mipmap = true,
        }
    end)

    if not success then
        log("scheduled_overlay", "Failed to load scheduled overlay '%s': %s", asset_name, tostring(image))
        return false
    end

    scheduled_overlay.image = image
    scheduled_overlay.current_asset = asset_name
    scheduled_overlay.source_trigger = trigger
    scheduled_overlay.enabled = true
    log("scheduled_overlay", "Loaded scheduled overlay from trigger '%s': %s", trigger, asset_name)
    return true
end

local function draw_scheduled_overlay()
    if not scheduled_overlay.enabled or not scheduled_overlay.image then
        return
    end

    local img_width, img_height = scheduled_overlay.image:size()
    local scaled_width = img_width * scheduled_overlay.scale
    local scaled_height = img_height * scheduled_overlay.scale

    local draw_x, draw_y

    if scheduled_overlay.position == "top-left" then
        draw_x = scheduled_overlay.margin
        draw_y = scheduled_overlay.margin
    elseif scheduled_overlay.position == "top-right" then
        draw_x = NATIVE_WIDTH - scaled_width - scheduled_overlay.margin
        draw_y = scheduled_overlay.margin
    elseif scheduled_overlay.position == "bottom-left" then
        draw_x = scheduled_overlay.margin
        draw_y = NATIVE_HEIGHT - scaled_height - scheduled_overlay.margin
    elseif scheduled_overlay.position == "bottom-right" then
        draw_x = NATIVE_WIDTH - scaled_width - scheduled_overlay.margin
        draw_y = NATIVE_HEIGHT - scaled_height - scheduled_overlay.margin
    elseif scheduled_overlay.position == "center" then
        draw_x = NATIVE_WIDTH / 2 - scaled_width / 2
        draw_y = NATIVE_HEIGHT / 2 - scaled_height / 2
    elseif scheduled_overlay.position == "custom" then
        draw_x = NATIVE_WIDTH * scheduled_overlay.custom_x / 100
        draw_y = NATIVE_HEIGHT * scheduled_overlay.custom_y / 100
    else
        draw_x = NATIVE_WIDTH - scaled_width - scheduled_overlay.margin
        draw_y = NATIVE_HEIGHT - scaled_height - scheduled_overlay.margin
    end

    scheduled_overlay.image:draw(draw_x, draw_y, draw_x + scaled_width, draw_y + scaled_height, scheduled_overlay.alpha)
end

-- Device Info Display System
local device_info_display = {
    enabled = true,
    position = "top-left",
    margin = 20,
    font_size = 30,
    text_color = {1, 1, 1, 1},  -- white
    bg_color = {0, 0, 0, 0.7},  -- semi-transparent black
    padding = 15
}

local function pretty_print_json(tbl, indent)
    indent = indent or ""
    local lines = {}
    local is_arr = #tbl > 0
    if is_arr then
        table.insert(lines, indent .. "[")
        for i, v in ipairs(tbl) do
            local c = i < #tbl and "," or ""
            if type(v) == "table" then
                for _, l in ipairs(pretty_print_json(v, indent .. "  ")) do table.insert(lines, l) end
            elseif type(v) == "string" then
                table.insert(lines, indent .. '  "' .. tostring(v) .. '"' .. c)
            else
                table.insert(lines, indent .. "  " .. tostring(v) .. c)
            end
        end
        table.insert(lines, indent .. "]")
    else
        table.insert(lines, indent .. "{")
        local cnt, i = 0, 0
        for _ in pairs(tbl) do cnt = cnt + 1 end
        for k, v in pairs(tbl) do
            i = i + 1
            local c = i < cnt and "," or ""
            if type(v) == "table" then
                table.insert(lines, indent .. '  "' .. tostring(k) .. '": {')
                local n = pretty_print_json(v, indent .. "    ")
                for j = 2, #n - 1 do table.insert(lines, n[j]) end
                table.insert(lines, indent .. "  }" .. c)
            elseif type(v) == "string" then
                table.insert(lines, indent .. '  "' .. tostring(k) .. '": "' .. tostring(v) .. '"' .. c)
            elseif type(v) == "boolean" or type(v) == "number" then
                table.insert(lines, indent .. '  "' .. tostring(k) .. '": ' .. tostring(v) .. c)
            else
                table.insert(lines, indent .. '  "' .. tostring(k) .. '": null' .. c)
            end
        end
        table.insert(lines, indent .. "}")
    end
    return lines
end

local function draw_device_info_page()
    if not device_info then
        local f = resource.load_font("default-font.ttf")
        local msg = "No device information available"
        local w = f:width(msg, 40)
        f:write((NATIVE_WIDTH - w) / 2, NATIVE_HEIGHT / 2, msg, 40, 1, 1, 1, 1)
        return
    end
    local f = resource.load_font("default-font.ttf")
    local fs, lh, m, ts = 28, 34, 30, 40
    local lines = pretty_print_json(device_info, "")
    resource.create_colored_texture(0, 0, 0, 1):draw(0, 0, NATIVE_WIDTH, NATIVE_HEIGHT)
    local title = "Device API Response: /api/v1/device/" .. tostring(device_info.id or "???")
    f:write(m, m, title, ts, 0.2, 0.8, 1, 1)
    local y, max_lines = m + ts + 30, math.floor((NATIVE_HEIGHT - m - ts - 130) / lh)
    for i = 1, math.min(#lines, max_lines) do
        f:write(m, y, lines[i], fs, 0.8, 0.8, 0.8, 1)
        y = y + lh
    end
    if #lines > max_lines then
        f:write(m, y, "... (" .. (#lines - max_lines) .. " more lines)", fs, 0.5, 0.5, 0.5, 1)
    end
    f:write(m, NATIVE_HEIGHT - m - 24, "API: /device_info/page/off to exit", 24, 0.5, 0.5, 0.5, 1)
end

local function draw_system_info_page()
    helper.draw_system_info_page(system_info, NATIVE_WIDTH, NATIVE_HEIGHT)
end

local function draw_device_info()
    if not device_info_display.enabled or not device_info then
        return
    end
    
    -- Load font
    local font = resource.load_font("default-font.ttf")
    
    -- Prepare text lines
    local lines = {
        "Device ID: " .. tostring(device_info.id or "N/A"),
        "Serial: " .. tostring(device_info.serial or "N/A"),
        "Description: " .. tostring(device_info.description or "N/A"),
        "Location: " .. tostring(device_info.location or "N/A"),
        "Run: " .. tostring(device_info.run or "N/A"),
        "Status: " .. (device_info.is_online and "Online" or "Offline"),
        "Timezone: " .. tostring(device_info.timezone or "UTC"),
    }
    
    -- Calculate text dimensions
    local line_height = device_info_display.font_size * 1.5
    local max_width = 0
    
    for _, line in ipairs(lines) do
        local w = font:width(line, device_info_display.font_size)
        if w > max_width then
            max_width = w
        end
    end
    
    -- Calculate box dimensions
    local box_width = max_width + (device_info_display.padding * 2)
    local box_height = (#lines * line_height) + (device_info_display.padding * 2)
    
    -- Calculate position
    local box_x, box_y
    if device_info_display.position == "top-left" then
        box_x = device_info_display.margin
        box_y = device_info_display.margin
    elseif device_info_display.position == "top-right" then
        box_x = NATIVE_WIDTH - box_width - device_info_display.margin
        box_y = device_info_display.margin
    elseif device_info_display.position == "bottom-left" then
        box_x = device_info_display.margin
        box_y = NATIVE_HEIGHT - box_height - device_info_display.margin
    elseif device_info_display.position == "bottom-right" then
        box_x = NATIVE_WIDTH - box_width - device_info_display.margin
        box_y = NATIVE_HEIGHT - box_height - device_info_display.margin
    else
        box_x = device_info_display.margin
        box_y = device_info_display.margin
    end
    
    -- Draw background box
    local white_pixel = resource.create_colored_texture(1, 1, 1, 1)
    white_pixel:draw(box_x, box_y, box_x + box_width, box_y + box_height, device_info_display.bg_color)
    
    -- Draw text lines
    local text_x = box_x + device_info_display.padding
    local text_y = box_y + device_info_display.padding
    
    for i, line in ipairs(lines) do
        local y = text_y + ((i - 1) * line_height)
        font:write(text_x, y, line, device_info_display.font_size, unpack(device_info_display.text_color))
    end
end

-- === END OVERLAY SYSTEM DEFINITIONS ===

util.data_mapper{
    ["event/keyboard"] = function(raw_event)
        local event = json.decode(raw_event)
        dispatch_to_all_tiles("on_keyboard", event)
        return scheduler.handle_keyboard(event)
    end,
    ["event/pad"] = function(raw_event)
        local event = json.decode(raw_event)
        dispatch_to_all_tiles("on_pad", event)
        return scheduler.handle_gamepad(event)
    end,
    ["event/gpio"] = function(raw_event)
        local event = json.decode(raw_event)
        dispatch_to_all_tiles("on_gpio", event)
        return scheduler.handle_gpio(event)
    end,
    ["remote/trigger"] = function(data)
        -- The unified handle_remote_trigger now handles both QR and scheduler
        return scheduler.handle_remote_trigger(data)
    end,
    ["sys/cec/key"] = scheduler.handle_cec,
    ["plugin/(.*)/(.*)"] = function(tile_name, path, data)
        local impl = tile_loader.modules[tile_name]
        if impl and impl.data_trigger then
            impl.data_trigger(path, data)
        end
    end,
    -- Debug: Log all incoming data mapper calls
    ["(.*)"] = function(path, data)
        -- Don't return anything so other handlers can still process
    end,
    ["qr/position"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" and payload.id and payload.settings then
            update_qr_position(payload.id, payload.settings)
        end
    end,
    ["device_info"] = function(data)
        if data == "" or data == nil then return end
        local success, parsed = pcall(json.decode, data)
        if success and parsed then
            device_info = parsed
        end
    end,
    ["device_info/page"] = function(data)
        -- Default behavior: toggle or turn on
        print("Testing device_info/page-----------------------------------------------------------------------------")
        device_info_page_mode = not device_info_page_mode
    end,
    ["device_info/page/on"] = function(data)
        device_info_page_mode = true
    end,
    ["device_info/page/off"] = function(data)
        device_info_page_mode = false
    end,
    ["device_info/pagce_info/page"] = function(data)
        local json_str = data or ""
        if json_str:match("^data%s*:") then
            json_str = json_str:match("^data%s*:%s*(.*)$") or ""
        elseif json_str ~= "" and json_str:match("=") then
            for k, v in json_str:gmatch("([^=&]+)=([^&]*)") do
                if k == "data" or k == "json" or k == "payload" or k == "body" then
                    json_str = v
                    break
                end
            end
        end
        if json_str and json_str ~= "" then
            local ok, p = pcall(json.decode, json_str)
            if ok and p then
                system_info = p
                system_info_page_mode = true
            elseif json_str == "off" then
                system_info_page_mode = false
            else
                system_info_page_mode = not system_info_page_mode
            end
        else
            system_info_page_mode = not system_info_page_mode
        end
    end,
    ["root/device_info/pagce_info/page"] = function(data)
        local json_str = data or ""
        if json_str:match("^data%s*:") then
            json_str = json_str:match("^data%s*:%s*(.*)$") or ""
        elseif json_str ~= "" and json_str:match("=") then
            for k, v in json_str:gmatch("([^=&]+)=([^&]*)") do
                if k == "data" or k == "json" or k == "payload" or k == "body" then
                    json_str = v
                    break
                end
            end
        end
        if json_str and json_str ~= "" then
            local ok, p = pcall(json.decode, json_str)
            if ok and p then
                system_info = p
                system_info_page_mode = true
            elseif json_str == "off" then
                system_info_page_mode = false
            else
                system_info_page_mode = not system_info_page_mode
            end
        else
            system_info_page_mode = not system_info_page_mode
        end
    end,
    ["device_info/pagce_info/page/on"] = function(data) system_info_page_mode = true end,
    ["root/device_info/pagce_info/page/on"] = function(data) system_info_page_mode = true end,
    ["device_info/pagce_info/page/off"] = function(data) system_info_page_mode = false end,
    ["root/device_info/pagce_info/page/off"] = function(data) system_info_page_mode = false end,
    ["device_info/toggle"] = function(data)
        device_info_display.enabled = json.decode(data).enabled
    end,
    ["device_info/position"] = function(data)
        local settings = json.decode(data)
        device_info_display.position = settings.position or "top-left"
        device_info_display.margin = settings.margin or 20
    end,
    ["device_info/appearance"] = function(data)
        local settings = json.decode(data)
        device_info_display.font_size = settings.font_size or 30
        device_info_display.padding = settings.padding or 15
        if settings.text_color then
            device_info_display.text_color = settings.text_color
        end
        if settings.bg_color then
            device_info_display.bg_color = settings.bg_color
        end
    end,
    ["qr/appearance"] = function(data)
        local settings = json.decode(data)
        local needs_regen = qrcode_overlay.update_appearance(settings)
        if needs_regen then
            for id, instance in pairs(qr_code_instances) do
                if instance.is_visible and instance.trigger_data then
                    local new_draw_details = qrcode_overlay.handle_remote_trigger(instance.trigger_data, current_setup_id)
                    if new_draw_details then
                        instance.draw_details = new_draw_details
                    else
                        instance.is_visible = false
                        instance.draw_details = nil
                    end
                end
            end
        end
    end,
    ["qr/validate"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" and payload.id then
            validate_qr_positioning(payload.id)
        elseif type(payload) == "string" then
            -- Allow direct string ID
            validate_qr_positioning(payload)
        else
            -- Validate all instances if no specific ID provided
            print("Validating all QR instances...")
            for id, _ in pairs(qr_code_instances) do
                validate_qr_positioning(id)
            end
        end
    end,
    -- === COKE ZERO OVERLAY HANDLERS ===
    ["coke/load"] = function(data)
        local asset_name = data and data ~= "" and data or "Coke_Zero_Revised_1_lowres.png"
        load_coke_overlay(asset_name)
        log("coke_overlay", "Load command received for asset: %s", asset_name)
    end,

    ["coke/toggle"] = function(data)
        coke_overlay.enabled = not coke_overlay.enabled
        log("coke_overlay", "Overlay toggled: %s", coke_overlay.enabled and "enabled" or "disabled")
    end,

    ["coke/position"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" then
            if payload.position then coke_overlay.position = payload.position end
            if payload.margin then coke_overlay.margin = payload.margin end
            if payload.custom_x then coke_overlay.custom_x = payload.custom_x end
            if payload.custom_y then coke_overlay.custom_y = payload.custom_y end
            log("coke_overlay", "Position updated: %s (margin: %d)", coke_overlay.position, coke_overlay.margin)
        elseif type(payload) == "string" then
            coke_overlay.position = payload
            log("coke_overlay", "Position set to: %s", payload)
        end
    end,

    ["coke/appearance"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" then
            if payload.scale then coke_overlay.scale = payload.scale end
            if payload.alpha then coke_overlay.alpha = payload.alpha end
            log("coke_overlay", "Appearance updated: scale=%.2f, alpha=%.2f", coke_overlay.scale, coke_overlay.alpha)
        end
    end,
    -- === SCHEDULED ASSET OVERLAY HANDLERS ===
    ["schedule_overlay/trigger"] = function(data)
        local trigger = data and tostring(data) or ""
        if trigger == "" then
            log("scheduled_overlay", "schedule_overlay/trigger requires trigger value in data")
            return
        end
        load_scheduled_overlay_from_trigger(trigger)
    end,

    ["schedule_overlay/toggle"] = function(data)
        scheduled_overlay.enabled = not scheduled_overlay.enabled
        log("scheduled_overlay", "Overlay toggled: %s", scheduled_overlay.enabled and "enabled" or "disabled")
    end,

    ["schedule_overlay/off"] = function(data)
        scheduled_overlay.enabled = false
        log("scheduled_overlay", "Overlay disabled")
    end,

    ["schedule_overlay/position"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" then
            if payload.position then scheduled_overlay.position = payload.position end
            if payload.margin then scheduled_overlay.margin = payload.margin end
            if payload.custom_x then scheduled_overlay.custom_x = payload.custom_x end
            if payload.custom_y then scheduled_overlay.custom_y = payload.custom_y end
            log("scheduled_overlay", "Position updated: %s (margin: %d)", scheduled_overlay.position, scheduled_overlay.margin)
        elseif type(payload) == "string" then
            scheduled_overlay.position = payload
            log("scheduled_overlay", "Position set to: %s", payload)
        end
    end,

    ["schedule_overlay/appearance"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" then
            if payload.scale then scheduled_overlay.scale = payload.scale end
            if payload.alpha then scheduled_overlay.alpha = payload.alpha end
            log("scheduled_overlay", "Appearance updated: scale=%.2f, alpha=%.2f", scheduled_overlay.scale, scheduled_overlay.alpha)
        end
    end,

    ["schedule_overlay/status"] = function(data)
        log("scheduled_overlay", "=== SCHEDULED OVERLAY STATUS ===")
        log("scheduled_overlay", "Enabled: %s", tostring(scheduled_overlay.enabled))
        log("scheduled_overlay", "Source trigger: %s", tostring(scheduled_overlay.source_trigger))
        log("scheduled_overlay", "Current asset: %s", tostring(scheduled_overlay.current_asset))
        log("scheduled_overlay", "Position: %s (margin: %d)", scheduled_overlay.position, scheduled_overlay.margin)
        log("scheduled_overlay", "Scale: %.2f, Alpha: %.2f", scheduled_overlay.scale, scheduled_overlay.alpha)
        log("scheduled_overlay", "Custom Position: %.1f%%, %.1f%%", scheduled_overlay.custom_x, scheduled_overlay.custom_y)
        log("scheduled_overlay", "Image Loaded: %s", scheduled_overlay.image and "yes" or "no")
        log("scheduled_overlay", "==============================")
    end,

    ["overlay/clear"] = function(data)
        -- Hide QR codes
        for id, instance in pairs(qr_code_instances) do
            instance.is_visible = false
        end
        -- Disable coke overlay 
        coke_overlay.enabled = false
        coke_overlay.image = nil
        -- Disable scheduled overlay
        scheduled_overlay.enabled = false
        scheduled_overlay.image = nil
        scheduled_overlay.current_asset = nil
        scheduled_overlay.source_trigger = nil
        log("overlay_clear", "Manual overlay clear requested")
        return "All overlays cleared"
    end,
    

    ["coke/status"] = function(data)
        log("coke_overlay", "=== COKE ZERO OVERLAY STATUS ===")
        log("coke_overlay", "Enabled: %s", tostring(coke_overlay.enabled))
        log("coke_overlay", "Position: %s (margin: %d)", coke_overlay.position, coke_overlay.margin)
        log("coke_overlay", "Scale: %.2f, Alpha: %.2f", coke_overlay.scale, coke_overlay.alpha)
        log("coke_overlay", "Custom Position: %.1f%%, %.1f%%", coke_overlay.custom_x, coke_overlay.custom_y)
        log("coke_overlay", "Image Loaded: %s", coke_overlay.image and "yes" or "no")
        log("coke_overlay", "===============================")
    end,
    
    -- === LOGO OVERLAY SWITCHING API ===

    ["logo/switch"] = function(data)
        local start_time = sys.now()
        print("LOGO SWITCH CALLED! Raw data: " .. tostring(data))
        
        -- Data comes directly as the form field value, no JSON parsing needed
        local trigger = data and tostring(data) or "1"
        
        print("FINAL TRIGGER VALUE: " .. trigger)
        
        -- Use the optimized preloaded system for instant switching
        if coke_overlay.logos[trigger] then
            local asset_name = coke_overlay.logos[trigger]
            load_coke_overlay(asset_name)
            local end_time = sys.now()
            local switch_time_ms = (end_time - start_time) * 1000
            print("INSTANT SWITCHED TO: " .. asset_name .. " in " .. string.format("%.2f", switch_time_ms) .. "ms")
            log("logo_switch", "Instant switch to: %s (%.2fms)", asset_name, switch_time_ms)
        else
            print("INVALID TRIGGER: " .. tostring(trigger) .. " (valid: 1, 2)")
            log("logo_switch", "Invalid trigger: %s. Use '1' for Courtside or '2' for Coke Zero", trigger)
        end
    end,
    

    ["logo/set"] = function(data)
        local logo_name = data and data ~= "" and data or "Courtside_logo.png"
        
        if logo_name == "Courtside_logo.png" or logo_name == "Coke_Zero_Revised_1_lowres.png" then
            load_coke_overlay(logo_name)
            log("logo_switch", "Logo set to: %s", logo_name)
        else
            log("logo_switch", "Invalid logo: %s", logo_name)
        end
    end,
    

    ["logo/toggle"] = function(data)
        local current_asset = coke_overlay.current_asset or "Courtside_logo.png"
        
        if current_asset == "Courtside_logo.png" then
            load_coke_overlay("Coke_Zero_Revised_1_lowres.png")
            log("logo_switch", "Toggled to Coke Zero logo")
        else
            load_coke_overlay("Courtside_logo.png")
            log("logo_switch", "Toggled to Courtside logo")
        end
    end,
    

    ["logo/status"] = function(data)
        log("logo_switch", "=== LOGO SYSTEM STATUS ===")
        log("logo_switch", "Current: %s, Enabled: %s", 
            coke_overlay.current_asset or "unknown", 
            tostring(coke_overlay.enabled))
        
        -- Show preloading status
        local preloaded_count = 0
        for asset_name, image in pairs(coke_overlay.preloaded_images) do
            preloaded_count = preloaded_count + 1
            log("logo_switch", "Preloaded: %s (%s)", asset_name, image and "ready" or "failed")
        end
        
        log("logo_switch", "Total preloaded assets: %d", preloaded_count)
        log("logo_switch", "Ready for instant switching: %s", preloaded_count > 0 and "YES" or "NO")
        log("logo_switch", "========================")
    end,
    
    -- Test endpoint to verify API calls are working
    ["logo/test"] = function(data)
        log("logo_switch", "=== LOGO TEST ENDPOINT ===")
        log("logo_switch", "API call received successfully!")
        log("logo_switch", "Data: '%s'", tostring(data))
        log("logo_switch", "Time: %s", tostring(sys.now()))
        log("logo_switch", "=========================")
    end,
    
    -- Comprehensive debug endpoint to test data formats
    
    -- API: Create or update QR code instance
    ["qr/instance"] = function(data)
        local payload = json.decode(data or "{}")
        
        if not payload.asset_id then
            log("qr", "ERROR: asset_id required")
            return
        end
        
        local position_config = {}
        if payload.position then position_config.position = payload.position end
        if payload.margin then position_config.margin = payload.margin end
        if payload.custom_x then position_config.custom_x = payload.custom_x end
        if payload.custom_y then position_config.custom_y = payload.custom_y end
        
        local instance_id = create_or_update_qr_instance(payload.asset_id, position_config)
        log("qr", "Created/updated QR instance: %s", instance_id)
        
        if payload.auto_show then
            local instance = qr_code_instances[instance_id]
            if instance then
                local new_draw_details = qrcode_overlay.handle_remote_trigger(instance.trigger_data, current_setup_id)
                if new_draw_details then
                    instance.draw_details = new_draw_details
                    instance.is_visible = true
                    log("qr", "Auto-showing QR instance: %s", instance_id)
                else
                    log("qr", "ERROR: Failed to generate QR for instance: %s", instance_id)
                end
            end
        end
    end,
    
    -- API: Remove QR code instance
    ["qr/instance/remove"] = function(data)
        local payload = json.decode(data)
        if payload.asset_id and remove_qr_instance(payload.asset_id) then
            log("qr", "Removed QR instance: %s", payload.asset_id)
        end
    end,
    
    -- API: List all QR code instances
    ["qr/instance/list"] = function(data)
        local instances = list_qr_instances()
        log("qr", "QR instances: %d", table.getn(instances))
        for id, info in pairs(instances) do
            log("qr", "  %s: %s (%s)", id, info.asset_id, info.position_config.position or "default")
        end
    end,
    
    -- API: Get specific QR instance info
    ["qr/instance/get"] = function(data)
        local payload = json.decode(data)
        
        if not payload.asset_id then
            print("[QR_PACKAGE] ERROR: asset_id is required")
            return
        end
        
        local instance = get_qr_instance(payload.asset_id)
        if instance then
            print("[QR_PACKAGE] QR instance for asset_id " .. payload.asset_id .. ":")
            print("[QR_PACKAGE]   ID: " .. instance.id)
            print("[QR_PACKAGE]   Visible: " .. tostring(instance.is_visible))
            print("[QR_PACKAGE]   Position: " .. (instance.position_config.position or "unknown"))
            print("[QR_PACKAGE]   Custom X/Y: " .. (instance.position_config.custom_x or 0) .. "%, " .. (instance.position_config.custom_y or 0) .. "%")
            print("[QR_PACKAGE]   Margin: " .. (instance.position_config.margin or 20))
        else
            print("[QR_PACKAGE] No QR instance found for asset_id: " .. payload.asset_id)
        end
    end,
    -- SETUP-WIDE: QR instance management that syncs across all devices
    ["setup/qr/instance"] = function(data)
        local payload = json.decode(data)
        
        if not payload.asset_id then
            log("QR_SETUP", "ERROR: asset_id is required")
            return
        end
        
        local position_config = {}
        
        -- Extract position configuration from payload
        if payload.position then position_config.position = payload.position end
        if payload.margin then position_config.margin = payload.margin end
        if payload.custom_x then position_config.custom_x = payload.custom_x end
        if payload.custom_y then position_config.custom_y = payload.custom_y end
        
        -- Create or update the instance
        local instance_id = create_or_update_qr_instance(payload.asset_id, position_config)
        
        log("QR_SETUP", "Setup-wide QR instance created/updated for asset_id: %s (instance: %s)", 
            payload.asset_id, instance_id)
        
        -- If auto_show is true, immediately make it visible and generate QR
        if payload.auto_show then
            local instance = qr_code_instances[instance_id]
            if instance then
                local new_draw_details = qrcode_overlay.handle_remote_trigger(instance.trigger_data, current_setup_id)
                if new_draw_details then
                    instance.draw_details = new_draw_details
                    instance.is_visible = true
                    log("QR_SETUP", "Auto-showing setup-wide QR instance: %s", instance_id)
                else
                    log("QR_SETUP", "ERROR: Failed to generate QR for auto_show instance: %s", instance_id)
                end
            end
        end
    end,
    
    -- SETUP-WIDE: Remove QR instance across all devices
    ["setup/qr/instance/remove"] = function(data)
        local payload = json.decode(data)
        
        if not payload.asset_id then
            log("QR_SETUP", "ERROR: asset_id is required for removal")
            return
        end
        
        local success = remove_qr_instance(payload.asset_id)
        if success then
            log("QR_SETUP", "Successfully removed setup-wide QR instance for asset_id: %s", payload.asset_id)
        else
            log("QR_SETUP", "No QR instance found for asset_id: %s", payload.asset_id)
        end
    end,
    
    -- SETUP-WIDE: List all QR instances (same as device-level)
    ["setup/qr/instance/list"] = function(data)
        local instances = list_qr_instances()
        log("QR_SETUP", "Setup-wide QR instances:")
        for id, info in pairs(instances) do
            log("QR_SETUP", "  %s: asset_id=%s, visible=%s, position=%s (%.1f%%, %.1f%%)", 
                id, info.asset_id, tostring(info.is_visible), 
                info.position_config.position or "unknown",
                info.position_config.custom_x or 0,
                info.position_config.custom_y or 0)
        end
        if not next(instances) then
            log("QR_SETUP", "  No QR instances found")
        end
    end,
    -- API: Create or update QR code instance (with root/ prefix)
    ["root/qr/instance"] = function(data)
        print("[QR_PACKAGE] root/qr/instance handler called")
        print("[QR_PACKAGE] Raw data type: " .. type(data))
        print("[QR_PACKAGE] Raw data content: '" .. tostring(data) .. "'")
        print("[QR_PACKAGE] Raw data length: " .. string.len(tostring(data)))
        
        -- If data is empty, it might be that info-beamer passes the data differently
        -- Let's try to handle both cases: direct JSON string and empty data
        local payload
        local success, err = pcall(function()
            if data == "" or data == nil then
                -- If data is empty, maybe info-beamer doesn't pass the inner JSON
                -- In this case, we'll assume the API call structure is different
                print("[QR_PACKAGE] Data is empty - this might be normal for info-beamer API")
                payload = {
                    asset_id = "3",  -- Default for testing
                    custom_x = 20,
                    custom_y = 30,
                    position = "custom",
                    auto_show = true
                }
            else
                payload = json.decode(data)
            end
        end)
        
        if not success then
            print("[QR_PACKAGE] JSON decode failed: " .. tostring(err))
            return
        end
        
        print("[QR_PACKAGE] Decoded payload type: " .. type(payload))
        print("[QR_PACKAGE] Payload asset_id: " .. tostring(payload.asset_id))
        
        if not payload.asset_id then
            print("[QR_PACKAGE] ERROR: asset_id is required")
            return
        end
        
        local position_config = {}
        
        -- Extract position configuration from payload
        if payload.position then position_config.position = payload.position end
        if payload.margin then position_config.margin = payload.margin end
        if payload.custom_x then position_config.custom_x = payload.custom_x end
        if payload.custom_y then position_config.custom_y = payload.custom_y end
        
        -- Create or update the instance
        local instance_id = create_or_update_qr_instance(payload.asset_id, position_config)
        
        print("[QR_PACKAGE] Successfully created/updated QR instance for asset_id: " .. payload.asset_id .. " (instance: " .. instance_id .. ")")
        
        -- If auto_show is true, immediately make it visible and generate QR
        if payload.auto_show then
            local instance = qr_code_instances[instance_id]
            if instance then
                print("[QR_PACKAGE] Attempting to auto-show QR instance: " .. instance_id .. " with trigger_data: " .. instance.trigger_data)
                print("[QR_PACKAGE] Current setup_id: " .. tostring(current_setup_id))
                local new_draw_details = qrcode_overlay.handle_remote_trigger(instance.trigger_data, current_setup_id)
                if new_draw_details then
                    instance.draw_details = new_draw_details
                    instance.is_visible = true
                    print("[QR_PACKAGE] Auto-showing QR instance: " .. instance_id .. " - SUCCESS")
                else
                    print("[QR_PACKAGE] ERROR: Failed to generate QR for auto_show instance: " .. instance_id)
                    print("[QR_PACKAGE] qrcode_overlay.handle_remote_trigger returned nil")
                end
            else
                print("[QR_PACKAGE] ERROR: Instance not found for auto_show: " .. instance_id)
            end
        end
    end,
    
    -- API: Remove QR code instance (with root/ prefix)
    ["root/qr/instance/remove"] = function(data)
        print("[QR_PACKAGE] root/qr/instance/remove handler called")
        local payload = json.decode(data)
        
        if not payload.asset_id then
            print("[QR_PACKAGE] ERROR: asset_id is required for removal")
            return
        end
        
        local success = remove_qr_instance(payload.asset_id)
        if success then
            print("[QR_PACKAGE] Successfully removed QR instance for asset_id: " .. payload.asset_id)
        else
            print("[QR_PACKAGE] No QR instance found for asset_id: " .. payload.asset_id)
        end
    end,
    
    -- API: List all QR code instances (with root/ prefix)
    ["root/qr/instance/list"] = function(data)
        print("[QR_PACKAGE] root/qr/instance/list handler called")
        local instances = list_qr_instances()
        print("[QR_PACKAGE] Current QR instances:")
        for id, info in pairs(instances) do
            print("[QR_PACKAGE]   " .. id .. ": asset_id=" .. info.asset_id .. ", visible=" .. tostring(info.is_visible) .. 
                  ", position=" .. (info.position_config.position or "unknown") .. 
                  " (" .. (info.position_config.custom_x or 0) .. "%, " .. (info.position_config.custom_y or 0) .. "%)")
        end
        if not next(instances) then
            print("[QR_PACKAGE]   No QR instances found")
        end
    end,
    -- Test endpoint to verify package is updated
    ["root/test"] = function(data)
        print("[TEST] Package is running latest code! Data: " .. tostring(data))
        print("[TEST] Current time: " .. tostring(sys.now()))
        print("[TEST] Total QR instances: " .. table.getn(qr_code_instances))
    end,
    
    -- Debug endpoint to test different data formats
    ["root/debug_data"] = function(data)
        
        -- Try parsing as JSON
        local success, parsed = pcall(function()
            return json.decode(data)
        end)
        
        if success then
            if type(parsed) == "table" then
                for k, v in pairs(parsed) do
                end
            end
        else
        end
    end,
    -- === STEPHEN A. SMITH GIF OVERLAY HANDLERS === (COMMENTED OUT)
    --[[

    ["gif/load"] = function(data)
        local asset_name = data and data ~= "" and data or "stephen_a_smith_weed.mp4"
        load_gif_overlay(asset_name)
        log("gif_overlay", "Load command received for asset: %s", asset_name)
    end,

    ["gif/toggle"] = function(data)
        gif_overlay.enabled = not gif_overlay.enabled
        log("gif_overlay", "Overlay toggled: %s", gif_overlay.enabled and "enabled" or "disabled")
    end,

    ["gif/position"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" then
            if payload.position then gif_overlay.position = payload.position end
            if payload.margin then gif_overlay.margin = payload.margin end
            if payload.custom_x then gif_overlay.custom_x = payload.custom_x end
            if payload.custom_y then gif_overlay.custom_y = payload.custom_y end
            log("gif_overlay", "Position updated: %s (margin: %d)", gif_overlay.position, gif_overlay.margin)
        elseif type(payload) == "string" then
            gif_overlay.position = payload
            log("gif_overlay", "Position set to: %s", payload)
        end
    end,

    ["gif/appearance"] = function(data)
        local payload = json.decode(data)
        if type(payload) == "table" then
            if payload.scale then gif_overlay.scale = payload.scale end
            if payload.alpha then gif_overlay.alpha = payload.alpha end
            log("gif_overlay", "Appearance updated: scale=%.2f, alpha=%.2f", gif_overlay.scale, gif_overlay.alpha)
        end
    end,

    ["gif/status"] = function(data)
        log("gif_overlay", "=== STEPHEN A. SMITH GIF OVERLAY STATUS ===")
        log("gif_overlay", "Enabled: %s", tostring(gif_overlay.enabled))
        log("gif_overlay", "Position: %s (margin: %d)", gif_overlay.position, gif_overlay.margin)
        log("gif_overlay", "Scale: %.2f, Alpha: %.2f", gif_overlay.scale, gif_overlay.alpha)
        log("gif_overlay", "Custom Position: %.1f%%, %.1f%%", gif_overlay.custom_x, gif_overlay.custom_y)
        log("gif_overlay", "Image Loaded: %s", gif_overlay.image and "yes" or "no")
        log("gif_overlay", "===============================")
    end,
    --]]
    -- Simple test to verify API is working
    ["logo/ping"] = function(data)
        print("LOGO PING RECEIVED: " .. tostring(data))
        log("logo_test", "PING received with data: %s", tostring(data))
    end,
    
    -- Debug endpoint to check stream status and settings
    ["stream/debug"] = function(data)
        log("stream_debug", "=== STREAM DEBUG INFO ===")
        local all_streams = streams._get_all_streams()
        local stream_count = 0
        for key, stream in pairs(all_streams) do
            stream_count = stream_count + 1
            log("stream_debug", "Stream %d: %s", stream_count, stream.url)
            log("stream_debug", "  - UDP: %s", tostring(stream.is_udp))
            log("stream_debug", "  - Last used: %d frames ago", frame - stream.last_used)
        end
        log("stream_debug", "Total active streams: %d", stream_count)
        log("stream_debug", "Current frame: %d", frame)
        log("stream_debug", "========================")
    end,
}

local function initialize_qr_codes()
    print("Initializing predefined QR codes...")
    for id, instance in pairs(qr_code_instances) do
        if instance.is_visible then -- Check if it's set to be visible initially
            print("Pre-generating QR for initially visible instance: " .. id)
            local new_draw_details = qrcode_overlay.handle_remote_trigger(instance.trigger_data, current_setup_id)
            if new_draw_details then
                instance.draw_details = new_draw_details
                print("Successfully pre-generated QR for instance: " .. id)
            else
                print("ERROR: Failed to pre-generate QR for instance: " .. id)
                instance.is_visible = false -- Don't show if generation failed
            end
        end
    end
end

-- Call initialization after everything is set up, perhaps before first render or after config loaded
-- For now, let's assume it's called after first config update where current_setup_id is known.
-- A better place might be at the very end of the script or after the first config_updated event.
-- To ensure current_setup_id is available, let's call it after the first config is processed.
local first_config_loaded = false
node.event("config_updated", function(config)
    -- ... (existing config_updated logic for layouts, background, current_setup_id) ...
    layouts = config.layouts
    for _, layout in ipairs(layouts) do
        for _, tile in ipairs(layout.tiles) do
            local asset = tile.asset
            if asset.type == "image" or asset.type == "video" then
                log("config_updated", "fixing layout asset %s", asset.asset_name)
                asset.asset_name = resource.open_file(asset.asset_name)
            end
        end
    end
    background = config.background
    
    if config.__metadata and config.__metadata.setup_id then
        current_setup_id = config.__metadata.setup_id
        log("config_updated", "Stored setup_id: %s", current_setup_id)
    else
        log("config_updated", "setup_id not found in config metadata")
        -- current_setup_id remains "UNKNOWN_SETUP" or its previous value
    end

    -- Initialize/Pre-generate QR codes after the first config (to ensure current_setup_id is set)
    if not first_config_loaded and current_setup_id ~= "UNKNOWN_SETUP" then
        initialize_qr_codes()
        first_config_loaded = true
    end

    -- ... (existing config_updated logic for scheduler reset) ...
    local setup_id = config.__metadata.setup_id
    local config_hash = config.__metadata.config_hash
    local reset_mode = config.reset_mode

    local force_reset

    if reset_mode == "config" then
        force_reset = config_hash ~= last_config_hash
    elseif reset_mode == "setup" then
        force_reset = setup_id ~= last_setup_id
    elseif reset_mode == "in_forever" then
        force_reset = scheduled_forever and config_hash ~= last_config_hash
    else -- "none"
        force_reset = false
    end

    if force_reset then
        print("config updated: forcing scheduler reset")
        reset_scheduler()
    end

    last_setup_id = setup_id
    last_config_hash = config_hash
end)
-- Override the render function to add QR code display
function node.render()
    streams.tick()
    FontCache.tick()
    ImageCache.tick()
    screen.setup()

    gl.clear(background.r, background.g, background.b, background.a)

    if device_info_page_mode then
        draw_device_info_page()
        return
    end
    
    if system_info_page_mode then
        draw_system_info_page()
        return
    end

    local now = sys.now()
    scheduler.tick(now)

    dispatch_to_all_tiles("each_frame")

    job_queue.tick(now)

    dispatch_to_all_tiles("overlay")

    -- === Draw QR Code Instances ===
    local now_for_qr = sys.now() -- Use consistent time for expiry checks
    local total_drawn_qr = 0

    for id, instance in pairs(qr_code_instances) do
        if instance.is_visible and instance.draw_details and instance.position_config then
            local pos_config = instance.position_config
            local dimensions = instance.draw_details.dimensions

            -- Use actual dimensions from draw_details for positioning calculation
            local qr_width = dimensions.total_width
            local qr_height = dimensions.total_height
            local margin = pos_config.margin or 20 -- Default margin if not set

            -- Calculate position based on selected corner or custom coordinates
            -- Note: The draw function expects the top-left corner (x, y) of the QR code *data* area,
            -- not including the border or title height.
            local qr_draw_x, qr_draw_y
            if pos_config.position == "top-left" then
                qr_draw_x = margin + dimensions.border_size
                qr_draw_y = margin + dimensions.title_height + dimensions.border_size
            elseif pos_config.position == "top-right" then
                qr_draw_x = NATIVE_WIDTH - qr_width - margin + dimensions.border_size
                qr_draw_y = margin + dimensions.title_height + dimensions.border_size
            elseif pos_config.position == "bottom-left" then
                qr_draw_x = margin + dimensions.border_size
                qr_draw_y = NATIVE_HEIGHT - qr_height - margin + dimensions.title_height + dimensions.border_size
            elseif pos_config.position == "bottom-right" then
                qr_draw_x = NATIVE_WIDTH - qr_width - margin + dimensions.border_size
                qr_draw_y = NATIVE_HEIGHT - qr_height - margin + dimensions.title_height + dimensions.border_size
            elseif pos_config.position == "custom" then
                -- Convert percentage to pixels for custom positioning
                -- Position relative to NATIVE_WIDTH/NATIVE_HEIGHT (gl.setup dimensions)
                -- This ensures consistent positioning regardless of screen settings
                local x_percent = pos_config.custom_x or 0
                local y_percent = pos_config.custom_y or 0
                
                -- Calculate base position as percentage of gl.setup dimensions
                local base_x = NATIVE_WIDTH * x_percent / 100
                local base_y = NATIVE_HEIGHT * y_percent / 100
                
                -- For custom positioning, we position the entire QR code (including border/title)
                -- at the specified percentage, then adjust to get the data area coordinates
                qr_draw_x = base_x + dimensions.border_size
                qr_draw_y = base_y + dimensions.title_height + dimensions.border_size
                
                -- Debug output for custom positioning
                print(string.format("QR Instance %s: Custom positioning - %.1f%% x %.1f%% = (%.0f, %.0f) on %dx%d canvas", 
                    id, x_percent, y_percent, base_x, base_y, NATIVE_WIDTH, NATIVE_HEIGHT))
            else
                -- Default to bottom-right if invalid position
                qr_draw_x = NATIVE_WIDTH - qr_width - margin + dimensions.border_size
                qr_draw_y = NATIVE_HEIGHT - qr_height - margin + dimensions.title_height + dimensions.border_size
            end

            -- Try to draw the QR code instance
            local drawn = qrcode_overlay.draw_qr(instance.draw_details, qr_draw_x, qr_draw_y, now_for_qr)

            if drawn then
                total_drawn_qr = total_drawn_qr + 1
            else
                -- If drawing failed (e.g., expired), mark as not visible
                print("QR instance " .. id .. " failed to draw (likely expired). Hiding.")
                instance.is_visible = false
                instance.draw_details = nil -- Clear details
            end
        end
    end

    draw_coke_overlay()
    draw_scheduled_overlay()
    draw_device_info()
end

load_qr_instances()
preload_logo_assets()