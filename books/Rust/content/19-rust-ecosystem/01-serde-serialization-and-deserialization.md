---
title: Serde Serialization and Deserialization
---

# Serde Serialization and Deserialization

`serde` is the backbone of the Rust data-serialization ecosystem. Rather than binding your types to JSON or YAML directly, `serde` splits the problem into two decoupled traits—`Serialize` and `Deserialize`—and generates zero-reflection, compile-time monomorphized code via procedural derive macros. That means one `#[derive(Serialize, Deserialize)]` struct works seamlessly with JSON (`serde_json`), TOML (`toml`), YAML (`serde_yaml`), MessagePack (`rmp-serde`), CBOR, and Postcard.

## Zero-Copy Deserialization and Tagged Enums

When parsing large JSON payloads in hot paths, borrowing strings directly from the input buffer with `#[serde(borrow)]` avoids heap-allocating a `String` for every field:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
enum Event<'a> {
    UserSignedIn {
        #[serde(borrow)]
        user_id: &'a str,
        timestamp: u64,
    },
    JobCompleted {
        job_id: u64,
        #[serde(default)]
        retries: u8,
    },
}
```

### High-Value Serde Attributes

| Attribute | Purpose |
| :--- | :--- |
| `#[serde(rename_all = "camelCase")]` | Maps Rust `snake_case` fields to JSON `camelCase` automatically |
| `#[serde(default)]` | Falls back to `Default::default()` when an optional field is missing |
| `#[serde(deny_unknown_fields)]` | Rejects typos in configuration files instead of silently ignoring them |
| `#[serde(tag = "type")]` | Internally tagged enum representation matching idiomatic REST/JSON APIs |
| `#[serde(skip_serializing_if = "Option::is_none")]` | Omits `null` keys from serialized output |

> [!TIP]
> Always add `#[serde(deny_unknown_fields)]` on user-edited configuration structs (TOML/YAML). Without it, a typo like `max_conections = 50` silently falls back to the default value and causes hours of debugging.

## Hands-On Lab: Round-Trip a Tagged Enum with `serde_json`

In a scratch Cargo project (`cargo new /tmp/serde-lab && cd /tmp/serde-lab && cargo add serde -F derive && cargo add serde_json`), replace `src/main.rs` and run `cargo run`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
enum Metric<'a> {
    Counter { name: &'a str, value: u64 },
    Gauge { name: &'a str, value: f64 },
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let raw = r#"{"kind":"counter","name":"requests_total","value":42}"#;
    let parsed: Metric<'_> = serde_json::from_str(raw)?;
    println!("Parsed zero-copy: {parsed:?}");

    let encoded = serde_json::to_string_pretty(&parsed)?;
    println!("Re-encoded:\n{encoded}");
    Ok(())
}
```
