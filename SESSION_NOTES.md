# Session Notes: FP Visualizations

*Conversation exploring folds, laziness, and type variance in Scala*

---

## Topics Covered

### 1. Recursive vs Tail-Recursive `toList`

**Visualization:** `lazylist-viz.html`

- Recursive version builds stack frames, risks overflow
- Tail-recursive uses accumulator, builds in reverse, then reverses
- Key insight: Work BEFORE recursing = tail position = stack safe

```scala
// Stack unsafe - work after recursing
def toListRecursive: List[A] = this match
  case Cons(h,t) => h() :: t().toListRecursive  // :: happens AFTER
  case Empty => Nil

// Stack safe - work before recursing
def toList: List[A] =
  @tailrec
  def go(ll: LazyList[A], acc: List[A]): List[A] = ll match
    case Cons(h, t) => go(t(), h() :: acc)  // :: happens BEFORE
    case Empty => acc.reverse
  go(this, Nil)
```

### 2. foldLeft vs foldRight Naming

**Visualization:** `foldright-viz.html`

The names refer to **parenthesis nesting**, not traversal direction:

```
foldLeft:   (((z ⊕ 1) ⊕ 2) ⊕ 3) ⊕ 4   ← nests LEFT
foldRight:  1 ⊕ (2 ⊕ (3 ⊕ (4 ⊕ z)))   ← nests RIGHT
```

**Accumulator position mnemonic:**
- `foldLeft`:  `f(acc, elem)` — accumulator on LEFT
- `foldRight`: `f(elem, acc)` — accumulator on RIGHT

### 3. `reverse` is a `foldLeft`

```scala
def reverse[A](list: List[A]): List[A] =
  list.foldLeft(List.empty[A])((acc, elem) => elem :: acc)
```

This enables the pattern: `foldRight = reverse + foldLeft`

### 4. Lazy foldRight and Early Termination

**Visualization:** `lazy-foldright-viz.html`

The key is **by-name parameters** (`=> B`):

```scala
def foldRight[B](z: => B)(f: (A, => B) => B): B
//                  ↑            ↑
//              by-name      by-name (thunk)
```

**For `exists`:**
```scala
def exists(p: A => Boolean): Boolean =
  foldRight(false)((a, b) => p(a) || b)
```

- If `p(a)` is `true`: `true || b` short-circuits, `b` never evaluated
- If `p(a)` is `false`: must evaluate `b` to continue

**Best case:** O(1) - find on first element
**Worst case:** O(n) - no match, full descent + unwind (same as strict)

### 5. `headOption` via foldRight

```scala
def headOption: Option[A] =
  this.foldRight(None: Option[A])((a, b) => Some(a))
//                                      ↑
//                               b completely ignored!
```

Since `b` is never referenced, the entire tail is never evaluated. O(1).

### 6. `takeWhile` via foldRight

**Correct version:**
```scala
def takeWhileViaFoldRight(p: A => Boolean): LazyList[A] =
  this.foldRight(empty)((a, b) => if p(a) then cons(a, b) else empty)
//                                                           ^^^^^
```

**Why `else empty`, not `else b`:**
- `else b` = skip non-matching, continue (acts like filter!)
- `else empty` = stop at first non-match (correct takeWhile semantics)

### 7. Strict vs By-Name in `append`

```scala
// ❌ WRONG - a2 evaluated immediately
def append(a2: LazyList[A2]): LazyList[A2]

// ✓ CORRECT - a2 only evaluated when needed
def append(a2: => LazyList[A2]): LazyList[A2]
//            ^^
```

With strict: `finiteList.append(infiniteList)` → 💥 evaluates infinite list immediately

With by-name: `finiteList.append(infiniteList)` → ✓ infinite list stays as thunk

**Laziness must be "all the way down"** — one strict link breaks the chain.

### 8. Variance: `[A2 >: A]`

Why this pattern is needed:

```scala
enum LazyList[+A]:  // + means covariant
  def append[A2 >: A](a2: => LazyList[A2]): LazyList[A2]
//           ^^^^^^^ lower bound: A2 is supertype of A
```

**Covariance (`+A`):** If `Dog <: Animal`, then `LazyList[Dog] <: LazyList[Animal]`

**The problem:** Can't put covariant `A` in input position (parameter type)

**The solution:** Introduce `A2 >: A`, widen the result type

```scala
dogs.append(cats)  // A=Dog, A2 inferred as Animal
// Result: LazyList[Animal]
```

### 9. Variance Comparison: Scala vs TypeScript

| Aspect | Scala | TypeScript |
|--------|-------|------------|
| Variance declaration | Required (`+A`, `-A`) | Optional (`out`, `in`) |
| Enforcement | Strict, compile-time | Partial, pragmatic |
| Arrays | Invariant (safe) | Covariant (unsound) |
| Philosophy | "If it compiles, it's correct" | "If it's useful, allow it" |

TypeScript chose **pragmatism over soundness**:
```typescript
// TypeScript allows this (but it's unsafe!)
const dogs: Dog[] = [fido];
const animals: Animal[] = dogs;  // ✓ OK
animals.push(cat);  // ✓ Compiles, 💥 Runtime error
```

---

## Key Insights

1. **Laziness is asymmetric** — helps best case, doesn't hurt worst case

2. **By-name = "sealed box"** — pass it along without opening; only open when needed

3. **foldRight + lazy cons = demand-driven pipeline** — only pay for what you consume

4. **Tail recursion = work before recursing** — nothing left to do after call returns

5. **Variance protects substitutability** — ensures subtypes work anywhere supertypes expected

---

## Files in This Project

- `lazylist-viz.html` — Recursive vs tail-recursive toList
- `foldright-viz.html` — foldRight with expression tree
- `lazy-foldright-viz.html` — Lazy foldRight, exists, short-circuiting
- `CONCEPTS.md` — Comprehensive concept summary
- `README.md` — Project overview
- `fpinscala` → symlink to exercise repo

---

## Session Date

January 2025
