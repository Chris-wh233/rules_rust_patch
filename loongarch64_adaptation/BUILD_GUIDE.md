# rules_rust 0.42.1 LoongArch64 适配指南

## 1. 项目背景
- 目标项目：`bazelbuild/rules_rust` 0.42.1。
- 目标架构：LoongArch64 原生 Linux，Rust triple 为 `loongarch64-unknown-linux-gnu`。
- 适配目的：为后续构建 Kong 3.9.1 提供可用的 `rules_rust` 0.42.1 和 `cargo-bazel` 0.12.0。
- 适配边界：当前以 LoongArch64 原生构建可用为目标，不处理交叉构建完整矩阵，也不声明 rules_rust 全量测试已在 LoongArch64 上完成。

## 2. 构建/适配过程概览
- 第一阶段：补齐 Bazel `platforms` 对 LoongArch64 CPU 约束和 host 检测能力。
- 第二阶段：补齐 rules_rust 对 LoongArch64 host triple、Rust 平台映射、默认工具链仓库和 Rust 1.77.2 artifact SHA 的支持。
- 第三阶段：修复 rules_rust 单元测试和 WORKSPACE 模式下 `platforms` 1.1.0 引入的 `package_metadata` 依赖。
- 第四阶段：增加 Bzlmod/WORKSPACE smoke workspace，验证 LoongArch64 平台下基本 Rust 编译链路。
- 第五阶段：转向 Kong 实际需要的 `cargo-bazel`，修复 crate_universe vendored crates 的平台兼容性、依赖 select 和 crate feature/cfg 问题。
- 第六阶段：构建并验证 `cargo_bazel_bin`，得到可供 Kong 通过 `CARGO_BAZEL_GENERATOR_URL` 使用的 LoongArch64 cargo-bazel 可执行文件。

## 3. 问题排查与解决过程

### 3.1 Bazel platforms 缺少 LoongArch64 CPU 约束
- **现象**：rules_rust 的 Rust platform label 需要映射到 Bazel CPU constraint，但旧版 `@platforms` 不包含 LoongArch64。
- **相关环境**：rules_rust 0.42.1 默认依赖 `platforms` 0.0.8。
- **排查过程**：
  - 检查 rules_rust 的 `rules_rust_dependencies()`，发现 `@platforms` 固定为 0.0.8。
  - 检查 LoongArch64 platform label 映射，确认需要 `@platforms//cpu:loongarch64`。
- **原因分析**：旧版 platforms 不提供 LoongArch64 CPU constraint，导致 rules_rust 即使添加 triple，也无法生成有效 Bazel platform 约束。
- **解决方案**：
  - `0001-platforms-0.0.8-add-loongarch64-cpu.patch` 用于说明旧 platforms 补丁路径。
  - 实际 rules_rust 适配中采用升级到 `platforms` 1.1.0 的方案，因为该版本已有 LoongArch64 CPU 约束。
- **验证结果**：后续 rules_rust 平台 triple 测试和 smoke 测试均能解析 LoongArch64 platform。

### 3.2 rules_rust 不识别 LoongArch64 host triple
- **现象**：LoongArch64 机器上 rules_rust 无法作为受支持 Linux host 架构完成 triple 检测和工具链注册。
- **相关环境**：`rust/platform/triple.bzl`、`rust/platform/triple_mappings.bzl`、`rust/platform/platform.bzl`、`rust/repositories.bzl`。
- **排查过程**：
  - 检查 `get_host_triple()`，发现 Linux supported_architectures 只有 `aarch64` 和 `x86_64`。
  - 检查 CPU/triple 映射，发现没有 `loongarch64-unknown-linux-gnu`。
  - 检查 `DEFAULT_TOOLCHAIN_TRIPLES`，发现没有 LoongArch64 默认仓库名。
- **原因分析**：rules_rust 0.42.1 发布时没有 LoongArch64 host 支持，导致 host triple、platform label、toolchain repo name 都无法闭环。
- **解决方案**：
  - 在 `platform.bzl` 添加 `loongarch64` CPU。
  - 在 `triple.bzl` 添加 Linux LoongArch64 host 支持，并将 `loong64` 归一化为 `loongarch64`。
  - 在 `triple_mappings.bzl` 添加 `loongarch64-unknown-linux-gnu` 和 Bazel CPU suffix 映射。
  - 在 `repositories.bzl` 添加 `loongarch64-unknown-linux-gnu` 到 `rust_linux_loongarch64` 的默认工具链映射。
  - 在 `known_shas.bzl` 添加 Rust 1.77.2 LoongArch64 host artifacts SHA。
- **验证结果**：`bazel test //test/unit/platform_triple:all` 通过。

### 3.3 `load_arbitrary_tool` 测试下载不存在的 Rust 1.53.0 LoongArch64 artifact
- **现象**：`bazel test //test/load_arbitrary_tool:all` 报错，尝试获取 `cargo-1.53.0-loongarch64-unknown-linux-gnu`。
- **相关环境**：rules_rust 测试 fixture 固定使用 Rust 1.53.0。
- **排查过程**：
  - 根据错误定位到 `test/load_arbitrary_tool/load_arbitrary_tool_test.bzl`。
  - 对照 Rust 官方发布历史，LoongArch64 host artifact 不存在于 1.53.0。
- **原因分析**：测试本身不是为了验证 LoongArch64 早期 Rust 版本，而是验证 arbitrary tool 下载逻辑；固定旧版本在新架构上不成立。
- **解决方案**：仅在 `host_triple.arch == "loongarch64"` 时将测试 cargo 版本切换为 rules_rust 默认 Rust 1.77.2，其他架构保留原行为。
- **验证结果**：应用 follow-up 补丁后 `bazel test //test/load_arbitrary_tool:all` 通过。

### 3.4 `platforms` 1.1.0 在 WORKSPACE 模式下缺少 `package_metadata`
- **现象**：`bazel test //crate_universe:all` 报错：
  `Repository '@@package_metadata' is not defined`。
- **相关环境**：rules_rust `rules_rust_dependencies()` 升级 `@platforms` 到 1.1.0 后触发。
- **排查过程**：
  - 报错来自 `@platforms//` root package 加载 license metadata。
  - `platforms` 1.1.0 的 BUILD 文件依赖 `@package_metadata//licenses/rules:license.bzl`。
- **原因分析**：Bzlmod 可以由模块依赖补齐，但 WORKSPACE 模式需要 rules_rust 显式声明 `package_metadata` 仓库。
- **解决方案**：在 `rules_rust_dependencies()` 中先注册 `package_metadata` http_archive，再注册 `platforms`。
- **验证结果**：后续 `//crate_universe:all` 能继续进入实际分析/构建阶段。

### 3.5 Bzlmod smoke 测试找不到 LoongArch64 Rust toolchain
- **现象**：`bazel test //:smoke_test` 报错：
  `No matching toolchains found for types @@rules_rust~//rust:toolchain_type`。
- **相关环境**：新增的 LoongArch64 Bzlmod smoke workspace。
- **排查过程**：
  - `bazel sync` 可执行，说明仓库加载阶段不再是主要问题。
  - 失败发生在目标分析期的 toolchain resolution。
- **原因分析**：smoke workspace 未显式声明目标 platform 时，Bazel 没有稳定匹配到 LoongArch64 Rust toolchain。
- **解决方案**：在 smoke workspace 中添加显式 LoongArch64 platform，并通过配置让 `bazel test //:smoke_test` 和 `bazel run //:smoke_bin` 使用该 platform。
- **验证结果**：应用 `0006` 后，`bazel sync`、`bazel test //:smoke_test`、`bazel run //:smoke_bin` 均通过。

### 3.6 `//crate_universe:cargo_bazel` 被判定 incompatible
- **现象**：执行 `bazel build //crate_universe:cargo_bazel` 报错：
  `Target //crate_universe:cargo_bazel is incompatible`，依赖链指向 vendored crate 的 `target_compatible_with`。
- **相关环境**：crate_universe vendored crates 的 `BUILD.*.bazel` 文件。
- **排查过程**：
  - 错误显示依赖如 `@@cui__anyhow-1.0.75//:anyhow` 未满足平台约束。
  - 检查 vendored BUILD 文件，发现 `target_compatible_with = select({ ... })` 没有 LoongArch64 分支。
- **原因分析**：rules_rust 平台层支持 LoongArch64 后，vendored crate 生成文件仍然只认识旧平台，导致 LoongArch64 落入 incompatible。
- **解决方案**：为 crate_universe vendored BUILD 文件的相关 select 增加：
  `@rules_rust//rust/platform:loongarch64-unknown-linux-gnu: []`。
- **验证结果**：incompatible 类错误消除，构建继续推进到具体 crate 编译错误。

### 3.7 `getrandom` 缺少 `libc` 依赖
- **现象**：构建 `cargo_bazel` 时，`external/cui__getrandom-0.2.10` 报错：
  `unresolved import libc`。
- **相关环境**：`getrandom` 的 vendored BUILD 文件和 LoongArch64 平台 select。
- **排查过程**：
  - 报错源码位于 `util_libc.rs` 和 `error.rs`。
  - 这些路径需要 `libc` crate，但 LoongArch64 分支没有进入正确 deps。
- **原因分析**：vendor 生成文件中 Linux/Unix 平台依赖 select 未覆盖 LoongArch64，导致 `libc` 没有进入 Rust target deps。
- **解决方案**：补齐 LoongArch64 到 vendored select 依赖逻辑。
- **验证结果**：`getrandom` 的 `libc` 缺失错误消除。

### 3.8 `rustix` 在 LoongArch64 上的 libc/raw syscall/termios feature 组合问题
- **现象**：
  - 构建错误转移到 `external/cui__rustix-0.38.21`。
  - 修复一部分后又出现 `gix-prompt` 报错：`could not find termios in rustix`。
- **相关环境**：`cargo-bazel` 依赖链中 `tempfile -> rustix`，以及 `gix-prompt -> rustix::termios`。
- **排查过程**：
  - 通过 `bazel query 'somepath(//crate_universe:cargo_bazel, @cui__rustix-0.38.21//:rustix)'` 确认依赖路径为 `cargo_bazel -> tempfile -> rustix`。
  - 初始修复为了消除 rustix 的 LoongArch64 报错去掉/改变了部分 feature，随后 `gix-prompt` 需要的 `termios` 被配置掉。
- **原因分析**：LoongArch64 上 `rustix` 的平台 cfg 和 vendored Bazel deps 没有正确表达。仅去掉 feature 会让下游 crate 失去必要 API，必须同时满足 LoongArch64 libc 路径和 `termios` API 需求。
- **解决方案**：
  - 为 `rustix` 添加 LoongArch64 的 libc 路径适配。
  - 补充 `linux-raw-sys` 相关依赖 select。
  - 恢复 `termios` feature，满足 `gix-prompt`。
- **验证结果**：应用 `0013` 后，`bazel build //crate_universe:cargo_bazel` 成功。

### 3.9 `cargo-bazel` 二进制产物确认
- **现象**：`bazel build //crate_universe:cargo_bazel` 只产出 rlib：
  `bazel-bin/crate_universe/libcargo_bazel-625281954.rlib`。
- **相关环境**：crate_universe 中 library target 与 binary target 分离。
- **排查过程**：
  - `//crate_universe:cargo_bazel` 是库目标，不是可执行文件。
  - 需要构建 `//crate_universe:cargo_bazel_bin` 或运行 `//crate_universe:bin`。
- **原因分析**：Kong 需要的是 cargo-bazel generator 可执行文件，而不是 Rust rlib。
- **解决方案**：执行：
  ```bash
  bazel build //crate_universe:cargo_bazel_bin
  bazel run //crate_universe:bin -- --help
  bazel-bin/crate_universe/cargo_bazel_bin --version
  ldd bazel-bin/crate_universe/cargo_bazel_bin
  ```
- **验证结果**：
  - `bazel run //crate_universe:bin -- --help` 正常输出命令帮助。
  - `--version` 输出 `cargo-bazel 0.12.0`。
  - `ldd` 显示链接到 LoongArch64 Linux 系统库和 `/lib64/ld-linux-loongarch-lp64d.so.1`。

### 3.10 cargo-bazel 体积偏大
- **现象**：LoongArch64 本地构建的 `cargo_bazel_bin` 约 46M，大于官方其它架构常见的 10M-30M。
- **相关环境**：默认 Bazel 构建配置。
- **排查过程**：
  - 当前构建命令未使用 release/opt strip 参数。
  - `.bazelrc` 或默认构建可能保留调试符号。
- **原因分析**：体积偏大主要可能来自默认 fastbuild、未 strip、保留调试符号或未采用官方 release 构建参数，不代表引入了无用逻辑。
- **解决方案**：如需接近发布体积，可使用：
  ```bash
  bazel build -c opt --strip=always //crate_universe:cargo_bazel_bin
  ```
- **验证结果**：该优化构建命令作为建议给出，是否执行和体积变化需在 LoongArch64 机器上确认。

## 4. 最终补丁结构
- `0001-platforms-0.0.8-add-loongarch64-cpu.patch`：旧 platforms 补丁路径记录。
- `0002-platforms-1.1.0-host-detect-loongarch64.patch`：platforms 1.1.0 host 检测补充。
- `0003-rules_rust-0.42.1-loongarch64.patch`：rules_rust 核心 LoongArch64 支持。
- `0004-loongarch64-followup-test-and-platforms-deps.patch`：测试与 `package_metadata` 依赖修复。
- `0005-add-loongarch64-bzlmod-smoke-workspace.patch`：新增 Bzlmod smoke workspace。
- `0006-loongarch64-smoke-explicit-platform.patch`：smoke workspace 显式平台修复。
- `0007-add-loongarch64-workspace-smoke-workspace.patch`：新增 WORKSPACE smoke workspace。
- `0008-add-loongarch64-to-cargo-bazel-vendored-crates.patch`：vendored crates target compatibility 初步补齐。
- `0009-add-loongarch64-to-cargo-bazel-vendored-selects.patch`：vendored select 依赖补齐。
- `0010-cargo-bazel-disable-clap-color-stack.patch`：规避 cargo-bazel 依赖中的 clap/color-stack 相关 LoongArch64 构建问题。
- `0011-cargo-bazel-rustix-0.38-loongarch64-libc-path.patch`：rustix LoongArch64 libc 路径修复。
- `0012-cargo-bazel-rustix-0.38-add-linux-raw-sys-dep.patch`：rustix linux-raw-sys 依赖补齐。
- `0013-cargo-bazel-rustix-0.38-restore-termios-for-gix-prompt.patch`：恢复 `termios` feature 以满足 `gix-prompt`。

## 5. 推荐复现命令
在 LoongArch64 Linux 机器上的 `rules_rust` 0.42.1 源码目录中应用补丁后，建议按以下顺序验证：

```bash
bazel test //test/unit/platform_triple:all
bazel test //test/load_arbitrary_tool:all
bazel test //crate_universe:all
```

smoke workspace 验证：

```bash
bazel sync
bazel test //:smoke_test
bazel run //:smoke_bin
```

构建 Kong 所需 cargo-bazel：

```bash
bazel build //crate_universe:cargo_bazel
bazel build //crate_universe:cargo_bazel_bin
bazel run //crate_universe:bin -- --help
bazel-bin/crate_universe/cargo_bazel_bin --version
ldd bazel-bin/crate_universe/cargo_bazel_bin
```

如需较小二进制：

```bash
bazel build -c opt --strip=always //crate_universe:cargo_bazel_bin
```

## 6. Kong 集成注意点
- Kong 使用 `rules_rust` 0.42.1 和 crate_universe，需要 LoongArch64 可执行的 `cargo-bazel` generator。
- 在 Kong 中不必修改 `rules_rust` 的 `crate_universe/private/urls.bzl` 常量；可通过 rules_rust 0.42.1 已支持的环境变量提供本地 generator：
  ```bash
  export CARGO_BAZEL_GENERATOR_URL=file:///absolute/path/to/cargo_bazel_bin
  ```
- `CARGO_BAZEL_URLS` 是 rules_rust 内部默认 URL 映射；实际外部覆盖入口是 `CARGO_BAZEL_GENERATOR_URL`。

## 7. 遗留问题与后续方向
- 当前 rules_rust 补丁服务于 LoongArch64 原生 Linux 和 Kong 构建路径，尚未整理为适合直接上游提交的最小 patch series。
- 若要上游化，建议重新拆分为：
  - 核心 platform/triple/toolchain 支持；
  - WORKSPACE 模式依赖修复；
  - LoongArch64 测试 fixture 修复；
  - crate_universe vendored 生成文件修复。
- 后续 Kong deb 构建还需要继续处理 Kong 项目自身依赖，包括 LoongArch LuaJIT2、本地 nFPM、WasmX/WebUI 可选组件等。
