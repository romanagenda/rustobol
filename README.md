# 🦀➡️📟 rustbol

> **Compile Rust to COBOL. Run anywhere that matters.**

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/rustbol/rustbol)
[![COBOL Version](https://img.shields.io/badge/COBOL-85%2F2002%2F2014-blue)](https://github.com/rustbol/rustbol)
[![z/OS Support](https://img.shields.io/badge/z%2FOS-2.4%2B-red)](https://github.com/rustbol/rustbol)
[![Crates.io](https://img.shields.io/badge/crates.io-v0.1.0--alpha-orange)](https://crates.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Mainframes Liberated](https://img.shields.io/badge/mainframes%20liberated-41%2C892-purple)](https://github.com/rustbol/rustbol)

```
rustbol - The Rust → COBOL transpiler
"If it compiles, we ship it to 1964."
```

---

## What is rustbol?

**rustbol** is a production-grade transpiler that converts Rust source code into standards-compliant COBOL, targeting IBM z/OS, Unisys ClearPath MCP, and Bull GCOS mainframe environments.

Stop choosing between memory safety and 99.999% uptime. With rustbol, you get both — plus the warm comfort of knowing your business logic is expressed in a language that was already mature when your senior engineers were born.

```rust
// Your modern Rust code
fn calculate_interest(principal: f64, rate: f64, years: u32) -> f64 {
    principal * (1.0 + rate).powi(years as i32)
}
```

Becomes:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALC-INTEREST.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01 WS-PRINCIPAL        PIC 9(12)V99 COMP-3.
       01 WS-RATE             PIC 9(4)V9(6) COMP-3.
       01 WS-YEARS            PIC 9(4) COMP.
       01 WS-RESULT           PIC 9(12)V99 COMP-3.
       01 WS-TEMP             PIC 9(12)V9(8) COMP-3.
       PROCEDURE DIVISION.
       CALCULATE-INTEREST-PARA.
           ADD 1.0 TO WS-RATE GIVING WS-TEMP
           COMPUTE WS-RESULT = WS-PRINCIPAL *
               FUNCTION INTEGER-PART(WS-TEMP ** WS-YEARS)
           STOP RUN.
```

---

## Why?

We asked ourselves: *"What if the borrow checker could enforce memory safety on a system that was installed before the concept of memory safety existed?"*

Turns out, it can. Mostly.

More seriously: a staggering **$3 trillion in daily commerce** flows through COBOL systems. Those systems aren't going anywhere. But their maintainers are — the average COBOL developer is 58 years old and has strong opinions about JCL that they are no longer able to share with anyone.

rustbol bridges this gap. Write in Rust. Run on the iron.

---

## Features

- ✅ **Full Rust 2021 Edition support** *(generics emulated via COPY book macros)*
- ✅ **Ownership model preserved** via `PERFORM THRU` structured sections
- ✅ **Lifetimes** compiled to `REDEFINES` clauses *(semantically equivalent, we think)*
- ✅ **Traits** transpiled to `CALL` interfaces and dynamically-linked subroutine libraries
- ✅ **Pattern matching** converted to nested `EVALUATE ALSO` statements
- ✅ **async/await** support via JES2 job stream scheduling *(latency: 2–8 hours)*
- ✅ **`Option<T>`** represented as `PIC X(1) VALUE SPACE` sentinel fields
- ✅ **`Result<T, E>`** mapped to standard IBM return codes (0 = Ok, 8 = Err, 12 = panic!)*
- ✅ **Cargo integration** — `cargo build --target cobol-ibm-zos`
- ✅ **Unit tests** compiled to batch test JCL with SYSOUT=\*
- ⚠️ **Closures** — *experimental, see known issues*
- ❌ **Threads** — COBOL runs one instruction at a time. This is a feature.

---

## Installation

### Prerequisites

- Rust 1.75+
- IBM z/OS 2.4+ **or** Unisys ClearPath MCP **or** access to a mainframe via a 3270 terminal emulator and someone who owes you a favour
- COBOL compiler: IBM Enterprise COBOL 6.3+, Micro Focus COBOL, or GnuCOBOL 3.1+ (for local testing)
- Optional: `ibm-jcl-formatter` for readable job streams

### Install via Cargo

```bash
cargo install rustbol
```

### Install from source

```bash
git clone https://github.com/rustbol/rustbol
cd rustbol
cargo build --release
# Binary at ./target/release/rustbol
# Note: first build takes 45 minutes. Subsequent builds: 44 minutes.
```

---

## Quick Start

### Transpile a single file

```bash
rustbol transpile src/main.rs --output MYMAIN.CBL
```

### Transpile a full Cargo project

```bash
rustbol build --target cobol-ibm-zos
```

Output files are placed in `target/cobol/`:
- `*.CBL` — COBOL source members
- `*.JCL` — Job Control Language to compile and link on z/OS
- `COPYLIB/` — Generated COPY books (equivalent to Rust modules)
- `PROCLIB/` — Catalogued procedures for CI/CD via JES

### Run locally with GnuCOBOL

```bash
rustbol build --target cobol-gnucobol
cobc -x target/cobol/MYMAIN.CBL -o mymain
./mymain
```

---

## Cargo Target

Add to your `.cargo/config.toml`:

```toml
[build]
target = "cobol-ibm-zos"

[target.cobol-ibm-zos]
runner = "rustbol-runner"
linker  = "iewl"   # IBM Linkage Editor
```

Supported targets:

| Target | Platform | COBOL Dialect |
|---|---|---|
| `cobol-ibm-zos` | IBM z/OS | IBM Enterprise COBOL |
| `cobol-ibm-zvsam` | z/OS + VSAM | VSAM-optimised I/O |
| `cobol-unisys-mcp` | Unisys ClearPath MCP | MCP COBOL-85 |
| `cobol-gnucobol` | Linux/macOS (testing) | GnuCOBOL 3.1 |
| `cobol-microfocus` | Windows/Linux | Micro Focus Visual COBOL |

---

## Transpilation Reference

### Types

| Rust Type | COBOL Equivalent |
|---|---|
| `u8` | `PIC 9(3) COMP` |
| `i32` | `PIC S9(9) COMP` |
| `f64` | `PIC S9(12)V9(6) COMP-3` |
| `bool` | `PIC X(1) 88 TRUE VALUE 'T' FALSE VALUE 'F'` |
| `String` | `PIC X(32767)` |
| `&str` | `PIC X(n)` *(n inferred at compile time — no heap)* |
| `Vec<T>` | `OCCURS 0 TO 32767 TIMES DEPENDING ON WS-VEC-LEN` |
| `HashMap<K,V>` | VSAM KSDS file *(requires z/OS)* |
| `Option<T>` | Field + `88 IS-NONE VALUE SPACE` |
| `Box<T>` | `POINTER` + `SET ADDRESS OF` |

### Control Flow

| Rust | COBOL |
|---|---|
| `if / else` | `IF / ELSE / END-IF` |
| `loop` | `PERFORM UNTIL EXIT` |
| `for x in iter` | `PERFORM VARYING` |
| `match` | `EVALUATE` |
| `?` operator | `IF RETURN-CODE NOT = 0 STOP RUN` |
| `panic!()` | `CALL 'CEE3DMP' USING ...` *(z/OS dump)* |
| `unreachable!()` | Blank `PERFORM` *(optimised away)* |

### Ownership & Borrowing

The Rust borrow checker is enforced at transpile time. Generated COBOL uses structured `PERFORM THRU` sections to simulate stack-scoped ownership:

```cobol
*> Rust: let x = String::from("hello");
*> Lifetime begins
       PERFORM OWN-WS-X-START THRU OWN-WS-X-END
*> Lifetime ends — field is zeroed
       MOVE SPACES TO WS-X
```

Mutable borrows generate a `REDEFINES` entry on the same storage location. Aliasing violations that pass the Rust borrow checker should not occur. If they do, please open an issue with your ABEND code.

---

## async/await on z/OS

Rust's async runtime is mapped to **JES2 batch job submission**. Each `async fn` becomes a separate JCL job member. `await` points become `INTRDR` (internal reader) submissions with checkpoint/restart via RACF-protected data sets.

```rust
async fn fetch_customer(id: u64) -> Customer { ... }

// In main:
let c = fetch_customer(42).await;
```

Compiles to a two-job JCL stream. Average latency per `await`: **4–6 hours** during business hours, **12 minutes** overnight batch window.

We consider this acceptable. Enterprise architects consider this *ahead of schedule*.

---

## Known Issues

- **Closures**: Closures that capture environment are transpiled to `WORKING-STORAGE` fields with `EXTERNAL` clause. Recursive closures cause `S0C4` abends on z/OS. Workaround: don't use recursive closures. Or closures generally.
- **Generics with more than 3 type parameters**: Generates COPY books that IBM Enterprise COBOL refuses to nest. Use `#[rustbol(inline_generics)]` attribute.
- **Floats**: COBOL `COMP-3` packed decimal is *not* IEEE 754. Results may vary by ±$0.01 per transaction. This is within rounding tolerance per GAAP. Probably.
- **The `HashMap` → VSAM bridge requires a 3390 DASD allocation of at least 15 cylinders.** This is not negotiable.
- **`std::thread`**: Not supported. COBOL is single-threaded. This is a philosophy, not a bug.
- **Compile times**: Comparable to standard Rust. COBOL compile times on z/OS are additional (budget 3–45 minutes depending on IBM software license tier).

---

## Configuration

`rustbol.toml` in project root:

```toml
[rustbol]
dialect = "ibm-enterprise"          # ibm-enterprise | gnucobol | microfocus
column-format = "fixed"             # fixed (cols 7-72) | free
line-length = 72                    # mainframes were built for 80 columns. we leave 8 for sequence numbers.
program-id-prefix = "RS"           # prefix for generated PROGRAM-IDs
working-storage-prefix = "WS"      # prefix for working storage vars
panic-behavior = "abend"           # abend | display-and-continue | ignore (prod only)
vsam-hlq = "SYS1.RUSTBOL"         # high-level qualifier for VSAM allocations
generate-jcl = true
jcl-job-card = """
//RUSTJOB  JOB (ACCT),'RUSTBOL BUILD',CLASS=A,
//             MSGCLASS=X,NOTIFY=&SYSUID
"""

[rustbol.optimizations]
dead-code-elimination = true       # removes unreachable COBOL paragraphs
paragraph-inlining = true          # inlines single-use PERFORMs
comp3-everywhere = true            # use COMP-3 for all numerics (faster on z/OS)
```

---

## CI/CD Integration

### GitHub Actions (compiles locally with GnuCOBOL)

```yaml
- name: Build with rustbol
  run: |
    cargo install rustbol
    rustbol build --target cobol-gnucobol
    cobc -x target/cobol/*.CBL
    ./target/cobol/main
```

### z/OS pipeline (via IBM Wazi)

```yaml
- name: Submit to JES
  uses: ibm/zos-deploy-action@v2
  with:
    jcl: target/cobol/BUILD.JCL
    sysout-class: X
    wait-for-completion: true
    max-wait-minutes: 480
```

---

## Philosophy

> *"The mainframe doesn't crash. The mainframe doesn't have supply chain attacks. The mainframe has been running your bank's money since before your parents met. Treat it with respect."*
>
> — `rustbol` contributor, ex-IBM, currently in therapy

COBOL processes an estimated **95% of ATM transactions** and **80% of in-person transactions** worldwide. It is not going away. It is simply waiting for a systems language worthy of generating it.

We believe that language is Rust.

The borrow checker is, in many ways, the spiritual successor to COBOL's `PERFORM THRU` structured programming discipline. Both exist to prevent you from doing something stupid with memory. One of them has been doing it since 1959.

---

## Contributing

PRs welcome. Please ensure:

1. `cargo test` passes
2. `rustbol build --target cobol-gnucobol` produces compilable COBOL
3. Generated COBOL conforms to COBOL-2014 standard
4. Column 7 is either `*` (comment) or space. **Never touch columns 1–6 or 73–80.** This is not a request.
5. All `PERFORM` statements have matching `END-PERFORM` or `THRU` pairs
6. You have read the COBOL-85 standard. Not summarised. Read.

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## Roadmap

- [ ] **v0.2** — Stable `HashMap` → VSAM bridge without manual IDCAMS JCL
- [ ] **v0.3** — `tokio` async runtime via JES3 complex (legacy IBM sites only)
- [ ] **v0.4** — `serde` → DFDL (Data Format Description Language) schema generation
- [ ] **v0.5** — CICS transaction support (`cargo run --bin cics-transaction`)
- [ ] **v1.0** — Full Rust standard library support *(estimated: 2031)*
- [ ] **Post-v1.0** — Explore Rust → RPG IV for AS/400 shops. We have been asked. We are not proud.

---

## License

MIT — see [LICENSE](LICENSE).

The generated COBOL is yours. IBM's COBOL compiler is not. Neither is the mainframe. You know what you signed.

---

## Acknowledgements

- The [GnuCOBOL](https://gnucobol.sourceforge.io/) project, without which local testing would require a mainframe
- Every COBOL programmer still employed because no one else understands their code
- IBM, for making hardware that outlasts the civilisations that bought it
- The Rust compiler team, for `rustc` error messages that are somehow clearer than any COBOL compile listing produced in the last 60 years

---

<p align="center">
  <em>rustbol: because your bank deserves memory safety too.</em><br><br>
  🦀 → 📟 → 💰
</p>
