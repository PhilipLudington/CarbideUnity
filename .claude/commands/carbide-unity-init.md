# Carbide Unity Project Initialization

Initialize a new Unity project or directory with Carbide Unity standards and structure.

## Instructions

Initialize a Carbide Unity project at `$ARGUMENTS`. If no path is provided, initialize in the current directory.

## Setup Steps

### 1. Gather Project Information

Ask the user:
1. **Project Name**: What should the project/namespace be called?
2. **Project Type**: Game, Tool, Package?
3. **Architecture Preference**: ScriptableObject-based, traditional singletons, or dependency injection?

### 2. Create Folder Structure

Create the standard Unity folder structure:

```
Assets/
├── _Project/                    # Project-specific assets (underscore keeps it at top)
│   ├── Scripts/
│   │   ├── Core/               # Managers, systems, utilities
│   │   │   ├── {ProjectName}.Core.asmdef
│   │   │   └── ...
│   │   ├── Gameplay/           # Player, enemies, items, mechanics
│   │   │   ├── {ProjectName}.Gameplay.asmdef
│   │   │   └── ...
│   │   ├── UI/                 # UI controllers, views
│   │   │   ├── {ProjectName}.UI.asmdef
│   │   │   └── ...
│   │   ├── Data/               # ScriptableObject definitions
│   │   │   └── ...
│   │   └── Editor/             # Editor scripts
│   │       ├── {ProjectName}.Editor.asmdef
│   │       └── ...
│   ├── Prefabs/
│   │   ├── Characters/
│   │   ├── Environment/
│   │   ├── UI/
│   │   └── VFX/
│   ├── Scenes/
│   │   ├── _Init/              # Bootstrap scene
│   │   ├── Levels/
│   │   └── Test/
│   ├── Art/
│   │   ├── Sprites/
│   │   ├── Textures/
│   │   ├── Materials/
│   │   └── Animations/
│   ├── Audio/
│   │   ├── Music/
│   │   ├── SFX/
│   │   └── Mixers/
│   └── ScriptableObjects/      # Asset instances
│       ├── Config/
│       ├── Events/
│       └── Data/
├── Plugins/                     # Third-party packages
└── Resources/                   # Only for runtime-loaded assets (use sparingly)
```

### 3. Create Assembly Definitions

Create assembly definition files for proper code organization:

**{ProjectName}.Core.asmdef**:
```json
{
    "name": "{ProjectName}.Core",
    "rootNamespace": "{ProjectName}.Core",
    "references": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

**{ProjectName}.Gameplay.asmdef**:
```json
{
    "name": "{ProjectName}.Gameplay",
    "rootNamespace": "{ProjectName}.Gameplay",
    "references": [
        "{ProjectName}.Core"
    ],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

**{ProjectName}.UI.asmdef**:
```json
{
    "name": "{ProjectName}.UI",
    "rootNamespace": "{ProjectName}.UI",
    "references": [
        "{ProjectName}.Core"
    ],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

**{ProjectName}.Editor.asmdef**:
```json
{
    "name": "{ProjectName}.Editor",
    "rootNamespace": "{ProjectName}.Editor",
    "references": [
        "{ProjectName}.Core",
        "{ProjectName}.Gameplay",
        "{ProjectName}.UI"
    ],
    "includePlatforms": ["Editor"],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

### 4. Create Starter Scripts

#### GameManager.cs (Core)
```csharp
using UnityEngine;

namespace {ProjectName}.Core
{
    /// <summary>
    /// Central game manager handling game state and core systems.
    /// </summary>
    [DefaultExecutionOrder(-100)]
    public class GameManager : MonoBehaviour
    {
        public static GameManager Instance { get; private set; }

        [Header("Configuration")]
        [SerializeField] private bool _dontDestroyOnLoad = true;

        public bool IsPaused { get; private set; }

        public event System.Action<bool> OnPauseChanged;

        private void Awake()
        {
            if (Instance != null && Instance != this)
            {
                Destroy(gameObject);
                return;
            }

            Instance = this;

            if (_dontDestroyOnLoad)
            {
                DontDestroyOnLoad(gameObject);
            }
        }

        public void SetPaused(bool paused)
        {
            if (IsPaused == paused) return;

            IsPaused = paused;
            Time.timeScale = paused ? 0f : 1f;
            OnPauseChanged?.Invoke(paused);
        }
    }
}
```

#### ScriptableObject Event (if SO architecture selected)
```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

namespace {ProjectName}.Core
{
    /// <summary>
    /// ScriptableObject-based event for decoupled communication.
    /// </summary>
    [CreateAssetMenu(fileName = "NewGameEvent", menuName = "{ProjectName}/Events/Game Event")]
    public class GameEventSO : ScriptableObject
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

        private void OnEnable()
        {
            _listeners.Clear();
        }
    }
}
```

#### Script Template
Create a template that users can reference:

```csharp
using UnityEngine;

namespace {ProjectName}.Gameplay
{
    /// <summary>
    /// Brief description of this component's responsibility.
    /// </summary>
    public class ScriptTemplate : MonoBehaviour
    {
        #region Serialized Fields

        [Header("Configuration")]
        [SerializeField, Tooltip("Description for designers")]
        private float _exampleValue = 1f;

        [Header("References")]
        [SerializeField]
        private Transform _targetTransform;

        #endregion

        #region Events

        public event System.Action OnExampleEvent;

        #endregion

        #region Properties

        public float ExampleValue => _exampleValue;

        #endregion

        #region Private Fields

        private bool _isInitialized;

        #endregion

        #region Unity Lifecycle

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

        #endregion

        #region Public Methods

        public void Initialize()
        {
            if (_isInitialized) return;
            _isInitialized = true;
        }

        #endregion

        #region Private Methods

        private void CacheReferences()
        {
            // Cache GetComponent calls here
        }

        private void SubscribeToEvents()
        {
            // Subscribe to events here
        }

        private void UnsubscribeFromEvents()
        {
            // Unsubscribe from events here
        }

        #endregion

        #region Editor

#if UNITY_EDITOR
        private void OnValidate()
        {
            // Validate serialized fields in editor
        }
#endif

        #endregion
    }
}
```

### 5. Copy Documentation

Copy these files to the project root:
- CARBIDE_UNITY.md
- STANDARDS.md
- docs/ folder

Or create a CLAUDE.md that references the Carbide Unity location:

```markdown
# {ProjectName}

This project follows [Carbide Unity](path/to/CarbideUnity) standards.

## Quick Links
- [AI Development Guide](path/to/CarbideUnity/CARBIDE_UNITY.md)
- [Coding Standards](path/to/CarbideUnity/STANDARDS.md)
- [Performance Patterns](path/to/CarbideUnity/docs/patterns/performance.md)
- [Architecture Patterns](path/to/CarbideUnity/docs/patterns/architecture.md)
- [Safety Guide](path/to/CarbideUnity/docs/safety/common-pitfalls.md)

## Project-Specific Notes
- Add project-specific conventions here
```

### 6. Create .gitignore (if needed)

Standard Unity .gitignore:
```
# Unity generated
[Ll]ibrary/
[Tt]emp/
[Oo]bj/
[Bb]uild/
[Bb]uilds/
[Ll]ogs/
[Uu]ser[Ss]ettings/
[Mm]emoryCaptures/
[Rr]ecordings/

# Asset meta data
*.pidb.meta
*.pdb.meta
*.mdb.meta

# Visual Studio
.vs/
*.csproj
*.unityproj
*.sln
*.suo
*.tmp
*.user
*.userprefs
*.pidb
*.booproj
*.svd
*.pdb
*.mdb
*.opendb
*.VC.db

# Rider
.idea/

# OS generated
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Builds
*.apk
*.aab
*.unitypackage
*.app

# Crashlytics
crashlytics-build.properties
```

## Output

After initialization, provide:
1. Summary of created structure
2. Next steps for the user
3. Reminder about assembly definitions needing Unity Editor reimport
