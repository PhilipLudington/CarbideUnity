# Carbide Unity - AI Development Guide

A framework for hardened Unity/C# development optimized for AI assistance.

## Quick Reference

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Classes/Structs | PascalCase | `PlayerController`, `EnemySpawner` |
| Interfaces | I + PascalCase | `IInteractable`, `IDamageable` |
| Methods | PascalCase | `TakeDamage()`, `Initialize()` |
| Properties | PascalCase | `Health`, `MaxSpeed` |
| Fields (private) | _camelCase | `_currentHealth`, `_isGrounded` |
| Fields (public/serialized) | camelCase | `moveSpeed`, `jumpForce` |
| Constants | PascalCase | `MaxPlayers`, `DefaultHealth` |
| Events | On + PascalCase | `OnDeath`, `OnDamageReceived` |
| Enums | PascalCase (singular) | `WeaponType`, `GameState` |
| Parameters | camelCase | `damage`, `targetPosition` |
| Local variables | camelCase | `hitCount`, `nearestEnemy` |
| Booleans | is/has/can/should prefix | `isAlive`, `hasWeapon`, `canJump` |

### MonoBehaviour Lifecycle

```
Awake()           → Cache references, initialize state (runs once, before Start)
OnEnable()        → Subscribe to events, reset state
Start()           → Initial setup requiring other objects (runs once, after Awake)
FixedUpdate()     → Physics calculations (fixed timestep)
Update()          → Game logic, input handling (every frame)
LateUpdate()      → Camera follow, post-processing (after all Updates)
OnDisable()       → Unsubscribe from events
OnDestroy()       → Final cleanup
```

### Memory Rules

1. **Pool frequently instantiated objects** - Never Instantiate/Destroy in gameplay loops
2. **Avoid allocations in Update** - No `new`, LINQ, string concatenation, or boxing
3. **Use structs for small data** - Value types avoid heap allocation
4. **Cache component references** - GetComponent in Awake, not Update
5. **Use NonAlloc methods** - `Physics.RaycastNonAlloc`, `GetComponentsInChildren(list)`
6. **StringBuilder for strings** - Never concatenate strings in loops
7. **Clear collections, don't recreate** - `list.Clear()` not `list = new List<T>()`

### Serialization Rules

1. **Mark serialized fields explicitly** - Use `[SerializeField]` for private fields
2. **Never serialize interfaces** - Unity can't serialize them
3. **Avoid serializing large collections** - Performance impact in Inspector
4. **Use ScriptableObjects for shared data** - Not static fields
5. **Initialize serialized references** - Check for null in Awake/OnEnable

### Event Handling

```csharp
// GOOD: Use C# events with null-conditional
public event Action<int> OnHealthChanged;

private void NotifyHealthChanged()
{
    OnHealthChanged?.Invoke(_currentHealth);
}

// GOOD: Always unsubscribe
private void OnEnable()
{
    GameEvents.OnGamePaused += HandleGamePaused;
}

private void OnDisable()
{
    GameEvents.OnGamePaused -= HandleGamePaused;
}
```

### Coroutine Rules

1. **Cache WaitForSeconds** - `private readonly WaitForSeconds _wait = new(1f);`
2. **Stop coroutines explicitly** - Store reference and StopCoroutine on disable
3. **Prefer async/await for non-frame-dependent** - UniTask or native async
4. **Never yield null in loops** - Use `WaitForEndOfFrame` if needed

### Physics Best Practices

1. **Use layers for collision filtering** - Don't check tags in OnCollision
2. **Raycast with layer masks** - `Physics.Raycast(ray, out hit, distance, layerMask)`
3. **OverlapSphereNonAlloc** - Pre-allocate results array
4. **Rigidbody in FixedUpdate only** - Never modify in Update
5. **Set rigidbody.isKinematic** - Before teleporting, then unset

### Null Safety

```csharp
// GOOD: Explicit null checks with meaningful response
if (target == null)
{
    Debug.LogWarning($"{name}: Target is null, skipping attack");
    return;
}

// GOOD: TryGetComponent pattern
if (collision.TryGetComponent<IDamageable>(out var damageable))
{
    damageable.TakeDamage(_damage);
}

// GOOD: Null-conditional for events
OnDeath?.Invoke();

// BAD: Silent failures
target?.TakeDamage(10); // Hides bugs
```

### Architecture Patterns

| Pattern | Use Case |
|---------|----------|
| ScriptableObject Events | Decoupled communication between systems |
| ScriptableObject Variables | Shared state without singletons |
| State Machine | Character states, game flow, UI states |
| Object Pool | Bullets, particles, enemies, UI elements |
| Service Locator | Runtime-swappable systems |
| Command Pattern | Input handling, undo/redo |

### Common Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Find in Update | Performance | Cache in Awake |
| GetComponent in Update | Performance | Cache reference |
| Public fields everywhere | Encapsulation | [SerializeField] private |
| Singletons for everything | Coupling, testing | ScriptableObjects, DI |
| String-based messaging | Fragile, slow | Typed events |
| Tag comparisons | Fragile, slow | Layer masks, components |
| Camera.main in Update | Performance | Cache reference |

### Required Attributes

```csharp
// Document serialized fields for designers
[SerializeField, Tooltip("Movement speed in units/second")]
private float _moveSpeed = 5f;

// Validate in inspector
[SerializeField, Range(0, 100)]
private int _health = 100;

// Require dependencies
[RequireComponent(typeof(Rigidbody))]
public class PhysicsObject : MonoBehaviour { }

// Prevent multiple instances
[DisallowMultipleComponent]
public class GameManager : MonoBehaviour { }
```

### File Organization

```
Assets/
├── _Project/                 # Project-specific assets
│   ├── Scripts/
│   │   ├── Core/            # Managers, systems, utilities
│   │   ├── Gameplay/        # Player, enemies, items
│   │   ├── UI/              # UI controllers, views
│   │   └── Data/            # ScriptableObjects
│   ├── Prefabs/
│   ├── Scenes/
│   ├── Art/
│   └── Audio/
├── Plugins/                  # Third-party packages
└── Resources/                # Only for runtime-loaded assets
```

### Script Template

```csharp
using UnityEngine;

namespace ProjectName.Gameplay
{
    /// <summary>
    /// Brief description of class responsibility.
    /// </summary>
    public class ExampleBehaviour : MonoBehaviour
    {
        [Header("Configuration")]
        [SerializeField, Tooltip("Description for designer")]
        private float _exampleValue = 1f;

        [Header("References")]
        [SerializeField]
        private Transform _targetTransform;

        // Events
        public event Action<float> OnValueChanged;

        // Cached references
        private Rigidbody _rigidbody;

        // State
        private bool _isInitialized;

        private void Awake()
        {
            CacheReferences();
        }

        private void OnEnable()
        {
            SubscribeToEvents();
        }

        private void OnDisable()
        {
            UnsubscribeFromEvents();
        }

        private void CacheReferences()
        {
            _rigidbody = GetComponent<Rigidbody>();
        }

        private void SubscribeToEvents() { }
        private void UnsubscribeFromEvents() { }
    }
}
```

## See Also

- [STANDARDS.md](STANDARDS.md) - Complete coding standards
- [docs/patterns/](docs/patterns/) - Implementation patterns
- [docs/safety/](docs/safety/) - Common pitfalls and prevention
