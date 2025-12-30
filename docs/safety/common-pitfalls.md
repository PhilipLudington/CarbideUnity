# Common Pitfalls and Code Safety

Patterns for avoiding bugs, null references, memory leaks, and other common Unity issues.

## Null Reference Prevention

### The Unity Null Problem

Unity overloads the `==` operator for `UnityEngine.Object`. A destroyed object is `== null` but not `is null`.

```csharp
// DANGEROUS: "is null" bypasses Unity's null check
if (target is null)  // May be true even for destroyed objects
{
    // This can access a destroyed object!
}

// SAFE: Use == for Unity objects
if (target == null)
{
    // Correctly detects both null and destroyed
}

// SAFE: Use the bool operator
if (!target)
{
    // Same as == null for Unity objects
}
```

### Required Reference Validation

```csharp
public class ValidatedBehaviour : MonoBehaviour
{
    [SerializeField] private Transform _targetTransform;
    [SerializeField] private AudioClip _soundEffect;
    [SerializeField] private ParticleSystem _particles;

    private void Awake()
    {
        ValidateReferences();
    }

    private void ValidateReferences()
    {
        var errors = new List<string>();

        if (_targetTransform == null)
            errors.Add("Target Transform is not assigned");

        if (_soundEffect == null)
            errors.Add("Sound Effect is not assigned");

        if (_particles == null)
            errors.Add("Particles is not assigned");

        if (errors.Count > 0)
        {
            foreach (var error in errors)
            {
                Debug.LogError($"{name}: {error}", this);
            }

            // Disable the component to prevent runtime errors
            enabled = false;
        }
    }

    #if UNITY_EDITOR
    private void OnValidate()
    {
        // Warn in editor when references are missing
        if (_targetTransform == null)
            Debug.LogWarning($"{name}: Target Transform should be assigned", this);
    }
    #endif
}
```

### Safe Access Patterns

```csharp
// Pattern 1: TryGetComponent
private void OnTriggerEnter(Collider other)
{
    // GOOD: TryGetComponent returns false if not found
    if (other.TryGetComponent<IDamageable>(out var damageable))
    {
        damageable.TakeDamage(_damage);
    }

    // BAD: GetComponent returns null, requires separate check
    var component = other.GetComponent<IDamageable>();
    if (component != null)
    {
        component.TakeDamage(_damage);
    }
}

// Pattern 2: Guard clauses
public void Attack(GameObject target)
{
    if (target == null)
    {
        Debug.LogWarning($"{name}: Cannot attack null target");
        return;
    }

    if (!target.TryGetComponent<Health>(out var health))
    {
        Debug.LogWarning($"{name}: Target {target.name} has no Health component");
        return;
    }

    if (!health.IsAlive)
    {
        return; // Silent return - target already dead is not an error
    }

    health.TakeDamage(_attackDamage);
}

// Pattern 3: Cached reference with validity check
private Transform _cachedTarget;

private void Update()
{
    // Check if cached reference is still valid
    if (_cachedTarget == null)
    {
        _cachedTarget = FindNewTarget();
        if (_cachedTarget == null) return;
    }

    MoveTowards(_cachedTarget.position);
}
```

---

## Event Subscription Leaks

### The Problem

Forgetting to unsubscribe from events causes:
- Memory leaks (objects can't be garbage collected)
- Null reference exceptions (handlers called on destroyed objects)
- Unexpected behavior (handlers called multiple times)

### Safe Event Subscription

```csharp
public class SafeEventSubscriber : MonoBehaviour
{
    [SerializeField] private Health _playerHealth;
    [SerializeField] private VoidEventSO _gameOverEvent;

    // Cache delegates for removal
    private Action<int> _healthChangedHandler;
    private Action _gameOverHandler;

    private void Awake()
    {
        // Create handlers once
        _healthChangedHandler = OnHealthChanged;
        _gameOverHandler = OnGameOver;
    }

    private void OnEnable()
    {
        // Subscribe when enabled
        if (_playerHealth != null)
        {
            _playerHealth.OnHealthChanged += _healthChangedHandler;
        }

        if (_gameOverEvent != null)
        {
            _gameOverEvent.Register(_gameOverHandler);
        }
    }

    private void OnDisable()
    {
        // ALWAYS unsubscribe when disabled
        if (_playerHealth != null)
        {
            _playerHealth.OnHealthChanged -= _healthChangedHandler;
        }

        if (_gameOverEvent != null)
        {
            _gameOverEvent.Unregister(_gameOverHandler);
        }
    }

    private void OnHealthChanged(int health)
    {
        // Handle event
    }

    private void OnGameOver()
    {
        // Handle event
    }
}
```

### Static Event Dangers

```csharp
// DANGEROUS: Static events persist across scene loads
public static class GameEvents
{
    public static event Action OnGameOver;

    // REQUIRED: Clear static events on scene unload
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void ClearEvents()
    {
        OnGameOver = null;
    }
}

// SAFER: ScriptableObject events with automatic cleanup
[CreateAssetMenu(menuName = "Events/Game Event")]
public class GameEventSO : ScriptableObject
{
    private readonly HashSet<Action> _listeners = new();

    public void Register(Action listener) => _listeners.Add(listener);
    public void Unregister(Action listener) => _listeners.Remove(listener);

    public void Raise()
    {
        foreach (var listener in _listeners)
        {
            listener?.Invoke();
        }
    }

    // Clear on domain reload (entering play mode)
    private void OnEnable()
    {
        _listeners.Clear();
    }
}
```

---

## Coroutine Safety

### Stopping Coroutines Properly

```csharp
public class SafeCoroutines : MonoBehaviour
{
    private Coroutine _activeRoutine;
    private bool _isRunning;

    public void StartAction()
    {
        // Always stop existing before starting new
        StopAction();
        _activeRoutine = StartCoroutine(ActionRoutine());
    }

    public void StopAction()
    {
        if (_activeRoutine != null)
        {
            StopCoroutine(_activeRoutine);
            _activeRoutine = null;
        }
        _isRunning = false;
    }

    private void OnDisable()
    {
        // Coroutines stop automatically on disable,
        // but we need to clean up our state
        _activeRoutine = null;
        _isRunning = false;
    }

    private IEnumerator ActionRoutine()
    {
        _isRunning = true;

        try
        {
            while (_isRunning)
            {
                // Do work
                yield return null;
            }
        }
        finally
        {
            // Cleanup runs even if coroutine is stopped
            _activeRoutine = null;
            _isRunning = false;
        }
    }
}
```

### Coroutine on Destroyed Object

```csharp
// DANGEROUS: Coroutine continues after object is destroyed
public class DangerousCoroutine : MonoBehaviour
{
    private IEnumerator Start()
    {
        yield return new WaitForSeconds(5f);
        // If object is destroyed during wait, this crashes
        transform.position = Vector3.zero;
    }
}

// SAFE: Check for destruction
public class SafeCoroutine : MonoBehaviour
{
    private IEnumerator Start()
    {
        yield return new WaitForSeconds(5f);

        // Check if we're still valid
        if (this == null) yield break;

        transform.position = Vector3.zero;
    }
}

// SAFER: Use cancellation
public class CancellableCoroutine : MonoBehaviour
{
    private CancellationTokenSource _cts;

    private async void Start()
    {
        _cts = new CancellationTokenSource();

        try
        {
            await Task.Delay(5000, _cts.Token);
            transform.position = Vector3.zero;
        }
        catch (OperationCanceledException)
        {
            // Expected when cancelled
        }
    }

    private void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

---

## Component Lifecycle Issues

### Accessing Components Before Initialization

```csharp
// DANGEROUS: Other scripts might access before Awake
public class Player : MonoBehaviour
{
    public Health Health { get; private set; }

    private void Awake()
    {
        Health = GetComponent<Health>();
    }
}

// SAFE: Lazy initialization with validation
public class Player : MonoBehaviour
{
    private Health _health;

    public Health Health
    {
        get
        {
            if (_health == null)
            {
                _health = GetComponent<Health>();
            }
            return _health;
        }
    }
}

// SAFER: Initialize in Awake, validate in Start
public class Player : MonoBehaviour
{
    private Health _health;

    public Health Health => _health;

    private void Awake()
    {
        _health = GetComponent<Health>();
    }

    private void Start()
    {
        // By Start, all Awake methods have run
        Debug.Assert(_health != null, "Health component missing");
    }
}
```

### Script Execution Order

```csharp
// Use [DefaultExecutionOrder] for critical initialization order
[DefaultExecutionOrder(-100)] // Runs before default (0)
public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }

    private void Awake()
    {
        Instance = this;
    }
}

[DefaultExecutionOrder(-50)]
public class AudioManager : MonoBehaviour
{
    // Guaranteed to run after GameManager
    private void Awake()
    {
        // Safe to access GameManager.Instance
    }
}

// Default scripts have order 0
public class Player : MonoBehaviour
{
    private void Awake()
    {
        // Both managers are initialized
    }
}
```

---

## Serialization Pitfalls

### Serialization Callbacks

```csharp
public class SerializationSafe : MonoBehaviour, ISerializationCallbackReceiver
{
    // Serializable data
    [SerializeField] private List<string> _keys = new();
    [SerializeField] private List<int> _values = new();

    // Runtime data (not serialized)
    private Dictionary<string, int> _dictionary;

    public void OnBeforeSerialize()
    {
        // Convert runtime data to serializable format
        _keys.Clear();
        _values.Clear();

        if (_dictionary == null) return;

        foreach (var kvp in _dictionary)
        {
            _keys.Add(kvp.Key);
            _values.Add(kvp.Value);
        }
    }

    public void OnAfterDeserialize()
    {
        // Reconstruct runtime data from serialized
        _dictionary = new Dictionary<string, int>();

        int count = Mathf.Min(_keys.Count, _values.Count);
        for (int i = 0; i < count; i++)
        {
            _dictionary[_keys[i]] = _values[i];
        }
    }
}
```

### Prefab Reference Issues

```csharp
public class PrefabSafe : MonoBehaviour
{
    [SerializeField] private GameObject _prefab;

    // DANGEROUS: Modifying prefab at runtime
    private void BadModifyPrefab()
    {
        // This modifies the PREFAB ASSET, not an instance!
        _prefab.GetComponent<Health>().MaxHealth = 200;
    }

    // SAFE: Modify the instance
    private void SafeModifyInstance()
    {
        var instance = Instantiate(_prefab);
        instance.GetComponent<Health>().MaxHealth = 200;
    }

    // SAFE: Check if it's a prefab or instance
    private void SafeModify(GameObject target)
    {
        #if UNITY_EDITOR
        if (UnityEditor.PrefabUtility.IsPartOfPrefabAsset(target))
        {
            Debug.LogError("Cannot modify prefab asset at runtime");
            return;
        }
        #endif

        target.GetComponent<Health>().MaxHealth = 200;
    }
}
```

---

## Physics Callback Safety

### Don't Destroy in Physics Callbacks

```csharp
// DANGEROUS: Destroying in physics callback
private void OnCollisionEnter(Collision collision)
{
    Destroy(gameObject); // Can cause physics engine issues
}

// SAFE: Delay destruction
private void OnCollisionEnter(Collision collision)
{
    // Disable immediately to prevent further collisions
    gameObject.SetActive(false);

    // Destroy after physics step completes
    Destroy(gameObject, 0.02f);
}

// SAFER: Use pooling instead of destroy
private void OnCollisionEnter(Collision collision)
{
    _pool.Return(this);
}
```

### Physics State Modification

```csharp
// DANGEROUS: Modifying rigidbody in Update
private void Update()
{
    _rigidbody.velocity = _targetVelocity; // Inconsistent with physics
}

// SAFE: Use FixedUpdate for physics
private void FixedUpdate()
{
    _rigidbody.velocity = _targetVelocity;
}

// SAFE: Teleporting with physics
public void Teleport(Vector3 position)
{
    // Disable physics temporarily
    _rigidbody.isKinematic = true;

    // Move instantly
    transform.position = position;

    // Re-enable physics
    _rigidbody.isKinematic = false;

    // Reset velocity if needed
    _rigidbody.linearVelocity = Vector3.zero;
    _rigidbody.angularVelocity = Vector3.zero;
}
```

---

## Threading Issues

### Main Thread Only

```csharp
// DANGEROUS: Unity API from background thread
private async Task LoadDataAsync()
{
    var data = await FetchFromServer();

    // CRASHES: Unity API on background thread
    transform.position = data.position;
}

// SAFE: Return to main thread
private async Task LoadDataAsync()
{
    var data = await FetchFromServer();

    // Return to main thread (requires UniTask or custom dispatcher)
    await UniTask.SwitchToMainThread();

    transform.position = data.position;
}

// SAFE: Use Unity's main thread dispatcher pattern
public class MainThreadDispatcher : MonoBehaviour
{
    private static readonly Queue<Action> _actions = new();
    private static MainThreadDispatcher _instance;

    private void Awake()
    {
        _instance = this;
    }

    public static void Enqueue(Action action)
    {
        lock (_actions)
        {
            _actions.Enqueue(action);
        }
    }

    private void Update()
    {
        lock (_actions)
        {
            while (_actions.Count > 0)
            {
                _actions.Dequeue()?.Invoke();
            }
        }
    }
}

// Usage
private async Task LoadDataAsync()
{
    var data = await FetchFromServer();

    MainThreadDispatcher.Enqueue(() =>
    {
        transform.position = data.position;
    });
}
```

---

## Resource Cleanup

### Disposable Resources

```csharp
public class ResourceCleanup : MonoBehaviour
{
    private RenderTexture _renderTexture;
    private ComputeBuffer _computeBuffer;
    private NativeArray<float> _nativeArray;

    private void Awake()
    {
        _renderTexture = new RenderTexture(256, 256, 0);
        _computeBuffer = new ComputeBuffer(1000, sizeof(float));
        _nativeArray = new NativeArray<float>(1000, Allocator.Persistent);
    }

    private void OnDestroy()
    {
        // MUST release manually - not garbage collected

        if (_renderTexture != null)
        {
            _renderTexture.Release();
            Destroy(_renderTexture);
        }

        if (_computeBuffer != null)
        {
            _computeBuffer.Release();
            _computeBuffer.Dispose();
        }

        if (_nativeArray.IsCreated)
        {
            _nativeArray.Dispose();
        }
    }
}
```

### AsyncOperation Cleanup

```csharp
public class AsyncCleanup : MonoBehaviour
{
    private AsyncOperation _loadOperation;
    private bool _isLoading;

    public void LoadScene(string sceneName)
    {
        if (_isLoading) return;

        _isLoading = true;
        _loadOperation = SceneManager.LoadSceneAsync(sceneName);
        _loadOperation.completed += OnLoadComplete;
    }

    private void OnLoadComplete(AsyncOperation operation)
    {
        operation.completed -= OnLoadComplete;
        _isLoading = false;
    }

    private void OnDestroy()
    {
        // Clean up if destroyed while loading
        if (_loadOperation != null)
        {
            _loadOperation.completed -= OnLoadComplete;
        }
    }
}
```

---

## Common Anti-Patterns

### String Comparisons

```csharp
// BAD: String tag comparison
private void OnTriggerEnter(Collider other)
{
    if (other.tag == "Enemy")  // String comparison every time
    {
        // Handle enemy
    }
}

// GOOD: CompareTag (optimized, no allocation)
private void OnTriggerEnter(Collider other)
{
    if (other.CompareTag("Enemy"))
    {
        // Handle enemy
    }
}

// BETTER: Component check
private void OnTriggerEnter(Collider other)
{
    if (other.TryGetComponent<Enemy>(out var enemy))
    {
        // Handle enemy with full type safety
    }
}

// BEST: Layer-based filtering (cheapest)
[SerializeField] private LayerMask _enemyLayer;

private void OnTriggerEnter(Collider other)
{
    if ((_enemyLayer.value & (1 << other.gameObject.layer)) != 0)
    {
        // Handle enemy
    }
}
```

### Camera.main Caching

```csharp
// BAD: Finds camera every frame
private void Update()
{
    Vector3 screenPos = Camera.main.WorldToScreenPoint(transform.position);
}

// GOOD: Cache on start
private Camera _mainCamera;

private void Awake()
{
    _mainCamera = Camera.main;
}

private void Update()
{
    if (_mainCamera == null) return;
    Vector3 screenPos = _mainCamera.WorldToScreenPoint(transform.position);
}
```

### Empty Unity Callbacks

```csharp
// BAD: Empty callbacks still have overhead
private void Update() { }
private void FixedUpdate() { }
private void LateUpdate() { }

// GOOD: Remove empty callbacks entirely
// (just delete the methods)

// If you need conditional updates, use enable/disable
private void Update()
{
    if (!_needsUpdate) return;
    // Do work
}

// Or use a manager pattern to batch updates
```
