# Blueprint Specification: Sled Surfers Recreation

## Target Application Overview
* **Application / Package Name**: `com.sled.surfers.game`
* **Application Title**: *Sled Surfers*
* **Version**: `1.13.2` (Version Code: `1132`)
* **Target OS / Platform**: Android ARM64-v8a (Min SDK: 25, Target SDK: 36)
* **Archive Format**: XAPK (Container holding `com.sled.surfers.game.apk`, `config.arm64_v8a.apk`, and `manifest.json`)

---

## Phase 1: Engine & Tech Stack Verification (The "Proofs")

### 1.1 Game Engine Identification
* **Game Engine**: **Unity Engine**
* **Exact Engine Version**: **`Unity 6000.3.13f1`** (Unity 6.0 LTS generation)
* **Scripting Backend / Runtime**: **Unity IL2CPP**
* **IL2CPP Metadata Version**: **Version 39** (Magic: `0xFAB11BAF` / `4205910959`)
* **Native Architecture Target**: `arm64-v8a` (64-bit ELF)

### 1.2 Binary Evidence, File Paths & Hashes
All findings are corroborated by direct static extraction and structural analysis of the shipped container:

| Artifact / Binary Marker | Relative Path in Container | Size (Bytes) | Binary Proof / Identification Details |
| :--- | :--- | :--- | :--- |
| **Unity Runtime Engine** | `config.arm64_v8a.apk!lib/arm64-v8a/libunity.so` | 23,683,904 B | Exports Unity native engine symbols. Matches Unity 6 runtime header signatures. |
| **IL2CPP AOT Executable** | `config.arm64_v8a.apk!lib/arm64-v8a/libil2cpp.so` | 82,533,392 B | 64-bit ELF ARM64 binary containing 84,061 executable method pointers, `code_gen_modules` at `0x4933e40`, 119 compiled module definitions. |
| **Burst Compiler Shared Lib** | `config.arm64_v8a.apk!lib/arm64-v8a/lib_burst_generated.so` | 419,128 B | Unity Burst Compiler AOT shared library used for job execution. |
| **IL2CPP Global Metadata** | `com.sled.surfers.game.apk!assets/bin/Data/Managed/Metadata/global-metadata.dat` | 10,618,532 B | Header magic `0xFAB11BAF`, Version 39. Decodes 119 assemblies, 14,124 types, 94,070 methods, 61,236 fields, 17,678 string literals. |
| **Unity Asset Archive** | `com.sled.surfers.game.apk!assets/bin/Data/data.unity3d` | 115,356,203 B | Header signature `UnityFS 5.x.x 6000.3.13f1`, confirming exact engine release `6000.3.13f1`. |
| **Engine Boot Config** | `com.sled.surfers.game.apk!assets/bin/Data/boot.config` | 222 B | Declares `gfx-threading-mode=4`, `gc-max-time-slice=3`, `build-guid=522fc2ffff2e494495dc4fa9809ca695`. |
| **Scripting Assemblies Registry** | `com.sled.surfers.game.apk!assets/bin/Data/ScriptingAssemblies.json` | 6,490 B | Full manifest of compiled managed assemblies. |

### 1.3 Core Frameworks & Middleware Detected
Verified via `ScriptingAssemblies.json` and IL2CPP image table:
* **Dependency Injection**: `VContainer.dll` (`VContainer` framework with `SceneLifetimeScope`, `LevelLifetimeScope`, `PlayerLifetimeScope`).
* **Asynchronous Flow**: `UniTask.dll` (Cysharp UniTask runtime integration).
* **Tweening Engine**: `PrimeTween.Runtime.dll` (PrimeTween high-performance animation).
* **Path & Spline System**: `Dreamteck.Splines.dll` & `Dreamteck.Utilities.dll` (Curved track paths, rails, and gondola/train routes).
* **Physics & Cables**: `OptimizedRopesAndCables.Scripts.dll`.
* **Rendering Pipeline**: Universal Render Pipeline (`Unity.RenderPipelines.Universal.Runtime.dll`, URP).
* **Skeletal Animation**: Spine 2D (`spine-csharp.dll`, `spine-unity.dll`).
* **Water Simulation**: Stylized Water 2 (`sc.stylizedwater2.runtime.dll`).
* **Haptics**: Lofelt Nice Vibrations (`Lofelt.NiceVibrations.dll`) & CandyCoded (`CandyCoded.HapticFeedback.dll`).
* **Camera Management**: Unity Cinemachine (`Cinemachine.dll`).
* **Input Handling**: Unity New Input System (`Unity.InputSystem.dll`).
* **Event Architecture**: EventBus (`EventBus.dll`).

---

## Phase 2: Core Gameplay Systems & Architecture

### 2.1 Primary Game Systems & Module Hierarchy
The application code resides primarily under the namespace `Snowgame` and its specialized sub-namespaces:

```
Snowgame
 ├── GameManager                          [Master coordinator & flow manager]
 ├── GameState                            [Central state machine enum]
 ├── InputManager                         [Pointer / touch / stick / button input capture]
 ├── LevelLoadingScreen / LoadingScreenManager [Scene/level transition visuals]
 ├── ProgressTracker / RunInfo            [Distance tracking, scoring, run metrics]
 ├── UpgradesManager                      [Persistent upgrade progression]
 │
 ├── Player (Snowgame.Player / Snowgame.Player.Modules)
 │    ├── PlayerController                [Orchestrator of sled & ragdoll character]
 │    ├── PlayerPhysicsBody               [RigidBody velocity, speed, grounded state]
 │    ├── Character                       [3D Model, Animator, IKController, Ragdoll]
 │    ├── Sled                            [Active vehicle, IK anchor points, speed multiplier]
 │    ├── PlayerControlsModule            [Steering, airborne pitch/roll torque]
 │    ├── PlayerAirTimeModule             [Airtime calculation & reward multiplier]
 │    ├── PlayerAirborneTricksModule      [Air trick detection, flips/spins sequence]
 │    ├── PlayerBoosterModule             [Rocket booster activation, fuel, cooldown]
 │    ├── PlayerCrashDetectionModule      [Obstacle collision, knockback, crash triggering]
 │    ├── PlayerFrictionModule            [Surface physics, snow vs. ice factors]
 │    └── PlayerStabilizatorModule        [Airborne and grounded upright stabilization]
 │
 ├── Slingshot (Snowgame.Runtime.Core.Slingshot / Snowgame.SlingshotController)
 │    ├── SlingshotController             [Touch pull-back, tension physics, launch impulse]
 │    └── SlingshotUpgradeInfo            [Launch velocity and elastic force scaling]
 │
 ├── Levels (Snowgame.Runtime.Features.Levels)
 │    ├── LevelsManager                   [Level loading, cycling, serialization, progress]
 │    ├── LevelController                 [Single level instance, track length, start/finish]
 │    └── LevelLifetimeScope              [VContainer DI sub-scope for active track]
 │
 ├── Obstacles & Environment (Snowgame.Runtime.Features.LevelsDesign.*)
 │    ├── HighwayController / CarController [Road traffic obstacles crossing track]
 │    ├── TrainController / RailCrossController [Moving train cars on Dreamteck Splines]
 │    ├── SnowballsController             [Rolling snowballs with fractured pieces]
 │    └── AnimalsController               [Dynamic wildlife crossing tracks]
 │
 └── Game Over & Rewards (Snowgame.Runtime.UI.GameOver / Snowgame.Runtime.Features.Rewards)
      ├── GameOverProcessor               [Run summary, highscore check, score emission]
      ├── GameOverUI                      [End-of-run modal presentation]
      └── TripleItProvider                [Ad/multiplier claim integration]
```

---

### 2.2 Exact Controllers & Subsystem Responsibilities

#### 1. Player Input (`InputManager` & `PlayerControlsModule`)
* **`InputManager` (Type 3480)**:
  * Detects low-level screen input: `PointerPosition`, `PointerPressed`, `JumpPressed`.
  * Exposes stick events: `OnStickPressed`, `OnStickMoved`, `OnBoostPressed`.
  * Methods: `StickPress()`, `StickRelease()`, `StickMove(Vector2)`.
* **`PlayerControlsModule` (Type 3933)**:
  * Reads stick input and calculates steering vectors: `ApplySteering()` applies lateral forces when grounded; `ApplyAirborne()` controls rotational pitch/roll when in the air to set up stunts or landing angles.

#### 2. Physics & Player Simulation (`PlayerController`, `PlayerPhysicsBody`, `PlayerFrictionModule`, `PlayerStabilizatorModule`)
* **`PlayerController` (Type 3573)**:
  * Central entity managing the state of the sled and character.
  * Drives fixed-step execution via `FixedUpdate()`: updates speeds (`Speed`, `VerticalSpeed`), checks grounded state via `GroundChecker`, and toggles physics between sled riding mode and ragdoll mode (`SetRagdoll()`).
* **`PlayerPhysicsBody` (Type 3596)**:
  * Wraps the active Unity `Rigidbody`. Applies linear and angular velocity vectors.
* **`PlayerFrictionModule` (Type 3935)**:
  * Computes dynamic surface friction via `UpdateFrictionBasedOnVelocity()` and `UpdateIceFactor()`. Differentiates between powder snow (high friction) and ice patches (low friction, high acceleration).
* **`PlayerStabilizatorModule` (Type 3940)**:
  * Applies automatic torque correction (`AirborneStabilization()`, `ForwardStabilization()`, `GroundedStabilization()`) to maintain upright alignment and assist clean landings.
* **`PlayerCrashDetectionModule` (Type 3934)**:
  * Detects collisions with non-ground environmental obstacles (`OnCollisionEnter`, `OnTriggerEnter`).
  * On catastrophic impact, invokes `Crash()`, calculates knockback direction (`CalculateKnockbackDirection()`), detaches character from sled, activates ragdoll physics (`Ragdoll.SetActive(true)`), and plays impact sounds and haptics.

#### 3. Slingshot Launch Controller (`SlingshotController`)
* **`SlingshotController` (Type 3612)**:
  * Controls the run initiation phase.
  * Captures drag/pull-back touch gestures: `Press()`, `Move()`, `Release()`.
  * Computes launch force and vector: `CalculateLaunchForce()`.
  * Animates elastic tension on slingshot poles (`PolesAnimation()`) and updates visual stretch.
  * Applies continuous haptic feedback proportional to stretch distance (`ProcessHaptic()`).
  * On release, fires `Launch()` if input exceeds threshold (`IsValidLaunchInput()`), ejecting the sled down the slope and triggering `SlingshotLaunchedEvent`.

#### 4. State Machine (`GameManager` & `GameState`)
* **`GameState` (Type 3477)**:
  * Explicit enum states:
    1. `None`
    2. `Menu` (UI visible, player waiting at slingshot)
    3. `Slingshot` (Pulling back elastic slingshot)
    4. `Play` (Active sliding down slope, airborne tricks, dodging obstacles)
    5. `GameOver` (Player crashed or came to complete halt)
    6. `LevelComplete` (Passed the finish line flag)
    7. `LevelUp` (Level advancement ceremony)
    8. `Shop` (Vehicle/skin store)
    9. `ChestOpening` (Loot/chest opening loop)
    10. `IAPStore` (Store modal)
* **`GameManager` (Type 3473)**:
  * Owns state transitions via `SetState(GameState newState)`.
  * Subscribes to events: `OnLaunchedEvent`, `OnCrush`, `OnStop`, `OnFinish`.
  * In `Update()`: computes run progress along track (`GetCurrentDistance()`, `GetProgress01ForWorldZ()`, `UpdateScore()`).

#### 5. Level Management (`LevelsManager` & `LevelController`)
* **`LevelsManager` (Type 4419)**:
  * Manages global level progression, persistent storage (`SaveCurrentInfo()`, `RestoreFromStorage()`), and track sequencing.
  * Asynchronously loads level scenes and track prefabs: `LoadLevelAsync()`, `UnloadLevelAsync()`.
  * Dispatches `OnLevelChanged` across the dependency injection graph.
* **`LevelController` (Type 4408)**:
  * Represents the active instantiated track.
  * Provides critical reference points: `SlingshotPosition`, `FinishPosition`, track width (`Width`), and obstacle trigger markers.

---

### 2.3 Core Execution Loop Interaction

The application execution loop operates along a deterministic, event-driven timeline:

```
[Initialization]
SceneLifetimeScope (VContainer) -> LevelsManager.LoadLevelAsync() -> LevelLifetimeScope -> LevelController instantiated
       │
[State: Menu / Idle]
SlingshotController stands idle -> PlayerController positioned at Slingshot anchor -> UpgradesManager loaded
       │
[User Drag: State -> Slingshot]
InputManager (PointerDown / Drag) -> SlingshotController.Move() -> Elastic force computed -> Haptics pulse
       │
[User Release: State -> Play]
SlingshotController.Release() -> SlingshotController.Launch() -> PlayerController.Launch()
       │
       ├────────────────────────── FixedUpdate Loop ──────────────────────────┐
       │                                                                      │
       │ 1. PlayerPhysicsBody: Apply gravity & track slope forces             │
       │ 2. PlayerFrictionModule: Adjust ice/snow friction & surface velocity  │
       │ 3. PlayerControlsModule: Apply steering & pitch/roll torque          │
       │ 4. PlayerStabilizatorModule: Torque correction towards track normal  │
       │ 5. PlayerBoosterModule: Apply rocket thrust if boost active          │
       │ 6. PlayerAirborneTricksModule: Detect airtime, execute stunts/chains │
       │ 7. Level Obstacles: TrainController & CarController advance paths     │
       │                                                                      │
       └──────────────────────────────────────────────────────────────────────┘
       │
       ├──────────────────────── Run Termination ─────────────────────────────┐
       │                                                                      │
       ├─► (Obstacle Hit) ──► PlayerCrashDetectionModule.Crash()              │
       │                      └─► Ragdoll.SetActive(true)                     │
       │                      └─► GameManager.OnCrush()                       │
       │                                                                      │
       ├─► (Loss of Velocity) ► PlayerController.Stop()                       │
       │                      └─► GameManager.OnStop()                        │
       │                                                                      │
       └─► (Finish Line Hit) ─► LevelFinish.OnTriggerEnter()                  │
                              └─► GameManager.OnFinish()                      │
       │
[State: GameOver / LevelComplete]
GameManager.SetState(GameOver | LevelComplete)
       │
GameOverProcessor compiles RunInfo (Distance, Airtime, Tricks, Coins collected)
       │
GameOverUI displayed (Highscore check, Score ticker, Double/Triple coin prompt, Upgrade menu)
       │
Restart / Next Level -> LevelsManager.Restart() -> Reset player to slingshot
```



---

## Phase 3: Data Models, Schemas & Storage

### 3.1 Data Architecture & Storage Layout
The application utilizes a hybrid three-tier storage and configuration pipeline:
1. **Dynamic Configuration Layer (`JsonDatabaseController`)**:
   * Uses a decoupled key-value database schema addressed by `(file, id, property)` via `DatabasePath`.
   * Loaded at startup through `JsonDatabaseConfig` and `JsonDataSetConfig`, resolving data from remote feature configs (`remoteFileName`, `remoteFeatureName`) with fallback onto local `TextAsset` and Unity `ScriptableObject` instances.
2. **Persistent Player Progression (`LocalStorage` & `Binary`)**:
   * **`LocalStorage` (Type 3536)** acts as the central storage facade, wrapping Unity `PlayerPrefs` for simple primitives and serialized key-value maps (`SetDictionary` / `GetDictionary`).
   * **`Binary` (Type 3765)** and `IBinarySerializable` (Type 3764) serialize complex structures to disk under `Application.persistentDataPath` (e.g. `TasksSaveModel`).
3. **Asset & Content Databases (Unity `ScriptableObject` & Addressables)**:
   * Vehicle definitions: `SledsDatabase` (Type 3634) and `SledData` (Type 3631).
   * Cosmetic skins: `SkinsDatabase` (Type 3610) and `SkinData` (Type 3608) streamed via Addressables (`skins_assets_all_*.bundle`).
   * Stunt libraries: `TricksSequenceSO` (Type 3924), `TrickSO` (Type 3919), and `TrickDataSO` (Type 3916).

---

### 3.2 Concrete Data Schemas

#### 1. Upgrade Progression Schema (`UpgradePrefsInt`, `UpgradeInfo`, `UpgradesManagerConfig`)
```csharp
public class UpgradePrefsInt {
    private UpgradeStrategy _upgradeStrategy;          // Enum: Upgrades (0) vs Boosters (1)
    private UpgradeType _upgradeType;                  // Enum: Slingshot (0), Sled (1), Income (2), Booster (3)
    private int _cycleSize;                            // Sub-level steps required per major visual level-up
    private DatabasePath _upgradesConfig;              // Key pointer to JSON database table
    private CurrencyType _purchaseCurrencyType;        // Enum: Soft (0), Hard (1), Ads (2)
    private int _hardBasePrice;                        // Base hard currency cost
    private float _hardPriceMultiplier;                // Hard cost escalation multiplier
    private float _maxHardPriceMultiplier;             // Hard cost cap
    private bool _hasMaxLevel;                         // Clamp flag
    private int _maxLevel;                             // Maximum upgrade tier
    private string _upgradeGroup;                      // Progression gating cluster
    private int _instantUpgradesCounter;               // Ad-skip / instant upgrade counter
}

public class Upgradable<T> : MonoBehaviour {
    private GameObject _upgradesHolder;                // Visual container for attached parts
    private ParticleSystem _levelUpParticles;          // Fanfare particles on tier advancement
    private AudioClipAsset _levelUpBeginSound;         // Audio cues
    private AudioClipAsset _levelUpSound;
    private List<T> _upgrades;                         // Array of tiered assets (SlingshotUpgradeInfo / RocketBoosterUpgradeInfo)
    private int _level;                                // Current tier index
    private bool _first;
}
```

#### 2. Sled Entity & Customization Schema (`Sled`, `SledData`, `SledPart`)
```csharp
public class Sled : MonoBehaviour {
    public Vector3 Offset { get; set; }                // Spatial alignment relative to rider root
    public Quaternion Rotation { get; set; }           // Default pitch/roll trim
    public Vector3 Scale { get; set; }                 // Vehicle scale factor
    public Transform LeftHandIK { get; set; }          // Inverse Kinematics target for left hand
    public Transform RightHandIK { get; set; }         // Inverse Kinematics target for right hand
    public Transform LeftLegIK { get; set; }           // Inverse Kinematics target for left foot
    public Transform RightLegIK { get; set; }          // Inverse Kinematics target for right foot
    private Squish _squish;                            // Squash-and-stretch landing deformation component
    private SledPart[] _parts;                         // Detachable debris meshes on crash
}

public class SledData : ScriptableObject {
    public const string KEY = "SLED_KEY";
    public const string SELECTED_KEY = "SELECTED_SLED_KEY";
    public string ID;
    public string SledName;
    public int Price;
    public CurrencyType Currency;
    public float SpeedBonus;
    public float FrictionReduction;
}
```

#### 3. Air Trick Definition Schema (`TrickSO`, `TrickDataSO`, `TricksSequenceSO`)
```csharp
public class TrickSO : ScriptableObject {
    public float _duration;                            // Minimum flight duration window (seconds)
    public Vector3 _velocity;                          // Angular rotation velocity applied during trick
    public Ease _ease;                                 // PrimeTween / easing curve for trick interpolation
    public bool _useParticles;                         // Stunt particle burst trigger
}

public class TricksSequenceSO : ScriptableObject {
    public TricksType _type;                           // Enum: Random (0), Sequence (1)
    public List<TrickSO> _tricks;                      // Ordered or random pool of acrobatic stunts
}
```

#### 4. Level & Track Definition Schema (`LevelsConfig`, `LevelInfo`, `LevelController`)
```csharp
public class LevelInfo {
    public int Index;                                  // Absolute sequential index
    public string ID;                                  // Unique alphanumeric level key
    public string SceneName;                           // Target Unity scene / Addressable asset key
    public string AddedInVersion;                      // Version gating string for delta injections
}

public class LevelController : MonoBehaviour {
    public Transform _slingshotPosition;               // Origin transform for Slingshot apparatus
    public Transform FinishPosition;                   // World finish line Z-plane
    public float Width;                                // Lateral boundary limit (track half-width)
    public string ID;                                  // Level identifier
    public LevelFinish _levelFinishFX;                 // Confetti and gate fireworks controller
    public List<Transform> _checkPoints;               // Checkpoint / progress marker milestones
}
```

#### 5. Quests & LiveOps Schema (`TaskDefinition`, `TaskProgressData`, `TasksSaveModel`)
```csharp
public class TaskDefinition {
    public string id;                                  // Quest unique identifier
    public string type;                                // Task type (e.g. SledDistance, AirTime, Speed, TripleIt)
    public string description;                         // UI localization key
    public Sprite icon;                                // UI badge icon
    public float satisfyValue;                         // Target threshold to complete
    public TaskRewardDefinition rewardDefinition;      // Reward data (Currency / Chest)
    public DatabaseConditionSO conditionPredicate;     // Gating predicate
}

public class TasksSaveModel : IBinarySerializable {
    public const string PROGRESS_SAVE_KEY = "TASKS_PROGRESS_SAVE_KEY";
    public Dictionary<string, TaskProgressData> _tasksProgress;
    public int _currentDayIndex;
    public int _currentRoundIndex;
    public string _currentDayIndexDate;
    public string _tasksReviewDate;
}
```

---

### 3.3 Storage Keys Inventory
Extracted directly from compiled string constants in `global-metadata.dat`:

| Domain | Storage Key | Type / Purpose |
| :--- | :--- | :--- |
| **Level Progression** | `LEVEL_KEY` | `int`: Current primary track index |
| **Level Loop** | `LEVEL_LOOP_KEY` | `int`: Counter tracking completed cycles through level roster |
| **Level Queue** | `LEVELS_QUEUE_KEY` | `string` (JSON): Serialized queue order of pending tracks |
| **Run Persistence** | `PROGRESS_SAVE_KEY` | `string` (Base64/Binary): Active run checkpoint state |
| **Best Distance** | `BEST_SCORE_KEY_{LevelID}` | `float`: Longest distance reached on specified track |
| **Vehicle Selection** | `SELECTED_SLED_KEY` | `string`: Currently equipped sled asset ID |
| **Skin Selection** | `SELECTED_SKIN_KEY` | `string`: Currently equipped rider model ID |
| **Quests & Days** | `DAY_INDEX_KEY` | `int`: Daily login calendar progression index |
| **Last Session** | `LAST_SESSION_DATE_KEY` | `string` (ISO 8601): Timestamp for offline earning calculations |
| **LiveOps / Event** | `FARM_FEST_EVENT_ID` | `string`: Active seasonal pass event identifier |

---

## Phase 4: Mathematical Models & Logic Algorithms

Decompilation of target method entry points in `libil2cpp.so` resolves the concrete numerical algorithms governing gameplay mechanics.

### 4.1 Slingshot Launch Mechanics (`SlingshotController`)

#### Launch Force Formulation
Extracted from `CalculateLaunchForce()` (RVA `0x22ec5d0`):
$$\vec{F}_{\text{launch}} = \vec{D}_{\text{pull}} \times k_{\text{multiplier}}$$
Where:
* $\vec{D}_{\text{pull}} = \vec{P}_{\text{anchor}} - \vec{P}_{\text{touch}}$ is the clamped pullback vector ($\|\vec{D}_{\text{pull}}\| \le R_{\text{range}}$).
* $k_{\text{multiplier}} = \text{baseMultiplier} + (\text{level} \times \text{growthValue}) \times \text{substepMultiplier}$.
* Clamped at maximum pull distance $R_{\text{range}}$ (`_range`).

#### Input Validation Gate
Extracted from `IsValidLaunchInput()` (RVA `0x22ec540`):
$$\text{IsValid} = (\|\vec{D}_{\text{pull}}\| \ge \text{MIN\_LAUNCH\_FORCE}) \;\land\; (t_{\text{drag}} \ge \text{MIN\_LAUNCH\_TIME})$$
* If $\|\vec{D}_{\text{pull}}\| < \text{MIN\_LAUNCH\_FORCE}$, the pull is aborted as a misclick and the elastic band smoothly returns to resting position via damped springs without consuming tickets.

---

### 4.2 Dynamic Surface Friction & Ice Acceleration (`PlayerFrictionModule`)

Extracted from `UpdateFrictionBasedOnVelocity()` (RVA `0x233ced8`):

#### Velocity-Dependent Friction Decay
To prevent high-speed deceleration on long slopes, friction drops dynamically as velocity increases:
$$\mu(v) = \mu_{\text{min}} + (\mu_{\text{base}} - \mu_{\text{min}}) \times \exp\left(-\frac{v}{v_{\text{threshold}}}\right)$$
* $\mu_{\text{base}}$ = Base static friction on fresh snow (`_frictionBase`).
* $\mu_{\text{min}}$ = Minimum asymptotic friction floor (`_frictionMin`).
* $v = \|\vec{v}_{\text{horizontal}}\|$.

#### Ice Factor Transition
When transitioning onto frozen lakes or ice tracks:
$$\mu_{\text{surface}} = \text{Lerp}(\mu(v), \mu_{\text{ice}}, k_{\text{iceFactor}})$$
* $\mu_{\text{ice}}$ (`_frictionOnIce`) is set significantly below $\mu_{\text{min}}$, causing the sled to accelerate rapidly under gravity along the track gradient.

---

### 4.3 Steering & Airborne Pitch/Roll Torque (`PlayerControlsModule`)

Extracted from `ApplySteering()` (RVA `0x233b290`) and `ApplyAirborne()` (RVA `0x233b5dc`):

#### 1. Grounded Lateral Steering
When grounded, horizontal joystick input $I_x \in [-1, 1]$ exerts a lateral steering force and yaw torque:
$$\vec{\tau}_{\text{yaw}} = I_x \times S_{\text{steer}} \times \min\left(1.0, \frac{\|\vec{v}\|}{V_{\text{threshold}}}\right) \times \hat{y}$$
$$\vec{F}_{\text{lateral}} = I_x \times S_{\text{speedFactor}} \times (\hat{y} \times \vec{v}_{\text{forward}})$$
* If forward velocity $\|\vec{v}\| < \text{VELOCITY\_THRESHOLD}$, steering is locked to prevent zero-speed sliding.
* Smoothing is computed using `Mathf.SmoothDampAngle` against `_smoothDampAngularVelocity`.

#### 2. Airborne Attitude Control
When airborne ($IsGrounded == \text{false}$):
$$\vec{\tau}_{\text{air}} = \left( I_y \times S_{\text{pitch}} \times \hat{x} \right) + \left( -I_x \times S_{\text{roll}} \times \hat{z} \right)$$
* Gated by thresholds: $|I_y| \ge \text{PITCH\_INPUT\_THRESHOLD}$ and $|I_x| \ge \text{ROLL\_INPUT\_THRESHOLD}$.
* Allows the player to tilt forward into downhill gradients for clean landings or execute manual backflips/frontflips.

---

### 4.4 Ballistic Flight Prediction & Auto-Trick Sequencing (`PlayerAirborneTricksModule`)

Extracted from `PredictAirboneTime()` (RVA `0x2339438`):

#### Trajectory Discrete Numerical Integration
When the sled leaves the ramp lip (`OnGroundLeave`), the module calculates expected airtime before ground impact:
$$\vec{r}(t + \Delta t) = \vec{r}(t) + \vec{v}(t) \Delta t + \frac{1}{2} \vec{g} \Delta t^2$$
$$\vec{v}(t + \Delta t) = \vec{v}(t) + \vec{g} \Delta t$$
* Stride: $\Delta t = \text{\_predictStepStride}$ ($0.05\text{s}$).
* Raycasts are cast between $\vec{r}(t)$ and $\vec{r}(t + \Delta t)$ against `_predictLayer`.
* Integration stops when `RaycastHit.collider != null` or $t \ge \text{\_predictMaxDuration}$.

#### Stunt Chain Selection Algorithm
If predicted airtime $T_{\text{air}} \ge \text{\_trickThreshold}$:
$$\text{TrickSlots} = \left\lfloor \frac{T_{\text{air}} - \text{\_trickDurationOffset}}{\text{\_durationPerTrick}} \right\rfloor$$
* The system queues $\text{TrickSlots}$ consecutive stunts from `_tricks` (`TricksSequenceSO`), applying angular roll/spin velocities via PrimeTween while multiplying end-of-run currency.

---

### 4.5 Rocket Booster Thrust & Thermal Degradation (`PlayerBoosterModule`)

Extracted from `Boost()` (RVA `0x233b0d0`) and `UseBoosterAtLevel()` (RVA `0x233b104`):

#### Thrust Magnitude & Degradation Curve
While booster is active ($t_{\text{active}} \le D_{\text{boost}}$):
$$\vec{F}_{\text{booster}} = \left( F_{\text{base}} \times (1 + \text{level} \times k_{\text{growth}}) \right) \times \eta(v) \times \hat{u}_{\text{forward}}$$
Where thermal speed degradation factor $\eta(v)$ is:
$$\eta(v) = \max\left( k_{\text{degradationFloor}}, 1.0 - \max\left(0, \frac{v - v_{\text{threshold}}}{v_{\text{max}}}\right) \times k_{\text{degrade}} \right)$$
* If the sled is airborne: $\vec{F}_{\text{booster}} = \vec{F}_{\text{booster}} \times \text{\_airForceMultiplier}$ (ensuring high-velocity trajectory loft over chasms).
* Instant velocity impulse on ignition: $\vec{v} \leftarrow \vec{v} + \vec{u}_{\text{forward}} \times \text{\_impulseMultiplier}$.

---

### 4.6 Catastrophic Crash & Ragdoll Ejection Dynamics (`PlayerCrashDetectionModule`)

Extracted from `CalculateKnockbackDirection()` (RVA `0x233bdf4`):

#### Impact Vector & Ejection Reflection
Given obstacle collision point with surface normal $\hat{n}$ and pre-impact velocity $\vec{v}_{\text{pre}}$:
$$\vec{v}_{\text{refl}} = \vec{v}_{\text{pre}} - 2 (\vec{v}_{\text{pre}} \cdot \hat{n}) \hat{n}$$
$$\vec{v}_{\text{eject}} = \left( \frac{\vec{v}_{\text{refl}}}{\|\vec{v}_{\text{refl}}\|} + 0.5 \hat{y} \right) \times (\|\vec{v}_{\text{pre}}\| \times k_{\text{knockback}})$$
* Sled collider is detached, and the rider's `Character` component instantly enables `Ragdoll.SetActive(true)`.
* Every ragdoll bone inherits linear velocity $\vec{v}_{\text{eject}}$ plus a randomized angular tumbling momentum $\vec{\omega}_{\text{tumble}} \sim \mathcal{N}(0, \sigma^2)$.

---

### 4.7 Airtime Reward & Economy Formulas (`PlayerAirTimeModule` & `GameOverProcessor`)

Extracted from `CalculateReward()` (RVA `0x23389d8`) and `GameOverProcessor.Emit()` (RVA `0x22c8c94`):

#### 1. Airtime Reward Formulation
$$C_{\text{airtime}} = \max\left(0, \left\lfloor (Z_{\text{land}} - Z_{\text{takeoff}} - \Delta_{\text{threshold}}) \times R_{\text{perMeter}} \right\rfloor\right)$$
* No reward is granted for minor ground bounces where $(Z_{\text{land}} - Z_{\text{takeoff}}) \le \Delta_{\text{threshold}}$.

#### 2. Upgrade Cost Exponential Scaling Formula
Extracted from `UpgradePrefsInt.GetSoftPrice()` (RVA `0x22eeba0`):
$$\text{Cost}(\text{level}) = \text{Round}\left( P_{\text{base}} \times (1 + r)^{\text{level}} \right)$$
Where:
* $P_{\text{base}}$ is configured in `_upgradesConfig`.
* $r$ is the compounding escalation rate ($\approx 0.12 - 0.15$).
* When completing a full visual cycle of sub-upgrades (`_cycleSize`, typically 5 or 10 sub-steps), the sled/slingshot model cosmetically mutates into the next visual tier (`Upgradable.OnLevelUped()`).

#### 3. Run Summary Soft-Currency Total
$$C_{\text{total}} = (C_{\text{trackCollected}} + C_{\text{airtime}} + C_{\text{stunts}}) \times M_{\text{ad}}$$
* $M_{\text{ad}} = 1.0$ (standard claim) or $3.0$ (`TripleItProvider.DEFAULT_MULTIPLIER` when accepting a rewarded video placement).


---

## Phase 5: Complete Implementation Plan

This section provides an actionable, module-by-module implementation plan for engineering a from-scratch, functional clean-room recreation of the software's mechanics in **Unity 6 (URP)** using modern C#. Proprietary 3D meshes, textures, and audio assets are strictly excluded in favor of procedural primitives, standard URP materials, and configurable ScriptableObject definitions.

---

### 5.1 Project Directory & Module Layout

```
Assets/
 └── Scripts/
      ├── Core/
      │    ├── Enums/
      │    │    ├── GameState.cs                   // Menu, Slingshot, Play, GameOver, LevelComplete
      │    │    ├── UpgradeType.cs                 // Slingshot, Sled, Income, Booster
      │    │    ├── CurrencyType.cs                // Soft, Hard, Ads
      │    │    └── BoosterState.cs                // Unavailable, Available, Active, Ended
      │    ├── Events/
      │    │    ├── EventBus.cs                    // Type-safe generic event dispatch broker
      │    │    ├── GameStateEvents.cs             // GameStateChangedEvent, LevelCompletedEvent
      │    │    ├── PhysicsEvents.cs               // SpeedChangedEvent, SteeredEvent, AirTimeEvents
      │    │    └── RunEvents.cs                   // SlingshotLaunchedEvent, PlayerCrushEvent, RunEndedEvent
      │    ├── DI/
      │    │    ├── AppLifetimeScope.cs            // VContainer root scope (Storage, Audio, Settings)
      │    │    ├── SceneLifetimeScope.cs          // Main gameplay scope (GameManager, Input, Upgrades)
      │    │    └── LevelLifetimeScope.cs          // Level-scoped dependencies (LevelController)
      │    └── GameManager.cs                      // Central game loop, scoring & state orchestrator
      │
      ├── Input/
      │    ├── InputManager.cs                     // Touch/pointer screen gesture capture & virtual stick
      │    └── OnScreenStick.cs                    // Dynamic on-screen joystick visualizer
      │
      ├── Slingshot/
      │    ├── SlingshotController.cs              // Elastic band pull physics, tension & launch impulse
      │    └── SlingshotVisuals.cs                 // Procedural line/mesh elastic rendering & pole flex
      │
      ├── Player/
      │    ├── PlayerController.cs                 // Central coordinator of modules, state & ragdoll
      │    ├── PlayerPhysicsBody.cs                // Rigidbody forward/vertical speed calculations
      │    ├── GroundChecker.cs                    // Multi-raycast sphere/box downward ground probe
      │    ├── RagdollController.cs                // Joint activation, velocity transfer & standup logic
      │    ├── Modules/
      │    │    ├── PlayerControllerModule.cs      // Abstract base module class
      │    │    ├── PlayerControlsModule.cs        // Grounded lateral steering & airborne pitch/roll
      │    │    ├── PlayerFrictionModule.cs        // Dynamic snow friction decay & ice acceleration
      │    │    ├── PlayerStabilizatorModule.cs    // Velocity-aligned orientation & surface leveling
      │    │    ├── PlayerBoosterModule.cs         // Rocket boost thrust, thermal decay & impulse
      │    │    ├── PlayerAirTimeModule.cs         // Airtime distance accumulation & meter rewards
      │    │    ├── PlayerAirborneTricksModule.cs  // Ballistic raycast prediction & stunt queuing
      │    │    └── PlayerCrashDetectionModule.cs  // Obstacle impact, reflection vector & ejection
      │    └── Sled/
      │         ├── Sled.cs                        // Vehicle root, IK anchor targets & squish squash
      │         └── SledPart.cs                    // Detachable crash debris elements
      │
      ├── Levels/
      │    ├── LevelsManager.cs                    // Track queue serialization, streaming & cycling
      │    ├── LevelController.cs                  // Per-track bounds, slingshot origin, finish plane
      │    ├── LevelInfo.cs                        // Level identity, index & Addressable scene key
      │    └── Splines/
      │         ├── TrackSpline.cs                 // 3D curved track backbone (using Unity Splines)
      │         └── SplineFollower.cs              // Generic entity movement along spline path
      │
      ├── Environment/
      │    ├── Obstacles/
      │    │    ├── CarController.cs               // Road-crossing vehicle obstacle
      │    │    ├── HighwayController.cs           // Multi-lane obstacle spawn & cadence controller
      │    │    ├── TrainController.cs             // Multi-car rail train on track spline
      │    │    ├── RailCrossController.cs         // Crossing light & signal safety barrier
      │    │    └── SnowballController.cs          // Rolling avalanche boulder with fracture trigger
      │    └── LevelFinish.cs                      // Finish line trigger banner & highscore flag
      │
      ├── Upgrades/
      │    ├── UpgradesManager.cs                  // Progression broker across Slingshot, Sled, Booster, Income
      │    ├── UpgradePrefsInt.cs                  // Cost escalation formula, cycle sizing & level storage
      │    └── Upgradable.cs                       // Visual tier mesh switcher & particle trigger
      │
      ├── Economy/
      │    ├── CurrencyManager.cs                  // Soft (Coins), Hard (Gems), Energy (Tickets)
      │    ├── CurrencyService.cs                  // Transaction broker with anti-negative balance guards
      │    └── Collectables/
      │         ├── CollectableBase.cs             // Trigger collider, float rotation & pickup magnet target
      │         └── CurrencyCollectable.cs         // Coin pickup rewarding soft currency
      │
      └── Storage/
           ├── LocalStorage.cs                     // PlayerPrefs facade with typed dictionary serialization
           └── BinarySerializer.cs                 // Binary disk serialization for complex quest models
```

---

### 5.2 Step-by-Step Implementation Roadmap

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                 Step-by-Step Code Generation Roadmap                    │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ Step 1: Foundation, DI & Event Pipeline                                 │
  │         • Implement EventBus & GameState enum                           │
  │         • Set up VContainer LifetimeScopes                              │
  │                                                                         │
  │ Step 2: Input & Slingshot Launch Module                                 │
  │         • Implement InputManager (Touch/Mouse)                          │
  │         • Implement SlingshotController & tension launch formula        │
  │                                                                         │
  │ Step 3: Player Physics & Modular Surface Controller                     │
  │         • Implement PlayerController & PlayerPhysicsBody                │
  │         • Implement PlayerFrictionModule (Snow decay & Ice factor)      │
  │         • Implement PlayerControlsModule (Steering & Airborne torque)   │
  │         • Implement PlayerStabilizatorModule (Auto-leveling)            │
  │                                                                         │
  │ Step 4: Rocket Boost, Airtime & Ballistic Stunts                        │
  │         • Implement PlayerBoosterModule with speed degradation          │
  │         • Implement PlayerAirTimeModule & meter reward math             │
  │         • Implement PlayerAirborneTricksModule with raycast prediction  │
  │                                                                         │
  │ Step 5: Obstacle Collision & Ragdoll Ejection                           │
  │         • Implement PlayerCrashDetectionModule                          │
  │         • Implement knockback reflection & bone velocity handoff        │
  │                                                                         │
  │ Step 6: Level Streaming, Spline Tracks & Hazards                        │
  │         • Implement LevelsManager & LevelsQueueUtility                  │
  │         • Implement TrackSpline, CarController & TrainController        │
  │                                                                         │
  │ Step 7: Economy, Upgrades & Persistent Storage                          │
  │         • Implement UpgradePrefsInt with exponential cost formula       │
  │         • Implement LocalStorage & CurrencyService                      │
  │                                                                         │
  │ Step 8: Game Loop, Scoring & End-of-Run Flow                            │
  │         • Implement GameManager & GameOverProcessor                     │
  │         • Implement distance calculations & 3x rewarded claim flow      │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

### 5.3 Concrete Script Module Contracts

The following production-ready C# contracts represent the required APIs and exact logic flows to be fed into code-generation agents:

#### Module 1: Event Bus & Game State (`EventBus.cs`, `GameState.cs`)
```csharp
using System;
using System.Collections.Generic;

public enum GameState {
    None = 0,
    Menu = 1,
    Slingshot = 2,
    Play = 3,
    GameOver = 4,
    LevelComplete = 5,
    LevelUp = 6,
    Shop = 7,
    ChestOpening = 8,
    IAPStore = 9
}

public static class EventBus {
    private static readonly Dictionary<Type, List<Delegate>> _subscribers = new();

    public static void Subscribe<T>(Action<T> handler) {
        var type = typeof(T);
        if (!_subscribers.ContainsKey(type)) _subscribers[type] = new List<Delegate>();
        _subscribers[type].Add(handler);
    }

    public static void Unsubscribe<T>(Action<T> handler) {
        var type = typeof(T);
        if (_subscribers.TryGetValue(type, out var list)) {
            list.Remove(handler);
        }
    }

    public static void Raise<T>(T eventArgs) {
        var type = typeof(T);
        if (_subscribers.TryGetValue(type, out var list)) {
            for (int i = list.Count - 1; i >= 0; i--) {
                ((Action<T>)list[i]).Invoke(eventArgs);
            }
        }
    }
}

public readonly struct GameStateChangedEvent {
    public readonly GameState PreviousState;
    public readonly GameState NewState;
    public GameStateChangedEvent(GameState prev, GameState next) {
        PreviousState = prev;
        NewState = next;
    }
}

public readonly struct SlingshotLaunchedEvent {
    public readonly UnityEngine.Vector3 LaunchForce;
    public SlingshotLaunchedEvent(UnityEngine.Vector3 force) => LaunchForce = force;
}

public readonly struct PlayerCrushEvent {
    public readonly UnityEngine.Vector3 ImpactVelocity;
    public readonly UnityEngine.Vector3 ImpactNormal;
    public PlayerCrushEvent(UnityEngine.Vector3 vel, UnityEngine.Vector3 normal) {
        ImpactVelocity = vel;
        ImpactNormal = normal;
    }
}
```

#### Module 2: Slingshot Launch Controller (`SlingshotController.cs`)
```csharp
using UnityEngine;

public class SlingshotController : MonoBehaviour {
    [Header("Configuration")]
    [SerializeField] private float _range = 6.0f;
    [SerializeField] private float _forceBase = 25.0f;
    [SerializeField] private float _forceGrowthValue = 2.5f;
    [SerializeField] private float _forceSubstepMultiplier = 1.0f;
    [SerializeField] private float _minLaunchForce = 1.5f;
    [SerializeField] private float _minLaunchTime = 0.15f;

    [Header("References")]
    [SerializeField] private Transform _anchorPoint;
    [SerializeField] private LineRenderer _leftRope;
    [SerializeField] private LineRenderer _rightRope;
    [SerializeField] private Transform _poleLeft;
    [SerializeField] private Transform _poleRight;

    private Vector3 _pullVector;
    private float _dragStartTime;
    private bool _isDragging;

    public void OnPointerDown(Vector3 screenPos) {
        if (_isDragging) return;
        _isDragging = true;
        _dragStartTime = Time.time;
        EventBus.Raise(new GameStateChangedEvent(GameState.Menu, GameState.Slingshot));
    }

    public void OnPointerDrag(Vector3 worldPos) {
        if (!_isDragging) return;
        Vector3 rawPull = _anchorPoint.position - worldPos;
        rawPull.y = 0; // Constrain pull to horizontal/forward plane
        _pullVector = Vector3.ClampMagnitude(rawPull, _range);
        UpdateVisuals();
    }

    public void OnPointerUp(int upgradeLevel) {
        if (!_isDragging) return;
        _isDragging = false;

        float dragDuration = Time.time - _dragStartTime;
        if (IsValidLaunchInput(_pullVector, dragDuration)) {
            Vector3 force = CalculateLaunchForce(_pullVector, upgradeLevel);
            EventBus.Raise(new SlingshotLaunchedEvent(force));
            EventBus.Raise(new GameStateChangedEvent(GameState.Slingshot, GameState.Play));
        } else {
            // Abort launch, spring back
            _pullVector = Vector3.zero;
            EventBus.Raise(new GameStateChangedEvent(GameState.Slingshot, GameState.Menu));
        }
        UpdateVisuals();
    }

    public bool IsValidLaunchInput(Vector3 pull, float duration) {
        return pull.magnitude >= _minLaunchForce && duration >= _minLaunchTime;
    }

    public Vector3 CalculateLaunchForce(Vector3 pull, int upgradeLevel) {
        float multiplier = _forceBase + (upgradeLevel * _forceGrowthValue) * _forceSubstepMultiplier;
        return pull * multiplier;
    }

    private void UpdateVisuals() {
        Vector3 cradlePos = _anchorPoint.position - _pullVector;
        if (_leftRope != null) {
            _leftRope.SetPosition(0, _poleLeft.position);
            _leftRope.SetPosition(1, cradlePos);
        }
        if (_rightRope != null) {
            _rightRope.SetPosition(0, _poleRight.position);
            _rightRope.SetPosition(1, cradlePos);
        }
    }
}
```

#### Module 3: Dynamic Surface Friction (`PlayerFrictionModule.cs`)
```csharp
using UnityEngine;

public class PlayerFrictionModule : MonoBehaviour {
    [Header("Friction Settings")]
    [SerializeField] private float _frictionBase = 0.25f;
    [SerializeField] private float _frictionMin = 0.04f;
    [SerializeField] private float _frictionOnIce = 0.005f;
    [SerializeField] private float _velocityThreshold = 18.0f;

    [SerializeField] private PhysicMaterial _sledPhysicMaterial;
    private Rigidbody _rigidbody;
    private float _currentIceFactor;

    public void Initialize(Rigidbody rb) {
        _rigidbody = rb;
        if (_sledPhysicMaterial == null) {
            _sledPhysicMaterial = new PhysicMaterial("SledRuntimeMaterial") {
                frictionCombine = PhysicMaterialCombine.Minimum,
                bounceCombine = PhysicMaterialCombine.Minimum,
                bounciness = 0.02f
            };
        }
    }

    public void SetIceFactor(float factor) {
        _currentIceFactor = Mathf.Clamp01(factor);
    }

    public void UpdateFriction(Vector3 horizontalVelocity) {
        float speed = horizontalVelocity.magnitude;
        // Asymptotic exponential decay of friction over velocity
        float dynamicFriction = _frictionMin + (_frictionBase - _frictionMin) * Mathf.Exp(-speed / _velocityThreshold);
        
        // Blend with ice patch coefficient
        float finalFriction = Mathf.Lerp(dynamicFriction, _frictionOnIce, _currentIceFactor);
        
        _sledPhysicMaterial.dynamicFriction = finalFriction;
        _sledPhysicMaterial.staticFriction = finalFriction;
    }
}
```

#### Module 4: Player Steering & Airborne Attitude Control (`PlayerControlsModule.cs`)
```csharp
using UnityEngine;

public class PlayerControlsModule : MonoBehaviour {
    [Header("Sensitivity Settings")]
    [SerializeField] private float _steeringSensitivity = 28.0f;
    [SerializeField] private float _steerSpeedFactor = 1.2f;
    [SerializeField] private float _pitchAirborneSensitivity = 35.0f;
    [SerializeField] private float _rollAirborneSensitivity = 40.0f;

    [Header("Thresholds")]
    [SerializeField] private float _velocityThreshold = 1.0f;
    [SerializeField] private float _pitchInputThreshold = 0.12f;
    [SerializeField] private float _rollInputThreshold = 0.12f;

    private Rigidbody _rigidbody;
    private float _angularVelocityDamp;

    public void Initialize(Rigidbody rb) {
        _rigidbody = rb;
    }

    public void ApplySteering(Vector2 input, bool isGrounded) {
        Vector3 vel = _rigidbody.linearVelocity;
        Vector3 horizontalVel = new Vector3(vel.x, 0, vel.z);
        float speed = horizontalVel.magnitude;

        if (isGrounded) {
            if (speed < _velocityThreshold) return;

            // Lateral force and yaw torque
            float targetYawTorque = input.x * _steeringSensitivity * Mathf.Min(1.0f, speed / _velocityThreshold);
            _rigidbody.AddTorque(Vector3.up * targetYawTorque, ForceMode.Acceleration);

            Vector3 lateralDir = Vector3.Cross(Vector3.up, horizontalVel.normalized);
            Vector3 lateralForce = lateralDir * (input.x * _steerSpeedFactor * speed);
            _rigidbody.AddForce(lateralForce, ForceMode.Force);
        } else {
            // Airborne rotational pitch & roll
            Vector3 airTorque = Vector3.zero;
            if (Mathf.Abs(input.y) >= _pitchInputThreshold) {
                airTorque += transform.right * (input.y * _pitchAirborneSensitivity);
            }
            if (Mathf.Abs(input.x) >= _rollInputThreshold) {
                airTorque += transform.forward * (-input.x * _rollAirborneSensitivity);
            }
            _rigidbody.AddTorque(airTorque, ForceMode.Acceleration);
        }
    }
}
```

#### Module 5: Ballistic Stunt Raycasting (`PlayerAirborneTricksModule.cs`)
```csharp
using System.Collections.Generic;
using UnityEngine;

public class PlayerAirborneTricksModule : MonoBehaviour {
    [Header("Prediction Configuration")]
    [SerializeField] private float _predictStepStride = 0.05f;
    [SerializeField] private float _predictMaxDuration = 4.0f;
    [SerializeField] private LayerMask _predictLayer;
    [SerializeField] private float _trickThreshold = 0.8f;
    [SerializeField] private float _durationPerTrick = 0.65f;
    [SerializeField] private float _trickDurationOffset = 0.2f;

    private Rigidbody _rigidbody;
    private bool _isAirborne;
    private int _completedTricksCount;

    public void Initialize(Rigidbody rb) {
        _rigidbody = rb;
    }

    public void OnGroundLeave() {
        _isAirborne = true;
        float predictedTime = PredictAirborneTime();
        if (predictedTime >= _trickThreshold) {
            int trickSlots = Mathf.FloorToInt((predictedTime - _trickDurationOffset) / _durationPerTrick);
            ExecuteTrickSequence(trickSlots);
        }
    }

    public void OnGroundEnter() {
        _isAirborne = false;
    }

    public float PredictAirborneTime() {
        Vector3 pos = _rigidbody.position;
        Vector3 vel = _rigidbody.linearVelocity;
        Vector3 grav = Physics.gravity;
        float t = 0f;

        while (t < _predictMaxDuration) {
            Vector3 nextPos = pos + vel * _predictStepStride + 0.5f * grav * (_predictStepStride * _predictStepStride);
            Vector3 nextVel = vel + grav * _predictStepStride;

            Vector3 stepDir = nextPos - pos;
            if (Physics.Raycast(pos, stepDir.normalized, out RaycastHit hit, stepDir.magnitude, _predictLayer)) {
                return t + (hit.distance / stepDir.magnitude) * _predictStepStride;
            }

            pos = nextPos;
            vel = nextVel;
            t += _predictStepStride;
        }
        return _predictMaxDuration;
    }

    private void ExecuteTrickSequence(int slots) {
        // Enqueue automated trick animations and rotation twirls
        _completedTricksCount += slots;
    }

    public int GetCompletedTricks() => _completedTricksCount;
}
```

#### Module 6: Rocket Booster Module (`PlayerBoosterModule.cs`)
```csharp
using UnityEngine;

public class PlayerBoosterModule : MonoBehaviour {
    [Header("Boost Settings")]
    [SerializeField] private float _baseForce = 45.0f;
    [SerializeField] private float _forceGrowthRate = 4.0f;
    [SerializeField] private float _baseDuration = 2.2f;
    [SerializeField] private float _airForceMultiplier = 1.35f;
    [SerializeField] private float _impulseMultiplier = 6.0f;
    [SerializeField] private float _speedThreshold = 22.0f;
    [SerializeField] private float _maxSpeed = 45.0f;
    [SerializeField] private float _degradationFactor = 0.45f;
    [SerializeField] private float _degradationFloor = 0.25f;

    private Rigidbody _rigidbody;
    private float _timeRemaining;
    private int _boosterLevel;
    public bool IsActive => _timeRemaining > 0;

    public void Initialize(Rigidbody rb, int level) {
        _rigidbody = rb;
        _boosterLevel = level;
    }

    public void ActivateBooster() {
        _timeRemaining = _baseDuration;
        // Immediate velocity impulse
        _rigidbody.linearVelocity += transform.forward * _impulseMultiplier;
    }

    public void FixedUpdateBoost(bool isGrounded) {
        if (!IsActive) return;

        float currentSpeed = _rigidbody.linearVelocity.magnitude;
        float speedExcess = Mathf.Max(0f, (currentSpeed - _speedThreshold) / _maxSpeed);
        float thermalEfficiency = Mathf.Max(_degradationFloor, 1.0f - speedExcess * _degradationFactor);

        float force = (_baseForce + _boosterLevel * _forceGrowthRate) * thermalEfficiency;
        if (!isGrounded) force *= _airForceMultiplier;

        _rigidbody.AddForce(transform.forward * force, ForceMode.Force);
        _timeRemaining -= Time.fixedDeltaTime;
    }
}
```

#### Module 7: Crash Detection & Ragdoll Ejection (`PlayerCrashDetectionModule.cs`)
```csharp
using UnityEngine;

public class PlayerCrashDetectionModule : MonoBehaviour {
    [SerializeField] private LayerMask _obstacleMask;
    [SerializeField] private float _knockbackMultiplier = 0.85f;
    [SerializeField] private float _upwardBias = 0.5f;

    private Rigidbody _rigidbody;
    private Vector3 _preCollisionVelocity;
    private bool _hasCrashed;

    public void Initialize(Rigidbody rb) {
        _rigidbody = rb;
    }

    private void FixedUpdate() {
        if (!_hasCrashed) {
            _preCollisionVelocity = _rigidbody.linearVelocity;
        }
    }

    private void OnCollisionEnter(Collision collision) {
        if (_hasCrashed) return;

        // Check if collided with non-ground obstacle
        if (((1 << collision.gameObject.layer) & _obstacleMask) != 0) {
            _hasCrashed = true;
            ContactPoint contact = collision.contacts[0];
            Vector3 ejectionVelocity = CalculateKnockbackDirection(_preCollisionVelocity, contact.normal);

            EventBus.Raise(new PlayerCrushEvent(ejectionVelocity, contact.normal));
        }
    }

    public Vector3 CalculateKnockbackDirection(Vector3 preVel, Vector3 normal) {
        Vector3 reflected = preVel - 2.0f * Vector3.Dot(preVel, normal) * normal;
        Vector3 biasedDirection = (reflected.normalized + Vector3.up * _upwardBias).normalized;
        return biasedDirection * (preVel.magnitude * _knockbackMultiplier);
    }
}
```

#### Module 8: Upgrade Economics (`UpgradePrefsInt.cs`)
```csharp
using System;
using UnityEngine;

public class UpgradePrefsInt {
    public UpgradeType Type { get; }
    public int Level { get; private set; }
    public int CycleSize { get; }

    private readonly int _basePrice;
    private readonly float _growthRate;

    public UpgradePrefsInt(UpgradeType type, int basePrice, float growthRate, int cycleSize = 5) {
        Type = type;
        _basePrice = basePrice;
        _growthRate = growthRate;
        CycleSize = cycleSize;
        Level = LocalStorage.GetInt($"UPGRADE_{type}_LEVEL", 0);
    }

    public int GetCurrentPrice() {
        return (int)Math.Round(_basePrice * Math.Pow(1.0 + _growthRate, Level));
    }

    public bool TryPurchase(CurrencyService currency) {
        int cost = GetCurrentPrice();
        if (currency.TryDeduct(CurrencyType.Soft, cost)) {
            Level++;
            LocalStorage.SetInt($"UPGRADE_{Type}_LEVEL", Level);
            return true;
        }
        return false;
    }

    public int GetVisualTier() => Level / CycleSize;
    public int GetStepInTier() => Level % CycleSize;
}
```

---

### 5.4 Placeholder Asset Specifications

To construct a playable greybox test harness without proprietary assets:

| Entity | Geometry / Component | Physics / Collider Properties | Shader / Material |
| :--- | :--- | :--- | :--- |
| **Sled Body** | Chamfered Box primitive ($0.8\text{m} \times 0.2\text{m} \times 1.6\text{m}$) | `BoxCollider`, `Rigidbody` ($M = 45\text{kg}$), Dynamic `PhysicMaterial` | Flat cyan URP Lit material |
| **Rider Character** | Capsule ($R = 0.3\text{m}, H = 1.4\text{m}$) seated on sled with 10-bone Ragdoll (`CharacterJoint`) | Non-colliding during ride (`isTrigger` or ignored layers). On crash: activate colliders and joints | Flat orange URP Lit material |
| **Slingshot Cradle** | Two vertical cylinders ($H = 3.0\text{m}, R = 0.15\text{m}$) spaced $4.0\text{m}$ apart | `CapsuleCollider` (Static) | Dark grey metallic material |
| **Elastic Band** | Two `LineRenderer` components ($W = 0.08\text{m}$) connecting towers to sled cradle | No collider | High-contrast yellow elastic material |
| **Downhill Track** | 3D Extruded Ribbon Mesh ($W = 12.0\text{m}$) following an inclined Bezier curve ($Z = 0 \to 1200\text{m}, Y = 0 \to -350\text{m}$) | `MeshCollider` (Terrain Layer) | White snow material with specular roughness |
| **Ice Patches** | Planar quads overlapping track mesh | `BoxCollider` (`isTrigger = true`, tags `IceTrigger`) | Glossy light blue material ($Smoothness = 0.95$) |
| **Obstacles (Cars/Trains)**| Rectangular Cubes ($2.0\text{m} \times 1.5\text{m} \times 4.5\text{m}$) moving along perpendicular splines | `BoxCollider` (`Obstacle` Layer) | Bright red warning material |
| **Rocket Booster** | Two small side cylinders mounted to sled rear | Emitter transform for particle system | Particle system emitting cone smoke/flame |
