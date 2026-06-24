# LoongArch64 Adaptation Patches for rules_rust 0.42.1

This directory contains staged local patches for adapting `rules_rust` 0.42.1 to `loongarch64-unknown-linux-gnu`.

## Stage 1: platforms

`0001-platforms-0.0.8-add-loongarch64-cpu.patch`

Use this only if you must keep `bazelbuild/platforms` at 0.0.8. It adds `@platforms//cpu:loongarch64`, which rules_rust needs when translating Rust triples to Bazel constraints.

`0002-platforms-1.1.0-host-detect-loongarch64.patch`

Use this if your workspace uses `@platforms//host` on a LoongArch64 host. `platforms 1.1.0` already has `cpu:loongarch64`, but its host extension does not currently translate `loongarch64` or `loong64` host arch strings.

For the rules_rust patch below, the preferred route is upgrading the platforms dependency to 1.1.0, so `0001` is documented as a fallback rather than the default path.

## Stage 2: rules_rust

`0003-rules_rust-0.42.1-loongarch64.patch`

This patch:
- upgrades the `platforms` dependency from 0.0.8 to 1.1.0;
- registers a default Linux LoongArch64 Rust toolchain repository;
- adds `loongarch64-unknown-linux-gnu` to supported platform triples;
- maps `loongarch64` to `@platforms//cpu:loongarch64`;
- allows LoongArch64 Linux host triple detection;
- adds Rust 1.77.2 LoongArch64 `.tar.xz` SHA256 values from the official Rust channel manifest.

Rust itself does not need to be upgraded for this adaptation: the official `channel-rust-1.77.2.toml` includes `cargo`, `clippy-preview`, `llvm-tools-preview`, `rust`, `rust-std`, `rustc`, and `rustfmt-preview` for `loongarch64-unknown-linux-gnu`.

## Stage 3: crate_universe

No patch is currently needed for `crate_universe`: `cfg-expr 0.15.5` already contains `loongarch64-unknown-linux-gnu` in its built-in target table.

## Stage 4: Follow-up From LoongArch64 Validation

`0004-loongarch64-followup-test-and-platforms-deps.patch`

This patch addresses the first LoongArch64 validation feedback:
- `//test/load_arbitrary_tool:all` used Rust `1.53.0`, which has no LoongArch64 host cargo artifact. The test now keeps `1.53.0` for existing hosts and uses `1.77.2` only on LoongArch64.
- Upgrading `platforms` to `1.1.0` makes its root `BUILD` load `@package_metadata`; WORKSPACE-mode `rules_rust_dependencies()` now defines that repository before `@platforms`.

## Stage 5: External Bzlmod Smoke Workspace

`0005-add-loongarch64-bzlmod-smoke-workspace.patch`

This adds `loongarch64_adaptation/smoke_bzlmod`, a small external workspace that uses a local override back to the patched rules_rust checkout. It validates the next layer after repository-internal tests:
- Bzlmod module extension wiring;
- LoongArch64 Rust toolchain registration and use;
- direct `rust_library`, `rust_test`, and `rust_binary`;
- `cargo_build_script` execution and `cargo:rustc-env` propagation;
- `crate_universe.from_specs` with a third-party crate.

`0006-loongarch64-smoke-explicit-platform.patch`

This follow-up makes the smoke workspace declare an explicit `//:loongarch64_linux` platform and adds a local `.bazelrc` that sets the LoongArch64 host, target, and execution platforms.

Reason: the smoke workspace should not depend on Bazel's implicit host/default platforms carrying `@platforms//cpu:loongarch64`. The Rust toolchains registered by rules_rust are constraint-based on both execution and target platforms, so the test must provide platform constraints that match the LoongArch64 toolchain.

## Stage 6: External WORKSPACE Smoke Workspace

`0007-add-loongarch64-workspace-smoke-workspace.patch`

This adds `loongarch64_adaptation/smoke_workspace`, a small external WORKSPACE-mode project that uses a local repository back to the patched rules_rust checkout. It validates the legacy macro path after the Bzlmod path:
- `rules_rust_dependencies()`;
- `rust_register_toolchains()`;
- `crate_universe_dependencies(bootstrap = True)`;
- `crates_repository()` with `supported_platform_triples = ["loongarch64-unknown-linux-gnu"]`;
- `rust_library`, `rust_test`, `rust_binary`, and `cargo_build_script`.

Reason: rules_rust 0.42.1 still supports WORKSPACE users. The earlier `package_metadata` fix specifically affects WORKSPACE-mode dependency setup after upgrading `platforms` to 1.1.0, so this path needs an explicit external validation.

## Stage 7: cargo-bazel Vendored Crate Platforms

`0008-add-loongarch64-to-cargo-bazel-vendored-crates.patch`

This patch adds `@rules_rust//rust/platform:loongarch64-unknown-linux-gnu` to the `target_compatible_with` platform selects in all generated `crate_universe/3rdparty/crates/BUILD.*.bazel` files.

Reason: `//crate_universe:cargo_bazel` is built from rules_rust's checked-in vendored crate BUILD files. Those files were generated before LoongArch64 was part of `SUPPORTED_PLATFORM_TRIPLES`, so every vendored crate falls through to `@platforms//:incompatible` on a LoongArch64 host. The first observed failure is `@cui__anyhow-1.0.75//:anyhow`, but the whole generated dependency set needs the same platform entry.

## Stage 8: cargo-bazel Vendored cfg Selects

`0009-add-loongarch64-to-cargo-bazel-vendored-selects.patch`

This patch adds `@rules_rust//rust/platform:loongarch64-unknown-linux-gnu` to generated vendored `select()` expressions that model generic Unix/Linux/not-windows cfg dependencies and crate features.

Reason: after `0008`, `//crate_universe:cargo_bazel` can analyze more vendored crates on LoongArch64, but some crates still fall through to empty dependency branches. The observed `getrandom-0.2.10` failure is caused by compiling Unix libc code without selecting the generated `libc` dependency. The patch avoids copying branches whose cfg is explicitly for other CPU architectures.

## LoongArch64 Verification

Run these on a LoongArch64 Linux machine after applying `0003` to rules_rust 0.42.1:

```bash
git apply loongarch64_adaptation/0003-rules_rust-0.42.1-loongarch64.patch
git apply loongarch64_adaptation/0004-loongarch64-followup-test-and-platforms-deps.patch
git apply loongarch64_adaptation/0005-add-loongarch64-bzlmod-smoke-workspace.patch
git apply loongarch64_adaptation/0006-loongarch64-smoke-explicit-platform.patch
git apply loongarch64_adaptation/0007-add-loongarch64-workspace-smoke-workspace.patch
git apply loongarch64_adaptation/0008-add-loongarch64-to-cargo-bazel-vendored-crates.patch
git apply loongarch64_adaptation/0009-add-loongarch64-to-cargo-bazel-vendored-selects.patch

bazel test //test/unit/platform_triple:all
bazel test //test/load_arbitrary_tool:all
bazel test //crate_universe:all
bazel build //crate_universe:cargo_bazel
bazel run //crate_universe:cargo_bazel -- --help

cd loongarch64_adaptation/smoke_bzlmod
bazel sync
bazel test //:smoke_test --host_platform=//:loongarch64_linux --platforms=//:loongarch64_linux
bazel run //:smoke_bin --host_platform=//:loongarch64_linux --platforms=//:loongarch64_linux

cd ../smoke_workspace
CARGO_BAZEL_REPIN=1 CARGO_BAZEL_REPIN_ONLY=crates bazel sync --only=crates
bazel sync
bazel test //:smoke_test
bazel run //:smoke_bin
```

Expected result: repository setup should detect `loongarch64-unknown-linux-gnu`, download Rust 1.77.2 LoongArch64 host tools and stdlib, and resolve the Rust toolchain for a LoongArch64 Linux target.
