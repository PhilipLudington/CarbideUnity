# Performance Patterns

Patterns for writing high-performance Unity code that avoids garbage collection and maintains consistent frame rates.

## Object Pooling

### Basic Pool

```csharp
using System.Collections.Generic;
using UnityEngine;

public class ObjectPool<T> where T : Component
{
    private readonly T _prefab;
    private readonly Transform _parent;
    private readonly Queue<T> _available = new();
    private readonly HashSet<T> _inUse = new();

    public ObjectPool(T prefab, Transform parent, int initialSize = 10)
    {
        _prefab = prefab;
        _parent = parent;

        for (int i = 0; i < initialSize; i++)
        {
            CreateInstance();
        }
    }

    public T Get()
    {
        T instance = _available.Count > 0
            ? _available.Dequeue()
            : CreateInstance();

        instance.gameObject.SetActive(true);
        _inUse.Add(instance);
        return instance;
    }

    public void Return(T instance)
    {
        if (!_inUse.Contains(instance)) return;

        instance.gameObject.SetActive(false);
        _inUse.Remove(instance);
        _available.Enqueue(instance);
    }

    public void ReturnAll()
    {
        foreach (var instance in _inUse)
        {
            instance.gameObject.SetActive(false);
            _available.Enqueue(instance);
        }
        _inUse.Clear();
    }

    private T CreateInstance()
    {
        var instance = Object.Instantiate(_prefab, _parent);
        instance.gameObject.SetActive(false);
        _available.Enqueue(instance);
        return instance;
    }
}
```

### Pooled Object Interface

```csharp
public interface IPoolable
{
    void OnSpawn();
    void OnDespawn();
}

public class PooledBullet : MonoBehaviour, IPoolable
{
    private ObjectPool<PooledBullet> _pool;
    private float _lifetime;
    private float _spawnTime;

    public void Initialize(ObjectPool<PooledBullet> pool)
    {
        _pool = pool;
    }

    public void OnSpawn()
    {
        _spawnTime = Time.time;
    }

    public void OnDespawn()
    {
        // Reset state for reuse
    }

    private void Update()
    {
        if (Time.time - _spawnTime > _lifetime)
        {
            _pool.Return(this);
        }
    }
}
```

### Pool Manager (Centralized)

```csharp
using System.Collections.Generic;
using UnityEngine;

public class PoolManager : MonoBehaviour
{
    public static PoolManager Instance { get; private set; }

    [System.Serializable]
    public class PoolConfig
    {
        public string id;
        public GameObject prefab;
        public int initialSize = 10;
    }

    [SerializeField] private PoolConfig[] _poolConfigs;

    private readonly Dictionary<string, Queue<GameObject>> _pools = new();
    private readonly Dictionary<string, GameObject> _prefabs = new();

    private void Awake()
    {
        Instance = this;
        InitializePools();
    }

    private void InitializePools()
    {
        foreach (var config in _poolConfigs)
        {
            var pool = new Queue<GameObject>();
            _prefabs[config.id] = config.prefab;

            for (int i = 0; i < config.initialSize; i++)
            {
                var obj = CreateInstance(config.prefab);
                pool.Enqueue(obj);
            }

            _pools[config.id] = pool;
        }
    }

    public GameObject Get(string id, Vector3 position, Quaternion rotation)
    {
        if (!_pools.TryGetValue(id, out var pool))
        {
            Debug.LogError($"Pool not found: {id}");
            return null;
        }

        GameObject obj = pool.Count > 0
            ? pool.Dequeue()
            : CreateInstance(_prefabs[id]);

        obj.transform.SetPositionAndRotation(position, rotation);
        obj.SetActive(true);

        if (obj.TryGetComponent<IPoolable>(out var poolable))
        {
            poolable.OnSpawn();
        }

        return obj;
    }

    public void Return(string id, GameObject obj)
    {
        if (!_pools.ContainsKey(id)) return;

        if (obj.TryGetComponent<IPoolable>(out var poolable))
        {
            poolable.OnDespawn();
        }

        obj.SetActive(false);
        _pools[id].Enqueue(obj);
    }

    private GameObject CreateInstance(GameObject prefab)
    {
        var obj = Instantiate(prefab, transform);
        obj.SetActive(false);
        return obj;
    }
}
```

---

## Collection Reuse

### Pre-allocated Arrays

```csharp
public class EnemyDetector : MonoBehaviour
{
    [SerializeField] private float _detectionRadius = 10f;
    [SerializeField] private LayerMask _enemyLayer;

    // Pre-allocate for maximum expected results
    private readonly Collider[] _overlapResults = new Collider[32];
    private readonly RaycastHit[] _raycastResults = new RaycastHit[16];

    public int GetNearbyEnemies(List<Enemy> results)
    {
        results.Clear();

        int count = Physics.OverlapSphereNonAlloc(
            transform.position,
            _detectionRadius,
            _overlapResults,
            _enemyLayer
        );

        for (int i = 0; i < count; i++)
        {
            if (_overlapResults[i].TryGetComponent<Enemy>(out var enemy))
            {
                results.Add(enemy);
            }
        }

        return results.Count;
    }

    public bool RaycastForward(out RaycastHit closestHit)
    {
        int count = Physics.RaycastNonAlloc(
            transform.position,
            transform.forward,
            _raycastResults,
            100f,
            _enemyLayer
        );

        if (count == 0)
        {
            closestHit = default;
            return false;
        }

        // Find closest hit
        closestHit = _raycastResults[0];
        for (int i = 1; i < count; i++)
        {
            if (_raycastResults[i].distance < closestHit.distance)
            {
                closestHit = _raycastResults[i];
            }
        }

        return true;
    }
}
```

### List Reuse Pattern

```csharp
public class TargetingSystem : MonoBehaviour
{
    // Reusable lists - never reallocated
    private readonly List<Enemy> _enemyCache = new(32);
    private readonly List<float> _distanceCache = new(32);

    public Enemy GetNearestEnemy(Vector3 position, float maxRange)
    {
        // Clear but don't reallocate
        _enemyCache.Clear();
        _distanceCache.Clear();

        // Populate from some source
        EnemyManager.GetAllEnemies(_enemyCache);

        if (_enemyCache.Count == 0) return null;

        Enemy nearest = null;
        float nearestDist = float.MaxValue;

        for (int i = 0; i < _enemyCache.Count; i++)
        {
            var enemy = _enemyCache[i];
            float dist = Vector3.Distance(position, enemy.transform.position);

            if (dist < maxRange && dist < nearestDist)
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

## Avoiding Garbage Collection

### String Handling

```csharp
using System.Text;
using TMPro;
using UnityEngine;

public class ScoreDisplay : MonoBehaviour
{
    [SerializeField] private TMP_Text _scoreText;

    // Reusable StringBuilder
    private readonly StringBuilder _sb = new(32);

    // Cached strings
    private const string ScorePrefix = "Score: ";

    private int _lastDisplayedScore = -1;

    public void UpdateScore(int score)
    {
        // Only update if changed
        if (score == _lastDisplayedScore) return;
        _lastDisplayedScore = score;

        _sb.Clear();
        _sb.Append(ScorePrefix);
        _sb.Append(score);
        _scoreText.SetText(_sb);
    }
}

// For TextMeshPro: Use SetText with format strings
public class TMPOptimized : MonoBehaviour
{
    [SerializeField] private TMP_Text _text;

    public void UpdateValue(int value)
    {
        // This doesn't allocate - TMP handles it internally
        _text.SetText("Value: {0}", value);
    }
}
```

### Avoiding Boxing

```csharp
// BAD: Boxing int to object
void LogValue(object value) { }
LogValue(42); // Boxes the int

// GOOD: Generic to avoid boxing
void LogValue<T>(T value) { }
LogValue(42); // No boxing

// BAD: Dictionary with value type values
Dictionary<string, object> data = new();
data["score"] = 100; // Boxes

// GOOD: Specific type
Dictionary<string, int> scores = new();
scores["player1"] = 100; // No boxing

// BAD: Enum in Dictionary key (causes boxing on lookup)
Dictionary<MyEnum, string> enumDict = new();

// GOOD: Use int-keyed dictionary or custom comparer
Dictionary<int, string> enumDict = new();
enumDict[(int)MyEnum.Value] = "data";
```

### Delegate Caching

```csharp
public class EventSystem : MonoBehaviour
{
    // BAD: Creates new delegate each frame
    private void Update()
    {
        SomeEvent += () => HandleEvent(); // Allocates!
    }

    // GOOD: Cache the delegate
    private Action _cachedHandler;

    private void Awake()
    {
        _cachedHandler = HandleEvent;
    }

    private void OnEnable()
    {
        SomeEvent += _cachedHandler;
    }

    private void OnDisable()
    {
        SomeEvent -= _cachedHandler;
    }
}
```

---

## Update Optimization

### Dirty Flag Pattern

```csharp
public class OptimizedUI : MonoBehaviour
{
    private bool _isDirty;
    private int _displayedHealth;
    private int _displayedAmmo;

    public void SetHealth(int health)
    {
        if (_displayedHealth != health)
        {
            _displayedHealth = health;
            _isDirty = true;
        }
    }

    public void SetAmmo(int ammo)
    {
        if (_displayedAmmo != ammo)
        {
            _displayedAmmo = ammo;
            _isDirty = true;
        }
    }

    private void LateUpdate()
    {
        if (_isDirty)
        {
            RefreshDisplay();
            _isDirty = false;
        }
    }

    private void RefreshDisplay()
    {
        // Expensive UI update only when needed
    }
}
```

### Throttled Updates

```csharp
public class ThrottledBehaviour : MonoBehaviour
{
    [SerializeField] private float _updateInterval = 0.1f;

    private float _timeSinceLastUpdate;

    private void Update()
    {
        _timeSinceLastUpdate += Time.deltaTime;

        if (_timeSinceLastUpdate >= _updateInterval)
        {
            _timeSinceLastUpdate = 0f;
            PerformExpensiveUpdate();
        }
    }
}

// Alternative: Staggered updates across frames
public class StaggeredUpdater : MonoBehaviour
{
    private static int _frameCounter;
    private int _updateFrame;
    private const int StaggerFrames = 5;

    private void Awake()
    {
        // Distribute updates across frames
        _updateFrame = _frameCounter++ % StaggerFrames;
    }

    private void Update()
    {
        if (Time.frameCount % StaggerFrames == _updateFrame)
        {
            PerformExpensiveUpdate();
        }
    }
}
```

### Update Manager Pattern

```csharp
// Centralized update to avoid many Update() calls
public class UpdateManager : MonoBehaviour
{
    public static UpdateManager Instance { get; private set; }

    private readonly List<IUpdateable> _updateables = new(256);
    private readonly List<IFixedUpdateable> _fixedUpdateables = new(64);

    private void Awake() => Instance = this;

    public void Register(IUpdateable updateable) => _updateables.Add(updateable);
    public void Unregister(IUpdateable updateable) => _updateables.Remove(updateable);

    public void Register(IFixedUpdateable updateable) => _fixedUpdateables.Add(updateable);
    public void Unregister(IFixedUpdateable updateable) => _fixedUpdateables.Remove(updateable);

    private void Update()
    {
        float deltaTime = Time.deltaTime;
        for (int i = _updateables.Count - 1; i >= 0; i--)
        {
            _updateables[i].OnUpdate(deltaTime);
        }
    }

    private void FixedUpdate()
    {
        float fixedDeltaTime = Time.fixedDeltaTime;
        for (int i = _fixedUpdateables.Count - 1; i >= 0; i--)
        {
            _fixedUpdateables[i].OnFixedUpdate(fixedDeltaTime);
        }
    }
}

public interface IUpdateable
{
    void OnUpdate(float deltaTime);
}

public interface IFixedUpdateable
{
    void OnFixedUpdate(float fixedDeltaTime);
}
```

---

## Coroutine Optimization

### Cached Yield Instructions

```csharp
public class CoroutineOptimization : MonoBehaviour
{
    // CACHE these - they're allocated once
    private static readonly WaitForEndOfFrame WaitEndOfFrame = new();
    private static readonly WaitForFixedUpdate WaitFixedUpdate = new();

    // Cache WaitForSeconds per duration
    private readonly Dictionary<float, WaitForSeconds> _waitCache = new();

    private WaitForSeconds GetWait(float seconds)
    {
        if (!_waitCache.TryGetValue(seconds, out var wait))
        {
            wait = new WaitForSeconds(seconds);
            _waitCache[seconds] = wait;
        }
        return wait;
    }

    private IEnumerator SpawnWaves()
    {
        var waitBetweenWaves = GetWait(2f);
        var waitBetweenEnemies = GetWait(0.5f);

        while (true)
        {
            for (int i = 0; i < 10; i++)
            {
                SpawnEnemy();
                yield return waitBetweenEnemies;
            }
            yield return waitBetweenWaves;
        }
    }
}
```

### Coroutine Lifecycle

```csharp
public class CoroutineController : MonoBehaviour
{
    private Coroutine _activeRoutine;

    public void StartAction()
    {
        // Stop existing before starting new
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
    }

    private void OnDisable()
    {
        // Always clean up on disable
        _activeRoutine = null; // Coroutine stops automatically
    }

    private IEnumerator ActionRoutine()
    {
        // Implementation
        yield return null;
        _activeRoutine = null;
    }
}
```

---

## Struct Usage

### Value Types for Data

```csharp
// GOOD: Small, immutable data as struct
public readonly struct DamageInfo
{
    public readonly int Amount;
    public readonly DamageType Type;
    public readonly Vector3 HitPoint;
    public readonly Vector3 HitNormal;

    public DamageInfo(int amount, DamageType type, Vector3 hitPoint, Vector3 hitNormal)
    {
        Amount = amount;
        Type = type;
        HitPoint = hitPoint;
        HitNormal = hitNormal;
    }
}

// Usage - no heap allocation
public void ProcessHit(Collider target, RaycastHit hit)
{
    var damage = new DamageInfo(
        _weaponDamage,
        DamageType.Physical,
        hit.point,
        hit.normal
    );

    if (target.TryGetComponent<IDamageable>(out var damageable))
    {
        damageable.ApplyDamage(damage);
    }
}
```

### NativeArray for Jobs

```csharp
using Unity.Collections;
using Unity.Jobs;
using UnityEngine;

public class JobSystemExample : MonoBehaviour
{
    private NativeArray<Vector3> _positions;
    private NativeArray<Vector3> _velocities;

    private void Awake()
    {
        // Allocate persistent native arrays
        _positions = new NativeArray<Vector3>(1000, Allocator.Persistent);
        _velocities = new NativeArray<Vector3>(1000, Allocator.Persistent);
    }

    private void OnDestroy()
    {
        // Must dispose native arrays
        if (_positions.IsCreated) _positions.Dispose();
        if (_velocities.IsCreated) _velocities.Dispose();
    }

    private void Update()
    {
        var job = new MoveJob
        {
            Positions = _positions,
            Velocities = _velocities,
            DeltaTime = Time.deltaTime
        };

        JobHandle handle = job.Schedule(_positions.Length, 64);
        handle.Complete();
    }

    private struct MoveJob : IJobParallelFor
    {
        public NativeArray<Vector3> Positions;
        [ReadOnly] public NativeArray<Vector3> Velocities;
        public float DeltaTime;

        public void Execute(int index)
        {
            Positions[index] += Velocities[index] * DeltaTime;
        }
    }
}
```

---

## Physics Optimization

### Layer-Based Filtering

```csharp
public class PhysicsOptimized : MonoBehaviour
{
    // Set in inspector - never use string-based layer names at runtime
    [SerializeField] private LayerMask _enemyLayer;
    [SerializeField] private LayerMask _obstacleLayer;
    [SerializeField] private LayerMask _interactableLayer;

    private readonly Collider[] _hitBuffer = new Collider[32];

    public bool HasLineOfSight(Vector3 target)
    {
        Vector3 direction = target - transform.position;
        float distance = direction.magnitude;

        // Single raycast with layer mask
        return !Physics.Raycast(
            transform.position,
            direction.normalized,
            distance,
            _obstacleLayer
        );
    }

    public int GetEnemiesInRadius(float radius, List<Enemy> results)
    {
        results.Clear();

        int count = Physics.OverlapSphereNonAlloc(
            transform.position,
            radius,
            _hitBuffer,
            _enemyLayer
        );

        for (int i = 0; i < count; i++)
        {
            if (_hitBuffer[i].TryGetComponent<Enemy>(out var enemy))
            {
                results.Add(enemy);
            }
        }

        return results.Count;
    }
}
```

### Spatial Partitioning

```csharp
// Simple grid-based spatial hash for fast neighbor queries
public class SpatialHash<T> where T : class
{
    private readonly Dictionary<long, List<T>> _cells = new();
    private readonly Dictionary<T, long> _itemCells = new();
    private readonly float _cellSize;

    public SpatialHash(float cellSize = 10f)
    {
        _cellSize = cellSize;
    }

    public void Insert(T item, Vector3 position)
    {
        long key = GetKey(position);

        // Remove from old cell if exists
        if (_itemCells.TryGetValue(item, out long oldKey) && oldKey != key)
        {
            _cells[oldKey].Remove(item);
        }

        // Add to new cell
        if (!_cells.TryGetValue(key, out var cell))
        {
            cell = new List<T>();
            _cells[key] = cell;
        }

        if (!cell.Contains(item))
        {
            cell.Add(item);
        }

        _itemCells[item] = key;
    }

    public void Remove(T item)
    {
        if (_itemCells.TryGetValue(item, out long key))
        {
            _cells[key].Remove(item);
            _itemCells.Remove(item);
        }
    }

    public void GetNearby(Vector3 position, float radius, List<T> results)
    {
        results.Clear();

        int cellRadius = Mathf.CeilToInt(radius / _cellSize);
        int centerX = Mathf.FloorToInt(position.x / _cellSize);
        int centerZ = Mathf.FloorToInt(position.z / _cellSize);

        for (int x = -cellRadius; x <= cellRadius; x++)
        {
            for (int z = -cellRadius; z <= cellRadius; z++)
            {
                long key = GetKey(centerX + x, centerZ + z);
                if (_cells.TryGetValue(key, out var cell))
                {
                    results.AddRange(cell);
                }
            }
        }
    }

    private long GetKey(Vector3 position)
    {
        int x = Mathf.FloorToInt(position.x / _cellSize);
        int z = Mathf.FloorToInt(position.z / _cellSize);
        return GetKey(x, z);
    }

    private long GetKey(int x, int z)
    {
        return ((long)x << 32) | (uint)z;
    }
}
```

---

## Profiling Helpers

### Scoped Profiler Samples

```csharp
using UnityEngine.Profiling;

public static class ProfilerScope
{
    public static ProfilerSample Begin(string name) => new(name);

    public readonly struct ProfilerSample : IDisposable
    {
        public ProfilerSample(string name)
        {
            Profiler.BeginSample(name);
        }

        public void Dispose()
        {
            Profiler.EndSample();
        }
    }
}

// Usage
public void ProcessEnemies()
{
    using (ProfilerScope.Begin("ProcessEnemies"))
    {
        // Code to profile
    }
}

// Or without scope
public void ManualProfiling()
{
    Profiler.BeginSample("MyOperation");
    // Expensive operation
    Profiler.EndSample();
}
```

### Memory Tracking

```csharp
#if UNITY_EDITOR
using UnityEngine;

public static class MemoryDebug
{
    private static long _lastAllocated;

    public static void MarkAllocation()
    {
        _lastAllocated = System.GC.GetTotalMemory(false);
    }

    public static void CheckAllocation(string context)
    {
        long current = System.GC.GetTotalMemory(false);
        long allocated = current - _lastAllocated;

        if (allocated > 0)
        {
            Debug.LogWarning($"[Memory] {context}: Allocated {allocated} bytes");
        }

        _lastAllocated = current;
    }
}

// Usage in development
public void SuspiciousMethod()
{
    MemoryDebug.MarkAllocation();
    // Suspected allocation
    MemoryDebug.CheckAllocation("SuspiciousMethod");
}
#endif
```
