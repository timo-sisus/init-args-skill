---
name: init-args
description: Using Init(args) for initializing services and injecting them to clients. Use when creating a component that has any dependencies to other non-static classes, when creating unit tests for one, and when the user mentions Init(args), dependency injection (or DI) clients or services.
---

# Init(args)

Init(args) is a DI framework for Unity that let's you replace Singletons and normal serialized fields with more flexible dependency injection that gets rid of hidden dependencies, and adds full support for interfaces, unit-testing and cross-scene references.

## MonoBehaviour\<T...\>

Components declare their dependencies as generic type arguments and receive them via their `Init` method.

## Where dependencies come from
- Global services (`[Service]` attribute).
- Local services (Service Tag, `Services` component).
- Initializers (per-instance Inspector configuration).
- code (`AddComponent`/`Instantiate` with arguments).

All of the API lives in the `Sisus.Init` namespace.

**Docs**: https://docs.sisus.co/init-args/

## Pick the right tool

| Need | Use | Details |
|---|---|---|
| A component that receives dependencies | `MonoBehaviour<T…>` | [clients.md](references/clients.md) |
| One shared instance for all clients | `[Service]` on the class | [global-services.md](references/global-services.md) |
| A new instance per client | `[Service]` on an `IValueProvider<T>` | [global-services.md](references/global-services.md) |
| Third-party service or custom initialization | `[Service]` on a `ServiceInitializer<TService, T…>` | [global-services.md](references/global-services.md) |
| A service visible only to part of a scene or prefab | Service Tag, `Services` component, `Service.AddFor` | [local-services.md](references/local-services.md) |
| Per-instance Inspector configuration of Init arguments | `Initializer<TClient, T…>` | [initializers.md](references/initializers.md) |
| Dynamic arguments (localized, randomized, async loaded) | Value Providers | [value-providers.md](references/value-providers.md) |
| Inject test doubles in unit tests | `gameObject.AddComponent<TClient, T…>(fakes)` | [testing.md](references/testing.md) |
| More flexible serialized fields outside of `MonoBehaviour<T…>` | `[SerializeField] Any<T>` | [any.md](references/any.md) |
| Plain C# classes attached to GameObjects | `Wrapper<T>` | [wrappers.md](references/wrappers.md) |

*Note: Prefer `MonoBehaviour<T…>` over Wrappers by default. Use wrappers only when it's an established pattern in the codebase or the user asks to use them.*

## The core pattern

```csharp
using Sisus.Init;
using UnityEngine;

[Service(typeof(IInputManager))]
class InputManager : IInputManager { }

class Player : MonoBehaviour<IInputManager, Camera>
{
    IInputManager inputManager;
    Camera camera;

    protected override void Init(IInputManager inputManager, Camera camera)
    {
        this.inputManager = inputManager;
        this.camera = camera;
    }

    protected override void OnAwake()
    {
        // Dependencies can be used here
    }
}
```

Because `Camera` is not a global service, it either needs to be assigned using an Initializer that is attached to `Player` in its scene/prefab, or by registering `Camera` as a local service in the scene/prefab that contains it.

```csharp
class PlayerInitializer : Initializer<Player, IInputManager, Camera> { }
```

In code, pass the arguments directly.

```csharp
var player = gameObject.AddComponent<Player, IInputManager, Camera>(inputManager, camera)
```

## The `[Init]` attribute

The `[Init]` attribute can be used to configure how a client reacts to missing dependencies.

```csharp
// Dependencies are optional (e.g. only passed in unit tests via AddComponent).
[Init(Enabled = false)]
class Player : MonoBehaviour<ILogger> { … }

// Some services only become available at runtime (e.g. local services in other scenes / prefabs).
[Init(WaitForServices = true)]
class Hud : MonoBehaviour<Player> { … }
```

## Rules that prevent most bugs

1. **Never declare `void Awake()` or `void Reset()` in a `MonoBehaviour<T…>`.** They hide the base methods, so injection never happens. Override `OnAwake()` / `OnReset()` instead.
2. **Keep `Init` to field assignment only.** In some cases it can get executed on inactive GameObjects or in Edit Mode. Put other logic in `OnAwake`.
3. **Automatic injection only happens when *every* Init argument is a service** the client can reach. Otherwise use an Initializer.
4. **`[Service]` with no arguments registers only the concrete type.** List interfaces explicitly, e.g. `[Service(typeof(IInputManager))]`.
5. **Compare interface-typed variables against `Null`, not `null`.** E.g. `if (inputManager == Null)`, so destroyed Unity Objects are detected. If the class does not derive from `MonoBehaviour<T…>`, add `using static Sisus.NullExtensions;`.
7. **Classes with `[Service]` must be in an assembly that references `InitArgs.Services`.** Clients reference `InitArgs`.

## Troubleshooting a client that gets no dependencies

1. Is any argument not a service? Attach an Initializer.
2. Does the defining type of the `[Service]` match the Init parameter type?
3. Is there a `void Awake()` in the class?
4. Is a local service out of range? Check the Service Tag availability, or the `Services` component's **For Clients** field.
5. Is a wrapped object null? See [wrappers.md](references/wrappers.md).
