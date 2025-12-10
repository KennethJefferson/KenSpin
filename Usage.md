# gospin Usage Guide

Complete guide to using gospin for spintax processing and text variation generation.

## Quick Start

```bash
# Build the binary
cd gospin && go build -o gospin.exe ./cmd/gospin

# Basic usage
gospin "[hello|hey] world"
# Output: hey world
```

## Syntax Reference

### Basic Spintax

Use brackets `[` and `]` with pipe `|` delimiters:

```
[option1|option2|option3]
```

**Examples:**
```bash
gospin "[red|green|blue] car"        # Output: green car
gospin "[Hello|Hey|Hi] there"        # Output: Hi there
```

### Nested Spintax

Brackets can be nested to any depth:

```bash
gospin "[a|[b|c]]"                   # Output: b (or a, or c)
gospin "[I [love|like]|[He|She] [loves|likes]] cats"
# Output: She likes cats
```

### Optional Content

Use an empty option for optional text:

```bash
gospin "The [very |]quick fox"
# Output: "The quick fox" or "The very quick fox"

gospin "[Hello|] world"
# Output: "world" or "Hello world"
```

### Variables

Variables ensure consistent values across multiple uses:

**Definition:** `[$name:option1|option2]`
**Reference:** `[$name]`

```bash
# Same color used twice
gospin "[$color:red|blue] dress with [$color] shoes"
# Output: blue dress with blue shoes (always matching)

# Multiple variables
gospin "[$size:small|large] [$color:red|blue] [$size] box is [$color]"
# Output: large blue large box is blue
```

**Variable Rules:**
- Names must be alphanumeric + underscore (e.g., `color`, `my_var`, `item1`)
- Names cannot start with a digit
- Variables are case-sensitive (`$Color` != `$color`)
- Referencing undefined variables produces an error

### Nested Variables

Variables can be defined with nested options:

```bash
gospin "[$speed:quick|[$emphasis:very |super ]fast] fox is [$speed]"
# Possible outputs:
#   "quick fox is quick"
#   "very fast fox is very fast"
#   "super fast fox is super fast"
```

## JSON Template Processing

gospin uses bracket notation specifically to preserve JSON curly braces:

```bash
# Curly braces are preserved, brackets are processed
gospin '{"color": "[$c:red|blue]", "item": "a [$c] dress"}'
# Output: {"color": "blue", "item": "a blue dress"}

# Complex JSON template
gospin '{
  "prompt": "A [$style:hyper|ultra]-realistic shot",
  "location": "[$loc:beach|office|studio]",
  "lighting": "[$light:soft|dramatic] [$light] shadows"
}'
# Output (formatted):
# {
#   "prompt": "A hyper-realistic shot",
#   "location": "studio",
#   "lighting": "dramatic dramatic shadows"
# }
```

**Note:** If your JSON contains arrays `[...]`, escape them with `\[...\]` to prevent spintax processing.

## CLI Options

```
gospin [text] [flags]

Flags:
  --times int          Generate multiple variations (default 1)
  --start string       Start character (default "[")
  --end string         End character (default "]")
  --delimiter string   Option delimiter (default "|")
  --escape string      Escape character (default "\\")
  -h, --help           Help
```

### Generate Multiple Variations

```bash
gospin "[a|b|c]" --times=5
# Output (JSON array): ["b","a","c","b","a"]
```

### Custom Delimiters

If you prefer curly brace syntax:

```bash
gospin "{a;b;c}" --start="{" --end="}" --delimiter=";"
# Output: b
```

## Escaping

Use backslash to include literal brackets or pipes:

```bash
gospin "Price: \[TBD\] for [red|blue] item"
# Output: Price: [TBD] for blue item

gospin "[yes\|no|maybe]"  # "yes|no" is one option
# Output: "yes|no" or "maybe"
```

## Error Handling

gospin validates input and provides helpful error messages:

### Mismatched Brackets

```bash
gospin "[$color:red|blue}"
# Error: variable 'color' has mismatched brackets - opened with '[' but found '}'
#        (near: [$color:red|blue})
```

**Fix:** Change `}` to `]`

### Unclosed Brackets

```bash
gospin "[$color:red|blue"
# Error: unclosed bracket '[' at position 0 (near: [$color:red|blue (variable: $color))
```

**Fix:** Add closing `]`

### Undefined Variables

```bash
gospin "Hello [$undefined]"
# Error: undefined variable: undefined
```

**Fix:** Define the variable first: `[$undefined:value1|value2] [$undefined]`

### Invalid Variable Names

```bash
gospin "[$123bad:a|b]"
# Error: invalid variable name: 123bad (must be alphanumeric/underscore, cannot start with digit)
```

**Fix:** Use valid name like `$var123` or `$my_var`

## Library Usage (Go)

```go
import "github.com/m1/gospin"

func main() {
    spinner := gospin.New(nil)

    // Basic spin
    result, err := spinner.Spin("[hello|hey] world")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(result) // hey world

    // Multiple variations
    results, err := spinner.SpinN("[a|b|c]", 10)
    // results = ["b", "a", "c", ...]

    // Variables
    result, _ = spinner.Spin("[$x:1|2] and [$x]")
    // result = "2 and 2" (consistent)
}
```

## Best Practices

1. **Use variables for consistency** - When the same choice should appear multiple times
2. **Test with `--times`** - Verify all branches are reachable
3. **Escape JSON arrays** - Use `\[...\]` for literal arrays in JSON
4. **Validate complex templates** - gospin will catch syntax errors early
5. **Keep nesting reasonable** - Deep nesting works but is harder to read

## Common Patterns

### Consistent Attribute References

```
[$material:cotton|silk|wool] [$color:red|blue] [$material] sweater in [$color]
```

### Optional Modifiers

```
[The |][very |]quick [brown |]fox
```

### Conditional Phrases

```
[I [love|like] [cats|dogs]|[He|She] [loves|likes] [cats|dogs]]
```

### JSON Prompt Templates

```json
{
  "style": "[$s:photorealistic|cinematic|artistic]",
  "subject": "a [$age:young|elderly] [$gender:man|woman]",
  "setting": "in a [$loc:forest|city|beach] at [$time:dawn|dusk|night]"
}
```
