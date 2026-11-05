# Unsafe code and FFI

Read this before writing or changing `unsafe` code, `extern` declarations, or anything that crosses a language boundary.

## When unsafe is justified

Use it for FFI, for a safe abstraction that neither std nor a maintained crate provides, or for a measured performance need that safe code cannot meet. Check the safe options first: iterators and slice patterns often remove bounds checks, and `bytemuck` or `zerocopy` cover most byte reinterpretation. If unsafe stays, encapsulate it in a small module. A safe API must uphold the invariants for every safe caller. Where foreign pointer validity or lifetime cannot be checked, expose an `unsafe` boundary with explicit caller obligations instead of pretending a null check makes it safe.

## Writing unsafe code

- The safety boundary is the module, not the block. Any code that can modify the fields an `unsafe` block relies on is part of the safety argument, so keep those fields private.
- Give every `unsafe` block a `// SAFETY:` comment naming the invariants it relies on and why they hold here. In a `# Safety` section, state the caller's obligations for each `unsafe fn` and the implementer's obligations for each `unsafe trait`. Enable `clippy::undocumented_unsafe_blocks`. Edition 2024 already warns on unsafe operations inside `unsafe fn` without an inner `unsafe` block.
- Common sources of undefined behavior:
  - creating a reference (`&`/`&mut`) to uninitialized, misaligned, or dangling memory. Use `MaybeUninit<T>` for uninitialized storage and `&raw const` / `&raw mut` when forming a reference would be invalid. `read_unaligned` relaxes alignment only: the pointer must still be valid for reads and point to an initialized `T`;
  - aliasing violations: violating exclusive access through a live `&mut` (valid reborrows are allowed; merely having another pointer variable is not itself UB). Follow the current [aliasing rules](https://doc.rust-lang.org/reference/behavior-considered-undefined.html), rather than assuming a complete formal model;
  - invalid values: a `bool` other than 0 or 1, an out-of-range `char`, an enum with an unlisted discriminant, a null reference, or a null `NonNull<T>`. Never transmute unvalidated integers into such types;
  - `slice::from_raw_parts` with a null or misaligned pointer, even for length 0 (use `NonNull::dangling()`), or with a total size above `isize::MAX` bytes; the range must also lie within one allocation and contain initialized valid elements;
  - calling `Vec::set_len` beyond capacity or over uninitialized elements; using references after their pointee's lifetime ends; data races through shared raw pointers.

## FFI boundaries

- Calling C: consider generated declarations with `bindgen`, in `build.rs` or pre-generated and committed, and compile bundled C/C++ with the `cc` crate. Exposing Rust to C: `extern "C"` functions with `#[unsafe(no_mangle)]` (or `#[no_mangle]` when targeting an MSRV before Rust 1.82) and headers from `cbindgen`. C++: prefer `cxx` for the types it supports over hand-written `extern "C"` shims.
- Edition 2024 syntax: `unsafe extern "C" { ... }`; items inside may be declared `safe fn` when calling them is always sound.
- Types: `#[repr(C)]` on shared structs; `std::ffi::{c_int, c_char, c_void, CStr, CString}`. Receive C enums as integers and convert with `TryFrom`, never as Rust enums. Use `bool` only where the C side uses `bool`/`_Bool`.
- Strings: `CString::new(s)?.as_ptr()` passed directly to a call is fine only if nothing keeps the pointer afterwards; storing it creates a dangling pointer as soon as the temporary drops. Bind the `CString` to a variable that outlives every use. `CStr::from_ptr` requires a valid NUL-terminated string that lives as long as the borrow.
- Ownership: memory is freed by the allocator that allocated it. Rust-allocated objects handed to C need an exported `*_free` function (`Box::into_raw` / `Box::from_raw`). Document who owns every pointer, and validate what can be checked. Null checks do not establish alignment, allocation size, initialization, aliasing, or lifetime.
- Panics: an unwinding Rust panic crossing a non-unwind ABI such as `extern "C"` aborts; a foreign exception entering Rust across it is undefined behavior. If the contract requires error recovery, catch Rust unwinding panics *inside* the boundary and restore invariants before returning an error. `catch_unwind` does not catch `panic=abort`, cannot repair corrupted state, and is not general foreign-exception handling. An intentional abort policy or a non-panicking entry point does not need a ceremonial catch. Use `extern "C-unwind"` only with a compatible unwinding design. See [FFI and unwinding](https://doc.rust-lang.org/nomicon/ffi.html#ffi-and-unwinding).
- Threads and callbacks: find out whether the foreign library is thread-safe and reentrant. Encode the answer in `Send`/`Sync` (raw pointers make a wrapper `!Send`/`!Sync` by default; add `unsafe impl` only with a documented reason). Callbacks use `extern "C" fn` trampolines with a `*mut c_void` context whose lifetime covers every possible invocation.

## Verifying unsafe and FFI code

- Run Miri on the pure-Rust unsafe code: `cargo +nightly miri test`, which needs the nightly toolchain with the `miri` component. Miri supports some foreign-function shims but cannot generally execute arbitrary native code; isolate the Rust logic and report the tested boundary.
- For FFI code, run the tests under AddressSanitizer (nightly `-Zsanitizer=address`; see the rustc unstable book for current flags) or Valgrind on the built test binary.
- Fuzz or property-test the safe API around the unsafe core with adversarial inputs, since the safe API must be sound for all of them.
- In review, check every `SAFETY:` comment against the code. A missing or vague justification is a finding.
