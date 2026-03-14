**Game Engine Architecture**

Revised Design Document v5

*OO-style framework · ECS internals · t2lang implementation*

# **1. Motivation and Goals**

This engine provides an OO-style framework for game developers, backed by ECS internals. Developers write classes and methods in a familiar style. The ECS machinery — component arrays, system scheduling, spatial subdivision — operates underneath and is not directly exposed. The OO framework is intentionally constrained by ECS needs, so developers will feel those constraints. The goal is not to hide ECS perfectly, but to make it approachable.

Implementation examples throughout this document are written in **t2lang** — a Lisp-style language with s-expression syntax that compiles to TypeScript/JavaScript. The parenthesized syntax is intentional. Developers coming from TypeScript or JavaScript will find the semantics familiar even if the surface syntax differs. Some sections show TypeScript directly where the type system concepts are clearer in that form.

### **Goals**

* Game developers write OO-style classes and methods. Direct ECS knowledge is not required but the constraints will be felt.

* All world state access is explicit and goes through the world API.

* Frame semantics are correct by construction — prev-frame is immutable, next-frame accumulates writes.

* Global systems handle cross-entity concerns that OO cannot express cleanly, such as collision detection.

* The engine internals can evolve without changing game code, because the world API is the stable contract.

* **Implementation language:** t2lang, targeting TypeScript/JavaScript.

### **Non-goals for MVP**

* No parallel scheduling.

* No incremental query recomputation.

* No event sourcing persistence or time-travel debugging.

* No generic query engine or optimizer — world structure is hardcoded and navigated explicitly.

* No automatic registration from interface declarations — explicit registration in constructors for MVP.

# **2. Type Vocabulary**

Short, consistent names are used throughout. Abbreviated names are preferred where the meaning is unambiguous:

## Entities:

- Entites are akin to Objects.
- Entities have Components, which are akin to Object fields.

Entity       : base class for all game objects  
Eid          : generational entity identifier  

## Components:
Pos          : position (type alias over V2)  
Vel          : velocity (type alias over V2)  
Rot          : rotation  
Acc          : acceleration (type alias over V2)  
Dmg          : damage value  
Hp           : hit points / health  
Rect         : axis-aligned bounding rectangle  

## Other
V2           : 2D vector — the fundamental math type  
Ctx          : dependency injection context  
World        : world frame (prev or next)

## **2.1 V2 — the math type**

All spatial quantities are V2 instances. V2 provides a full complement of methods — no operator overloading, no verbose component arithmetic at call sites:

(class V2  
  (class-body  
    (constructor ((private x: f32) (private y: f32)))  
    (method add   ((other: V2): V2)  (V2 (+ this.x other.x) (+ this.y other.y)))  
    (method sub   ((other: V2): V2)  (V2 (- this.x other.x) (- this.y other.y)))  
    (method scale ((s: f32): V2)     (V2 (* this.x s) (* this.y s)))  
    (method dot   ((other: V2): f32) (+ (* this.x other.x) (* this.y other.y)))  
    (method len   ((): f32)          (Math.sqrt (+ (* this.x this.x) (* this.y this.y))))  
    (method len2  ((): f32)          (+ (* this.x this.x) (* this.y this.y)))  
    (method norm  ((): V2)           (this.scale (/ 1.0 (this.len))))  
    (method perp  ((): V2)           (V2 (- this.y) this.x))  
    (method clip  ((min: V2) (max: V2): V2)  
      (V2 (Math.max min.x (Math.min max.x this.x))  
          (Math.max min.y (Math.min max.y this.y))))))

(type Pos V2)  
(type Vel V2)  
(type Acc V2)

# **3. The OO-Style Framework**

This is what game developers write. The sections below explain the conventions and constraints.

## **3.1 A complete example**

ArmedFighter creates and owns its Shield — this is correct encapsulation. The caller that creates an ArmedFighter does not need to know anything about shields:

(class ArmedFighter  
  (implements Physical Damageable Renderable)  
  (class-body  
    (field (shield_id: Eid))

    (constructor ((private ctx: Ctx)  
                  (private prevPos: Pos)  
                  (private prevVel: Vel))  
      ; ArmedFighter creates and owns its Shield  
      (let* ((shield: Shield (new Shield ctx this.entity_id)))  
        (set! this.shield_id shield.entity_id))  
      ; explicit registration  
      (ctx.renderManager.register this))

    ; physics stage — base class getters handle component lookup  
    (method onPhysics ((prev: World) (next: World) (dt: f32))  
      (next.setValue Pos this.entity_id  
        (this.prevPos.add (this.prevVel.scale dt))))

    ; collision reaction — engine calls this after collision detection  
    (method onCollide ((prev: World) (next: World) (other: Eid))  
      (if-let* ((dmg: Dmg (prev.get Dmg other)))  
        (this.addHpDelta next (- dmg.value))))

    ; render stage  
    (method onRender ((prev: World) (next: World))  
      (next.renderQueue.add  
        (Sprite this.sprite_id this.prevPos this.z_order)))))

; caller — no knowledge of Shield required:  
(next.add (new ArmedFighter ctx spawnPos initialVel))

## **3.2 Shield — self-contained, self-registering**

(class Shield  
  (implements Collidable Renderable)  
  (class-body  
    (field (owner_id: Eid))  
    (field (prevPos:  Pos))  
    (field (bounds:   Rect))  
    (field (shape:    Circle))  
    (field (layer:    CollisionLayer))  
    (field (mask:     CollisionLayer))

    (constructor ((private ctx: Ctx) (owner_id: Eid))  
      (set! this.owner_id owner_id)  
      (ctx.collisionManager.register this)  
      (ctx.renderManager.register this))

    (method onCollide ((prev: World) (next: World) (other: Eid))  
      (if-let* ((dmg: Dmg (prev.get Dmg other)))  
        ; damage flows to owner  
        (next.events Hp this.owner_id add  
          (HpDelta (- dmg.value)))))

    (method onRender ((prev: World) (next: World))  
      (next.renderQueue.add  
        (Sprite this.sprite_id this.prevPos this.z_order)))))

*Explicit registration is required in MVP. Automatic registration from implements declarations is a future t2lang compiler feature — TypeScript's runtime type erasure makes it impossible without compiler support, and multiple extends is not supported for carrying runtime kind fields across multiple interfaces.*

## **3.3 The prev and next convention**

**prev** and **next** are the canonical names for the two world frame arguments on every pipeline method.

* **prev** — fully immutable. Committed state from end of last frame. Read from this for a stable baseline.

* **next** — accumulates writes during the current frame. Readable by later pipeline stages. Always written to. Becomes prev after commit.

Component variable names are prefixed with their frame — prevPos, nextPos, prevHp — so the frame source is unambiguous at every use site:

(let* ((prevPos: Pos (prev.get Pos this.entity_id))  
       (nextPos: Pos (next.get Pos this.entity_id)))  
  ...)

## **3.4 The _reset() object model and single template instance**

OO instances are long-lived but stateless between pipeline stage invocations. The engine maintains exactly one template instance per class. Before invoking any pipeline method on an entity, the engine calls _reset() to rebind the instance to that entity:

(method _reset ((entityId: Eid) (p: World) (n: World))  
  (set! this.entity_id entityId)  
  (set! this.prev p)  
  (set! this.next n))

; engine loop per stage — one template instance per class:  
; for each entity_id in collection:  
;   template._reset(entity_id, prev, next)  
;   template.onPhysics(prev, next, dt)

This is valid because the engine is single-threaded and pipeline stages run sequentially. All persistent per-entity state lives in the component arrays — the instance carries no state between resets.

*Do not hold references to entity instances across frame boundaries or pipeline stages. An instance reference is only valid for the duration of the current _reset() binding. Stale instance references have caused real bugs in practice. Generational EntityIds catch stale cross-entity references automatically; stale instance references are the developer's responsibility to avoid.*

## **3.5 Base class getters — no boilerplate component reads**

The Entity base class defines getters for every known component type. All use this.entity_id implicitly:

(class Entity  
  (class-body  
    (get prevPos ((): Pos)  (this.prev.get Pos this.entity_id))  
    (get prevVel ((): Vel)  (this.prev.get Vel this.entity_id))  
    (get prevHp  ((): f32)  (this.prev.get Hp  this.entity_id))  
    (get prevDmg ((): f32)  (this.prev.get Dmg this.entity_id))  
    (get nextPos ((): Pos)  (this.next.get Pos this.entity_id))  
    (get nextVel ((): Vel)  (this.next.get Vel this.entity_id))  
    (get nextHp  ((): f32)  (this.next.get Hp  this.entity_id))  
    ; ... one pair per component type

    (method addHpDelta ((next: World) (delta: f32))  
      (next.events Hp this.entity_id add (HpDelta delta)))  
    (method setHpAbsolute ((next: World) (value: f32))  
      (next.events Hp this.entity_id add (HpSet value)))))

In debug mode, getters assert that the component exists. A null result on an own component is an engine lifecycle bug, not a game code concern. If a subclass calls a getter for a component type it does not have registered, the assert fires with a clear message.

## **3.6 Cross-entity access — components only, never instances**

You never fetch another entity's OO instance. Cross-entity access always goes through component lookups. No such API exists in MVP:

; NEVER — no such API exists in MVP:  
; (world.getInstance Fighter other_id)

; ALWAYS — fetch their components directly:  
(prev.get Pos other_id)    ; Pos?  
(prev.get Hp  other_id)    ; f32?

This constraint exists for two reasons: it enforces the ECS model where data lives in component arrays, and it prevents stale instance reference bugs. Fetching components instead of instances means the generational Eid check handles validity automatically.

# **4. Desugaring — What the Framework Code Means**

## **4.1 Component read — via base class getter**

; getter call:  
this.prevPos

; expands to:  
(this.prev.get Pos this.entity_id)

; which desugars to:  
; 1. validate generational ID  
; 2. dense_index = Pos_sparse[entity_id.index]  
; 3. return Pos_dense[dense_index]   ; asserted non-null for own components

## **4.2 Component write**

(next.setValue Pos this.entity_id newPos)

; desugars to:  
; 1. validate generational ID  
; 2. dense_index = Pos_sparse[entity_id.index]  
; 3. Pos_dense[dense_index] = newPos

## **4.3 Aggregate write**

(this.addHpDelta next -5)

; expands to:  
(next.events Hp this.entity_id add (HpDelta -5))

; desugars to:  
; 1. append HpDelta(-5) to Hp event log for this entity  
; 2. immediately: Hp_running_total[entity] += -5

; running total readable immediately by later pipeline stages:  
(next.get Hp this.entity_id)   ; returns running total so far

## **4.4 Collision flow**

; 1. collision global system produces symmetric Reports:  
;    Reports = Map<src: Eid, dsts: Set<Eid>>  
;    each pair filtered by layer/mask visibility  
;    each colliding pair appears twice — once as src, once as dst

; 2. engine bridges Reports to OO instances:  
;    for each (src, dsts) in Reports:  
;      template._reset(src, prev, next)  
;      for each dst in dsts:  
;        template.onCollide(prev, next, dst)

; 3. onCollide() reacts:  
(method onCollide ((prev: World) (next: World) (other_id: Eid))  
  (if-let* ((dmg: Dmg (prev.get Dmg other_id)))  
    (this.addHpDelta next (- dmg.value))))

*The collision system does not know about Hp, Dmg, or any game-specific type. It only produces Eid pairs that overlap and pass the visibility filter. All game logic consequences are handled in onCollide().*

# **5. Frame Semantics**

## **5.1 Two frames**

* **prev** — fully immutable. Committed state from end of last frame. Discarded after commit.

* **next** — mutable. Accumulates writes from pipeline stages. Readable by later stages. Becomes prev after commit.

The renderer always reads from a fully committed snapshot — never from a partially updated next-frame.

## **5.2 Input snapshot**

Input events arrive between frames and are snapshotted into a fixed buffer at the start of each frame. Stable throughout the frame. Cleared after commit.

## **5.3 Aggregate components**

Components modified by multiple independent systems in the same frame use an event-sourced aggregate model:

; poison:  (this.addHpDelta next -3)  
; sword:   (this.addHpDelta next -10)  
; heal:    (this.addHpDelta next +20)  
; result:  hp + 7   (all contributions preserved)

; SetAbsolute resets the running total:  
; [HpDelta(-3), HpSet(100), HpDelta(-10)] → final: 90

A running total is maintained incrementally. Later stages read it via the normal component read API — no end-of-frame scan needed.

## **5.4 Commit**

* Validators run against next-frame state.

* If no blocking validation events exist, next-frame is atomically committed.

* Components and relationships persist. Event logs are cleared. Aggregate running totals persist; their logs are cleared.

* Input snapshot is cleared. prev is discarded.

# **6. The Update Pipeline**

Each stage in this engine's pipeline corresponds directly to a **system** in ECS terminology. A stage/system declares what it operates on and the engine runs it once per frame in a fixed order. The difference from a traditional ECS system is that our stages dispatch through the OO method model — _reset() then a named method — rather than iterating component arrays directly.

1.  Input snapshot          engine owned  
2.  Buff reduction          global system → writes modifier events to targets  
3.  Effects                 implements Effectable   → onEffect(prev, next, dt)  
4.  Physics / movement      implements Physical     → onPhysics(prev, next, dt)  
5.  Collision detection     global system           → produces CollisionReports  
6.  Collision reaction      engine bridges reports  → onCollide(prev, next, other)  
7.  Input processing        implements InputReceiver → onInput(prev, next, input)  
8.  Spawning                result of input processing  
9.  Damage application      implements Damageable   → onDamage(prev, next)  
10. Animation / render prep implements Renderable   → onRender(prev, next)  
11. Validators              global system  
12. Commit

*Buff reduction runs before physics so that velocity and damage modifiers are already written when physics and damage stages run. A buff created this frame influences next frame's physics — one frame latency, consistent with the pipeline model.*

## **6.1 OO methods vs global systems**

* **Per-entity OO methods** — the entity is the subject. Movement, collision reaction, rendering.

* **Global systems** — the world is the subject. Collision detection, buff reduction, validators. No single entity owns the logic.

*If an entity method needs to query the entire world to do its work, it should be a global system instead. onCollide(other) is correct OO-style. figureOutWhatIOverlap() is not.*

# **7. Storage Model and Performance**

Understanding the storage model is important for setting correct performance expectations. Our choices — generational IDs, sparse sets, no dense array ordering guarantees — have direct consequences for cache behavior.

## **7.1 Generational entity IDs**

struct Eid {  
  index:      u32   ; slot in the component arrays  
  generation: u32   ; incremented each time the slot is reused  
}

When an entity is destroyed, its index returns to a free list and its generation is incremented. Any stored Eid with the old generation returns null from get() automatically. Stored EntityIds are automatically weak references — this is why cross-entity access uses EntityIds and not instance references.

## **7.2 Sparse set storage**

Each component type has its own independent sparse set. There is no ordering guarantee between different component type arrays — the dense index for entity 42 in Pos_dense has no relationship to the dense index for entity 42 in Vel_dense. They were inserted in different orders as entities were created and destroyed over time:

; one independent sparse set per component type:  
Pos_sparse[]    ; entity_id.index → dense_index  (O(1) lookup)  
Pos_dense[]     ; dense_index → Pos value        (compact, unordered)  
Pos_entity[]    ; dense_index → Eid         (reverse mapping)

Vel_sparse[]    ; completely independent from Pos arrays  
Vel_dense[]     ; different ordering, different length  
Vel_entity[]

; looking up both Pos and Vel for one entity:  
const pos_di = Pos_sparse[entity_id.index]   ; random access  
const pos    = Pos_dense[pos_di]              ; random access  
const vel_di = Vel_sparse[entity_id.index]   ; random access  
const vel    = Vel_dense[vel_di]              ; random access  
; all four accesses are potentially cache misses

## **7.3 Cache behavior — honest assessment**

Because the dense arrays have no ordering guarantee relative to each other, there is no way to align two different component arrays for sequential access. Every component lookup — including own component reads via base class getters — goes through sparse array indirection:

; what onPhysics actually does per entity:  
;   _reset(entity_id, prev, next)          ; bind entity  
;   this.prevPos                            ; sparse lookup → random dense access  
;   this.prevVel                            ; sparse lookup → random dense access  
;   next.setValue Pos entity_id newPos ; sparse lookup → random dense write  
; every component access is a potential cache miss

This is effectively array-of-structs performance characteristics dressed in sparse set clothing. The access pattern per entity is similar to plain OO — not terrible for complex entities with rich behavior and large working sets, but not the tight linear scan that makes ECS compelling for simple high-count entities.

## **7.4 Comparison with archetype ECS**

Archetype ECS engines like Bevy and Flecs group entities by their exact component combination. All entities with (Pos, Vel) are stored in one archetype's arrays; all entities with (Pos, Vel, Collidable) in another. Within one archetype, iterating Pos_dense and Vel_dense in parallel is sequential for both because they share the same storage block:

; archetype ECS inner loop — fully sequential:  
for i in 0..archetype.len {  
  pos[i] += vel[i] * dt   ; both arrays accessed sequentially  
}

; our model inner loop — indirect per component:  
for entity_id in collection {  
  template._reset(entity_id, prev, next)  
  template.onPhysics(prev, next, dt)  
  ; inside: sparse lookup for every component access  
}

However, archetype ECS has its own costs and complexities:

* **Component add/remove is expensive.** Adding a component to an entity physically moves all its existing component data from one archetype's arrays to another. This is O(components) per change and happens invisibly to game code.

* **Data movement is hard to reason about.** When an entity moves between archetypes, its dense indices change. The developer has no visibility into which archetype an entity currently lives in.

* **Archetype ECS resembles denormalized OO.** A large archetype that groups many components together is structurally similar to an array of structs — a full object per entity. The further you push archetype sizes, the weaker the ECS cache efficiency argument becomes.

* **Sync hazards.** There is only ever one copy of each component value per entity, but it can move between archetypes at runtime in ways that are difficult to predict or trace during debugging.

## **7.5 Where our model performs well**

The random access cost of our sparse set model is most significant for simple high-count entities — bullets, particles, swarm enemies — where an archetype ECS would give tight linear loops over small structs. For these cases a future optimization can bypass the OO dispatch entirely and run purpose-built tight loops directly over the relevant dense arrays.

For complex entities — ArmedFighter, Shield, Boss — the random access pattern is unavoidable regardless of storage model because the working set per entity is large. The cache efficiency argument for archetype ECS weakens as entity complexity grows. Our model performs comparably to any approach for these cases, with significantly less operational complexity.

*The sparse set model trades cache efficiency on simple high-count entities for simplicity, flexibility, and cheap component add/remove. This is the right tradeoff for an MVP where correctness and developer ergonomics matter more than peak throughput. The evolution path toward tighter storage is clear and does not require changing game code.*

## **7.6 Named typed collections — MVP world structure**

In MVP the world has named typed collections accessed directly. There is no generic query engine:

world.fighters      ; Dict<Eid, Fighter components>  
world.shields       ; Dict<Eid, Shield components>  
world.shots         ; Dict<Eid, Shot components>  
world.collidables   ; Dict<Eid, CollidableComponent>

; fundamental primitive — single component lookup:  
(prev.get Pos entity_id)    ; Pos?  
(prev.get Hp  entity_id)    ; f32?

; iteration over a typed collection:  
(world.forEachFighter (lambda ((id: Eid)) ...))

*MVP storage may begin as array-of-structs for implementation simplicity. The world API is storage-agnostic — the backend can be upgraded to true SoA sparse sets without changing game code.*

# **8. Entity Lifecycle**

## **8.1 Two categories of component access failure**

* **Own components — assert, never null.** An instance and its components have identical lifespans by construction. Reads of own components within a pipeline method are unconditionally safe.

* **Cross-entity references — nullable, handled via if-let*.** The generational ID check inside get() returns null automatically if the referenced entity is gone.

## **8.2 Creation**

; spawning a bullet during input processing stage:  
(next.add (new Bullet ctx spawnPos vel this.entity_id))

(constructor ((private ctx: Ctx)  
              (private prevPos: Pos)  
              (private prevVel: Vel)  
              (owner_id: Eid))  
  (set! this.owner_id owner_id)  
  (ctx.collisionManager.register this)  
  (ctx.renderManager.register this))

## **8.3 Destruction**

(next.events Destroy this.entity_id add (DestroyEvent this.entity_id))

; at commit:  
; - generation incremented at this entity's index  
; - index returned to free list  
; - any stored Eid with old generation is now stale

## **8.4 Lifecycle coupling**

* **Coupled — composition.** ArmedFighter creates and owns its Shield. Fighter destruction triggers shield destruction.

* **Independent — weak reference.** A power-up shield can be lost independently. Generational check returns null when shield is gone.

* **Third-party governed.** A shield granted by a StatusEffect is controlled by a third entity. Better modeled as a component on the fighter.

# **9. Nullable Handling — if-let***

Cross-entity component lookups return nullable values. The if-let* macro provides flat, non-nested nullable handling matching let* binding syntax:

(if-let* ((dmg: Dmg (prev.get Dmg other))  
          (pos: Pos (prev.get Pos other)))  
  (then  
    (doSomething dmg pos)))

; if-let* is a statement, not an expression.  
; silently does nothing if any binding returns null.  
; each binding can reference earlier bindings.  
; execution stops at the first null.

# **10. Interfaces and Registration**

## **10.1 Structural vs nominal interfaces**

* **Structural interfaces** — compile-time shape checking. Used for pipeline stage contracts: Physical, Damageable, Renderable. No runtime identity.

* **Nominal interfaces** — carry runtime identity via a kind field. Used when the engine needs runtime type checks.

(interface Physical  
  (method onPhysics ((prev: World) (next: World) (dt: f32)) void))

(nominal-interface Collidable  
  (prevPos:  Pos)  
  (bounds:   Rect)  
  (shape:    Shape)  
  (layer:    CollisionLayer)   ; permanent type identity  
  (mask:     CollisionLayer))  ; can change — cleared for invincibility frames

## **10.2 Explicit registration**

; MVP — explicit in each constructor:  
(ctx.collisionManager.register this)   ; type system validates Collidable  
(ctx.renderManager.register this)      ; type system validates Renderable

; future t2lang compiler feature:  
; generate register() calls from implements declarations automatically.

# **11. Global Systems**

## **11.1 The collision system**

struct CollidableComponent {  
  entity_id: Eid  
  prevPos:   Pos  
  bounds:    Rect  
  shape:     Shape  
  layer:     CollisionLayer  
  mask:      CollisionLayer  
}

; collision stage:  
; 1. rebuild spatial structure from registered Collidables  
; 2. broad phase: bucket by spatial proximity  
; 3. layer/mask filter: a.layer & b.mask  
; 4. narrow phase: shape overlap test  
; 5. produce symmetric Reports: Map<src: Eid, dsts: Set<Eid>>

; engine bridges Reports to OO instances:  
; for each (src, dsts): template._reset(src) → onCollide(prev, next, dst)

## **11.2 Validators**

(fn checkDeath ((next: World))  
  (next.forEachDamageable (lambda ((id: Eid))  
    (if-let* ((hp: f32 (next.get Hp id)))  
      (if (<= hp 0)  
        (then  
          (next.events Validation id add  
            (DeathEvent id ValidationSeverity.blocking))))))))

* **warning** — log and continue.

* **blocking** — abort commit. Engine can branch or rollback.

* **fatal** — end the session immediately.

# **12. Buffs and the Component Type System**

Buffs are temporary modifiers applied to entity attributes. They reveal an important design pattern: the set of buffable attributes is exactly the set of ECS component array types, and TypeScript's type system can enforce this connection statically.

## **12.1 ComponentType — the closed union**

type ComponentType = Pos | Vel | Hp | Dmg | AttackSpeed

; single source of truth for buffable component types.  
; adding a new type to the union surfaces all places needing updates.

## **12.2 DeltaOf<T> — typed deltas**

type DeltaOf<T extends ComponentType> =  
  T extends Pos         ? V2     :  
  T extends Vel         ? V2     :  
  T extends Hp          ? number :  
  T extends Dmg         ? number :  
  T extends AttackSpeed ? number :  
  never

; Buff<Vel> has delta: V2   — cannot be created with a scalar delta  
; Buff<Hp>  has delta: number — cannot be created with a V2 delta

; the never branch is a compiler exhaustiveness check:  
; adding a new ComponentType without handling it in DeltaOf  
; is a compile error, not a silent runtime bug

## **12.3 Buff<T> — typed buff entity**

interface Buff<T extends ComponentType> {  
  target_id: Eid  
  attribute: T  
  delta:     DeltaOf<T>   ; type enforced by T  
  duration:  number  
}

## **12.4 Aggregate event types use DeltaOf**

class Delta<T extends ComponentType> implements AggregateEvent<T> {  
  constructor(readonly value: DeltaOf<T>) {}  
  apply(current: DeltaOf<T>): DeltaOf<T> { ... }  
}

class SetAbsolute<T extends ComponentType> implements AggregateEvent<T> {  
  constructor(readonly value: DeltaOf<T>) {}  
  apply(current: DeltaOf<T>): DeltaOf<T> { return this.value }  
}

## **12.5 The buff global system**

Runs at stage 2 — before physics — so modifier components are ready when physics and damage stages run. Entity classes never mention buffs — they just read their component values:

(fn buffReductionStage ((prev: World) (next: World) (dt: f32))  
  (prev.forEachBuff (lambda ((id: Eid))  
    (if-let* ((buff (prev.get Buff id)))  
      (then  
        (next.events buff.attribute buff.target_id add (Delta buff.delta))  
        (next.events BuffDuration id add (Delta (- dt)))  
        (if (<= (- buff.duration dt) 0)  
          (next.events Destroy id add (DestroyEvent id))))))))

; ArmedFighter.onPhysics reads this.prevVel and gets effective velocity  
; — base velocity plus any active buff contributions —  
; without knowing buffs exist.

## **12.6 Exhaustiveness guarantee**

* Adding a new type to ComponentType without handling it in DeltaOf is a compile error.

* The buff reducer switch gets an exhaustiveness warning if a new type is unhandled.

* The Entity base class getter list is the one place requiring manual discipline — add a prevX/nextX getter pair for each new component type.

# **13. Evolution Path**

Every step is enabled by all game logic going through the world API. The API is the stable contract; the implementation beneath it can change.

* **Step 1 — True SoA sparse sets:** swap named typed collections for proper struct-of-arrays with sparse set indexing. World API unchanged.

* **Step 2 — Tight loops for simple entities:** for high-count simple entities (bullets, particles), bypass OO dispatch and run purpose-built dense array loops. OO dispatch remains for complex entities.

* **Step 3 — Automatic registration:** t2lang compiler generates register() calls from implements declarations.

* **Step 4 — Scheduler:** infer read/write sets from pipeline method signatures. Schedule non-conflicting stages in parallel.

* **Step 5 — Event sourcing:** persist the event log across frames for replay and time-travel debugging.

* **Step 6 — Generic query engine:** replace hardcoded named collections with a query planner. Same world API surface.

# **14. Open Questions**

* **Query engine design:** requires profiling real game access patterns across multiple game genres.

* **Rendering pipeline:** render stage detail, sprites, animations, client/server split.

* **The Ctx object:** full specification of contents, lifetime, threading through the system.

* **Pipeline stage declaration:** how a class method is registered as belonging to a specific stage in t2 syntax.

* **Spatial subdivision algorithm:** simple 2D bucket grid vs BSP vs other.

* **Testing:** constructing a minimal world with known state for unit testing a pipeline method.

* **Networking:** client/server split, state synchronization, lag compensation.

* **apply() implementation in Delta:** V2.add() for spatial types vs arithmetic for scalars needs a clean resolution.

# **15. Summary**

The engine's key ideas:

* **OO-style framework, ECS internals:** game developers write classes and methods. ECS component arrays power it underneath. The framework is constrained by ECS needs — developers will feel those constraints.

* **t2lang:** implementation language. Lisp-style s-expression syntax compiling to TypeScript/JavaScript.

* **Pipeline stages = ECS systems:** each stage corresponds directly to an ECS system. The difference is that our stages dispatch through the OO method model rather than iterating component arrays directly.

* **prev and next:** two explicit world frame arguments on every pipeline method. Component variable names are prefixed — prevPos, nextPos — so the frame source is unambiguous at every use site.

* **Single template instance per class:** one instance per class, _reset(entityId, prev, next) called before each pipeline method invocation. Valid because the engine is single-threaded.

* **Base class getters:** Entity defines prevPos, nextPos, prevVel etc. for all component types. All use this.entity_id implicitly. Assert in debug mode if component is missing.

* **Components only for cross-entity access:** you never fetch another entity's instance. Stale instance references are a real bug category — this constraint prevents them.

* **Generational EntityIds:** stored EntityIds are automatically weak references. Stale references return null from get().

* **Sparse set storage:** each component type has its own independent sparse set. No ordering guarantee between different component arrays. Every component lookup is a sparse array indirection — honest random access. Simpler and more flexible than archetype ECS, with weaker cache efficiency for simple high-count entities.

* **Archetype ECS tradeoffs:** archetype ECS gives sequential multi-component iteration but at the cost of expensive component add/remove, invisible data movement, and operational complexity that resembles denormalized OO for large archetypes.

* **Aggregate components:** multiple systems contribute deltas in the same frame. Running totals avoid end-of-frame scans. Base class helpers hide the event machinery.

* **ArmedFighter creates its own Shield:** correct encapsulation. Caller needs no knowledge of Shield. Shield registers itself in its own constructor.

* **Explicit registration in MVP:** constructors call ctx.systemName.register(this) explicitly.

* **Collision flow:** collision system produces symmetric Eid pair Reports filtered by layer/mask. Engine calls onCollide(). Collision system never touches game types.

* **Buffs as entities:** buff reduction is a global system running before physics. Entity classes never mention buffs.

* **ComponentType union and DeltaOf<T>:** closed union is the single source of truth. DeltaOf<T> ensures buff deltas are correctly typed. never branch gives exhaustiveness checking throughout.

* **if-let* for nullable chains:** flat non-nested nullable handling. Matches let* binding syntax. A statement, not an expression.

* **Fixed pipeline in MVP:** twelve stages, fixed order, enforced by the engine.

* **Validators:** post-step pre-commit global systems. Tiered severity. Solve the one-frame latency edge case.

*The design deliberately defers complexity. The MVP is naive by choice. The architecture earns its complexity budget by making each future upgrade a local change to the engine internals rather than a global rewrite of game logic.*

# TODOs

* Explicit Registration: MVP requires explicit registration in constructors, which is error-prone and could be automated sooner. Unfortunately there is no simple, minimal, and robust way to achieve automatic registration with TypeScript. Eventually add some kind of static linter rules to check for registration calls based on which interfaces are implemented by a class.
* Query Engine: Incrementally develop a query engine based on real needs that arise during development.
* Testing Details: The section on testing is brief; more detail on how to unit test pipeline methods or simulate world state would be valuable.
