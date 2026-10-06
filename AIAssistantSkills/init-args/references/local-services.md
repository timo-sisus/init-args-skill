# Local services

A local services are services that are loaded as part of a scene or a prefab, and are only available to clients within a specified radius, and only while the GameObject that contains the local service remains active.

Local services have priority over global services, so they can override global services for all clients within their radius (e.g. a prefab instance's child hierarchy).

## Availability

The `Clients` enum specifies the options for local service availability.

| Value | Visible to clients… |
|---|---|
| `InGameObject` | on the same GameObject |
| `InChildren` | on the same GameObject and all of its children |
| `InParents` | on the same GameObject and all of its parents |
| `InHierarchyRootChildren` | anywhere under the same root |
| `InScene` | in the same scene |
| `InAllScenes` | in any scene |
| `Everywhere` | anywhere, including outside scenes; this acts as a global service |

## Service Tag (Scene and Root Components)

1. Open the component's context menu.
2. Choose **Make Service Of Type…** and pick one or more defining types.
3. Right-click the tag icon in the component header and choose **Set Availability…** to set the `Clients` range.

Service Tags are best suited for registering components from scenes and the root GameObject of prefabs.

## `Services` component (Assets and Child Components)

1. Attach the **Services** component to a GameObject.
2. For each **Provides Services** entry, drag in an Object and pick its defining type.
3. Set **For Clients** to the desired availability, relative to this GameObject.

- **What you can drag in directly:** components and ScriptableObject assets.
- **Plain old C# objects:** it's recommended to create a Wrapper for the object and to drag the **Wrapper** in.
- **Per-client (transient) services**: drag in a **value provider**.
- **Complex prefabs:** put one `Services` component on the root, drag in services from its children, and set **For Clients** to `InChildren`.

## In code

```csharp
public class Tooltip : MonoBehaviour
{
    void OnMouseEnter() => Service.AddFor(Clients.InScene, this);
    void OnMouseExit() => Service.RemoveFrom(Clients.InScene, this);
}
```

- Availability is measured from the `registerer` component. Pass the same registerer to `RemoveFrom`. The method is `RemoveFrom`; there is no `RemoveFor`.
- You can pass an `IValueProvider<T>`, `IValueProviderAsync<T>` or `IValueByTypeProvider` instead of an instance. The provider is resolved **per client**, which makes the local service transient.
- Prefer `Service Tag` and `Services` component over manual registration in code when possible for better Inspector visualization and Edit Mode validation of clients' dependencies.

## Best practices

- **Optimally keep a local service within its own scene or prefab.** Clients then initialize synchronously, and the Edit Mode missing dependency warnings stay accurate.
- **Service in another scene or prefab:** make the client wait by adding `[Init(WaitForServices = true)]` to its class or by selecting **Wait For Service** for the argument in the client's Initializer.
- **Cross-scene references** can also be used instead of local services when clients and services exist in different scenes / prefabs.