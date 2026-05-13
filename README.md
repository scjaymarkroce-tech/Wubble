# Simple Roguelike (Godot 4.4.1) — Project Guide

## Goal
Build a **simple top‑down roguelike** with reusable components and minimal art/animation (no sprite sheets). The focus is gameplay systems, not assets.

---

## Tech Stack
- **Engine:** Godot 4.4.1
- **Style:** 2D top‑down
- **Animation:** procedural wobble (sine scale/rotation)
- **Input Actions:** `move_up`, `move_down`, `move_left`, `move_right`, `attack`

---

## ✅ Progress So Far

### ✅ Wobble Animation Component
**Attach to:** `Sprite2D` or `AnimatedSprite2D`  
**Behavior:** sine scale + rotation, supports idle wobble + moving wobble.

Key settings:
- `speed`
- `amount`
- `x_sway`
- `rotation_amount`
- `only_when_moving`
- `idle_scale_multiplier`
- `idle_rotation_amount`
- `velocity_provider` (set to `../` if wobble is on Sprite2D)

---

### ✅ Player Movement (Input Map)
Movement now uses the new input actions:
- `move_up`
- `move_down`
- `move_left`
- `move_right`

Player script uses:
```gdscript
Input.get_action_strength("move_right") - Input.get_action_strength("move_left")
Input.get_action_strength("move_down") - Input.get_action_strength("move_up")
```

---

### ✅ Player Visuals
Player uses **AnimatedSprite2D** with HP‑based animations:
- `idle_hp1`, `attack_hp1`
- `idle_hp2`, `attack_hp2`
- `idle_hp3`, `attack_hp3`

---

## Core Systems (Reusable)

### 1) Wobble Animation Component
**Purpose:** Fake idle/walk animation via sine scaling & rotation.  
**Attach to:** any `Sprite2D` or `AnimatedSprite2D`.

---

### 2) Stats Component (Planned)
Reusable stats for players/enemies.

**Stats:**
- `max_hp`
- `hp`
- `speed`
- `damage`
- `attack_speed`
- `contact_damage`

**Features:**
- `take_damage(amount)`
- `on_death` signal
- clamps & limits

---

## Player Design

### Movement
- Top‑down WASD/Analog
- Uses wobble component for idle/walk

### Bow (Next Step)
Bow is a **separate Sprite2D** with 3 images: use animated sprite 2d and you give name to the animation
1. **Default** (aims at mouse)
2. **Pulled** (holding)
3. **Ready** (arrow visible on bow)

**Flow:**
- Hold mouse → pull → ready  
- Release → spawn arrow → return to default

---

## Enemies (Current Plan)

### 1) Zombie
- Chases player
- Contact damage: **1**

### 2) Bomb
- Chases player  
- If close: **wait → explode**
- Explosion damages player + enemies
- Explodes on death  
- Dies after explosion

### 3) Slime
- Contact damage: **1**
- On death: spawn **3 smaller slimes** (scaled down)

### 4) Ghost
- Invisible → teleport → visible
- Shoots at player **3 times**
- Invisible for **3 seconds** then repeats

---

## Map / Room Plan

### Rooms
- Total: **7 rooms**
- Each room = full screen
- Randomized room types

### Room Types
1. Normal enemy room (1–4 enemies)
2. Boss room (future, 5 bosses planned)
3. Witch room (NPC with gear)

### Rules
- Room doors **close on entry**
- Doors **open only after all enemies are killed**
- Player starts in **center hub** with 4 exits
- Some rooms lead to dead ends, but all rooms must be cleared

---

## Gear (Start Simple)

### Planned
1. **Triple Shot** — shoot 3 arrows instead of 1
2. More coming later

---

## Development Roadmap

### Phase 1 — Core Loop
- ✅ Movement + wobble
- ⏳ Bow shooting + arrow
- ⏳ 1 enemy (zombie)
- ⏳ 1 room with door lock/unlock

### Phase 2 — Expand Enemies
- Bomb
- Slime
- Ghost

### Phase 3 — Room System
- 7 rooms
- Random room types
- Boss + witch rooms

### Phase 4 — Gear System
- Triple shot
- Additional gears

---

## Next Step
Start building **bow shooting + arrow**.

Move → Shoot → Kill → Door unlock
