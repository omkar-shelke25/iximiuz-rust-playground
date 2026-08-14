# 🦀 Rust Playground

A disposable, ready-to-code Rust development environment powered by Cargo and Rust stable — no setup required.

---

## What's Inside

| Tool | Version |
|---|---|
| Rust | `stable` (via rustup) |
| Cargo | included |
| rustfmt | included |
| Clippy | included |
| rust-analyzer | VS Code extension |

---

## Getting Started

Open the **IDE** or **Term** tab and start coding right away.

```bash
# Create a new project
cargo new my-project
cd my-project

# Run it
cargo run
```

---

## Cargo Commands

```
cargo new <name>   →  Create a new project
cargo run          →  Compile and run
cargo build        →  Build without running
cargo check        →  Fast type and syntax check
cargo fmt          →  Auto-format your code
cargo clippy       →  Lint and catch mistakes
cargo test         →  Run your test suite
cargo add <crate>  →  Add a dependency
cargo update       →  Update dependencies
cargo doc --open   →  Open docs in browser
cargo clean        →  Remove build artifacts
```

---

## Environment

| Variable | Value |
|---|---|
| `RUSTUP_HOME` | `/usr/local/rustup` |
| `CARGO_HOME` | `/usr/local/cargo` |
| `PATH` | includes `/usr/local/cargo/bin` |

---

## Tabs

| Tab | Description |
|---|---|
| **IDE** | VS Code in the browser with rust-analyzer |
| **Term** | Terminal with full Rust toolchain |
| **doc.rust** | Embedded [The Rust Book](https://doc.rust-lang.org/book/) |

---

## Machine

| Resource | Value |
|---|---|
| CPU | 4 cores |
| RAM | 10 GiB |
| Disk | 30 GiB |
| OS | Ubuntu (Docker-enabled) |

---

## Docker Image

```
ghcr.io/omkar-shelke25/rust-playground:0.0.5
```

Build args:

```dockerfile
ARG RUST_VERSION=stable
ARG LAB_USER=laborant
```

---

## Resources

- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [crates.io](https://crates.io)
- [Rust Standard Library](https://doc.rust-lang.org/std/)
- [Clippy Lints](https://rust-lang.github.io/rust-clippy/)

---

Have fun building with Rust! 🦀
