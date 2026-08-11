# KernelSU-Next `ksud.exe` 构建说明

本文档说明如何在当前仓库中构建可用于 PC 侧 `boot-patch` 的 `ksud.exe`，以及怎样准备内嵌资源，使其成为“完整可用”的 Windows 可执行文件。

适用场景：

- 在 Windows 上本地编译 `ksud.exe`
- 将 `ksuinit` 和一个或多个 `*_kernelsu.ko` 一起嵌入 `ksud.exe`
- 切换到指定 `tag` 后重新准备匹配版本的资源并重建

## 1. 基本原理

当前仓库中，`ksud` 在非 Android 平台会走 `cli_non_android` 入口，主要提供这些命令：

- `boot-patch`
- `boot-patch-rmvr`
- `boot-restore`
- `get-sign`
- `supported-kmis`

关键代码位置：

- [userspace/ksud/src/main.rs](../userspace/ksud/src/main.rs)
- [userspace/ksud/src/cli_non_android.rs](../userspace/ksud/src/cli_non_android.rs)
- [userspace/ksud/src/assets.rs](../userspace/ksud/src/assets.rs)
- [userspace/ksud/src/boot_patch.rs](../userspace/ksud/src/boot_patch.rs)

其中资源嵌入规则以 [userspace/ksud/src/assets.rs](../userspace/ksud/src/assets.rs) 为准：

- Android `x86_64` 目标只嵌入 `userspace/ksud/bin/x86_64`
- Android `aarch64` 目标只嵌入 `userspace/ksud/bin/aarch64`
- 非 Android 目标会递归嵌入整个 `userspace/ksud/bin`

同时，PC 侧 `boot-patch` 读取内嵌资源时，会按 `--arch` 选择子目录：

- `ksuinit`：`{arch}/ksuinit`
- 内嵌 LKM：`{arch}/{kmi}_kernelsu.ko`

当前 `--arch` 默认值是 `aarch64`。也就是说：

- 只做常见 Android ARM64 设备的 `boot-patch` 时，至少需要准备 `userspace/ksud/bin/aarch64`
- 如果还要支持 `x86_64`，就必须同时准备 `userspace/ksud/bin/x86_64`

## 2. 什么叫“完整的 `ksud.exe`”

能编译出 `ksud.exe` 本体，不等于它已经完整可用。

对 PC 侧 `boot-patch` 来说，真正关键的内嵌资源至少有：

- `ksuinit`
- 一个或多个 `*_kernelsu.ko`

仓库默认还维护：

- `busybox`
- `bootctl`

但对非 Android 平台的 `boot-patch` 来说，决定“是否能补丁启动镜像”的核心资源是前两类。`busybox` 和 `bootctl` 更偏向 Android 侧运行时配套。

## 3. 资源目录布局

### 3.1 只支持默认 `aarch64`

如果你的目标只是做常见 ARM64 设备的 `boot-patch`，最小目录通常是：

```txt
userspace/ksud/bin/
  aarch64/
    ksuinit
    android12-5.10_kernelsu.ko
    android13-5.10_kernelsu.ko
    android13-5.15_kernelsu.ko
    android14-5.15_kernelsu.ko
    android14-6.1_kernelsu.ko
    android15-6.6_kernelsu.ko
    android16-6.12_kernelsu.ko
```

### 3.2 同时支持 `aarch64` 和 `x86_64`

如果你希望一个 `ksud.exe` 同时覆盖两种架构，则应准备：

```txt
userspace/ksud/bin/
  aarch64/
    ksuinit
    android13-5.10_kernelsu.ko
    ...
  x86_64/
    ksuinit
    android13-5.10_kernelsu.ko
    ...
```

这也是当前 CI 的打包方式，参考：

- [.github/workflows/ksud.yml](../.github/workflows/ksud.yml)
- [.github/workflows/ksuinit.yml](../.github/workflows/ksuinit.yml)
- [.github/workflows/build-lkm.yml](../.github/workflows/build-lkm.yml)

## 4. `ko` 文件命名规则

当前实现通过扫描文件名后缀 `_kernelsu.ko` 来识别可支持的 KMI，因此命名必须符合下面这种格式：

```txt
android12-5.10_kernelsu.ko
android13-5.10_kernelsu.ko
android13-5.15_kernelsu.ko
android14-5.15_kernelsu.ko
android14-6.1_kernelsu.ko
android15-6.6_kernelsu.ko
android16-6.12_kernelsu.ko
```

如果你还要准备 vivo 兼容模块，则文件名格式为：

```txt
android14-6.1_vivo_kernelsu.ko
```

说明：

- `supported-kmis` 会忽略 `_vivo_kernelsu.ko`
- `boot-patch` 在 Android 侧会尝试把 `_vivo_kernelsu.ko` 当作 fallback
- PC 侧按当前实现只直接读取 `{arch}/{kmi}_kernelsu.ko`

## 5. 资源从哪里来

### 5.1 `ksuinit`

来源通常有两种：

1. 本地自行编译
2. 使用目标 `tag` 对应的 CI 或 release 产物

源码位置：

- [userspace/ksuinit](../userspace/ksuinit)

CI 参考：

- [.github/workflows/ksuinit.yml](../.github/workflows/ksuinit.yml)

### 5.2 `kernelsu.ko`

来源也通常有两种：

1. 使用目标 `tag` 对应的 CI 或 release 产物
2. 在 Linux / CI 环境中按 KMI 分别构建

CI 参考：

- [.github/workflows/build-lkm.yml](../.github/workflows/build-lkm.yml)
- [.github/workflows/ddk-lkm.yml](../.github/workflows/ddk-lkm.yml)

如果当前目标只是尽快做出一个可用的 `ksud.exe`，最省事的做法通常是：

1. 先切到目标 `tag`
2. 再拿这个 `tag` 对应的 `ksuinit` 和 `*_kernelsu.ko`
3. 把它们放到正确的 `bin/<arch>/` 目录后再重编

## 6. 先切到目标版本再准备资源

推荐先切到目标发布版本 `tag`，再准备与该版本匹配的资源。这样可以尽量避免“源码是一个版本，`ksuinit`/`ko` 是另一个版本”的混搭问题。

### 6.1 同步并查看 `tag`

```powershell
git fetch origin --tags
git tag -l
```

如果只想筛选某一系列：

```powershell
git tag -l "v3.*"
```

### 6.2 查看某个 `tag` 对应的提交

以 `v3.3.0` 为例：

```powershell
git show -s --format="%H%n%D%n%s" v3.3.0
```

### 6.3 基于 `tag` 新建本地分支

```powershell
git switch -c local/v3.3.0 v3.3.0
```

如果本地还没有这个 `tag`：

```powershell
git fetch origin tag v3.3.0
git switch -c local/v3.3.0 v3.3.0
```

### 6.4 确认当前版本

```powershell
git status --short --branch
git show -s --format="%H%n%D%n%s" HEAD
```

## 7. 构建 `ksuinit`

如果你已经拿到现成的 `ksuinit` 产物，可以跳过本节。

### 7.1 `aarch64` 版本

先安装 target：

```powershell
rustup target add aarch64-unknown-linux-musl
```

使用 Android NDK 的 clang 作为 linker，示例：

```powershell
$clang = Get-ChildItem $env:ANDROID_NDK_HOME -Recurse -Filter aarch64-linux-android26-clang.cmd -ErrorAction SilentlyContinue |
  Select-Object -First 1 -ExpandProperty FullName

$env:CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_LINKER = $clang
$env:RUSTFLAGS = "-C link-arg=-Wno-unused-command-line-argument"

cargo build --target aarch64-unknown-linux-musl --release --manifest-path userspace/ksuinit/Cargo.toml
```

产物通常位于：

- [userspace/ksuinit/target/aarch64-unknown-linux-musl/release/ksuinit](../userspace/ksuinit/target/aarch64-unknown-linux-musl/release/ksuinit)

复制到资源目录：

```powershell
Copy-Item userspace\ksuinit\target\aarch64-unknown-linux-musl\release\ksuinit userspace\ksud\bin\aarch64\ksuinit -Force
```

### 7.2 `x86_64` 版本

如果你还要支持 `--arch x86_64`，继续准备 `x86_64` 版 `ksuinit`：

```powershell
rustup target add x86_64-unknown-linux-musl
```

```powershell
$clang = Get-ChildItem $env:ANDROID_NDK_HOME -Recurse -Filter x86_64-linux-android26-clang.cmd -ErrorAction SilentlyContinue |
  Select-Object -First 1 -ExpandProperty FullName

$env:CARGO_TARGET_X86_64_UNKNOWN_LINUX_MUSL_LINKER = $clang
$env:RUSTFLAGS = "-C link-arg=-Wno-unused-command-line-argument"

cargo build --target x86_64-unknown-linux-musl --release --manifest-path userspace/ksuinit/Cargo.toml
Copy-Item userspace\ksuinit\target\x86_64-unknown-linux-musl\release\ksuinit userspace\ksud\bin\x86_64\ksuinit -Force
```

## 8. 构建 `ksud.exe`

### 8.1 Windows 本地最直接的构建方式

当前项目在 Windows 本机可以直接使用默认 host target 构建，我已在本仓库实际验证过下面的命令可通过：

```powershell
cargo build --release --manifest-path userspace/ksud/Cargo.toml
```

产物位于：

- [userspace/ksud/target/release/ksud.exe](../userspace/ksud/target/release/ksud.exe)

这条命令通常会生成本机默认的 `x86_64-pc-windows-msvc` 版本。

### 8.2 与仓库 CI 保持一致的构建方式

当前仓库 CI 中，Windows PC 产物使用的是 `x86_64-pc-windows-gnu`，并且是在 Ubuntu 上通过 `cross` 构建，参考：

- [.github/workflows/build-manager.yml](../.github/workflows/build-manager.yml)
- [.github/workflows/ksud.yml](../.github/workflows/ksud.yml)

也就是说：

- 本地 Windows 直接 `cargo build` 可以得到可用的 `msvc` 版
- 如果你想尽量和仓库 CI 产物保持一致，应参考 CI 的 `cross build --target x86_64-pc-windows-gnu`

### 8.3 什么时候需要 `cargo clean`

当你替换了这些内嵌资源后，建议清理再重建：

- `userspace/ksud/bin/aarch64/ksuinit`
- `userspace/ksud/bin/x86_64/ksuinit`
- 任意 `*_kernelsu.ko`

命令：

```powershell
cargo clean --manifest-path userspace/ksud/Cargo.toml
cargo build --release --manifest-path userspace/ksud/Cargo.toml
```

原因是当前项目使用 `rust-embed` 做资源嵌入，替换二进制资源后，完整 clean 一次最稳妥。

## 9. 验证流程

### 9.1 先验证命令入口

```powershell
.\userspace\ksud\target\release\ksud.exe boot-patch --help
```

如果能正常显示帮助，说明 `ksud.exe` 本体已可运行。

### 9.2 再验证内嵌 KMI

```powershell
.\userspace\ksud\target\release\ksud.exe supported-kmis
```

注意：

- 非 Android 目标会递归扫描整个 `bin/`
- 因此如果同时嵌入 `aarch64/` 和 `x86_64/`，输出可能会带上目录前缀，例如 `aarch64/android13-5.10`
- 这是当前实现的自然结果，不代表异常

而真正执行 `boot-patch` 时：

- `--kmi` 仍然填写纯 KMI，例如 `android13-5.10`
- `--arch` 决定去哪个目录拿资源，例如 `aarch64` 或 `x86_64`

### 9.3 实际 patch 示例

默认 `aarch64`：

```powershell
.\userspace\ksud\target\release\ksud.exe boot-patch -b .\boot.img --kmi android13-5.10
```

显式指定输出文件名：

```powershell
.\userspace\ksud\target\release\ksud.exe boot-patch -b .\boot.img --kmi android13-5.10 --out-name patched-boot.img
```

如果要读取 `x86_64` 目录下的内嵌资源：

```powershell
.\userspace\ksud\target\release\ksud.exe boot-patch -b .\boot.img --kmi android13-5.10 --arch x86_64
```

如果不想依赖内嵌资源，也可以手动指定：

```powershell
.\userspace\ksud\target\release\ksud.exe boot-patch `
  -b .\boot.img `
  --kmi android13-5.10 `
  -m .\android13-5.10_kernelsu.ko `
  -i .\ksuinit `
  --out-name patched-boot.img
```

## 10. 最短复现路径

如果你的目标只是最快做出一个“完整可用”的 `ksud.exe`，建议按下面顺序：

1. 切到目标 `tag`
2. 准备该 `tag` 对应的 `aarch64` 版 `ksuinit`
3. 准备该 `tag` 对应的 `aarch64` 版 `*_kernelsu.ko`
4. 放入 `userspace/ksud/bin/aarch64/`
5. 执行：

```powershell
cargo clean --manifest-path userspace/ksud/Cargo.toml
cargo build --release --manifest-path userspace/ksud/Cargo.toml
.\userspace\ksud\target\release\ksud.exe boot-patch --help
.\userspace\ksud\target\release\ksud.exe supported-kmis
```

如果之后还要支持 `x86_64`，再补齐 `userspace/ksud/bin/x86_64/` 并重编即可。

## 11. 常见问题

### 11.1 `supported-kmis` 没有任何输出

优先检查：

1. 资源是否真的放进了 `userspace/ksud/bin/<arch>/`
2. 文件名是否符合 `android13-5.10_kernelsu.ko` 这种格式
3. 资源替换后是否执行了 `cargo clean`
4. 当前运行的 `ksud.exe` 是否确实是最新构建产物

### 11.2 `boot-patch` 报找不到内嵌资源

优先检查：

1. `--kmi` 是否填写了纯 KMI，例如 `android13-5.10`
2. `--arch` 是否和资源目录匹配
3. `ksuinit` 是否位于 `userspace/ksud/bin/<arch>/ksuinit`
4. 对应的 `userspace/ksud/bin/<arch>/<kmi>_kernelsu.ko` 是否存在

### 11.3 资源替换后，`cargo build` 很快结束但结果没变化

最稳妥做法仍然是：

```powershell
cargo clean --manifest-path userspace/ksud/Cargo.toml
cargo build --release --manifest-path userspace/ksud/Cargo.toml
```

### 11.4 `cargo clean` 失败并提示 `os error 5`

这通常是 Windows 文件占用问题。建议：

1. 关闭正在运行的 `ksud.exe`
2. 关闭可能占用 `target` 目录的终端或工具
3. 再重新执行 `cargo clean`

### 11.5 是否必须用 `nightly`

以当前仓库状态来看，不是必须。

当前 [userspace/ksud/src/main.rs](../userspace/ksud/src/main.rs) 已不再依赖文档旧版本中提到的 `#![feature(decl_macro)]`，本仓库在 Windows 上可以直接用 stable Rust 完成 `ksud.exe` 构建。

## 12. 说明范围

本文档描述的是 PC 侧 `ksud.exe` 的资源嵌入与构建流程，不包含：

- Android 设备侧完整运行环境部署
- `manager` APK 打包
- `kernelsu.ko` 的完整 DDK 编译细节展开

如果后续继续补充，建议优先新增这些主题：

- 如何判断设备应使用哪个 KMI
- 如何从 CI artifact 快速回填 `ksuinit` 与 `ko`
- Linux / WSL 下复刻仓库 CI 的 `x86_64-pc-windows-gnu` 构建流程
