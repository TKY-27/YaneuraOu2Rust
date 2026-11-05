# Verification, review, and delivery

Read the relevant sections while planning substantial work, before designing the first working path or public interface. For a review-only request, start with **Review existing code** below. Return to the applicable checks for review and delivery. The table is a menu of product risks, not a requirement to run every check on every task.

## Map requirements to evidence

Start with the user's workflow and contract, including the assumptions made from a brief request. Check that the requirements cover a usable result, failure handling, and delivery before deriving tests from them. A checklist copied from the implementation can preserve the same omissions. Reuse existing specifications and checks where they still apply.

For each consequential requirement, choose evidence that would distinguish the intended behavior from a plausible wrong implementation. Record the result and its limits beside the requirement it supports; test totals do not measure coverage. Use these questions to select cases:

- **Input:** which valid boundaries, malformed values, encodings, and size limits matter? Check each independently handled field; a boundary test for one field does not establish another's behavior.
- **Path:** does the check reach the real entry point and the branch that owns the behavior? A successful helper, startup, or help path does not establish the normal processing path's errors or cleanup.
- **Prior state:** what changes when work has already succeeded, a resource is occupied, a dependency fails, or an operation is retried? Exercise the transitions where failure can leave partial state.
- **Observable result:** assert the relevant return value, output, error or exit status, state, and side effects after success and failure. Check ordering or atomicity when the contract promises it.

Combine these dimensions where they interact; do not generate their full Cartesian product. For example, a file exporter needs a check that failure during replacement preserves the promised existing data, as well as a successful export. Rejecting an invalid destination before any work starts does not exercise that transition. Use fault injection at the boundary when practical, and identify any gap between an injected unit test and the real installed artifact.

For a risky unproven assumption in an algorithm, parser rule, failure transition, or dependency, seek an independent discriminator: a simple exhaustive solver on bounded inputs, a small reference model, a characterized reference behavior, a known result, or a separately derived invariant. Choose a method likely to fail differently from the implementation. Repeating its calculations or merely checking that it does not panic offers little evidence about a wrong result; round trips alone can also preserve a shared encoder/decoder error. A second implementation is unnecessary when direct cases already distinguish the relevant failures.

| Product | Checks that usually matter |
| --- | --- |
| CLI | Run the built binary: stdout and stderr kept separate, exit codes, stdin and `-` where supported, normal-processing write/flush failures, early downstream closure without a panic, missing and unreadable files, a directory passed as a file, empty and non-UTF-8 input, large input against the time and memory requirements, `--help` matching the README. For migrations, preserve the source's output and exit-status contract. |
| Library | Integration tests in `tests/` using only the public API; compiling and running examples; doctests; error types satisfying caller bounds (often `Send + Sync + 'static`); public types with the intended `Send`/`Sync` properties; supported feature combinations; documentation building without warnings |
| Service | Real requests to a server bound to `127.0.0.1:0`: malformed and oversized requests, request size limits and timeouts, peers that stop reading responses, recovery of saturated worker capacity, dependency failures, graceful shutdown with in-flight requests, configuration errors at startup |
| Concurrent code | Shutdown and cancellation paths, error propagation from tasks and threads, bounded queues under load, repeated runs to expose flakiness; `loom` for custom lock-free or atomic protocols that can be modeled with it |
| Persistence | Interrupted writes (same-filesystem temporary file, flush/sync as required, atomic rename where supported, and parent-directory sync for crash durability on platforms that require it), recovery on restart, behavior when reading files from older or newer versions, and concurrent access (or explicit locking/rejection) when multiple readers or writers can occur |
| Parsers and decoders of untrusted input | Property tests (`proptest`) for round trips and invariants; fuzzing (`cargo fuzz`, nightly) for crash-freedom; explicit nesting, encoded token/line length, decompression, and retained-allocation cases. Exercise valid long encodings as well as rejection; value/count limits alone do not establish bounded memory |
| Performance requirements | See [performance.md](performance.md) |

## Commands worth knowing

```sh
cargo test --workspace --all-features            # includes doctests
cargo test --doc                                 # doctests only; --all-targets skips them
cargo test --no-default-features                 # when the crate supports it
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps   # broken intra-doc links, missing docs if linted
cargo build --release && ./target/release/<bin>  # the artifact users run
cargo install --locked --path . --root "$(mktemp -d)"  # install with the tested Cargo.lock, then run it
cargo package --list                             # files a published crate will contain
cargo package                                    # builds the crate from its packaged contents
cargo tree --duplicates                          # duplicate dependency versions
```

Use relevant optional tools when available, and follow the project's existing verification requirements. Report a check as run only after actually executing it: `cargo semver-checks` before releasing a library, `cargo audit` or `cargo deny check advisories` for security advisories, `cargo deny check licenses` for license policy, `cargo +nightly miri test` for unsafe code (see [unsafe-ffi.md](unsafe-ffi.md)), `cargo hack --feature-powerset check` for feature combinations, and `cargo msrv verify` or `cargo +<msrv> check` for a declared `rust-version`.

Compiling across feature combinations does not test their behavior. A passing `cargo check` on another target does not show the program works there. During implementation, run checks for the changed requirement first; verify working slices before building further, with full integration checks at meaningful boundaries and delivery.

## Test design notes

- Put unit tests next to the code (`#[cfg(test)] mod tests`) for internal logic and integration tests in `tests/` for the public surface. Integration tests compile as separate crates and see only public items.
- Use isolated temporary directories for filesystem tests (`tempfile::TempDir` is convenient). Do not mutate source fixtures or depend on an incidental current directory.
- For complex textual output, golden files or snapshot tests (`insta`) catch regressions. Review every snapshot update rather than accepting it blindly.
- For isolated async timer logic, use `#[tokio::test(start_paused = true)]` with Tokio's `test-util` feature and current-thread runtime. Real transport tests still need bounded real deadlines.
- A passing test can provide evidence even if you did not observe it failing first. For a bug-fix regression test, confirm that it fails without the fix when feasible so you know it detects the actual defect.
- Inspect whether important assertions would detect a plausible mistake, such as committing partial state on error, dropping a tie rule, accepting an invalid boundary, or returning success after an output failure. If detection is unclear, try a focused mutation in a disposable copy and confirm the relevant test fails. Use this to resolve a concrete doubt, not to meet a mutation score or test-count quota.

## Review existing code

Use this workflow for a whole-project audit or a review of a diff, commit, or working-tree change. The finding format and premise checks below also apply here.

- Establish the target revision, scope, and comparison base where applicable. Read the repository's instructions, contracts, build configuration, and relevant tests. For a large repository, map its entry points and module boundaries, prioritize the consequential paths, and state what remains unexamined.
- Trace callers, data ownership, state transitions, error paths, and downstream compatibility before judging a local pattern. Use the product risks above and relevant Skill references; open [migration.md](migration.md) when assessing a port's fidelity. Check the resolved dependency versions, enabled features, and target assumptions behind a finding.
- Compare the tests with the contract and actual execution paths. Look for a consequential failure the assertions would miss, including failure after progress and independent evidence for central calculations. Explain the missing protection without treating a missing test alone as proof that the implementation is wrong.
- Preserve the reviewed files and unrelated user changes. Use checking modes rather than automatic formatting or fixes; use a disposable copy for reproducers or commands that may alter tracked files. Run repository code within the host's trust and permission boundaries.
- Report concrete defects first, ordered by impact. For each, give the location, triggering conditions, violated contract or invariant, consequence, and evidence. Distinguish a reproduced failure from a source-based inference or an unresolved question. For a diff review, distinguish introduced problems from pre-existing ones; do not present speculative performance concerns as measured regressions.
- Report maintainability concerns separately. Trace an actual shared concept, such as parsing policy, configuration, domain state, or error mapping, through its definition, callers, adapters, and tests. Identify inconsistent contracts, duplicated rules, unclear ownership, or changes requiring synchronized edits. Use a real subsequent request to assess change cost when available; do not invent features to justify refactoring. Respect deliberate design and conventions; LOC, module or enum counts, and preferred style alone do not establish a defect.
- Deliver findings and practical correction directions, then the reviewed revision and scope, checks actually run, and remaining gaps. If no actionable findings remain, say so without certifying correctness. Do not invent findings to fill a quota or start repairs unless requested.

## Independent review

Give the reviewer:

- the user's original request and the requirements checklist, including accepted assumptions;
- the artifact: repository path and commit, or the diff, plus build and run commands;
- the risk areas of the product, such as untrusted input, unsafe code, concurrency, and compatibility commitments. These describe the product, not your conclusions about it.

Do not give the reviewer your own verdict, your summary of which tests pass as a reason to trust the code, continuation notes, or a list of areas already checked. Ask the reviewer to compare the contract with the original request and inspect whether the tests would catch consequential wrong behavior, not just whether they pass. Request defects first, then the concrete maintenance assessment described under **Review existing code**. Ask for findings in this form:

```text
[severity: critical/high/medium/low] [defect|maintainability] <title>
Where: <file:line or component>
Requirement, invariant, or maintenance cost: <...>
Evidence: <reproducer, failing input, concrete code path, or the change it would complicate>
Suggested direction: <optional>
```

Check each premise before reporting or acting on it. For example, an ignored `fmt::Result` from formatting integers into a `String` is not an ignored I/O error, and `process::exit` called after `run()` returns does not skip destructors that already ran inside `run()`. Maintainability findings are judgments; weigh them against the cost and risk of changing working code, and do not let a restructuring change behavior outside the request. For a confirmed defect, explain how the fix restores the violated requirement or invariant and add a regression check where feasible. Reject false or unsupported findings with a concrete reason; distinguish them from valid findings deferred because of scope or tradeoffs. If you and the reviewer still disagree about a material finding, report both positions to the user. After fixes, rerun checks for changed behavior and affected shared paths, including previously valid behavior; earlier passes for those paths are stale. Recheck the integrated artifact at a meaningful boundary. Expand or repeat review and testing only for unresolved risks, changed assumptions, or repository requirements. Finish when those obligations are resolved, without numeric quality scores or a prescribed number of review passes.

## Delivery checks

- The requirements checklist is satisfied, or each gap is stated.
- Each feature is wired into the delivered interface; no placeholder, duplicate, or unreachable implementation remains.
- Final checks pass on the final revision, with affected checks (including doctests when examples change) rerun after the last edit.
- The delivered artifact works from a clean location: installed binary, packaged crate (`cargo package`), or container/service configuration.
- Applicable README commands and examples were run; platform-specific or unavailable deployment steps are clearly marked unverified.
- Inspect `git status` and `cargo package --list` (for Cargo packages) for unintended scratch files, task notes, trial logs or data, and secrets. Retain intentional test fixtures and regression corpora.
- The report distinguishes checks that ran from checks that were not possible here, such as missing tools, platforms, or credentials.
