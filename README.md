# Helper

Lightweight utility framework for Roblox/Luau. State management, rate
limiting, and deterministic cleanup in a modular, strongly-typed API.

Built for gameplay systems that need predictable state, resource lifetime,
and server-side request control without unnecessary dependencies.

## Install

**Model** — download `Helper.rbxm` from [Releases](https://github.com/your-user/helper/releases)
and drag it into `ReplicatedStorage`.

**Wally**

```toml
[dependencies]
Helper = "jmhfields3-star/helper@1.0.0"
```

## Structure

Helper is split into focused modules while remaining a single package.

```text
Helper
├── CleanupManager
├── RateLimiter
└── StateManager
```

Each system can be required independently when needed.

## Quick start

### State

```lua
local StateManager = require(Helper.StateManager)

StateManager.Set(character, "Blocking", true)

if StateManager.Has(character, "Blocking") then
    -- ...
end

StateManager.Apply(character, "Stunned", true, 1.5)

print(StateManager.TimeLeft(character, "Stunned"))

StateManager.Clear(character, "Blocking")
```

Timed states automatically expire and re-applying a state refreshes its
expiration.

### Stat modifiers

```lua
StateManager.AddModifier(
    character,
    "WalkSpeed",
    "Slowed",
    0.5,
    "Multiply",
    0,
    2
)
```

Modifiers can stack without systems fighting over the underlying value.

```text
Base
  ↓
Add modifiers
  ↓
Multiply modifiers
  ↓
Highest-priority Set modifier
  ↓
Final value
```

Built-in Humanoid stats such as `WalkSpeed`, `JumpPower`, `JumpHeight`,
`HipHeight`, and `MaxSlopeAngle` can be modified automatically.

Custom stats can also be defined:

```lua
StateManager.DefineStat(character, "Damage", 10, function(value)
    -- apply value
end)
```

### Rate limiting

```lua
local limiter = RateLimiter.new({
    Rate = 5,
    Per = 1,
    Burst = 8,
})

local allowed, retryAfter = limiter:Check(player)

if not allowed then
    return
end
```

The limiter uses a token-bucket model, allowing sustained rates while still
supporting controlled bursts.

Costs can be specified per request:

```lua
limiter:Check(player, 3)
```

For simple cooldowns:

```lua
local cooldown = RateLimiter.Cooldown(2)

if cooldown:Check(player) then
    -- allowed once every 2 seconds
end
```

Callbacks can also be wrapped:

```lua
remote.OnServerEvent:Connect(
    limiter:Wrap(function(player, ...)
        -- only runs when allowed
    end)
)
```

Player buckets are automatically released when the player leaves, and idle
buckets are periodically pruned.

### Cleanup

```lua
local cleanup = CleanupManager.new()

cleanup:Add(part)
cleanup:Connect(humanoid.Died, onDied)

cleanup:Set("Effect", emitter)

cleanup:Clean()
```

Cleanup supports:

* `Instance` → `:Destroy()`
* `RBXScriptConnection` → `:Disconnect()`
* `thread` → `task.cancel()`
* `function` → called
* objects implementing `Destroy`
* objects implementing `Disconnect`
* objects implementing `Clean`

Tasks are released last-in-first-out.

Keyed resources automatically clean their previous value:

```lua
cleanup:Set("Effect", firstEffect)

cleanup:Set("Effect", secondEffect)
-- firstEffect is cleaned automatically
```

For instance-owned resources:

```lua
local cleanup = CleanupManager.For(character)

cleanup:Add(connection)
cleanup:Add(effect)
```

The manager automatically destroys itself when the instance is removed.

## API

### `StateManager`

| Call                                                  | Returns             |
| ----------------------------------------------------- | ------------------- |
| `StateManager.Set(entity, key, value)`                |                     |
| `StateManager.Get(entity, key, default?)`             | `any`               |
| `StateManager.Has(entity, key)`                       | `boolean`           |
| `StateManager.Clear(entity, key)`                     |                     |
| `StateManager.Apply(entity, key, value?, duration)`   |                     |
| `StateManager.Extend(entity, key, value?, duration)`  |                     |
| `StateManager.TimeLeft(entity, key)`                  | `number`            |
| `StateManager.GetAll(entity)`                         | `{ [string]: any }` |
| `StateManager.Observe(entity, key, callback)`         | `() -> ()`          |
| `StateManager.DefineStat(entity, name, base, apply?)` |                     |
| `StateManager.AddModifier(...)`                       |                     |
| `StateManager.RemoveModifier(...)`                    |                     |
| `StateManager.HasModifier(...)`                       | `boolean`           |
| `StateManager.ResetStat(entity, name)`                |                     |
| `StateManager.SetBase(entity, name, base)`            |                     |
| `StateManager.GetStat(entity, name)`                  | `number?`           |
| `StateManager.ClearAll(entity)`                       |                     |

### `RateLimiter`

| Call                            | Returns           |
| ------------------------------- | ----------------- |
| `RateLimiter.new(config)`       | `RateLimiter`     |
| `RateLimiter.Cooldown(seconds)` | `RateLimiter`     |
| `limiter:Check(key, cost?)`     | `boolean, number` |
| `limiter:Peek(key)`             | `number`          |
| `limiter:Reset(key)`            |                   |
| `limiter:Clear()`               |                   |
| `limiter:Wrap(callback, cost?)` | function          |
| `limiter:Destroy()`             |                   |

### `CleanupManager`

| Call                                | Returns               |
| ----------------------------------- | --------------------- |
| `CleanupManager.new()`              | `CleanupManager`      |
| `CleanupManager.For(instance)`      | `CleanupManager`      |
| `cleanup:Add(task)`                 | task                  |
| `cleanup:Connect(signal, callback)` | `RBXScriptConnection` |
| `cleanup:Remove(task)`              |                       |
| `cleanup:Release(task)`             |                       |
| `cleanup:Set(key, task?)`           | task?                 |
| `cleanup:Get(key)`                  | task?                 |
| `cleanup:Clean()`                   |                       |
| `cleanup:Destroy()`                 |                       |
| `cleanup:IsDestroyed()`             | `boolean`             |
| `cleanup:LinkToInstance(instance)`  | `CleanupManager`      |

## Design

Helper is designed around three principles:

**State**

Gameplay state and modifiers should have a single source of truth instead
of multiple systems directly modifying the same properties.

**Limits**

Network-facing actions should be easy to control with reusable,
server-side rate limiters.

**Lifetime**

Anything that creates a connection, instance, task, or other disposable
resource should have an explicit owner and cleanup path.

## Performance

Helper avoids permanent update loops for state, rate limiting, and cleanup.

Rate limiter buckets are lazily created and periodically pruned when they
return to their full capacity.

State entries are created only when required and automatically released for
Instance entities when they are destroyed or removed.

Cleanup managers release their owned resources in a controlled order and
continue cleaning even if an individual task fails.

## Server Security

`RateLimiter` is intended to be used at server boundaries such as:

* `RemoteEvent` handlers
* ability requests
* combat requests
* interaction requests
* inventory actions
* purchase requests
* matchmaking requests

Client-side checks should never be treated as authoritative security.

## Requirements

* Roblox
* Luau
* `--!strict` compatible

No external dependencies beyond Roblox APIs.

## License

MIT — see [LICENSE](LICENSE).
