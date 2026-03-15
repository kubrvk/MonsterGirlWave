# Monster Girl Wave

> **Top-down action survival** — Unreal Engine 5.7 · C++ · Android · Solo Development  
> [ArtStation](https://www.artstation.com/kubrik)

---

## Overview

Monster Girl Wave is a fast-paced top-down action survival game built in Unreal Engine 5.7 using C++, targeting Android. The player faces endless waves of procedurally composed enemy groups, each wave increasing in size, speed, and enemy type diversity. Between waves, a weapon upgrade selection system allows progressive build customization from a large weapon pool spanning firearms, launchers, and energy weapons.

The game operates on a **wave-rush loop**: survive the wave, upgrade, repeat — with global leaderboards and weekly tournament structures providing competitive longevity. The core technical challenges were: building a performant top-down enemy swarm system that handles high simultaneous enemy counts on mobile hardware, designing a wave composition engine that produces escalating difficulty without repetition, and sustaining 60 fps under dense particle and projectile load on mid-range Android devices.

All gameplay systems, enemy AI, wave generation, weapon framework, UI, and assets were developed by a single developer.

---

## Engine & Technical Stack

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5.7 |
| Primary Language | C++ (gameplay, AI, systems) |
| Platform | Android (primary) |
| Input | Custom touch input — virtual joystick + auto-aim |
| AI | Unreal Behavior Tree + custom C++ EnemyAIController |
| Physics | Chaos — projectile collision, knockback |
| Rendering | Mobile forward renderer, scalable quality tiers |
| Leaderboards | Platform online services (Google Play Games) |
| Build | Android SDK/NDK, Gradle, UE5 Android packaging |
| 3D / 2D Pipeline | ZBrush → Maya → Substance Painter → UE5 |

---

## Architecture Overview

```
MonsterGirlWave/
├── Source/
│   ├── Core/
│   │   ├── MGWCharacter.h/.cpp                # Player character, movement, combat
│   │   ├── MGWGameMode.h/.cpp                 # Wave sequencing, state machine, score
│   │   └── MGWPlayerController.h/.cpp         # Touch input routing, HUD init
│   ├── Systems/
│   │   ├── TouchInputSystem/                   # Virtual joystick, aim assist, fire input
│   │   ├── WaveSystem/                         # Wave composition, spawning, escalation
│   │   ├── EnemySpawnSystem/                   # Spawn point selection, density, pooling
│   │   ├── WeaponSystem/                       # Weapon registry, fire logic, projectiles
│   │   ├── UpgradeSystem/                      # Post-wave upgrade selection, build state
│   │   ├── EnemyAISystem/                      # Per-type behavior, weak spot, aggro
│   │   ├── DifficultySystem/                   # Wave power budget, enemy tier unlocks
│   │   ├── ProjectileSystem/                   # Pooled projectiles, hit resolution
│   │   ├── LeaderboardSystem/                  # Score submission, Google Play Games API
│   │   ├── TournamentSystem/                   # Weekly cycle, reward distribution
│   │   └── PerformanceSystem/                  # Adaptive quality, thermal management
│   ├── Enemies/
│   │   ├── MGWEnemyBase.h/.cpp                # Base enemy class, weak spot, aggro
│   │   ├── Runner/                             # Fast, low-HP melee variant
│   │   ├── Bruiser/                            # Slow, high-HP, knockback-resistant
│   │   ├── RangedAttacker/                     # Projectile-firing enemy variant
│   │   └── [Additional variants...]
│   └── UI/
│       ├── HUD/                                # HP bar, wave counter, score, weapon slots
│       ├── UpgradeWidget/                      # Post-wave upgrade card selection
│       ├── LeaderboardWidget/                  # Global board, weekly tournament UI
│       └── Menus/                              # Main menu, loadout, settings
```

---

## Core Systems: Technical Detail

### 1. Wave System

`UWaveSystem` is the central controller for session progression. It operates as a state machine governing the full wave lifecycle.

**Wave State Machine:**

```
LOBBY
  │
  ▼
WAVE_START ──► WAVE_ACTIVE ──► WAVE_CLEAR
                   │                │
                   ▼                ▼
              PLAYER_DEATH     UPGRADE_PHASE
                                    │
                                    ▼
                               WAVE_START (next wave)
```

**Wave Composition:**
- Each wave is defined by a `FWaveConfig` generated at runtime by `UDifficultySystem`.
- `FWaveConfig` contains: `int32 WaveIndex`, `float PowerBudget`, `TArray<FSpawnGroup> SpawnGroups`, `float SpawnInterval`, `float GroupSpawnDelay`.
- `PowerBudget` is the primary difficulty axis — each enemy type has a `float EnemyPowerCost` in its data asset; the composition engine fills the budget with enemy groups.

**Wave Composition Algorithm:**
- `UDifficultySystem::ComposeWave(int32 WaveIndex)` generates the `FWaveConfig`:
  1. Compute `PowerBudget = BaseBudget + (WaveIndex * BudgetScalePerWave)`.
  2. Determine unlocked enemy pool: enemy types gate behind `UnlockWave` thresholds.
  3. Fill budget: weighted-random selection from unlocked pool; each selection deducts `EnemyPowerCost` from remaining budget.
  4. Group selected enemies into spawn groups (spatially clustered spawns).
  5. Apply late-wave modifiers at configurable thresholds: speed multipliers, health multipliers, ranged attacker bias.

```cpp
FWaveConfig UDifficultySystem::ComposeWave(int32 WaveIndex)
{
    FWaveConfig Config;
    Config.WaveIndex = WaveIndex;
    Config.PowerBudget = BaseBudget + (WaveIndex * BudgetScalePerWave);

    float RemainingBudget = Config.PowerBudget;
    FRandomStream Stream(SessionSeed + WaveIndex);

    TArray<UEnemyDataAsset*> UnlockedPool = GetUnlockedEnemyPool(WaveIndex);

    while (RemainingBudget > 0.f && UnlockedPool.Num() > 0)
    {
        UEnemyDataAsset* Selected = SelectWeightedEnemy(UnlockedPool, Stream, WaveIndex);
        if (Selected->PowerCost > RemainingBudget) break;

        Config.SpawnGroups.Last().Enemies.Add(Selected);
        RemainingBudget -= Selected->PowerCost;

        if (Config.SpawnGroups.Last().Enemies.Num() >= MaxGroupSize)
            Config.SpawnGroups.Add(FSpawnGroup());
    }

    ApplyWaveModifiers(Config, WaveIndex);
    return Config;
}
```

**Spawn Execution:**
- `UEnemySpawnSystem` executes the `FWaveConfig` over time.
- Spawn points are `ASpawnPointActor` instances placed around the arena perimeter; spawn point selection uses furthest-from-player logic to avoid spawn-camping.
- Groups spawn with `GroupSpawnDelay` between them — staggered arrival prevents single-frame enemy floods.
- Wave complete condition: `ActiveEnemyCount == 0` and `RemainingSpawnQueue.IsEmpty()`.

---

### 2. Enemy AI System

Each enemy type is a `AMGWEnemyBase` subclass with a dedicated Behavior Tree. Shared base logic is in `AMGWEnemyBase`; type-specific overrides live in subclasses and data assets.

**Base Enemy Architecture:**
- `FEnemyStats` struct per type: `MaxHP`, `MoveSpeed`, `AttackDamage`, `AttackRange`, `AttackCooldown`, `PowerCost`, `WeakSpotDamageMultiplier`, `EnemyTier`, `UnlockWave`.
- `FWeakSpotConfig`: socket name on the skeletal mesh, multiplier, optional visual indicator toggle.
- All stats defined in `UEnemyDataAsset` — no hardcoded values in C++ class bodies.

**Enemy Types:**

| Type | Movement | Attack Style | HP | Weak Spot |
|---|---|---|---|---|
| Runner | Fast sprint toward player | Melee on contact | Low | Back of head |
| Bruiser | Slow advance | Charge attack, knockback on hit | Very High | Exposed core (chest) |
| Ranged Attacker | Maintains distance | Fires projectiles at player | Medium | Head |
| Swarm | Fast, small, group movement | Contact damage | Very Low | None (kill count reliant) |
| Elite | Varied | Phase-based: ranged then melee | High | Changes per phase |

**AI State Per Type:**

*Runner:*
```
Idle → DetectPlayer → Sprint → MeleeAttack → (cooldown) → Sprint
```

*Bruiser:*
```
Idle → DetectPlayer → Advance → [ChargeDecision] → Charge → Impact → Recovery → Advance
```

*Ranged Attacker:*
```
Idle → DetectPlayer → PositionToRange → FireCooldown → Fire → Reposition → FireCooldown
```

**Weak Spot System:**
- `UWeakSpotComponent` attached to each enemy with a socket-aligned `USphereComponent`.
- Hit resolution checks if the projectile's impact point is within `WeakSpotRadius` of the weak spot component's world location.
- Weak spot hit: `Damage *= WeakSpotDamageMultiplier`; optional visual pulse on confirmation.
- Weak spot location communicated to player via a subtle highlight material parameter on the mesh — not a floating icon, preserving visual cleanliness.

**Aggro System:**
- `AGgroManager` singleton tracks `TMap<AMGWEnemyBase*, float> AggroTable`.
- Player actions generate aggro events: taking damage (high aggro), dealing damage (moderate), using area weapons (area aggro to all enemies in radius).
- Enemies query `AggroManager` to confirm target — allows future multi-player extension without changing enemy AI logic.
- All enemies target the player in single-player — aggro table primarily used for priority ordering when player uses distraction mechanics.

**Knockback Resistance:**
- `float KnockbackResistance` stat on `FEnemyStats` — Bruiser class has near-full resistance; Runners have none.
- Knockback applied as `AddImpulse`; effective magnitude: `ImpulseForce * (1.f - KnockbackResistance)`.

---

### 3. Weapon System

**Weapon Registry:**
- All weapons defined via `UWeaponDataAsset`: `EWeaponCategory` (Firearm / Launcher / Energy), fire rate, damage, projectile class, spread, magazine size, reload time, ammo type, upgrade slots.
- `UWeaponSystem` maintains the player's active weapon set as `TArray<UWeaponInstance*>` — runtime instances wrapping their data asset with current ammo, upgrade state, and cooldown timers.

**Weapon Categories:**

| Category | Examples | Fire Mode | Special |
|---|---|---|---|
| Firearm | Pistol, SMG, Shotgun, Sniper | Single / Burst / Auto | High ammo, varied spread |
| Launcher | Grenade Launcher, Rocket, Mine Layer | Projectile arc | AoE damage, knockback |
| Energy | Laser, Plasma Cannon, Chain Lightning | Beam / Charge / Bounce | Elemental effects, no reload |

**Fire Logic:**
- Auto-fire: `FTimerHandle` fires at `1.f / FireRate` interval while fire input held.
- Burst: fires N projectiles with `BurstInterval` delay between each; `BTimerHandle` sequences shots.
- Beam (Energy): `ULineTraceComponent` sweeps each tick while active; applies damage per-tick via `DamagePerSecond * DeltaTime`.
- Charge: hold input accumulates `ChargeLevel` (0–1 over `ChargeTime`); release fires projectile with damage scaled by `ChargeDamageMultiplier * ChargeLevel`.

**Projectile System:**
- `AProjectileBase` subclasses per damage profile: standard, explosive, energy, bouncing.
- All projectiles pooled via `UProjectilePool` — pre-warmed at session start.
- Explosive: on hit, `UGameplayStatics::ApplyRadialDamage` with `ExplosionRadius` and falloff curve.
- Bouncing: on hit, reflects direction via `FVector::MirrorByVector(HitNormal)`; bounce count tracked; final bounce detonates as explosive.
- Chain Lightning: on hit, evaluates `TArray<AMGWEnemyBase*>` within `ChainRadius` sorted by distance; applies damage to N nearest via `UGameplayStatics::ApplyDamage` with `ChainDamageFalloff` per hop.

**Ammo & Reload:**
- Ammo pools per `EAmmoType` tracked on `UWeaponSystem`.
- Reload: plays reload montage, applies `FTimerHandle` delay, restores magazine on completion.
- Energy weapons use a `ManaPool` resource instead of ammo — regenerates over time.

**Auto-Aim:**
- `UAutoAimComponent` evaluates enemies within `AimAssistRadius` and `AimAssistConeAngle` from the player's current fire direction.
- Selects the highest-threat target (nearest to crosshair vector, within cone) and applies a soft pull to the fire direction.
- Pull magnitude configurable — strong enough to compensate for touch input imprecision, subtle enough not to feel like aimbot.
- Weak spot auto-aim: at close range, applies additional pull toward the nearest enemy's weak spot socket position.

---

### 4. Upgrade System

Post-wave upgrade selection is the primary build expression mechanic. After each wave clear, the player selects one of three presented upgrade cards.

**Upgrade Card Generation:**
- `UUpgradeSystem::GenerateUpgradeOptions(int32 WaveIndex, FPlayerBuildState BuildState)` produces a `TArray<FUpgradeOption>` of size 3.
- Upgrade pool filtered by: weapon affinity (upgrades relevant to owned weapons weighted higher), build archetype (if player has 2+ elemental weapons, elemental upgrades weighted higher), and wave index (power upgrades gated behind minimum wave thresholds).
- Deduplication: same upgrade cannot appear twice in the same offer.
- Rarity tier: `Common / Rare / Epic` — higher tiers appear at lower base weight, scaling up with wave index.

**Upgrade Categories:**

| Category | Example Effects |
|---|---|
| Weapon Stat | +damage, +fire rate, +magazine, -reload time |
| Projectile Modifier | Adds pierce, adds bounce, splits on hit, explodes on hit |
| Player Stat | +max HP, +move speed, +pickup radius, +regen |
| Elemental Add | Adds burn/freeze/shock proc to a weapon |
| Passive Ability | Auto-collect coins in radius, reload on kill, HP on kill |
| Weapon Unlock | Adds a new weapon to the active weapon set |

**Build State:**
- `FPlayerBuildState` accumulates all selected upgrades as `TArray<FAppliedUpgrade>`.
- Each frame, `UWeaponSystem` and `UCharacterStatComponent` query `BuildState` — stats are derived dynamically from base values + all active upgrade modifiers.
- This avoids baking upgrade effects into permanent state, allowing future respec or prestige mechanics without architectural changes.

```cpp
float UCharacterStatComponent::GetFinalStat(EPlayerStat Stat) const
{
    float Base = BaseStats.GetStat(Stat);
    float Additive = 0.f;
    float Multiplicative = 1.f;

    for (const FAppliedUpgrade& Upgrade : BuildState->ActiveUpgrades)
    {
        for (const FStatModifier& Mod : Upgrade.StatModifiers)
        {
            if (Mod.TargetStat != Stat) continue;
            if (Mod.Type == EModifierType::Additive) Additive += Mod.Value;
            else Multiplicative *= (1.f + Mod.Value);
        }
    }

    return (Base + Additive) * Multiplicative;
}
```

---

### 5. Touch Input System

Top-down mobile controls use a **left-side virtual joystick** for movement and **right-side tap/hold** for firing, with auto-aim handling targeting.

**Virtual Joystick:**
- `UVirtualJoystickComponent` renders a floating joystick UI anchored to the first touch-down point on the left half of the screen (dynamic anchor — not fixed position).
- Joystick input vector: `(TouchCurrentPos - JoystickAnchor).GetSafeNormal()` clamped to `JoystickRadius`.
- Dead zone: input below `DeadZoneRadius` treated as zero — prevents drift from imprecise thumb position.
- `InputVector` passed to `UCharacterMovementComponent` each tick as movement direction.

**Fire Input:**
- Right half of screen: touch-down begins firing (auto-fire active while held).
- No manual aim input — `UAutoAimComponent` handles target selection.
- Tap on right side: fires single shot; hold: continuous fire at weapon's fire rate.

**Multi-Touch:**
- Left and right zones tracked independently — simultaneous movement + fire fully supported.
- Up to 2 simultaneous touch points handled: index 0 = left zone (movement), index 1 = right zone (fire). Additional touches ignored.

**Weapon Switch:**
- Swipe up on right zone while not firing: cycle to next weapon.
- Implemented via: track `TouchDuration` and `TouchDelta`; if delta exceeds swipe threshold before fire threshold duration, classify as weapon-switch swipe rather than fire input.

---

### 6. Leaderboard & Tournament System

**Leaderboard:**
- `ULeaderboardSystem` wraps Google Play Games API via UE5's `OnlineSubsystem` interface.
- Score submission: `IOnlineLeaderboards::WriteLeaderboardScore` called on run-end with `FinalScore`.
- Score retrieval: `IOnlineLeaderboards::ReadLeaderboardsForFriends` + global board; results cached in `FLeaderboardCache` for offline display.
- Display: `ULeaderboardWidget` renders paginated board with player rank highlight; async refresh on widget open.

**Weekly Tournament:**
- `UTournamentSystem` manages weekly cycle state: `CycleStartTime`, `CycleEndTime`, `TournamentLeaderboardID`.
- Cycle determined server-side; client queries tournament status on session start.
- Separate leaderboard ID per tournament cycle — historical cycles preserved.
- Reward tiers defined in `FTournamentRewardConfig` data asset: rank thresholds → reward item sets.
- Reward distribution: on cycle end, server-side validation; client receives `FPendingReward` on next login, claimed via `UTournamentSystem::ClaimPendingRewards`.

**Score Composition:**
- `FinalScore = (WavesCleared * WaveScoreBase) + (EnemiesKilled * KillScore) + (WeakSpotKills * WeakSpotBonus) + (DamageTaken == 0 per wave ? NoDamageBonus : 0)`.
- Multipliers: active upgrade count, difficulty modifier (if player declined upgrades — an intentional hardmode option).

---

### 7. Mobile Performance & Optimization

Target: **60 fps on mid-range Android hardware** under peak swarm density (50+ simultaneous enemies, active projectiles, particle effects).

**Enemy Count Management:**
- `UEnemySpawnSystem` enforces a `MaxSimultaneousEnemies` cap — additional enemies from the wave queue are held until active count drops below threshold.
- Off-screen enemies: tick rate reduced to 10Hz for enemies beyond `FullTickRadius`; full tick only within player proximity.
- Animation LOD: `USkeletalMeshComponent::AnimationUpdateRateParams` — distant enemies run animations at reduced frame rate.
- Physics: knockback impulses only applied for enemies within camera frustum; off-screen enemies receive instant position correction.

**Projectile Pooling:**
- `UProjectilePool` pre-warms N instances of each projectile class at session start.
- No `SpawnActor` / `DestroyActor` during gameplay — pool acquire/release only.
- Pool size tuned per weapon: high fire-rate weapons get larger pools; launcher projectiles get smaller (fewer in flight simultaneously).

**Rendering Budget:**
- Mobile forward renderer; target < 130 draw calls per frame.
- Enemy meshes use `UHierarchicalInstancedStaticMeshComponent` for same-type enemies — one draw call per enemy type regardless of count.
- Projectile meshes: instanced rendering via `UInstancedStaticMeshComponent` on `UProjectilePool`.
- Texture budget: enemy max 512×512 ASTC; player 1024×1024 ASTC; VFX atlases 512×512.
- Particle cap: Niagara `MaxParticleCount` per system; budget-shared across all simultaneous emitters via `UNiagaraBudgetPlugin` settings.

**Tick Optimization:**
- `UWaveSystem` state transitions: event-driven, not polled.
- `ULeaderboardSystem` operations: async, non-blocking — never on game thread.
- Score update: `OnScoreChanged` delegate → UI update; not polled per tick.
- `UDifficultySystem::ComposeWave`: runs during wave-clear / upgrade phase — never during active combat.

**Adaptive Quality:**
- `UPerformanceSystem` monitors rolling frame time average (identical architecture to [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) and [Real Cat Runner](#)).
- Quality tiers control: enemy render distance, particle density cap, shadow quality, post-process features.
- Tiers drop on sustained overrun; restore on sustained recovery.

**Memory:**
- Enemy assets streamed per wave tier: tier 1 enemies loaded at boot; higher-tier enemies async-loaded as `UTournamentSystem` detects wave approach.
- `TSoftObjectPtr` on all enemy and weapon data asset references.
- Pooled actors never destroyed — GC pressure near-zero during active gameplay.

**Android-Specific:**
- Portrait orientation locked.
- `r.MobileHDR=0` — standard forward renderer.
- `r.Shadow.CSM.MaxCascades=1` on mobile device profile.
- ASTC texture compression; ETC2 fallback via App Bundle split.
- Haptic feedback: `FAndroidApplication::Vibrate` on player damage, wave clear, and upgrade selection.
- Google Play Games integration: Play Games SDK linked via UE5 Android build config.

---

## Build & Packaging

| Setting | Value |
|---|---|
| Minimum SDK | API 26 (Android 8.0) |
| Target SDK | API 34 (Android 14) |
| ABI | arm64-v8a primary, armeabi-v7a fallback |
| Texture Format | ASTC primary, ETC2 fallback (App Bundle split) |
| Orientation | Portrait locked |
| HDR | Disabled (`r.MobileHDR=0`) |
| Shadow Cascades | 1 (mobile device profile) |
| Online Services | Google Play Games SDK |

---

## Performance Targets

| Metric | Target | Approach |
|---|---|---|
| Frame rate | 60 fps at 50+ enemies | HISM instancing, tick LOD, projectile pool |
| RAM | < 200MB | Tiered async asset load, full pooling |
| APK size | < 150MB | App Bundle, ASTC split, asset compression |
| Thermal stability | 30-min session | Adaptive quality, off-screen tick reduction |
| Leaderboard latency | < 2s submit | Async non-blocking submission, local cache |

---

## Development Scope

| Category | Detail |
|---|---|
| Developer count | 1 (solo) |
| Engine | Unreal Engine 5.7 |
| Languages | C++ |
| Platform | Android |
| Enemy types | 5+ with distinct AI behaviors |
| Weapon categories | 3 (Firearm, Launcher, Energy) |
| Upgrade categories | 6 |
| Online features | Global leaderboard, weekly tournament |
| Gameplay systems | 11 discrete systems (see above) |
| Development tools | UE5 Editor, Android Studio, ZBrush, Maya, Substance Painter |

---

## Related Projects

| Project | Description |
|---|---|
| [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) | Mobile precision platformer; touch controls, physics movement — UE5.7 |
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; parkour, time-as-resource — UE5.1 |
| [U.N. Owen Was Her](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) | Third-person horror; AI, bullet-hell boss — UE5.3 |
| [Olympus of the Heavens](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) | Isometric co-op ARPG; 12 bosses, Steam co-op — UE5.3 |
| [Blood Garden](https://kubrik.itch.io/bloodgarden) | Souls-like melee combat; stamina, parry, elemental — UE5.4 |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling — characters, creatures, props, environments |

---

## Developer

**Kubrik** — Developer & 3D Artist  
9 years web development · 7 years 3D modeling · 5 years Unreal Engine C++  
5+ shipped commercial games as sole developer.

[ArtStation](https://www.artstation.com/kubrik) · [Google Play](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) · [Steam](https://store.steampowered.com/search/?developer=Kubrik)

---

*All code, art, design, and marketing assets produced by a single developer. No third-party gameplay code or purchased asset packs used in core systems.*
