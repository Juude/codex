# 跨平台编译与持续集成中的 SSL 依赖架构分析

在 C++ 或 Rust 等系统级编程语言的项目中，处理网络请求时通常需要依赖 SSL/TLS 库（最常见的是 OpenSSL）。在跨平台开发（尤其是 Windows 和 macOS）以及持续集成（CI）环境中，**如何优雅地处理 SSL 依赖**一直是一个痛点。

如果在 CI 机器上没有预装相应的 SSL 库，或者版本不兼容，编译时经常会遇到找不到头文件（`.h`）或链接库（`.lib`/`.so`/`.dylib`）的报错，导致开发者需要编写复杂的脚本去手动下载和编译 OpenSSL。

本项目在架构设计上采用了两种主要策略来彻底解决这个问题：**基于构建系统的源码静态编译**（针对 OpenSSL）和**使用纯语言实现的 TLS 库**（针对 Rustls）。本文档将详细说明这两种方式的区别及其在 CI 中的优势，以供其他 C++ / Rust 项目参考。

---

## 策略一：通过构建系统自动下载并静态编译 OpenSSL (以 Bazel 为例)

在 C++ 或某些强依赖 C 库的 Rust 项目中，有时候不可避免地需要使用 OpenSSL。本项目的解决方案是：**不在操作系统层面寻找 OpenSSL，而是让构建工具自己去下载并静态编译它。**

### 实现方式
在项目的 `MODULE.bazel` 文件中，你可以看到如下配置：

```bzl
# 声明依赖特定版本的 OpenSSL
bazel_dep(name = "openssl", version = "3.5.4.bcr.0")

# 配置 OpenSSL 进行静态编译
crate.annotation(
    build_script_env = {
        "OPENSSL_DIR": "$(execpath @openssl//:gen_dir)",
        "OPENSSL_NO_VENDOR": "1",
        "OPENSSL_STATIC": "1", # 关键：强制静态链接
    },
    crate = "openssl-sys",
    data = ["@openssl//:gen_dir"],
    gen_build_script = "on",
)
```

### 为什么这样做？
1. **统一版本**：无论 CI 运行在 Windows、macOS 还是 Linux Ubuntu 24.04，Bazel 都会去下载版本为 `3.5.4.bcr.0` 的 OpenSSL 源码并从头编译。保证了各平台上版本绝对一致。
2. **无需预装环境**：你的 CI 配置（如 GitHub Actions）不需要写 `apt-get install libssl-dev` 或者使用 `vcpkg` / `brew install openssl`。构建系统屏蔽了底层的 OS 差异。
3. **静态链接 (`OPENSSL_STATIC="1"`)**：编译出的最终可执行文件包含了 OpenSSL 的二进制代码。当产物被分发到用户的机器上时，即使用户机器上没有安装 OpenSSL 也能正常运行（对于 C++ 项目的分发尤为重要）。

> **对 C++ 项目的启发：**
> 如果你的 C++ 项目使用 CMake 或 Bazel，强烈建议使用包管理器（如 vcpkg, Conan）或 FetchContent/bazel_dep 机制，在构建时抓取 OpenSSL 源码并**静态链接**。这样能彻底消除 CI 机器上缺失 SSL 库的痛苦。

---

## 策略二：使用纯语言栈的 TLS 实现 (以 Rustls 为例)

在 Rust 生态中，更现代且被广泛推荐的做法是彻底抛弃 OpenSSL，转而使用纯 Rust 编写的 TLS 库，例如 **`rustls`**。

### 实现方式
查看本项目的 `codex-rs/Cargo.toml`，可以看到依赖项被精心配置为使用 `rustls`，并明确关闭了默认的 `native-tls`（`native-tls` 会试图在 Linux 上寻找 OpenSSL，在 macOS 上寻找 SecureTransport，在 Windows 上寻找 Schannel）。

```toml
# 核心 rustls 依赖
rustls = { version = "0.23", default-features = false, features = ["ring", "std"] }

# 网络请求库 reqwest，关闭默认的 native-tls，使用 rustls
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }

# 数据库驱动 sqlx，明确指定使用 rustls
sqlx = { version = "0.8.6", default-features = false, features = ["runtime-tokio-rustls", "sqlite"] }

# WebSocket 库
tokio-tungstenite = { version = "0.28.0", features = ["proxy", "rustls-tls-native-roots"] }
```

### 为什么这样做？
1. **零 C 语言依赖**：`rustls` 是 100% 用 Rust 编写的。它不需要 C 编译器，不需要 `pkg-config`，也不需要任何系统的 SSL 动态链接库。
2. **极简的持续集成 (CI)**：因为没有任何外部 C 库依赖，你可以直接在任何极其干净的 Docker 镜像或 CI Runner（如干净的 Windows runner）中运行 `cargo build`，一次通过，没有任何前置配置。
3. **跨平台一致性**：`native-tls` 在不同操作系统上调用的底层 API 完全不同，这可能会导致诡异的跨平台 Bug。而 `rustls` 在所有平台上的代码是同一套，行为高度一致。

---

## 两种方式的区别与总结比较

| 特性 | 策略一：静态编译 OpenSSL (Bazel/CMake+源码) | 策略二：纯语言实现 (Rustls / C++的替代品) |
| :--- | :--- | :--- |
| **适用语言** | C++ / C / Rust (需用到 C 库的场景) | 主要是 Rust，Go（自带纯 Go TLS），Java等 |
| **CI 配置难度** | 中（需在构建脚本配置下载和静态编译规则） | 极低（直接 `build` 即可） |
| **编译速度** | 较慢（需要从源码编译庞大的 OpenSSL C 代码） | 快（纯语言代码，可利用语言自身的缓存机制） |
| **可执行文件大小** | 较大（包含了整个静态 OpenSSL） | 较小至中等 |
| **平台依赖性** | 依赖 C 编译器和对应的目标平台构建工具链 | 零系统依赖 |
| **使用建议** | 遗留 C++ 项目，或必须使用特定 OpenSSL 加密算法的场景 | 新的 Rust 项目，强烈推荐优先使用此方案 |

### 给您的 C++ 项目的最终建议

针对您的 C++ 项目在 Windows 和 macOS 上持续集成遇到 SSL 库麻烦的问题，您有两个优化方向：

1. **引入现代 C++ 包管理器（推荐）**：使用 `vcpkg` 或 `Conan`。在 CI 脚本的开始阶段，只需一条命令 `vcpkg install openssl:x64-windows-static`。在 CMake 中配置 `CMAKE_TOOLCHAIN_FILE` 指向 vcpkg。这样每次 CI 跑的时候，会自动获取预编译好（或现场编译）的 OpenSSL 并静态链接，免去了您手动下载和配置环境变量的麻烦。
2. **拥抱 CMake FetchContent / ExternalProject**：在 CMakeLists.txt 中直接定义去 GitHub 下载特定 Tag 的 OpenSSL / BoringSSL（或者更轻量的 mbedTLS）的源码，并随同您的项目一起编译，输出静态链接。这与本项目 Bazel 的处理方式如出一辙，最大程度保证了跨平台的一致性。