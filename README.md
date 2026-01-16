# FP Visualizations

Interactive browser-based visualizations for understanding functional programming concepts.

Built while working through [Functional Programming in Scala (2nd Edition)](https://www.manning.com/books/functional-programming-in-scala-second-edition).

## Visualizations

### 1. `lazylist-viz.html` - LazyList toList
Compare recursive vs tail-recursive implementations of converting a LazyList to a List.

- See how the recursive version builds up stack frames
- Watch the tail-recursive version accumulate in reverse, then reverse at the end
- Understand why one risks stack overflow and the other doesn't

### 2. `foldright-viz.html` - foldRight
Visualize how foldRight works, comparing naive recursive vs tail-recursive (via reverse + foldLeft).

- Switch between addition, subtraction, and string concatenation
- See the expression tree build up
- Understand why "parentheses nest to the right"

### 3. `lazy-foldright-viz.html` - Lazy foldRight & Early Termination
See how lazy evaluation enables short-circuiting in operations like `exists`.

- Watch thunks (suspended computations) get created
- See how `||` short-circuits to skip evaluating the rest of the list
- Understand by-name parameters (`=> B`)

## Usage

Open any HTML file directly in your browser:

```bash
open lazylist-viz.html
```

Or serve locally:

```bash
python3 -m http.server 8000
# Then visit http://localhost:8000
```

## Key Concepts Illustrated

- **Tail recursion** - How to avoid stack overflow with accumulators
- **foldLeft vs foldRight** - Associativity and evaluation order
- **Laziness** - By-name parameters and thunks
- **Short-circuit evaluation** - Early termination via lazy foldRight

## License

MIT
