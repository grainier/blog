---
layout: post
title: "tryparse: Parsing LLM Output in Rust Without Losing Your Mind"
date: 2025-10-27
categories: rust llm parsing
---

## The Problem

If you've worked with LLMs in production, you know the pain. You ask Claude or GPT to return JSON, and you get... creative interpretations.

Sometimes it's wrapped in markdown:

```
Sure! Here's your data:
\```json
{"name": "Alice", "age": 30}
\```
Hope that helps!
```

Sometimes the JSON is broken:

```json
{
  name: "Alice",
  age: "30",
}
```

Sometimes it's just text with some JSON-like structure embedded in prose:

```
The user Alice is 30 years old, here's the data: {name: 'Alice', age: 30}
```

Your app expects clean, valid JSON. It gets this mess. `serde_json::from_str()` throws an error. Your carefully designed type-safe parsing fails. Now you're writing regex hacks, string replacements, and special cases. Every LLM provider is slightly different. Every model has its own quirks.

I got tired of writing the same defensive parsing code in every project. So I built `tryparse`.

## Why Rust?

LLMs are everywhere now. Rust applications need to integrate with them. But Rust's type system and error handling expect predictable, well-structured data. LLMs don't provide that.

Other languages have flexible JSON parsers and type coercion built into the runtime. Python's `json.loads()` is forgiving. JavaScript doesn't care much about types. But Rust? `serde_json::from_str()` expects valid JSON and exact type matches. That's normally a feature - it's why we use Rust. But when dealing with LLM output, it becomes a problem.

Every Rust project I worked on that integrated with LLMs had the same boilerplate: try parsing, catch errors, strip markdown, fix common JSON issues, try again. Repeat across projects.

## Why tryparse?

I looked at existing solutions. [BAML](https://github.com/BoundaryML/baml) has great ideas around schema-aligned parsing, but it's a DSL that generates code for multiple languages. For a Rust-only project, that felt like overkill. I wanted something that integrated directly into my Rust codebase - no build step, no code generation, just a library I could import and use.

The goal was simple: make LLM output parsing work like it does in Python or JavaScript, but keep Rust's type safety. Use `serde` when possible. Let developers write normal Rust types. Handle the messy parts transparently.

## What is tryparse?

tryparse is a Rust library that parses messy, real-world data from LLMs. It's not a DSL or a code generator. It's a regular Rust crate you add to your `Cargo.toml` and use directly in your code.

The core idea: instead of one parsing strategy, try multiple strategies in parallel, score the results, and pick the best one. Handle common errors automatically. Apply type coercion when safe. Give you control when you need it.

## How It Works

### Multi-Strategy Parsing Pipeline

When you call `parse()`, tryparse runs your input through a pipeline:

**1. Pre-processing**: Clean up the input - remove BOM markers, zero-width characters, fix excessive nesting, normalize backslashes.

**2. Strategy Execution**: Run multiple parsing strategies in parallel. Each strategy knows how to extract structured data from different formats:

- **DirectJson** (priority 1): Try `serde_json::from_str()` first. If your input is valid JSON, this is the fastest path.
- **Markdown** (priority 2): Look for markdown code blocks with ` ```json ` or ` ```yaml ` tags. Extract the content and score by relevance (keywords like "json", position, block size).
- **YAML** (priority 15): Parse as YAML and convert to JSON. Useful because YAML is more lenient (unquoted strings work).
- **JsonFixer** (priority 20): Apply a series of fixes for common JSON syntax errors - trailing commas, unquoted keys, single quotes, missing commas, unclosed braces, comments, smart quotes, template literals.
- **Heuristic** (priority 30): Pattern-based extraction from prose. Looks for JSON-like structures in plain text. Last resort.

Each strategy produces a `FlexValue` - a JSON value with metadata about where it came from and what transformations were applied.

**3. Scoring and Ranking**: Every candidate gets a score based on:
- Base score from the source (Direct=0, Markdown=10, YAML=15, Fixed=20+, Heuristic=50)
- Transformation penalties (string→number coercion adds +2, field renames add +4, etc.)
- Confidence adjustment (each transformation reduces confidence by 5%)

Lower scores are better. Direct JSON with no transformations scores 0 - the ideal case.

**4. Deserialization**: Try to deserialize candidates in order (best score first) into your target type. Apply type coercion during deserialization if needed. Return the first successful result.

### Tracking Transformations

Every change is logged in the `FlexValue`. This immutable transformation log tells you exactly what happened:

```rust
pub struct FlexValue {
    pub value: Value,               // The JSON value
    pub source: Source,             // Where it came from
    transformations: Vec<Transformation>,  // What changed
    confidence: f32,                // How confident we are
}
```

You can inspect the transformations to understand what the parser did:

```rust
use tryparse::{parse_with_candidates, scoring::score_candidate};

let (result, candidates) = parse_with_candidates::<User>(messy_input).unwrap();

for candidate in &candidates {
    println!("Source: {:?}, Score: {}",
        candidate.source(),
        score_candidate(candidate)
    );
    for t in candidate.transformations() {
        println!("  - {:?}", t);
    }
}
```

### Type Coercion

Type coercion happens during deserialization and works with both `serde::Deserialize` and `LlmDeserialize`:

- `"42"` → `42` (string to number)
- `42` → `"42"` (number to string)
- `"true"` → `true` (string to bool)
- `42.0` → `42` (float to int, if no precision loss)
- `"item"` → `["item"]` (single value to array)

All coercions are tracked as `Transformation` records.

## Key Features

### Works with Standard Serde

Basic usage requires only `serde::Deserialize`:

```rust
use tryparse::parse;
use serde::Deserialize;

#[derive(Deserialize)]
struct User {
    name: String,
    age: u32,
}

let messy = r#"
Here's your data:
\```json
{
  name: "Alice",
  age: "30",
}
\```
"#;

let user: User = parse(messy).unwrap();
// User { name: "Alice", age: 30 }
```

Type coercion (`"30"` → `30`) works automatically. Markdown wrapper is stripped. Trailing comma is fixed.

### Advanced Features with Derive

Add the `derive` feature for fuzzy matching and union types:

```toml
[dependencies]
tryparse = { version = "0.3", features = ["derive"] }
tryparse-derive = "0.3"
```

**Fuzzy Field Matching**: Handles different naming conventions automatically.

```rust
use tryparse::parse_llm;
use tryparse_derive::LlmDeserialize;

#[derive(LlmDeserialize)]
struct Config {
    user_name: String,  // Matches: userName, UserName, user-name, USER_NAME
    max_count: i64,     // Matches: maxCount, MaxCount, max-count, MAX_COUNT
}

let data: Config = parse_llm(r#"{"userName": "Alice", "maxCount": 30}"#).unwrap();
```

Field matching normalizes both struct field names and JSON keys to snake_case, then compares case-insensitively. Removes separators like `-`, `.`, and spaces.

**Enum Fuzzy Matching**: Case-insensitive, handles different separators.

```rust
#[derive(LlmDeserialize)]
enum Status {
    InProgress,  // Matches: "in_progress", "in-progress", "inprogress", "IN PROGRESS"
    Completed,   // Matches: "complete", "COMPLETED", "done"
    Cancelled,
}
```

**Union Types**: Automatically picks the best-fitting variant.

```rust
#[derive(LlmDeserialize)]
#[llm(union)]
enum Value {
    Number(i64),
    Text(String),
    List(Vec<String>),
}

// The parser tries each variant, picks the one with lowest transformation penalty
let v1: Value = parse_llm("42").unwrap();        // Number(42)
let v2: Value = parse_llm(r#""hello""#).unwrap(); // Text("hello")
let v3: Value = parse_llm(r#"["a", "b"]"#).unwrap(); // List(...)
```

**Implied Keys**: Single-field structs can unwrap direct values.

```rust
#[derive(LlmDeserialize)]
struct Wrapper {
    data: String,
}

// Direct string wraps into the field automatically
let w: Wrapper = parse_llm(r#""hello world""#).unwrap();
assert_eq!(w.data, "hello world");
```

### JSON Fixes Applied Automatically

The `JsonFixer` strategy handles:

- Trailing commas: `{"a": 1,}` → `{"a": 1}`
- Unquoted keys: `{name: "x"}` → `{"name": "x"}`
- Single quotes: `{'a': 1}` → `{"a": 1}`
- Missing commas: `{"a":1 "b":2}` → `{"a":1,"b":2}`
- Unclosed braces: `{"a": 1` → `{"a": 1}`
- Comments: `{"a": 1 /* test */}` → `{"a": 1}`
- Smart quotes: `{"a": "value"}` → `{"a": "value"}`
- Double-escaped JSON: `"{\"a\":1}"` → `{"a":1}`
- Template literals: `` {`key`: "value"} `` → `{"key": "value"}`
- Hex numbers: `{"a": 0xFF}` → `{"a": 255}`

### Inspectable Parse Results

Get all candidates and their scores:

```rust
let (result, candidates) = parse_with_candidates::<User>(input).unwrap();

println!("Best result: {:?}", result);
for candidate in &candidates {
    println!("Source: {:?}, Score: {}",
        candidate.source(),
        score_candidate(candidate)
    );
}
```

## How to Use It

### Basic Usage (serde::Deserialize)

```rust
use tryparse::parse;
use serde::Deserialize;

#[derive(Deserialize)]
struct Data {
    count: i64,      // "42" → 42
    price: f64,      // "3.14" → 3.14
    active: bool,    // "true" → true
    tags: Vec<String>, // "tag" → ["tag"]
}

let data: Data = parse(r#"{"count": "42", "price": "3.14", "active": "true", "tags": "tag"}"#).unwrap();
```

Type coercion works automatically with standard `Deserialize`.

### Advanced Usage (LlmDeserialize)

```rust
use tryparse::parse_llm;
use tryparse_derive::LlmDeserialize;

#[derive(LlmDeserialize)]
struct User {
    user_name: String,
    age: u32,
    email: Option<String>,
}

// Handles camelCase, trailing commas, markdown wrappers
let messy = r#"
Sure! Here's the user:
\```json
{
  userName: "Alice",
  age: "30",
  email: "alice@example.com",
}
\```
"#;

let user: User = parse_llm(messy).unwrap();
```

### Custom Parser Configuration

Build your own parser with specific strategies:

```rust
use tryparse::parser::{FlexibleParser, strategies::*};

let parser = FlexibleParser::new()
    .with_strategy(DirectJsonStrategy)
    .with_strategy(MarkdownStrategy::new())
    .with_strategy(JsonFixerStrategy::new());

let data: User = parse_with_parser(input, &parser).unwrap();
```

## When to Use It

**Good for**:
- Parsing LLM responses in Rust applications
- Handling inconsistent output formats
- Small to medium inputs (<1MB)
- Applications where you control the parsing code
- Prototyping and experimentation

**Not ideal for**:
- Streaming LLM responses (requires complete input)
- High-frequency parsing loops (has overhead from multiple strategies)
- Multi-language projects (Rust only)
- Cases where you can enforce strict JSON output from the LLM

## What's Next

Some areas I'm thinking about for future versions:

**Async support**: Right now parsing is synchronous. Adding async would make it easier to integrate with async Rust applications without blocking.

**Streaming**: Being able to parse incomplete JSON as it arrives would help with real-time LLM streaming responses. This is tricky because you need to handle partial structures, but would be useful.

**Better acronym handling**: The current field name normalization turns `XMLParser` into `x_m_l_parser` instead of `xml_parser`. Smarter handling of common acronyms would improve fuzzy matching.

**Configurable scoring**: Right now the scoring weights are hardcoded. Letting users tune them based on their use case might make sense.

**More strategies**: There are probably other common patterns in LLM output I haven't seen yet. If you find JSON-like formats that don't parse, open an issue.

## Try It

```toml
[dependencies]
tryparse = "0.3"
serde = { version = "1.0", features = ["derive"] }

# For advanced features:
# tryparse = { version = "0.3", features = ["derive"] }
# tryparse-derive = "0.3"
```

The library is Apache-2.0 licensed and available on [crates.io](https://crates.io/crates/tryparse). Source code is on [GitHub](https://github.com/microagents/tryparse).

If you're building Rust applications that consume LLM output, give it a try. File issues if you find broken JSON formats it doesn't handle. Contributions welcome.
