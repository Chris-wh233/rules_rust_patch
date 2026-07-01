# Kong LoongArch64 Adaptation Patches

This directory stores local patch notes and patch files for adapting Kong 3.9.1
to LoongArch64.

Scope:
- Kong version: 3.9.1
- Source checkout: `../kong`
- Patch handoff repository: `rules_rust_patch`

Workflow:
- Put Kong-specific patches in this directory.
- Update this README when a new Kong adaptation stage is added.
- Do not push changes to the remote repository unless explicitly requested.

## 0001-kong-3.9.1-loongarch64-native-deb-bootstrap.patch

Purpose:
- Enable a native LoongArch64 Linux build path for Kong 3.9.1 deb packaging.
- Keep the patch focused on the deb path instead of claiming complete support
  for all optional Kong build features.

Changes:
- Adds `//:generic-loongarch64` and `//:loongarch64-linux` so Kong BUILD files
  can select LoongArch64-specific inputs.
- Adds `LOONGARCH_LUAJIT2=/opt/luajit2` and selects `@loongarch_luajit2` as
  the LuaJIT source on LoongArch64. This is needed because the OpenResty
  bundled LuaJIT does not provide the required LoongArch backend.
- Adds `NFPM_BIN` support to `@nfpm`, because upstream nFPM does not publish a
  LoongArch64 binary. The package rule maps Bazel LoongArch CPUs to Debian's
  `loong64` architecture name.
- Adds Rust 1.82.0 LoongArch64 host artifacts to Kong's `rust_register_toolchains`.
- Patches rules_rust 0.42.1 minimally for LoongArch64 host/triple/platform
  support. This patch intentionally does not vendor or build cargo-bazel inside
  rules_rust.

Cargo-bazel:
- Do not patch rules_rust's `crate_universe/private/urls.bzl` for Kong.
- Provide the LoongArch64 cargo-bazel binary through the existing
  `CARGO_BAZEL_GENERATOR_URL` repository environment variable.
- `CARGO_BAZEL_URLS` is an internal default map; in rules_rust 0.42.1 the
  documented external override for `crates_repository` is
  `CARGO_BAZEL_GENERATOR_URL`.

Known first-stage limitations:
- Build with `--//:wasmx=false` unless a LoongArch64 Wasm runtime package is
  added later. Kong 3.9.1 only declares x86_64/aarch64 Wasm runtime archives.
- Build with `--//:skip_webui=true` unless the Kong Manager asset download path
  is adapted. Its GitHub CLI helper only selects x86_64/aarch64 host binaries.

Suggested LoongArch64 verification:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel sync
bazel build --config=release --platforms=//:generic-loongarch64 --//:wasmx=false --//:skip_webui=true //build:kong
bazel build --config=release --platforms=//:generic-loongarch64 --//:wasmx=false --//:skip_webui=true //:kong_deb
```

Expected package output:

```text
bazel-bin/pkg/kong.loong64.deb
```

## 0002-kong-pin-platforms-1.1.0-for-loongarch64.patch

Purpose:
- Fix `@platforms//cpu:loongarch64` not found during Kong analysis.

Reason:
- `bazel sync` is not required before every `bazel build`; build fetches
  repositories on demand.
- The failure is caused by Kong's WORKSPACE loading helper macros before
  rules_rust can declare its patched `@platforms` dependency. An older
  transitive `@platforms` repository can therefore win, and old releases do
  not define `cpu:loongarch64`.

Change:
- Declare `@platforms` 1.1.0 explicitly near the top of Kong's WORKSPACE,
  before `bazel_skylib_workspace()` and other workspace helper macros.

After applying this patch, retry:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build --config=release --platforms=//:generic-loongarch64 --//:wasmx=false --//:skip_webui=true //build:kong
```

## 0005-libxcrypt-4.4.27-add-loongarch64-config.patch

Purpose:
- Let the bundled `cross_deps_libxcrypt` source configure on native
  LoongArch64 Linux.

Reason:
- `libxcrypt-4.4.27` carries old autotools config files.
- On LoongArch64, `config.guess` cannot infer the build type and `config.sub`
  rejects `loongarch64-unknown-linux-gnu`.

Change:
- Add `build/cross_deps/libxcrypt/002-4.4.27-add-loongarch64-config.patch`.
- Apply it from `build/cross_deps/libxcrypt/repositories.bzl`.
- The patch adds `loongarch64:Linux:*:*` detection to `config.guess` and adds
  `loongarch64` to `config.sub`'s recognized CPU list.

After applying, clear the already-fetched `cross_deps_libxcrypt` repository or
start from a clean external cache:

```bash
bazel clean --expunge
```

Then retry the Kong build:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build \
  --noenable_bzlmod \
  --config=release \
  --platforms=//:generic-loongarch64 \
  --//:wasmx=false \
  --//:skip_webui=true \
  //build:kong
```

## 0006-libxcrypt-add-loongarch64-minver.patch

Purpose:
- Avoid libxcrypt 4.4.27 failing with
  `libxcrypt port to loongarch64-unknown-linux-gnu is incomplete`.

Reason:
- After 0005, libxcrypt reaches configure successfully on LoongArch64.
- It then tries to compute compatibility symbol versions for the obsolete
  `libcrypt.so.1` ABI, but `lib/libcrypt.minver` has no LoongArch64 entry and
  intentionally fails.
- Newer upstream libxcrypt releases define LoongArch64 as `GLIBC_2.36` for the
  64-bit lp64* ABI.
- `libcrypt.map.in` must also list `GLIBC_2.36` in its `%chain`; otherwise
  `config.status` fails while generating symbol-version makefile fragments.

Change:
- Add `build/cross_deps/libxcrypt/003-4.4.27-add-loongarch64-minver.patch`.
- Apply it from `build/cross_deps/libxcrypt/repositories.bzl`.
- The patch backports the upstream LoongArch64 `GLIBC_2.36` minver entry so
  libxcrypt 4.4.27 can keep building obsolete-compatible `libcrypt.so.1`.
- The patch also adds `GLIBC_2.36` to the symbol-version chain in
  `lib/libcrypt.map.in`.

If Bazel still uses the previously fetched old `@platforms`, force that
repository to refresh:

```bash
bazel sync --only=platforms
```

## 0007-libxcrypt-disable-werror-on-loongarch64.patch

Purpose:
- Let libxcrypt 4.4.27 continue compiling on LoongArch64 when newer GCC emits
  `-Wunterminated-string-initialization` for `lib/util-base64.c`.

Reason:
- After 0006, libxcrypt gets past configure and reaches compilation.
- The current failure is:
  `cc1: all warnings being treated as errors`.
- This is not a sign that 0006 should be reverted. It means the next build
  layer is now exposed.
- `configure` supports `--disable-werror`, so use the package's own
  compatibility switch instead of patching bundled C source.

Change:
- Add `--disable-werror` only when building `@kong//:loongarch64-linux`.
- Other platforms keep the original warning policy.

Retry after applying:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build \
  --noenable_bzlmod \
  --config=release \
  --platforms=//:generic-loongarch64 \
  --//:wasmx=false \
  --//:skip_webui=true \
  //build:kong \
  --sandbox_debug --verbose_failures
```

## 0008-build-system-support-apparent-and-canonical-label-template-vars.patch

Purpose:
- Fix `luarocks_exec.sh` generation leaving placeholders such as
  `{{@@libexpat//:libexpat}}` unexpanded.

Reason:
- `@libexpat` is already listed as a `srcs` dependency of
  `@luarocks//:luarocks_exec`, so this is not a missing-dependency error.
- The generated script tries to access a literal path under
  `execroot/kong/{{@@libexpat//:libexpat}}/...`, proving that template
  substitution failed before the shell script looked for `libexpat`.
- Different Bazel versions/configurations can stringify labels as either
  apparent repository labels (`@repo//:target`) or canonical repository labels
  (`@@repo//:target`).

Change:
- Teach `kong_template_genrule` rendering to register both placeholder spellings
  for labels that start with `@`.
- Keep the existing templates unchanged, including the current `@@...`
  placeholders.

Retry after applying:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build \
  --noenable_bzlmod \
  --config=release \
  --platforms=//:generic-loongarch64 \
  --//:wasmx=false \
  --//:skip_webui=true \
  //build:kong \
  --sandbox_debug --verbose_failures
```

## 0009-luarocks-force-lock-for-stale-tree-lock.patch

Purpose:
- Allow `luarocks_make` to recover from a stale `luarocks_tree/lockfile.lfs`
  left by a previous failed or interrupted Bazel action.

Reason:
- The observed failure is:
  `command 'make' requires exclusive write access ... try --force-lock`.
- Kong already orders `luarocks_make` after `luarocks_target` to avoid the
  known concurrent-write issue.
- In this adaptation flow, repeated failed builds can leave LuaRocks' lock file
  in Bazel's output tree, causing retries to fail before actual rock
  compilation proceeds.

Change:
- Run `luarocks make --no-doc --force-lock`.

Retry after applying:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build \
  --noenable_bzlmod \
  --config=release \
  --platforms=//:generic-loongarch64 \
  --//:wasmx=false \
  --//:skip_webui=true \
  //build:kong \
  --sandbox_debug --verbose_failures
```

## 0010-deb-drop-obsolete-libpcre3-dependency.patch

Purpose:
- Allow the generated deb to install on newer Debian bases such as
  `debian:forky-slim`, where `libpcre3` is not available.

Reason:
- `libpcre3` is Debian/Ubuntu's old PCRE 1.x runtime package.
- Kong 3.9.1 uses PCRE2 10.44.
- The Bazel PCRE target builds `libpcre2-8.a`, and Kong's manifest checks
  expect `/usr/local/openresty/nginx/sbin/nginx` to contain PCRE2 symbols while
  not requiring dynamic `libpcre` or `libpcre2` shared libraries.
- Therefore the stale `libpcre3` deb dependency should be removed, not replaced
  with `libpcre2-8-0`.

After applying, rebuild the deb:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build \
  --noenable_bzlmod \
  --config=release \
  --platforms=//:generic-loongarch64 \
  --//:wasmx=false \
  --//:skip_webui=true \
  //:kong_deb
```

## 0011-nfpm-depend-on-kong-install-tree.patch

Purpose:
- Make `bazel build //:kong_deb` work from a clean Bazel output tree.

Reason:
- `//:kong_deb` should build the Kong install tree automatically.
- The original `nfpm_pkg` action only created `nfpm-prefix` as a symlink to
  `bazel-out/.../build/kong-dev`, but did not declare `//build:kong` as an
  input.
- After `bazel clean --expunge`, nFPM can run before the install tree exists
  and fail with `matching "./nfpm-prefix/bin": file does not exist`.

Change:
- Add `//build:kong` as a hidden input of `nfpm_pkg`.
- Add the nFPM config file to the action inputs as well.

After applying, this should work directly from a clean output tree:

```bash
bazel clean --expunge

export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build \
  --noenable_bzlmod \
  --config=release \
  --platforms=//:generic-loongarch64 \
  --//:wasmx=false \
  --//:skip_webui=true \
  //:kong_deb
```

## 0012-pdk-nginx-support-loongarch64-ffi-arch.patch

Purpose:
- Fix runtime startup logging `Unsupported arch: loongarch64` from
  `kong/pdk/nginx.lua`.

Reason:
- This is a Kong Lua runtime architecture whitelist issue, not an nginx binary
  support issue.
- LoongArch64 LuaJIT reports `ffi.arch == "loongarch64"`.
- `kong/pdk/nginx.lua` selects Nginx statistic counter pointer width based on
  `ffi.arch`; it only recognized `x64` and `arm64` as 64-bit architectures.

Change:
- Treat `loongarch64` as a 64-bit architecture and use the existing `uint64_t *`
  counter definitions.

Runtime verification:

```bash
docker run --rm \
  -e KONG_DATABASE=off \
  kong:3.9.1 kong docker-start
```

The direct `docker run kong:3.9.1` command still expects PostgreSQL unless
`KONG_DATABASE=off` or a PostgreSQL service is configured.

## 0003-repin-atc-router-lock-for-loongarch64.patch

Purpose:
- Update `crate_locks/atc_router.lock` after enabling LoongArch64 in
  rules_rust/crate_universe.

Reason:
- `crates_repository` checks that the checked-in lockfile checksum matches the
  current generator config, splicing manifest, Cargo, and Rust toolchain.
- Adding `loongarch64-unknown-linux-gnu` changes the generated platform mapping,
  so Kong's original `atc_router.lock` checksum no longer matches.

How it was generated on LoongArch64:

```bash
CARGO_BAZEL_REPIN=true \
CARGO_BAZEL_REPIN_ONLY=atc_router_crate_index \
bazel sync --noenable_bzlmod --only=atc_router_crate_index
```

Expected effect:
- `cfg(all(target_arch = "loongarch64", target_os = "linux"))` now maps to
  `loongarch64-unknown-linux-gnu`.
- The lockfile checksum is updated to the LoongArch-aware generated state.
- The checksum is generated with `--noenable_bzlmod`, matching the Kong build
  command used for this adaptation.

## 0004-patch-platforms-loongarch64-cpu.patch

Purpose:
- Make Kong's explicit `@platforms` repository provide
  `@platforms//cpu:loongarch64`.

Reason:
- Pinning `@platforms` to 1.1.0 in WORKSPACE is not sufficient in the observed
  Kong build environment: Bazel still reports that `cpu:loongarch64` is missing.
- Patch the `@platforms` archive directly so Kong's `//:generic-loongarch64`
  platform can be analyzed.

Change:
- Add `third_party/platforms_loongarch64.patch`.
- Apply it from the top-level `@platforms` `http_archive`.
- The patch adds the LoongArch64 CPU constraint and maps host `loongarch64` /
  `loong64` to that CPU in `host/extension.bzl`.

After applying this patch, clear the already-fetched stale `@platforms` repo:

```bash
bazel clean --expunge
```

Then retry:

```bash
export NFPM_BIN=/path/to/nfpm
export CARGO_BAZEL_GENERATOR_URL=file:///path/to/cargo_bazel_bin

bazel build --config=release --platforms=//:generic-loongarch64 --//:wasmx=false --//:skip_webui=true //build:kong
```
