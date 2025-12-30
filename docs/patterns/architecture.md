# Architecture Patterns

Patterns for structuring Unity projects with maintainable, testable, and decoupled code.

## ScriptableObject Architecture

### ScriptableObject Events

Decouple systems using ScriptableObject-based events. No direct references between systems required.

```csharp
// Base event type
[CreateAssetMenu(menuName = "Events/Void Event")]
public class VoidEventSO : ScriptableObject
{
    private readonly HashSet<Action> _listeners = new();

    public void Register(Action listener)
    {
        _listeners.Add(listener);
    }

    public void Unregister(Action listener)
    {
        _listeners.Remove(listener);
    }

    public void Raise()
    {
        foreach (var listener in _listeners)
        {
            listener?.Invoke();
        }
    }
}

// Generic typed event
public abstract class GameEventSO<T> : ScriptableObject
{
    private readonly HashSet<Action<T>> _listeners = new();

    public void Register(Action<T> listener) => _listeners.Add(listener);
    public void Unregister(Action<T> listener) => _listeners.Remove(listener);

    public void Raise(T value)
    {
        foreach (var listener in _listeners)
        {
            listener?.Invoke(value);
        }
    }
}

// Specific event types
[CreateAssetMenu(menuName = "Events/Int Event")]
public class IntEventSO : GameEventSO<int> { }

[CreateAssetMenu(menuName = "Events/Float Event")]
public class FloatEventSO : GameEventSO<float> { }

[CreateAssetMenu(menuName = "Events/Vector3 Event")]
public class Vector3EventSO : GameEventSO<Vector3> { }
```

**Usage:**

```csharp
// Publisher - doesn't know about subscribers
public class Player : MonoBehaviour
{
    [SerializeField] private IntEventSO _onHealthChanged;
    [SerializeField] private VoidEventSO _onPlayerDied;

    public void TakeDamage(int damage)
    {
        _health -= damage;
        _onHealthChanged.Raise(_health);

        if (_health <= 0)
        {
            _onPlayerDied.Raise();
        }
    }
}

// Subscriber - doesn't know about publisher
public class HealthUI : MonoBehaviour
{
    [SerializeField] private IntEventSO _onHealthChanged;
    [SerializeField] private TMP_Text _healthText;

    private void OnEnable() => _onHealthChanged.Register(UpdateDisplay);
    private void OnDisable() => _onHealthChanged.Unregister(UpdateDisplay);

    private void UpdateDisplay(int health)
    {
        _healthText.text = $"HP: {health}";
    }
}
```

### ScriptableObject Variables

Shared state without singletons. References are assigned in the Inspector.

```csharp
// Base variable type
public abstract class VariableSO<T> : ScriptableObject
{
    [SerializeField] protected T _initialValue;

    private T _runtimeValue;

    public T Value
    {
        get => _runtimeValue;
        set => _runtimeValue = value;
    }

    private void OnEnable()
    {
        _runtimeValue = _initialValue;
    }

    public static implicit operator T(VariableSO<T> variable) => variable.Value;
}

// Specific types
[CreateAssetMenu(menuName = "Variables/Int")]
public class IntVariableSO : VariableSO<int> { }

[CreateAssetMenu(menuName = "Variables/Float")]
public class FloatVariableSO : VariableSO<float> { }

[CreateAssetMenu(menuName = "Variables/Bool")]
public class BoolVariableSO : VariableSO<bool> { }
```

**Usage:**

```csharp
public class Player : MonoBehaviour
{
    [SerializeField] private IntVariableSO _playerHealth;
    [SerializeField] private IntVariableSO _playerScore;

    public void AddScore(int amount)
    {
        _playerScore.Value += amount;
    }
}

public class ScoreUI : MonoBehaviour
{
    [SerializeField] private IntVariableSO _playerScore;
    [SerializeField] private TMP_Text _scoreText;

    private void Update()
    {
        // Both reference the same ScriptableObject
        _scoreText.text = $"Score: {_playerScore.Value}";
    }
}
```

### ScriptableObject Runtime Sets

Track active objects without FindObjectsOfType.

```csharp
public abstract class RuntimeSetSO<T> : ScriptableObject
{
    private readonly List<T> _items = new();

    public IReadOnlyList<T> Items => _items;
    public int Count => _items.Count;

    public void Add(T item)
    {
        if (!_items.Contains(item))
        {
            _items.Add(item);
        }
    }

    public void Remove(T item)
    {
        _items.Remove(item);
    }

    public void Clear()
    {
        _items.Clear();
    }
}

[CreateAssetMenu(menuName = "Runtime Sets/Enemy Set")]
public class EnemyRuntimeSetSO : RuntimeSetSO<Enemy> { }
```

**Usage:**

```csharp
public class Enemy : MonoBehaviour
{
    [SerializeField] private EnemyRuntimeSetSO _enemySet;

    private void OnEnable() => _enemySet.Add(this);
    private void OnDisable() => _enemySet.Remove(this);
}

public class EnemyManager : MonoBehaviour
{
    [SerializeField] private EnemyRuntimeSetSO _enemies;

    public Enemy GetNearestEnemy(Vector3 position)
    {
        Enemy nearest = null;
        float nearestDist = float.MaxValue;

        foreach (var enemy in _enemies.Items)
        {
            float dist = Vector3.Distance(position, enemy.transform.position);
            if (dist < nearestDist)
            {
                nearest = enemy;
                nearestDist = dist;
            }
        }

        return nearest;
    }
}
```

---

## State Machine

### Interface-Based State Machine

```csharp
public interface IState
{
    void Enter();
    void Update();
    void FixedUpdate();
    void Exit();
}

public class StateMachine
{
    private IState _currentState;
    private readonly Dictionary<Type, IState> _states = new();

    public IState CurrentState => _currentState;

    public void AddState(IState state)
    {
        _states[state.GetType()] = state;
    }

    public void SetState<T>() where T : IState
    {
        if (!_states.TryGetValue(typeof(T), out var newState))
        {
            Debug.LogError($"State {typeof(T).Name} not found");
            return;
        }

        _currentState?.Exit();
        _currentState = newState;
        _currentState.Enter();
    }

    public void Update() => _currentState?.Update();
    public void FixedUpdate() => _currentState?.FixedUpdate();
}
```

### Player State Example

```csharp
public class PlayerController : MonoBehaviour
{
    private StateMachine _stateMachine;

    private void Awake()
    {
        _stateMachine = new StateMachine();

        // Create states with reference to controller
        _stateMachine.AddState(new IdleState(this));
        _stateMachine.AddState(new WalkState(this));
        _stateMachine.AddState(new JumpState(this));
        _stateMachine.AddState(new FallState(this));

        _stateMachine.SetState<IdleState>();
    }

    private void Update() => _stateMachine.Update();
    private void FixedUpdate() => _stateMachine.FixedUpdate();

    // State can access controller methods
    public void SwitchToState<T>() where T : IState => _stateMachine.SetState<T>();
}

public class IdleState : IState
{
    private readonly PlayerController _controller;

    public IdleState(PlayerController controller)
    {
        _controller = controller;
    }

    public void Enter()
    {
        // Play idle animation
    }

    public void Update()
    {
        if (Input.GetAxis("Horizontal") != 0)
        {
            _controller.SwitchToState<WalkState>();
        }
        else if (Input.GetButtonDown("Jump"))
        {
            _controller.SwitchToState<JumpState>();
        }
    }

    public void FixedUpdate() { }
    public void Exit() { }
}
```

### ScriptableObject-Based States

For data-driven state machines with Inspector configuration.

```csharp
public abstract class StateSO : ScriptableObject
{
    public abstract void Enter(StateMachineController controller);
    public abstract void Update(StateMachineController controller);
    public abstract void Exit(StateMachineController controller);
}

[CreateAssetMenu(menuName = "States/Patrol State")]
public class PatrolStateSO : StateSO
{
    [SerializeField] private float _moveSpeed = 3f;
    [SerializeField] private float _waitTime = 2f;

    public override void Enter(StateMachineController controller)
    {
        // Setup patrol
    }

    public override void Update(StateMachineController controller)
    {
        // Patrol logic
    }

    public override void Exit(StateMachineController controller)
    {
        // Cleanup
    }
}

public class StateMachineController : MonoBehaviour
{
    [SerializeField] private StateSO _initialState;

    private StateSO _currentState;

    private void Start()
    {
        TransitionTo(_initialState);
    }

    private void Update()
    {
        _currentState?.Update(this);
    }

    public void TransitionTo(StateSO newState)
    {
        _currentState?.Exit(this);
        _currentState = newState;
        _currentState?.Enter(this);
    }
}
```

---

## Service Locator

A lightweight alternative to full dependency injection.

```csharp
public static class ServiceLocator
{
    private static readonly Dictionary<Type, object> _services = new();

    public static void Register<T>(T service) where T : class
    {
        _services[typeof(T)] = service;
    }

    public static T Get<T>() where T : class
    {
        if (_services.TryGetValue(typeof(T), out var service))
        {
            return (T)service;
        }

        Debug.LogError($"Service {typeof(T).Name} not registered");
        return null;
    }

    public static bool TryGet<T>(out T service) where T : class
    {
        if (_services.TryGetValue(typeof(T), out var obj))
        {
            service = (T)obj;
            return true;
        }

        service = null;
        return false;
    }

    public static void Unregister<T>() where T : class
    {
        _services.Remove(typeof(T));
    }

    public static void Clear()
    {
        _services.Clear();
    }
}
```

**Usage:**

```csharp
// Bootstrapper registers services
public class GameBootstrap : MonoBehaviour
{
    [SerializeField] private AudioManager _audioManager;
    [SerializeField] private SaveManager _saveManager;

    private void Awake()
    {
        ServiceLocator.Register<IAudioService>(_audioManager);
        ServiceLocator.Register<ISaveService>(_saveManager);
    }

    private void OnDestroy()
    {
        ServiceLocator.Clear();
    }
}

// Consumers get services
public class Player : MonoBehaviour
{
    private IAudioService _audio;

    private void Start()
    {
        _audio = ServiceLocator.Get<IAudioService>();
    }

    public void PlayJumpSound()
    {
        _audio.PlaySFX("jump");
    }
}
```

---

## Command Pattern

For input handling, undo/redo, and replay systems.

```csharp
public interface ICommand
{
    void Execute();
    void Undo();
}

public class CommandInvoker
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();

    public void Execute(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);
        _redoStack.Clear();
    }

    public void Undo()
    {
        if (_undoStack.Count == 0) return;

        var command = _undoStack.Pop();
        command.Undo();
        _redoStack.Push(command);
    }

    public void Redo()
    {
        if (_redoStack.Count == 0) return;

        var command = _redoStack.Pop();
        command.Execute();
        _undoStack.Push(command);
    }
}
```

**Example Commands:**

```csharp
public class MoveCommand : ICommand
{
    private readonly Transform _target;
    private readonly Vector3 _direction;
    private readonly float _distance;
    private Vector3 _previousPosition;

    public MoveCommand(Transform target, Vector3 direction, float distance)
    {
        _target = target;
        _direction = direction.normalized;
        _distance = distance;
    }

    public void Execute()
    {
        _previousPosition = _target.position;
        _target.position += _direction * _distance;
    }

    public void Undo()
    {
        _target.position = _previousPosition;
    }
}

public class PlaceObjectCommand : ICommand
{
    private readonly GameObject _prefab;
    private readonly Vector3 _position;
    private readonly Quaternion _rotation;
    private GameObject _instance;

    public PlaceObjectCommand(GameObject prefab, Vector3 position, Quaternion rotation)
    {
        _prefab = prefab;
        _position = position;
        _rotation = rotation;
    }

    public void Execute()
    {
        _instance = Object.Instantiate(_prefab, _position, _rotation);
    }

    public void Undo()
    {
        if (_instance != null)
        {
            Object.Destroy(_instance);
        }
    }
}
```

---

## Observer Pattern (Unity-Style)

### UnityEvent Approach

Good for designer-configurable callbacks.

```csharp
using UnityEngine.Events;

public class Health : MonoBehaviour
{
    [SerializeField] private int _maxHealth = 100;

    [Header("Events")]
    [SerializeField] private UnityEvent<int> _onHealthChanged;
    [SerializeField] private UnityEvent _onDeath;

    private int _currentHealth;

    private void Awake()
    {
        _currentHealth = _maxHealth;
    }

    public void TakeDamage(int damage)
    {
        _currentHealth = Mathf.Max(0, _currentHealth - damage);
        _onHealthChanged?.Invoke(_currentHealth);

        if (_currentHealth == 0)
        {
            _onDeath?.Invoke();
        }
    }
}
```

### C# Events Approach

Better performance, code-only.

```csharp
public class Health : MonoBehaviour
{
    public event Action<int, int> OnHealthChanged; // current, max
    public event Action OnDeath;

    private int _maxHealth;
    private int _currentHealth;

    public void TakeDamage(int damage)
    {
        _currentHealth = Mathf.Max(0, _currentHealth - damage);
        OnHealthChanged?.Invoke(_currentHealth, _maxHealth);

        if (_currentHealth == 0)
        {
            OnDeath?.Invoke();
        }
    }
}

// Subscriber
public class HealthBar : MonoBehaviour
{
    [SerializeField] private Health _health;
    [SerializeField] private Image _fillImage;

    private void OnEnable()
    {
        _health.OnHealthChanged += UpdateBar;
    }

    private void OnDisable()
    {
        _health.OnHealthChanged -= UpdateBar;
    }

    private void UpdateBar(int current, int max)
    {
        _fillImage.fillAmount = (float)current / max;
    }
}
```

---

## Composition Over Inheritance

### Component-Based Abilities

```csharp
// Base interface
public interface IAbility
{
    string Name { get; }
    float Cooldown { get; }
    bool CanActivate { get; }
    void Activate();
}

// Ability implementations as components
public class DashAbility : MonoBehaviour, IAbility
{
    [SerializeField] private float _dashDistance = 5f;
    [SerializeField] private float _cooldown = 1f;

    private float _lastActivateTime;

    public string Name => "Dash";
    public float Cooldown => _cooldown;
    public bool CanActivate => Time.time - _lastActivateTime >= _cooldown;

    public void Activate()
    {
        if (!CanActivate) return;

        transform.position += transform.forward * _dashDistance;
        _lastActivateTime = Time.time;
    }
}

public class FireballAbility : MonoBehaviour, IAbility
{
    [SerializeField] private GameObject _fireballPrefab;
    [SerializeField] private float _cooldown = 2f;

    private float _lastActivateTime;

    public string Name => "Fireball";
    public float Cooldown => _cooldown;
    public bool CanActivate => Time.time - _lastActivateTime >= _cooldown;

    public void Activate()
    {
        if (!CanActivate) return;

        Instantiate(_fireballPrefab, transform.position + transform.forward, transform.rotation);
        _lastActivateTime = Time.time;
    }
}

// Ability manager discovers abilities on the same GameObject
public class AbilityManager : MonoBehaviour
{
    private IAbility[] _abilities;

    private void Awake()
    {
        _abilities = GetComponents<IAbility>();
    }

    public void ActivateAbility(int index)
    {
        if (index >= 0 && index < _abilities.Length)
        {
            _abilities[index].Activate();
        }
    }
}
```

### Decorator Pattern for Modifiers

```csharp
public interface IDamageModifier
{
    int ModifyDamage(int baseDamage, DamageContext context);
}

public class CriticalHitModifier : MonoBehaviour, IDamageModifier
{
    [SerializeField] private float _critChance = 0.1f;
    [SerializeField] private float _critMultiplier = 2f;

    public int ModifyDamage(int baseDamage, DamageContext context)
    {
        if (Random.value < _critChance)
        {
            context.IsCritical = true;
            return Mathf.RoundToInt(baseDamage * _critMultiplier);
        }
        return baseDamage;
    }
}

public class ElementalDamageModifier : MonoBehaviour, IDamageModifier
{
    [SerializeField] private ElementType _element;
    [SerializeField] private int _bonusDamage = 5;

    public int ModifyDamage(int baseDamage, DamageContext context)
    {
        context.Element = _element;
        return baseDamage + _bonusDamage;
    }
}

public class Weapon : MonoBehaviour
{
    [SerializeField] private int _baseDamage = 10;

    private IDamageModifier[] _modifiers;

    private void Awake()
    {
        _modifiers = GetComponents<IDamageModifier>();
    }

    public int CalculateDamage()
    {
        var context = new DamageContext();
        int damage = _baseDamage;

        foreach (var modifier in _modifiers)
        {
            damage = modifier.ModifyDamage(damage, context);
        }

        return damage;
    }
}
```

---

## Dependency Injection (Manual)

### Constructor Injection for Plain Classes

```csharp
// Service interface
public interface IEnemySpawner
{
    Enemy SpawnEnemy(Vector3 position);
}

// Implementation
public class EnemySpawner : IEnemySpawner
{
    private readonly EnemyFactory _factory;
    private readonly EnemyRuntimeSetSO _enemySet;

    public EnemySpawner(EnemyFactory factory, EnemyRuntimeSetSO enemySet)
    {
        _factory = factory;
        _enemySet = enemySet;
    }

    public Enemy SpawnEnemy(Vector3 position)
    {
        var enemy = _factory.Create(position);
        _enemySet.Add(enemy);
        return enemy;
    }
}

// Composition root
public class GameInstaller : MonoBehaviour
{
    [SerializeField] private EnemyRuntimeSetSO _enemySet;
    [SerializeField] private EnemyFactory _enemyFactory;

    private void Awake()
    {
        var spawner = new EnemySpawner(_enemyFactory, _enemySet);
        ServiceLocator.Register<IEnemySpawner>(spawner);
    }
}
```

### Method Injection for MonoBehaviours

```csharp
public class Enemy : MonoBehaviour
{
    private IPlayerLocator _playerLocator;
    private IObjectPool<Enemy> _pool;

    // Called after instantiation
    public void Initialize(IPlayerLocator playerLocator, IObjectPool<Enemy> pool)
    {
        _playerLocator = playerLocator;
        _pool = pool;
    }

    private void Update()
    {
        if (_playerLocator.TryGetPlayer(out var player))
        {
            MoveTowards(player.Position);
        }
    }

    public void Die()
    {
        _pool.Return(this);
    }
}

// Factory handles dependency injection
public class EnemyFactory : MonoBehaviour
{
    [SerializeField] private Enemy _prefab;
    [SerializeField] private PlayerLocatorSO _playerLocator;

    private ObjectPool<Enemy> _pool;

    private void Awake()
    {
        _pool = new ObjectPool<Enemy>(_prefab, transform);
    }

    public Enemy Create(Vector3 position)
    {
        var enemy = _pool.Get();
        enemy.transform.position = position;
        enemy.Initialize(_playerLocator, _pool);
        return enemy;
    }
}
```

---

## Scene Management

### Scene Loading Controller

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.SceneManagement;

public class SceneController : MonoBehaviour
{
    public static SceneController Instance { get; private set; }

    [SerializeField] private CanvasGroup _fadeCanvasGroup;
    [SerializeField] private float _fadeDuration = 0.5f;

    private bool _isLoading;

    private void Awake()
    {
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }

        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    public void LoadScene(string sceneName)
    {
        if (_isLoading) return;
        StartCoroutine(LoadSceneRoutine(sceneName));
    }

    private IEnumerator LoadSceneRoutine(string sceneName)
    {
        _isLoading = true;

        // Fade out
        yield return FadeRoutine(1f);

        // Load scene async
        var operation = SceneManager.LoadSceneAsync(sceneName);
        while (!operation.isDone)
        {
            yield return null;
        }

        // Fade in
        yield return FadeRoutine(0f);

        _isLoading = false;
    }

    private IEnumerator FadeRoutine(float targetAlpha)
    {
        float startAlpha = _fadeCanvasGroup.alpha;
        float elapsed = 0f;

        while (elapsed < _fadeDuration)
        {
            elapsed += Time.unscaledDeltaTime;
            _fadeCanvasGroup.alpha = Mathf.Lerp(startAlpha, targetAlpha, elapsed / _fadeDuration);
            yield return null;
        }

        _fadeCanvasGroup.alpha = targetAlpha;
    }
}
```

### Additive Scene Loading

```csharp
public class AdditiveSceneLoader : MonoBehaviour
{
    [SerializeField] private string[] _additiveScenes;

    private readonly List<AsyncOperation> _loadOperations = new();

    private void Start()
    {
        foreach (var sceneName in _additiveScenes)
        {
            var operation = SceneManager.LoadSceneAsync(sceneName, LoadSceneMode.Additive);
            _loadOperations.Add(operation);
        }
    }

    public void UnloadAdditiveScenes()
    {
        foreach (var sceneName in _additiveScenes)
        {
            SceneManager.UnloadSceneAsync(sceneName);
        }
    }
}
```
