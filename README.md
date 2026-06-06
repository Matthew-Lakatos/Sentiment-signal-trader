# COBB CAN MOVE — Unity 6 Replica
### Full C# Unity implementation of the indie horror game

---

## QUICK START

1. **Create project:** Unity 6 → New → **2D (Built-in Render Pipeline)**
2. **Copy scripts:** Place all `.cs` files into `Assets/Scripts/` preserving the subfolder structure shown below
3. **Open scene:** Use the default `SampleScene` (or create a new empty scene)
4. **Add bootstrap:** Create an empty GameObject → name it `Bootstrap` → attach `SceneBootstrap.cs`
5. **Press Play** — the entire game builds itself in code. No prefabs or manual wiring needed.

---

## FILE STRUCTURE

```
Assets/Scripts/
├── Core/
│   ├── GameManager.cs          State machine, level flow, entity wiring
│   ├── GameManagerExtras.cs    Extension methods (GetRipples)
│   ├── GameManagerWiring.cs    Post-level HUD + minimap init
│   ├── ObjectiveManager.cs     Furnace, breakers, heat slots, exit logic
│   ├── ObjectiveManagerAdditions.cs  Notes on required additions
│   └── SceneBootstrap.cs       ★ ENTRY POINT — creates everything in code
├── Data/
│   ├── GameConstants.cs        All tuning values, enums, K.* constants
│   └── LevelConfig.cs          Exact per-level objectives + rule database
├── Generation/
│   ├── AStarPathfinder.cs      Grid A* used by Cobb AI
│   ├── ChunkTemplates.cs       All 7×6 chunk tile layouts (9 variants)
│   └── MapGenerator.cs         Seeded chunk-based dungeon generation
├── Player/
│   └── PlayerController.cs     Movement, carry, interact, survival stats
├── Enemy/
│   ├── CobbController.cs       Full AI: wander/chase, A*, grab animation
│   └── IKArm.cs               Triangle IK arm solver + LineRenderer
├── Items/
│   └── (item types via WorldItem, WorldFurnace, etc. in ObjectiveManager)
├── Lighting/
│   └── LightingSystem.cs       Shadow-cast 2D darkness (RenderTexture overlay)
├── Audio/
│   ├── AudioManager.cs         Singleton, spatial audio, SFX pool
│   └── SoundSynthesizer.cs     100% procedural audio — no audio files needed
├── UI/
│   ├── UIManager.cs            Title, options, rule intro, death, win screens
│   └── HUDController.cs        Rule bar, coal counter, bars, sense icons, compass
└── Utils/
    ├── CameraController.cs     Smooth follow + world-bounds clamping
    ├── ConveyorAnimator.cs     Scrolling UV for conveyor belt material
    ├── MinimapController.cs    M-key minimap overlay (Cobb, items, objectives)
    ├── ProceduralSprites.cs    ★ All pixel art generated via Texture2D
    └── TileMapRenderer.cs      Combined-mesh tile rendering (3 draw calls total)
```

---

## REQUIRED CODE FIXES (apply before pressing Play)

These are small additions to files that couldn't be auto-patched:

### 1. GameManager.cs — make `_ripples` public
Find this line:
```csharp
private List<SoundRipple> _ripples = new();
```
Change to:
```csharp
public List<SoundRipple> _ripples = new();
```

### 2. ObjectiveManager.cs — expose HeatSlots
In the `// Public accessors` region at the bottom, add:
```csharp
public List<WorldHeat> HeatSlots => _heatSlots;
```

### 3. SceneBootstrap.cs — attach GameManagerWiring
At the end of `BuildGameManager()`, before the final `return gm;`, add:
```csharp
go.AddComponent<GameManagerWiring>();
```

### 4. TileMapRenderer.cs — attach ConveyorAnimator
In `Build()`, after calling `AssignMat(_convMR, "tile_conv");`, add:
```csharp
var convAnim = _convMF.gameObject.GetComponent<ConveyorAnimator>()
               ?? _convMF.gameObject.AddComponent<ConveyorAnimator>();
```

### 5. HUDController.cs — add Canvas parameter to Initialise signature
The `Initialise` method signature in `HUDController.cs` should match what
`GameManagerWiring.cs` calls:
```csharp
public void Initialise(Canvas canvas,
    PlayerController player, ObjectiveManager obj,
    LevelConfig cfg, bool[] rules,
    List<SoundRipple> ripples, CameraController cam)
```
(The method body stores canvas and calls Build() — already written correctly.)

---

## LEVEL OBJECTIVES (exact, matching original game)

| Lv | Rules Active                               | Coal | Breakers | Special            |
|----|--------------------------------------------|------|----------|--------------------|
|  1 | MOVE                                       |   3  |    1     | Tutorial           |
|  2 | MOVE + HEAR                                |   3  |    1     | Rocks matter       |
|  3 | MOVE + HEAR + SEE                          |   4  |    2     | Batteries spawn    |
|  4 | MOVE + HEAR + SEE + SMELL                  |   4  |    2     | Deodorant spawns   |
|  5 | MOVE + HEAR + SEE + SMELL + REACH          |   5  |    3     | Longer arms        |
|  6 | MOVE + HEAR + SEE + SMELL + REACH + DUP    |   5  |    3     | 2nd Cobb appears   |
|  7 | ALL RULES + FREEZE + STARVE               |   6  |    0     | ★ BOSS: 6 batteries|
|    | *(boss: batteries → heat slots → Cobb burns 15s → exit opens)*                |

---

## CONTROLS

| Key          | Action                                      |
|--------------|---------------------------------------------|
| WASD / ←↑↓→ | Move                                        |
| E / SPACE    | Interact (pick up, deposit, flip, open)     |
| F            | Throw rock (distracts HEAR)                 |
| Q            | Place carried candle                         |
| M            | Toggle minimap overlay                       |
| ESC          | Pause / options                             |

---

## GAME RULES SUMMARY

| Rule           | Effect                                                            | Counter        |
|----------------|-------------------------------------------------------------------|----------------|
| MOVE           | Cobb wanders and hunts you                                        | None           |
| HEAR           | Footsteps/breakers attract Cobb. Rocks distract.                  | Throw rocks    |
| SEE            | Cobb spots you when you're lit. Darkness hides you.               | Stay dark      |
| SMELL          | Cobb sniffs periodically; rampages up close.                      | Deodorant      |
| REACH          | Catch radius doubles. Arms visually extend.                       | Keep distance  |
| DUPLICATE      | A clone Cobb appears (speed ×0.78 each).                          | None           |
| FREEZE         | Low-light drains warmth. Near furnace/candles → restore.          | Light sources  |
| STARVE         | Hunger drains over time. Carrots restore +38.                     | Find carrots   |

---

## ARCHITECTURE

```
SceneBootstrap (Awake)
  │
  ├─ AudioManager    (DontDestroyOnLoad singleton)
  ├─ Camera          + LightingSystem (OnPostRender GL overlay)
  │                  + CameraController
  ├─ Player          + PlayerController
  ├─ Cobb            + CobbController
  │    └─ ArmL/ArmR  + IKArm (LineRenderer)
  ├─ CobbClone×2     + CobbController (inactive until needed)
  ├─ TileMapRenderer (3 combined meshes: floor / wall / conveyor)
  ├─ ObjectiveManager
  ├─ UIManager       (Canvas, sortOrder 100)
  ├─ HUDController   (Canvas, sortOrder 90)
  ├─ MinimapController (Canvas, sortOrder 85)
  └─ GameManager     + GameManagerWiring
       │
       └─ OnStateChanged → Playing
            ├─ MapGenerator.Generate()     (seeded chunk drunk-walker)
            ├─ TileMapRenderer.Build()     (stamps meshes)
            ├─ SpawnWorldItems()           (furnace, coal, candles, etc.)
            ├─ PlacePlayer() / PlaceCobb()
            ├─ LightingSystem.Sources populated
            └─ GameManagerWiring.DoWirePlay()
                 ├─ HUD.Initialise()
                 ├─ Minimap.Map = …
                 └─ ConveyorAnimator attached
```

---

## PROCEDURAL AUDIO

No `.wav` or `.mp3` files are needed. All sounds are synthesised at runtime:

- **White noise** filtered through 1-pole LP/HP chains → footsteps, coal, rocks
- **Sawtooth oscillators** → Cobb growl, furnace drone, death sting  
- **Square oscillators** → pickup chimes, level start fanfare
- **Spatial panning** via `AudioSource.panStereo` calculated from Cobb→Player delta
- **Ambient loop** → 8-second looping noise buffer through 90 Hz LP filter

---

## LIGHTING SYSTEM

A custom shadow-casting darkness system runs per-frame via `OnPostRender`:

1. Fill a 320×240 `RenderTexture` buffer with `alpha=0.97` (near-black)
2. For each light source, cast `SHADOW_RAYS=96` rays outward, stopping at walls
3. Exponentially fade alpha along each ray from 1→0 at the radius edge
4. `GL.Begin(QUADS)` blits the darkness texture over the rendered scene

Light radii (in tiles):
- Player base: **1.9** | Carrying coal: **2.4** | Carrying battery: **3.2**
- Placed candle: **1.6** | Decor candle: **1.2** | Furnace: **4.0**

---

## ADDING MORE LEVELS / ENDLESS MODE

**Story:** Add entries to `LevelDatabase.Levels[]` in `LevelConfig.cs`

**Endless:** `GameManager.StartEndlessMode()` starts at level 3 config.
To make it truly endless, modify `LevelComplete()` in GameManager:
```csharp
private void LevelComplete()
{
    if (Mode == GameMode.Endless)
    {
        Level++;   // increment indefinitely
        // optionally add a random extra rule each loop
        StartCoroutine(TransitionToNextLevel(Level));
    }
    // ... rest unchanged
}
```

---

## UNITY VERSION

Tested architecture targets: **Unity 6000.x** (Built-in Render Pipeline).  
Compatible with Unity 2022 LTS+ with no changes.  
Does **not** require URP, HDRP, or any Package Manager packages beyond defaults.

---

*Replica built from gameplay analysis, developer devlogs, and player reports.*  
*Original game "Cobb Can Move" by abho — play it at https://abho.itch.io/cobb-can-move*
