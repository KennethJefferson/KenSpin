# Article Spinner Architecture

## Overview

This document describes the algorithm design, complexity analysis, and theoretical foundations of the PowerShell-based article spinner implementation. The design is informed by both the reference Go implementation (`gospin`) and Kevin Burleigh's discussion on parsing, data structures, and concurrency theory.

## What is Spintax?

Spintax (spin syntax) is a text format that allows generating multiple variations of content from a single template. Options are enclosed in curly braces and separated by pipes:

```
{hello|hey|hi} {world|everyone}
```

This template can produce 6 unique combinations:
- "hello world", "hello everyone"
- "hey world", "hey everyone"
- "hi world", "hi everyone"

### Nested Spintax

The real power (and complexity) comes from **nested** structures:

```
{The {quick|slow} {fox|dog}|A {lazy|sleepy} cat}
```

This creates hierarchical options where inner braces must be resolved before outer ones.

## Algorithm Design

### Why Regex Alone Fails

As Kevin explained in his discussion (timestamp ~14:30), this is a **Context-Free Grammar (CFG)** problem. Regular expressions cannot handle arbitrary nesting because they lack memory—they can't "remember" how many opening braces they've seen to match closing ones.

> "Regular Expressions generally cannot handle arbitrary nesting (infinite recursion) because they lack memory (a stack)." — Kevin's CFG discussion

### The Stack-Based Solution

The algorithm uses a two-phase approach:

#### Phase 1: Syntax Validation (Stack-Based)

Before parsing, we validate that braces are balanced using a simple counter (equivalent to a stack for this purpose):

```
Input: "{a|{b|c}}"

Scan: { → stack=1
       a → stack=1
       | → stack=1
       { → stack=2
       b → stack=2
       | → stack=2
       c → stack=2
       } → stack=1
       } → stack=0

Result: Balanced (stack ends at 0, never went negative)
```

This O(n) pass catches malformed input before the more expensive parsing phase.

#### Phase 2: Multi-Pass Innermost-First Resolution

Rather than building an explicit tree structure, the algorithm uses a **multi-pass regex replacement** strategy that resolves innermost braces first:

```
Input: "{a|{b|c}}"

Pass 1: Find pattern \{([^{}]+)\} → matches "{b|c}"
        Replace with random selection → "b"
        Result: "{a|b}"

Pass 2: Find pattern \{([^{}]+)\} → matches "{a|b}"
        Replace with random selection → "a"
        Result: "a"

No more braces → Done
```

The key insight is that the regex `\{([^{}]+)\}` only matches **innermost** braces (those containing no nested braces), naturally implementing depth-first, innermost-first traversal.

### Comparison with gospin Implementation

The Go implementation in `gospin/gospin.go` uses a similar approach:

```go
func (s *Spinner) walk(str string, level int) string {
    // Find matching braces at current level
    // Recursively process inner content
    // Select random option
}
```

Both implementations share:
- Innermost-first resolution strategy
- In-place string modification (no AST construction)
- Multi-pass iteration until no braces remain

The PowerShell version simplifies this by leveraging regex's built-in matching, while gospin manually tracks brace positions.

## Complexity Analysis

### Time Complexity: O(m × d)

Where:
- **m** = length of the input string
- **d** = maximum nesting depth

Each pass of the regex replacement scans the entire string once, and we need at most **d** passes to resolve all nesting levels.

**Example:**
```
{1|{2|{3|{4|5}}}}  →  depth = 4
Pass 1: Resolve {4|5} → "4"
Pass 2: Resolve {3|4} → "3"
Pass 3: Resolve {2|3} → "2"
Pass 4: Resolve {1|2} → "1"
```

### Space Complexity: O(m + d)

- **O(m)**: String buffer for the result (strings are immutable, so each replacement creates a new string)
- **O(d)**: Implicit recursion depth in regex engine (though PowerShell's regex doesn't recurse for this pattern)

### Comparison with Tree-Based Approach

An alternative is to first parse into an explicit AST (Abstract Syntax Tree) then walk it:

| Approach | Time | Space | Tradeoff |
|----------|------|-------|----------|
| Multi-pass regex | O(m × d) | O(m) | Simple, no tree overhead |
| AST + walk | O(m) + O(n) | O(n) | Single parse, tree storage |

Where n = number of nodes in the tree.

For typical spintax with moderate nesting (d < 10), the multi-pass approach is simpler and performs well.

## Concurrency Design

### The Fork Bomb Problem

Kevin's discussion (~25:00) highlights the danger of naively spawning threads for each branch:

> "If every branch of the tree spawns a new thread/goroutine, you risk running out of memory... Exponential growth of threads leads to a crash." — Kevin on Fork Bombing

### Worker Pool Solution

The `Spin-TextParallel` function implements Kevin's recommended approach:

1. **Bounded Worker Pool**: Fixed number of workers (defaults to CPU count)
2. **Job Queue**: Tasks submitted to pool, not spawned immediately
3. **DFS-Style Collection**: Results collected sequentially to prevent memory explosion

```powershell
$pool = [runspacefactory]::CreateRunspacePool(1, $MaxWorkers)
```

### When to Use Parallelism

Parallelism is beneficial when:
- Generating many variations (Count > 1)
- Each variation is independent (no shared state)
- The overhead of pool creation is amortized across many jobs

For single generations, sequential processing is faster due to pool overhead.

## Kevin's Key Insights Applied

### 1. Stack for Brace Matching (02:30)

> "Encounter Open Brace `{`: Push to Stack. Encounter Close Brace `}`: Pop from Stack."

Implemented in `Test-SpintaxBalance` using a counter (simplified stack).

### 2. Two Operations: Selection vs Concatenation (08:00)

> "Random Selection (OR logic): `{fox|bear|cat}` → Pick one. Concatenation (AND logic): `fox jumps` → Use both."

The algorithm handles this implicitly:
- `{a|b}` → OR operation (select one)
- `{a}{b}` → AND operation (both appear, each selected independently)

### 3. DFS vs BFS Scheduling (25:00)

> "DFS: Process A, then A1, then A1's children... Memory usage scales with the **height of the tree**."

The multi-pass regex approach naturally implements DFS—innermost braces (deepest in tree) are resolved first.

### 4. Priority Queue for Concurrency (42:00)

> "Use a Priority Queue to enforce DFS logic. Prioritize tasks that are deeper in the tree."

While the current implementation doesn't use an explicit priority queue, the sequential result collection (`EndInvoke` in order) achieves similar memory-bounded behavior.

## Function Reference

### Test-SpintaxBalance

```powershell
Test-SpintaxBalance -Text "{hello|world}"  # Returns $true
Test-SpintaxBalance -Text "{hello|world"   # Returns $false
```

### Select-RandomOption

```powershell
Select-RandomOption -Options "a|b|c"  # Returns "a", "b", or "c"
Select-RandomOption -Options "word|"  # Returns "word" or ""
```

### Spin-Text

```powershell
Spin-Text -Text "{hello|hey} {world|everyone}"
# Returns one of: "hello world", "hello everyone", "hey world", "hey everyone"
```

### Spin-TextParallel

```powershell
Spin-TextParallel -Text "{a|b}" -Count 10 -MaxWorkers 4
# Returns array of 10 variations, generated using 4 parallel workers
```

## Testing

Test cases covering edge conditions:

| Input | Expected Output | Test |
|-------|----------------|------|
| `{hello\|hey} world` | "hello world" or "hey world" | Simple selection |
| `{a\|{b\|c}}` | "a", "b", or "c" | Nested braces |
| `{1\|{2\|{3\|{4\|5}}}}` | Any of "1" through "5" | Deep nesting |
| `{a\|b} {c\|d}` | 4 combinations | Multiple groups |
| `{word\|}` | "word" or "" | Empty option |
| `plain text` | "plain text" | No spintax |
| `{a\|b` | Error | Unbalanced braces |

## Future Enhancements

1. **Escape Character Support**: Allow `\{` and `\|` for literal braces/pipes
2. **Weighted Selection**: Syntax like `{common:3|rare:1}` for probability weighting
3. **Named References**: `{$adjective}` to reference previously defined choices
4. **Performance Profiling**: Benchmark against large inputs with deep nesting

## References

- Kevin Burleigh's CFG/Concurrency Discussion (Mynotes.md)
- gospin Go Implementation (gospin/gospin.go)
- PowerShell Runspace Documentation
