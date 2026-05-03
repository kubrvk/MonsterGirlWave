# Monster Girl Wave
<img align="left" width="50%"   src="https://github.com/kubrvk/MonsterGirlWave/blob/main/Content/MonsterGirlWave/banner.jpg?raw=true"/>
<h3> <a href="https://play.google.com/store/apps/details?id=com.Kubrick.MonsterGirlWave"><img src="https://img.shields.io/badge/Google_Play:-com.Kubrick.MonsterGirlWave-000000?style=flat-square&logo=google-play&logoColor=white&labelColor=000000" height="25"/> </a></h3>

![](https://img.shields.io/badge/Mobile-0c5299?style=) ![](https://img.shields.io/badge/Wave--Rush-a17736?style=) ![](https://img.shields.io/badge/Random--Enemy-7b1717?style=) ![Blueprint](https://img.shields.io/badge/Blueprint-00599C?style=logo=c%2B%2B&logoColor=white)  ![Blueprint](https://img.shields.io/badge/Unreal_Engine_5.7-0E1128?style=for-the-badges&logo=unrealengine&logoColor=white)  ![Blueprint](https://img.shields.io/badge/Status-Shipped-success?style=for-the-badges) 
<br>
Fast-paced top-down survival game targeting Android. Face endless waves of procedurally composed enemies, escalating in size and diversity. Between waves, customize your build from a large weapon pool spanning firearms, launchers, and energy weapons.<br>

The game operates on a **wave-rush loop**: survive the wave, upgrade, repeat , with global leaderboards and weekly tournament structures providing competitive longevity. The core technical challenges were: building a performant top-down enemy swarm system that handles high simultaneous enemy counts on mobile hardware, designing a wave composition engine that produces escalating difficulty without repetition, and sustaining 60 fps under dense particle and projectile load on mid-range Android devices.
<br clear="left"/>
<p align="center">
<img src="https://play-lh.googleusercontent.com/xdiJa3CxPyRcLTwJH_Yk7udN9ikT4dnzqw3QLMDNmiR0wcnC-OAE82kwTWIYJahTWT6O8oOazvLDV9KKvJyf=w2560-h1440-rw" width="25%"/><img src="https://play-lh.googleusercontent.com/U1ThnF9l_Gqo0lRM-ENmZaJdQ75htxTguHCx9c2Y14hWfJL8FB7cbODkI05Yz4QOWiaoxcwiFX9V2t1TGaBQjv8=w2560-h1440-rw" width="25%"/><img src="https://play-lh.googleusercontent.com/p3HPLfIhEkOwdaPrsXzIMJUc0Hb4T56YqdQ1_dALwkPIB_AlSGqq8vUsMUd7vRtbCNyFn4jjx7XXrEcmS8GPEg=w2560-h1440-rw" width="25%"/><img src="https://play-lh.googleusercontent.com/N_SunhmeRk1UUn8IekV3LC0n9UFlHZI026ZNZbrzdPoaaAbBIPAzDj9XQpFu0AY8pja5jQIGiYDigh0l3UZ-cA=w2560-h1440-rw" width="25%"/>
</p>

---

## Technical Detail

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5.7 |
| Primary Language | Blueprint (gameplay, AI, systems) |
| Platform | Android (primary) |
| Input | Custom touch input , virtual joystick + auto-aim |
| AI | Unreal Behavior Tree + custom Blueprint EnemyAIController |
| Physics | Chaos , projectile collision, knockback |
| Rendering | Mobile forward renderer, scalable quality tiers |
| Leaderboards | Platform online services (Google Play Games) |
| Build | Android SDK/NDK, Gradle, UE5 Android packaging |
| 3D / 2D Pipeline | Blender, UE5 |

---
<img src="https://play-lh.googleusercontent.com/p3HPLfIhEkOwdaPrsXzIMJUc0Hb4T56YqdQ1_dALwkPIB_AlSGqq8vUsMUd7vRtbCNyFn4jjx7XXrEcmS8GPEg=w2560-h1440-rw" width="100%"/>

# Core Systems

---

## 1. Wave System

`BP_WaveSystem` is the central controller for session progression. It operates as a state machine governing the full wave lifecycle.

### Wave Composition

- Each wave is defined by a `WaveConfig` data structure generated at runtime by `BP_DifficultySystem`.
- `WaveConfig` contains: `WaveIndex` (Integer), `PowerBudget` (Float), `SpawnGroups` (Array of SpawnGroup), `SpawnInterval` (Float), `GroupSpawnDelay` (Float).
- `PowerBudget` is the primary difficulty axis — each enemy type has an `EnemyPowerCost` (Float) in its Data Asset; the composition logic fills the budget with enemy groups.

### Wave Composition Algorithm

`BP_DifficultySystem , ComposeWave(WaveIndex)` generates the `WaveConfig`:

1. Compute `PowerBudget = BaseBudget + (WaveIndex × BudgetScalePerWave)`.
2. Determine unlocked enemy pool — enemy types are gated behind `UnlockWave` thresholds.
3. Fill budget: weighted-random selection from the unlocked pool; each selection deducts `EnemyPowerCost` from the remaining budget.
4. Group selected enemies into spawn groups (spatially clustered spawns).
5. Apply late-wave modifiers at configurable thresholds: speed multipliers, health multipliers, ranged attacker bias.

### Spawn Execution

- `BP_EnemySpawnSystem` executes the `WaveConfig` over time.
- Spawn points are `BP_SpawnPoint` actors placed around the arena perimeter; spawn point selection uses furthest-from-player logic to avoid spawn-camping.
- Groups spawn with `GroupSpawnDelay` between them — staggered arrival prevents single-frame enemy floods.
- Wave complete condition: `ActiveEnemyCount == 0` and `RemainingSpawnQueue` is empty.

---

## 2. Enemy AI System

Each enemy type is a child Blueprint of `BP_EnemyBase` with a dedicated Behavior Tree. Shared base logic lives in `BP_EnemyBase`; type-specific overrides are in child Blueprints and Data Assets.

### Base Enemy Architecture

- `EnemyStats` struct per type: `MaxHP`, `MoveSpeed`, `AttackDamage`, `AttackRange`, `AttackCooldown`, `PowerCost`, `WeakSpotDamageMultiplier`, `EnemyTier`, `UnlockWave`.
- `WeakSpotConfig`: socket name on the Skeletal Mesh, damage multiplier, optional visual indicator toggle.
- All stats defined in `DA_Enemy` (Data Asset) — no hardcoded values inside Blueprint class defaults.

### Enemy Types

| Type | Movement | Attack Style | HP | Weak Spot |
|---|---|---|---|---|
| Runner | Fast sprint toward player | Melee on contact | Low | Back of head |
| Bruiser | Slow advance | Charge attack, knockback on hit | Very High | Exposed core (chest) |
| Ranged Attacker | Maintains distance | Fires projectiles at player | Medium | Head |
| Swarm | Fast, small, group movement | Contact damage | Very Low | None (kill count reliant) |
| Elite | Varied | Phase-based: ranged then melee | High | Changes per phase |

### AI States Per Type

**Runner:**
```
Idle , DetectPlayer , Sprint , MeleeAttack , (cooldown) , Sprint
```

**Bruiser:**
```
Idle , DetectPlayer , Advance , [ChargeDecision] , Charge , Impact , Recovery , Advance
```

**Ranged Attacker:**
```
Idle , DetectPlayer , PositionToRange , FireCooldown , Fire , Reposition , FireCooldown
```

### Weak Spot System

- A `WeakSpot` Scene Component (Sphere Collision) is attached to each enemy, aligned to its designated socket.
- On projectile hit, the system checks if the impact point falls within `WeakSpotRadius` of the weak spot component's world location.
- Weak spot hit: `Damage × WeakSpotDamageMultiplier` is applied; an optional visual pulse plays on confirmation.
- The weak spot location is communicated to the player via a subtle highlight material parameter on the mesh — not a floating icon, preserving visual cleanliness.

### Aggro System

- `BP_AggroManager` (singleton) tracks a Map of `Enemy Reference , Aggro Float`.
- Player actions generate aggro events: taking damage (high aggro), dealing damage (moderate), using area weapons (area aggro broadcast to all enemies in radius).
- Enemies query `BP_AggroManager` to confirm their current target — this allows future multiplayer extension without changing enemy AI logic.
- In single-player, all enemies target the player; the aggro table is primarily used for priority ordering when distraction mechanics are active.

### Knockback Resistance

- `KnockbackResistance` (Float) is a stat in the `EnemyStats` struct — Bruiser has near-full resistance; Runners have none.
- Knockback is applied via **Add Impulse**; effective magnitude: `ImpulseForce × (1 − KnockbackResistance)`.

---

## 3. Weapon System

### Weapon Registry

- All weapons are defined in `DA_Weapon` (Data Asset): `WeaponCategory` Enum (Firearm / Launcher / Energy), fire rate, damage, projectile class, spread, magazine size, reload time, ammo type, upgrade slots.
- `BP_WeaponSystem` maintains the player's active weapon set as an **Array of WeaponInstance** — runtime objects wrapping their Data Asset with current ammo, upgrade state, and cooldown timers.

### Weapon Categories

| Category | Examples | Fire Mode | Special |
|---|---|---|---|
| Firearm | Pistol, SMG, Shotgun, Sniper | Single / Burst / Auto | High ammo, varied spread |
| Launcher | Grenade Launcher, Rocket, Mine Layer | Projectile arc | AoE damage, knockback |
| Energy | Laser, Plasma Cannon, Chain Lightning | Beam / Charge / Bounce | Elemental effects, no reload |

### Fire Logic

- **Auto-fire:** A looping Timer fires at `1 / FireRate` interval while the fire input is held.
- **Burst:** Fires N projectiles with `BurstInterval` delay between each shot, sequenced via a Timer.
- **Beam (Energy):** A Line Trace runs each tick while the weapon is active; applies `DamagePerSecond × DeltaTime` per tick.
- **Charge:** Holding the input accumulates `ChargeLevel` (0–1 over `ChargeTime`); releasing fires a projectile with damage scaled by `ChargeDamageMultiplier × ChargeLevel`.

### Projectile System

- `BP_ProjectileBase` child Blueprints per damage profile: standard, explosive, energy, bouncing.
- All projectiles are managed by `BP_ProjectilePool` — pre-warmed at session start. **No Spawn Actor / Destroy Actor during gameplay** — pool acquire/release only.
- **Explosive:** On hit, fires an **Apply Radial Damage** node with `ExplosionRadius` and a falloff curve.
- **Bouncing:** On hit, reflects direction using **Mirror Vector by Normal**; bounce count tracked; final bounce detonates as explosive.
- **Chain Lightning:** On hit, collects all enemies within `ChainRadius` sorted by distance; applies damage to N nearest using **Apply Damage** with `ChainDamageFalloff` per hop.

### Ammo & Reload

- Ammo pools per `AmmoType` Enum are tracked on `BP_WeaponSystem`.
- Reload: plays the reload Animation Montage, starts a Timer delay, then restores the magazine on completion.
- Energy weapons use a `ManaPool` resource instead of ammo — regenerates passively over time.

### Auto-Aim

- `BP_AutoAimComponent` evaluates all enemies within `AimAssistRadius` and `AimAssistConeAngle` from the player's current fire direction.
- Selects the highest-threat target (nearest to the crosshair vector, within the cone) and applies a soft pull to the fire direction.
- Pull magnitude is configurable — strong enough to compensate for touch input imprecision, subtle enough not to feel like an aimbot.
- **Weak spot auto-aim:** At close range, applies an additional pull toward the nearest enemy's weak spot socket position.

---

## 4. Upgrade System

Post-wave upgrade selection is the primary build expression mechanic. After each wave clear, the player chooses one of three presented upgrade cards.

### Upgrade Card Generation

`BP_UpgradeSystem , GenerateUpgradeOptions(WaveIndex, PlayerBuildState)` produces an Array of 3 `UpgradeOption` structs.

- **Upgrade pool filtering:** Weapon affinity (upgrades relevant to owned weapons weighted higher), build archetype (if the player has 2+ elemental weapons, elemental upgrades are weighted higher), and wave index (power upgrades gated behind minimum wave thresholds).
- **Deduplication:** The same upgrade cannot appear twice in the same offer.
- **Rarity tier:** Common / Rare / Epic — higher tiers appear at lower base weight, scaling up with wave index.

### Upgrade Categories

| Category | Example Effects |
|---|---|
| Weapon Stat | +Damage, +Fire Rate, +Magazine, −Reload Time |
| Projectile Modifier | Adds Pierce, adds Bounce, splits on hit, explodes on hit |
| Player Stat | +Max HP, +Move Speed, +Pickup Radius, +Regen |
| Elemental Add | Adds Burn / Freeze / Shock proc to a weapon |
| Passive Ability | Auto-collect coins in radius, reload on kill, HP on kill |
| Weapon Unlock | Adds a new weapon to the active weapon set |

### Build State

- `PlayerBuildState` struct accumulates all selected upgrades as an **Array of AppliedUpgrade**.
- Each frame, `BP_WeaponSystem` and `BP_CharacterStatComponent` read from `PlayerBuildState` — stats are derived dynamically from base values + all active upgrade modifiers.
- This avoids baking upgrade effects into permanent state, allowing future respec or prestige mechanics without architectural changes.

---

## 5. Touch Input System

Top-down mobile controls use a **left-side virtual joystick** for movement and **right-side tap/hold** for firing, with auto-aim handling targeting.

### Virtual Joystick

- `BP_VirtualJoystick` renders a floating joystick UI anchored to the first touch-down point on the left half of the screen (dynamic anchor — not a fixed position).
- Joystick input vector: `(TouchCurrentPos − JoystickAnchor)` normalized, clamped to `JoystickRadius`.
- **Dead zone:** Input below `DeadZoneRadius` is treated as zero — prevents drift from imprecise thumb placement.
- The resulting `InputVector` is passed to the **Character Movement Component** each tick as the movement direction.

### Fire Input

- Right half of screen: touch-down begins firing (auto-fire active while held).
- No manual aim input — `BP_AutoAimComponent` handles all target selection.
- Tap on right side: fires a single shot. Hold: continuous fire at the weapon's fire rate.

### Multi-Touch

- Left and right zones are tracked independently — simultaneous movement + fire is fully supported.
- Up to 2 simultaneous touch points are handled: index 0 = left zone (movement), index 1 = right zone (fire). Additional touches are ignored.

### Weapon Switch

- Swipe up on right zone while not firing: cycle to next weapon.
- Logic: track `TouchDuration` and `TouchDelta`; if delta exceeds the swipe threshold before the fire threshold duration, classify as a weapon-switch swipe rather than a fire input.

---

## 6. Leaderboard & Tournament System

### Leaderboard

- `BP_LeaderboardSystem` wraps the Google Play Games API via the Online Subsystem interface.
- **Score submission:** `Write Leaderboard Score` is called on run-end with `FinalScore`.
- **Score retrieval:** Reads leaderboards for friends and the global board; results are cached in a `LeaderboardCache` struct for offline display.
- **Display:** `WBP_Leaderboard` renders a paginated board with player rank highlight; async refresh occurs on widget open.

### Weekly Tournament

- `BP_TournamentSystem` manages the weekly cycle state: `CycleStartTime`, `CycleEndTime`, `TournamentLeaderboardID`.
- The cycle is determined server-side; the client queries tournament status on session start.
- A separate leaderboard ID is used per tournament cycle — historical cycles are preserved.
- Reward tiers are defined in `DA_TournamentReward` (Data Asset): rank thresholds , reward item sets.
- **Reward distribution:** On cycle end, server-side validation occurs; the client receives a `PendingReward` struct on next login, claimed via `BP_TournamentSystem , ClaimPendingRewards`.

### Score Composition

```
FinalScore = (WavesCleared × WaveScoreBase)
           + (EnemiesKilled × KillScore)
           + (WeakSpotKills × WeakSpotBonus)
           + (NoDamageBonus if zero damage taken that wave)
```

**Multipliers:** active upgrade count, difficulty modifier (if the player declined upgrades — an intentional hard-mode option).

---

## 7. Mobile Performance & Optimization

**Target: 60 fps on mid-range Android hardware** under peak swarm density (50+ simultaneous enemies, active projectiles, and particle effects).

### Enemy Count Management

- `BP_EnemySpawnSystem` enforces a `MaxSimultaneousEnemies` cap — additional enemies from the wave queue are held until the active count drops below the threshold.
- **Off-screen enemies:** Tick rate reduced to 10 Hz for enemies beyond `FullTickRadius`; full tick only within player proximity.
- **Animation LOD:** Distant enemies run animations at a reduced frame rate via Animation Update Rate settings on the Skeletal Mesh Component.
- **Physics:** Knockback impulses are only applied for enemies within the camera frustum; off-screen enemies receive instant position correction instead.

### Projectile Pooling

- `BP_ProjectilePool` pre-warms N instances of each projectile Blueprint class at session start.
- **No Spawn Actor / Destroy Actor during gameplay** — pool acquire/release only.
- Pool size is tuned per weapon: high fire-rate weapons get larger pools; launcher projectiles get smaller (fewer in flight simultaneously).

### Rendering Budget

- Mobile forward renderer; target < 130 draw calls per frame.
- Same-type enemies use **Hierarchical Instanced Static Mesh Component** — one draw call per enemy type regardless of count.
- Projectile meshes use **Instanced Static Mesh Component** on `BP_ProjectilePool`.
- Texture budget: enemy max 512×512 ASTC; player 1024×1024 ASTC; VFX atlases 512×512.
- Particle cap: Niagara `MaxParticleCount` per system; budget shared across all simultaneous emitters via Niagara Budget Plugin settings.

### Tick Optimization

- `BP_WaveSystem` state transitions are **event-driven**, not polled each tick.
- `BP_LeaderboardSystem` operations are **async and non-blocking** — never executed on the game thread.
- Score updates use an **OnScoreChanged Event Dispatcher** , UI update; not polled per tick.
- `BP_DifficultySystem , ComposeWave` runs during the wave-clear / upgrade phase — never during active combat.

### Adaptive Quality

- `BP_PerformanceSystem` monitors a rolling frame time average.
- Quality tiers control: enemy render distance, particle density cap, shadow quality, and post-process features.
- Tiers drop on sustained frame overrun; restore on sustained recovery.

### Memory

- Enemy assets are streamed per wave tier: Tier 1 enemies are loaded at boot; higher-tier enemies are async-loaded (using **Soft Object References**) as `BP_TournamentSystem` detects the approaching wave.
- All enemy and weapon Data Asset references use **Soft Object References**.
- Pooled actors are never destroyed — GC pressure is near-zero during active gameplay.

### Android-Specific

- Portrait orientation locked.
- `r.MobileHDR=0` — standard forward renderer.
- `r.Shadow.CSM.MaxCascades=1` on the mobile device profile.
- ASTC texture compression; ETC2 fallback via App Bundle split.
- **Haptic feedback:** Play Haptic Feedback on player damage, wave clear, and upgrade selection.
- Google Play Games integration via UE5 Android build config.

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
| Developer count | 1 |
| Engine | Unreal Engine 5.7 |
| Languages | Blueprint |
| Platform | Android |
| Enemy types | 5+ with distinct AI behaviors |
| Weapon categories | 3 (Firearm, Launcher, Energy) |
| Upgrade categories | 6 |
| Online features | Global leaderboard, weekly tournament |
| Gameplay systems | 11 discrete systems (see above) |
| Development tools | UE5 Editor, Android Studio, Blender |

---

## Related Projects

| Project | Description |
|---|---|
| [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) | Mobile precision platformer; touch controls, physics movement , UE5.7 |
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; parkour, time-as-resource , UE5.1 |
| [U.N. Owen Was Her](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) | Third-person horror; AI, bullet-hell boss , UE5.3 |
| [Olympus of the Heavens](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) | Isometric co-op ARPG; 12 bosses, Steam co-op , UE5.3 |
| [Blood Garden](https://kubrik.itch.io/bloodgarden) | Souls-like melee combat; stamina, parry, elemental , UE5.4 |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling , characters, creatures, props, environments |

---

## Developer

**Kubrik** , Developer & 3D Artist  
[ArtStation](https://www.artstation.com/kubrik) · [Google Play](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) · [Steam](https://store.steampowered.com/search/?developer=Kubrik)

---

*All code, art, design, and marketing assets produced by a single developer. No third-party gameplay code or purchased asset packs used in core systems.*
