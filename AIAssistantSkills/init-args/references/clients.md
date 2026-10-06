# Clients

## MonoBehaviour<T…>

Derive from `MonoBehaviour<T…>` and implement `Init` to receive dependencies.

Methods are always executed in this order:

1. `Init`
2. `OnAwake` (instead of `Awake`)
3. `OnEnable`
4. `Start`

```csharp
class Player : MonoBehaviour<IInputManager, Camera>
{
    IInputManager inputManager;
    Camera camera;

    protected override void Init(IInputManager inputManager, Camera camera)
    {
        this.inputManager = inputManager;
        this.camera = camera;
    }

    protected override void OnAwake() => inputManager.Enable();
    protected override void OnReset() { /* editor Reset logic */ }
}
```

### Where arguments come from (highest priority first)
1. Arguments passed in code (`AddComponent`/`Instantiate`).
2. An attached Initializer.
3. Services the client can reach, local before global. These are only used if **all** of the parameters match defining types of services.

### What happens when services are missing at load.
- By default the component is disabled until all their dependencies become available. See `[Init]` in SKILL.md.

- **Add-on base classes** are available as `.unitypackage` files in `Add-Ons/`:
  - `NetworkBehaviour<T…>` for Netcode for GameObjects, FishNet and PurrNet;
  - `SerializedMonoBehaviour<T…>` for Odin.

## Optional Init Arguments

If a client can optionally receive dependencies through its Init method at runtime, but can also function without this, then `[Init(Enabled = false)]` can be used to hide the Init section in the Inspector and disable validation against missing dependencies.

The `bool Init(Context context)` method can also optionally be overriden to assign / validate fallback services, and to make the Service Debugger show the client as `Initialized`.

```csharp
[Init(Enabled = false)]
class MyClient : MonoBehaviour<Collider>
{
    [SerializeField] Collider collider;

    protected override void Init(Collider collider) => this.collider = collider;

    protected override bool Init(Context context)
    {
        // Use global / local services, AddComponent / Instantiate arguments, if provided
        if(base.Init(context))
        {
            return true;
        }

        // Otherwise, fallback to defaults
        if(!collider)
        {
            collider = GetComponent<Collider>();
            ValidateArgument(collider);
        }

        // Mark as Initialized.
        return true;
    }
}
```

*Note: Optional init arguments can be useful for making components that use serialized fields or GetComponent while still being unit testable.*

## ScriptableObject<T…>

- Override `Init`; use `OnAwake`/`OnReset` instead of `Awake`/`Reset`.
- Create with `Create.Instance<TSO, TArg>(arg)`, or clone with `asset.Instantiate(arg)`.


## Custom base classes and inheritance

Any class can be a client by implementing `IInitializable<T…>` (the `Init(...)` method) and pulling its arguments in `Awake`. `InitArgs.TryGet` uses arguments passed via `AddComponent`/`Instantiate` if any, and otherwise resolves services:

```csharp
public abstract class BaseBehaviour : MonoBehaviour, IInitializable<ILogger>
{
    protected ILogger Logger { get; private set; }
    public void Init(ILogger logger) => Logger = logger;
    protected virtual void Awake() => InitArgs.TryGet<BaseBehaviour, ILogger>(this);
}
```

When a derived class needs more arguments than its `MonoBehaviour<A, B>` base, implement the larger interface, initialize the base, and override `Init(Context)`:

```csharp
public class Derived : Base, IInitializable<A, B, C>
{
    C c;
    public void Init(A a, B b, C c) { Init(a, b); this.c = c; }
    protected override bool Init(Context context) => InitArgs.TryGet<Derived, A, B, C>(this);
}
```

For whole families of classes, generate custom generic base classes with **Create > Init(args) > Base Class Generator**. The base class needs a `protected virtual void Awake()`.

## Creating clients in code

These APIs work with `MonoBehaviour<T…>`, or any class implementing `IArgs<T…>` + `IInitializable<T…>`. Arguments passed this way override services and Initializers.

```csharp
// AddComponent with arguments (1–12)
Player player = gameObject.AddComponent<Player, IInputManager, Camera>(inputManager, camera);
gameObject.AddComponent(out Player p2, (IInputManager)inputManager, camera); // inferred: types must match Init params EXACTLY (cast if needed)

// Instantiate with arguments (components and ScriptableObjects; parent/position/rotation overloads exist)
Player clone = prefab.Instantiate(inputManager, camera);

// ScriptableObject<T…>
var settings = Create.Instance<GameSettings, Difficulty>(difficulty);

// GameObject builder: up to 3 components; call Init1/Init2/Init3 in declaration order, even when empty
Player a = new GameObject<Player>("Player").Init(inputManager, camera);
Player b = new GameObject<Collider, Player>("Player").Init1().Init2(inputManager, First.Component);
```

- **Unreceived arguments throw.** If the component never takes the arguments, `InitArgumentsNotReceivedException` is thrown.
- **Late delivery.** Components that implement only `IInitializable<T…>`, without `IArgs<T…>` or `InitArgs.TryGet`, receive the arguments **after** `Awake`/`OnEnable`.