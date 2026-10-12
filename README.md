# easy-peasy-rust

[![Check](https://github.com/skatromb/easy-peasy-rust/actions/workflows/check.yml/badge.svg)](https://github.com/skatromb/easy-peasy-rust/actions/workflows/check.yml)
[![Rust](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fskatromb%2Feasy-peasy-rust%2Fmain%2FCargo.toml&query=%24.package.rust-version&prefix=%E2%89%A5&label=rust)](https://crates.io/crates/easy-peasy-rust)
[![Crates.io](https://img.shields.io/crates/v/easy-peasy-rust.svg)](https://crates.io/crates/easy-peasy-rust)
[![Downloads](https://img.shields.io/crates/d/easy-peasy-rust.svg)](https://crates.io/crates/easy-peasy-rust)
[![easy-peasy-rust](https://img.shields.io/badge/style-easy--peasy-fecc02.svg)](https://github.com/skatromb/easy-peasy-rust)

An opinionated lint preset for Rust. Keeps human- and AI-written code readable.

Inspired by [wemake-python-styleguide](https://wemake-python-styleguide.readthedocs.io/).

## Usage

Run it inside a workspace or a single crate, or pass the path.

```sh
cargo install easy-peasy-rust
cargo easy-peasy [path]           # add what you lack, keep your lints
cargo easy-peasy --drop-existing  # apply the whole preset, drop your other lints
cargo easy-peasy --interactive    # ask about each block of lint rules that differs
cargo easy-peasy --check          # print what a run would change, fail if any
```

Then let your IDE run clippy, or run it yourself:

```sh
cargo clippy --workspace --all-targets
```

## Example

Code that compiles fine with default lints, where `read_primary` and `read_replica` are any `fn() -> Vec<u8>`:

```rust
use std::thread;

pub struct Header {
    pub id: u16,
    pub code: u8,
    pub count: u16,
}

pub fn parse(buf: &[u8]) -> Option<Header> {
    if buf.len() < 8 {
        return None;
    }
    let flags = u16::from_be_bytes([buf[2], buf[3]]);
    Some(Header {
        id: u16::from_be_bytes([buf[0], buf[1]]),
        code: (flags & 0x000F) as u8,
        count: u16::from_be_bytes([buf[6], buf[7]]),
    })
}

fn main() {
    let results = thread::scope(|s| {
        let a = s.spawn(read_primary);
        let b = s.spawn(read_replica);
        [a.join().expect("a panicked"), b.join().expect("b panicked")]
    });
    for h in results.iter().filter_map(|r| parse(r)) {
        println!("{} {} {}", h.id, h.code, h.count);
    }
}
```

With the preset, `cargo clippy` fails. Among the hits:

- `indexing_slicing` (deny): `buf[2]` panics if the length check drifts
- `expect_used` (deny): a panicked thread becomes a second panic
- `as_conversions`: `as u8` truncates silently
- `print_stdout`: `println!` panics on a closed pipe
- `min_ident_chars`, `disallowed_names`: `s`, `a`, `b`, `h`, `r`, `id`, `results`
- `missing_docs`, `must_use_candidate`, `missing_copy_implementations`: a `pub` item owes docs, `#[must_use]` and `Copy`

The same code with the preset:

```rust
use std::io::{Result as IoResult, Write as _, stdout};
use std::panic::resume_unwind;
use std::thread;

const HEADER_LEN: usize = 8;
const CODE_MASK: u8 = 0x0F;

struct Header {
    request_id: u16,
    code: u8,
    count: u16,
}

fn parse(buf: &[u8]) -> Option<Header> {
    let header: &[u8; HEADER_LEN] = buf.first_chunk()?;
    let [id_high, id_low, _, flags_low, _, _, count_high, count_low] = *header;
    Some(Header {
        request_id: u16::from_be_bytes([id_high, id_low]),
        code: flags_low & CODE_MASK,
        count: u16::from_be_bytes([count_high, count_low]),
    })
}

fn main() -> IoResult<()> {
    let replies = thread::scope(|scope| {
        [scope.spawn(read_primary), scope.spawn(read_replica)].map(|worker| {
            worker
                .join()
                .unwrap_or_else(|payload| resume_unwind(payload))
        })
    });
    let mut out = stdout().lock();
    for header in replies.iter().filter_map(|reply| parse(reply)) {
        let Header {
            request_id,
            code,
            count,
        } = header;
        writeln!(out, "{request_id} {code} {count}")?;
    }
    Ok(())
}
```

- The header is destructured from `first_chunk`: no indexes, no casts, a short buffer is `None`
- `resume_unwind` passes a thread's panic on instead of adding one
- Output goes to a `stdout()` handle, and a write error returns from `main`
- Names say what they hold, and private items need no docs

## Rules

Every lint pick is explained in [`lints.toml`](rules/lints.toml) and [`clippy.toml`](rules/clippy.toml).

## What it writes

- `Cargo.toml`: the preset lints, and `[lints] workspace = true` in every member crate
- `clippy.toml`: lint settings

## Overriding

Per crate, at the top of `src/lib.rs` or `src/main.rs`:

```rust
#![allow(clippy::some_lint)]
```

Cargo forbids own lint keys next to `workspace = true`.

Per workspace: set the lint to `"allow"` in `Cargo.toml`, or change a threshold in `clippy.toml`. The next `cargo easy-peasy` keeps it, `--drop-existing` resets it.

## Versioning

`1.99.x` has the lints of Rust 1.99. Older toolchains get only the lints and settings they know, per [`validity.toml`](rules/validity.toml): upgrade Rust and rerun `cargo easy-peasy` to add the rest.

## Badge

[![easy-peasy-rust](https://img.shields.io/badge/style-easy--peasy-fecc02.svg)](https://github.com/skatromb/easy-peasy-rust)

```md
[![easy-peasy-rust](https://img.shields.io/badge/style-easy--peasy-fecc02.svg)](https://github.com/skatromb/easy-peasy-rust)
```

## License

MIT
