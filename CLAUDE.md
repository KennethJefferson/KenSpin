# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains article spinning / spintax engines for generating text variations from templates like `[hello|hey] world`. Two implementations are available:

1. **KenSpin** - PowerShell Windows Forms GUI application (`Main.psf`)
2. **gospin** - Go CLI tool and library (`gospin/`)

## KenSpin (PowerShell GUI)

### Overview
Windows Forms-based article spinner with dark/light theme support and unlimited nested spintax. **Now powered by gospin binary** for 100% algorithm compatibility.

### Architecture
KenSpin shells out to `gospin.exe` for all spintax processing, ensuring identical behavior to the Go CLI tool. The PowerShell GUI handles:
- User interface (input/output textboxes, Generate button)
- Theme switching (dark/light mode)
- Calling gospin.exe with appropriate arguments
- Parallel batch processing for large generation counts

### Files
- `Main.psf` - PowerShell Studio project file (edit this directly)
- `Main.Export.ps1` - Exported PS1 for reference (auto-generated)
- `architecture.md` - Algorithm documentation and complexity analysis
- `gospin/gospin.exe` - **Required** Go binary (must be built first)

### Building & Running

**Step 1: Build gospin binary** (required)
```bash
cd gospin && go build -o gospin.exe ./cmd/gospin
```

**Step 2: Run KenSpin**
- Open `Main.psf` in PowerShell Studio 2025 and run, OR
- Execute the exported `Main.Export.ps1`

### Core Functions (in Main.psf)

**Spin-Text** - Process spintax via gospin.exe
```powershell
Spin-Text -Text "[hello|hey] [world|everyone]"
# Calls: gospin.exe "[hello|hey] [world|everyone]"
# Returns one of: "hello world", "hello everyone", "hey world", "hey everyone"
```

**Spin-TextParallel** - Generate multiple variations
```powershell
# Small batches (≤100): Uses gospin --times flag
Spin-TextParallel -Text "[a|b]" -Count 10
# Calls: gospin.exe "[a|b]" --times=10

# Large batches (>100): Spawns parallel gospin processes
Spin-TextParallel -Text "[a|b]" -Count 500 -MaxWorkers 4
# Spawns 4 parallel gospin.exe processes
```

### Why Shell to gospin?
- **100% compatibility** - Identical algorithm behavior guaranteed
- **Performance** - Go binary is faster than PowerShell interpretation
- **Maintainability** - Single source of truth for spintax logic
- **Proven** - gospin is battle-tested with comprehensive test suite

---

## gospin (Go CLI)

### Build Commands

```bash
# Build the CLI tool
cd gospin && go build -o gospin.exe ./cmd/gospin

# Run tests
cd gospin && go test ./...

# Run tests with coverage
cd gospin && go test -coverprofile=coverage.out ./...
```

### CLI Usage

```bash
# Basic usage
gospin "[hello|hey] friend"

# Generate multiple variations (outputs JSON array)
gospin "The [slow|quick] [fox|deer]" --times=5

# With variables (consistent values within a spin)
gospin "[$color:red|blue] dress with [$color] shoes"

# JSON-friendly: curly braces are preserved, brackets are spintax
gospin '{"color": "[$c:red|blue]", "text": "A [$c] item"}'

# Custom syntax characters (use curly braces for spintax if needed)
gospin "{a;b;c}" --start="{" --end="}" --delimiter=";"
```

### Architecture

**Core Library** (`gospin/gospin.go`):
- `Spinner` struct with configurable syntax characters (start/end/delimiter/escape)
- `Spin(str)` - returns single random variation
- `SpinN(str, n)` - returns n variations
- Recursive `walk()` function handles nested spintax parsing
- Supports UTF-8 text and escaped characters

**CLI** (`gospin/cmd/gospin/main.go`):
- Uses Cobra for CLI flags
- Single output for `--times=1`, JSON array for multiple

---

## Spintax Format

Default syntax uses `[option1|option2]` (bracket notation) with support for:
- Nested options: `[a|[b|c]]`
- Optional phrases: `[word|]` (empty option)
- Variables: `[$name:opt1|opt2]` defines, `[$name]` references
- Escaped characters: `\[literal\]`
- Custom delimiters via `--start`, `--end`, `--delimiter` flags

**Why brackets?** JSON uses curly braces `{}` for objects. Using brackets `[]` for spintax allows processing JSON templates without escaping every brace.

## Error Handling & Validation

gospin validates input before processing and provides helpful error messages:

```bash
# Mismatched bracket (typed } instead of ])
gospin '[$color:red|blue}'
# Error: variable 'color' has mismatched brackets - opened with '[' but found '}' (near: [$color:red|blue})

# Unclosed bracket
gospin '[$color:red|blue'
# Error: unclosed bracket '[' at position 0 (near: [$color:red|blue (variable: $color))

# Undefined variable
gospin 'Hello [$undefined]'
# Error: undefined variable: undefined
```

## Development Notes

- Platform: Windows
- PowerShell: 5.1+ or PowerShell 7+
- Go: Modern Go with modules support
- UI Framework: Windows Forms (System.Windows.Forms)
