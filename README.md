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
│   ├── Player.gd
│   ├── Bow.gd
│   ├── Arrow.gd
│   ├── Player.tscn
│   ├── Arrow.tscn
│   └── globals/
│       ├── Effects.gd              (autoload)
│   └── enemies/
│       ├── Zombie.gd
│       ├── Bomb.gd
│       ├── Slime.gd
        ├── Zombie.tscn
│       ├── Bomb.tscn
│       ├── Slime.tscn
│       └── Ghost.tscn
│       └── Ghost.gd
├── assets/
│   └── shaders/
│       └── hit_flash.gdshader
```

---

## Collision Layers

| Layer | Name | Used By |
|-------|------|---------|
| 1 | player | Player body |
| 2 | enemies | All enemy bodies, ContactArea, Hurtbox |
| 3 | player_arrows | Arrow (Area2D) |
| 4 | enemy_projectiles | Ghost arrow, future projectiles |

| Node | Layer | Mask |
|------|-------|------|
| Player | 1 | 2, 4 |
| Enemy CharacterBody2D | 2 | 1, 3 |
| ContactArea (on enemies) | 2 | 1 |
| Hurtbox (on enemies) | 2 | 3 |
| Arrow | 3 | 2 |
| Enemy projectiles | 4 | 1 |

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

**File:** `scripts/Effects.gd`
**Register as:** Autoload named `Effects` in Project Settings

All effects are spawned purely in code using `CPUParticles2D` — no nodes need to be added to any scene.

#### Color Constants

| Constant | Color | Used For |
|----------|-------|---------|
| `BROWN` | Brown | Player arrow impact |
| `DARK_GREEN` | Dark green | Zombie |
| `GREEN` | Green | Slime |
| `BLACK_SMOKE` | Near black | Bomb |
| `DIRTY_WHITE` | Off-white | Ghost |
| `ORANGE` | Orange | Bomb fire |
| `YELLOW` | Yellow | Bomb flash |
| `RED` | Red | Bomb sparks |

#### Public API

| Method | Description |
|--------|-------------|
| `arrow_impact(pos)` | Brown splinter burst, fired on arrow hit or expiry |
| `hit_flash(sprite)` | Flashes only non-transparent pixels white via shader |
| `contact_damage(pos, color)` | Wide burst in enemy color on contact hit |
| `projectile_trail(pos, color)` | Small puff trail for enemy projectiles |
| `explosion(pos)` | 4-layer bomb explosion: smoke, fire, flash, sparks |
| `zombie_death(pos)` | Dark green burst |
| `slime_death(pos)` | Green burst + downward drip |
| `ghost_death(pos)` | Dirty white burst + upward wisp float |
| `death_bounce(enemy, on_done)` | Knock up + fall off screen + spin, random left/right direction |

#### Hit Flash Shader

**File:** `assets/shaders/hit_flash.gdshader`
Only pixels with `alpha > 0.01` are flashed white — transparent background is never affected. Shader is created and assigned in code by `hit_flash()`, no manual setup needed.

#### Death Bounce

Called from all enemies except Slime. Behavior:
- Flips sprite upside down instantly (negative `scale.y`)
- Knocks enemy upward `-120px` with random left/right drift
- Falls `800px` off screen with spin in knock direction
- `queue_free` fires after 3 second buffer to ensure it's off screen

Slime skips `death_bounce` — it spawns children and frees immediately.

---

## Player

### Scene Tree

```
Player (CharacterBody2D) — Player.gd       group: "player"
├── StatsComponent                          max_hp=3, speed=100
├── AnimatedSprite2D
│   └── WobbleComponent
├── CollisionShape2D
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

### Notes

- No sprite flipping
- Bow rotates to follow mouse via `look_at()`
- `set_attacking(bool)` called by Bow to switch player animation between `idle_hpX` and `attack_hpX`
- Death uses `call_deferred` to avoid physics callback crash
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

### Shared Death Behavior (except Slime)

```gdscript
set_physics_process(false)
contact_area.monitoring = false
hurtbox.monitoring = false
Effects.death_bounce(self, queue_free)
```

---

### 1) Zombie ✅

- Chases player continuously
- On contact: plays `attack` animation briefly, deals contact damage
- Death: dark green burst + death bounce

**Stats:** `speed=60`, `contact_damage=1`, `max_hp=2`

---

### 2) Bomb ✅

- Chases player, plays `walk`
- Within `trigger_distance` → stops, plays `attack`, wobble disabled, starts shrinking slowly over fuse duration
- After `fuse_time` → pops to 2.2x scale briefly → explodes in radius → damages player + all enemies in radius
- Also triggers fuse on death by arrow (does not explode instantly — fuse runs first)
- Explosion: 4-layer particle effect (smoke, fire, flash, sparks)
- No death bounce — explosion is its exit

**Stats:** `speed=55`, `max_hp=2`
**Exports:** `explosion_radius=80`, `explosion_damage=2`, `fuse_time=1.5`, `trigger_distance=100`
**Scale:** `BASE_SCALE = Vector2(0.17, 0.17)`

---

### 3) Slime ✅

- Chases player
- Contact damage: **1**
- On death: spawns **3 smaller slimes** at scaled-down size, spread in a circle
- Child slimes (`is_child=true`) do not spawn further slimes on death
- No death bounce — spawns children then frees immediately via `call_deferred`

**Stats:** `speed=45`, `contact_damage=1`, `max_hp=2`
**Exports:** `slime_scene` (assign `Slime.tscn`), `child_scale=0.55`, `child_count=3`

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
- Squish on each shot: shrinks to 0.6x → pops to 1.35x → returns to base, arrow fires at pop moment
- **Can only be damaged while visible** (`modulate.a >= 0.5`)
- Death: dirty white burst + upward wisp + death bounce

**Stats:** `speed=0`, `max_hp=3`
**Exports:** `arrow_scene`, `arrow_speed=200`, `shoot_count=3`, `shoot_interval=0.4`, `invisible_duration=3.0`, `teleport_radius=120`, `walk_before_attack=0.3`, `windup_before_shoot=0.3`
**Scale:** `BASE_SCALE = Vector2(0.14, 0.14)`

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

---

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
- [x] Hurtbox system for arrow detection
- [x] Collision layer/mask documentation

### Phase 2.5 — Effects ✅

- [x] Effects autoload (CPUParticles2D, no scene nodes needed)
- [x] Arrow impact particles (brown)
- [x] Contact damage particles (per enemy color)
- [x] Projectile trail (enemy arrows only)
- [x] Hit flash shader (non-transparent pixels only)
- [x] Death particles per enemy (zombie, slime, ghost)
- [x] Bomb explosion (4-layer: smoke, fire, flash, sparks)
- [x] Death bounce animation (knock up, random direction, spin, fall off screen)
- [x] Bomb fuse shrink + pop scale tween

### Phase 3 — Room System 🔲

- [ ] Single test room with walls and portal
- [ ] Portal closes on player entry
- [ ] Portal opens when all enemies are dead
- [ ] Enemy container node — room tracks `tree_exited` per enemy
- [ ] 7 rooms total, each full screen
- [ ] Room types: normal, boss, witch (NPC)
- [ ] Center hub room with 4 exits (portal)
- [ ] Random room type assignment

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
- Two animations per enemy: `walk`, `attack`
- Six animations for player: `idle_hp1/2/3`, `attack_hp1/2/3`
- Three animations for bow: `default`, `pulled`, `ready`
- Keep sprites simple and readable
