# Unit testing

During unit tests test doubles can be injected to clients directly via `AddComponent`, removing the need to create test prefabs or scenes and configure them using the Inspector. Passing dependencies via `AddComponent works both in Edit Mode and Play Mode.

## AddComponent with arguments

```csharp
using NUnit.Framework;
using Sisus.Init;
using UnityEngine;
using Is = NUnit.Framework.Is;   // avoid clash with Sisus.Init.Is

public class PlayerTests
{
    GameObject gameObject;

    [SetUp] public void SetUp() => gameObject = new GameObject("Test");
    [TearDown] public void TearDown() => Object.DestroyImmediate(gameObject);

    [Test]
    public void Player_Uses_Injected_InputManager()
    {
        var inputManager = new FakeInputManager { MoveInput = Vector2.right };
        var camera = gameObject.AddComponent<Camera>();

        var player = gameObject.AddComponent<Player, IInputManager, Camera>(inputManager, camera);

        Assert.That(player.InputManager, Is.SameAs(inputManager));
    }

    class FakeInputManager : IInputManager
    {
        public Vector2 MoveInput { get; set; }
    }
}
```

## new GameObject<T...>

`new GameObject<T...>` can also be used to create a GameObject and initialize one or more components in a single line.

```csharp
[Test]
public void Player_Uses_Injected_InputManager()
{
    var inputManager = new FakeInputManager { MoveInput = Vector2.right };

    Player player = new GameObject<Player, Camera>().Init1(inputManager, Second.Component);

    Assert.That(player.InputManager, Is.SameAs(inputManager));

    Object.DestroyImmediate(player.gameObject);
}
```

## Testable

`Testable` (in the `Sisus.Init.Testing` namespace) can also be used to wrap a `GameObject` and execute Unity event methods in Edit Mode.
The GameObject it wraps can be easily cleaned up after the test is done with the `using` statement.

```csharp
[Test]
public void PlayerMoves_InDirectionOfMoveInput_EveryFrame()
{
    var inputManager = new FakeInputManager();

    using var testable = new Testable(new GameObject<Player, Camera>().Init1(inputManager, Second.Component), invokeAllInitMethods: true);

    inputManager.MoveInput = Vector2.right;

    testable.Update(); // Execute Update.

    Assert.That(testable.transform.position.x, Is.GreaterThan(0f));
}
```

## What runs when

- **`Init`** gets executed before `AddComponent` returns, whether the GameObject is active or inactive, and in both Edit Mode and Play Mode tests.
- **Unity event methods** like `OnAwake`, `OnEnable` and `Start` don't get executed in Edit Mode by default, but can be executed using `Testable`.
- **Use Play Mode tests for lifecycle behavior**, i.e. anything in `OnAwake`, `OnEnable`, `Start`, `Update` or coroutines.

## Injecting global services

If a Play Mode test requires instantiating a prefab or loading a scene containing several clients, their services can be configured via the `Service` API, rather than being passed individually via `AddComponent` / `Instantiate`.

```csharp
[SetUp] public void SetUp() => Service.Set<IInputManager>(fakeInputManager);
[TearDown] public void TearDown() => Service.Unset<IInputManager>(fakeInputManager);
```

## Tips

- **Interfaces can improve unit-testability** (`IInputManager`, `ILogger`, `ITime`), enabling test-doubles to be used.
- **Mark test-only dependencies optional.** If dependencies are only ever passed in tests, mark the client `[Init(Enabled = false)]`. Scenes then show no missing-argument warnings, and the client doesn't wait for services.
- **Wrapped plain C# classes can be tested without any GameObject.** with simple manual constructor injection.
- **Assembly references:**
  - Test assemblies should reference `InitArgs` to pass arguments via `AddComponent` / `Instantiate` / `new GameObject<T>` / the `Service` API.
  - Edit Mode test assemblies may also reference `InitArgs.Editor` for the `Sisus.Init.Testing.Testable` helper.
