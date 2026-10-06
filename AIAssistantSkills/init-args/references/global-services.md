# Global Services

Documents how to register Global Services using the Init(args) DI framework.

## 1. The `[Service]` Attribute

Define a global service by adding the `[Service]` attribute to a class. Init(args) automatically initializes and caches a single instance of the class when the project starts and injects it to clients that request it.

```csharp
[Service]
class GameManager { }
```

### Key Features
*   **Init Order:** Non-lazy services are initialized in optimal order by dependencies *before* any objects in the initial scene become active. They are ready to be used in `Awake` and `OnEnable`.
*   **Component Injection:** Automatically injected into components deriving from `MonoBehaviour<T...>` where a generic type argument matches a service's defining type.
*   **Service-to-Service Injection:** Delivered to other global services via constructor injection or by implementing `IInitializable<T...>`.

### Defining Types
Services can be registered using an interface or base type to decouple clients from concrete implementations.

```csharp
public interface IInputManager { }

// Provided to clients requesting IInputManager
[Service(typeof(IInputManager))]
class KeyboardInputManager : IInputManager { }
```
*Note: Multiple types can be specified in the constructor, and preprocessor directives (`#if UNITY_STANDALONE`) can be used for platform-specific implementations.*

### Additional Properties

The `[Service]` attribute has several properties to control how the service gets initialized:

| Property | Effect |
|---|---|
| `ResourcePath = "Path"` | Load from Resources. |
| `AddressableKey = "Key"` | Load from Addressables. Requires the Addressables package. |
| `FindFromScene = true` | Find the instance in the initial scene. Implies `LazyInit`. |
| `LoadScene = "Services"` (or a build index) | Load the scene additively if needed and take the service from it. |
| `Instantiate = true/false` | By default prefabs are cloned and ScriptableObjects are used as-is; this overrides that. |
| `LazyInit = true` | Create the service when the first client needs it. |
| `LoadAsync = true` | Load or instantiate asynchronously. Clients that depend on it should wait (see `[Init(WaitForServices = true)]`). |
| `DontDestroyOnLoad` | True by default for created services. Instances found in a scene stay in their scene. |
| `World = "Gameplay"` | ECS only: register a `SystemBase` or `World` as a service. |

```csharp
[Service(AddressableKey = "IconDatabase", LoadAsync = true)] class IconDatabase : ScriptableObject { }
[Service(LoadScene = "PreloadScene")] class AudioManager : MonoBehaviour { }
[Service(FindFromScene = true)] class Player : MonoBehaviour { }
```

*Note: Only one instance of each prefab gets instantiated if multiple global service share the same `ResourcePath` of `AddressableKey`.*

### Lifetime Events

Non-MonoBehaviour services can implement these interfaces to receive Unity events:

- `IAwake`, `IOnEnable` - Executed after all non-lazy services have been created they have received their dependencies. `Awake`, then every `OnEnable`, then every `Start`.
- `IAwake`, `IOnEnable` and `IStart` run in phases. Every service's `Init` is called first, then every `Awake`, then every `OnEnable`, then every `Start`.
- `IUpdate`, `IFixedUpdate` and `ILateUpdate` take `float deltaTime`.
- `IOnDisable`, `IOnDestroy` and `IDisposable` are called when the application quits or Play Mode exits.

---

## 2. Third-Party Services

When you cannot add the `[Service]` attribute directly to a class (e.g., it belongs to a third-party package or Unity itself), use a **Service Initializer**.

### Automatic Initialization
Derive from `ServiceInitializer<T...>` and add the `[Service]` attribute to the initializer class. 

```csharp
[Service(typeof(SomeService))]
class SomeServiceInitializer : ServiceInitializer<SomeService> { }
```

### Custom Initialization (e.g., Singletons)
Override the `InitTarget` method to control exactly how the service is acquired or created.

```csharp
[Service(typeof(SomeSingleton), LazyInit = true)]
class SomeSingletonInitializer : ServiceInitializer<SomeSingleton> 
{
    public override SomeSingleton InitTarget() => SomeSingleton.Instance; 
}
```

### Injecting Dependencies

List other global services as generic arguments to use them when initializing the service.

```csharp
[Service(typeof(SomeService))]
class SomeServiceInitializer : ServiceInitializer<SomeService, SomeDependency>
{
    public override SomeService InitTarget(SomeDependency dependency) => new SomeService(dependency);
}
```

### Initializer Assets

Make the initializer derive from `ScriptableObject` and implement `IServiceInitializer<T...>` to use Inspector-assigned serialized fields.

```csharp
[Service(typeof(IPlayer), ResourcePath = "PlayerInitializer"), CreateAssetMenu]
class PlayerInitializer : ScriptableObject, IServiceInitializer<Player>
{
   [SerializeField] Any<string> name;
   [SerializeField] Any<Color> color;

   public Player InitTarget() => new Player(name, color);
}
```

### Asynchronous Initialization
If the third-party service requires async loading, derive from `ServiceInitializerAsync<T...>`.

```csharp
[Service(typeof(SomeService))]
class SomeServiceInitializer : ServiceInitializerAsync<SomeService>
{
    public override async Task<SomeService> InitTargetAsync(CancellationToken cancellationToken)
    {
        var loadService = Resources.LoadAsync<GameObject>("SomeService");
        await loadService;
        return ((GameObject)loadService.asset).GetComponentInChildren<SomeService>();
    }
}
```

---

## 3. Per-Client Services

Value providers (`IValueProvider<T>`, `IValueByTypeProvider`, `IValueProviderAsync<T>IValueProvider<T> or `IValueByTypeProviderAsync`) can be used to provide a unique instance to every client that requests them (also known as transient services).

```csharp
[Service(typeof(ILogger))]
class LoggerProvider : IValueProvider<Logger>
{
    public Logger Value = new Logger(context: null);

    public bool TryGetFor(Component client, out Logger logger)
    {
        logger = new Logger(context: client);
        return true;
    }
}
```

---

## 4. Generic Services

Init(args) supports registering types with generic type parameters as global services.

```csharp
[Service]
public class Logger<T>
{
    public void Info(string message) => Debug.Log(typeof(T).Name + " - " + message);
}
```

### Key Behaviors of Generic Services
*   **Lazy Instantiation:** Generic services are *never* created automatically at startup. They are created lazily only when a client requests them.
*   **Unique Instances:** A unique global service instance is created for *each unique combination* of generic type arguments (e.g., `Logger<Player>` and `Logger<Enemy>` will be separate instances).
*   **Interface Registration:** You can register generic interfaces using the open generic type: `[Service(typeof(ILogger<>))]`.

### Third-Party Generic Services
You can combine Service Initializers with Generic Services by defining an initializer with a generic type parameter. Init(args) will dynamically determine the generic type argument based on the client's request.

```csharp
[Service(typeof(ILogger<>))]
sealed class LoggerInitializer<T> : ServiceInitializer<ILogger<T>>
{
    public override ILogger<T> InitTarget()
    {
        var factory = LoggerFactory.Create();
        return factory.CreateLogger<T>();
    }
}
```