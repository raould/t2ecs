# **Design Document: Reconcile‑Based OO Model for an ECS‑Ready Runtime**

## **1. Overview**

This document describes a **reconcile‑based, non‑inheritance OO programming model** designed for a future ECS (Entity Component System) runtime, while remaining simple, ergonomic, and familiar for developers in the MVP phase.

The model provides:

- **Plain OO objects** with fields and methods  
- **Per‑system lifecycle stages** (phys, move, collide, stats)  
- **Ephemeral object semantics** via hydration → mutate → diff → commit  
- **A single reusable instance per entity** in JS  
- **Debug‑mode dual‑instance validation** to guarantee correctness  
- **A declarative query API** for world access  
- **A clean path to ECS extraction** without requiring ECS now  

This architecture is inspired by proven systems such as:

- iOS Core Animation’s model/presentation layers  
- React’s virtual DOM + reconciliation  
- JS animation loops  
- Unity DOTS hybrid components  
- Flecs entity views  

The result is a model that *feels* like OO but *behaves* like ECS.

---

# **2. Motivation and Context**

### **2.1 Why OO?**

Roblox, Unity, Godot, and countless game engines demonstrate that:

- Developers love **objects**, **methods**, and **encapsulation**.  
- OO is easy to teach, easy to reason about, and easy to prototype with.  
- Most gameplay logic is naturally expressed as “this object does X.”

OO is the right ergonomic surface for an MVP.

### **2.2 Why ECS later?**

ECS excels at:

- Data‑oriented performance  
- Parallel scheduling  
- Deterministic simulation  
- Clear system boundaries  
- Explicit read/write sets  
- Incremental updates  
- Event sourcing and replay  

But ECS is *not* the right ergonomic surface for an MVP.

### **2.3 The key insight**

> **If the OO layer is structured correctly, it can be lowered into ECS later without rewriting gameplay code.**

The reconcile‑based OO model is that structure.

---

# **3. Architectural Principles**

1. **World is authoritative**  
   Objects are ephemeral projections of world state.

2. **Per‑system hydration**  
   Each system stage sees a fresh snapshot of the world.

3. **Ephemeral object mutation**  
   Objects mutate themselves freely during a stage.

4. **Diff‑and‑commit reconciliation**  
   Only changed fields are written back to the world.

5. **Single reusable instance (JS)**  
   For performance, one instance per entity is reused across stages.

6. **Debug dual‑instance validation**  
   Debug mode runs both a recycled and fresh instance and diffs them.

7. **Composition over inheritance**  
   Behavior is attached via a behavior struct, not class inheritance.

8. **Structured lifecycle stages**  
   phys → move → collide → stats (extensible).

9. **Declarative query API**  
   Queries are storage‑agnostic and ECS‑friendly.

10. **ECS‑ready semantics**  
    The model preserves system boundaries, read/write sets, and ordering.

---

# **4. Entity Model**

### **4.1 Entity identity**

Each entity has:

- `EntityId`
- A set of components stored in the world
- A behavior struct defining its logic

### **4.2 Ephemeral object view**

Each entity type has a corresponding **ephemeral object class**:

```ts
class Enemy {
  id: EntityId
  behavior: EnemyBehavior

  // hydrated component fields
  position: Vec2
  velocity: Vec2
  health: number

  // transient fields
  cooldown: number
  _scratch: any

  reset(id, frameState) { ... }
}
```

This object is **not** the authoritative state.  
It is a **per‑system projection** of world state.

### **4.3 Behavior struct**

Behavior is attached via composition:

```ts
struct EnemyBehavior {
  phys(dt, ctx) {}
  move(dt, ctx) {}
  collide(dt, ctx) {}
  stats(dt, ctx) {}
  tick(dt, ctx) {} // optional god‑method
}
```

---

# **5. Frame Execution Model**

## **5.1 Lifecycle stages**

The engine defines a fixed pipeline:

```
Phys → Move → Collide → Stats
```

Each stage is a future ECS system.

## **5.2 Per‑system hydration**

Before each stage:

```ts
enemy.reset(id, worldSnapshotAtThisStage)
```

This ensures:

- No stale state leaks across systems  
- Each system sees the correct world snapshot  
- ECS semantics are preserved  

## **5.3 Running behavior**

```ts
behavior[stage](dt, ctx.with(enemy))
```

The object mutates itself freely.

## **5.4 Diff‑and‑commit**

After the stage:

```ts
reconcile(world, enemy)
```

Only changed fields are written back.

---

# **6. Reset() Contract**

`reset(id, frameState)` must:

- Overwrite all component‑backed fields  
- Clear all transient fields  
- Rebind behavior  
- Rebind context if needed  
- Leave no stale state from previous stages  

This is the most important correctness boundary.

---

# **7. Debug Mode: Dual‑Instance Validation**

In debug mode, the engine performs:

### **1. Recycled instance path**

```ts
recycled.reset(id, snapshot)
behavior[stage](dt, ctx.with(recycled))
```

### **2. Fresh instance path**

```ts
fresh = new Enemy()
fresh.reset(id, snapshot)
behavior[stage](dt, ctx.with(fresh))
```

### **3. Diff the two**

```ts
if (!deepEqual(recycled, fresh)) {
  throw new Error("Reset leak detected")
}
```

### **4. Commit using either instance**

They must be identical.

This guarantees:

- reset() correctness  
- no stale state  
- no hidden dependencies  
- ECS‑compatible semantics  

---

# **8. Query API**

Queries are declarative and storage‑agnostic:

```ts
ctx.query(Position).withinRadius(self.position, 5)
ctx.query(StatusEffect).on(self.id)
ctx.query(Collider).intersecting(self.bounds)
```

Queries return iterables or lists.

This API is:

- OO‑friendly  
- ECS‑friendly  
- future‑proof  

---

# **9. Reconciliation**

After each stage:

```ts
if (enemy.position !== initial.position)
  world.write(Position, id, enemy.position)

if (enemy.health !== initial.health)
  world.write(Health, id, enemy.health)
```

This produces:

- explicit write sets  
- explicit diffs  
- explicit system boundaries  

Perfect for ECS extraction.

---

# **10. ECS Lowering Path**

This OO model maps cleanly to ECS:

### **10.1 System extraction**

Each lifecycle method → one ECS system.

### **10.2 Read/write sets**

Extracted from:

- hydration reads  
- diff writes  
- queries  

### **10.3 Scheduling**

Systems can be ordered or parallelized based on read/write sets.

### **10.4 Storage backend swap**

The world can be replaced with:

- ECS archetype storage  
- persistent world  
- event‑sourced world  

### **10.5 Deterministic replay**

Diffs become event logs.

---

# **11. Summary**

This reconcile‑based OO model provides:

- **OO ergonomics**  
- **ECS semantics**  
- **Debug‑mode correctness guarantees**  
- **A clean, minimal MVP**  
- **A direct path to ECS, FP reducers, event sourcing, and time‑travel debugging**  

It is the simplest architecture that:

- feels like OO,  
- behaves like ECS,  
- and evolves into a high‑performance engine without rewriting gameplay code.

If you want, I can now generate:

- a reference implementation sketch,  
- a sample entity + behavior,  
- a world implementation,  
- or a diagram of the full reconcile pipeline.
  