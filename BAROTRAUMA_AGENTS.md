# Barotrauma LuaCsForBarotrauma Modding — AGENTS.md

General-purpose context document for AI agents starting work on any Barotrauma Lua or CS mod.
Read this in full before writing or modifying any code.

---

## REPOSITORIES AND REFERENCES

- **Barotrauma source code**: https://github.com/FakeFishGames/Barotrauma
  Split into BarotraumaClient, BarotraumaServer, BarotraumaShared. Game assets are NOT in the repo — code only. Use this constantly to verify field names, method signatures, and vanilla behavior before writing patches. Do not guess.
- **LuaCsForBarotrauma**: https://github.com/evilfactory/LuaCsForBarotrauma
  The mod runtime. Source of truth for what Lua APIs are exposed — Hook, Networking, Timer, Game, Entity, LuaUserData, etc.
- **Barotrauma Wiki**: https://barotraumagame.com/wiki/Main_Page
  Item identifiers, affliction names, game content references.

**On bundling repo files as context**: Tempting but becomes stale quickly — Barotrauma updates frequently. Reference the live GitHub URLs above to verify signatures rather than bundling a snapshot. When uncertain about a method or field, look it up rather than assuming.

---

## ENVIRONMENT

- **Engine**: Barotrauma (C#, MonoGame, FarseerPhysics/Box2D)
- **Mod runtime**: LuaCsForBarotrauma — Lua scripting (MoonSharp interpreter) + custom C# assemblies loaded at runtime via Harmony patching
- **Languages**: Lua (MoonSharp) and C# (Roslyn-compiled ACL assemblies)
- **Physics**: FarseerPhysics. Two unit systems: display units (pixels) and sim units (display/100). Never compare them directly.
- **Coordinate system**: Y increases downward in display space, upward in sim space.
- **Multiplayer**: Authoritative server. Clients receive physics state sync from server each frame.
- **Validated on**: Barotrauma 1.11.5.0

---

## ARCHITECTURE RULES

### Shared vs Client-only CS files

RunConfig.xml controls compilation scope.

- Physics patches (colliders, forces, movement) MUST be Shared. Client-only physics patches get overridden every frame by server sync, causing drag/jitter on the client.
- Rendering, particles, sound, UI patches go in Client-only files.
- Note: some content the base game treats as client-side by default (sounds, certain UI systems). Check the vanilla source before assuming something needs to be Shared.
- When unsure: Shared is always safe. Client-only is an optimization, not a default.

### Lua execution context

- `SERVER` — true on dedicated server or singleplayer host
- `CLIENT` — true on client, including singleplayer host (which is BOTH SERVER and CLIENT simultaneously)
- `Game.IsMultiplayer` — true only in actual multiplayer sessions
- Files in `Lua/Autorun/` run on both server and client unless guarded with `if SERVER then` / `if CLIENT then`
- **Lua globals persist between sessions. CS static fields reset on assembly reload.**

### State ownership and networking — the most important architecture rule

**Lua owns all state. Lua does all networking. CS only reads state and applies effects.**

This is not just a convention — it is forced by how the engine works in multiplayer:

- `Networking.Start`, `Networking.Send`, and `Networking.Receive` are Lua APIs. CS has no direct equivalent.
- CS console command handlers execute server-side only in multiplayer. Any CS code that needs to broadcast to clients must do it by calling back into Lua via `GameMain.LuaCs.Lua.DoString(...)`.
- Lua globals persist between sessions. CS static fields reset on every assembly reload. Lua is the only reliable persistent store.
- CS `DebugConsole.NewMessage` only appears on the server console. Client-visible feedback must go through Lua networking.

The division of responsibility:

| Layer | Responsibility |
|---|---|
| Lua | All state globals, all networking send/receive, all config, session persistence |
| CS | Reading Lua globals via getter methods, applying physics patches, rendering patches |
| CS command handlers | Toggle state via setter (writes to Lua), then DoString to trigger Lua broadcast |

CS should never attempt to own state or drive networking directly. If you find yourself writing CS networking code, stop — route it through Lua instead.

---

## PERFORMANCE — READ THIS FIRST

**The game is CPU-bound and already under significant load during normal play. Every hook you add has a cost.**

- Never run heavy logic every frame. The `think` hook fires every tick — use a cooldown counter.
- Never iterate all characters every frame. Throttle updates to every 1-2 seconds minimum.
- When iterating characters for updates, spread them over time using staggered `Timer.Wait` calls. This prevents a single-frame spike when many characters are active.
- Prefer running logic once — on `roundStart`, on client connect, on load — over checking a condition every frame.
- The simple solution (check every frame, iterate everything) is almost never the right solution in Barotrauma. Always ask: can this run less often?

```lua
-- Standard throttled staggered update pattern
local cooldown = 0
local interval = 120  -- ~2 seconds at 60fps

Hook.Add("think", "mymod_update", function()
    cooldown = cooldown - 1
    if cooldown > 0 then return end
    cooldown = interval

    local chars = {}
    for _, c in pairs(Character.CharacterList) do
        if not c.IsDead and c.IsHuman and c.Enabled then
            table.insert(chars, c)
        end
    end
    if #chars == 0 then return end

    for i, c in ipairs(chars) do
        Timer.Wait(function()
            -- Always double-guard: character may have died between scheduling and execution
            if c ~= nil and not c.Removed and not c.IsDead then
                updateCharacter(c)
            end
        end, (i / #chars) * (interval / 60) * 1000)
    end
end)
```

---

## DEBUGGING PROTOCOL — MANDATORY

**If a bug's cause is not directly known, or you fail to solve a problem more than twice in a row, you MUST do one of the following before making another attempt:**

**Option 1 — Add explicit debug output**
Add `print()` statements or a diagnostic console command that dumps all relevant state at the point of failure: CS fields, Lua globals, patch counts, command registration counts, network message traces. Do not add more code changes without first understanding the actual state at the point of failure.

**Option 2 — Request user help**
Ask the user to run a specific in-game command or reproduce a specific scenario and report back what the console output says. Be explicit about exactly what information you need and why. This is not a failure — it is the correct engineering approach when you lack ground truth.

**Do not make a third attempt at fixing a bug without data from one of these two approaches.** Guessing a third time without knowing the actual state wastes time and risks breaking things that were already working. Reaching out to the user earlier almost always resolves issues faster than continued guessing.

A useful diagnostic command pattern:
```csharp
// Register in OnLoadCompleted — remove stale version first
DebugConsole.Commands.RemoveAll(c => c.Names.Any(n => n.Value == "mymod_state"));
DebugConsole.Commands.Add(new DebugConsole.Command("mymod_state", "Dump all mod state",
    (string[] args) => {
        DebugConsole.NewMessage("[MyMod] === State Dump ===", Color.Cyan);
        DebugConsole.NewMessage("[MyMod] CS FeatureFlag=" + FeatureFlag, Color.Cyan);
        try {
            var luaVal = GameMain.LuaCs.Lua.Globals["featureFlag"];
            DebugConsole.NewMessage("[MyMod] Lua featureFlag=" + luaVal, Color.Cyan);
        } catch (Exception e) {
            DebugConsole.NewMessage("[MyMod] Lua read failed: " + e.Message, Color.Red);
        }
        int cmdCount = DebugConsole.Commands.Count(c =>
            c.Names.Any(n => n.Value == "MyToggle"));
        DebugConsole.NewMessage("[MyMod] Commands registered=" + cmdCount + " (expect 1)", Color.Cyan);
        int patchCount = 0;
        foreach (var m in Harmony.GetAllPatchedMethods()) {
            var info = Harmony.GetPatchInfo(m);
            foreach (var p in info.Prefixes)
                if (p.owner.StartsWith("mymod")) { patchCount++; break; }
        }
        DebugConsole.NewMessage("[MyMod] Patched methods=" + patchCount, Color.Cyan);
        DebugConsole.NewMessage("[MyMod] === End Dump ===", Color.Cyan);
    }
));
```

---

## CRITICAL PITFALLS

### Reflection

**AccessTools.Property vs AccessTools.Field**
Fields and properties look identical in C# source but require different reflection accessors. `AccessTools.Property` on a field returns null silently — no error is thrown, any math using the result produces 0. Always check the Barotrauma source repo to confirm whether a member is a field or property before writing the accessor.
```csharp
// WRONG — returns null silently when target is a field
var p = AccessTools.Property(typeof(SomeClass), "SomeField");
// CORRECT
var f = AccessTools.Field(typeof(SomeClass), "SomeField");
```

**Hull.Surface is independent of WaterVolume**
`Hull.Surface` is a private `surface` field maintained by the wave simulation independently of `WaterVolume`. Setting `WaterVolume = 0` does NOT change `surface`. `HumanoidAnimController.UpdateStanding` reads `surface` directly to compute water slowdown. To suppress slowdown: temporarily zero `surface` via reflection in a Harmony prefix, restore it in the postfix.

**LuaUserData.MakeFieldAccessible crashes for mod assembly types**
Works for vanilla Barotrauma types. For types defined in your own mod assembly, throws:
`cannot convert a userdata to a clr type MoonSharp.Interpreter.Interop.IUserDataDescriptor`
Workaround: expose state via static getter/setter methods instead of fields.

**LuaUserData.CreateStatic is unreliable for mod assembly types**
May return nil or fail silently. Always wrap in `pcall` and design a fallback that does not depend on the call succeeding.

**Hook.Patch cannot reach private methods from Lua**
Use CS Harmony with `AccessTools.all` to patch private methods instead.

**partial void methods are patchable**
`partial void` methods compile to regular methods and are patchable with Harmony via `AccessTools.all`. The compiled name is just the method name with no modifiers.

**Use Descriptors[] not LuaUserData.GetType() for MakeFieldAccessible**
`Descriptors["Barotrauma.TypeName"]` is a global table pre-populated by LuaCsForBarotrauma and is reliable for vanilla types.
```lua
-- Correct for vanilla types:
LuaUserData.MakeFieldAccessible(Descriptors["Barotrauma.CharacterHealth"], "afflictions")
LuaUserData.MakeMethodAccessible(Descriptors["Barotrauma.DebugConsole"], "FindMatchingCharacter")
```

---

### Multiplayer

**Console commands route server-side**
In multiplayer, all console commands execute on the server regardless of who typed them. `DebugConsole.NewMessage` output only appears on the server console. To notify clients of a state change, explicitly broadcast via `Networking.Send` after the command runs.

**Game.ExecuteCommand routes server-side**
Calling `Game.ExecuteCommand("MyCommand")` from a Lua client receive handler runs the command on the server. The CS handler fires server-side only. Never use this for client-side state sync.

**CS static fields reset on assembly reload, Lua globals persist**
Between sessions, CS assemblies reload (resetting static fields to defaults) while Lua globals survive in memory. CS defaults then diverge from Lua persisted state. Always re-sync CS fields from Lua globals in `OnLoadCompleted`.

**DebugConsole.Commands double-registration**
If the old ACL assembly hasn't finished unloading when the new one loads, `OnLoadCompleted` fires twice and commands register twice. Always call `RemoveAll` before re-adding. Requires `using System.Linq` for `.Any()` on `ImmutableArray<Identifier>`.
```csharp
DebugConsole.Commands.RemoveAll(c => c.Names.Any(n => n.Value == "MyCommand"));
```

**Server physics drags client collider**
A client-only Harmony patch zeroing water forces gets overridden every frame by server physics sync. Physics suppression patches must run on the server — put them in Shared CS files.

**Lua After hooks fire regardless of CS prefix return value**
A Lua `Hook.Patch` After hook fires even if a CS Harmony prefix returned `false` (replacing the method). If both CS and Lua hook the same method, both will run. Design interactions explicitly.

**Item spawning race condition**
`Entity.Spawner` may be null immediately on load. Always guard before using the spawn queue, and retry via `Timer.Wait(fn, 35)` if null.
```lua
if not Entity.Spawner then
    Timer.Wait(function() spawnMyItem() end, 35)
    return
end
```

---

### Harmony

**Per-instance state in prefix/postfix pairs**
Never use a shared static field for state that must survive between a prefix and its postfix when multiple instances update in the same frame. Use a `Dictionary<InstanceType, Value>` keyed by instance. Clean up in a `Character.Remove` postfix to prevent memory leaks.
```csharp
// WRONG — clobbered when multiple characters update same frame
private static float savedValue;

// CORRECT
private static readonly Dictionary<HumanoidAnimController, float> savedValues = new();
// In prefix: savedValues[__instance] = currentValue; set to 0;
// In postfix: if (savedValues.TryGetValue(__instance, out var v)) { restore v; savedValues.Remove(__instance); }
// In Character.Remove postfix: if (__instance.AnimController is HumanoidAnimController a) savedValues.Remove(a);
```

---

### Particle system

**inWater parameter controls which draw target renders**
`ParticleManager.Draw(spriteBatch, inWater, inSub, blendState)`: `inWater=true` renders Water-target particles, `inWater=false` renders Air-target particles. Outside the sub, the engine calls `Draw(inWater=true, inSub=false)`. Particles with `drawtarget="Air"` never render outside because the Water draw pass skips them. Fix: patch `ParticleManager.Draw` to override `inWater=false` when `inSub==false`.

**Explosion underwater detection uses raw hull parameter**
Vanilla `ExplodeProjSpecific`: `hull ??= Hull.FindHull(worldPosition)` then `bool underwater = hull == null`. When outside, the raw `hull` parameter is null so `underwater=true` and only water particles spawn. To intercept: check the raw `hull` parameter BEFORE the null-coalescing fires. Do not call `Hull.FindHull` to decide whether to intercept.

---

### Hardcoded engine behaviors — accept as limitations

- **Platforms outside hulls**: Vanilla `OnLimbCollision` skips platforms when `currentHull == null`. Remove that check via Harmony to enable outdoor platform collision.
- **Stairs outside hulls**: Require holding movement direction. `GetFloorY` ray cast logic is hardcoded and not cleanly patchable without a transpiler.
- **Pressure affliction**: Resets every 0.1s. Must set `PressureProtection` every frame, not once.
- **AI waypoints outside hulls**: `WayPoint.IsInWater` returns true when `CurrentHull == null`. Patch `WayPoint.FindHull` postfix to assign nearest hull to outdoor waypoints so AI can path correctly.
- **NeedsDivingGear outside hulls**: Returns true when `hull == null`. Patch to return false.
- **Hull.Surface**: Wave simulation owns it, independent of WaterVolume. Only suppressable via reflection on the private field.

---

## REQUIRED PATTERNS

### CS state getter/setter (multiplayer-safe)

Always read Lua globals from CS rather than caching in static fields. Static field is a fallback only.

```csharp
public static bool FeatureFlag = false; // default / fallback

public static bool GetFeatureFlag() {
    try {
        var val = GameMain.LuaCs?.Lua?.Globals["featureFlag"];
        if (val is bool b) { FeatureFlag = b; return b; }
        if (val is DynValue dv && dv.Type == DataType.Boolean)
            { FeatureFlag = dv.Boolean; return dv.Boolean; }
    } catch { }
    return FeatureFlag; // fallback if Lua unavailable
}

public static void SetFeatureFlag(bool value) {
    FeatureFlag = value;
    try { GameMain.LuaCs.Lua.Globals["featureFlag"] = DynValue.NewBoolean(value); } catch {}
}
```

### Multiplayer-safe console command (CS, in OnLoadCompleted)

```csharp
DebugConsole.Commands.RemoveAll(c => c.Names.Any(n => n.Value == "MyToggle")); // dedup
DebugConsole.Commands.Add(new DebugConsole.Command("MyToggle", "Toggle my feature",
    (string[] args) => {
        GetFeatureFlag();           // sync CS from Lua first
        SetFeatureFlag(!FeatureFlag); // toggle, writes back to Lua via setter
        DebugConsole.NewMessage("[MyMod] MyToggle: " + (FeatureFlag ? "ON" : "OFF"), Color.LightGreen);
        // Broadcast new state to all clients via Lua
        try { GameMain.LuaCs.Lua.DoString(
            "if SERVER then " +
            "local m = Networking.Start('mymod_flag') " +
            "m.WriteBoolean(featureFlag) " +
            "for c in Client.ClientList do " +
            "Networking.Send(m, c.Connection, DeliveryMethod.Reliable) end end");
        } catch {}
    }
));
```

### Multiplayer state sync (Lua)

```lua
featureFlag = false  -- Lua global is source of truth

local function sendState(client)
    if not SERVER then return end
    local msg = Networking.Start("mymod_flag")
    msg.WriteBoolean(featureFlag)
    if client then
        Networking.Send(msg, client.Connection, DeliveryMethod.Reliable)
    else
        for c in Client.ClientList do
            Networking.Send(msg, c.Connection, DeliveryMethod.Reliable)
        end
    end
end

Hook.Add("roundStart", "mymod_roundstart", function()
    sendState(nil)
end)

Hook.Add("client.connected", "mymod_connect", function(c)
    sendState(c)
end)

if CLIENT then
    Networking.Receive("mymod_flag", function(msg)
        featureFlag = msg.ReadBoolean()
        print("[MyMod] featureFlag synced: " .. tostring(featureFlag))
    end)
end
```

### Safe item spawning

```lua
local function spawnItem(identifier, position)
    if Game.IsMultiplayer and CLIENT then return end
    if not Entity.Spawner then
        Timer.Wait(function() spawnItem(identifier, position) end, 35)
        return
    end
    Timer.Wait(function()
        local prefab = ItemPrefab.GetItemPrefab(identifier)
        Entity.Spawner.AddItemToSpawnQueue(prefab, position, nil, nil, nil)
    end, 35)
end
```

### Gated debug commands

```lua
-- Register expensive debug commands only when explicitly enabled, not on every load
local debugEnabled = false

local function registerDebugCommands()
    Game.AddCommand("mymod_dumpstate", "Dump all mod state", function(args)
        -- print all relevant state
    end, nil, true)
end

Game.AddCommand("mymod_debug", "Enable debug commands", function()
    if debugEnabled then return end
    debugEnabled = true
    registerDebugCommands()
    if SERVER then
        local msg = Networking.Start("mymod_debug_enabled")
        Networking.Send(msg)
    end
end, nil, true) -- server-only

if CLIENT and Game.IsMultiplayer then
    Networking.Receive("mymod_debug_enabled", function()
        registerDebugCommands()
    end)
end
```

### Mod conflict detection

```lua
local function checkConflicts()
    for prefab in ItemPrefab.Prefabs do
        if prefab.Identifier.Value == "mymod_sentinel_item" then
            local mod = prefab.ConfigElement.ContentPackage.Name
            if not string.find(string.lower(mod), "mymodname") then
                print("[MyMod] WARNING: Conflict detected with mod: " .. mod)
            end
        end
    end
end
Timer.Wait(checkConflicts, 1000) -- delay so all mods have fully loaded
```

### Error reporting with stack trace

```lua
LuaUserData.RegisterType("Barotrauma.ModUtils+Logging")
local Logging = LuaUserData.CreateStatic("Barotrauma.ModUtils+Logging")

local function traceError(msg)
    msg = "[MyMod] Error: " .. (msg or "unknown")
    Logging.PrintError(debug.traceback(msg, 2)) -- 2 excludes this wrapper from trace
end
```

### Environment isolation for modules

```lua
-- Give a Lua module its own environment so it doesn't pollute _G
local _ENV = setmetatable({}, { __index = _G })
-- ... module code here, isolated from global scope ...
return setmetatable({}, {
    __index = function(t, k) return _ENV[k] end,
    __newindex = function() error("Attempted to modify protected module table") end
})
```

---

## CODE STRUCTURE PATTERNS

Well-made mods in this space tend to follow consistent structural patterns. These are worth adopting:

**Helper function table**: Collect all utility functions into a single named global table (e.g. `HF = {}`). Avoids polluting global namespace and makes dependencies explicit. Worth building: `HF.Clamp`, `HF.Lerp`, `HF.Chance(probability)`, `HF.GetAfflictionStrength`, `HF.SetAffliction`, `HF.AddAffliction`, `HF.GiveItem`, `HF.RemoveItem`, `HF.CharacterToClient`, `HF.NormalizeLimbType`.

**Namespace separation**: Main mod table = mod-specific API. Helper table = general utilities reusable by add-on mods. This keeps the surface clean for extensibility without coupling.

**Config via JSON**: Store config in `Game.SaveFolder .. "/ModConfigs/MyMod.json"`. Serialize the entire config table as a JSON string. Send the full JSON as a single network message rather than individual fields. On receive, iterate the received table and apply only known keys — ignore unknown keys for forward compatibility with expansions.

**Config authority**: Server owns config. On `client.connected`, server sends full config to the connecting client. Client can send update requests; server validates `sender.HasPermission(ClientPermissions.ManageSettings)` before applying and rebroadcasting.

**SendDirectChatMessage for per-client feedback**: `Game.SendDirectChatMessage(chatMessage, client)` sends to one client only — useful for command feedback in multiplayer without broadcasting to everyone.

**NormalizeLimbType**: Barotrauma has granular limb types (Forearm, Thigh, Foot, Hand) but most affliction logic needs only the 6 canonical types (Head, Torso, LeftArm, RightArm, LeftLeg, RightLeg). Build a normalizer that maps granular → canonical before applying afflictions.

**TraceError with stack trace**: `debug.traceback(errorMessage, 2)` gives meaningful Lua stack traces. The level `2` excludes the error wrapper function itself from the trace output.

---

## FILELIST.XML AND DUMMY FILE

The `filelist.xml` declares all content to the engine using `%ModDir%` as the root path.

**The Dummy.xml rule**: You need a `Dummy.xml` — a minimal file with just an `<Items />` root tag — to prevent engine load errors when your mod defines no real item, structure, or affliction XML. You do NOT need it if your mod already defines or overrides any of those content types, since the engine will have a valid XML root to parse.

Note that some content types (sounds, certain UI elements) are treated as client-side by the base game. Keep this in mind when declaring files — a sound file referenced server-side may simply not load.

**When in doubt, use `<Other />`**. It is the catch-all tag and works broadly for any file type. You do not need to use every specific content type tag — only use typed tags when the engine needs to process that content in a specific way (e.g. for item inheritance, affliction overriding, particle registration).

```xml
<!-- Minimal filelist.xml -->
<ContentPackage name="MyMod" ...>
    <Other file="%ModDir%/Lua/Autorun/init.lua" />
    <!-- Add only what you actually have: -->
    <!-- <Item file="%ModDir%/Items.xml" /> -->
    <!-- <Afflictions file="%ModDir%/Afflictions.xml" /> -->
</ContentPackage>
```

---

## RUNCONFIG.XML TEMPLATE

```xml
<ContentPackage>
  <CSharp>
    <Shared>
      <Assembly name="MyMod">
        <Files>
          <File path="CSharp/Shared/MyMod.cs" />
        </Files>
      </Assembly>
    </Shared>
    <Client>
      <Assembly name="MyModClient">
        <Files>
          <File path="CSharp/Client/MyModClient.cs" />
        </Files>
      </Assembly>
    </Client>
  </CSharp>
</ContentPackage>
```

---

## DEFINITIONS TEMPLATE

Fill this out at the start of any project to document all persistent state. An agent reading this should be able to reconstruct the full mod state from scratch without running the game.

### Lua globals (source of truth)
```
name        type    default    Purpose. Which file owns it. When it changes.
```

### CS static fields (mirrors — read via getter methods only, never directly in patches)
```
ClassName.FieldName    type    default    Which Lua global this mirrors.
```

### Networking message identifiers
```
"mymod_messagename"    direction (server→client / client→server)    Data payload. When sent.
```

### Console commands
```
CommandName    Shared/Client    What it does. Whether it broadcasts to clients. Expected registration count.
```

### Harmony patches
```
TypeName.MethodName    Prefix/Postfix/Both    Shared/Client    What it does. Known gotchas.
```
