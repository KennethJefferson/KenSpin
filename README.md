# Article Spinner

A spintax/article spinning toolkit for generating text variations from templates. Includes both a PowerShell GUI application and a Go CLI tool.

## What is Spintax?

Spintax (spin syntax) allows you to create multiple variations of text from a single template:

```
{Hello|Hey|Hi} {world|everyone|there}!
```

This template can generate 9 unique combinations like "Hello world!", "Hey everyone!", "Hi there!", etc.

### Features

- **Unlimited nesting**: `{a|{b|{c|d}}}` resolves correctly at any depth
- **Empty options**: `{word|}` can output "word" or nothing
- **Multiple groups**: `{a|b} {c|d}` processes each group independently
- **Escape support**: Use `\{` and `\|` for literal characters

## KenSpin (PowerShell GUI)

Windows Forms application with dark/light theme support.

### Screenshot

```
┌─────────────────────────────────────────┐
│ KenSpin                          [─][□][×]│
├─────────────────────────────────────────┤
│ Theme ▼                                  │
├─────────────────────────────────────────┤
│ ┌─────────────────────────────────────┐ │
│ │ {The|A} {quick|slow} {fox|dog}      │ │
│ │ {jumps|leaps} over the {lazy|}cat   │ │
│ └─────────────────────────────────────┘ │
│                                         │
│            [ Generate ]                 │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ A slow fox leaps over the cat       │ │
│ │                                     │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Requirements

- Windows 10/11
- PowerShell 5.1+ or PowerShell 7+
- [PowerShell Studio 2025](https://www.sapien.com/software/powershell_studio) (for editing .psf files)

### Running

**Option 1**: Open `Main.psf` in PowerShell Studio and click Run

**Option 2**: Execute the exported script:
```powershell
.\Main.Export.ps1
```

### Usage

1. Enter spintax text in the top textbox
2. Click **Generate**
3. View the randomly generated result in the bottom textbox
4. Click Generate again for a different variation

## gospin (Go CLI)

Command-line spintax processor written in Go.

### Building

```bash
cd gospin
go build -o gospin.exe ./cmd/gospin
```

### Usage

```bash
# Basic usage
./gospin "{hello|hey} friend"
# Output: hey friend

# Generate multiple variations
./gospin "The {slow|quick} {fox|deer}" --times=5
# Output: ["The quick fox", "The slow deer", ...]

# Custom syntax characters
./gospin "[a;b;c]" --start="[" --end="]" --delimiter=";"
# Output: b
```

### Running Tests

```bash
cd gospin
go test ./...
```

## Algorithm

Both implementations use a two-phase approach:

1. **Validation**: Stack-based brace matching (O(n))
2. **Resolution**: Multi-pass innermost-first regex replacement (O(m × d))

Where m = string length and d = maximum nesting depth.

See [architecture.md](architecture.md) for detailed algorithm documentation including complexity analysis and theoretical foundations.

## Project Structure

```
ArticleSpinner/
├── Main.psf              # PowerShell Studio project (GUI)
├── Main.Export.ps1       # Exported PowerShell script
├── architecture.md       # Algorithm documentation
├── README.md             # This file
├── CLAUDE.md             # Claude Code guidance
├── Mynotes.md            # Kevin's CFG/concurrency discussion notes
└── gospin/               # Go implementation
    ├── gospin.go         # Core library
    ├── gospin_test.go    # Tests
    └── cmd/gospin/       # CLI entry point
        └── main.go
```

## Examples

### Simple Selection
```
Input:  {red|green|blue} car
Output: green car
```

### Nested Spintax
```
Input:  {I {love|like}|{He|She} {loves|likes}} {cats|dogs}
Output: She likes cats
```

### Optional Content
```
Input:  The {very |}quick fox
Output: The quick fox
   or:  The very quick fox
```

### Complex Template
```
Input:  {Dear|Hello} {friend|colleague},

{I hope this {message|email} finds you well.|Greetings!}

{Best|Kind} regards,
{John|Jane}

Output: Hello colleague,

Greetings!

Kind regards,
Jane
```

## License

MIT
