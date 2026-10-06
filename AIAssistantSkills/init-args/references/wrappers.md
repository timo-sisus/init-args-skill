# Wrappers (optional)

> Prefer **`MonoBehaviour<T…>`**. Use wrappers only when the user or codebase favors plain C# classes, for constructor injection, immutability, or Unity-free tests.
>
> Wrappers come with costs:
> - an extra class per wrapped type;
> - more concepts to learn;
> - no support for circular dependency injection.
>
> Don't use both styles for the same object.

A `Wrapper<T>` is a component that holds a plain C# object and forwards Unity events to it.

```csharp
public class Player : IUpdate                    // plain C# class
{
    readonly IInputManager input;
    public Player(IInputManager input) => this.input = input;
    public void Update(float deltaTime) { /* … */ }
}

[AddComponentMenu("Wrapper/Player")]
class PlayerComponent : Wrapper<Player> { }      // or: script context menu > Generate Wrapper
```

## Supplying the wrapped object

There are four ways. If you use none of them, the wrapped object is null.

1. Mark the class `[Serializable]`. Unity then creates and serializes it.
2. Give the wrapper a parameterless constructor: `public PlayerComponent() : base(new Player(...)) { }`.
3. Pass the object in code: `gameObject.AddComponent<PlayerComponent, Player>(player)`.
4. Use a **WrapperInitializer** (1–6 arguments), whose arguments are services or set in the Inspector:

```csharp
public class PlayerInitializer : WrapperInitializer<PlayerComponent, Player, IInputManager>
{
    protected override Player CreateWrappedObject(IInputManager input) => new Player(input);
}
```

## Unity events and coroutines

The wrapped object receives Unity events by implementing these interfaces:

- `IAwake`, `IOnEnable`, `IStart`
- `IUpdate`, `IFixedUpdate`, `ILateUpdate` (each takes `float deltaTime`)
- `IOnDisable`, `IOnDestroy`
- `IDisposable` or `IAsyncDisposable`, called on destroy when `IOnDestroy` isn't implemented

For coroutines, implement `ICoroutines`. It exposes `ICoroutineRunner CoroutineRunner { get; set; }`, which the wrapper assigns. The coroutines are tied to the GameObject's lifetime.

## As services

- To make the wrapped object a service, put a Service Tag on the wrapper component, or drag the wrapper into a `Services` component.
- Clients receive the **wrapped object**, not the component.
- To look wrapped objects up, use `Find.WrappedObject<Player>()` and `Find.WrapperOf(player)`. `Player p = playerComponent;` also works, through an implicit conversion.

## ScriptableWrapper<T>

`ScriptableWrapper<T>` is the ScriptableObject equivalent, useful for settings assets:

```csharp
[CreateAssetMenu]
public class SettingsAsset : ScriptableWrapper<Settings> { }

Settings settings = settingsAsset.WrappedObject;
```

It forwards Awake, OnEnable, Update, FixedUpdate, LateUpdate, OnDisable and OnDestroy. It does **not** forward `IStart` or `ICoroutines`.
