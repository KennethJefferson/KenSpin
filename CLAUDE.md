# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains article spinning / spintax engines for generating text variations from templates like `{hello|hey} world`. Two implementations are available:

1. **KenSpin** - PowerShell Windows Forms GUI application (`Main.psf`)
2. **gospin** - Go CLI tool and library (`gospin/`)

## KenSpin (PowerShell GUI)

### Overview
Windows Forms-based article spinner with dark/light theme support and unlimited nested spintax.

### Files
- `Main.psf` - PowerShell Studio project file (edit this directly)
- `Main.Export.ps1` - Exported PS1 for reference (auto-generated)
- `architecture.md` - Algorithm documentation and complexity analysis

### Running
Open `Main.psf` in PowerShell Studio 2025 and run, or execute the exported `Main.Export.ps1`.

### Core Functions (in Main.psf)

**Test-SpintaxBalance** - Stack-based brace validation
```powershell
Test-SpintaxBalance -Text "{hello|world}"  # Returns $true
Test-SpintaxBalance -Text "{hello|world"   # Returns $false (unbalanced)
```

**Spin-Text** - Process spintax and return random variation
```powershell
Spin-Text -Text "{hello|hey} {world|everyone}"
# Returns one of: "hello world", "hello everyone", "hey world", "hey everyone"
```

**Spin-TextParallel** - Generate multiple variations concurrently
```powershell
Spin-TextParallel -Text "{a|b}" -Count 10 -MaxWorkers 4
# Returns array of 10 variations using 4 parallel workers
```

### Algorithm
- **Phase 1**: Stack-based validation (O(n) scan)
- **Phase 2**: Multi-pass innermost-first regex resolution
- **Complexity**: O(m × d) time, O(m + d) space (m=length, d=nesting depth)

See `architecture.md` for detailed algorithm explanation and Kevin's CFG/concurrency theory.

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
gospin "{hello|hey} friend"

# Generate multiple variations (outputs JSON array)
gospin "The {slow|quick} {fox|deer}" --times=5

# Custom syntax characters
gospin "[a;b;c]" --start="[" --end="]" --delimiter=";"
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

Default syntax uses `{option1|option2}` with support for:
- Nested options: `{a|{b|c}}`
- Optional phrases: `{word|}` (empty option)
- Escaped characters: `\{literal\}`
- Custom delimiters via Config (gospin only)

## Development Notes

- Platform: Windows
- PowerShell: 5.1+ or PowerShell 7+
- Go: Modern Go with modules support
- UI Framework: Windows Forms (System.Windows.Forms)
