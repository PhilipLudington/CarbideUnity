# Carbide Unity

A framework for hardened Unity/C# development optimized for AI assistance.

## What is Carbide Unity?

Carbide Unity provides coding standards, patterns, and Claude Code slash commands designed to help developers write safe, maintainable, and performant Unity code with AI assistance.

### Core Principles

1. **Prescriptive over Permissive** - Clear rules are easier to follow than vague guidelines
2. **Explicit over Implicit** - Null handling, event lifecycle, and ownership should be obvious
3. **Validated over Assumed** - Patterns that catch issues early
4. **Simple over Clever** - Readable code beats clever optimizations
5. **Performant by Default** - Patterns avoid common Unity performance pitfalls

## Quick Start

### Without Setup

Use the review commands immediately on any Unity project:

```
/carbide-unity-review Assets/Scripts/Player.cs
/carbide-unity-safety Assets/Scripts/
```

### Full Project Setup

```
/carbide-unity-init MyGame
```

This creates a complete project structure with:
- Assembly definitions for fast compilation
- Folder structure following best practices
- Starter scripts (GameManager, Events)
- Script templates

## Documentation

| Document | Purpose |
|----------|---------|
| [CARBIDE_UNITY.md](CARBIDE_UNITY.md) | Quick reference for AI development |
| [STANDARDS.md](STANDARDS.md) | Complete coding standards |
| [docs/patterns/performance.md](docs/patterns/performance.md) | Object pooling, GC avoidance, optimization |
| [docs/patterns/architecture.md](docs/patterns/architecture.md) | ScriptableObject events, state machines, DI |
| [docs/safety/common-pitfalls.md](docs/safety/common-pitfalls.md) | Null refs, event leaks, threading issues |

## Slash Commands

| Command | Mode | Purpose |
|---------|------|---------|
| `/carbide-unity-init` | Setup | Create new Carbide-compliant Unity projects |
| `/carbide-unity-review` | Standalone | Review code against naming, memory, lifecycle standards |
| `/carbide-unity-safety` | Standalone | Security-focused review for crashes, leaks, race conditions |

## Key Standards

### Naming

```csharp
public class PlayerController : MonoBehaviour    // PascalCase class
{
    [SerializeField] private float _moveSpeed;   // _camelCase private field

    public event Action OnDeath;                 // On + PascalCase event

    private bool _isGrounded;                    // is/has/can prefix for bools

    public void TakeDamage(int amount) { }       // PascalCase method
}
```

### MonoBehaviour Lifecycle

```csharp
private void Awake()      // Cache own components
private void OnEnable()   // Subscribe to events
private void Start()      // Setup requiring other objects
private void OnDisable()  // Unsubscribe from events
private void OnDestroy()  // Final cleanup
```

### Memory Rules

- **No allocations in Update** - No `new`, LINQ, or string concatenation
- **Cache everything** - GetComponent in Awake, not Update
- **Pool frequently instantiated objects** - Never Instantiate/Destroy in loops
- **Use NonAlloc methods** - `Physics.RaycastNonAlloc`, not `Physics.RaycastAll`

### Event Safety

```csharp
private void OnEnable()
{
    GameEvents.OnGamePaused += HandlePause;  // Subscribe
}

private void OnDisable()
{
    GameEvents.OnGamePaused -= HandlePause;  // Always unsubscribe!
}
```

## Integration

### With Existing Projects

1. Copy slash commands to your project's `.claude/commands/` folder
2. Reference Carbide Unity in your project's `CLAUDE.md`:

```markdown
# MyGame

This project follows [Carbide Unity](/path/to/CarbideUnity) standards.
```

### With New Projects

Run `/carbide-unity-init` to scaffold a complete project structure.

## License

MIT License - See [LICENSE](LICENSE)
