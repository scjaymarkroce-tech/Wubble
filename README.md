# Wubble (Godot 4.4.1) — Project Guide

## Goal

Build a **simple top-down roguelike** with reusable components and minimal art/animation. The focus is gameplay systems, not assets.

---

## Tech Stack

- **Engine:** Godot 4.4.1
- **Style:** 2D top-down
- **Animation:** procedural wobble (sine scale/rotation) + AnimatedSprite2D

---

## Project Structure

```
res://
├── scripts/
│   ├── components/
│   │   ├── WobbleComponent.gd
│   │   └── StatsComponent.gd
│   ├── globals/
│   │   └── Effects.gd              (autoload)
│   ├── player/
│   │   ├── Player.gd
│   │   ├── Bow.gd
│   │   └── Arrow.gd
│   ├── enemies/
│   │   ├── Zombie.gd
│   │   ├── Bomb.gd
│   │   ├── Slime.gd
│   │   ├── Ghost.gd
│   │   ├── GhostArrow.gd
│   │   ├── Spidy.gd
│   │   ├── SpidyWeb.gd
│   │   ├── Mushy.gd
│   │   └── MushySpore.gd
│   ├── rooms/
│   │   ├── Room.gd
│   │   └── RoomData.gd
│   ├── ui/
│   │   └── Portal.gd
│   └── world/
│       └── World.gd
├── scenes/
│   ├── player/
│   │   ├── Player.tscn
│   │   └── Arrow.tscn
│   ├── enemies/
│   │   ├── Zombie.tscn
│   │   ├── Bomb.tscn
│   │   ├── Slime.tscn
│   │   ├── Ghost.tscn
│   │   ├── GhostArrow.tscn
│   │   ├── Spidy.tscn
│   │   ├── SpidyWeb.tscn
│   │   ├── Mushy.tscn
│   │   └── MushySpore.tscn
│   ├── rooms/
│   │   ├── RoomHub.tscn
│   │   ├── RoomNormal1.tscn
│   │   ├── RoomNormal2.tscn
│   │   ├── RoomNormal3.tscn
│   │   ├── RoomNormal4.tscn
│   │   ├── RoomNormal5.tscn
│   │   ├── RoomNormal6.tscn
│   │   ├── RoomWitch1.tscn
│   │   ├── RoomBoss.tscn
│   │   └── Portal.tscn
│   └── world/
│       └── World.tscn
└── assets/
    └── shaders/
        └── hit_flash.gdshader
```

---

## Collision Layers

| Layer | Name | Used By |
|-------|------|---------|
| 1 | player | Player body |
| 2 | enemies | All enemy bodies, ContactArea, Hurtbox |
| 3 | player_arrows | Arrow (Area2D) |
| 4 | enemy_projectiles | GhostArrow, SpidyWeb, MushySpore |
| 5 | walls | Boundaries StaticBody2D in each room |

| Node | Layer | Mask |
|------|-------|------|
| Player | 1 | 2, 4 |
| Enemy CharacterBody2D | 2 | 1, 3 |
| ContactArea (on enemies) | 2 | 1 |
| Hurtbox (on enemies) | 2 | 3 |
| Arrow | 3 | 2 |
| Enemy projectiles | 4 | 1 |
| Boundaries | 5 | - |

---

## Core Systems

### 1) Wobble Animation Component

**File:** `scripts/components/WobbleComponent.gd`
**Attach to:** child `Node` inside any `AnimatedSprite2D`
**Call** `set_moving(bool)` from the parent script every physics frame

| Export | Description |
|--------|-------------|
| `speed` | Wobble speed while moving (default 7.0) |
| `amount` | Scale bounce while moving (default 0.03) |
| `x_sway` | Rotation sway while moving (default 0.2) |
| `idle_speed` | Wobble speed while idle (default 2.0) |
| `idle_amount` | Scale bounce while idle (default 0.01) |
| `idle_x_sway` | Rotation sway while idle (default 0.05) |

Wobble automatically switches between moving and idle values based on `set_moving()`.

---

### 2) Stats Component

**File:** `scripts/components/StatsComponent.gd`
**Attach to:** any enemy or player as a child `Node`

| Stat | Type |
|------|------|
| `max_hp` | int |
| `hp` | int |
| `speed` | float |
| `damage` | int |
| `attack_speed` | float |
| `contact_damage` | int |

**Signals:** `on_death`, `hp_changed(new_hp)`
**Methods:** `take_damage(amount)`, `heal(amount)`, `is_alive()`

---

### 3) Effects (Autoload)

**File:** `scripts/globals/Effects.gd`
**Register as:** Autoload named `Effects` in Project Settings

All effects are spawned purely in code using `CPUParticles2D` — no nodes need to be added to any scene.

#### Color Constants

| Constant | Color | Used For |
|----------|-------|---------|
| `BROWN` | Brown | Player arrow impact |
| `DARK_GREEN` | Dark green | Zombie |
| `BLUE` | Blue | Slime |
| `BLACK_SMOKE` | Near black | Bomb |
| `DIRTY_WHITE` | Off-white | Ghost |
| `ORANGE` | Orange | Bomb fire |
| `YELLOW` | Yellow | Bomb flash |
| `RED` | Red | Bomb sparks |
| `SPIDY_RED` | Dark red | Spidy hit/death |
| `SPIDY_WEB` | Near white | SpidyWeb trail |
| `MUSHY_RED` | Red | Mushy hit/death |

#### Public API

| Method | Description |
|--------|-------------|
| `arrow_impact(pos)` | Brown splinter burst, fired on arrow hit or expiry |
| `hit_flash(sprite)` | Flashes only non-transparent pixels white via shader |
| `contact_damage(pos, color)` | Wide burst in enemy color on contact hit |
| `projectile_trail(pos, color)` | Small puff trail for enemy projectiles |
| `explosion(pos)` | 4-layer bomb explosion: smoke, fire, flash, sparks |
| `zombie_death(pos)` | Dark green burst |
| `slime_death(pos)` | Blue burst + downward drip |
| `ghost_death(pos)` | Dirty white burst + upward wisp float |
| `spidy_death(pos)` | Dark red burst + lighter red pop |
| `mushy_death(pos)` | Red burst + white spore spots burst |
| `death_bounce(enemy, on_done)` | Knock up + fall off screen + spin, random left/right |
| `spawn_preview(pos, color)` | 3-layer burst effect at enemy spawn position before enemy appears |

#### Hit Flash Shader

**File:** `assets/shaders/hit_flash.gdshader`
Only pixels with `alpha > 0.01` are flashed white — transparent background is never affected. Shader is created and assigned in code by `hit_flash()`, no manual setup needed.

#### Death Bounce

Called from all enemies except Slime and Bomb. Behavior:
- Flips sprite upside down instantly (negative `scale.y`)
- Knocks enemy upward `-120px` with random left/right drift
- Falls `800px` off screen with spin in knock direction
- `queue_free` fires after 3 second buffer to ensure it is off screen

Slime skips `death_bounce` — spawns children then frees immediately.
Bomb skips `death_bounce` — explosion handles its own exit.


#### Spawn Preview
Called by Room.gd before enemies appear. Returns a Node2D root containing 3 particle layers — caller is responsible for queue_free() on the returned node after the delay:

| Layer | Description | 
|----------|-------|
| `Enemy color burst` | 40 particles, all directions, continuous | 
| `Black accent burst` | 20 smaller dark particles mixed in | 
| `White sparkle burst` | 15 fast bright particles for pop | 

---

## Player

### Scene Tree

```
Player (CharacterBody2D) — Player.gd       group: "player"
├── StatsComponent                          max_hp=3, speed=100
├── AnimatedSprite2D
│   └── WobbleComponent
├── CollisionShape2D
├── Camera2D                                Current=true
└── Bow (Node2D) — Bow.gd
    └── BowAnimatedSpirte2d (AnimatedSprite2D)
```

### Animations (AnimatedSprite2D)

| Animation | Trigger |
|-----------|---------|
| `idle_hp3` | full health, not shooting |
| `idle_hp2` | 2 hp, not shooting |
| `idle_hp1` | 1 hp, not shooting |
| `attack_hp3` | full health, bow pulled/ready |
| `attack_hp2` | 2 hp, bow pulled/ready |
| `attack_hp1` | 1 hp, bow pulled/ready |

### Bow Animations

| Animation | Trigger |
|-----------|---------|
| `default` | idle, not holding |
| `pulled` | holding mouse button |
| `ready` | held long enough (0.3s) |

### Bow Flow

Hold mouse → `pulled` → `ready` → release → spawn Arrow → `default`

### Dash

| Constant | Value |
|----------|-------|
| `DASH_SPEED` | 500.0 |
| `DASH_DURATION` | 0.25 |
| `DASH_COOLDOWN` | 0.9 |
| `GHOST_INTERVAL` | 0.04 |
| `GHOST_COLOR` | Brown at 45% alpha |

- Input: `dash` action (default: Shift)
- Direction priority: keyboard input first, mouse direction if no key pressed
- Sprite rolls (spins) in the dash direction during the dash
- Player is **invincible** during dash — `take_contact_damage` returns early
- Brown ghost trail spawns every `GHOST_INTERVAL` seconds, fades out over 0.2s
- Ghost uses exact current animation frame texture

### Arrow Knockback and Screenshake

- On fire: player receives knockback `150.0` away from shot direction
- Camera shakes strength `2.5` for `0.12s` on every arrow fired
- Both values are constants tunable at top of `Player.gd`

### Death

- _on_death() calls call_deferred("_deferred_death")
- _deferred_death() guards with is_inside_tree() then pauses the tree
- Game over is a placeholder print("GAME OVER") — full UI in future phase
- World.reset() is called externally when UI is implemented to unpause and reload Hub
- 
### Notes

- No sprite flipping
- Bow rotates to follow mouse via `look_at()`
- `set_attacking(bool)` called by Bow to switch player animation between `idle_hpX` and `attack_hpX`
- Death uses call_deferred to avoid physics callback crash
- Contact damage triggers `hit_flash` + red `contact_damage` particles

---

## Enemies

### Shared Scene Tree (all enemies)

```
Enemy (CharacterBody2D)
├── StatsComponent
├── AnimatedSprite2D
│   └── WobbleComponent
├── CollisionShape2D
├── ContactArea (Area2D)     layer 2, mask 1
│   └── CollisionShape2D
└── Hurtbox (Area2D)         layer 2, mask 3
    └── CollisionShape2D
```

### Shared Animations

| Animation | Trigger |
|-----------|---------|
| `walk` | chasing / moving |
| `attack` | attacking / fusing / shooting |

### Arrow Detection Pattern

All enemies use `Hurtbox` (Area2D, mask 3) to detect arrows via `area_entered`. The arrow destroys itself on hitting any area named `"Hurtbox"`. Enemies do **not** rely on `body_entered` from the arrow. Arrow provides `get_damage()` as the duck-type contract all hurtboxes use.

### Shared On-Hit Behavior

Every enemy calls these two lines in `take_damage()`:

```gdscript
Effects.hit_flash(anim)
Effects.contact_damage(global_position, ENEMY_COLOR)
```

### Shared Death Behavior (except Slime and Bomb)

```gdscript
set_physics_process(false)
contact_area.set_deferred("monitoring", false)
hurtbox.set_deferred("monitoring", false)
Effects.death_bounce(self, queue_free)
```

Note: always use `set_deferred("monitoring", false)` — setting it directly inside a signal callback causes a physics lock error.

---

### 1) Zombie ✅

- Chases player continuously
- On contact: plays `attack` animation briefly then returns to `walk`
- Death: dark green burst + death bounce

**Stats:** `speed=60`, `contact_damage=1`, `max_hp=2`

---

### 2) Bomb ✅

- Chases player, plays `walk`
- Within `trigger_distance` → stops, plays `attack`, wobble disabled, shrinks slowly over fuse duration
- After `fuse_time` → pops to 2.2x scale → explodes in radius → damages player + all enemies in radius
- Arrow kill triggers fuse first, does not explode instantly
- Explosion: 4-layer particle effect (smoke, fire, flash, sparks)
- No death bounce — explosion is its exit

**Stats:** `speed=55`, `max_hp=2`
**Exports:** `explosion_radius=80`, `explosion_damage=2`, `fuse_time=1.5`, `trigger_distance=100`
**Scale:** `BASE_SCALE = Vector2(0.17, 0.17)`

---

### 3) Slime ✅

- Chases player
- Contact damage: **1**
- On death: spawns 3 smaller slimes at scaled-down size, spread in a circle
- Child slimes (`is_child=true`) do not spawn further slimes on death
- No death bounce — spawns children then frees immediately via `call_deferred`

**Stats:** `speed=45`, `contact_damage=1`, `max_hp=2`
**Exports:** `slime_scene` (assign `Slime.tscn`), `child_scale=0.55`, `child_count=3`
**Effects color:** Blue

---

### 4) Ghost ✅

- Starts invisible (`modulate.a = 0`)
- Sequence per cycle:
  1. Invisible for `invisible_duration`
  2. Teleports near player
  3. Becomes visible, plays `walk` for `walk_before_attack` (0.3s)
  4. Plays `attack` for `windup_before_shoot` (0.3s)
  5. Fires `shoot_count` arrows with squish scale effect per shot
  6. Short wait → goes invisible, repeats
- Squish per shot: shrinks to 0.6x → pops to 1.35x → returns to base, arrow fires at pop moment
- Uses dedicated `GhostArrow.tscn` (layer 4, mask 1) — not the player Arrow
- **Can only be damaged while visible** (`modulate.a >= 0.5`)
- Death: dirty white burst + upward wisp + death bounce

**Stats:** `speed=0`, `max_hp=3`
**Exports:** `ghost_arrow_scene`, `arrow_speed=200`, `shoot_count=3`, `shoot_interval=0.4`, `invisible_duration=3.0`, `teleport_radius=120`, `walk_before_attack=0.3`, `windup_before_shoot=0.3`
**Scale:** `BASE_SCALE = Vector2(0.14, 0.14)`

---

### 5) Spidy ✅

- Chases player for `chase_time` seconds, then stops and attacks
- Attack: squishes small (hold 0.5s) → pops → fires webs in 4 cardinal directions (N, E, S, W) simultaneously
- Returns to chasing after attack
- Wobble disabled during attack
- Death: dark red burst + death bounce

**Stats:** `speed=65`, `contact_damage=1`, `max_hp=2`
**Exports:** `web_scene` (assign `SpidyWeb.tscn`), `web_speed=160`, `chase_time=2.5`
**Scale:** `BASE_SCALE = Vector2(0.15, 0.15)`
**Effects color:** `SPIDY_RED`

#### SpidyWeb Projectile

```
SpidyWeb (Area2D) — SpidyWeb.gd    layer 4, mask 1
├── Sprite2D                        (web image)
└── CollisionShape2D
```

- Trail color: `SPIDY_WEB` (near white)
- Damage: 1, Lifetime: 2.0s
- Connect: `body_entered → _on_body_entered`, `area_entered → _on_area_entered`

---

### 6) Mushy ✅

- Chases player for `chase_time` seconds, then stops and attacks
- Attack: squishes small (hold 0.5s) → pops → fires spores in 4 diagonal directions (NE, SE, SW, NW) simultaneously
- Spore rotates on its own axis while traveling (`rotate_speed=5.0`)
- Returns to chasing after attack
- Wobble disabled during attack
- Death: red burst + white spore spots burst + death bounce

**Stats:** `speed=55`, `contact_damage=1`, `max_hp=2`
**Exports:** `spore_scene` (assign `MushySpore.tscn`), `spore_speed=140`, `chase_time=2.5`
**Scale:** `BASE_SCALE = Vector2(0.15, 0.15)`
**Effects color:** `MUSHY_RED`

#### MushySpore Projectile

```
MushySpore (Area2D) — MushySpore.gd    layer 4, mask 1
├── Sprite2D                             (spore image)
└── CollisionShape2D
```

- Trail color: `MUSHY_RED`
- Damage: 1, Lifetime: 2.0s, `rotate_speed=5.0`
- Connect: `body_entered → _on_body_entered`, `area_entered → _on_area_entered`

---

## Arrow

### Scene Tree

```
Arrow (Area2D) — Arrow.gd    layer 3, mask 2
├── Sprite2D
└── CollisionShape2D
```

### Signals to connect in scene

- `body_entered → _on_body_entered`
- `area_entered → _on_area_entered`

### Behavior

- Moves in launch direction every `_process`
- Destroys itself on hitting a body or any area named `"Hurtbox"`
- `_hit` flag prevents double-hit in same frame
- `get_damage()` method used by enemy hurtboxes as duck-type contract
- Fires `Effects.arrow_impact()` on any destroy including expiry
- `trail_color` export — set to enemy color for enemy projectiles, leave `Color.TRANSPARENT` for player arrow (no trail)
- `trail_interval` default `0.12` — controls spacing between trail puffs

---

## Input Map

| Action | Input |
|--------|-------|
| `move_up` | W / Up Arrow |
| `move_down` | S / Down Arrow |
| `move_left` | A / Left Arrow |
| `move_right` | D / Right Arrow |
| `shoot` | Mouse Button Left |
| `dash` | Shift |

---
## Room System

### World Scene Tree
World (Node2D) — World.gd
├── RoomContainer (Node)
└── Player (Player.tscn — instanced)

Player lives in World, never inside a Room. World repositions the player to each room's PlayerSpawn on load. This ensures HP and gear persist across rooms without re-instantiation.

## Room Scene Tree (all rooms share this structure)
RoomXxx (Node2D) — Room.gd
├── Boundaries (StaticBody2D)       4 CollisionShape2D surrounding the map (N,E,S,W) — layer 5
├── Tiles (Node2D)
│   ├── grass (TileMapLayer)
│   ├── water (TileMapLayer)
│   └── designs (TileMapLayer)
├── EnemyContainer (Node2D)
├── SpawnArea (Node2D)
│   └── SpawnPolygon (Polygon2D)    draw spawn zone manually in editor, avoiding walls and portal
├── Portals (Node2D)
│   └── Portal (Portal.tscn)        position manually in editor
└── PlayerSpawn (Marker2D)          place at room center


## Portal Scene Tree
Portal (Area2D) — Portal.gd    layer 0, mask 1
├── AnimatedSprite2D               animations: "open", "closed"
└── CollisionShape2D

## Room Sequence
[Hub] → [R1] → [R2] → [Witch] → [R3] → [R4] → [Witch] → [R5] → [R6] → [Witch] → [Boss] → (loop back to Hub)

## Room IDs and Types
| ID | Type | Scene |
|----------|-------|---------|
| `0` | HUB | RoomHub.tscn |
| `1` | NORMAL | RoomNormal1.tscn |
| `2` | NORMAL | RoomNormal2.tscn |
| `3` | WITCH | RoomWitch1.tscn |
| `4` | NORMAL | RoomNormal3.tscn |
| `5` | NORMAL | RoomNormal4.tscn |
| `6` | WITCH | RoomWitch1.tscn |
| `7` | NORMAL | RoomNormal5.tscn |
| `8` | NORMAL | RoomNormal6.tscn |
| `9` | WITCH | RoomWitch1.tscn |
| `10` | BOSS | RoomBoss.tscn |

## Enemy Pool Per Normal Room
| Room | Enemy Pool | 
|----------|-------|
| `N1 (id 1)` | Zombie, Slime |
| `N2 (id 2)` | Zombie, Slime, Bomb |
| `N3 (id 4)` | Zombie, Slime, Bomb, Spidy |
| `N4 (id 5)` | Zombie, Slime, Bomb, Spidy, Mushy|
| `N5 (id 7)` | Zombie, Bomb, Spidy, Mushy, Ghost |
| `N6 (id 8)` | Zombie, Bomb, Spidy, Mushy, Ghost |
| `BOSS (id 10)` | All 6, always 10 enemies |

Each normal room spawns 7–10 enemies (random). Witch and Hub rooms spawn none.

## Enemy Scaling (Cycles)

- Every time the player clears Boss and loops back to Hub, cycle increments by 1
- Every enemy spawned receives +2 max_hp per cycle injected after instantiation via StatsComponent
- Player keeps HP and gear across cycles — only enemies scale

## Enemy Spawn Sequence

- Room loads — all spawn positions calculated immediately from SpawnPolygon
- spawn_preview() particle effect fires at every spawn position simultaneously (3-layer burst in enemy color)
- After SPAWN_DELAY (2.0s) — particles freed, enemies instantiated at their positions
- Each enemy scales from Vector2.ZERO to normal scale over SPAWN_SCALE_DURATION (0.25s) using Tween.TRANS_BACK for a pop-in bounce feel

## Portal Behavior

- All portals start closed on room load (collision disabled, "closed" animation)
- HUB and WITCH rooms open portals immediately in start()
- NORMAL and BOSS rooms open portals only after _enemies_remaining reaches 0
- Portal uses a 2-frame physics delay (await get_tree().physics_frame x2) before detecting player body — prevents instant trigger on spawn

## Game Over (Placeholder)

- Player death pauses the tree (get_tree().paused = true)
- print("GAME OVER") is the current placeholder
- World.reset() exists ready for UI wiring — resets cycle = 0, unpauses, loads Hub
- Full Game Over UI planned for a future phase


## Development Roadmap

### Phase 1 — Core Loop ✅

- [x] WASD movement
- [x] Wobble component (idle + moving states)
- [x] Bow shooting with pull/ready states
- [x] Arrow with collision and lifetime
- [x] Zombie enemy
- [x] Stats component with signals
- [x] Player HP-based animation switching

### Phase 2 — Expand Enemies ✅

- [x] Bomb (fuse + explosion radius + shrink scale during fuse)
- [x] Slime (split on death, no bounce)
- [x] Ghost (invisible + teleport + walk + windup + shoot with squish)
- [x] Spidy (chase + 4-way web burst with squish + pop delay)
- [x] Mushy (chase + 4-diagonal spore burst with squish + pop delay + rotating spore)
- [x] Hurtbox system for arrow detection
- [x] Dedicated GhostArrow projectile (layer 4)
- [x] Collision layer/mask documentation

### Phase 2.5 — Effects ✅

 - [x] Effects autoload (CPUParticles2D, no scene nodes needed)
 - [x] Arrow impact particles (brown)
 - [x] Contact damage particles (per enemy color)
 - [x] Projectile trail (enemy projectiles only)
 - [x] Hit flash shader (non-transparent pixels only)
 - [x] Death particles per enemy (zombie, slime, ghost, spidy, mushy)
 - [x] Bomb explosion (4-layer: smoke, fire, flash, sparks)
 - [x] Death bounce animation (knock up, random direction, spin, fall off screen)
 - [x] Bomb fuse shrink + pop scale tween
 - [x] Ghost squish scale per shot
 - [x] Spidy and Mushy squish + pop delay before firing

### Phase 2.6 — Player Feel ✅

 - [x] Dash with invincibility frames
 - [x] Brown ghost trail during dash
 - [x] Dash direction: keyboard priority, mouse fallback
 - [x] Sprite rolls in dash direction
 - [x] Arrow firing knockback (150.0 force)
 - [x] Camera screenshake on arrow fire (strength 2.5, duration 0.12s)

### Phase 3 — Room System 🔲

 - [x] World scene with RoomContainer and persistent Player
 - [x] Room.gd base script shared by all room scenes
 - [x] RoomData resource injected at runtime by World.gd
 - [x] 11 rooms total: Hub, 6 Normal, 3 Witch (shared scene), 1 Boss
 - [x] Linear room sequence with single exit portal per room
 - [x] Portal opens only after all enemies cleared (NORMAL/BOSS)
 - [x] Portal opens immediately (HUB/WITCH)
 - [x] Portal physics frame delay prevents instant trigger on player spawn
 - [x] Enemy container tracks tree_exited per enemy
 - [x] Curated enemy pool per normal room (easier early, harder late)
 - [x] Random 7–10 enemies per normal room
 - [x] Enemy spawn delay (2.0s) with spawn_preview() particle effect
 - [x] Spawn positions from Polygon2D drawn manually per room
 - [x] Enemy pop-in scale tween on spawn (TRANS_BACK)
 - [x] Enemy HP scaling per cycle (+2 max_hp per cycle)
 - [x] Player HP and gear persist across rooms and cycles
 - [x] Boss loop resets room sequence, increments cycle
 - [x] Game over placeholder (pause + print, World.reset() ready)

### Phase 4 — Gear System 🔲

- [ ] Gear pickup base class
- [ ] Triple shot (3 arrows per fire)
- [ ] Witch room NPC offers gear choice
- [ ] Additional gears TBD

### Phase 5 — Boss 🔲

- [ ] 5 bosses planned (designs TBD)
- [ ] Boss room type

---

## Asset Notes

- No walking animations required — wobble handles the feel
- Two animations per enemy: walk, attack
- Six animations for player: idle_hp1/2/3, attack_hp1/2/3
- Three animations for bow: default, pulled, ready
- Two animations for portal: open, closed
- Keep sprites simple and readable
