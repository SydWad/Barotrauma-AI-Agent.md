# Barotrauma LuaCsForBarotrauma Modding — AGENTS.md

## ABOUT THIS FILE

This document is **machine-readable primary**. It is structured for consumption by AI coding agents
before writing or modifying any Barotrauma mod code. Rules are stated as hard constraints, not
suggestions. Keep all entries terse and precise. Do not include project-specific identifiers,
mod names, file paths from any particular project, or content that only applies to one mod.
When adding entries, generalize or abstract from the specific case that motivated the entry.

---

## REPOSITORIES AND REFERENCES

- **Barotrauma source**: https://github.com/FakeFishGames/Barotrauma
  Split into BarotraumaClient, BarotraumaServer, BarotraumaShared. Code only — no assets.
  Verify all field names, method signatures, and vanilla behavior here before writing patches.
- **LuaCsForBarotrauma**: https://github.com/evilfactory/LuaCsForBarotrauma
  Source of truth for exposed Lua APIs: Hook, Networking, Timer, Game, Entity, LuaUserData, etc.
- **LuaCS API docs**: https://evilfactory.github.io/LuaCsForBarotrauma/lua-docs/
- **LuaCS Server CS docs**: https://evilfactory.github.io/LuaCsForBarotrauma/cs-docs/baro-server/html/
- **BaroModDoc**: https://regalis11.github.io/BaroModDoc/
- **Barotrauma Wiki**: https://barotraumagame.com/wiki/Main_Page

Do not bundle repo snapshots as context — Barotrauma updates frequently. Look up live sources
when uncertain about any method or field.

---

## ENVIRONMENT

- **Engine**: Barotrauma (C#, MonoGame, FarseerPhysics/Box2D)
- **Mod runtime**: LuaCsForBarotrauma — MoonSharp Lua interpreter + Roslyn-compiled ACL assemblies
- **Languages**: Lua (MoonSharp) and C# (Roslyn ACL)
- **Physics**: FarseerPhysics. Display units (pixels) vs sim units (display/100). Never mix.
- **Coordinate system**: Y increases downward in display space, upward in sim space.
- **Multiplayer**: Authoritative server. Clients receive physics state sync from server each frame.

---

## ARCHITECTURE RULES

### CS compilation scope

`RunConfig.xml` controls which assemblies compile for which context.

- `CSharp/Shared/` — compiles for both server and client. Required for physics patches (client-only patches are overridden every frame by server sync). Safe default when scope is uncertain.
- `CSharp/Client/` — rendering, particles, sound, UI patches only.
- `CSharp/Server/` — methods that only exist in the server assembly (e.g. `Item.CreateServerEvent`). Calling these from a Shared file causes CS1061 at compile time.

### Preprocessor directives DO NOT work in Roslyn ACL

`#if SERVER` and `#if CLIENT` are **not defined** as C# preprocessor constants in the Roslyn ACL environment. Blocks guarded by them are always skipped. Use runtime checks instead:

```csharp
// WRONG — block is always skipped in Roslyn ACL shared assemblies
#if SERVER
    item.CreateServerEvent(panel);
#endif

// CORRECT — runtime guard
if (GameMain.NetworkMember?.IsServer ?? false)
{
    // still won't compile in Shared if the method is server-only — use CSharp/Server/
}
```

Methods that only exist in the server assembly (like `Item.CreateServerEvent`) must live in a
`CSharp/Server/` file. Use a static delegate callback registered by the server assembly so
Shared code can invoke server-only behavior without a compile-time reference:

```csharp
// In Shared:
public static Action<Item, Item> OnSomeServerEvent;  // null on client

// In CSharp/Server/MyServerHelper.cs:
public void Initialize() {
    SharedHelper.OnSomeServerEvent = (a, b) => {
        a.CreateServerEvent(a.GetComponent<ConnectionPanel>());
    };
}
public void Dispose() { SharedHelper.OnSomeServerEvent = null; }
```

### Lua execution context

| Global | True when |
|--------|-----------|
| `SERVER` | Dedicated server OR singleplayer host |
| `CLIENT` | Client OR singleplayer host (host is BOTH simultaneously) |
| `Game.IsMultiplayer` | Actual multiplayer session only |

Files in `Lua/Autorun/` run on both sides unless guarded with `if SERVER then` / `if CLIENT then`.

**Lua globals persist between sessions. CS static fields reset on every assembly reload.**

### State ownership

- Lua owns all persistent state and all networking.
- CS reads state and applies effects — it does not own state or drive networking directly.
- `Networking.Start`, `Networking.Send`, `Networking.Receive` are Lua-only APIs.
- CS console output (`DebugConsole.Log`) is server-console-only and not visible to clients.
  Write diagnostic values to Lua globals from CS; read them from the client console with `cl_lua`.

---

## PERFORMANCE

- Never run heavy logic every frame. The `think` hook fires every tick.
- Throttle iterations to every 1–2 seconds minimum. Use `Timer.Wait` for periodic work.
- `Timer.Wait` works in both the sub editor and in-game. `Timing.Step` is nil in the sub editor.
- Prefer one-shot logic (on `roundStart`, on connect, on load) over continuous polling.

```lua
local function scheduleLoop()
    Timer.Wait(function()
        pcall(myWorkFunction)
        scheduleLoop()
    end, 1000)
end
scheduleLoop()
```

---

## DEBUGGING PROTOCOL — MANDATORY

**If a bug's cause is not directly known, or two attempts have failed: add diagnostics before
attempting again. Do not make a third guess without ground-truth data.**

**Option 1 — Write to Lua globals from CS, read with `cl_lua`:**
```csharp
try { if (lua != null) lua.Globals["mymod_diag_value"] = someValue; } catch { }
```
Client reads: `cl_lua print(mymod_diag_value)`

**Option 2 — Expose a status function:**
```lua
_G.mymod_status = function()
    print("state_a=" .. tostring(mymod_state_a))
    print("state_b=" .. tostring(mymod_state_b))
end
```
Run: `cl_lua mymod_status()`

**Option 3 — Ask the user for specific console output.**

---

## COMMON COMPILE ERRORS

| Error | Cause | Fix |
|-------|-------|-----|
| CS0246 `Rectangle` | XNA type missing | `using Microsoft.Xna.Framework;` |
| CS0234 `FarseerPhysics.Dynamics.BodyType` | Wrong namespace | `FarseerPhysics.BodyType` (no `.Dynamics`) |
| CS0535 `IAssemblyPlugin` | Missing interface methods | Add all four methods |
| CS0656 `CSharpArgumentInfo.Create` | `dynamic` not available in Roslyn ACL | Use `IEnumerable` + `as` cast + reflection |
| CS1061 `Item.OnItemLoaded` | Wrong target | Method belongs to `ItemComponent`, not `Item` |
| CS1061 `Item.CreateServerEvent` | Server-only method | Move call to `CSharp/Server/` file |
| Null Harmony method | Wrong constructor param order | Verify from engine stack traces |

---

## IASSEMBLYPLUGIN — ALL FOUR METHODS REQUIRED

```csharp
public void Initialize() { }
public void OnLoadCompleted() { }
public void PreInitPatching() { }
public void Dispose() { }
```

Older examples may show only `Initialize` + `Dispose` — those were written against an older
interface. Missing any method causes CS0535.

---

## SESSION GUARD

```csharp
// CORRECT — true during normal play AND editor test mode (TestGameMode)
GameMain.GameSession?.IsRunning ?? false

// WRONG — Level.Loaded is null in TestGameMode (editor test sessions have no level)
Level.Loaded != null
```

---

## MULTIPLAYER ENTITY SPAWNING

**`new Item(..., Entity.NullEntityID)` does NOT sync to clients in multiplayer.**

Direct item construction bypasses the entity sync system. The server assigns an ID from its local
pool with no broadcast — clients never receive the entity. When the EntitySpawner later assigns
that same ID to a legitimate entity, both sides crash with an ID collision error.

**Always use `Entity.Spawner.AddItemToSpawnQueue` for any entity that clients need to see:**

```csharp
Entity.Spawner?.AddItemToSpawnQueue(prefab, worldPosition,
    onSpawned: spawnedItem =>
    {
        var component = spawnedItem.GetComponent<MyComponent>();
        if (component == null) { spawnedItem.Remove(); return; }
        // configure item here
    });
```

- Callback is **async** — fires on the next server tick, not immediately. Track in-flight spawns
  with a `HashSet<(ushort, ushort)>` (keyed on sorted ID pairs) to prevent duplicate spawns if
  the spawning function is called again before the callback fires. Remove from the set in the
  callback and in any error paths.
- Guard MP spawning to server only:
  ```csharp
  if (GameMain.NetworkMember != null && !GameMain.NetworkMember.IsServer) return;
  ```

### Wire-specific spawning

When programmatically connecting both ends of a wire via `AddItemToSpawnQueue`:

```csharp
wire.DropOnConnect = false;   // DEFAULT IS TRUE — wire ejects to floor when both ends connect
wire.IsActive = false;
if (wireItem.body != null) wireItem.body.Enabled = false;
connA.ConnectWire(wire);
connB.ConnectWire(wire);
wire.Connect(connA, 0, addNode: false, sendNetworkEvent: true);
wire.Connect(connB, 1, addNode: false, sendNetworkEvent: true);
```

`DropOnConnect = true` (default) causes the wire to become a loose physics item when both ends
are connected. Always set `false` when connecting programmatically.
`sendNetworkEvent: true` broadcasts connection state to clients.
Panel state sync (`CreateServerEvent`) must be in `CSharp/Server/` — use the delegate pattern.

---

## LUA HARD LIMITS

- **`List<T>` is NOT iterable from Lua.** `pairs()`, bracket indexing, and `for x in list`
  all fail on CLR list types. Any `List<MapEntity>`, `List<Item>`, etc. returned from the engine
  must be iterated in CS static helper methods, not Lua.
- **`Timing` is nil outside an active round and in the sub editor.** Use `Timer.Wait`.
- **`Item.ItemList` is empty at autorun time.** Use `roundStart` hook or `Timer.Wait(fn, 2000)`.
- **`#table` returns 0 on non-sequential (integer-keyed) tables.** Count manually with `pairs`.
- **`LuaUserData.RegisterType()` must be called before `CreateStatic()`.**
- **`LuaUserData.MakeFieldAccessible` crashes for mod-assembly types** — works only for vanilla
  Barotrauma types. Expose state via static getter/setter methods instead.

---

## RUNTIME ADDCOMPONENT — CLONE SAFETY

`ItemPrefab.Components` is `ImmutableArray` — cannot be mutated at runtime. Adding a component
to a live item without patching every spawn causes clone component-count mismatch errors.

**Pattern**: Register eligible prefab identifiers in a static `HashSet`. Patch the `Item`
constructor postfix to call `AddComponent` on every spawn, including clones and level-transition
reloads. This ensures injected components exist before the engine reloads saved wire/connection
state on level transition.

```csharp
internal static readonly HashSet<Identifier> PrefabsNeedingInjection = new();

[HarmonyPostfix]
static void Item_ctor_Postfix(Item __instance) {
    if (!PrefabsNeedingInjection.Contains(__instance.Prefab.Identifier)) return;
    if (__instance.GetComponent<MyComponent>() != null) return;
    InjectComponent(__instance);
}
```

---

## ITEM PREFAB PROPERTIES

- `item.Prefab.Linkable` — public bool, true when XML has `linkable="true"`. Items with
  `allowedLinks` set always also have `linkable="true"`, so `Linkable` alone is a complete filter.
- `ItemPrefab.allowedLinks` — private `ImmutableHashSet<Identifier>`. Patch via reflection:

```csharp
var field = typeof(ItemPrefab).GetField("allowedLinks",
    BindingFlags.NonPublic | BindingFlags.Instance);
var current = field.GetValue(prefab) as ImmutableHashSet<Identifier>
              ?? ImmutableHashSet<Identifier>.Empty;
field.SetValue(prefab, current.Add(new Identifier("targetidentifier")));
```

Track patched prefab identifiers in a static `HashSet` to avoid re-patching on every pass.

### Linkable items and component injection

Wire items (`GetComponent<Wire>() != null`) have `linkable="true"` in vanilla XML. If your
eligibility filter includes `item.Prefab.Linkable`, **always explicitly exclude wire items**:

```csharp
if (item.GetComponent<Wire>() != null) continue;
```

Failing to do this injects components into active wire items at construction time, breaking
vanilla wiring behavior (wires detach, cannot connect, fall to floor, etc.).

---

## LINKEDTO RULES

`item.linkedTo` is `List<MapEntity>`. Rules:

- **Never iterate from Lua** — will throw. Move all iteration to CS static helpers.
- Safe to temporarily remove/restore items within a single-threaded Harmony prefix+postfix pair.
- Always guard `AddLinked` with a duplicate check — vanilla editor links may already exist.
- `DisplaySideBySideWhenLinked`: assign **unconditionally** on both items every link cycle.
  Do not assume pre-existing XML values are correct.
- Never cache `Connection` object references across level transitions. The engine recreates
  `ConnectionPanel` components on reload; cached references become stale. Always use a fresh
  `panel.Connections.FirstOrDefault(...)` lookup.

---

## PERIODIC SPAWN / SETUP LOGIC

When spawning entities or performing setup that should happen once per round:

- **Fire a small number of times with delays** from `roundStart`, not on a repeating poll.
  Complex items have their relationships fully resolved later than simple items:
  ```lua
  Hook.Add("roundStart", "mymod.setup", function()
      Timer.Wait(setupFn, 2000)
      Timer.Wait(setupFn, 7000)
      Timer.Wait(setupFn, 15000)
  end)
  ```
- **Do NOT include setup logic in a repeating poll indefinitely.** If the setup undoes itself
  (e.g. a link is severed), a repeating poll will fight the undo continuously.
- Use a `_pendingSpawns` HashSet to prevent duplicate spawns between async callback cycles.
- Use a `processed` HashSet per-call (keyed on sorted entity ID pairs) to avoid processing the
  same pair from both directions in the same pass.

---

## CS ENFORCEMENT PATTERN

To require client-side CS from all connecting players:

**Client file** (runs only if CS is working):
```lua
if SERVER then return end
if Game.IsSingleplayer then return end
Networking.Send(Networking.Start("mymod.cs_confirmed"))
```

**Server file**:
```lua
if CLIENT then return end  -- on listen server CLIENT=true on host; guard skips enforcement there

local TIMER = 30
local clients = {}

for _, c in pairs(Client.ClientList) do clients[c] = { ok=true, t=0 } end

Networking.Receive("mymod.cs_confirmed", function(msg, client)
    if clients[client] then clients[client].ok = true end
end)

Hook.Add("client.connected",    "mymod.cs.join", function(c) clients[c] = { ok=false, t=0 } end)
Hook.Add("client.disconnected", "mymod.cs.left", function(c) clients[c] = nil end)

Hook.Add("think", "mymod.cs.think", function()
    for c, d in pairs(clients) do
        if not d.ok then
            d.t = d.t + 1/60
            if d.t > TIMER then
                clients[c] = nil
                c.Kick("Client-side CS is required for this mod.")
            end
        end
    end
end)
```

On a listen server, `SERVER` and `CLIENT` are both true on the host process. The
`if CLIENT then return end` guard means this script does not run on the host — the host is
never kicked regardless of their CS state.

---

## FABRICATOR INTERNALS

- `containers[0]` = inputContainer, `containers[1]` = outputContainer
- `StartFabricating(recipe, null, true)` — private, user=null is valid
- `CancelFabricating(null)` — private
- `CanBeFabricated(recipe, availableIngredients, null)` — private
- `RefreshAvailableIngredients` iterates `linkedTo`, calls `GetComponent<ItemContainer>()` which
  returns inputContainer (index 0) by default.
- Cross-fabricator ingredient sharing requires a Harmony prefix+postfix bracket: prefix removes
  linked fabricators from `linkedTo`, vanilla runs normally, postfix restores them and manually
  adds only their outputContainer contents.
- `dynamic` / `Cast<dynamic>()` are **not available** in Roslyn ACL (CS0656). Enumerate
  `ImmutableDictionary.Values` via `IEnumerable` + `as TargetType` cast instead.
- Neither `Fabricator` nor `Deconstructor` override `ItemComponent.Use()` — patch the base class.

---

## HARMONY PATTERNS

### Per-instance state in prefix/postfix pairs

```csharp
// WRONG — clobbered when multiple instances update in the same frame
private static float _saved;

// CORRECT
private static readonly Dictionary<MyType, float> _saved = new();
// prefix:  _saved[__instance] = current; current = 0;
// postfix: if (_saved.TryGetValue(__instance, out var v)) { restore v; _saved.Remove(__instance); }
// cleanup: hook Remove/Dispose to call _saved.Remove(instance) to prevent memory leaks
```

### AccessTools: field vs property

`AccessTools.Property` on a field returns null **silently** — no exception, math using the
result produces 0. Always verify in source whether a member is a field or property:

```csharp
// WRONG — silent null when target is a field
var p = AccessTools.Property(typeof(SomeClass), "SomeMember");
// CORRECT
var f = AccessTools.Field(typeof(SomeClass), "SomeMember");
```

---

## MULTIPLAYER GOTCHAS

- **Console commands route server-side.** All console commands execute on the server in MP.
  `DebugConsole.NewMessage` appears on server console only.
- **CS static fields reset on assembly reload; Lua globals persist.** Re-sync CS fields from Lua
  globals in `OnLoadCompleted` to avoid divergence between sessions.
- **Command double-registration.** If the old assembly hasn't unloaded when the new one loads,
  `OnLoadCompleted` fires twice. Always `RemoveAll` before re-adding:
  ```csharp
  DebugConsole.Commands.RemoveAll(c => c.Names.Any(n => n.Value == "MyCommand"));
  ```
- **Server physics overrides client-only patches.** Physics suppression patches must be in Shared
  or Server assemblies — client-only patches are overridden every frame by server sync.
- **`Entity.Spawner` may be null on load.** Guard before use.

---

## FILELIST.XML

```xml
<contentpackage name="MyMod" modversion="1.0.0" corepackage="False" gameversion="1.11.5.0">
  <Item file="%ModDir%/dummyitem.xml" />
  <Other file="%ModDir%/CSharp/RunConfig.xml" />
  <Other file="%ModDir%/CSharp/Shared/MyHelper.cs" />
  <Other file="%ModDir%/CSharp/Server/MyServer.cs" />
  <Other file="%ModDir%/CSharp/Client/MyClient.cs" />
  <Other file="%ModDir%/Lua/Autorun/init.lua" />
</contentpackage>
```

Use `<Other />` as the catch-all tag. Only use typed tags when the engine needs to process that
content type specifically (item inheritance, affliction overriding, particle registration, etc.).
A `dummyitem.xml` with just `<Items />` as root is required when the mod defines no real XML
content, to prevent engine load errors.

---

## RUNCONFIG.XML

```xml
<?xml version="1.0" encoding="utf-8"?>
<RunConfig>
  <!--Options: "None" / "Forced" (always) / "Standard" (when CS is enabled)-->
  <Server>Standard</Server>
  <Client>Standard</Client>
</RunConfig>
```

---

## DEFINITIONS TEMPLATE

Fill at project start. An agent reading this should reconstruct all persistent state without
running the game.

```
## Lua globals (source of truth)
name | type | default | purpose | owning file | when it changes

## CS static fields (mirrors — read via getter methods only)
ClassName.FieldName | type | default | Lua global it mirrors

## Network messages
"identifier" | direction | payload | when sent

## Console commands
CommandName | Shared/Client | what it does | broadcasts? | expected registration count

## Harmony patches
TypeName.MethodName | Prefix/Postfix/Both | Shared/Client/Server | purpose | gotchas
```
