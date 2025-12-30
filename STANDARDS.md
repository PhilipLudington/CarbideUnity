# Carbide Unity - Coding Standards

Comprehensive coding standards for Unity/C# development optimized for AI assistance and maintainable, performant code.

## Table of Contents

1. [Language Standards](#language-standards)
2. [Naming Conventions](#naming-conventions)
3. [Code Organization](#code-organization)
4. [MonoBehaviour Guidelines](#monobehaviour-guidelines)
5. [Memory Management](#memory-management)
6. [Error Handling](#error-handling)
7. [Performance Guidelines](#performance-guidelines)
8. [Serialization](#serialization)
9. [Architecture Guidelines](#architecture-guidelines)
10. [Testing Guidelines](#testing-guidelines)

---

## Language Standards

### Version Requirements

- **C# Version**: C# 9.0+ (Unity 2021.2+)
- **Unity Version**: 2021.3 LTS or newer recommended
- **.NET**: .NET Standard 2.1

### Language Features to Use

```csharp
// Pattern matching
if (collision.gameObject.TryGetComponent<IDamageable>(out var damageable))

// Null-conditional operators
OnDeath?.Invoke();
var health = player?.Stats?.Health ?? 0;

// Expression-bodied members for simple properties
public int Health => _currentHealth;
public bool IsAlive => _currentHealth > 0;

// Target-typed new
private readonly List<Enemy> _enemies = new();
private readonly Dictionary<string, int> _scores = new();

// Switch expressions
private Color GetHealthColor(int health) => health switch
{
    > 75 => Color.green,
    > 25 => Color.yellow,
    _ => Color.red
};

// Records for immutable data (Unity 2021.2+)
public readonly record struct DamageInfo(int Amount, DamageType Type, Vector3 Origin);
```

### Language Features to Avoid

```csharp
// AVOID: Dynamic typing
dynamic value = GetValue(); // No compile-time checking

// AVOID: Reflection in gameplay code
var method = type.GetMethod("Update"); // Slow, fragile

// AVOID: LINQ in Update loops
var nearest = enemies.OrderBy(e => Distance(e)).First(); // Allocates

// AVOID: Async void (except for event handlers)
async void BadMethod() { } // Can't be awaited, exceptions are lost

// AVOID: Finalizers
~MyClass() { } // Unpredictable timing, GC pressure
```

---

## Naming Conventions

### Identifiers

| Element | Convention | Example |
|---------|------------|---------|
| Namespace | PascalCase, hierarchical | `MyGame.Gameplay.Combat` |
| Class | PascalCase, noun | `PlayerController` |
| Struct | PascalCase, noun | `DamageInfo` |
| Interface | I + PascalCase | `IDamageable` |
| Enum | PascalCase, singular | `WeaponType` |
| Enum value | PascalCase | `WeaponType.Sword` |
| Method | PascalCase, verb | `CalculateDamage()` |
| Property | PascalCase | `CurrentHealth` |
| Event | On + PascalCase | `OnHealthChanged` |
| Delegate | PascalCase + Handler/Action | `DamageHandler` |
| Private field | _camelCase | `_currentHealth` |
| Serialized field | camelCase (public) or _camelCase with [SerializeField] | `moveSpeed` |
| Constant | PascalCase | `MaxHealth` |
| Static readonly | PascalCase | `DefaultConfig` |
| Parameter | camelCase | `targetPosition` |
| Local variable | camelCase | `nearestEnemy` |
| Generic type | T + Descriptor | `TItem`, `TKey`, `TValue` |

### Boolean Naming

Booleans must use interrogative prefixes:

```csharp
// Fields
private bool _isGrounded;
private bool _hasWeapon;
private bool _canJump;
private bool _shouldRespawn;
private bool _wasHit;

// Properties
public bool IsAlive => _currentHealth > 0;
public bool HasAmmo => _ammoCount > 0;
public bool CanFire => _hasWeapon && _ammoCount > 0 && !_isReloading;

// Methods returning bool
public bool IsInRange(Vector3 position) { }
public bool HasLineOfSight(Transform target) { }
public bool CanAfford(int cost) { }
```

### Method Naming Patterns

| Prefix | Usage | Example |
|--------|-------|---------|
| Get | Return a value | `GetNearestEnemy()` |
| Set | Assign a value | `SetHealth(int value)` |
| Try | May fail, returns bool | `TryGetComponent<T>(out T)` |
| Calculate | Compute and return | `CalculateDamage()` |
| Find | Search for something | `FindPath(Vector3 target)` |
| Create | Factory method | `CreateBullet(Vector3 pos)` |
| Initialize | Setup after construction | `Initialize(Config config)` |
| Handle | Event handler | `HandlePlayerDeath()` |
| On | Unity callback or event | `OnTriggerEnter()` |
| Validate | Check and possibly fix | `ValidateReferences()` |
| Update | Refresh state | `UpdateUI()` |
| Apply | Apply changes | `ApplyDamage()` |

### File Naming

- **One class per file** (except nested classes)
- **File name matches class name**: `PlayerController.cs`
- **Partial classes**: `PlayerController.Movement.cs`, `PlayerController.Combat.cs`
- **Interfaces**: `IDamageable.cs`
- **ScriptableObjects**: `PlayerStats.cs` (class), `PlayerStats.asset` (instance)

---

## Code Organization

### File Structure

```csharp
// 1. Using statements (sorted, system first)
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using MyGame.Core;

// 2. Namespace
namespace MyGame.Gameplay
{
    // 3. Class declaration with XML docs
    /// <summary>
    /// Controls player movement, input, and state.
    /// </summary>
    [RequireComponent(typeof(CharacterController))]
    public class PlayerController : MonoBehaviour, IDamageable
    {
        // 4. Constants and static readonly
        private const float GroundCheckDistance = 0.1f;
        private static readonly int AnimatorSpeedHash = Animator.StringToHash("Speed");

        // 5. Serialized fields (grouped with Headers)
        [Header("Movement")]
        [SerializeField, Tooltip("Movement speed in units/second")]
        private float _moveSpeed = 5f;

        [SerializeField, Range(1f, 20f)]
        private float _jumpForce = 10f;

        [Header("References")]
        [SerializeField]
        private Transform _groundCheck;

        // 6. Events
        public event Action<int> OnHealthChanged;
        public event Action OnDeath;

        // 7. Properties
        public int CurrentHealth => _currentHealth;
        public bool IsAlive => _currentHealth > 0;

        // 8. Private fields (runtime state)
        private CharacterController _characterController;
        private Vector3 _velocity;
        private bool _isGrounded;
        private int _currentHealth;

        // 9. Unity lifecycle methods (in execution order)
        private void Awake() { }
        private void OnEnable() { }
        private void Start() { }
        private void FixedUpdate() { }
        private void Update() { }
        private void LateUpdate() { }
        private void OnDisable() { }
        private void OnDestroy() { }

        // 10. Public methods
        public void TakeDamage(int damage) { }
        public void Heal(int amount) { }

        // 11. Private methods
        private void HandleMovement() { }
        private void CheckGround() { }

        // 12. Event handlers
        private void HandleGamePaused(bool isPaused) { }

        // 13. Unity callbacks (collision, trigger, etc.)
        private void OnTriggerEnter(Collider other) { }
        private void OnCollisionEnter(Collision collision) { }

        // 14. Editor-only code
        #if UNITY_EDITOR
        private void OnValidate() { }
        private void OnDrawGizmosSelected() { }
        #endif
    }
}
```

### Region Usage

Avoid regions in most cases. If a class needs regions, it's probably too large.

```csharp
// AVOID: Regions hiding complexity
#region Movement
// 500 lines of code
#endregion

// PREFER: Smaller, focused classes
// PlayerMovement.cs - handles movement
// PlayerCombat.cs - handles combat
// PlayerInput.cs - handles input
```

### Assembly Definitions

Organize code into assembly definitions for faster compilation:

```
Assets/_Project/
├── Scripts/
│   ├── Core/
│   │   ├── MyGame.Core.asmdef
│   │   └── *.cs
│   ├── Gameplay/
│   │   ├── MyGame.Gameplay.asmdef  (references: MyGame.Core)
│   │   └── *.cs
│   ├── UI/
│   │   ├── MyGame.UI.asmdef        (references: MyGame.Core)
│   │   └── *.cs
│   └── Tests/
│       ├── MyGame.Tests.asmdef     (Editor only, references all)
│       └── *.cs
```

---

## MonoBehaviour Guidelines

### Lifecycle Method Usage

```csharp
public class ExampleBehaviour : MonoBehaviour
{
    // Awake: Called once, before Start, even if disabled
    // USE FOR: Caching own components, initializing state
    private void Awake()
    {
        _rigidbody = GetComponent<Rigidbody>();
        _currentHealth = _maxHealth;
    }

    // OnEnable: Called every time object is enabled
    // USE FOR: Subscribing to events, resetting per-activation state
    private void OnEnable()
    {
        GameEvents.OnGamePaused += HandleGamePaused;
        _isActive = true;
    }

    // Start: Called once, before first Update, only if enabled
    // USE FOR: Setup requiring other objects to be initialized
    private void Start()
    {
        _target = FindObjectOfType<Player>().transform;
    }

    // FixedUpdate: Called at fixed intervals (default 50Hz)
    // USE FOR: Physics calculations, rigidbody manipulation
    private void FixedUpdate()
    {
        _rigidbody.AddForce(_moveDirection * _force);
    }

    // Update: Called every frame
    // USE FOR: Input handling, game logic, non-physics movement
    private void Update()
    {
        HandleInput();
        UpdateState();
    }

    // LateUpdate: Called after all Update calls
    // USE FOR: Camera following, UI updates, post-processing
    private void LateUpdate()
    {
        FollowTarget();
    }

    // OnDisable: Called when object is disabled or destroyed
    // USE FOR: Unsubscribing from events, cleanup
    private void OnDisable()
    {
        GameEvents.OnGamePaused -= HandleGamePaused;
    }

    // OnDestroy: Called when object is destroyed
    // USE FOR: Final cleanup, saving state
    private void OnDestroy()
    {
        SaveProgress();
    }
}
```

### Component Dependencies

```csharp
// GOOD: Declare dependencies with RequireComponent
[RequireComponent(typeof(Rigidbody))]
[RequireComponent(typeof(Collider))]
public class PhysicsObject : MonoBehaviour
{
    private Rigidbody _rigidbody;

    private void Awake()
    {
        // Safe: RequireComponent guarantees it exists
        _rigidbody = GetComponent<Rigidbody>();
    }
}

// GOOD: Validate optional references
[SerializeField]
private AudioSource _audioSource; // Optional

private void Awake()
{
    if (_audioSource == null)
    {
        _audioSource = GetComponent<AudioSource>();
    }
}

// GOOD: Editor validation
#if UNITY_EDITOR
private void OnValidate()
{
    if (_targetTransform == null)
    {
        Debug.LogWarning($"{name}: Target Transform is not assigned", this);
    }
}
#endif
```

### Disabling vs Destroying

```csharp
// Use SetActive(false) for temporary removal (pooling)
public void ReturnToPool()
{
    gameObject.SetActive(false);
    _pool.Return(this);
}

// Use Destroy for permanent removal
public void Die()
{
    OnDeath?.Invoke();
    Destroy(gameObject);
}

// Never destroy immediately in physics callbacks
private void OnCollisionEnter(Collision collision)
{
    // BAD: Can cause issues
    // Destroy(gameObject);

    // GOOD: Delay destruction
    Destroy(gameObject, 0.01f);

    // BETTER: Disable and pool
    gameObject.SetActive(false);
}
```

---

## Memory Management

### Allocation Rules

```csharp
// RULE 1: Never allocate in Update/FixedUpdate/LateUpdate

// BAD: Creates garbage every frame
private void Update()
{
    var enemies = FindObjectsOfType<Enemy>();           // Allocates array
    var filtered = enemies.Where(e => e.IsAlive);      // Allocates enumerator
    var text = "Score: " + score;                       // Allocates string
    var list = new List<int>();                         // Allocates list
}

// GOOD: Pre-allocate and reuse
private readonly List<Enemy> _enemyCache = new();
private readonly StringBuilder _stringBuilder = new();
private Enemy[] _enemyArray;

private void Awake()
{
    _enemyArray = new Enemy[MaxEnemies];
}

private void Update()
{
    int count = GetEnemiesNonAlloc(_enemyArray);

    _stringBuilder.Clear();
    _stringBuilder.Append("Score: ");
    _stringBuilder.Append(score);
    _scoreText.text = _stringBuilder.ToString();
}
```

### Object Pooling

```csharp
// RULE 2: Pool frequently instantiated objects

public class BulletPool : MonoBehaviour
{
    [SerializeField] private Bullet _prefab;
    [SerializeField] private int _initialSize = 20;

    private readonly Queue<Bullet> _available = new();

    private void Awake()
    {
        for (int i = 0; i < _initialSize; i++)
        {
            var bullet = Instantiate(_prefab, transform);
            bullet.gameObject.SetActive(false);
            bullet.Initialize(this);
            _available.Enqueue(bullet);
        }
    }

    public Bullet Get(Vector3 position, Quaternion rotation)
    {
        Bullet bullet;

        if (_available.Count > 0)
        {
            bullet = _available.Dequeue();
        }
        else
        {
            bullet = Instantiate(_prefab, transform);
            bullet.Initialize(this);
        }

        bullet.transform.SetPositionAndRotation(position, rotation);
        bullet.gameObject.SetActive(true);
        return bullet;
    }

    public void Return(Bullet bullet)
    {
        bullet.gameObject.SetActive(false);
        _available.Enqueue(bullet);
    }
}
```

### Struct vs Class

```csharp
// Use STRUCT for:
// - Small data (< 16 bytes ideally)
// - Immutable data
// - Short-lived values
// - Data passed by value

public readonly struct DamageInfo
{
    public int Amount { get; }
    public DamageType Type { get; }
    public Vector3 Origin { get; }

    public DamageInfo(int amount, DamageType type, Vector3 origin)
    {
        Amount = amount;
        Type = type;
        Origin = origin;
    }
}

// Use CLASS for:
// - Large data
// - Mutable state
// - Reference semantics needed
// - Inheritance needed

public class Enemy : MonoBehaviour
{
    // Class because it's a MonoBehaviour with mutable state
}
```

### String Handling

```csharp
// BAD: String concatenation allocates
string result = "Player " + playerName + " scored " + score + " points";

// GOOD: StringBuilder for multiple concatenations
private readonly StringBuilder _sb = new();

private string BuildScoreText(string playerName, int score)
{
    _sb.Clear();
    _sb.Append("Player ");
    _sb.Append(playerName);
    _sb.Append(" scored ");
    _sb.Append(score);
    _sb.Append(" points");
    return _sb.ToString();
}

// GOOD: String interpolation for simple cases (still allocates, but cleaner)
string result = $"Score: {score}";

// GOOD: Cached strings for repeated use
private static class UIStrings
{
    public const string ScorePrefix = "Score: ";
    public const string HealthPrefix = "HP: ";
}
```

---

## Error Handling

### Null Checking

```csharp
// RULE: Check nulls explicitly with meaningful responses

// Pattern 1: Guard clause with return
public void Attack(IDamageable target)
{
    if (target == null)
    {
        Debug.LogWarning($"{name}: Cannot attack null target");
        return;
    }

    target.TakeDamage(_damage);
}

// Pattern 2: TryGetComponent
private void OnTriggerEnter(Collider other)
{
    if (other.TryGetComponent<IPickup>(out var pickup))
    {
        pickup.Collect(this);
    }
}

// Pattern 3: Null-conditional for events (silent is OK for events)
OnDeath?.Invoke();

// Pattern 4: Null-coalescing for defaults
var config = customConfig ?? DefaultConfig;

// AVOID: Silent null-conditional hiding bugs
// BAD: Silently does nothing if target is null
target?.TakeDamage(10);

// GOOD: Explicit check with logging
if (target != null)
{
    target.TakeDamage(10);
}
else
{
    Debug.LogWarning($"{name}: Target is null");
}
```

### Unity Object Null

```csharp
// Unity overloads == for destroyed objects
// A destroyed object is "== null" but not "is null"

public void SafeDestroy()
{
    // GOOD: Works with Unity's null check
    if (_target == null) return;

    // BAD: Bypasses Unity's null check, may access destroyed object
    if (_target is null) return;

    // GOOD: Explicit destroyed check
    if (_target == null || _target.Equals(null)) return;
}
```

### Exception Handling

```csharp
// RULE: Only catch exceptions you can handle

// GOOD: Specific exception, specific handling
public bool TryLoadConfig(string path, out Config config)
{
    try
    {
        string json = File.ReadAllText(path);
        config = JsonUtility.FromJson<Config>(json);
        return true;
    }
    catch (FileNotFoundException)
    {
        Debug.LogWarning($"Config file not found: {path}, using defaults");
        config = Config.Default;
        return false;
    }
    catch (JsonException ex)
    {
        Debug.LogError($"Invalid config JSON: {ex.Message}");
        config = Config.Default;
        return false;
    }
}

// BAD: Catching everything
try
{
    DoSomething();
}
catch (Exception) // Hides bugs
{
    // Now what?
}

// AVOID: Exceptions for flow control
// BAD
try
{
    var component = GetComponent<Rigidbody>();
    component.AddForce(force);
}
catch (NullReferenceException)
{
    // No rigidbody
}

// GOOD
if (TryGetComponent<Rigidbody>(out var rb))
{
    rb.AddForce(force);
}
```

### Assertions and Validation

```csharp
using UnityEngine.Assertions;

public class Player : MonoBehaviour
{
    [SerializeField] private Transform _spawnPoint;
    [SerializeField] private PlayerStats _stats;

    private void Awake()
    {
        // Assertions for required references (development only)
        Assert.IsNotNull(_spawnPoint, "Spawn point must be assigned");
        Assert.IsNotNull(_stats, "Player stats must be assigned");
        Assert.IsTrue(_stats.MaxHealth > 0, "Max health must be positive");
    }

    public void Initialize(int level)
    {
        // Validate parameters
        if (level < 1 || level > MaxLevel)
        {
            throw new ArgumentOutOfRangeException(nameof(level),
                $"Level must be between 1 and {MaxLevel}");
        }
    }
}
```

---

## Performance Guidelines

### Update Optimization

```csharp
// RULE: Minimize work in Update

// Pattern 1: Dirty flags
private bool _needsUIUpdate;
private int _lastDisplayedScore;

public int Score
{
    get => _score;
    set
    {
        if (_score != value)
        {
            _score = value;
            _needsUIUpdate = true;
        }
    }
}

private void Update()
{
    if (_needsUIUpdate)
    {
        UpdateScoreDisplay();
        _needsUIUpdate = false;
    }
}

// Pattern 2: Throttled updates
private float _lastUpdateTime;
private const float UpdateInterval = 0.1f;

private void Update()
{
    if (Time.time - _lastUpdateTime >= UpdateInterval)
    {
        ExpensiveUpdate();
        _lastUpdateTime = Time.time;
    }
}

// Pattern 3: Coroutine for periodic work
private IEnumerator SlowUpdateRoutine()
{
    var wait = new WaitForSeconds(0.5f);
    while (enabled)
    {
        ExpensiveWork();
        yield return wait;
    }
}
```

### Physics Optimization

```csharp
// RULE: Use NonAlloc methods

private readonly Collider[] _overlapResults = new Collider[32];
private readonly RaycastHit[] _raycastResults = new RaycastHit[16];

private void FindNearbyEnemies()
{
    int count = Physics.OverlapSphereNonAlloc(
        transform.position,
        _detectionRadius,
        _overlapResults,
        _enemyLayerMask
    );

    for (int i = 0; i < count; i++)
    {
        ProcessEnemy(_overlapResults[i]);
    }
}

private void RaycastForward()
{
    int count = Physics.RaycastNonAlloc(
        transform.position,
        transform.forward,
        _raycastResults,
        _maxDistance,
        _hitLayerMask
    );

    for (int i = 0; i < count; i++)
    {
        ProcessHit(_raycastResults[i]);
    }
}
```

### Caching

```csharp
// RULE: Cache everything accessed repeatedly

public class OptimizedBehaviour : MonoBehaviour
{
    // Cache component references
    private Transform _transform;
    private Rigidbody _rigidbody;

    // Cache expensive lookups
    private Camera _mainCamera;
    private Transform _playerTransform;

    // Cache animator hashes
    private static readonly int SpeedHash = Animator.StringToHash("Speed");
    private static readonly int JumpHash = Animator.StringToHash("Jump");

    // Cache shader property IDs
    private static readonly int ColorProperty = Shader.PropertyToID("_Color");

    // Cache WaitForSeconds
    private readonly WaitForSeconds _waitOneSecond = new(1f);
    private readonly WaitForEndOfFrame _waitEndOfFrame = new();

    private void Awake()
    {
        _transform = transform;
        _rigidbody = GetComponent<Rigidbody>();
        _mainCamera = Camera.main;
    }

    private void Start()
    {
        // Cache after other objects initialize
        var player = FindObjectOfType<Player>();
        if (player != null)
        {
            _playerTransform = player.transform;
        }
    }
}
```

---

## Serialization

### SerializeField Usage

```csharp
public class SerializationExample : MonoBehaviour
{
    // GOOD: Private with SerializeField
    [SerializeField]
    private float _moveSpeed = 5f;

    // GOOD: With tooltip for designers
    [SerializeField, Tooltip("Time in seconds before respawn")]
    private float _respawnDelay = 3f;

    // GOOD: With range constraint
    [SerializeField, Range(0, 100)]
    private int _health = 100;

    // GOOD: Grouped with header
    [Header("Audio")]
    [SerializeField] private AudioClip _jumpSound;
    [SerializeField] private AudioClip _landSound;

    // AVOID: Public fields (no encapsulation)
    public float moveSpeed = 5f; // Anyone can modify

    // GOOD: Public property with private backing field
    public float MoveSpeed => _moveSpeed;
}
```

### What Can't Be Serialized

```csharp
// CANNOT serialize:
// - Interfaces
// - Properties
// - Static fields
// - Readonly fields
// - Dictionaries (use serializable dictionary class)
// - Multidimensional arrays
// - Nested generic types

// WORKAROUND: Custom serializable types
[Serializable]
public class SerializableDictionary<TKey, TValue> : ISerializationCallbackReceiver
{
    [SerializeField] private List<TKey> _keys = new();
    [SerializeField] private List<TValue> _values = new();

    private Dictionary<TKey, TValue> _dictionary = new();

    public void OnBeforeSerialize()
    {
        _keys.Clear();
        _values.Clear();
        foreach (var kvp in _dictionary)
        {
            _keys.Add(kvp.Key);
            _values.Add(kvp.Value);
        }
    }

    public void OnAfterDeserialize()
    {
        _dictionary.Clear();
        for (int i = 0; i < _keys.Count; i++)
        {
            _dictionary[_keys[i]] = _values[i];
        }
    }
}
```

### ScriptableObject Data

```csharp
// GOOD: Use ScriptableObjects for shared configuration
[CreateAssetMenu(fileName = "EnemyConfig", menuName = "Config/Enemy")]
public class EnemyConfig : ScriptableObject
{
    [Header("Stats")]
    public int maxHealth = 100;
    public float moveSpeed = 3f;
    public int damage = 10;

    [Header("Behavior")]
    public float detectionRange = 10f;
    public float attackRange = 2f;
    public float attackCooldown = 1f;
}

// Usage
public class Enemy : MonoBehaviour
{
    [SerializeField] private EnemyConfig _config;

    private void Awake()
    {
        _currentHealth = _config.maxHealth;
        _agent.speed = _config.moveSpeed;
    }
}
```

---

## Architecture Guidelines

### Single Responsibility

```csharp
// BAD: One class doing everything
public class Player : MonoBehaviour
{
    // Movement code...
    // Combat code...
    // Inventory code...
    // UI code...
    // Audio code...
    // 2000 lines later...
}

// GOOD: Separated concerns
public class Player : MonoBehaviour
{
    // Coordinates sub-components
    [SerializeField] private PlayerMovement _movement;
    [SerializeField] private PlayerCombat _combat;
    [SerializeField] private PlayerInventory _inventory;
}

public class PlayerMovement : MonoBehaviour
{
    // Only movement logic
}

public class PlayerCombat : MonoBehaviour
{
    // Only combat logic
}
```

### Dependency Injection (Simple)

```csharp
// GOOD: Constructor/method injection for non-MonoBehaviours
public class GameService
{
    private readonly ILogger _logger;
    private readonly IAnalytics _analytics;

    public GameService(ILogger logger, IAnalytics analytics)
    {
        _logger = logger;
        _analytics = analytics;
    }
}

// GOOD: Property/method injection for MonoBehaviours
public class Enemy : MonoBehaviour
{
    private IPlayerLocator _playerLocator;

    public void Initialize(IPlayerLocator playerLocator)
    {
        _playerLocator = playerLocator;
    }
}

// GOOD: ScriptableObject as injectable service
public class EnemySpawner : MonoBehaviour
{
    [SerializeField] private PlayerLocatorSO _playerLocator;

    private void SpawnEnemy()
    {
        var enemy = Instantiate(_enemyPrefab);
        enemy.Initialize(_playerLocator);
    }
}
```

### Event-Driven Communication

```csharp
// Pattern 1: C# events
public class Health : MonoBehaviour
{
    public event Action<int, int> OnHealthChanged; // current, max
    public event Action OnDeath;

    private void TakeDamage(int damage)
    {
        _current -= damage;
        OnHealthChanged?.Invoke(_current, _max);

        if (_current <= 0)
        {
            OnDeath?.Invoke();
        }
    }
}

// Pattern 2: ScriptableObject events (decoupled)
[CreateAssetMenu(menuName = "Events/Void Event")]
public class VoidEventSO : ScriptableObject
{
    private readonly List<Action> _listeners = new();

    public void Register(Action listener) => _listeners.Add(listener);
    public void Unregister(Action listener) => _listeners.Remove(listener);

    public void Raise()
    {
        for (int i = _listeners.Count - 1; i >= 0; i--)
        {
            _listeners[i]?.Invoke();
        }
    }
}

// Usage
public class GameManager : MonoBehaviour
{
    [SerializeField] private VoidEventSO _onGameOver;

    public void EndGame() => _onGameOver.Raise();
}

public class UIManager : MonoBehaviour
{
    [SerializeField] private VoidEventSO _onGameOver;

    private void OnEnable() => _onGameOver.Register(ShowGameOverScreen);
    private void OnDisable() => _onGameOver.Unregister(ShowGameOverScreen);
}
```

---

## Testing Guidelines

### Test Structure

```csharp
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;

namespace MyGame.Tests
{
    public class HealthTests
    {
        private Health _health;
        private GameObject _gameObject;

        [SetUp]
        public void SetUp()
        {
            _gameObject = new GameObject("TestHealth");
            _health = _gameObject.AddComponent<Health>();
            _health.Initialize(100);
        }

        [TearDown]
        public void TearDown()
        {
            Object.DestroyImmediate(_gameObject);
        }

        [Test]
        public void TakeDamage_ReducesHealth()
        {
            // Arrange
            int initialHealth = _health.Current;
            int damage = 25;

            // Act
            _health.TakeDamage(damage);

            // Assert
            Assert.AreEqual(initialHealth - damage, _health.Current);
        }

        [Test]
        public void TakeDamage_WhenFatal_InvokesDeath()
        {
            // Arrange
            bool deathInvoked = false;
            _health.OnDeath += () => deathInvoked = true;

            // Act
            _health.TakeDamage(100);

            // Assert
            Assert.IsTrue(deathInvoked);
        }
    }
}
```

### Testable Design

```csharp
// BAD: Hard to test (direct dependencies)
public class Enemy : MonoBehaviour
{
    private void Update()
    {
        var player = FindObjectOfType<Player>();
        if (player != null)
        {
            MoveTowards(player.transform.position);
        }
    }
}

// GOOD: Testable (injected dependencies)
public class Enemy : MonoBehaviour
{
    private ITarget _target;

    public void Initialize(ITarget target)
    {
        _target = target;
    }

    private void Update()
    {
        if (_target != null && _target.IsValid)
        {
            MoveTowards(_target.Position);
        }
    }
}

// In tests
public class MockTarget : ITarget
{
    public bool IsValid { get; set; } = true;
    public Vector3 Position { get; set; } = Vector3.zero;
}
```

---

## Appendix: Quick Reference Card

### Must Do
- Cache component references in Awake
- Unsubscribe from events in OnDisable
- Use object pooling for frequent instantiation
- Use NonAlloc physics methods
- Use TryGetComponent pattern
- Initialize all serialized references

### Must Not Do
- Allocate in Update/FixedUpdate
- Use Find methods in Update
- Use public fields for serialization
- Catch generic exceptions
- Use string comparison for tags
- Destroy in physics callbacks

### Prefer
- Structs for small, immutable data
- Events over SendMessage
- ScriptableObjects over singletons
- Composition over inheritance
- Explicit null checks over null-conditional
