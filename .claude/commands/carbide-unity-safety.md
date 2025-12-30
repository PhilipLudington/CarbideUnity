# Carbide Unity Safety Review

Perform a security and safety-focused review of Unity/C# code, targeting common pitfalls that cause crashes, memory leaks, and runtime errors.

## Instructions

Review the code at `$ARGUMENTS` for safety issues. If no path is provided, ask the user which files to review.

## Safety Checklist

### 1. Null Reference Prevention

#### Unity Object Null
- [ ] Using `== null` not `is null` for Unity objects
- [ ] Checking destroyed objects properly
- [ ] Validating serialized references in Awake/Start

#### Defensive Coding
- [ ] Guard clauses at method entry
- [ ] TryGetComponent over GetComponent + null check
- [ ] Null checks before accessing component properties

```csharp
// UNSAFE
target.GetComponent<Health>().TakeDamage(10);

// SAFE
if (target.TryGetComponent<Health>(out var health))
{
    health.TakeDamage(10);
}
```

### 2. Event/Delegate Leaks

#### Subscription Safety
- [ ] Every += has a corresponding -=
- [ ] Unsubscribe in OnDisable (not OnDestroy)
- [ ] Static events cleared on domain reload
- [ ] No anonymous lambdas for events that need unsubscription

```csharp
// LEAK: Can't unsubscribe from lambda
GameEvents.OnPause += () => HandlePause();

// SAFE: Named method can be unsubscribed
GameEvents.OnPause += HandlePause;
```

### 3. Coroutine Safety

- [ ] Stored coroutine references for stopping
- [ ] Cleanup on OnDisable
- [ ] No accessing destroyed objects after yield
- [ ] Cached WaitForSeconds instances

```csharp
// UNSAFE: Lost reference, can't stop
StartCoroutine(DoSomething());

// SAFE: Tracked reference
_routine = StartCoroutine(DoSomething());
```

### 4. Physics Callback Safety

- [ ] No Destroy() in OnCollision/OnTrigger (delay instead)
- [ ] No heavy computation in physics callbacks
- [ ] Physics modifications in FixedUpdate only
- [ ] Proper kinematic toggling for teleportation

### 5. Threading Issues

- [ ] Unity API only on main thread
- [ ] Proper synchronization for shared state
- [ ] Main thread dispatch for async callbacks

### 6. Resource Leaks

- [ ] RenderTexture.Release() called
- [ ] ComputeBuffer.Dispose() called
- [ ] NativeArray.Dispose() called
- [ ] AsyncOperation callbacks cleaned up
- [ ] Texture2D cleanup for runtime-created textures

### 7. Serialization Safety

- [ ] No modifying prefab assets at runtime
- [ ] ISerializationCallbackReceiver implemented correctly
- [ ] No circular references in serialized data
- [ ] Handling of missing/null serialized references

### 8. Scene/Object Lifecycle

- [ ] Proper DontDestroyOnLoad handling
- [ ] Scene load/unload event cleanup
- [ ] Checking object validity after async operations
- [ ] Proper handling of disabled objects

### 9. Input Handling

- [ ] Null checks on input system references
- [ ] Handling of disconnected controllers
- [ ] Touch input bounds checking

### 10. Collection Safety

- [ ] No modification during iteration
- [ ] Bounds checking on array access
- [ ] Null checks in collection operations

```csharp
// UNSAFE: Modifying during iteration
foreach (var enemy in _enemies)
{
    if (enemy.IsDead)
        _enemies.Remove(enemy); // Throws!
}

// SAFE: Reverse iteration or copy
for (int i = _enemies.Count - 1; i >= 0; i--)
{
    if (_enemies[i].IsDead)
        _enemies.RemoveAt(i);
}
```

## Output Format

```
## Safety Review: [Filename]

### Critical Issues (Must Fix)
Issues that will cause crashes, leaks, or data corruption.

### High Risk (Should Fix)
Issues likely to cause problems under certain conditions.

### Medium Risk (Consider Fixing)
Potential issues that could manifest under edge cases.

### Low Risk (Best Practice)
Minor improvements for robustness.

---

### Issue: [Title]
**Severity**: Critical/High/Medium/Low
**Line(s)**: X-Y
**Category**: [Null Reference/Event Leak/etc.]

**Problem**:
Description of what could go wrong.

**Current Code**:
```csharp
// problematic code
```

**Suggested Fix**:
```csharp
// fixed code
```

**Impact**:
What happens if not fixed (crash, leak, corruption, etc.)
```

## Example Output

```
## Safety Review: EnemySpawner.cs

### Critical Issues (Must Fix)

### Issue: Event subscription without cleanup
**Severity**: Critical
**Line(s)**: 15, 45
**Category**: Event Leak

**Problem**:
Subscribing to static event in Start() but never unsubscribing. If this object is destroyed and recreated (e.g., scene reload), handlers accumulate.

**Current Code**:
```csharp
private void Start()
{
    GameManager.OnWaveComplete += SpawnNextWave;
}
```

**Suggested Fix**:
```csharp
private void OnEnable()
{
    GameManager.OnWaveComplete += SpawnNextWave;
}

private void OnDisable()
{
    GameManager.OnWaveComplete -= SpawnNextWave;
}
```

**Impact**:
Memory leak, duplicate event handling, null reference exceptions when destroyed object's method is called.

---

### High Risk (Should Fix)

### Issue: Destroy in OnTriggerEnter
**Severity**: High
**Line(s)**: 78
**Category**: Physics Safety

**Problem**:
Calling Destroy() directly in physics callback can cause physics engine issues.

**Current Code**:
```csharp
private void OnTriggerEnter(Collider other)
{
    if (other.CompareTag("Player"))
    {
        Destroy(gameObject);
    }
}
```

**Suggested Fix**:
```csharp
private void OnTriggerEnter(Collider other)
{
    if (other.CompareTag("Player"))
    {
        gameObject.SetActive(false);
        Destroy(gameObject, 0.1f);
    }
}
```

**Impact**:
Potential physics engine errors, especially with multiple simultaneous collisions.
```
