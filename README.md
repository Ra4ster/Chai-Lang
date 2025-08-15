# Chai — quickstart & language reference

**Chai** (pronounced like the tea) is a compact systems language built around *memory proximity*: declarations place values relative to each other, letting you express stride, interleave, reverse, fill, and exact placement directly in source. Chai aims for predictable, cache-friendly layouts and concise, readable syntax for low-level memory control.

---

## Table of contents

1. Overview & philosophy  
2. Quick hello world  
3. Primitives (basic types)  
4. Memory model & arena concepts  
5. Placement / proximity operators (`+`, `-`, `/`, `*`, `~`, `%`)  
6. Anchors & global bases (`origin`, `warp`, `cacheline`, etc.)  
7. Views & reinterpretation semantics  
8. Objects, structs, inheritance, methods, and overload rules  
9. Vectors / collections and counts  
10. GPU / warp directives & profiles  
11. Intrinsics & runtime helpers (`closest`, `spacefrom`, `move`, `align`, `delete`)  
12. Compile-time vs runtime checks, warnings, and sanitizer mode  
13. Examples (memory visuals)  
14. Performance notes & comparison to C/stack/heap patterns  
15. FAQ & best practices  
16. Glossary, licensing, and next steps

---

## 1. Overview & philosophy

- Chai makes **memory layout a first-class part of the language**. Declarations on the left of `=` specify *where and how* a value sits in memory; the right side supplies the value.
- All placements are **additive & non-destructive**: nothing is moved to make room for something else.
- Operators enable: spacing (`/`), repeating (`*`), reversing (`~`), filling gaps (`%`), and proximity (`+`, `-`).
- Aim: explicit, predictable locality for CPU/GPU optimization without manually managing pointer arithmetic.

---

## 2. Quick hello

Indented code blocks are used for examples:

    // One-line scalar
    int x = 0;

    // Place y immediately after x
    int y + x = 1;

    // Place z immediately before x (left)
    int z - x = 2;

    // Spread bytes of `a` with 3 gaps between each used byte (0-based)
    float a/3 = 1.0;

    // Fill first available gap(s) inside `a` with the bytes of b
    char b%a = 42;

---

## 3. Primitives

Chai uses familiar short names:

- `int` — signed integer (target-dependent width; consult platform doc)
- `float` — 32-bit IEEE float
- `double` — 64-bit IEEE double
- `char` — 8-bit byte
- `bool` — 1-byte boolean
- `short`, `long` — conventional sizes per target
- (future: fixed-point, specialized vector primitives)

**Note:** Size/promotion details go in the platform reference; programmers use `sizeof()` to query compile-time size.

---

## 4. Memory model & arenas

- Program memory is organized into *arenas*; every declaration uses the current arena unless an explicit anchor says otherwise.
- `origin` is a global anchor mapping to the start of the program-data arena.
- All placements are byte-precise. Values declared become metadata entries: `(base_offset, stride, elem_size, count, direction)`.
- Nothing is moved automatically — new declarations occupy free bytes in the described footprint or fallback allocation.

---

## 5. Placement / proximity operators (detailed)

### `+` and `-` (adjacency)
- `T name + anchor = val;` — place `name` immediately after `anchor` (adjacent at anchor’s base or end depending on anchor type).
- `T name - anchor = val;` — place `name` immediately before `anchor`.
- `+` and `-` accept chain expressions: `int x + y + 8 = 3;` (means `x` after `y`, plus 8 bytes).

### `/` (stride / spread) — **0-based**
- `T name / K` spreads each byte (or element part) with `K` gaps between used slots.
- Implementation note: `/K` => effective stride = `1 + K` (i.e., K is the number of empty slots between each used unit).
- Example: `float a/3` ⇒ each byte of `a` occupies a slot; between each used slot are 3 empty bytes.

### `*` (repeat)
- `T name * Count` ⇒ allocate `Count` contiguous copies of `T` (contiguous array repetition).
- Example: `vec3 p*8;` creates 8 vec3s contiguous.

### `~` (reverse)
- `~X` constructs a mirrored view/layout of `X` (element order reversed), allocated as a separate region (unless declared as view).
- Example: `float r~a` produces a reversed layout of `a` (new memory).

### `%` (fill — LHS only)
- LHS-only operator.
- `T filler%container` fills the **first available unused slots** inside `container`'s footprint with exactly the number of bytes required by `T`.  
- **Rules**:
  - `%` does **not** move `container`'s bytes; it only occupies free slots within the container's footprint.
  - If `T` requires `M` bytes and there are `≥ M` contiguous free bytes available somewhere in container footprint, `%` will fill the earliest such run sequentially.
  - If no block of `M` free bytes exists inside the container, the compiler emits a **warning** and `filler` is allocated in the next free area outside the container (fallback allocation).
  - `%` fills exactly the bytes needed; it does not replicate across gaps.
  - `%` is recursive: you can `x%a`, then `y%x` (subject to space).

**Example**:

    // a/3: stride=1+3=4; each of a's bytes at offset 0,4,8,12
    float a/3;
    // b is 4 bytes -> fills first 4 unused slots in a-footprint (without moving a)
    float b%a;

Memory after `a/3`:

    [ a0 ][ ? ][ ? ][ ? ]
    [ a1 ][ ? ][ ? ][ ? ]
    [ a2 ][ ? ][ ? ][ ? ]
    [ a3 ][ ? ][ ? ][ ? ]

After `b%a` (b is 4 bytes):

    [ a0 ][ b0 ][ b1 ][ b2 ]
    [ a1 ][ b3 ][ ? ][ ? ]
    [ a2 ][ ? ][ ? ][ ? ]
    [ a3 ][ ? ][ ? ][ ? ]

---

## 6. Anchors & global bases

- `origin` — program data base.
- `warp` — special GPU-dedicated base (aligns to warp/transaction boundaries with `@warp` directive).
- `cacheline` — CPU cache-line aligned anchor.
- You can anchor to other variables by name: `T x + other` uses `other` as anchor.
- Absolute numeric anchors: `1024 = x + y;` writes to absolute address 1024 of arena (use cautiously; allows linker offsets).

---

## 7. Views & reinterpretation

- A view converts memory to a new type **without changing the original memory sizes or offsets**.
- Syntax: `NewType view < oldVar;` — this creates a **temporary copy** of `oldVar`’s bytes reinterpreted as `NewType`. The copy’s life is scope-limited and does not change the original variable.
- Views are safe: they cannot alter an object’s footprint or cause relocation of other objects.
- For zero-copy reinterpretation, use a `raw_view(name)` intrinsic (only in unsafe contexts) — compiler will warn.

---

## 8. Objects, structs, inheritance, and methods

### Structs
- Declaration identical to C-like structs:

    struct Vec3 {
        float x;
        float y;
        float z;
    }

- Default: fields declared inside maintain insertion order and get offsets within the struct.

### Inheritance & adjacency
- Inheritance exists but **physical adjacency is optional**:
  - `struct Child : Parent` — normal inheritance (no adjacency requirement).
  - `struct Child :adjacent Parent` — request physical adjacency; compiler will attempt to place `Child` immediately after `Parent` (one-time preference). If multiple children request adjacency to the same parent and cannot all fit, compiler errors unless you pick one.
- Multiple inheritance is allowed, but `:adjacent` must explicitly specify which parent(s) to attach to and can only attach to at most one anchor in a contiguous way.

### Methods & overloading (memory-aware)
- Method overloading includes **memory layout** as part of signature:
  - You can overload on type *and* layout/anchor: `fn blend(Vec3 / Vec3 a)` differs from `fn blend(Vec3 a)` (AoS vs SoA).
  - You can restrict overloads to exact address/offset: `fn special(Vec3 a 42)` — only matches if `a`’s base offset equals 42.
  - You can overload on reverse: `fn rev(Vec3 ~a)`.
- Dispatch is static where possible; dynamic dispatch uses layout-ID keys if needed (rare).

---

## 9. Vectors and counts

- Syntax:
  - `vector<T> name / stride = {...};` — stride is bytes between elements (or `T` element-size default).
  - `vector<T> name * count;` — contiguous repetition.
  - `vector<T> name / anchor` — interleave adjacent to `anchor` (pairwise).
  - `vector<T> name / y / x (N);` — triplet interleave anchored to `x` with count `N`.
- `~` reversal can be applied to vectors: `~name` is the reversed element ordering (separate allocation unless declared as view).

---

## 10. GPU / warp directives & profiles

- Declaration directives (optional): `@warp`, `@128B`, `@stride(1/thread)`.
- Example: `vector<float4> albedo / normal / depth @warp @128B;`
  - `@warp` aligns base so that typical device warps coalesce (e.g., 32 threads × 4B lanes).
  - `@128B` forces 128-byte segment alignment.
- Runtime/compile profiles:
  - `cpu` (default) — align to cache lines.
  - `gpu` — default interleaves and segment alignment for device.
  - `embed` — forbids runtime relocation of placed objects (link-time only addresses).

---

## 11. Intrinsics & runtime helpers

- `addr closest(obj)` — returns byte offset of the nearest declared object to current point (negative = behind).
- `int spacefrom(obj)` — returns byte distance between current object and `obj`.
- `move(obj, new_addr)` — relocate `obj` (updates normalized offsets). Disallowed in `embed` profile and may produce relocation cost; compiler will update dependent canonical offsets.
- `align(obj, bytes)` — adjust `obj` base to next `bytes` boundary (warning if overlaps).
- `size(obj)` — number of bytes occupied by an object.
- `layout_id(obj)` — canonical layout token (meta) for overload resolution or hashing.

---

## 12. Compile-time vs runtime checks, warnings, sanitizer

- **Compile-time**:
  - Invalid adjacency, impossible overlaps, forbidden `:adjacent` for multiple children detected at compile-time if all information is known.
  - `require` statements enforce alignment and adjacency early.
- **Runtime**:
  - Fallback allocations, `move()`, and dynamic `.equalsLayout()` checks.
  - `PlacementError` and `OverlapError` thrown at runtime on illegal writes (in sanitizer mode).
- **Sanitizer mode**:
  - Fill guard regions around placements, detect cross-writes, can randomize padding to expose memory races.

---

## 13. Examples (concise with visualizations)

### Example A — Pairwise Add (N elements)

    // declare N
    N = 4321;

    // interleave three arrays for perfect linear walk
    vector<float> x / 3 = {...};         // stride chosen so element width+gaps fit
    vector<float> y / x = {...};         // y[i] adjacent to x[i]
    vector<float> z / y / x (N);         // z adjacent to y adjacent to x (triplet)

Visualization (per-row for one cache line sample):

    [ x0 ][ y0 ][ z0 ][ ? ]   <-- element 0 packed
    [ x1 ][ y1 ][ z1 ][ ? ]   <-- element 1 packed
    ...

Loop lowering (conceptual):

    // linear walk; compiler emits pointer increment code (no multiplies inside)
    xp = ptr_to(x[0]);
    yp = ptr_to(y[0]);
    zp = ptr_to(z[0]);
    for i in 0..N-1:
        *zp = *xp + *yp;
        xp += stride_bytes; yp += stride_bytes; zp += stride_bytes;

### Example B — Fill inside footprint

    float a/3;
    float b%a;    // b fills first 4 empty bytes inside a (if b is 4B)
    char c%a;     // c fills the next 1 empty byte
    short d%a;    // d fills next 2 unused bytes
    // If any filler cannot fit inside a's footprint, compiler warns and allocates elsewhere

Visualization:

    Step1 a/3:
    [ a0 ][ ? ][ ? ][ ? ]
    [ a1 ][ ? ][ ? ][ ? ]
    ...

    After b%a (b=4B):
    [ a0 ][ b0 ][ b1 ][ b2 ]
    [ a1 ][ b3 ][ ? ][ ? ]
    ...

---

## 14. Performance notes & comparison to C/stack/heap

- **Memory footprint** can be identical to C arrays, but Chai gives you *control* to ensure the right bytes are co-located.
- Consider the vector-add example: with `N=4321` and 4B floats, both C and Chai use `~51,852` bytes. The difference is **cache line utilization**:
  - C (three separate arrays): `x`, `y`, `z` may reside in different cache lines → up to 3 cache misses per element in worst-case scattered heap.
  - Chai interleaved: one cache line often contains `x[i]`, `y[i]`, and `z[i]` → significantly fewer misses, better prefetching, and simpler generated loops (pointer increments).
- **Linear walk** reduces address arithmetic (often no multiply per element), reduces branch/miss penalties, and results in higher throughput on CPU & coalesced accesses on GPU.

---

## 15. FAQ & best practices

**Q:** When should I use `%` vs `/`?  
**A:** Use `/` to create gaps/strides deliberately. Use `%` to fill *inside* an existing footprint (useful when you want to pack multiple logical fields into leftover space).

**Q:** Does `%` move existing memory?  
**A:** No. It never moves any existing bytes; it only occupies free slots in a container’s footprint.

**Q:** What happens when a filler doesn’t fit?  
**A:** The compiler emits a warning and allocates the filler in the next free region outside the container.

**Q:** Should I always interleave for speed?  
**A:** Interleaving is best when you repeatedly access related fields together. For other access patterns, SoA (separate arrays) might still be better. Use `require` statements and microbenchmarks.

**Best practices**
- Prefer `vector<T> z / y / x` for reductions/combines to minimize cache misses.
- Use `@warp`, `@128B` where targeting GPUs.
- Use `align(obj, bytes)` and `require` to enforce coalescing invariants.
- Keep `embed` profile for systems where relocation must be disallowed.

---

## 16. Glossary, license & next steps

**Glossary**
- `anchor` — the variable or base that defines a placement reference.
- `stride` — bytes between logical elements (for `/`).
- `fill` — `%` operator semantics (first available gap fill).
- `reverse` — `~` operator; mirrored layout.
- `layout id` — canonical metadata describing anchor + offsets + stride.

**License & contribution**
- Chai language is initially specified under your chosen OSS license (e.g., MIT/Apache). Include source and examples in a repo with tests for placement semantics.

**Next steps**
- Formal grammar (BNF) and spec for parser.
- Two-pass layout solver (support forward references).
- Compiler lowering to C++/LLVM with pointer arithmetic optimization.
- GPU backend using `@warp` hints, and a visualization tool for layout DOT/ASCII outputs.

---

Thank you — this file should contain everything a new user needs: syntax, semantics, examples, visuals, and best practices. If you want, I can now produce a formal BNF grammar, a minimal parser sketch, or a one-file runnable interpreter prototype that enforces these placement rules. Which would you like next?
