[![gitter](https://img.shields.io/gitter/room/leopotam/ecs.svg)](https://gitter.im/leopotam/ecs)
[![license](https://img.shields.io/github/license/Leopotam/ecs.svg)](https://github.com/Leopotam/ecs/blob/develop/LICENSE)
# Another one Entity Component System framework
Performance and zero memory allocation / no gc pressure / small size, no dependencies on any game engine - main goals of this project.

> Tested on unity 2017.4 (not dependent on it) and contains assembly definition for compiling to separate assembly file for performance reason.

# Main parts of ecs

## Component
Container for user data without / with small logic inside. Can be used any user class without any additional inheritance:
```
class WeaponComponent {
    public int Ammo;
    public string GunName;
}
```

## Entity
Сontainer for components. Implemented with int id-s for more simplified api:
```
int entity = _world.CreateEntity ();
_world.RemoveEntity (entity);
```

## System
Сontainer for logic for processing filtered entities. User class should implements `IEcsPreInitSystem`, `IEcsInitSystem` or `IEcsRunSystem` interfaces:
```
class WeaponSystem : IEcsPreInitSystem, IEcsInitSystem {
    void IEcsPreInitSystem.PreInitialize () {
        // Will be called once during world initialization before IEcsInitSystem.Initialize().
    }

    void IEcsInitSystem.Initialize () {
        // Will be called once during world initialization.
    }

    void IEcsPreInitSystem.PreDestroy () {
        // Will be called once during world destruction before IEcsInitSystem.Destroy().
    }

    void IEcsInitSystem.Destroy () {
        // Will be called once during world destruction.
    }
}
```

```
class HealthSystem : IEcsRunSystem {
    void IEcsRunSystem.Run () {
        // Will be called on each EcsSystems.Run() call.
    }
}
```

# Data injection
With `[EcsWorld]`, `[EcsFilterInclude(typeof(X))]` and `[EcsFilterExclude(typeof(X))]` attributes any compatible field of custom `IEcsSystem` class can be auto initialized (auto injected):
```
class HealthSystem : IEcsSystem {
    [EcsWorld]
    EcsWorld _world;

    [EcsFilterInclude(typeof(WeaponComponent))]
    EcsFilter _weaponFilter;
}
```

# Special classes

## EcsFilter
Container for keep filtered entities with specified component list:
```
class WeaponSystem : IEcsInitSystem, IEcsRunSystem {
    [EcsWorld]
    EcsWorld _world;

    // We wants to get entities with WeaponComponent and without HealthComponent.
    [EcsFilterInclude(typeof(WeaponComponent))]
    [EcsFilterExclude(typeof(HealthComponent))]
    // If this filter not exists (will be created) - force scan world for compatible entities.
    [EcsFilterFill]
    EcsFilter _filter;

    void IEcsInitSystem.Initialize () {
        var newEntity = _world.CreateEntity ();
        _world.AddComponent<WeaponComponent> (newEntity);
    }

    void IEcsInitSystem.Destroy () { }

    void IEcsRunSystem.Run () {
        // Important: foreach-loop cant be used for filtered entities!
        for (var i = 0; i < _filter.EntitiesCount; i++) {
            var entity = _filter.Entities[i];
            var weapon = _world.GetComponent<WeaponComponent> (entity);
            weapon.Ammo = System.Math.Max (0, weapon.Ammo - 1);
        }
    }
}
```
> Important: filter.Entities cant be iterated with foreach-loop, for-loop should be used instead with filter.EntitiesCount value as upper-bound.

## EcsWorld
Root level container for all entities / components, works like isolated environment.

## EcsSystems
Group of systems to process `EcsWorld` instance:
```
class Startup : MonoBehaviour {
    EcsSystems _systems;

    void OnEnable() {
        // create ecs environment.
        var world = new EcsWorld ();
        _systems = new EcsSystems(world)
            .Add (new WeaponSystem ());
        _systems.Initialize ();
    }
    
    void Update() {
        // process all dependent systems.
        _systems.Run ();
    }

    void OnDisable() {
        // destroy systems logical group.
        _systems.Destroy ();
    }
}
```

# Reaction on component / filter changes
## Process events from EcsFilter as stream
`EcsReactSystem` class can be used for this case.

> Important: this system not supported processing of `OnRemove` event.

> **Will not work with LEOECS_DISABLE_REACTIVE preprocessor define.**

```
public sealed class TestReactSystem : EcsReactSystem {
    [EcsWorld]
    EcsWorld _world;

    [EcsFilterInclude (typeof (WeaponComponent))]
    EcsFilter _weaponFilter;

    // Should returns filter that react system will watch for.
    public override EcsFilter GetReactFilter () {
        return _weaponFilter;
    }

    // On which event at filter this react-system should be alerted -
    // "new entity in filter" or "entity inplace update".
    public override EcsReactSystemType GetReactSystemType () {
        return EcsReactSystemType.OnUpdate;
    }

    // Filtered entities processing, will be raised only if entities presents.
    public override void RunReact (int[] entities, int count) {
        for (var i = 0; i < count; i++) {
            var entity = entities[i];
            var weapon = _world.GetComponent<WeaponComponent> (entity);
            Debug.LogFormat ("Weapon updated on {0}", entity);
        }
    }
}
```

## Process events from EcsFilter immediately
`EcsInstantReactSystem` class can be used for this case.

Useful case for using this type of processing - reaction from OnRemove event.

> **Will not work with LEOECS_DISABLE_REACTIVE preprocessor define.**

```
public sealed class TestReactInstantSystem : EcsReactInstantSystem {
    [EcsWorld]
    EcsWorld _world;

    [EcsFilterInclude (typeof (WeaponComponent))]
    EcsFilter _weaponFilter;

    // Should returns filter that react system will watch for.
    public override EcsFilter GetReactFilter () {
        return _weaponFilter;
    }

    // On which event at filter this react-system should be alerted -
    // "enity was removed from filter".
    public override EcsReactSystemType GetReactSystemType () {
        return EcsReactSystemType.OnRemove;
    }

    // Entity processing, will be raised only when entity will be removed from filter.
    public override void RunReact (int entity, object reason) {
        // not works - component already detached at this moment.
        var weapon = _world.GetComponent<WeaponComponent> (entity);

        // reason - detached component instance.
        Debug.LogFormat ("{1} removed from {0}", entity, reason.GetType().Name);
    }
}
```

## Custom reaction
For handling of filter events `custom class` should implements `IEcsFilterListener` interface with 3 methods: `OnFilterEntityAdded` / `OnFilterEntityRemoved` / `OnFilterEntityUpdated`. Then it can be added to any filter as compatible listener.

> **Not recommended if you dont understand how it works internally.**

> **Will not work with LEOECS_DISABLE_REACTIVE preprocessor define.**

```
public sealed class TestSystem1 : IEcsInitSystem, IEcsFilterListener {
    [EcsWorld]
    EcsWorld _world;

    [EcsFilterInclude (typeof (WeaponComponent))]
    EcsFilter _weaponFilter;

    void IEcsInitSystem.Initialize () {
        _weaponFilter.AddListener(this);

        var entity = _world.CreateEntity ();
        _world.AddComponent<WeaponComponent> (entity);
        _world.AddComponent<HealthComponent> (entity);
    }

    void IEcsInitSystem.Destroy () {
        _weaponFilter.RemoveListener(this);
    }

    void IEcsFilterListener.OnFilterEntityAdded (int entity, object reason) {
        // Entity "entityId" was added to _weaponFilter due to component "reason" with type "WeaponComponent" was added to entity.
    }

    void IEcsFilterListener.OnFilterEntityUpdated(int entityId, object reason) {
        // Component "reason" with type "WeaponComponent" was updated inplace on entity "entityId".
    }

    void IEcsFilterListener.OnFilterEntityRemoved (int entity, object reason) {
        // Entity "entityId" was removed from _weaponFilter due to component "reason" with type "WeaponComponent" was removed from entity.
    }
}
```

# Sharing data between systems
If `EcsWorld` class should contains some shared fields (useful for sharing assets / prefabs), it can be implemented in this way:
```
class MySharedData : ScriptableObject {
    public string PlayerName = "Unknown";
    public GameObject PlayerModel;
}

class MyWorld: EcsWorld {
    public readonly MySharedData Assets;

    public MyWorld(MySharedData data) {
        Assets = data;
    }
}

class ChangePlayerName : IEcsInitSystem {
    [EcsWorld]
    MyWorld _world;

    // This field will be initialized with same reference as _world field.
    [EcsWorld]
    EcsWorld _standardWorld;

    void IEcsInitSystem.Initialize () {
        _world.Assets.PlayerName = "Jack";
    }

    void IEcsInitSystem.Destroy () { }
}

class SpawnPlayerModel : IEcsInitSystem {
    [EcsWorld]
    MyWorld _world;

    void IEcsInitSystem.Initialize () {
        GameObject.Instantiate(_world.Assets.PlayerModel);
    }

    void IEcsInitSystem.Destroy () { }
}

class Startup : Monobehaviour {
    [SerializedField]
    MySharedData _sharedData;

    EcsSystems _systems;

    void OnEnable() {
        var world = new MyWorld (_sharedData);
        _systems = new EcsSystems(world)
            .Add (ChangePlayerName())
            .Add (SpawnPlayerModel());
        _systems.Initialize();
    }
}
```

# Examples
[Snake game](https://github.com/Leopotam/ecs-snake)

# Extensions
[Unity integration](https://github.com/Leopotam/ecs-unityintegration)

[uGui event bindings](https://github.com/Leopotam/ecs-ui)

# License
The software released under the terms of the MIT license. Enjoy.

# FAQ

### My project complex enough, I need more than 256 components. How I can do it?

There are no components limit, but for performance / memory usage reason better to keep amount of components on each entity less or equals 8.

### I want to create alot of new entities with new components on start, how to speed up this process?

In this case custom component creator can be used (for speed up 2x or more):

```
class MyComponent { }

class Startup : Monobehaviour {
    EcsSystems _systems;

    void OnEnable() {
        var world = new MyWorld (_sharedData);
        
        EcsWorld.RegisterComponentCreator<MyComponent> (() => new MyComponent());
        
        _systems = new EcsSystems(world)
            .Add (MySystem());
        _systems.Initialize();
    }
}
```

### I want to process one system at MonoBehaviour.Update() and another - at MonoBehaviour.FixedUpdate(). How I can do it?

For splitting systems by `MonoBehaviour`-method multiple `EcsSystems` logical groups should be used:
```
EcsSystems _update;
EcsSystems _fixedUpdate;

void OnEnable() {
    var world = new EcsWorld();
    _update = new EcsSystems(world).Add(new UpdateSystem());
    _fixedUpdate = new EcsSystems(world).Add(new FixedUpdateSystem());
}

void Update() {
    _update.Run();
}

void FixedUpdate() {
    _fixedUpdate.Run();
}
```

### I dont need reactive events processing and want more performance. How I can do it?

Reactive events support can be removed with **LEOECS_DISABLE_REACTIVE** preprocessor define:
* No any reaction on filter entities list change.
* Related code will be eliminated from ECS assembly.
* Filters should be updated faster.
* Less memory usage.

### How it fast relative to Entitas?

Some test can be found at [this repo](https://github.com/echeg/unityecs_speedtest). Tests can be obsoleted, better to grab last versions of frameworks and check locally.