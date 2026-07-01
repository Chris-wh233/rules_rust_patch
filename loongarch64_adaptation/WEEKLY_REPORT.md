# rules_rust 0.42.1 LoongArch64 适配周报

## 本周主要进展
- 完成 `rules_rust` 0.42.1 对 LoongArch64 原生 Linux 环境的阶段性适配，补丁集整理为 `0001` 到 `0013`。
- 修复 Bazel 平台识别、Rust host triple、工具链下载、WORKSPACE/Bzlmod smoke 测试、crate_universe vendored crates 兼容性等问题。
- 在 LoongArch64 机器上完成 `cargo-bazel` 构建验证，得到可执行产物 `bazel-bin/crate_universe/cargo_bazel_bin`，版本为 `cargo-bazel 0.12.0`。
- 已将补丁和说明同步到 `rules_rust_patch/loongarch64_adaptation`，并推送过当前 rules_rust 适配成果。

## 主要问题与处理
- **问题**：`rules_rust` 0.42.1 默认不识别 LoongArch64 host triple，平台映射和默认工具链表中缺少 `loongarch64-unknown-linux-gnu`。
  **分析**：`rust/platform/triple.bzl`、`triple_mappings.bzl`、`platform.bzl`、`repositories.bzl` 都只覆盖了既有主流架构；Bazel `@platforms` 旧版本也缺少 LoongArch64 CPU 约束。
  **解决**：添加 LoongArch64 CPU/triple 映射、默认工具链仓库名、Rust 1.77.2 LoongArch64 host artifact SHA，并将 `@platforms` 升级到 1.1.0。

- **问题**：`bazel test //test/load_arbitrary_tool:all` 在 LoongArch64 上尝试下载 Rust 1.53.0 的 LoongArch64 cargo artifact，导致失败。
  **分析**：Rust 1.53.0 发布时间早于 LoongArch64 官方 host artifact 支持，不能下载 `cargo-1.53.0-loongarch64-unknown-linux-gnu`。
  **解决**：保留原有架构的 1.53.0 测试逻辑，仅 LoongArch64 改用 rules_rust 默认 Rust 1.77.2。

- **问题**：`bazel test //crate_universe:all` 触发 `@platforms//` 加载失败，提示 `@package_metadata` 未定义。
  **分析**：升级到 `platforms` 1.1.0 后，其 root BUILD 文件会加载 `@package_metadata`，WORKSPACE 模式下 rules_rust 需要显式注册该依赖。
  **解决**：在 `rules_rust_dependencies()` 中补充 `package_metadata` http_archive。

- **问题**：Bzlmod/WORKSPACE smoke 测试起初无法解析 LoongArch64 Rust toolchain。
  **分析**：测试工作区未显式设置目标平台时，Bazel toolchain resolution 不能稳定选择新加的 LoongArch64 平台。
  **解决**：新增 smoke workspace，并在测试配置中显式设置 `--platforms=//:loongarch64_linux`，验证 `bazel sync`、`bazel test //:smoke_test`、`bazel run //:smoke_bin` 均通过。

- **问题**：`bazel build //crate_universe:cargo_bazel` 起初因 vendored crates 的 `target_compatible_with` 缺少 LoongArch64 分支而被判定 incompatible。
  **分析**：crate_universe 的 vendored BUILD 文件中大量 select 只列出既有平台，LoongArch64 未匹配时落入 incompatible 分支。
  **解决**：批量为 vendored crates 的平台 select 添加 `@rules_rust//rust/platform:loongarch64-unknown-linux-gnu`。

- **问题**：继续构建 cargo-bazel 时，`getrandom`、`rustix` 等 crate 在 LoongArch64 上出现 `libc`、`linux_raw_sys`、`termios` 相关编译错误。
  **分析**：这些错误不是 rules_rust 核心源码错误，而是 vendored crate 生成的 Bazel 依赖和 cfg/feature 选择不完整。`rustix` 在 LoongArch64 上需要走 libc 路径，同时仍要保留 `termios` feature 供 `gix-prompt` 使用。
  **解决**：补充 vendored select 依赖，调整 cargo-bazel 相关 crate 的 LoongArch64 条件，恢复 `termios` feature，最终 `//crate_universe:cargo_bazel` 与 `//crate_universe:cargo_bazel_bin` 构建成功。

## 验证结果
- `bazel test //test/unit/platform_triple:all`：通过。
- `bazel test //test/load_arbitrary_tool:all`：应用 follow-up 补丁后通过。
- `bazel test //crate_universe:all`：应用 follow-up 补丁后通过。
- `bazel sync`、`bazel test //:smoke_test`、`bazel run //:smoke_bin`：smoke workspace 修复后通过。
- `bazel build //crate_universe:cargo_bazel`：应用 `0013` 后通过。
- `bazel build //crate_universe:cargo_bazel_bin`：通过。
- `bazel-bin/crate_universe/cargo_bazel_bin --version`：输出 `cargo-bazel 0.12.0`。
- `ldd bazel-bin/crate_universe/cargo_bazel_bin`：确认产物为 LoongArch64 Linux 动态可执行文件。

## 遗留问题与方向
- 当前补丁目标是服务 Kong 构建所需的 rules_rust 0.42.1 和 cargo-bazel，不等同于 rules_rust 全功能 LoongArch64 上游化完成。
- 若后续要进入上游贡献，需要拆分补丁粒度，区分核心平台支持、测试修复、crate_universe vendored 生成文件修复，并补充 CI/文档说明。
- Kong 构建阶段仍需在 Kong 项目内处理 OpenResty LuaJIT2、nFPM、WasmX/WebUI 等非 rules_rust 依赖。
