## Any\<T\>

`Any<T>` is a serialized field type that can hold any of these:
- A reference to an `UnityEngine.Object` assignable to `T`.
- An plain old C# object assignable to `T`.
- A value provider that can provide a value of type `T` (including a cross-scene reference).
- Global or local service with the defining type `T`.

```csharp
class Interactor : MonoBehaviour
{
    [SerializeField] Any<IInteractable> target;

    IInteractable Target => target.GetValue(client: this);
}
```

Use `GetValue(client)` rather than `.Value`. Only `GetValue(client)` resolves local services and client-specific values from value providers.