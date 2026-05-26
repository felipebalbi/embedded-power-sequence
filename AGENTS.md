# AGENTS.md

Guidance for AI coding agents (GitHub Copilot, Claude Code, Cursor, Aider,
Codex, Continue, etc.) working in the `embedded-power-sequence` repository.
Human contributors should also find this useful as a quick orientation guide.

This file is the authoritative agent-facing document. Tool-specific instruction
files (such as `.github/copilot-instructions.md`) intentionally stay small and
point here for the full picture.

---

## 1. Project overview

`embedded-power-sequence` is a small `no_std` Rust crate that defines a
Hardware Abstraction Layer (HAL) trait for **SoC power sequencing**. It is
intended for use in client-device Embedded Controller (EC) firmware, where the
EC is responsible for bringing an SoC through its power states (power-on,
power-off, idle, suspend-to-RAM, suspend-to-disk, etc.) in the correct order
and with the correct timing.

The crate is part of the [Open Device Partnership](https://github.com/OpenDevicePartnership)
(ODP) collection of embedded-systems building blocks.

Key properties:

- **`no_std`** — the library compiles without the standard library and is
  expected to run on bare-metal Cortex-M class microcontrollers (the CI
  target is `thumbv8m.main-none-eabihf`).
- **Async-first** — every state transition is an `async fn` on the
  `PowerSequence` trait. `async_fn_in_trait` is allowed crate-wide.
- **Trait-only** — the crate provides traits and error types, not concrete
  drivers. Downstream HAL crates implement `PowerSequence` for their
  particular board / SoC.
- **Optional `defmt` integration** — gated by the `defmt` cargo feature.
- **Macro-driven pre/post hooks** — the companion `macros` proc-macro crate
  expands each `#[power_state]`-annotated trait method into three variants:
  `pre_<name>`, `<name>`, and `post_<name>`. Agents touching trait methods
  must understand this expansion (see §4).

### 1.1 Repository layout

```
.
├── AGENTS.md                  <-- you are here
├── Cargo.toml                 <-- root crate (embedded-power-sequence)
├── Cargo.lock
├── src/
│   └── lib.rs                 <-- PowerSequence trait, Error, ErrorKind
├── macros/                    <-- path-dependency proc-macro crate
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs             <-- #[power_state] attribute macro
├── rustfmt.toml               <-- max_width=120, StdExternalCrate import groups
├── deny.toml                  <-- cargo-deny configuration
├── supply-chain/              <-- cargo-vet audits (owned by crate-auditors)
├── CODEOWNERS                 <-- ec-code-owners + crate-auditors for supply-chain
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── LICENSE                    <-- MIT
├── README.md
└── .github/
    ├── copilot-instructions.md
    └── workflows/
        ├── check.yml          <-- fmt, clippy, doc, hack, deny, msrv
        ├── nostd.yml          <-- no-std build on thumbv8m.main-none-eabihf
        ├── cargo-vet.yml
        └── cargo-vet-pr-comment.yml
```

### 1.2 Is this a workspace?

**No.** The root `Cargo.toml` is a single `[package]` with no `[workspace]`
table. The `macros/` directory is a separate crate consumed by the root via a
`path` dependency:

```toml
[dependencies]
macros = { path = "macros" }
```

Practical consequence for agents: `cargo` commands run from the repo root
operate on the `embedded-power-sequence` package only. To touch `macros`
directly, either `cd macros` or use `cargo -p macros …` after declaring a
workspace (do not do that lightly — it is an architectural change). For the
day-to-day `cargo build / check / clippy / test / doc` cycle, running at the
repo root is correct and will compile `macros` transitively.

### 1.3 MSRV, edition, license

- **Edition:** `2021` (both crates).
- **MSRV:** `1.75` — pinned because `embedded-hal-async` requires 1.75 and
  the workflow `msrv` job validates it. Do **not** silently raise it.
- **License:** MIT (root and `macros`).

---

## 2. Quick start for agents

Before making any code change, run the same checks CI runs. They are fast on
this crate (a few seconds each):

```bash
cargo fmt --check
cargo clippy --all-features --all-targets -- -D warnings
cargo build --all-features
cargo test --all-features
cargo doc --no-deps --all-features
```

The CI also runs:

```bash
# Feature-powerset check (cargo-hack must be installed):
cargo hack --feature-powerset check

# no-std target check:
rustup target add thumbv8m.main-none-eabihf
cargo check --target thumbv8m.main-none-eabihf --no-default-features

# Supply-chain checks:
cargo deny check --all-features
cargo vet
```

Install one-time tooling if you need it locally:

```bash
cargo install cargo-hack
cargo install cargo-deny
cargo install cargo-vet
rustup component add rustfmt clippy
rustup target add thumbv8m.main-none-eabihf
```

If a change you are making cannot pass `cargo check --no-default-features
--target thumbv8m.main-none-eabihf`, you have broken `no_std` support and
must fix it before considering the task complete.

---

## 3. Coding conventions

### 3.1 Formatting

`rustfmt.toml` pins:

```
group_imports = "StdExternalCrate"
imports_granularity = "Module"
max_width = 120
```

Always run `cargo fmt` before committing. Do **not** hand-format imports —
let rustfmt handle the grouping. `imports_granularity = "Module"` means
imports from the same module collapse into a single `use foo::{a, b};`
statement.

### 3.2 Lints

`cargo clippy` is run by CI on both `stable` and `beta`. Treat all warnings
as errors locally (`-D warnings`) so that beta-channel lint regressions are
caught before they hit CI.

If you genuinely need to silence a lint, prefer the narrowest possible scope
(`#[allow(...)]` on the item, not crate-wide) and add a one-line comment
explaining why.

### 3.3 `no_std`, `core`, and `alloc`

- The crate sets `#![cfg_attr(not(test), no_std)]`. **Never** introduce a
  dependency on `std::` from non-test code. Use `core::` equivalents.
- There is no `alloc` dependency today. Do not add one without a strong
  justification — many EC firmwares have no heap.
- `async_fn_in_trait` is intentionally allowed crate-wide. Do not remove the
  `#![allow(async_fn_in_trait)]` attribute.

### 3.4 Errors

The error model is the small `Error` / `ErrorKind` / `ErrorType` triple in
`src/lib.rs`, mirroring the embedded-hal conventions. When adding new error
variants:

1. Add the variant to `ErrorKind` (the enum is `#[non_exhaustive]`, so new
   variants are not a breaking change for downstream matchers that use a
   wildcard arm).
2. Update the `Display` impl with a human-readable message.
3. Update the `defmt::Format` derivation is automatic via the cfg-attr — no
   action needed, but verify with `cargo build --features defmt`.

### 3.5 Documentation

- Every public item must have a doc comment. `cargo doc --no-deps
  --all-features` is run on nightly with `RUSTDOCFLAGS=--cfg docsrs` — keep
  doc examples buildable.
- Use `///` for items and `//!` for module / crate docs.
- Cross-reference other items with intra-doc links: `[`PowerSequence`]`
  rather than `PowerSequence`.

### 3.6 Commit messages

Defined in `.github/copilot-instructions.md`, repeated here:

- Subject line: **capitalized**, **≤ 50 characters**, imperative mood
  (“Add foo”, not “Added foo” or “Adds foo”).
- Blank line between subject and body.
- Wrap body at **72 characters**.
- Body should explain **what** and **why**, not **how**.

#### AI attribution

Every commit that includes AI-generated or AI-assisted work **must** carry
an `Assisted-by` trailer:

```
Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
```

- `AGENT_NAME` is the AI tool / framework (e.g. `GitHub Copilot`,
  `Claude Code`, `Cursor`).
- `MODEL_VERSION` is the actual model in use (e.g. `claude-opus-4.7`,
  `gpt-5.3-codex`). **Verify your own model version at session start; do
  not hard-code a value from a previous run.**
- Optional bracketed tool names list specialized static-analysis or
  refactoring tools that materially shaped the change (e.g. `coccinelle`,
  `sparse`, `smatch`). General tools (git, cargo, your editor) do not
  belong here.

AI agents **must not** add `Signed-off-by` trailers. Only a human can
certify the Developer Certificate of Origin.

Example commit:

```
Add hibernate retry on transient I2C errors

The previous implementation propagated the first I2C NACK during the
SoC's hibernate handshake, leaving the platform in an unknown state.
Retry up to three times with a 5 ms back-off; if still failing, surface
ErrorKind::Other so the caller can decide what to do.

Assisted-by: GitHub Copilot:claude-opus-4.7
```

---

## 4. The `#[power_state]` macro — read this before touching the trait

The `macros` crate exports a single attribute macro: `#[power_state]`. It is
applied to every `async fn` on the `PowerSequence` trait (and on the
forwarding `impl<T: PowerSequence + ?Sized> PowerSequence for &mut T` block).

For a method `foo(...)`, the macro expands to **three** methods on the
trait:

| Generated method | Purpose |
| ---------------- | ------- |
| `pre_foo(...)`   | Pre-hook: called before the main transition. |
| `foo(...)`       | The transition itself (the body you wrote). |
| `post_foo(...)`  | Post-hook: called after the main transition. |

The macro also rewrites method calls and free-function calls inside the
body, prefixing them with `pre_` (in the pre-hook copy) and `post_` (in the
post-hook copy). Calls to `Ok(...)` are explicitly **not** prefixed.

Implications for agents:

1. When you call another `PowerSequence` method from within a
   `#[power_state]` body — e.g. the default `hibernate` body calls
   `self.power_off().await` — that call **becomes** `self.pre_power_off()`
   in the pre-hook copy and `self.post_power_off()` in the post-hook copy.
   That is intentional: the hook for `hibernate` composes the hooks of
   `power_off`. Do not "fix" this by trying to use UFCS or aliasing.
2. **Free-function calls are also rewritten**, except for `Ok`. If you call
   any function other than `Ok` from a `#[power_state]` body, make sure the
   prefixed variants (`pre_<fn>`, `post_<fn>`) exist or are valid no-ops.
   In practice this means: keep `#[power_state]` bodies small and limited
   to calling other `PowerSequence` methods plus `Ok(...)`.
3. The macro currently uses `expect("must have a segment")` and
   `unwrap()` on parse failures. Any change to the macro that broadens what
   it accepts must also handle the error case explicitly (`compile_error!`
   via `syn::Error::to_compile_error`) — never panic in user-facing
   proc-macros.
4. If you add a new method to the `PowerSequence` trait, you must:
   - Annotate it with `#[power_state]`.
   - Add a corresponding forwarding impl in the `impl<T> PowerSequence for
     &mut T` block, also annotated with `#[power_state]`.
   - Document the method.
   - Ensure any default body only calls other `PowerSequence` methods (or
     `Ok(())`), per point 2 above.

When in doubt, run `cargo expand` (install with
`cargo install cargo-expand`) on a small downstream test to inspect what
the macro produces.

---

## 5. Continuous integration

CI is GitHub Actions, configured under `.github/workflows/`. Pull requests
trigger:

### `check.yml`

| Job     | What it runs                                                | Toolchain        |
| ------- | ----------------------------------------------------------- | ---------------- |
| fmt     | `cargo fmt --check`                                         | stable           |
| clippy  | `giraffate/clippy-action` (PR-check reporter)               | stable, beta     |
| doc     | `cargo doc --no-deps --all-features` (`RUSTDOCFLAGS=--cfg docsrs`) | nightly  |
| hack    | `cargo hack --feature-powerset check`                       | stable           |
| deny    | `EmbarkStudios/cargo-deny-action@v2`, `--all-features`      | stable           |
| msrv    | `cargo +1.75 check`                                         | 1.75             |

A `semver` job exists in commented-out form; do not enable it until the
crate has had its first release on crates.io.

### `nostd.yml`

Runs `cargo check --target thumbv8m.main-none-eabihf --no-default-features`
on stable. Any change that breaks this is a regression.

### `cargo-vet.yml` and `cargo-vet-pr-comment.yml`

Enforce supply-chain audits via `cargo vet`. New or upgraded dependencies
require an audit entry in `supply-chain/`. Edits under `supply-chain/`
require approval from `@OpenDevicePartnership/crate-auditors` per
`CODEOWNERS`.

### Local pre-push checklist

Before pushing, the minimum agents should do:

```bash
cargo fmt --check
cargo clippy --all-features --all-targets -- -D warnings
cargo build --all-features
cargo test --all-features
cargo doc --no-deps --all-features
cargo check --target thumbv8m.main-none-eabihf --no-default-features
```

If you change `Cargo.toml` dependencies, also run `cargo deny check
--all-features` and, when applicable, `cargo vet`.

---

## 6. Per-pilot notes

The repository is agent-agnostic. The conventions in §3 and the `#[power_state]`
contract in §4 apply regardless of which tool is driving the keyboard.
Tool-specific guidance below is informational, not normative.

### 6.1 GitHub Copilot (chat / CLI / code review / coding agent)

- The dedicated file is `.github/copilot-instructions.md`. It contains the
  commit-message and AI-attribution rules and a pointer to this `AGENTS.md`.
- When asked to "verify the model version" before composing the
  `Assisted-by` trailer, do so — do not reuse a model name from a prior
  session.
- Copilot Code Review should flag any commit lacking the `Assisted-by`
  trailer on an AI-authored change.

### 6.2 Claude Code / Anthropic-driven agents

- Honour the §3 conventions and the §4 macro contract exactly. The
  `#[power_state]` expansion is non-obvious; do not refactor calls inside
  a `#[power_state]` body to use `<Self as PowerSequence>::foo(self)` UFCS
  — the macro's path rewriter is path-segment based and may behave
  unexpectedly with deeper paths.
- When proposing trait additions, generate both the trait method and its
  `&mut T` forwarding impl in the same edit.

### 6.3 Cursor / Continue / Aider / other IDE-embedded agents

- Respect `rustfmt.toml` and run `cargo fmt` after edits.
- Do not auto-add `Signed-off-by` trailers; only humans certify the DCO.
- The `Assisted-by` trailer is mandatory for any AI-touched commit.

### 6.4 Autonomous / "coding agent" runs

- Do **not** open PRs that touch `supply-chain/` without an explicit
  request — that area is governed by a separate code-owners team.
- Do **not** raise the MSRV (currently `1.75`) without an explicit request
  and a corresponding update to the matrix in `check.yml`.
- Do **not** add a workspace (`[workspace]` in the root `Cargo.toml`)
  without an explicit request — it changes how downstream consumers vendor
  this crate.
- Keep PR scope tight: README, trait surface, and CI tweaks are good
  candidates; sweeping refactors are not.

---

## 7. Common tasks and how to approach them

### 7.1 Add a new power state to the trait

1. Add the `async fn` to `PowerSequence` in `src/lib.rs` with a doc
   comment, a sensible default body, and the `#[power_state]` attribute.
2. Mirror it in the `impl<T: PowerSequence + ?Sized> PowerSequence for &mut T`
   block (also `#[power_state]`-annotated, forwarding to `T::your_method(self).await`).
3. If your default body calls another `PowerSequence` method, confirm the
   macro-generated `pre_*` and `post_*` variants are what you want (see §4).
4. Run the full local check list (§5).
5. Update `README.md` if the addition is user-visible.

### 7.2 Add a new `ErrorKind` variant

1. Add the variant to the `ErrorKind` enum (it is `#[non_exhaustive]`).
2. Extend the `Display` impl with a message.
3. Confirm `cargo build --features defmt` still passes (the derive is
   conditional).
4. Document downstream impact in the commit body if any.

### 7.3 Change a dependency

1. Edit `Cargo.toml`.
2. Run `cargo update -p <crate>` (or `cargo update`) and commit
   `Cargo.lock`.
3. Run `cargo deny check --all-features`.
4. Run `cargo vet` and, if required, add an audit entry in `supply-chain/`.
   Note that `supply-chain/` changes require auditor approval.
5. Run the full local check list (§5).

### 7.4 Modify the `#[power_state]` macro

1. Edit `macros/src/lib.rs`.
2. Replace any `unwrap()` / `expect()` on user input with proper
   `compile_error!` emission via `syn::Error::to_compile_error`.
3. Add a snapshot or `trybuild`-style test if you introduce new behaviour.
   (None exist today; adding the first one is a reasonable scope expansion.)
4. Run `cargo build` from the repo root — this rebuilds both crates.
5. Optionally inspect expansion with `cargo expand` against a downstream
   consumer.

---

## 8. Things to avoid

- ❌ Removing `#![cfg_attr(not(test), no_std)]` or adding `use std::` in
  non-test code.
- ❌ Removing `#![allow(async_fn_in_trait)]`.
- ❌ Adding a `[workspace]` table without explicit request.
- ❌ Raising the MSRV without explicit request.
- ❌ Hand-editing import grouping (let rustfmt do it).
- ❌ Panicking in proc-macro code on user input.
- ❌ Adding `Signed-off-by` trailers from an agent.
- ❌ Pushing commits without an `Assisted-by` trailer when the work was
  AI-assisted.
- ❌ Force-pushing shared branches.
- ❌ Editing `supply-chain/` without auditor sign-off.

---

## 9. Reference

- Open Device Partnership: <https://github.com/OpenDevicePartnership>
- This crate: <https://github.com/OpenDevicePartnership/embedded-power-sequence>
- Conventional Commits (loose inspiration only, not enforced):
  <https://www.conventionalcommits.org/>
- embedded-hal error conventions:
  <https://docs.rs/embedded-hal/latest/embedded_hal/>

If anything in this document conflicts with an explicit user instruction in
a session, the user instruction wins for that session — but please flag the
conflict so the document can be updated.

## Model selection & cost discipline

Premium models (Opus, GPT-5 family, "high"/"xhigh" reasoning variants)
cost an order of magnitude more than standard models (Sonnet, Haiku,
mini). Most steps in a typical task do not need premium reasoning,
and over-using premium models wastes credits without improving
outcomes. The rules below apply to *all* model selection: your own
session, sub-agents launched via the `task` tool, and parallel work
launched via `/fleet`.

### Default posture

- **Default to the cheapest model that can do the job.** Reach for a
  premium model only when one of the escalation triggers below is hit.
- **Plan with premium, execute with cheap.** Spend at most one or two
  premium turns on design / planning, then downshift to a cheaper
  model for mechanical execution of the plan.
- **Never bump the model "just in case."** If you cannot articulate
  *why* a cheaper model would fail, use the cheaper model.

### Escalation triggers (use a premium model)

Reach for a premium model when *any* of these are true:

- Cross-module refactor, architectural design, or API design from
  scratch.
- Subtle correctness reasoning: concurrency, lifetimes, `unsafe`,
  FFI ABI, cryptography, safety-critical control paths.
- Debugging a failure that survived one prior cheap-model attempt.
- Reviewing code on a safety-, security-, or money-critical path.
- The diff cannot be predicted in advance — i.e. there is genuine
  creative or design work to do, not just typing.

### De-escalation triggers (use a cheap model)

Use the cheapest available model when *any* of these are true:

- Searching, reading, summarising files or docs.
- Single-file mechanical edits: rename, format, lint fix, dependency
  bump, boilerplate, scaffolding from a known template.
- Generating tests for code that already works.
- Running builds, tests, linters, or other commands where the model
  only needs to report success/failure.
- Routine commits, PR descriptions, changelog entries.
- The diff is essentially predictable before generation.

### Sub-agent routing (the `task` tool)

When delegating with the `task` tool, set `model:` explicitly. Do not
let sub-agents inherit a premium default for cheap work.

| Sub-agent type    | Default model             | Override to                                     |
|-------------------|---------------------------|-------------------------------------------------|
| `explore`         | cheap                     | keep cheap (`claude-haiku-4.5` or `gpt-5-mini`) |
| `task` (run cmd)  | cheap                     | keep cheap                                      |
| `research`        | cheap for breadth         | premium only for the final synthesis            |
| `general-purpose` | match task                | cheap for mechanical work; premium for design   |
| `rubber-duck`     | premium                   | keep premium — this is where reasoning pays off |
| `code-review`     | premium on critical paths | cheap on cosmetic / mechanical diffs            |

### `/fleet` (parallel sub-agents) rules

- Fleet mode multiplies cost by the fleet width. Apply the rules
  above *per worker*, not in aggregate.
- Split a fleet job along complexity lines: route the cheap,
  parallelisable workers (file edits, test runs, doc updates) to a
  cheap model; reserve premium models for the small number of
  workers that need real reasoning.
- If every worker in a fleet would need a premium model, the work is
  probably not a good fit for fleet mode — reconsider the
  decomposition before paying N× premium.

### Session hygiene

- Keep sessions short and focused. Long premium sessions are the
  single largest source of waste because every turn re-processes the
  full history.
- Use `/compact` when the conversation grows long, and `/new` for
  unrelated work.
- Prefer `/ask` for one-off side questions so they don't extend the
  main session.

### When in doubt

Ask: *"If a cheaper model produced the wrong answer here, would I
catch it in seconds (compiler, tests, my own review) or in
weeks (production incident)?"* If the former, use the cheap model
and let the feedback loop do its job.
