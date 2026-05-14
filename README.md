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
│   └── enemies/
│       ├── Zombie.gd
│       ├── Bomb.gd
│       ├── Slime.gd
│       └── Ghost.gd
        ├── Zombie.tscn
│       ├── Bomb.tscn
│       ├── Slime.tscn
│       └── Ghost.tscn
└── assets/
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

| Export | Default | Moving value |
|--------|---------|--------------|
| `speed` | 7.0 | 7.0 |
| `amount` | 0.03 | 0.03 |
| `x_sway` | 0.2 | 0.2 |
| `idle_speed` | 2.0 | — |
| `idle_amount` | 0.01 | — |
| `idle_x_sway` | 0.05 | — |

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
| `ready` | held long enough |

### Bow Flow

Hold mouse → `pulled` → `ready` → release → spawn Arrow → `default`

### Notes

- No sprite flipping
- Bow rotates to follow mouse via `look_at()`
- `set_attacking(bool)` called by Bow to switch player animation set
- Death uses `call_deferred` to avoid physics callback crash

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

All enemies use `Hurtbox` (Area2D, mask 3) to detect arrows via `area_entered`. The arrow destroys itself on hitting any area named `"Hurtbox"`. Enemies do **not** rely on `body_entered` from the arrow.

---

### 1) Zombie ✅

- Chases player continuously
- Contact damage: **1**
- No special death behavior

**Stats:** `speed=60`, `contact_damage=1`, `max_hp=2`

---

### 2) Bomb ✅

- Chases player
- Within `trigger_distance` → stops, plays `attack`, starts fuse timer
- After `fuse_time` → explodes in radius, damages player + all enemies
- Also explodes on death (arrow kill)
- Destroys itself after explosion

**Stats:** `speed=55`, `max_hp=2`  
**Exports:** `explosion_radius=80`, `explosion_damage=2`, `fuse_time=1.5`, `trigger_distance=50`

---

### 3) Slime ✅

- Chases player
- Plays 'attack' and shoots slime attack on 4 directions (N,E,S,W) so he moves a little, shoots, then moves again, that's his behavior
- On death: spawns **3 smaller slimes** at scaled-down size, spread in a circle
- Child slimes (`is_child=true`) do not spawn further slimes on death

**Stats:** `speed=45`, `contact_damage=1`, `max_hp=2`  
**Exports:** `slime_scene` (assign `Slime.tscn`), `child_scale=0.55`, `child_count=3`  
**Note:** spawn and `queue_free` use `call_deferred` to avoid physics flush crash

---

### 4) Ghost ✅

- Starts invisible (`modulate.a = 0`)
- After `invisible_duration` → teleports near player → becomes visible
- Plays `attack`, shoots player **3 times** with `shoot_interval` delay between shots
- After shooting → short wait → goes invisible again, repeats
- **Can only be damaged while visible** (`modulate.a >= 0.5`)

**Stats:** `speed=0`, `max_hp=3`  
**Exports:** `arrow_scene`, `arrow_speed=200`, `shoot_count=3`, `shoot_interval=0.4`, `invisible_duration=3.0`, `teleport_radius=120`

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
- Auto-destroys after `lifetime` seconds

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

- [x] Bomb (fuse + explosion radius)
- [x] Slime (split on death)
- [x] Ghost (invisible + teleport + shoot)
- [x] Hurtbox system for arrow detection
- [x] Collision layer/mask documentation

### Phase 3 — Room System 🔲

- [ ] Single test room with walls and portal
- [ ] Portal closes on player entry
- [ ] Portal opens when all enemies are dead
- [ ] Enemy container node — room tracks `tree_exited` per enemy (not sure)
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
