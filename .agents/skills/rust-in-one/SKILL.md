---
name: rust-in-one
description: Build and verify new Rust CLIs, libraries, and services; migrate C/C++ or other software to Rust with explicit compatibility and performance checks; fix, extend, refactor, review, audit, or optimize existing Rust code. Applies before Cargo.toml exists and when resuming Rust work. Use only for Rust work, including Rust components of mixed repositories. Do not choose Rust merely because this skill is available when the language is undecided.
license: MIT
---

# Rust-in-One

Own Rust implementation requests through a usable, verified deliverable, even when the request is brief or informal. Infer the intended outcome, make routine product and technical decisions, and carry out implementation, verification, review, and corrections. Do not stop at a plan, scaffold, first working path, or an offer to continue unless the user limited the task to that stage. Follow explicit user scope, host permissions, and repository instructions. Reuse planning, checks, and review supplied by the host or another skill; do not duplicate valid work for the same revision.

Files under `references/` in this skill's directory are not loaded with this file. When a trigger below applies, open that file with your file-reading tool before relying on it. If a reference file cannot be read, report that limitation and consult relevant primary documentation.

## Choose the scope and checks

- **Review or audit:** open [verification.md](references/verification.md) and follow **Review existing code**. Establish the revision and scope, then return findings and coverage. Leave the reviewed files unchanged unless fixes were also requested. The implementation, repair, and delivery steps below apply only when that work is requested; a review does not require another reviewer to review it.
- **New project or substantial feature:** turn the request into a complete user workflow and observable acceptance conditions. Build one real end-to-end path, then expand until the requested behavior and delivery requirements are covered.
- **Migration:** first open [migration.md](references/migration.md) and follow it. Establish the source behavior and performance baseline before designing the Rust replacement. A wrapper is not a native migration unless the user requested one.
- **Change to an existing project:** before editing, trace what the change touches: entry points, which module owns the data and state transitions, error and output conventions, dependency boundaries, and public or compatibility obligations. Extend the types, helpers, and tests that already own the behavior instead of adding a parallel implementation.
- **Focused fix:** inspect the affected code and tests, reproduce the defect, make the smallest coherent correction in the surrounding style, and run relevant checks. Keep unrelated refactoring out of the fix. No planning document or extra reviewer by default. Escalate review when the *changed behavior* has material safety, security, concurrency, persistence, or compatibility risk.

Read applicable repository instructions and, where present, Cargo manifests, toolchain pins, and CI configuration. Preserve the edition, MSRV, features, conventions, and unrelated working-tree changes. Run relevant baseline checks in existing projects; distinguish pre-existing failures from regressions. For a new project, check the available toolchain and choose a supported edition and layout.

## Turn intent into completion conditions

Recover intent from the request, conversation, repository, and existing users' entry points. Missing detail is a design responsibility: choose coherent, conservative defaults and state material assumptions without making routine decisions a confirmation gate. When product choice is delegated, select a useful, bounded workflow and build it completely. Ask a focused question only when a consequential choice cannot be resolved from context or a reasonable default; continue independent work while it is unresolved. Respect requests for research, planning, or review only.

For substantial work, keep a short requirements checklist (the task contract), reusing an existing specification or host task list. Cover the complete user workflow, failure handling, installation or use, public boundaries, state invariants, observable side effects, supported environments, and resource or compatibility obligations. Associate consequential requirements with observable acceptance checks. Distinguish **implemented** from **verified**; retain unmet requirements, blockers, and the next action. Evidence identifies the artifact revision and conditions checked. Keep any planning for small, self-contained work in a brief in-chat checklist, without adding planning files to the deliverable. A focused fix needs only its affected contract and regression check.

Choose the least complicated design that satisfies the whole contract. Preserve correctness, safety, required compatibility, and resource constraints while improving readability and maintainability. Do not silently reduce scope, weaken acceptance checks to fit the implementation, or add unrelated features. Update the contract when a justified decision resolves an assumption; keep material behavior changes visible.

For substantial implementation work, read the relevant product row and checks in [verification.md](references/verification.md) **while planning the first working path**, not only at delivery. Read additional references at their triggers below, and reassess those triggers when the design changes: an initially safe rewrite can acquire an FFI boundary.

## Implement, observe, and finish

Build one real path early: CLI input to output, a library call from outside the crate, or a request through the service's actual transport. First test assumptions that could invalidate the design, such as format semantics, dependency capabilities, public API usability, cancellation, or performance feasibility. For larger work, establish module responsibilities, shared invariants, and dependency order before distributing implementation. Keep one owner of integration; give delegated work clear boundaries and acceptance conditions.

Repeat the following until the requested outcome and its required checks are complete:

1. **Select the next obligation.** Choose an unmet requirement, demonstrated defect, or important unverified assumption. Prioritize what blocks a complete user workflow or could invalidate later work. Identify the observation that would settle it.
2. **Make a coherent change or investigation.** Trace the relevant owners and callers, load the applicable references, and implement the smallest integrated step toward that obligation. Preserve existing behavior outside the intended change.
3. **Observe the artifact.** Run the most decisive relevant check on the actual execution path. Compare the result with the contract, not the implementation's expectations. Record what the result establishes and what remains unverified; a successful command alone does not discharge unrelated requirements.
4. **Resolve and recheck.** Reproduce failures, correct their cause, and rerun affected checks. If an approach stalls, reduce the case, revisit the assumption, or seek an independent diagnosis instead of repeating the same attempt. Integrate completed work and update the remaining obligations.

Run focused checks during development and broader integration checks at meaningful boundaries and delivery. Batch independent checks where useful, retaining each result. Reuse information that still matches the current revision, and stop exploring once the next decision is supported. A change can invalidate earlier evidence even when its local test passes. Spend additional testing and review on concrete remaining risks; avoid repeated full suites, speculative refactoring, or extra reviewers that supply no new evidence. A working slice is progress, not completion of the remaining scope.

As the code grows, keep one implementation of each concept. Before adding a parser, error type, configuration field, or helper, look for an existing one and extend it. Follow the established module layout, naming, and error conventions unless you change them deliberately throughout. Each delivered feature is reachable from the real interface, documented where users look, and tested there; do not deliver `todo!()`, inert options, unused modules, or parallel old and new paths. New projects ship useful tests and a concise README or equivalent usage documentation; reducing process overhead must not remove these. After a change, check interactions with existing state, errors, and resource ownership; measure when it touches a path with a performance requirement.

Choose the smallest useful design:

- Use `cargo new` / `cargo init` when appropriate. Keep `Cargo.lock` for reproducible development unless the repository's existing policy excludes it. A published library's lockfile does not pin its consumers' dependencies. Set and verify `rust-version` when promising an MSRV. Do not change existing compatibility policy incidentally.
- Separate reusable logic from CLI wiring when it improves testing or reuse; a small binary need not have a library target. Use workspaces for genuinely separate crates.
- Verify dependency versions and features through Cargo, the existing lockfile, or registry/docs; `cargo add` is convenient, not mandatory. Read docs or source for the resolved version when uncertain. Respect offline and vendored builds.
- Prefer private items, borrow inputs when useful, and choose owned results or borrowed views according to lifetime and allocation needs. Use enums and validating constructors for meaningful invariants.
- Before cloning to satisfy the borrow checker, consider smaller borrow scopes, splitting fields, or indices into an arena. Shared ownership and locks are legitimate when the design needs them.
- Treat public types as API decisions. `#[non_exhaustive]` permits evolution but restricts downstream construction and exhaustive matching; provide constructors or accessors and test the API from a separate consumer. Add traits such as `Debug`, `Clone`, or `Default` only when their semantics fit.
- Return useful errors. Libraries normally return errors rather than printing or exiting; typed errors are useful when callers need to distinguish cases. Applications may use `anyhow` or simpler errors. Attach operation/input context at boundaries. Invalid input and I/O failures are not reasons to panic.

## Keep the code readable and changeable

A maintainer should be able to see what the program does, in what order, and what each result means without reconstructing it from details. Unclear structure hides defects and makes later changes risky. `cargo fmt` fixes layout, not structure.

- **Results that state their meaning.** When a value decides later behavior, make that meaning visible where it is handled. A `bool` or tuple is fine when the function name makes it obvious. A consequential state, such as input that ended mid-record and changes the exit status, reads better as a named enum variant or struct field than as `Ok(true)`. Name items for the domain concept, not the mechanism.
- **Visible control flow.** Let the top-level function read as the product's stages, such as configure, read, process, write, and report, and map outcomes to exit statuses in one place. Separate logical stages with a blank line and keep each stage's statements together.
- **Boundaries by responsibility.** Split functions and modules where parts change for different reasons, such as input parsing, domain rules, I/O, and presentation. Keep together what changes together. Length alone is not a reason to split, and a split does not by itself require a library target, trait, or new type.
- **Validate at boundaries.** Convert external input into checked values and pass those on, rather than repeating range or format checks where they can drift apart. Recheck at a new trust boundary or when a state change can invalidate the earlier check.
- **Reuse matching semantics.** Prefer std, an existing dependency, or the repository's helper when its behavior and resource use fit the contract, for example `str::parse`, `checked_add`, `split_once`, or `sort_by_key`. Hand-written parsing or arithmetic is extra code to test and maintain; keep it when the contract differs from the library or measurements justify it, and test the boundary that justifies it. When replacing lower-level or foreign APIs, check differences in accepted input, errors, and edge cases.
- **Earned abstractions.** Add a type, trait, generic, or module when it names a real concept, removes real duplication, or isolates something expected to vary. Remove dead code, unused parameters, and placeholder branches. Comments state contracts, invariants, and non-obvious reasons rather than narrating code.

Restructuring preserves observable behavior unless the requirements change it: output emitted before a failure, which error is reported when several apply, side effects and state changes, flushing and cleanup, limits, exit statuses, and public interfaces. Validating all input before writing output, or streaming input that was previously collected, can be an improvement or a compatibility break; check the contract first. When existing tests do not pin the behavior being restructured, add a characterization test before changing it.

## Rust pitfalls to check

- **Observable order:** hash collection iteration is unspecified. Sort when deterministic output is required; define tie behavior where it affects results. Do not add a new tie-breaker that changes a compatibility contract.
- **Bytes and text:** `String` requires valid UTF-8. Files and process output may contain non-UTF-8 bytes, and paths may not be representable as UTF-8. Decide whether to reject, preserve, or transform invalid UTF-8 sequences. Use `Path`/`OsStr` for paths. `String::len` counts bytes, and string slicing requires UTF-8 character boundaries.
- **Resource growth:** identify what grows with external input and outstanding work: buffered bytes, decoded size, allocations, nesting, or queued tasks. A numeric value or item-count limit need not bound its encoded representation. Use incremental processing or justified limits where needed, preserving required output and failure behavior; check the boundary that makes the choice necessary.
- **Arithmetic:** default Cargo profiles check ordinary integer overflow in debug and wrap in release, but flags can change this; division/remainder by zero and signed `MIN / -1` or `MIN % -1` still panic. Use checked, saturating, or wrapping operations to express the required behavior. Validate narrowing conversions with `TryFrom`; choose count/offset widths from the format and platform contract. Define a NaN policy before ordering floats (`total_cmp` provides a total order).
- **CLI output:** flush buffered writers explicitly and propagate write and `flush()` errors; dropping a `BufWriter` ignores flush failures. For ordinary filter CLIs, handle `BrokenPipe` without panicking; for migrations, match the source's broken-pipe, output, error, and exit-status contract. Return a deliberate exit status and readable stderr: returning `Result` from `main` formats errors with `Debug`, and `process::exit` skips destructors of values still alive in the calling frames.
- **Global state:** tests run concurrently. Prefer injected configuration or subprocess tests over mutating environment/current directory in-process. Edition 2024 makes `std::env::set_var`/`remove_var` unsafe; a test mutex alone does not constrain other threads or foreign code.
- **Crates:** use maintained parsers and cryptography rather than inventing them. Common options include `clap`, `serde`, `thiserror`/`anyhow`, `tokio`, `axum`, `reqwest`, `tracing`, and `rayon`; choose only what the product warrants. Enable needed features. Prefer std facilities such as `OnceLock`/`LazyLock` when they cover the need and MSRV.

Before writing or reviewing a service, async code, or nontrivial multithreading, read [concurrency.md](references/concurrency.md). Choose sync, threads, or async from the workload; specify limits and shutdown/cancellation behavior before spawning work.

Before writing, changing, or reviewing unsafe code or FFI, read [unsafe-ffi.md](references/unsafe-ffi.md). Keep the boundary narrow, document the safety obligations, and use safe alternatives where they meet the requirements.

Before optimizing, benchmarking, reviewing performance claims, or claiming speed/memory improvements, read [performance.md](references/performance.md). Measure a correct optimized artifact against a defined baseline, workload, and acceptance threshold. Streaming and buffer reuse help where input size or profiling justifies them.

## Verify and review

Use the project's checks. For new projects, a useful starting point is:

```sh
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Then test supported feature/target combinations; `--all-features` is useful only if that combination is supported. `cargo test --all-targets` skips doctests, so run doctests separately when using it. Exercise release builds where profiles affect correctness. Also verify the delivered boundary: the installed binary, the packaged library from an external consumer, or the running service over its real transport.

Choose tests from the contract and failure risks: vary input, actual execution path, prior state, and observable result. Check consequential boundaries in each independently handled input and failures after useful work has begun; assert what output or state remains. A helper or help-path test does not cover a different runtime path. For central algorithms or state machines, use an independent small-case oracle or invariant where it can expose errors ordinary examples miss. See [verification.md](references/verification.md) for selecting these checks. Reproduce a regression without the fix when feasible; avoid exhaustive combinations or machinery that does not address a concrete risk.

For new projects, substantial migrations/features, and materially risky changes, arrange a bounded independent review after an integrated artifact works. Reserve time for it and corrections within the task allowance. Bring forward review of an assumption when getting it wrong would invalidate substantial work. Give a fresh reviewer the request, requirements checklist, artifact revision, and build/run commands, without the implementer's verdict or continuation notes. Ask for concrete defects with evidence, gaps between the request and the contract, and consequential wrong behavior that tests would miss. Include maintainability findings that name code and explain the cost, such as duplicated concepts, mixed responsibilities, misleading state, or unnecessary layers. Use an actual requested change to inspect where responsibilities and invariants would need to change; do not invent features to justify abstractions. A maintainability finding needs a location and a reason, not necessarily a failing test. Reuse an existing independent review when it covers the final changes.

Track independent review as **pending**, **completed**, **not performed**, or **unavailable**. Use **not performed** when review was omitted without a host or budget constraint, including focused fixes that did not require review. Use **unavailable** only when a specific host or budget constraint prevented independent review; state that constraint. In that case, perform a distinct adversarial self-review and report it separately; it does not change the independent-review status to **completed**.

Wait for an invoked reviewer to finish; starting a review leaves it **pending**. Reproduce or refute findings, fix material defects, and rerun affected checks. Ask the reviewer to recheck changed areas when fixes invalidate earlier review assumptions; if independent review is unavailable, label the recheck as self-review. Do not claim independence or quietly omit an unfinished review. [verification.md](references/verification.md) supplies a compact review brief.

## Resume and deliver

Use the host's actual capabilities for execution, delegation, persistent task state, and continuation; do not assume a particular plugin or command exists. For work that may span contexts, retain a short private checkpoint: goal and task contract, current artifact revision, next obligation, loaded reference paths, evidence with revision/results, review status, and unresolved findings. Use an existing host task list or private state facility, or a scratch location outside the repository and deliverable, and retain its location through the host's continuation mechanism. Do not create repository instruction files for task tracking or commit handoff notes. A Skill cannot restart a stopped host; report a missing capability instead of claiming it worked.

After context loss, reread this skill and the references needed for the next step. Reconcile the checkpoint with the actual diff and artifacts. Missing evidence is unverified; stale test/review results do not become current by being copied into a summary. Rerun checks affected by changes, not every unrelated check automatically.

Before delivery, reconcile every consequential requirement with the final artifact and its evidence, and reread the diff as its next maintainer would. Formatter success, passing tests, a completed review, or idiomatic-looking syntax do not by themselves establish completeness or maintainability. Run affected final checks and documented commands in the supported environment. Check install/package contents for scratch files and unintended dependencies. Resolve demonstrated material defects before claiming completion. If a genuine access, environment, product-decision, or budget blocker remains, finish independent work, preserve the checkpoint, and identify the unmet requirement and what would unblock it; do not label a partial result complete. Report the delivered behavior, material decisions, evidence and its revision, review status, accepted differences, and remaining gaps concisely.
