# Initializers

An Initializer is a component attached to a client component that lets you configure each Init argument in the Inspector. It injects the arguments to the client's `Init` method before its `Awake` and then removes itself.

Use an Initializer when:
- Not all arguments are services.
- A service should be overridden for one particular client.
- You need to use a value provider, a cross-scene reference or a prefab-instance reference.
- You need to pick the concrete type of an interface type argument and/or configure its serialized state using the Inspector.

Service-typed arguments default to the service automatically, so only the rest have to be provided using the Inspector.

## Creating an Initializer

- **In the editor:** click **+** in the client's Init section in the Inspector, then **Generate Initializer**. This creates `<Client>Initializer.cs` next to the client script. For components without an Init section (built-in or third-party), use **Generate Initializer…** in the component's context menu.
- **By hand:** the first generic argument is the client type, followed by the Init parameter types in order (1–12 arguments):

```csharp
class PlayerInitializer : Initializer<Player, IInputManager, Camera> { }
```

- **What each argument field can hold:** Objects, `SerializeReference` values (including interface implementations), value providers, cross-scene or prefab-instance references, or **Wait For Service**.
- **Wait For Service:** the client waits for that service to be registered at runtime, and the Edit Mode warning is suppressed.

## Initializers and PropertyAttributes

To control how an initializer's Init arguments are drawn, declare a nested `Init` class inside the Initializer class, define one field per each Init argument. Any PropertyAttributes placed on the fields will get reflected in the Inspector:

```csharp
public class PlayerInitializer : Initializer<Player, IInputManager, float>
{
    #if UNITY_EDITOR
    private class Init
    {
        public IInputManager inputManager;
        [Range(0f, 100f)] public float speed;
    }
    #endif
}
```

## Full control: InitializerBase<TClient, T…>

`InitializerBase<…>` gives you full control over how Init arguments are serialized / resolved. Use it for types that `SerializeReference` can't handle. You need to implement the `*Argument` properties for each argument (`Argument` / `FirstArgument`, `SecondArgument` ...).

```csharp
class EnemyInitializer : InitializerBase<Enemy, Transform>
{
    [SerializeField] Any<Player[]> players;

    protected override Transform Argument
    {
        get => players.GetValue(this)
            .OrderBy(p => Vector3.Distance(p.transform.position, transform.position))
            .FirstOrDefault()?.transform;
        set { }
    }
}
```

## Wrapper initializers

Wrappers use `WrapperInitializer<TWrapper, TWrapped, T…>`; see wrappers.md.
