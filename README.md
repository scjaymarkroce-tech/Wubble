# Simple Roguelike (Godot 4.4.1) — Project Guide

## Goal
Build a **simple top‑down roguelike** with reusable components and minimal art/animation (no sprite sheets). The focus is gameplay systems, not assets.

---

## Tech Stack
- **Engine:** Godot 4.4.1
- **Style:** 2D top‑down
- **Animation:** procedural wobble (sine scale/rotation)

---

## Core Systems (Reusable)

### 1) Wobble Animation Component
**Purpose:** Fake idle/walk animation via sine scaling & rotation.  
**Attach to:** any `Sprite2D` or `AnimatedSprite2D`.

**Settings:**
- `speed` (float)
- `amount` (float)
- `x_sway` (float)
- `only_when_moving` (bool)

---

### 2) Stats Component
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

### 3) Damage Visuals (Player)
Player has **3 HP**, each HP has a unique sprite.

| HP | Sprite |
|----|--------|
| 3  | Normal |
| 2  | Damaged 1 |
| 1  | Damaged 2 |

---

## Player Design

### Movement
- Top‑down WASD/Analog
- Uses wobble component for idle/walk

### Bow
Bow is a **separate Sprite2D** with 3 images:
1. **Default** (aims at mouse)
2. **Pulled** (holding)
3. **Ready** (arrow visible on bow)

**Flow:**
- Hold mouse → pull → ready  
- Release → spawn arrow → return to default

---

## Enemies (Current)

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
- Movement + wobble
- Bow shooting + arrow
- 1 enemy (zombie)
- 1 room with door lock/unlock

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

## Asset Notes
- No walking animations required
- Use procedural wobble for life
- Keep assets simple and readable

---

## Next Step
Start by building **one test room** and the **core loop**:
> Move → Shoot → Kill → Door unlock
