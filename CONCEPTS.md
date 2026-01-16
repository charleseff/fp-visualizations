# Functional Programming Concepts: Folds & Laziness

A companion guide to the interactive visualizations, summarizing key insights about folds, tail recursion, and lazy evaluation.

---

## Table of Contents

1. [foldLeft vs foldRight: The Names](#foldleft-vs-foldright-the-names)
2. [Tail Recursion and Stack Safety](#tail-recursion-and-stack-safety)
3. [Lazy foldRight and Early Termination](#lazy-foldright-and-early-termination)
4. [Common Patterns](#common-patterns)

---

## foldLeft vs foldRight: The Names

The names refer to **where the parentheses pile up** (associativity), not the traversal direction.

### foldLeft — Parentheses Nest LEFT

```
foldLeft([1,2,3,4], z, f)  =  f(f(f(f(z, 1), 2), 3), 4)
                           =  (((z ⊕ 1) ⊕ 2) ⊕ 3) ⊕ 4
                                 ↖____↖____↖
                                 nests to the left
```

### foldRight — Parentheses Nest RIGHT

```
foldRight([1,2,3,4], z, f)  =  f(1, f(2, f(3, f(4, z))))
                            =  1 ⊕ (2 ⊕ (3 ⊕ (4 ⊕ z)))
                                       ↗____↗____↗
                                       nests to the right
```

### As Trees

```
        foldLeft                         foldRight
      (left-heavy)                     (right-heavy)

           ⊕                                ⊕
          / \                              / \
         ⊕   4                            1   ⊕
        / \                                  / \
       ⊕   3                                2   ⊕
      / \                                      / \
     ⊕   2                                    3   ⊕
    / \                                          / \
   z   1                                        4   z
```

### The Accumulator Position

Another mnemonic—where does the accumulator sit in `f(?, ?)`?

```scala
foldLeft:   f(acc, elem)    // accumulator on the LEFT
foldRight:  f(elem, acc)    // accumulator on the RIGHT
```

---

## Tail Recursion and Stack Safety

### The Problem with Naive Recursion

```scala
// Recursive toList - NOT stack safe
def toListRecursive: List[A] = this match
  case Cons(h, t) => h() :: t().toListRecursive  // work AFTER recursive call
  case Empty => Nil
```

Each call **waits** for the recursive call to return before it can `::` (cons) the head. This builds up stack frames:

```
toList([A, B, C, D])
  A :: toList([B, C, D])        ← waiting
       B :: toList([C, D])      ← waiting
            C :: toList([D])    ← waiting
                 D :: toList([])
                      Nil       ← base case, now unwind
                 D :: Nil = [D]
            C :: [D] = [C, D]
       B :: [C, D] = [B, C, D]
  A :: [B, C, D] = [A, B, C, D]
```

**Stack depth: O(n)** — will overflow for large lists.

### Tail Recursion: Work BEFORE Recursing

```scala
// Tail-recursive toList - stack safe
def toList: List[A] =
  @tailrec
  def go(ll: LazyList[A], acc: List[A]): List[A] = ll match
    case Cons(h, t) => go(t(), h() :: acc)  // work BEFORE recursive call
    case Empty => acc.reverse
  go(this, Nil)
```

The recursive call is in **tail position**—nothing happens after it returns. The compiler can reuse the same stack frame:

```
go([A, B, C, D], [])
go([B, C, D], [A])        ← same stack frame, just update params
go([C, D], [B, A])
go([D], [C, B, A])
go([], [D, C, B, A])
[D, C, B, A].reverse = [A, B, C, D]
```

**Stack depth: O(1)** — just one frame, reused.

### Why Build in Reverse?

Prepending (`::`) is O(1) for immutable lists, but it puts elements in reverse order. So we:
1. Build the list in reverse: O(n)
2. Reverse at the end: O(n)
3. Total: O(2n) = O(n) time, O(1) stack

This is better than O(n) time with O(n) stack!

### `reverse` is a `foldLeft`

```scala
def reverse[A](list: List[A]): List[A] =
  list.foldLeft(List.empty[A])((acc, elem) => elem :: acc)
```

So the stack-safe `foldRight` pattern is:
```
foldRight(list, z, f) = foldLeft(reverse(list), z, flip(f))
```

---

## Lazy foldRight and Early Termination

### Strict vs Lazy Parameters

```scala
// STRICT foldRight - recursive call evaluated BEFORE f runs
def foldRight[B](z: B)(f: (A, B) => B): B

// LAZY foldRight - recursive call passed as THUNK, evaluated only IF f uses it
def foldRight[B](z: => B)(f: (A, => B) => B): B
//                  ↑            ↑
//              by-name      by-name (thunk)
```

The `=> B` means "this argument is not evaluated until used."

### How `exists` Uses Laziness

```scala
def exists(p: A => Boolean): Boolean =
  foldRight(false)((a, b) => p(a) || b)
//                              ↑
//                     b is a thunk!
```

The `||` operator short-circuits:
- `true || b` → returns `true` without evaluating `b`
- `false || b` → must evaluate `b` to get result

### Best Case: Short-Circuit (Match Found Early)

```
exists(_ == 2) on [1, 2, 3, 4, 5]

f(1, ⟨thunk⟩)
  p(1) = false
  false || ⟨thunk⟩ → must evaluate thunk

  f(2, ⟨thunk⟩)
    p(2) = true
    true || ⟨thunk⟩ → SHORT CIRCUIT! Return true immediately.
                      Thunk never evaluated.
                      Elements 3, 4, 5 never touched!
```

**Stack depth: O(k)** where k is position of first match.

### Worst Case: No Match (Full Traversal)

```
exists(_ == 99) on [1, 2, 3, 4, 5]

f(1, ⟨thunk⟩)           ← stack frame 1
  false || ⟨thunk⟩
  f(2, ⟨thunk⟩)         ← stack frame 2
    false || ⟨thunk⟩
    f(3, ⟨thunk⟩)       ← stack frame 3
      false || ⟨thunk⟩
      f(4, ⟨thunk⟩)     ← stack frame 4
        false || ⟨thunk⟩
        f(5, ⟨thunk⟩)   ← stack frame 5
          false || ⟨thunk⟩
          false         ← base case (Empty)
        false || false = false  ← unwind
      false || false = false
    false || false = false
  false || false = false
false || false = false
```

**Stack depth: O(n)** — same as strict foldRight when no short-circuit occurs.

### Key Insight

> **Laziness is about preserving the *option* to not do work.**
>
> - If you exercise the option (short-circuit): O(1) to O(k) stack
> - If you forfeit the option (no match): O(n) stack, same as strict

It's asymmetric—helps best case, doesn't hurt worst case.

---

## Common Patterns

### Operations via foldRight

```scala
// exists - short-circuits on first true
def exists(p: A => Boolean): Boolean =
  foldRight(false)((a, b) => p(a) || b)

// forall - short-circuits on first false
def forall(p: A => Boolean): Boolean =
  foldRight(true)((a, b) => p(a) && b)

// find - short-circuits when found
def find(p: A => Boolean): Option[A] =
  foldRight(None: Option[A])((a, b) => if p(a) then Some(a) else b)

// takeWhile - short-circuits when predicate fails
def takeWhile(p: A => Boolean): LazyList[A] =
  foldRight(empty)((a, b) => if p(a) then cons(a, b) else empty)
```

All of these benefit from lazy foldRight because they can terminate early.

### When Laziness Helps vs Doesn't Help

| Operation | Short-circuits? | Laziness helps? |
|-----------|-----------------|-----------------|
| `exists` (match found) | ✓ | ✓ Best case O(k) |
| `exists` (no match) | ✗ | ✗ Still O(n) |
| `forall` (early false) | ✓ | ✓ Best case O(k) |
| `forall` (all true) | ✗ | ✗ Still O(n) |
| `map` | ✗ | ✗ Must visit all |
| `filter` | ✗ | ✗ Must visit all |
| `find` (found) | ✓ | ✓ Best case O(k) |
| `takeWhile` | ✓ | ✓ Stops at first false |

### Why Infinite Lists Work

```scala
val ones: LazyList[Int] = LazyList.cons(1, ones)  // infinite!
ones.exists(_ == 1)  // returns true immediately!
ones.take(5).toList  // [1, 1, 1, 1, 1]
```

If foldRight were strict, `ones.exists(_ == 1)` would loop forever. But lazy evaluation means we only compute what we need—the moment we find a `1`, we stop.

---

## Summary

| Concept | Key Insight |
|---------|-------------|
| **foldLeft vs foldRight** | Named for parenthesis nesting, not traversal direction |
| **Tail recursion** | Work before recursing → O(1) stack |
| **Build-reverse pattern** | O(2n) time beats O(n) time with O(n) stack |
| **Lazy foldRight** | Thunks enable short-circuiting |
| **Short-circuit benefit** | Best case improves; worst case unchanged |
| **Infinite lists** | Only possible with lazy evaluation |
