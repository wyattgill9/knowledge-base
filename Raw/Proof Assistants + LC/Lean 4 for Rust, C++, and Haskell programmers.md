
Lean 4 is a dependently-typed functional programming language that doubles as an interactive theorem prover — and unlike earlier proof assistants, it's designed to write real software. If you know Rust's type-level reasoning, Haskell's monadic abstractions, or C++'s performance ambitions, Lean 4 synthesizes all three into a single coherent system where proofs _are_ programs and the compiler optimizes pure functional code into efficient imperative executables. This tutorial skips FP fundamentals and focuses on what will surprise you coming from those languages.

---

## 1. Project setup with Lake and the module system

Lake (Lean Make) ships with the Lean 4 toolchain and fills the same role as Cargo or Cabal/Stack. Version management uses `elan`, analogous to `rustup` or `ghcup`.

```bash
# Install elan (manages Lean versions, like rustup)
curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh

# Create a new project (like `cargo new`)
lake new myproject

# Build and run
cd myproject && lake build && lake exe myproject
```

A generated project looks like this:

```
myproject/
├── lakefile.toml         # Build configuration (TOML, default format)
├── lean-toolchain        # Pins Lean version: "leanprover/lean4:v4.12.0"
├── lake-manifest.json    # Lock file (auto-generated)
├── Main.lean             # Executable entry point
├── Myproject.lean        # Library root (imports sub-modules)
└── Myproject/
    └── Basic.lean        # Library code
```

The default `lakefile.toml` is declarative:

```toml
name = "myproject"
version = "0.1.0"
defaultTargets = ["myproject"]

[[lean_lib]]
name = "Myproject"

[[lean_exe]]
name = "myproject"
root = "Main"

[[require]]
name = "mathlib"
git = "https://github.com/leanprover-community/mathlib4"
rev = "main"
```

For advanced configuration, Lake also supports `lakefile.lean` written in full Lean — unlike Cargo, your build file can contain arbitrary computation. **There is no central package registry like crates.io**; dependencies point at Git repos. The [Reservoir](https://reservoir.lean-lang.org/) site provides discovery.

**Modules map directly to files.** The path `Myproject/Utils/String.lean` becomes module `Myproject.Utils.String`. Unlike Haskell, modules and namespaces are orthogonal — a file `Greeting/Smile.lean` can define names in namespace `Expression.happy`. Imports must appear at the file top, are absolute from the project root, and cannot be circular.

```lean
import Myproject.Utils          -- import a module (file)
open Myproject.Utils (helper)   -- bring specific names into scope
open Myproject.Utils in #eval helper 5  -- open temporarily for one expression
```

Every file implicitly imports the Prelude (`Nat`, `Bool`, `String`, `List`, `Option`, etc.). The `namespace`/`end` blocks organize names; `section`/`end` blocks scope `variable` declarations. `private` restricts to the current file; `protected` requires full qualification even when the namespace is opened.

The entry point is `def main : IO Unit` (or `IO UInt32` for exit codes), directly analogous to Haskell's `main :: IO ()` and Rust's `fn main()`:

```lean
def main (args : List String) : IO UInt32 := do
  for arg in args do
    IO.println s!"arg: {arg}"
  return 0
```

---

## 2. Inductive types, structures, and typeclasses

### Inductive types are Lean's algebraic data types

They unify Rust's `enum` and Haskell's `data`, but with dependent typing:

```lean
-- Simple enum (like Rust's fieldless enum)
inductive Direction where
  | north | south | east | west
  deriving Repr, BEq

-- Sum type with data (like Rust's enum with fields)
inductive Shape where
  | circle    : Float → Shape
  | rectangle : Float → Float → Shape

-- Recursive type (like Haskell's data List a = Nil | Cons a (List a))
inductive MyList (α : Type) where
  | nil  : MyList α
  | cons : α → MyList α → MyList α
```

Constructors live in the type's namespace (`Shape.circle`, not bare `circle`), and dot shorthand `.circle` works when the expected type is known. The `deriving` clause works like Rust's `#[derive(...)]` or Haskell's `deriving`.

**What Lean adds over Rust/Haskell**: indexed families, where constructors can target _different indices_ of the type. This is impossible in Rust and requires GADTs in Haskell:

```lean
-- Length-indexed vector: the Nat index tracks length at the type level
inductive Vec (α : Type) : Nat → Type where
  | nil  : Vec α 0
  | cons : α → Vec α n → Vec α (n + 1)

-- head is ONLY callable on non-empty vectors — the type prevents it
def Vec.head : Vec α (n + 1) → α
  | .cons x _ => x
```

### Structures are single-constructor inductives with batteries

Structures provide named fields, dot notation, defaults, functional update syntax, and inheritance:

```lean
structure Point (α : Type) where
  x : α
  y : α
  deriving Repr

-- Three equivalent ways to construct:
def p1 := Point.mk 1 2
def p2 : Point Nat := { x := 1, y := 2 }
def p3 : Point Nat := ⟨1, 2⟩              -- anonymous constructor (single-constructor types only)

-- Functional update (like Rust's `..` or Haskell record update):
def p4 := { p1 with x := 10 }

-- Structure inheritance (Rust has no equivalent; Haskell uses no inheritance):
structure Point3D (α : Type) extends Point α where
  z : α
```

Methods use dot notation naturally — `def Point.add (p q : Point Nat) := ...` lets you write `p.add q`.

### Typeclasses are structures with instance resolution

Lean 4's typeclasses are literally structures, registered in an instance database. The `class` keyword is sugar for `structure` plus instance registration:

```lean
class Greet (α : Type) where
  greet : α → String

instance : Greet String where
  greet s := s!"Hello, {s}!"

-- Square brackets trigger instance resolution (like Haskell's =>)
def greetAll [Greet α] (xs : List α) : List String :=
  xs.map Greet.greet
```

|Feature|Lean 4|Rust|Haskell|
|---|---|---|---|
|Declaration|`class C (α : Type) where`|`trait C { }`|`class C a where`|
|Implementation|`instance : C Nat where`|`impl C for u64 { }`|`instance C Int where`|
|Constraint|`[C α]`|`T: C`|`C a =>`|
|Orphan rules|**None** — instances anywhere|Strict orphan rules|Partial|
|Resolution|Backtracking search|Monomorphization|Dictionary passing|
|Output params|`outParam`|Associated types|Type families|

Common built-in typeclasses: **`ToString`**, **`BEq`** (==), **`Hashable`**, **`Inhabited`** (default value), **`Repr`** (#eval display), **`Ord`**, **`Add`**/**`Mul`**, **`Monad`**/**`Functor`**.

### Universe polymorphism prevents Russell's paradox

Lean's type hierarchy is `Prop : Type : Type 1 : Type 2 : ...` (equivalently `Sort 0 : Sort 1 : Sort 2 : ...`). Without universe polymorphism, `List` could only hold values in `Type 0`. Universe variables fix this:

```lean
universe u v

-- The stdlib List is actually:
inductive List (α : Type u) : Type u where
  | nil : List α
  | cons : α → List α → List α

-- With autoImplicit (default: on), universe variables are inferred automatically
def map (f : α → β) : List α → List β
  | [] => []
  | x :: xs => f x :: map f xs
```

Haskell's kind system (`Type`, `Type -> Type`, `Constraint`) is much simpler — it has no infinite hierarchy because it doesn't need logical consistency. Lean's universes prevent `Type : Type`, which would make the logic unsound.

---

## 3. Pattern matching, do-notation, monads, and core syntax

### Pattern matching is pervasive

You can pattern-match directly in function definitions (no `match` needed), with `match` expressions, or in lambdas:

```lean
-- Direct definition patterns (like Haskell equation-style):
def fib : Nat → Nat
  | 0 => 0
  | 1 => 1
  | n + 2 => fib (n + 1) + fib n

-- Match expression (like Rust's match):
def describe (n : Nat) : String :=
  match n with
  | 0 => "zero"
  | 1 => "one"
  | _ => "many"

-- Lambda pattern matching:
def isZero := fun | 0 => true | _ => false

-- Or-patterns (merged):
def classify : Nat → String
  | 0 | 1 => "small"
  | _     => "big"

-- if-let equivalent in do blocks:
def getName (x : Option String) : IO Unit := do
  let some name := x | IO.println "no name"; return
  IO.println s!"Hello, {name}"
```

### Implicit arguments — the biggest syntax surprise

Lean 4 uses three kinds of parameter brackets, and understanding them is essential:

```lean
-- (x : α)  — Explicit: must be supplied
-- {x : α}  — Implicit: inferred by unification
-- [C α]    — Instance implicit: resolved by typeclass search

def identity {α : Type} (x : α) : α := x
#eval identity 5            -- α inferred as Nat
#eval @identity Nat 5       -- @ makes all implicits explicit
#eval identity (α := Nat) 5 -- named implicit argument
```

Greek letters (`α`, `β`, `γ`) are auto-bound as implicit type parameters by default. The `·` (cdot) shorthand creates anonymous lambdas: `[1,2,3].map (· + 1)` is `[1,2,3].map (fun x => x + 1)`.

### Lambdas, let, and where

```lean
-- Lambdas use fun/=>
def add := fun (x y : Nat) => x + y

-- let introduces local bindings; let rec for local recursion
def example1 : Nat :=
  let x := 5
  let y := x + 3
  x * y

-- where defines helpers after the main body (like Haskell's where)
def fibonacci (n : Nat) : Nat := (loop n 0 1).1
where
  loop : Nat → Nat → Nat → Nat × Nat
    | 0, a, b => (a, b)
    | n + 1, a, b => loop n b (a + b)
```

---

## 4. Do-notation and monadic programming — Lean 4 vs Haskell

Lean 4's `do`-notation is a **superset** of Haskell's. It adds mutable variables, for-loops, early return, and nested actions. If you know Haskell monads, you'll feel at home — but with imperative ergonomics.

```lean
-- IO monad (like Haskell's IO):
def main : IO Unit := do
  IO.println "What is your name?"
  let name ← IO.getLine          -- ← extracts from monad (Haskell: <-)
  IO.println s!"Hello, {name.trim}!"
```

**Key differences from Haskell:**

**`return` is early return, not `pure`.** This is the biggest gotcha. In Haskell, `return` is just `pure` — it doesn't affect control flow. In Lean 4, `return` exits the `do` block immediately, desugaring to `ExceptT`:

```lean
def findFirst (p : α → Bool) : List α → Option α
  | [] => none
  | x :: xs => do
    if p x then return x   -- early exit!
    findFirst p xs
```

**`let mut` provides mutable variables** (desugars to `StateT`):

```lean
def sum (xs : List Nat) : Nat := Id.run do
  let mut total := 0
  for x in xs do
    total := total + x
  return total
-- Id.run runs a do-block in the identity monad (pure computation)
```

**For-loops and single-branch `if`** just work:

```lean
def countVowels (s : String) : Nat := Id.run do
  let mut count := 0
  for c in s.toList do
    if "aeiouAEIOU".contains c then
      count := count + 1
  return count
```

**Nested actions** with `(← expr)` inline monadic extraction:

```lean
-- Instead of: let line ← IO.getLine; pure s!"Hello, {line.trim}!"
def greeting : IO String := do
  pure s!"Hello, {(← IO.getLine).trim}!"
```

|Feature|Lean 4|Haskell|
|---|---|---|
|Bind|`let x ← action`|`x <- action`|
|Return|`return x` **(early return!)**|`return x` (just `pure`)|
|Mutable vars|`let mut x := 0`|Not available|
|For loops|`for x in xs do`|Not built-in|
|Nested actions|`(← expr)`|Not available|
|Single-branch if|`if cond then action`|Requires else|

### The three essential monads

**`IO`** — effects, file I/O, console:

```lean
def main : IO Unit := do
  IO.FS.writeFile "out.txt" "hello"
  let contents ← IO.FS.readFile "out.txt"
  IO.println contents
```

**`Option`** — nullable values (Haskell's `Maybe`, Rust's `Option`). Use `?` suffix for operations that might fail:

```lean
def safeIndex (xs : List α) : Option (α × α) := do
  let first ← xs[0]?
  let third ← xs[2]?
  pure (first, third)
```

**`Except ε α`** — error handling (Haskell's `Either`, Rust's `Result`):

```lean
def divide (x y : Float) : Except String Float :=
  if y == 0 then .error "division by zero"
  else .ok (x / y)

def computation : Except String Float := do
  let a ← divide 10 2
  let b ← divide a 3
  pure (a + b)
```

### Monad transformers and stacking

```lean
-- StateM σ α = σ → (α × σ)
def countdown : StateM Nat (List Nat) := do
  let mut result : List Nat := []
  while (← get) > 0 do
    let n ← get
    result := n :: result
    set (n - 1)
  return result

#eval countdown.run 5   -- ([1, 2, 3, 4, 5], 0)

-- ReaderT for configuration:
structure Config where
  verbose : Bool
  maxRetries : Nat

def greet : ReaderT Config IO Unit := do
  let cfg ← read
  if cfg.verbose then IO.println s!"[VERBOSE] Max retries: {cfg.maxRetries}"
  IO.println "Hello!"
```

---

## 5. Dependent types, Curry-Howard, and theorem proving

This is what fundamentally separates Lean from Rust and Haskell. In Lean, **types can depend on values**, **propositions are types**, and **proofs are programs**.

### Dependent types let return types vary with input values

```lean
-- The return type depends on the runtime boolean:
def eitherNatOrString (b : Bool) : if b then Nat else String :=
  if b then 42 else "hello"
```

This is impossible in Rust or standard Haskell. The dependent function type `(x : α) → β x` (Pi type) generalizes the ordinary arrow `α → β` — when `β` doesn't depend on `x`, you get a normal function. Lean's `Fin n` (naturals less than `n`), `Vector α n` (length-indexed arrays), and subtypes `{ x : α // p x }` all exploit this:

```lean
-- Fin n: natural numbers guaranteed < n
def three : Fin 5 := ⟨3, by omega⟩

-- Subtype: a Nat with a proof it's positive
def PosNat := { n : Nat // n > 0 }
def safeDivide (a : Nat) (b : PosNat) : Nat := a / b.val
```

### Propositions as types (Curry-Howard)

The correspondence is complete in Lean: every logical connective maps to a type.

|Logic|Type|Notation|
|---|---|---|
|Implication `p → q`|Function `α → β`|`→`|
|Conjunction `p ∧ q`|Product `α × β`|`∧`|
|Disjunction `p ∨ q`|Sum `α ⊕ β`|`∨`|
|True|Unit `()`|—|
|False|Empty|—|
|∀ x, P x|Dependent function `(x : α) → P x`|`∀`|
|∃ x, P x|Dependent pair `Σ x : α, P x`|`∃`|
|¬p|`p → False`|`¬`|

A `theorem` is just a `def` whose return type lives in `Prop`. The proof is the term you construct:

```lean
-- Proving conjunction is like building a pair:
theorem and_swap (hp : p) (hq : q) : q ∧ p := ⟨hq, hp⟩

-- Proving implication is writing a function:
theorem modus_ponens (hp : p) (hpq : p → q) : q := hpq hp

-- Existential proof provides witness + evidence:
example : ∃ n : Nat, n > 0 := ⟨1, by omega⟩
```

### Prop vs Type — proof irrelevance and erasure

**`Prop`** (Sort 0) is the universe of propositions. It has a critical property: **proof irrelevance** — any two proofs of the same `Prop` are definitionally equal. This means proofs carry no computational content and are **erased at runtime**.

```lean
-- These two proofs are definitionally equal:
theorem proof_irrel (h₁ h₂ : 2 + 2 = 4) : h₁ = h₂ := rfl

-- Practical consequence: bounds proofs add zero runtime overhead
def safeGet (arr : Array α) (i : Nat) (h : i < arr.size) : α := arr[i]
-- h is erased — only i and arr exist at runtime
```

Use **`Bool`** when you need runtime branching; use **`Prop`** when you need logical properties. The `Decidable` typeclass bridges them — `if h : p then ... else ...` requires `[Decidable p]` and gives you proof access in each branch:

```lean
def safeDiv (a b : Nat) : Nat :=
  if h : b = 0 then 0        -- h : b = 0 available here
  else a / b                   -- h : ¬(b = 0) available here
```

### Tactic mode — interactive proof construction

The `by` keyword enters tactic mode, where you transform goals step-by-step instead of constructing proof terms directly:

```lean
theorem add_comm_example (n m : Nat) : n + m = m + n := by
  omega  -- linear arithmetic solver
```

**Essential tactics:**

**`intro` / `intros`** move quantified variables into the context:

```lean
example : ∀ (n : Nat), n = n := by
  intro n    -- context now has n : Nat, goal is n = n
  rfl
```

**`apply`** unifies the goal with a lemma's conclusion, creating subgoals for its hypotheses:

```lean
example (hp : p) (hq : q) : p ∧ q := by
  apply And.intro  -- splits into goals ⊢ p and ⊢ q
  · exact hp
  · exact hq
```

**`exact`** closes a goal by providing the exact proof term:

```lean
example (h : p) : p := by exact h
```

**`rfl`** proves `a = a` (or anything definitionally equal):

```lean
example : 2 + 3 = 5 := by rfl
```

**`rw`** rewrites using equalities:

```lean
example (h : a = b) : a + 1 = b + 1 := by rw [h]
example (h : a = b) : b + 1 = a + 1 := by rw [← h]  -- rewrite right-to-left
```

**`simp`** is the workhorse — it repeatedly applies `@[simp]`-tagged lemmas:

```lean
example (n : Nat) : n + 0 = n := by simp
example (xs : List α) : xs ++ [] = xs := by simp
example (h : a = b) : a + 1 = b + 1 := by simp [h]
```

**`omega`** solves linear arithmetic over `Nat` and `Int`:

```lean
example (n : Nat) (h : n > 0) : n ≥ 1 := by omega
example (a b : Nat) (h1 : a ≤ b) (h2 : b ≤ a) : a = b := by omega
```

**`cases`** destructs inductive types:

```lean
example (h : p ∨ q) : q ∨ p := by
  cases h with
  | inl hp => exact Or.inr hp
  | inr hq => exact Or.inl hq
```

**`induction`** performs structural induction:

```lean
theorem append_nil (as : List α) : as ++ [] = as := by
  induction as with
  | nil => rfl
  | cons a as ih => simp [ih]
```

**`calc`** chains equalities/inequalities:

```lean
example (a b c : Nat) (h1 : a ≤ b) (h2 : b < c) : a < c := by
  calc a ≤ b := h1
       _ < c := h2
```

**`decide`** evaluates decidable propositions: `example : 2 + 2 = 4 := by decide`

**`sorry`** is the escape hatch — proves anything but marks the proof incomplete.

### Where proofs meet programs

Array indexing with `arr[i]` requires a proof that `i < arr.size`. For literals, Lean auto-generates it. For variables, you either provide proof or use safe alternatives:

```lean
-- Proof-carrying access (zero runtime overhead — proof erased):
def safeGet (arr : Array Nat) (i : Nat) (h : i < arr.size) : Nat := arr[i]

-- Dependent if provides the proof:
def getOrZero (arr : Array Nat) (i : Nat) : Nat :=
  if h : i < arr.size then arr[i] else 0

-- Option-returning (no proof needed):
#eval #[10, 20, 30][5]?   -- none

-- Panic on failure:
#eval #[10, 20, 30][1]!   -- 20
```

---

## 6. Memory management — functional but in-place

Lean 4 uses **reference counting** (not tracing GC) and a key optimization: when a data structure's reference count is exactly 1, operations that would normally copy can **mutate in place**. The semantics stay purely functional — the optimization is invisible to user code.

This comes from the "Counting Immutable Beans" paper (Ullrich & de Moura, 2019). The core idea: **`Array.set` checks the reference count at runtime; if the array is exclusively owned, it modifies the buffer in place (O(1)); otherwise it copies first.** The same applies to `push`, `pop`, `swap`, `modify`, and `reverse`.

```lean
-- O(1) when arr is exclusively owned:
def updateArray (arr : Array Nat) : Array Nat :=
  arr.set! 0 42

-- O(n) if you keep a reference to the original:
def updateArrayShared (arr : Array Nat) : Array Nat × Array Nat :=
  (arr.set! 0 42, arr)   -- arr is shared, so set! must copy
```

**Borrowing** further optimizes by eliding reference count updates for parameters that are only read, not consumed. The compiler infers this automatically.

### How this compares to Rust's ownership

|Aspect|Lean 4|Rust|
|---|---|---|
|When checked|**Runtime** (RC inspection)|**Compile time** (borrow checker)|
|Programmer effort|None — automatic|Must satisfy borrow checker|
|Failure mode|Silently copies if shared (safe, slower)|Won't compile|
|Cyclic data|Cannot create cycles (inductive types)|Requires `Rc`+`RefCell`|
|Mental model|"Write pure code; compiler optimizes"|"Explicitly manage ownership"|

Rust gives you **compile-time guarantees** about uniqueness. Lean gives you **runtime optimization** based on uniqueness. Rust never silently copies; Lean never rejects your code for ownership violations. The tradeoff is that Lean may unexpectedly copy if you accidentally share a reference, causing a performance cliff rather than a compile error.

The `@[inline]` attribute inlines functions at call sites; `@[specialize]` generates specialized variants for specific type parameters (similar to Rust's monomorphization, but opt-in rather than default).

---

## 7. Standard library essentials

### Array — the primary sequence type

`Array α` is a contiguous dynamic buffer — Lean's recommended default over `List` for computational code:

```lean
-- Creation
let arr := #[1, 2, 3, 4, 5]
let zeros := Array.replicate 5 0          -- #[0, 0, 0, 0, 0]
let range := Array.ofFn (n := 5) (·.val)  -- #[0, 1, 2, 3, 4]

-- Core operations (all O(1) amortized when exclusively owned)
#eval #[1, 2, 3].push 4       -- #[1, 2, 3, 4]
#eval #[1, 2, 3].pop          -- #[1, 2]
#eval #[1, 2, 3].set! 1 99    -- #[1, 99, 3]

-- Functional transforms
#eval #[1, 2, 3].map (· * 2)          -- #[2, 4, 6]
#eval #[1, 2, 3, 4].filter (· > 2)    -- #[3, 4]
#eval #[1, 2, 3, 4].foldl (· + ·) 0   -- 10
#eval #[1, 2] ++ #[3, 4]              -- #[1, 2, 3, 4]

-- Indexing: three styles
#eval #[10, 20, 30][1]?   -- some 20 (Option, safe)
#eval #[10, 20, 30][1]!   -- 20 (panics if OOB)
-- arr[i] with proof requires i < arr.size
```

### List — for pattern matching and proofs

```lean
def sum : List Nat → Nat
  | [] => 0
  | x :: xs => x + sum xs

#eval [1, 2, 3].map (· * 2)     -- [2, 4, 6]
#eval [1, 2, 3].reverse         -- [3, 2, 1]
#eval [1, 2, 3].length          -- 3
#eval [1, 2] ++ [3, 4]          -- [1, 2, 3, 4]
```

Due to RC memory reuse, `List.map` on a uniquely-referenced list reuses cons cells — **zero allocation**.

### HashMap

```lean
import Std.Data.HashMap

def main : IO Unit := do
  let mut map : Std.HashMap String Nat := {}
  map := map.insert "alice" 30
  map := map.insert "bob" 25
  IO.println s!"{map.find? "alice"}"   -- some 30
  IO.println s!"{map.find? "eve"}"     -- none
  IO.println s!"{map.size}"            -- 2
```

The official docs warn: **"Users should make sure that the hash map is used linearly to avoid expensive copies"** — keeping references to both old and new maps forces a full copy.

### String and IO

Strings are UTF-8 encoded. Indexing uses byte-offset positions (`String.Pos`), not character indices — similar to Rust's `String`:

```lean
#eval "Hello" ++ " World"                  -- "Hello World"
#eval s!"x = {1 + 2}"                     -- "x = 3"
#eval "Hello World".splitOn " "            -- ["Hello", "World"]
#eval "  hello  ".trim                     -- "hello"
#eval "42".toNat?                          -- some 42

-- File I/O
def processFile : IO Unit := do
  IO.FS.writeFile "output.txt" "Hello, Lean 4!"
  let contents ← IO.FS.readFile "output.txt"
  IO.println contents
  let lines ← IO.FS.lines "data.txt"     -- Array String
  for line in lines do
    IO.println s!"Line: {line}"

-- Process execution
def runCmd : IO Unit := do
  let output ← IO.Process.output { cmd := "echo", args := #["hello"] }
  IO.println output.stdout
```

---

## 8. Termination checking — total by default

Lean 4 requires all functions to terminate unless explicitly opted out. This is non-negotiable for logical consistency — a non-terminating function of type `False` would prove anything. Haskell and Rust have no such requirement.

**Structural recursion** is accepted automatically when you recurse on strict sub-terms:

```lean
def length : List α → Nat
  | [] => 0
  | _ :: xs => 1 + length xs    -- xs is structurally smaller
```

**`termination_by`** specifies a decreasing measure when structural recursion isn't detected:

```lean
def gcd (m n : Nat) : Nat :=
  if m = 0 then n
  else gcd (n % m) m
termination_by m                -- m decreases at each call

def binarySearch (arr : Array Nat) (target : Nat) (lo hi : Nat) : Option Nat :=
  if lo >= hi then none
  else
    let mid := (lo + hi) / 2
    if arr[mid]! == target then some mid
    else if arr[mid]! < target then binarySearch arr target (mid + 1) hi
    else binarySearch arr target lo mid
termination_by hi - lo          -- this quantity strictly decreases
```

**`decreasing_by`** provides a manual proof when the automatic tactic (`omega`, `simp_wf`) can't discharge the termination obligation:

```lean
def gcd' (m n : Nat) : Nat :=
  if m = 0 then n
  else gcd' (n % m) m
termination_by m
decreasing_by
  simp_wf
  exact Nat.mod_lt _ (by omega)
```

**`partial`** opts out entirely — the function is compiled normally but becomes opaque to the proof checker. **It cannot be used in proofs:**

```lean
-- Nobody has proved Collatz terminates
partial def collatz (n : Nat) : Nat :=
  if n ≤ 1 then 0
  else if n % 2 = 0 then 1 + collatz (n / 2)
  else 1 + collatz (3 * n + 1)

-- Server loops, REPLs, and event loops need partial:
partial def serverLoop (state : ServerState) : IO Unit := do
  let req ← receiveRequest
  let (resp, newState) := handleRequest state req
  sendResponse resp
  serverLoop newState
```

**`unsafe`** is even more permissive — it allows non-termination, pointer equality (`ptrEq`), and unchecked casts.

---

## 9. Metaprogramming — extending the language from within

Lean 4 is self-hosted (~90% written in Lean), so `import Lean` gives you access to the compiler's internals. There are three levels of extensibility, from simple to fully powered.

### Level 1: Notation declarations

```lean
-- Infix operators:
infixl:65 " ⊕ " => HAdd.hAdd    -- left-associative, precedence 65
infixr:80 " ** " => HPow.hPow   -- right-associative, precedence 80
prefix:100 "√" => Nat.sqrt

-- General mixfix notation:
notation "⟪" x ", " y "⟫" => (x, y)
#check ⟪1, 2⟫   -- (1, 2) : Nat × Nat

-- scoped limits visibility to when the namespace is opened:
namespace MyDSL
  scoped infixl:65 " |+| " => HAdd.hAdd
end MyDSL
```

### Level 2: Macros — syntax-to-syntax transformations

Macros operate on `Syntax` trees before type-checking, like Rust proc macros but hygienic and definable in the same file:

```lean
-- `macro` combines syntax declaration + transformation:
macro l:term:10 " XOR " r:term:11 : term =>
  `((!$l && $r) || ($l && !$r))

#eval true XOR false   -- true

-- More complex: a repeat tactic
syntax "rep" tactic : tactic
macro_rules
  | `(tactic| rep $t) => `(tactic| first | $t; rep $t | skip)
```

Quotation with `` `(...) `` creates/matches syntax; `$x` is anti-quotation (splicing). `$[$xs],*` handles comma-separated repetitions. Lean's **hygienic macro system** uses macro scopes to prevent accidental name capture — identifiers introduced by macros get unique scopes.

### Level 3: Elaborators — syntax-to-Expr with full type information

When macros aren't enough (you need type information, metavariable manipulation, or environment access), use `elab`:

```lean
import Lean
open Lean Elab Tactic

-- Custom command:
elab "#assertType " t:term " : " ty:term : command => do
  let tp ← Command.liftTermElabM (Term.elabType ty)
  discard <| Command.liftTermElabM (Term.elabTermEnsuringType t tp)
  logInfo "type checks!"

#assertType 5 : Nat   -- type checks!

-- Custom tactic:
elab "my_omega" : tactic => do
  let goal ← getMainGoal
  let goalType ← goal.getDecl >>= (pure ·.type)
  dbg_trace f!"Solving: {goalType}"
  evalTactic (← `(tactic| omega))
```

The monad hierarchy is **`CoreM` ⊂ `MetaM` ⊂ `TermElabM` ⊂ `TacticM`**, each layer adding capabilities (environment access → metavariables → elaboration → goal management).

### Custom DSLs via syntax categories

```lean
declare_syntax_cat arith
syntax num : arith
syntax arith " + " arith : arith
syntax arith " * " arith : arith
syntax "(" arith ")" : arith
syntax "eval! " arith : term

macro_rules
  | `(eval! $x:num)           => `($x)
  | `(eval! $x:arith + $y:arith) => `(eval! $x + eval! $y)
  | `(eval! $x:arith * $y:arith) => `(eval! $x * eval! $y)
  | `(eval! ($x:arith))       => `(eval! $x)

#eval eval! (2 + 3) * 4   -- 20
```

---

## 10. Practical examples — Lean 4 as a real programming language

### A file-processing CLI tool

This `cat` clone (adapted from _Functional Programming in Lean_) demonstrates real-world IO, error handling, and command-line argument processing:

```lean
def bufsize : USize := 20 * 1024

partial def dump (stream : IO.FS.Stream) : IO Unit := do
  let buf ← stream.read bufsize
  if buf.isEmpty then return
  (← IO.getStdout).write buf
  dump stream

def fileStream (filename : System.FilePath) : IO (Option IO.FS.Stream) := do
  if !(← filename.pathExists) then
    (← IO.getStderr).putStrLn s!"File not found: {filename}"
    return none
  let handle ← IO.FS.Handle.mk filename .read
  return some (IO.FS.Stream.ofHandle handle)

def process (exitCode : UInt32) : List String → IO UInt32
  | [] => pure exitCode
  | "-" :: args => do dump (← IO.getStdin); process exitCode args
  | filename :: args => do
    match ← fileStream ⟨filename⟩ with
    | none => process 1 args
    | some stream => dump stream; process exitCode args

def main (args : List String) : IO UInt32 :=
  match args with
  | [] => do dump (← IO.getStdin); return 0
  | _  => process 0 args
```

### Stateful computation with monad transformers

```lean
structure AppState where
  counter : Nat
  log : List String

def increment : StateM AppState Unit :=
  modify fun s => { s with counter := s.counter + 1 }

def logMsg (msg : String) : StateM AppState Unit :=
  modify fun s => { s with log := msg :: s.log }

def doWork : StateM AppState Nat := do
  increment; logMsg "incremented"
  increment; logMsg "incremented again"
  return (← get).counter

#eval doWork.run { counter := 0, log := [] }
-- (2, { counter := 2, log := ["incremented again", "incremented"] })
```

### Error handling patterns

```lean
-- Except (like Rust's Result<T, E>):
def parseAge (s : String) : Except String Nat := do
  let n ← s.toNat? |>.toExcept s!"Not a number: {s}"
  if n > 150 then throw "Age unreasonable"
  return n

-- IO error handling with try/catch:
def safeReadFile (path : String) : IO String := do
  try
    IO.FS.readFile path
  catch e =>
    return s!"Error: {e}"

-- Data pipeline combining everything:
def pipeline : IO Unit := do
  let contents ← IO.FS.readFile "data.csv"
  let lines := contents.splitOn "\n" |>.filter (· ≠ "")
  let values := lines.filterMap (·.toNat?)
  let sum := values.foldl (· + ·) 0
  IO.println s!"Sum: {sum}, Count: {values.length}"
```

### Mixing proofs and programs

The real power of Lean 4 is writing **verified programs** — code with compile-time correctness guarantees that cost nothing at runtime:

```lean
-- A sort function that carries a proof obligation
def insertSorted (x : Nat) : List Nat → List Nat
  | [] => [x]
  | y :: ys =>
    if x ≤ y then x :: y :: ys
    else y :: insertSorted x ys

-- Array processing with bounds proofs that vanish at runtime
def sumArray (arr : Array Nat) : Nat := Id.run do
  let mut total := 0
  for h : i in [0:arr.size] do
    total := total + arr[i]  -- h provides proof that i < arr.size
  return total

-- Fin ensures indices are always valid — no runtime bounds checks
def rotateFin (i : Fin n) : Fin n :=
  ⟨(i.val + 1) % n, by omega⟩
```

---

## Conclusion

Lean 4 occupies a unique position in the language landscape. For **Rust programmers**, the key insight is that Lean replaces static ownership analysis with runtime reference counting plus destructive updates — you get functional purity without fighting a borrow checker, at the cost of occasional silent copies. For **Haskell programmers**, Lean's `do`-notation is a strict superset with mutable variables, early return, and for-loops, while dependent types eliminate the need for type-level hacks like singletons and DataKinds. For **C++ programmers**, Lean proves that zero-cost abstractions can come from proofs erased at compile time rather than template metaprogramming.

The most distinctive practical features are **`termination_by`** (you must prove your recursion terminates, or explicitly opt out with `partial`), **`Prop` erasure** (proofs add safety guarantees but zero runtime overhead), and the **three-level metaprogramming stack** (notation → macros → elaborators) that lets you extend the language from within itself. The `by omega` and `by simp` tactics handle the vast majority of proof obligations automatically, so dependent types in everyday code feel surprisingly lightweight. The learning curve is steep, but the reward is a language where the type checker catches bugs that no other mainstream language can express.