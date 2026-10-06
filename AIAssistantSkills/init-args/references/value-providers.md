# Value providers

Value providers can be used to resolve dependencies dynamically at runtime, rather than using a fixed value. They have four use cases:

- Resolve a client's Init argument via an Initializer.
- Resolve a serialized field's value via `Any<T>`.
- Provide per-client (transient) services globally via `[Service]` (see global-services.md).
- Provide per-client (transient) services locally via `Services` component or `Service Tag` (see local-services.md).

## Types Of Value Providers

| Interface | Implement | Use Case |
|---|---|---|
| `IValueProvider<T>` | `T Value { get; }`; optionally `bool TryGetFor(Component client, out T value)` | Provides value of a specific type. Implement `TryGetFor` for client-specific values. |
| `IValueByTypeProvider` | `bool TryGetFor<TValue>(Component client, out TValue value)`; | Provides values of several types. E.g. GetComponent-style lookups. |
| `IValueProviderAsync<T>` | `Awaitable<T> GetForAsync(Component client, CancellationToken cancellationToken = default)` | Asynchonous variant of `IValueProvider<T>`, e.g. Addressables. |
| `IValueByTypeProviderAsync` | `Awaitable<TValue> GetForAsync<TValue>(Component client, CancellationToken cancellationToken = default)` | Asynchonous variant of `IValueByTypeProvider`. |

The `client` argument may be `null`. Value providers are often ScriptableObjects, but can be MonoBehaviours as well.

## Examples

```csharp
[CreateAssetMenu]
class MainCameraProvider : ScriptableObject, IValueProvider<Camera>
{
    public Camera Value => Camera.main;
}

[CreateAssetMenu]
class RandomName : ScriptableObject, IValueProvider<string>
{
    [SerializeField] string[] names = { };
    public string Value => names.Length == 0 ? null : names[Random.Range(0, names.Length)];
    public bool HasValueFor(Component client) => names.Length > 0;
}
```

## [ValueProviderMenu]

Put this attribute on a ScriptableObject or MonoBehaviour value provider to add it to the dropdown of every matching Init argument and `Any<T>` field.

```csharp
[ValueProviderMenu("Get From GameObject By Tag", Is.SceneObject), CreateAssetMenu]
class GetFromGameObjectByTag : ScriptableObject, IValueByTypeProvider
{
    [SerializeField] string tag;

    public bool TryGetFor<TValue>(Component client, out TValue value)
    {
        if (!client)
        {
            value = default;
            return false;
        }

        var gameObject = GameObject.FindWithTag(tag);
        return Find.In(gameObject, out value);
    }
}
```

- **Constructors:**
  - `(string itemName, Is whereAny, params Type[] isAny)`
  - `(string itemName, params Type[] isAny)`
  - `(Is whereAny, params Type[] isAny)`
  - `(params Type[] isAny)`
- **Properties:**
  - `ItemName`, `Tooltip`
  - `WhereAny`, `WhereAll`, `WhereNone` (all of type `Is`)
  - `IsAny`, `NotAny`, `Is`, `Not` (types)
  - `Order`: lower values appear higher; `1.2` means group 1, item 2.
- **`Is` flags** (combine with `|`): `Class`, `ValueType`, `Concrete`, `Abstract`, `Interface`, `BuiltIn`, `Component`, `WrappedObject`, `SceneObject`, `Asset`, `Collection`, `Service`, `BaseType`.
- **Shared vs. embedded.** A provider with no serialized fields and exactly one asset in the project is shared by reference. Otherwise each selection embeds its own instance.
- **Embedded providers in prefabs.** These can't be serialized inside prefab assets. Back them with a `[CreateAssetMenu]` asset, or use **Make Sub-Asset of Prefab**.
- **NUnit clash.** `Sisus.Init.Is` clashes with NUnit's `Is`. In tests, alias it: `using Is = NUnit.Framework.Is;`.

## Built-in value providers

- **Hierarchy/…**
  - **GetComponent**, **GetComponentInParent**, **GetComponentInChildren**
  - **GetComponents**, **GetComponentsInChildren**, **GetComponentsInParent**
  - **AddComponent**, **GetOrAddComponent**
  - **FindAnyObjectByType**
- **MainCamera**
- **WaitForService** (client waits indefinitely at runtime until the service becomes available)
- **LocalizedString** (Localization package)
- **Load Addressable/.**
  - **LoadAddressable**,
  - **LoadAddressableSprite**, **LoadAddressableAtlasedSprite**
  - **LoadAddressableTexture**, **LoadAddressableTexture2D**.
- **CrossSceneReference.** (allow dragging an Object reference across scene and prefab assets)
- **Prefab-instance reference.** Drag a **prefab asset** into the field and choose **Prefab Instance**. It resolves to the instance spawned at runtime.
- **If the target doesn't exist yet**, the client is disabled and waits for it.