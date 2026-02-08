# prost (fork with UUID support)

This is a fork of [tokio-rs/prost](https://github.com/tokio-rs/prost) — a Protocol Buffers implementation for Rust. For full documentation on prost itself, see the [original README](https://github.com/tokio-rs/prost/blob/master/README.md).

## What this fork adds

This fork adds native `uuid::Uuid` support for Protobuf `bytes` fields. Instead of generating `Vec<u8>` or `Bytes` for fields that represent UUIDs, prost will generate `uuid::Uuid` directly in your Rust structs.

UUIDs are serialized/deserialized as 16 raw bytes on the wire (standard Protobuf `bytes` encoding), but on the Rust side you get a proper `uuid::Uuid` type with all its methods and type safety.

## How to use

### 1. Add dependencies

In `Cargo.toml` of your project:

```toml
[dependencies]
prost = { git = "https://github.com/tolkichat/prost", branch = "uuid-support", features = ["uuid"] }
uuid = "1"

[build-dependencies]
prost-build = { git = "https://github.com/tolkichat/prost", branch = "uuid-support" }
```

### 2. Create an options.proto file

Define a custom field option so you can annotate UUID fields directly in your `.proto` files:

```protobuf
syntax = "proto3";

import "google/protobuf/descriptor.proto";

// Custom field option for UUID fields.
// Usage: bytes id = 1 [(uuid) = "v4"];
//
// Supported values:
//   "v4"  - Random UUID (default for most cases)
//   "v7"  - Time-ordered UUID (good for database keys)
//   "v5"  - Name-based UUID (deterministic)
//   "any" - Any valid UUID version
extend google.protobuf.FieldOptions {
  optional string uuid = 50001;
}
```

### 3. Use `[(uuid) = "..."]` in your proto files

Mark UUID fields with the `bytes` type and the `[(uuid)]` option:

```protobuf
syntax = "proto3";
package example;

import "options.proto";

message User {
  bytes id = 1 [(uuid) = "v4"];       // UUID v4 -> uuid::Uuid
  string name = 2;
  bytes avatar = 3;                    // regular bytes -> Vec<u8>
}
```

### 4. Configure build.rs

The `[(uuid)]` annotation in proto files is not read by prost directly — you need to parse it in `build.rs` and pass matching field paths to `config.uuid()`. Here is a complete working example:

```rust
use std::path::{Path, PathBuf};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let proto_dir = PathBuf::from("proto");

    // Auto-discover all fields annotated with [(uuid)] in proto files
    let uuid_patterns = collect_uuid_patterns(&proto_dir);

    let mut config = prost_build::Config::new();
    for pattern in &uuid_patterns {
        config.uuid([pattern.as_str()]);
    }

    config.compile_protos(
        &["proto/example.proto"],
        &["proto/"],
    )?;

    Ok(())
}

/// Recursively scan proto files and collect full paths of fields with [(uuid)].
fn collect_uuid_patterns(proto_dir: &Path) -> Vec<String> {
    let mut patterns = Vec::new();
    collect_patterns_from_dir(proto_dir, &mut patterns);
    patterns
}

fn collect_patterns_from_dir(dir: &Path, patterns: &mut Vec<String>) {
    let Ok(entries) = std::fs::read_dir(dir) else { return };
    for entry in entries.flatten() {
        let path = entry.path();
        if path.is_dir() {
            collect_patterns_from_dir(&path, patterns);
        } else if path.extension().map_or(false, |ext| ext == "proto") {
            collect_patterns_from_proto(&path, patterns);
        }
    }
}

fn collect_patterns_from_proto(path: &Path, patterns: &mut Vec<String>) {
    let Ok(content) = std::fs::read_to_string(path) else { return };

    let mut package: Option<String> = None;
    let mut message_stack: Vec<String> = Vec::new();
    let mut brace_depth: usize = 0;

    for line in content.lines() {
        let trimmed = line.trim();

        // Extract package name
        if let Some(rest) = trimmed.strip_prefix("package ") {
            if let Some(pkg) = rest.split(';').next() {
                package = Some(pkg.trim().to_string());
            }
        }

        // Track message nesting
        if let Some(rest) = trimmed.strip_prefix("message ") {
            if let Some(name_end) = rest.find(|c: char| c == '{' || c.is_whitespace()) {
                let msg_name = rest[..name_end].trim();
                if !msg_name.is_empty() {
                    message_stack.push(msg_name.to_string());
                }
            }
        }

        for ch in trimmed.chars() {
            match ch {
                '{' => brace_depth += 1,
                '}' => {
                    brace_depth -= 1;
                    if brace_depth < message_stack.len() && !message_stack.is_empty() {
                        message_stack.pop();
                    }
                }
                _ => {}
            }
        }

        // Found a [(uuid)] field — build full path like ".example.User.id"
        if trimmed.contains("[(uuid)") && !message_stack.is_empty() {
            if let Some(field_name) = extract_uuid_field_name(trimmed) {
                let msg_path = message_stack.join(".");
                let pattern = if let Some(ref pkg) = package {
                    format!(".{}.{}.{}", pkg, msg_path, field_name)
                } else {
                    format!(".{}.{}", msg_path, field_name)
                };
                patterns.push(pattern);
            }
        }
    }
}

/// Extracts the field name from a line like: `bytes id = 1 [(uuid) = "v4"];`
fn extract_uuid_field_name(line: &str) -> Option<String> {
    let before_option = line.split("[(uuid)").next()?.trim();
    let parts: Vec<&str> = before_option.split_whitespace().collect();
    let bytes_idx = parts.iter().position(|&p| p == "bytes")?;
    let field_name = parts.get(bytes_idx + 1)?.trim_end_matches('=');
    if field_name.is_empty() { return None; }
    Some(field_name.to_string())
}
```

**How it works:** the `build.rs` scans all `.proto` files, finds lines with `[(uuid)]`, extracts the full field path (e.g. `.example.User.id`), and passes it to `config.uuid()`. This way you only annotate fields once in the proto file — the build script picks them up automatically.

The `.uuid()` method on `Config` accepts field path patterns (same format as `.bytes()`):
- Full path: `".example.User.id"` — matches the `id` field in `example.User`
- Short name: `"id"` — matches any field named `id` in any message

### 5. Result

prost will generate:

```rust
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct User {
    #[prost(bytes = "uuid", tag = "1")]
    pub id: ::uuid::Uuid,
    #[prost(string, tag = "2")]
    pub name: String,
    #[prost(bytes = "vec", tag = "3")]
    pub avatar: Vec<u8>,
}
```

Now you can work with `id` as a real `uuid::Uuid`:

```rust
let user = User {
    id: uuid::Uuid::new_v4(),
    name: "Alice".to_string(),
    avatar: vec![],
};

println!("User ID: {}", user.id); // prints: "User ID: 550e8400-e29b-41d4-a716-446655440000"
```

## How it works internally

- `prost-build`: new `BytesType::Uuid` variant and `Config::uuid()` method that marks matching `bytes` fields for UUID generation
- `prost-derive`: handles `#[prost(bytes = "uuid", ...)]` attribute — generates `uuid::Uuid` type, uses `Uuid::nil()` as default, treats `Uuid` as `Copy`
- `prost` (runtime): implements `BytesAdapter` for `uuid::Uuid` — serializes/deserializes as 16 raw bytes

## License

Same as the original prost — Apache License (Version 2.0). See [LICENSE](https://github.com/tokio-rs/prost/blob/master/LICENSE).
