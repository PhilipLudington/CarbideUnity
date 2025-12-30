# Carbide Unity Code Review

Review Unity/C# code against Carbide Unity standards. This is a standalone review that doesn't require tooling.

## Instructions

Review the code at `$ARGUMENTS` against the Carbide Unity standards. If no path is provided, ask the user which files to review.

## Review Categories

### 1. Naming Conventions
Check against STANDARDS.md naming rules:
- Classes/Structs: PascalCase
- Interfaces: I + PascalCase
- Methods/Properties: PascalCase
- Private fields: _camelCase
- Serialized fields: [SerializeField] private _camelCase
- Constants: PascalCase
- Events: On + PascalCase
- Booleans: is/has/can/should prefix

### 2. MonoBehaviour Lifecycle
- Awake() for caching own components
- OnEnable() for event subscription
- Start() for setup requiring other objects
- OnDisable() for event unsubscription
- Proper cleanup in OnDestroy()
- Physics logic in FixedUpdate()
- Camera/post-processing in LateUpdate()

### 3. Memory Management
- No allocations in Update/FixedUpdate/LateUpdate
- No LINQ in hot paths
- No string concatenation in loops
- Cached component references
- Object pooling for frequently instantiated objects
- Pre-allocated arrays for NonAlloc physics methods
- StringBuilder for string building

### 4. Null Safety
- Explicit null checks with meaningful responses
- TryGetComponent pattern usage
- Guard clauses at method start
- Unity null check (== null) not C# null check (is null)
- Validated serialized references

### 5. Event Handling
- Events subscribed in OnEnable
- Events unsubscribed in OnDisable
- Cached delegates when needed
- Null-conditional for raising events

### 6. Serialization
- [SerializeField] on private fields (not public)
- [Tooltip] for designer-facing fields
- [Header] for grouping
- [Range] where applicable
- No serializing interfaces or complex types

### 7. Code Organization
- One class per file
- Proper using statement order
- Logical member ordering
- Namespaces used
- XML documentation on public API

### 8. Performance
- No Find/GetComponent in Update
- Cached expensive references
- Throttled or dirty-flag updates where appropriate
- Layer masks for physics filtering

## Output Format

For each file reviewed, provide:

```
## [Filename]

### Summary
Brief overall assessment

### Issues Found

#### [Category]: [Severity: Critical/Warning/Suggestion]
- Line X: Description of issue
  - Current: `code as written`
  - Suggested: `improved code`

### Positive Patterns
- List of things done well
```

## Severity Levels

- **Critical**: Will cause bugs, crashes, or significant performance issues
- **Warning**: Violates standards, potential issues
- **Suggestion**: Could be improved but functional

## Example Review

```
## PlayerController.cs

### Summary
Generally well-structured but has memory allocation issues in Update and missing event cleanup.

### Issues Found

#### Memory Management: Critical
- Line 45: String concatenation in Update creates garbage
  - Current: `_debugText.text = "Speed: " + _speed;`
  - Suggested: `_debugText.SetText("Speed: {0}", _speed);`

#### Event Handling: Critical
- Line 23: Event subscribed in Start but never unsubscribed
  - Add OnDisable to unsubscribe from GameManager.OnGamePaused

#### Naming: Warning
- Line 12: Public field should be private with SerializeField
  - Current: `public float moveSpeed = 5f;`
  - Suggested: `[SerializeField] private float _moveSpeed = 5f;`

### Positive Patterns
- Good use of RequireComponent
- Proper component caching in Awake
- Clear method organization
```
