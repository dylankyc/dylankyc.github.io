# 不使用 Anchor 开发 solana program

<!-- toc -->

# 初始化工程

## 使用 Cargo 初始化工程

我们可以使用 cargo 来初始化工程。

```bash
cargo init hello_world --lib
```

# 编写代码

## 程序入口 entrypoint

下面利用 `entrypoint` 来编写程序入口。

`entrypoint` macro 需要一个函数参数，作为 solana program 的入口函数。

```rust
pub fn process_instruction() -> ProgramResult {
    msg!("Hello, world!");
    Ok(())
}
```

如果传递给 `entrypoint` macro 的函数签名不符合要求，编译时会报错：

```bash
    Checking hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
error[E0061]: this function takes 0 arguments but 3 arguments were supplied
 --> src/lib.rs:6:1
  |
6 | entrypoint!(process_instruction);
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  | |
  | unexpected argument #1 of type `&Pubkey`
  | unexpected argument #2 of type `&Vec<AccountInfo<'_>>`
  | unexpected argument #3 of type `&[u8]`
  |
note: function defined here
 --> src/lib.rs:8:8
  |
8 | pub fn process_instruction() -> ProgramResult {
  |        ^^^^^^^^^^^^^^^^^^^
  = note: this error originates in the macro `entrypoint` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0061`.
error: could not compile `hello_world` (lib) due to 1 previous error
```

## 修改 process_instruction 函数的签名

给 `process_instruction` 函数添加三个参数：

- `program_id`: `&Pubkey` 类型，表示当前程序的公钥地址
- `accounts`: `&[AccountInfo]` 类型，是一个 AccountInfo 数组的引用，包含了交易涉及的所有账户信息
- `instruction_data`: `&[u8]` 类型，是指令的输入数据，以字节数组的形式传入

这三个参数是 Solana 程序执行时的基本要素：

- `program_id` 用于验证程序身份和权限
- `accounts` 包含了程序需要读取或修改的所有账户数据
- `instruction_data` 携带了调用程序时传入的具体指令数据

```rust
pub fn process_instruction(
    _program_id: &Pubkey,
    _accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello, world!");
    Ok(())
}
```

注意这里参数名前加了下划线前缀（`_`），是因为在这个简单的示例中我们暂时没有使用这些参数，这样可以避免编译器的未使用变量警告。在实际开发中，这些参数都是非常重要的，我们会在后续的示例中详细介绍如何使用它们。

关于函数签名，我们也可以[参考 solana_program_entrypoint 这个 crate 的文档](https://docs.rs/solana-program-entrypoint/latest/solana_program_entrypoint/macro.entrypoint.html):

```rust
/// fn process_instruction(
///     program_id: &Pubkey,      // Public key of the account the program was loaded into
///     accounts: &[AccountInfo], // All accounts required to process the instruction
///     instruction_data: &[u8],  // Serialized instruction-specific data
/// ) -> ProgramResult;
```

# 构建程序

## 使用 cargo build-sbf 构建程序

为了构建 solana program，我们需要使用 `cargo build-sbf` 程序。

```bash
cargo build-sbf
```

构建失败了，以下是报错信息。

```
dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf
error: package `solana-program v2.1.4` cannot be built because it requires rustc 1.79.0 or newer, while the currently active rustc version is 1.75.0-dev
Either upgrade to rustc 1.79.0 or newer, or use
cargo update solana-program@2.1.4 --precise ver
where `ver` is the latest version of `solana-program` supporting rustc 1.75.0-dev
```

我们可以通过 `--version` 参数来查看 `rustc` 的版本信息。

```bash
cargo-build-sbf --version
```

输出：

```
solana-cargo-build-sbf 1.18.25
platform-tools v1.41
rustc 1.75.0
```

关于系统版本的 rust compiler 和 build-sbf 使用的 rust compiler 不对应的问题，可以参考这个 issue。
https://github.com/solana-labs/solana/issues/34987

## 解决 build-sbf 编译失败问题

一种方式是使用旧版本的 `solana-program`，如 `=1.17.0` 版本。

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[dependencies]
solana-program = "=1.17.0"
# solana-program = "=1.18.0"
```

但是运行 `cargo build-sbf` 之后，出现了另外的错误。

```bash
error: failed to parse lock file at: /Users/dylan/Code/solana/projects/hello_world/Cargo.lock

Caused by:
  lock file version 4 requires `-Znext-lockfile-bump`
```

猜测可能是 `build-sbf` 使用的 cargo 版本不支持 version = 4 版本的 `Cargo.lock` 文件，而这个是编辑器（vscode/cursor）打开的状态下，rust-analyser 自动生成的。

安装 `stable` 版本的 solana cli 工具链: `sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"`，发现还是无法编译，报错如下：

```bash
dylan@smalltown ~/Code/solana/projects/hello_world (master)> sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
downloading stable installer
  ✨ stable commit 7104d71 initialized
dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf --version
solana-cargo-build-sbf 2.0.17
platform-tools v1.42

dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf
[2024-12-04T11:14:48.052020000Z ERROR cargo_build_sbf] Failed to install platform-tools: HTTP status client error (404 Not Found) for url (https://github.com/anza-xyz/platform-tools/releases/download/v1.42/platform-tools-osx-x86_64.tar.bz2)
```

在进行 `cargo build-sbf` 编译的时候，需要下载对应版本的 `platform-tools`，因为未发布针对 Mac(Intel) 的 `v1.42` 版本 的 `platform-tools`，所以上述命令运行失败。

```bash
dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf
   Compiling cc v1.2.2
   Compiling serde v1.0.215
   Compiling solana-frozen-abi-macro v1.17.0
   Compiling ahash v0.7.8
   Compiling solana-frozen-abi v1.17.0
   Compiling either v1.13.0
   Compiling bs58 v0.4.0
   Compiling log v0.4.22
   Compiling hashbrown v0.11.2
   Compiling itertools v0.10.5
   Compiling solana-sdk-macro v1.17.0
   Compiling bytemuck v1.20.0
   Compiling borsh v0.9.3
   Compiling num-derive v0.3.3
   Compiling blake3 v1.5.5
   Compiling solana-program v1.17.0
   Compiling bv v0.11.1
   Compiling serde_json v1.0.133
   Compiling serde_bytes v0.11.15
   Compiling bincode v1.3.3
Error: Function _ZN112_$LT$solana_program..instruction..InstructionError$u20$as$u20$solana_frozen_abi..abi_example..AbiEnumVisitor$GT$13visit_for_abi17hc69c00f4c61717f8E Stack offset of 6640 exceeded max offset of 4096 by 2544 bytes, please minimize large stack variables. Estimated function frame size: 6680 bytes. Exceeding the maximum stack offset may cause undefined behavior during execution.

   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished `release` profile [optimized] target(s) in 25.19s
+ ./platform-tools/rust/bin/rustc --version
+ ./platform-tools/rust/bin/rustc --print sysroot
+ set +e
+ rustup toolchain uninstall solana
info: uninstalling toolchain 'solana'
info: toolchain 'solana' uninstalled
+ set -e
+ rustup toolchain link solana platform-tools/rust
+ exit 0
⏎

dylan@smalltown ~/Code/solana/projects/hello_world (master)> ls target/deploy/
hello_world-keypair.json  hello_world.so
dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf --version
solana-cargo-build-sbf 2.1.4
platform-tools v1.43
rustc 1.79.0

dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> sh -c "$(curl -sSfL https://release.anza.xyz/beta/install)"
downloading beta installer
  ✨ beta commit 024d047 initialized
dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf --version
solana-cargo-build-sbf 2.1.4
platform-tools v1.43
rustc 1.79.0
dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf
Error: Function _ZN112_$LT$solana_program..instruction..InstructionError$u20$as$u20$solana_frozen_abi..abi_example..AbiEnumVisitor$GT$13visit_for_abi17hc69c00f4c61717f8E Stack offset of 6640 exceeded max offset of 4096 by 2544 bytes, please minimize large stack variables. Estimated function frame size: 6680 bytes. Exceeding the maximum stack offset may cause undefined behavior during execution.

    Finished `release` profile [optimized] target(s) in 0.23s
```

使用 `beta` 版本的 solana cli tool suites 虽然能够编译，但是遇到了这个错误：

`Exceeding the maximum stack offset may cause undefined behavior during execution.`

```
   Compiling bincode v1.3.3
Error: Function _ZN112_$LT$solana_program..instruction..InstructionError$u20$as$u20$solana_frozen_abi..abi_example..AbiEnumVisitor$GT$13visit_for_abi17hc69c00f4c61717f8E Stack offset of 6640 exceeded max offset of 4096 by 2544 bytes, please minimize large stack variables. Estimated function frame size: 6680 bytes. Exceeding the maximum stack offset may cause undefined behavior during execution.
```

具体原因依旧是老生常谈的版本问题，原因分析可以参考：
https://solana.stackexchange.com/questions/16443/error-function-stack-offset-of-7256-exceeded-max-offset-of-4096-by-3160-bytes

尝试更新 `solana-program` 的版本到 `2.1.4` 之后（运行 `sh -c "$(curl -sSfL https://release.anza.xyz/v2.1.4/install)"`），用以下版本的工具链进行编译：

```bash
> cargo build-sbf --version
solana-cargo-build-sbf 2.1.4
platform-tools v1.43
rustc 1.79.0

# solana-cargo-build-sbf 2.2.0
# platform-tools v1.43
# rustc 1.79.0
```

运行 `cargo build-sbf`:

```bash
> cargo build-sbf
   Compiling serde v1.0.215
   Compiling equivalent v1.0.1
   Compiling hashbrown v0.15.2
   Compiling toml_datetime v0.6.8
   Compiling syn v2.0.90
   Compiling winnow v0.6.20
   Compiling cfg_aliases v0.2.1
   Compiling once_cell v1.20.2
   Compiling borsh v1.5.3
   Compiling solana-define-syscall v2.1.4
   Compiling solana-sanitize v2.1.4
   Compiling solana-atomic-u64 v2.1.4
   Compiling bs58 v0.5.1
   Compiling bytemuck v1.20.0
   Compiling five8_core v0.1.1
   Compiling five8_const v0.1.3
   Compiling solana-decode-error v2.1.4
   Compiling solana-msg v2.1.4
   Compiling cc v1.2.2
   Compiling solana-program-memory v2.1.4
   Compiling log v0.4.22
   Compiling solana-native-token v2.1.4
   Compiling solana-program-option v2.1.4
   Compiling indexmap v2.7.0
   Compiling blake3 v1.5.5
   Compiling toml_edit v0.22.22
   Compiling serde_derive v1.0.215
   Compiling bytemuck_derive v1.8.0
   Compiling solana-sdk-macro v2.1.4
   Compiling thiserror-impl v1.0.69
   Compiling num-derive v0.4.2
   Compiling proc-macro-crate v3.2.0
   Compiling borsh-derive v1.5.3
   Compiling thiserror v1.0.69
   Compiling solana-secp256k1-recover v2.1.4
   Compiling solana-borsh v2.1.4
   Compiling solana-hash v2.1.4
   Compiling bincode v1.3.3
   Compiling bv v0.11.1
   Compiling solana-serde-varint v2.1.4
   Compiling serde_bytes v0.11.15
   Compiling solana-fee-calculator v2.1.4
   Compiling solana-short-vec v2.1.4
   Compiling solana-sha256-hasher v2.1.4
   Compiling solana-pubkey v2.1.4
   Compiling solana-instruction v2.1.4
   Compiling solana-sysvar-id v2.1.4
   Compiling solana-slot-hashes v2.1.4
   Compiling solana-clock v2.1.4
   Compiling solana-epoch-schedule v2.1.4
   Compiling solana-last-restart-slot v2.1.4
   Compiling solana-rent v2.1.4
   Compiling solana-program-error v2.1.4
   Compiling solana-stable-layout v2.1.4
   Compiling solana-serialize-utils v2.1.4
   Compiling solana-account-info v2.1.4
   Compiling solana-program-pack v2.1.4
   Compiling solana-bincode v2.1.4
   Compiling solana-slot-history v2.1.4
   Compiling solana-program-entrypoint v2.1.4
   Compiling solana-cpi v2.1.4
   Compiling solana-program v2.1.4
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished `release` profile [optimized] target(s) in 50.87s
```

总算编译成功了，开瓶香槟庆祝一下吧！

这里是 Cargo.toml 文件：

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[dependencies]
solana-program = "2.1.4"
# solana-program = "=1.17.0"
```

## 构建产物

`cargo build-sbf` 是 Solana 提供的一个特殊的构建命令，用于将 Rust 程序编译成可以在 Solana 运行时环境中执行的 BPF (Berkeley Packet Filter) 字节码。这个命令做了以下几件事：

1. 使用特定的 Rust 工具链编译代码

   - 使用针对 Solana 优化的 Rust 编译器
   - 使用 `bpfel-unknown-unknown` 目标平台
   - 启用发布模式优化

2. 生成必要的部署文件

   - 编译出 `.so` 文件（共享对象文件）
   - 生成程序密钥对（如果不存在）
   - 优化和压缩最终的二进制文件

3. 验证编译结果
   - 检查程序大小是否在限制范围内
   - 验证程序格式是否正确

命令执行流程：

1. 首先检查并下载必要的工具链
2. 使用 cargo 编译项目
3. 对编译产物进行后处理（如剥离调试信息）
4. 将最终文件放置在 `target/deploy` 目录

这个命令替代了早期的 `cargo build-bpf`，提供了更好的构建体验和更现代的工具链支持。

我们来看看具体生成了哪些文件，运行 `cargo build-sbf` 这个命令之后会在 `target/deploy` 目录下生成两个重要文件：

- `hello_world.so`：编译后的程序文件，这是一个 BPF (Berkeley Packet Filter) 格式的可执行文件
- `hello_world-keypair.json`：程序的密钥对文件，用于程序的部署和升级

如果你看到类似下面的输出，说明构建成功：

```bash
BPF SDK: /Users/username/.local/share/solana/install/releases/1.14.x/solana-release/bin/sdk/bpf
cargo-build-sbf child: rustup toolchain list -v
cargo-build-sbf child: cargo +bpf build --target bpfel-unknown-unknown --release
    Finished release [optimized] target(s) in 0.20s
cargo-build-sbf child: /Users/username/.local/share/solana/install/releases/1.14.x/solana-release/bin/sdk/bpf/scripts/strip.sh /Users/username/projects/hello_world/target/bpfel-unknown-unknown/release/hello_world.so /Users/username/projects/hello_world/target/deploy/hello_world.so
```

# 部署

现在我们可以将编译好的程序部署到 Solana 网络上。在开发阶段，我们通常使用本地测试网（localhost）或开发网（devnet）进行测试。

首先确保你的 Solana CLI 配置指向了正确的集群：

```bash
# 切换到开发网
solana config set --url devnet
# 切换到本地测试网
solana config set --url localnet

# 查看当前配置
solana config get
```

然后使用以下命令部署程序：

```bash
solana program deploy target/deploy/hello_world.so
```

部署成功后，你会看到程序的 ID（公钥地址）。请保存这个地址，因为在后续与程序交互时会需要它。

但是，当我们通过运行 `solana program deploy` 命令来部署程序的时候，部署失败了。

```bash
dylan@smalltown ~/Code/solana/projects/helloworld (master)> solana program deploy ./target/deploy/helloworld.so
⠁   0.0% | Sending 1/173 transactions               [block height 2957; re-sign in 150 blocks]
    thread 'main' panicked at quic-client/src/nonblocking/quic_client.rs:142:14:
QuicLazyInitializedEndpoint::create_endpoint bind_in_range: Os { code: 55, kind: Uncategorized, message: "No buffer space available" }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

那么这个 `No buffer space available` 是什么意思呢？

排查了很久终于无果，凭借多年的经验，大概率应该是 **版本** 的问题，因为通过 `Anchor` 创建的工程是能够正常部署的。

这里记录一下 `solana` 命令的版本信息:

```bash
> solana --version
solana-cli 2.2.0 (src:67704836; feat:1081947060, client:Agave)
```

## 回到 Anchor 工程验证部署失败源自版本的问题

我们可以通过 `anchor init helloworld` 新建工程，并通过 `anchor build` 和 `anchor deploy` 来部署程序。

```bash
anchor init helloworld
cd helloworld
anchor build
anchor deploy
```

从出错信息了解到，全新生成的 anchor 工程部署的时候会发生同样的错误：`No buffer space available`

```bash
dylan@smalltown ~/tmp/helloworld (main)> anchor deploy
Deploying cluster: https://api.devnet.solana.com
Upgrade authority: /Users/dylan/.config/solana/id.json
Deploying program "helloworld"...
Program path: /Users/dylan/tmp/helloworld/target/deploy/helloworld.so...
⠁   0.0% | Sending 1/180 transactions               [block height 332937196; re-sign in 150 blocks]                                                       thread 'main' panicked at quic-client/src/nonblocking/quic_client.rs:142:14:
QuicLazyInitializedEndpoint::create_endpoint bind_in_range: Os { code: 55, kind: Uncategorized, message: "No buffer space available" }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
There was a problem deploying: Output { status: ExitStatus(unix_wait_status(25856)), stdout: "", stderr: "" }.
```

检查下 anchor 的版本：

```bash
dylan@smalltown ~/tmp/helloworld (main)> anchor deploy --help
Deploys each program in the workspace

Usage: anchor-0.30.1 deploy [OPTIONS] [-- <SOLANA_ARGS>...]

Arguments:
  [SOLANA_ARGS]...  Arguments to pass to the underlying `solana program deploy` command

Options:
  -p, --program-name <PROGRAM_NAME>        Only deploy this program
      --provider.cluster <CLUSTER>         Cluster override
      --program-keypair <PROGRAM_KEYPAIR>  Keypair of the program (filepath) (requires program-name)
      --provider.wallet <WALLET>           Wallet override
  -v, --verifiable                         If true, deploy from path target/verifiable
  -h, --help                               Print help
```

检查下 solana 的版本:

```bash
> solana --version
solana-cli 2.2.0 (src:67704836; feat:1081947060, client:Agave)
```

这个 `2.2.0` 的版本看着有些奇怪，忽然想到为了编译 solana 程序，我安装了 edge 版本的 solana cli，其携带的 solana cli 的版本是 `2.2.0`:

```bash
sh -c "$(curl -sSfL https://release.anza.xyz/edge/install)"
```

于是换回了 `stable` 版本：

```bash
> sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
downloading stable installer
  ✨ stable commit fbead11 initialized
```

而 stable 版本的 solana 是 `2.0.19`。

```bash
> solana --version
solana-cli 2.0.19 (src:fbead118; feat:607245837, client:Agave)
```

重新部署程序之前，我们先来清理下之前部署失败的程序的 `buffers`，也就是 buffer accounts。关于什么是 buffer accounts，请参考 Tips 3。

- 查看所有的 buffer accounts: `solana program show --buffers`
- 关闭所有的 buffer accounts: `solana program close --buffers`
  - 关闭 buffer accounts 可以回收存储在 buffer accounts 里的 SOL

```bash
Error: error sending request for url (https://api.devnet.solana.com/): operation timed out
dylan@smalltown ~/tmp/helloworld (main)> solana program show --buffers

Buffer Address                               | Authority                                    | Balance
CcKFVBzcsrcReZHBLnwzkQbNGXoK4hUee7hkgtbHCKtL | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 0.12492504 SOL
62wFzMYBhxWg4ntEJmFZcQ3P3Qtm9SbaBcbTmV8o8yPk | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 0.12492504 SOL
9q88jzvR5AdPdNTihxWroxRL7cBWQ5xXepNfDdaqmMTv | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 1.26224472 SOL
3nqzHv9vUphsmAjoR1C5ShgZ54muTzkZZ6Z4NKfqrKqt | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 1.26224472 SOL
8tZ8YYA1WS6WFVyEbJAdgnszXYZwwq7b9RLdoiry2Fb1 | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 0.12492504 SOL

dylan@smalltown ~/tmp/helloworld (main)> solana program close --buffers

Buffer Address                               | Authority                                    | Balance
CcKFVBzcsrcReZHBLnwzkQbNGXoK4hUee7hkgtbHCKtL | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 0.12492504 SOL
62wFzMYBhxWg4ntEJmFZcQ3P3Qtm9SbaBcbTmV8o8yPk | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 0.12492504 SOL
9q88jzvR5AdPdNTihxWroxRL7cBWQ5xXepNfDdaqmMTv | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 1.26224472 SOL
3nqzHv9vUphsmAjoR1C5ShgZ54muTzkZZ6Z4NKfqrKqt | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 1.26224472 SOL
8tZ8YYA1WS6WFVyEbJAdgnszXYZwwq7b9RLdoiry2Fb1 | FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH | 0.12492504 SOL
```

好了 buffer accounts 清理完毕，此时我们也换回了 `stable` 版本的 solana cli，我们再尝试部署程序:

```bash
> anchor deploy
Deploying cluster: https://api.devnet.solana.com
Upgrade authority: /Users/dylan/.config/solana/id.json
Deploying program "helloworld"...
Program path: /Users/dylan/tmp/helloworld/target/deploy/helloworld.so...
Program Id: DiSGTiXGq4HXCxq1pAibuGZjSpKT4Av8WShvuuYhTks9

Signature: 2EXHmU68k9SmJ5mXuM61pFDnUgozbJZ5ihHChPqFMVgjRJy4zCqnq6NAbvDkfiHd29xsmW4Vr3Kk6wHFbLEdCEZb

Deploy success
```

成功了 🎉，再开一瓶香槟庆祝下吧！

这更加深了我们的猜测：版本问题导致程序无法部署。

## 再回来部署我们的 hello_world 工程

好了，验证了部署失败不是工程类型（anchor project or cargo projct）导致的原因之后，我们再回到 `cargo init` 创建的工程：`hello_world`.

我们可以通过 `solana` 的子命令来部署程序: 运行 `solana program deploy ./target/deploy/helloworld.so` 部署程序。

我们会分别在 `localnet` 和 `devnet` 部署。

### localnet 部署

首先是 `localnet` 部署。

切换环境到 localnet：

```bash
dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana_local
Config File: /Users/dylan/.config/solana/cli/config.yml
RPC URL: http://localhost:8899
WebSocket URL: ws://localhost:8900/ (computed)
Keypair Path: /Users/dylan/.config/solana/id.json
Commitment: confirmed
dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana config get
Config File: /Users/dylan/.config/solana/cli/config.yml
RPC URL: http://localhost:8899
WebSocket URL: ws://localhost:8900/ (computed)
Keypair Path: /Users/dylan/.config/solana/id.json
Commitment: confirmed
```

部署程序：

```bash
dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana program deploy ./target/deploy/hello_world.so
Program Id: DhQr1KGGQcf8BeU5uQvR35p2kgKqEinD45PRTDDRqx7z

Signature: 3WVEWN4NUodsb8ZDjbjrTWXLikZ7wbWCuzuRZtSBmyKL4kVvESSeLwKZ3cJo1At4vDcaBs5iEcHhdteyXCwqwmDw
```

### devnet 部署

下面是 `devnet` 部署。

切换环境到 localnet：

```bash
dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana_devnet
Config File: /Users/dylan/.config/solana/cli/config.yml
RPC URL: https://api.devnet.solana.com
WebSocket URL: wss://api.devnet.solana.com/ (computed)
Keypair Path: /Users/dylan/.config/solana/id.json
Commitment: confirmed

dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana config get
Config File: /Users/dylan/.config/solana/cli/config.yml
RPC URL: https://api.devnet.solana.com
WebSocket URL: wss://api.devnet.solana.com/ (computed)
Keypair Path: /Users/dylan/.config/solana/id.json
Commitment: confirmed

dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana program deploy ./target/deploy/hello_world.so
Program Id: DhQr1KGGQcf8BeU5uQvR35p2kgKqEinD45PRTDDRqx7z

Signature: 4P89gHNUNccQKJAsE3aXJVpFrWeqLxcmk9SYHbQCX7T1sEvyPrxcbrAeJbk8F8YKwWT79nTswSZkz7mtSb55nboF
```

我们可以通过 solana balance 来查询下部署前后的余额

```bash
# 部署之前余额
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana balance
75.153619879 SOL

# 部署之后余额
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana balance
75.152378439 SOL
```

而此时的版本：

```bash
dylan@smalltown ~/Code/solana/projects/helloworld (master)> solana --version
solana-cli 2.0.19 (src:fbead118; feat:607245837, client:Agave)
```

由此可见，不要尝鲜用最新的版本（solana-cli `2.2.0`），否则会弄巧成拙。

# Tips

## Tip 1: solana cli 的版本和 Cargo.toml 里的版本保持一致

在 [solana 的官方教程](https://solana.com/developers/guides/getstarted/local-rust-hello-world#create-a-new-rust-library-with-cargo)里提到这个 Tip:

> It is highly recommended to keep your solana-program and other Solana Rust dependencies in-line with your installed version of the Solana CLI. For example, if you are running Solana CLI 2.0.3, you can instead run:

```bash
cargo add solana-program@"=2.0.3"
```

> This will ensure your crate uses only 2.0.3 and nothing else. If you experience compatibility issues with Solana dependencies, check out the

## Tip 2: 不要在 dependencies 里添加 solana-sdk，因为这是 offchain 使用的

参考这里的说明：
https://solana.stackexchange.com/questions/9109/cargo-build-bpf-failed

> I have identified the issue. The solana-sdk is designed for off-chain use only, so it should be removed from the dependencies.

错误将 `solana-sdk` 添加到 dependencies 报错：

```bash
   Compiling autocfg v1.4.0
   Compiling jobserver v0.1.32
error: target is not supported, for more information see: https://docs.rs/getrandom/#unsupported-targets
   --> src/lib.rs:267:9
    |
267 | /         compile_error!("\
268 | |             target is not supported, for more information see: \
269 | |             https://docs.rs/getrandom/#unsupported-targets\
270 | |         ");
    | |__________^

error[E0433]: failed to resolve: use of undeclared crate or module `imp`
   --> src/lib.rs:291:5
    |
291 |     imp::getrandom_inner(dest)
    |     ^^^ use of undeclared crate or module `imp`

For more information about this error, try `rustc --explain E0433`.
error: could not compile `getrandom` (lib) due to 2 previous errors
warning: build failed, waiting for other jobs to finish...
```

## Tip 3: 关于 buffer accounts

在 Solana 中，buffer accounts 是用于程序部署过程中的一种临时账户，它是 Solana 部署程序时的一个重要机制。由于 Solana 的交易大小限制为 `1232` 字节，部署程序时通常需要多个交易步骤。在这个过程中，buffer account 的作用是存储程序的字节码，直到部署完成。

buffer account 的关键点：

- 临时存储：buffer account 用于存放程序的字节码，确保在部署过程中能够处理较大的程序。
- 自动关闭：一旦程序成功部署，相关的 buffer account 会自动关闭，从而释放占用的资源。
- 失败处理：如果部署失败，buffer account 不会自动删除，用户可以选择：
  - 继续使用现有的 buffer account 来完成部署。
  - 关闭 buffer account，以便回收已分配的 SOL（租金）。
- 检查 buffer accounts：可以通过命令 `solana program show --buffers` 来检查当前是否存在未关闭的 buffer accounts。
- 关闭 buffer accounts：可以通过命令 `solana program close --buffers` 来关闭 buffer accounts。

关于 solana 程序部署的过程的解释，可以查考官方文档： https://solana.com/docs/programs/deploying#program-deployment-process

# 重新部署

重新部署只需要编辑代码之后运行 `cargo build-sbf` 编译代码，再通过 `solana program deply ./target/deploy/hello_world.so` 部署即可。

```bash
cargo build-sbf
solana program deploy ./target/deploy/hello_world.so
```

可以通过运行测试和 client 脚本来验证运行的是新版本的 program。

```bash
# 运行测试
cargo test-sbf
# 运行 client 脚本
cargo run --example client
```

比如，我修改 `msg!` 输入内容为 `Hello, world! GM!GN!`，运行测试和 client 脚本能够看到 log 里有这个输出。

```rust
pub fn process_instruction(
    _program_id: &Pubkey,
    _accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello, world! GM!GN!");
    Ok(())
}
```

运行测试:

```
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo test-sbf
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished release [optimized] target(s) in 1.76s
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 13.92s
     Running unittests src/lib.rs (target/debug/deps/hello_world-ee1a919556768e26)

running 1 test
[2024-12-06T08:06:57.714248000Z INFO  solana_program_test] "hello_world" SBF program from /Users/dylan/Code/solana/projects/hello_world/target/deploy/hello_world.so, modified 19 seconds, 228 ms, 255 µs and 392 ns ago
[2024-12-06T08:06:57.947344000Z DEBUG solana_runtime::message_processor::stable_log] Program 1111111QLbz7JHiBTspS962RLKV8GndWFwiEaqKM invoke [1]
[2024-12-06T08:06:57.947695000Z DEBUG solana_runtime::message_processor::stable_log] Program log: Hello, world! GM!GN!
[2024-12-06T08:06:57.947738000Z DEBUG solana_runtime::message_processor::stable_log] Program 1111111QLbz7JHiBTspS962RLKV8GndWFwiEaqKM consumed 140 of 200000 compute units
[2024-12-06T08:06:57.947897000Z DEBUG solana_runtime::message_processor::stable_log] Program 1111111QLbz7JHiBTspS962RLKV8GndWFwiEaqKM success
test test::test_hello_world ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.24s

   Doc-tests hello_world

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

TODO: image

# 最佳实践

## 安装 solana-cli 的最佳实践

最好的方式是安装指定版本的 solana cli，如可以用以下方式安装 `2.0.3` 的版本：

```bash
# 安装 stable 和 beta 都不推荐
# sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
# sh -c "$(curl -sSfL https://release.anza.xyz/beta/install)"
# 推荐安装指定版本
sh -c "$(curl -sSfL https://release.anza.xyz/v2.0.3/install)"
```

输出：

```
downloading v2.0.3 installer
  ✨ 2.0.3 initialized
```

运行 `cargo build-sbf --version` 查看下 `cargo build-sbf` 的版本：

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> cargo build-sbf --version
solana-cargo-build-sbf 2.0.3
platform-tools v1.41
rustc 1.75.0
```

可以看到，这里的 rustc 版本是 `1.75.0`，比较老旧，编译的时候必须带上 `-Znext-lockfile-bump` 参数，否则编译出错：

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf
info: uninstalling toolchain 'solana'
info: toolchain 'solana' uninstalled
error: failed to parse lock file at: /Users/dylan/Code/solana/projects/hello_world/Cargo.lock

Caused by:
  lock file version 4 requires `-Znext-lockfile-bump`
```

以下是传递 `-Znext-lockfile-bump` 参数之后，完整的编译过程：

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf -- -Znext-lockfile-bump
   Compiling proc-macro2 v1.0.92
   Compiling unicode-ident v1.0.14
   Compiling version_check v0.9.5
   Compiling typenum v1.17.0
   Compiling autocfg v1.4.0
   Compiling serde v1.0.215
   Compiling syn v1.0.109
   Compiling cfg-if v1.0.0
   Compiling equivalent v1.0.1
   Compiling hashbrown v0.15.2
   Compiling semver v1.0.23
   Compiling generic-array v0.14.7
   Compiling ahash v0.8.11
   Compiling winnow v0.6.20
   Compiling indexmap v2.7.0
   Compiling toml_datetime v0.6.8
   Compiling shlex v1.3.0
   Compiling quote v1.0.37
   Compiling subtle v2.6.1
   Compiling cc v1.2.2
   Compiling syn v2.0.90
   Compiling once_cell v1.20.2
   Compiling rustversion v1.0.18
   Compiling feature-probe v0.1.1
   Compiling zerocopy v0.7.35
   Compiling cfg_aliases v0.2.1
   Compiling borsh v1.5.3
   Compiling bv v0.11.1
   Compiling rustc_version v0.4.1
   Compiling num-traits v0.2.19
   Compiling memoffset v0.9.1
   Compiling thiserror v1.0.69
   Compiling toml_edit v0.22.22
   Compiling blake3 v1.5.5
   Compiling block-buffer v0.10.4
   Compiling crypto-common v0.1.6
   Compiling solana-program v2.0.3
   Compiling digest v0.10.7
   Compiling hashbrown v0.13.2
   Compiling constant_time_eq v0.3.1
   Compiling bs58 v0.5.1
   Compiling arrayvec v0.7.6
   Compiling arrayref v0.3.9
   Compiling keccak v0.1.5
   Compiling sha2 v0.10.8
   Compiling toml v0.5.11
   Compiling sha3 v0.10.8
   Compiling proc-macro-crate v3.2.0
   Compiling borsh-derive-internal v0.10.4
   Compiling borsh-schema-derive-internal v0.10.4
   Compiling getrandom v0.2.15
   Compiling lazy_static v1.5.0
   Compiling bytemuck v1.20.0
   Compiling log v0.4.22
   Compiling proc-macro-crate v0.1.5
   Compiling serde_derive v1.0.215
   Compiling thiserror-impl v1.0.69
   Compiling num-derive v0.4.2
   Compiling solana-sdk-macro v2.0.3
   Compiling bytemuck_derive v1.8.0
   Compiling borsh-derive v1.5.3
   Compiling borsh-derive v0.10.4
   Compiling borsh v0.10.4
   Compiling serde_bytes v0.11.15
   Compiling bincode v1.3.3
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished release [optimized] target(s) in 2m 28s
+ ./platform-tools/rust/bin/rustc --version
+ ./platform-tools/rust/bin/rustc --print sysroot
+ set +e
+ rustup toolchain uninstall solana
info: uninstalling toolchain 'solana'
info: toolchain 'solana' uninstalled
+ set -e
+ rustup toolchain link solana platform-tools/rust
+ exit 0
```

值得注意的是，无论是安装 stable 版本还是 beta 版本都会导致编译失败，stable 版本运行 `cargo build-sbf` 会去 github release 页面下载针对 `x86_64` 架构的 platform-tools，但是官方没有发布提供针对这个版本的 platform-tools。以下是出错信息：

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> cargo build-sbf --version
solana-cargo-build-sbf 2.0.19
platform-tools v1.42

(base) dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> cargo build-sbf
[2024-12-05T06:17:30.547088000Z ERROR cargo_build_sbf] Failed to install platform-tools: HTTP status client error (404 Not Found) for url (https://github.com/anza-xyz/platform-tools/releases/download/v1.42/platform-tools-osx-x86_64.tar.bz2)
```

发现如果指定 `--tools-version` 为 `v1.43` 也不能成功编译。

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> cargo build-sbf --tools-version v1.43
    Blocking waiting for file lock on package cache
    Blocking waiting for file lock on package cache
   Compiling blake3 v1.5.5
   Compiling solana-program v2.0.3
   Compiling bs58 v0.5.1
   Compiling solana-sdk-macro v2.0.3
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished `release` profile [optimized] target(s) in 1m 16s
+ curl -L https://github.com/anza-xyz/platform-tools/releases/download/v1.42/platform-tools-osx-x86_64.tar.bz2 -o platform-tools-osx-x86_64.tar.bz2
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100     9  100     9    0     0     16      0 --:--:-- --:--:-- --:--:--    16
+ tar --strip-components 1 -jxf platform-tools-osx-x86_64.tar.bz2
tar: Error opening archive: Unrecognized archive format
+ return 1
+ popd
+ return 1
/Users/dylan/.local/share/solana/install/releases/stable-fbead118867c08e6c3baaf8d196897c2536f067a/solana-release/bin/sdk/sbf/scripts/strip.sh: line 23: /Users/dylan/.local/share/solana/install/releases/stable-fbead118867c08e6c3baaf8d196897c2536f067a/solana-release/bin/sdk/sbf/dependencies/platform-tools/llvm/bin/llvm-objcopy: No such file or directory
```

所以还是老老实实安装指定版本的 solana cli 吧。

# 如何查看部署的 program

我们可以通过访问以下地址来查看部署的 program。

https://explorer.solana.com/?cluster=custom

它会自动用本地的 localhost:8899 作为 rpc endpoint，在搜索栏搜索 program id，即可看到 transaction 详情。

# 客户端调用

## 客户调用程序 (Rust) (invoke solana program)

首先创建 `examples` 目录，并在 `examples` 目录下创建 `client.rs` 文件。

```bash
mkdir -p examples
touch examples/client.rs
```

在 `Cargo.toml` 增加以下内容：

```toml
[[example]]
name = "client"
path = "examples/client.rs"
```

添加 `solana-client` 依赖:

```bash
cargo add solana-client@1.18.26 --dev
```

添加以下代码到 `examples/client.rs`，注意替换你自己部署的 program ID:

```rust
use solana_client::rpc_client::RpcClient;
use solana_sdk::{
    commitment_config::CommitmentConfig,
    instruction::Instruction,
    pubkey::Pubkey,
    signature::{Keypair, Signer},
    transaction::Transaction,
};
use std::str::FromStr;

#[tokio::main]
async fn main() {
    // Program ID (replace with your actual program ID)
    let program_id = Pubkey::from_str("85K3baeo8tvZBmuty2UP8mMVd1vZtxLkmeUkj1s6tnT6").unwrap();

    // Connect to the Solana devnet
    let rpc_url = String::from("http://127.0.0.1:8899");
    let client = RpcClient::new_with_commitment(rpc_url, CommitmentConfig::confirmed());

    // Generate a new keypair for the payer
    let payer = Keypair::new();

    // Request airdrop
    let airdrop_amount = 1_000_000_000; // 1 SOL
    let signature = client
        .request_airdrop(&payer.pubkey(), airdrop_amount)
        .expect("Failed to request airdrop");

    // Wait for airdrop confirmation
    loop {
        let confirmed = client.confirm_transaction(&signature).unwrap();
        if confirmed {
            break;
        }
    }

    // Create the instruction
    let instruction = Instruction::new_with_borsh(
        program_id,
        &(),    // Empty instruction data
        vec![], // No accounts needed
    );

    // Add the instruction to new transaction
    let mut transaction = Transaction::new_with_payer(&[instruction], Some(&payer.pubkey()));
    transaction.sign(&[&payer], client.get_latest_blockhash().unwrap());

    // Send and confirm the transaction
    match client.send_and_confirm_transaction(&transaction) {
        Ok(signature) => println!("Transaction Signature: {}", signature),
        Err(err) => eprintln!("Error sending transaction: {}", err),
    }
}
```

这个简单的脚本能够调用已部署的 solana program，它主要做了以下几件事：

- 连接本地 RPC
- 创建新账户
- 空投 1 SOL 给新开的账户
- 创建 hello_world program 所需的指令（Instruction）
- 发送交易 （通过 `send_and_confirm_transaction`）

关于 program ID，我们可以通过 `solana address -k <program keypair>.json` 命令来获取 program ID:

```bash
solana address -k ./target/deploy/hello_world-keypair.json
```

`-k` 参数接收 keypair 的文件，可以获得 PublicKey。

运行 client:

```bash
cargo run --example client
```

运行 client 代码的输出：

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo run --example client
    Blocking waiting for file lock on package cache
    Blocking waiting for file lock on package cache
    Blocking waiting for file lock on package cache
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 5.13s
     Running `target/debug/examples/client`
Transaction Signature: iPcYzbBCM6kkXvdx5GQLS9WYunT6yWFAp8NeRyNH5ZHbjXNpGuT1pqLAmQZSa2g7mubuFmaCTxqPVS54J4Zz22h
```

## 客户端调用(TypeScript)

我们可以通过建立 nodejs 工程来发送交易：

```bash
mkdir -p helloworld
npm init -y
npm install --save-dev typescript
npm install @solana/web3.js@1 @solana-developers/helpers@2
```

建立 `tsconfig.json` 配置文件：

```json
{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "types": ["node"],
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

创建 `hello-world-client.ts` 文件，注意修改 `PublicKey` 的参数为你部署时生成的 programID:

```typescript
import {
  Connection,
  PublicKey,
  Transaction,
  TransactionInstruction,
} from "@solana/web3.js";
import { getKeypairFromFile } from "@solana-developers/helpers";

async function main() {
  const programId = new PublicKey(
    "DhQr1KGGQcf8BeU5uQvR35p2kgKqEinD45PRTDDRqx7z"
  );

  // Connect to a solana cluster. Either to your local test validator or to devnet
  const connection = new Connection("http://localhost:8899", "confirmed");
  //const connection = new Connection("https://api.devnet.solana.com", "confirmed");

  // We load the keypair that we created in a previous step
  const keyPair = await getKeypairFromFile("~/.config/solana/id.json");

  // Every transaction requires a blockhash
  const blockhashInfo = await connection.getLatestBlockhash();

  // Create a new transaction
  const tx = new Transaction({
    ...blockhashInfo,
  });

  // Add our Hello World instruction
  tx.add(
    new TransactionInstruction({
      programId: programId,
      keys: [],
      data: Buffer.from([]),
    })
  );

  // Sign the transaction with your previously created keypair
  tx.sign(keyPair);

  // Send the transaction to the Solana network
  const txHash = await connection.sendRawTransaction(tx.serialize());

  console.log("Transaction sent with hash:", txHash);

  await connection.confirmTransaction({
    blockhash: blockhashInfo.blockhash,
    lastValidBlockHeight: blockhashInfo.lastValidBlockHeight,
    signature: txHash,
  });

  console.log(
    `Congratulations! Look at your ‘Hello World' transaction in the Solana Explorer:
  https://explorer.solana.com/tx/${txHash}?cluster=custom`
  );
}

main();
```

运行:

```bash
npx ts-node hello-world-client.ts
```

输出：

```bash
(base) dylan@smalltown ~/Code/solana/projects/solana-web3-example (master)> npx ts-node hello-world-client.ts
(node:4408) ExperimentalWarning: CommonJS module /usr/local/lib/node_modules/npm/node_modules/debug/src/node.js is loading ES Module /usr/local/lib/node_modules/npm/node_modules/supports-color/index.js using require().
Support for loading ES Module in require() is an experimental feature and might change at any time
(Use `node --trace-warnings ...` to show where the warning was created)
(node:4467) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
Transaction sent with hash: 29aFYDNv1cyrByA8FTBxrhohJx3H1FVLSUordaA1RVcXSNSy7zN5mGW5rwj6pDuopMvvoBaKNHeKmQ8c17uVnqoN
Congratulations! Look at your ‘Hello World' transaction in the Solana Explorer:
  https://explorer.solana.com/tx/29aFYDNv1cyrByA8FTBxrhohJx3H1FVLSUordaA1RVcXSNSy7zN5mGW5rwj6pDuopMvvoBaKNHeKmQ8c17uVnqoN?cluster=custom
```

# 一些实验

## 哪些版本能成功编译和测试

首先看一下我们安装的 `build-sbf` 和 `test-sbf` 的版本:

```bash
# build-sbf 版本
> cargo build-sbf --version
solana-cargo-build-sbf 2.1.4
platform-tools v1.43
rustc 1.79.0

# test-sbf 版本
> cargo test-sbf --version
solana-cargo-test-sbf 2.1.4
```

我们通过这个命令来测试哪些版本能够正确编译和测试: `rm -rf target Cargo.lock && cargo build-sbf && cargo test-sbf`

| version   | DevDependencies & Dependencies                                                                             | NOTE           |
| --------- | ---------------------------------------------------------------------------------------------------------- | -------------- |
| ✅2.1.4   | `cargo add solana-sdk@2.1.4 solana-program-test@2.1.4 tokio --dev && cargo add solana-program@2.1.4`       | latest version |
| ✅2.0.18  | `cargo add solana-sdk@2.0.18 solana-program-test@2.0.18 tokio --dev && cargo add solana-program@2.0.18`    | latest version |
| ✅2.0.3   | `cargo add solana-sdk@2.0.3 solana-program-test@2.0.3 tokio --dev && cargo add solana-program@2.0.3`       |                |
| ✅1.18.26 | `cargo add solana-sdk@1.18.26 solana-program-test@1.18.26 tokio --dev && cargo add solana-program@1.18.26` |                |

这里是 `Cargo.toml` 的例子（对应版本是 `2.0.3`）：

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[dependencies]
solana-program = "2.0.3"

[dev-dependencies]
solana-program-test = "2.0.3"
solana-sdk = "2.0.3"
tokio = "1.42.0"
```

# 测试

关于 solana 程序的测试，我们一般采用

bankrun 是一个用于在 Node.js 中测试 Solana 程序的轻量级框架。与传统的 solana-test-validator 相比，bankrun 提供了更高的速度和便利性。它能够实现一些 solana-test-validator 无法做到的功能，例如时间回溯和动态设置账户数据。

它会启动一个轻量级的 BanksServer，这个服务类似于一个 RPC 节点，但速度更快，并且创建一个 BanksClient 来与服务器进行通信

主要特点：

- 高效性：比 solana-test-validator 快得多。
- 灵活性：支持时间回溯和动态账户数据设置。
- solana-bankrun 底层基于 solana-program-test，使用轻量级的 BanksServer 和 BanksClient。

接下来，我们来看看如何用 Rust(`solana-program-test`) 和 NodeJS(`solana-bankrun`) 编写测试用例。

## 测试(Rust)

首先，我们来用 Rust 代码进行测试。

首先安装测试所需要的依赖：

```bash
cargo add solana-sdk@1.18.26 solana-program-test@1.18.26 tokio --dev
# NOTE: There's no error like `Exceeding maximum ...` when building with solana-program = 2.1.4
# We use solana cli with version `2.1.4`
# To install solana-cli with version 2.1.4, run this command:
#
# sh -c "$(curl -sSfL https://release.anza.xyz/v2.1.4/install)"
#
# cargo add solana-sdk@=2.1.4 solana-program-test@=2.1.4 tokio --dev
# cargo add solana-program@=2.1.4
```

因为我们已经测试过，对于版本 `2.1.4`, `2.0.18`, `2.0.3`, `1.18.26` 都能成功编译和测试，所以我们只选择了其中一个版本 `1.18.26` 来做演示。

测试结果输出：

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo test-sbf
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished `release` profile [optimized] target(s) in 2.46s
    Blocking waiting for file lock on build directory
   Compiling hello_world v0.1.0 (/Users/dylan/Code/solana/projects/hello_world)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 14.29s
     Running unittests src/lib.rs (target/debug/deps/hello_world-823cf88515d0fd05)

running 1 test
[2024-12-06T02:00:47.545448000Z INFO  solana_program_test] "hello_world" SBF program from /Users/dylan/Code/solana/projects/hello_world/target/deploy/hello_world.so, modified 16 seconds, 964 ms, 380 µs and 220 ns ago
[2024-12-06T02:00:47.750627000Z DEBUG solana_runtime::message_processor::stable_log] Program 1111111QLbz7JHiBTspS962RLKV8GndWFwiEaqKM invoke [1]
[2024-12-06T02:00:47.750876000Z DEBUG solana_runtime::message_processor::stable_log] Program log: Hello, world!
[2024-12-06T02:00:47.750906000Z DEBUG solana_runtime::message_processor::stable_log] Program 1111111QLbz7JHiBTspS962RLKV8GndWFwiEaqKM consumed 137 of 200000 compute units
[2024-12-06T02:00:47.750953000Z DEBUG solana_runtime::message_processor::stable_log] Program 1111111QLbz7JHiBTspS962RLKV8GndWFwiEaqKM success
test test::test_hello_world ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.21s

   Doc-tests hello_world

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 测试(NodeJS)

接下来，我们来用 NodeJS 编写测试用例。

首先使用 pnpm 新建工程。

```bash
mkdir hello_world_frontend
cd hello_world_frontend

# 初始化 pnpm 项目
pnpm init
```

接下来安装依赖:

```bash
# 安装必要的依赖
pnpm add -D typescript ts-node @types/node chai ts-mocha solana-bankrun
pnpm add @solana/web3.js solana-bankrun
```

然后，编写测试程序：

```typescript
import {
  PublicKey,
  Transaction,
  TransactionInstruction,
} from "@solana/web3.js";
import { start } from "solana-bankrun";
import { describe, test } from "node:test";
import { assert } from "chai";

describe("hello-solana", async () => {
  // load program in solana-bankrun
  const PROGRAM_ID = PublicKey.unique();
  const context = await start(
    [{ name: "hello_world", programId: PROGRAM_ID }],
    []
  );
  const client = context.banksClient;
  const payer = context.payer;

  test("Say hello!", async () => {
    const blockhash = context.lastBlockhash;
    // We set up our instruction first.
    let ix = new TransactionInstruction({
      // using payer keypair from context to sign the txn
      keys: [{ pubkey: payer.publicKey, isSigner: true, isWritable: true }],
      programId: PROGRAM_ID,
      data: Buffer.alloc(0), // No data
    });

    const tx = new Transaction();
    tx.recentBlockhash = blockhash;
    // using payer keypair from context to sign the txn
    tx.add(ix).sign(payer);

    // Now we process the transaction
    let transaction = await client.processTransaction(tx);

    assert(transaction.logMessages[0].startsWith("Program " + PROGRAM_ID));
    const message = "Program log: " + "Hello, world! GM!GN!";
    console.log("🌈🌈🌈 ");
    console.log(transaction.logMessages[1]);
    // NOTE: transaction.logMesages is an array:
    //
    // [
    //     'Program 11111111111111111111111111111112 invoke [1]',
    //     'Program log: Hello, world! GM!GN!',
    //     'Program 11111111111111111111111111111112 consumed 340 of 200000 compute units',
    //     'Program 11111111111111111111111111111112 success'
    // ]
    assert(transaction.logMessages[1] === message);
    assert(
      transaction.logMessages[2] ===
        "Program log: Our program's Program ID: " + PROGRAM_ID
    );
    assert(
      transaction.logMessages[3].startsWith(
        "Program " + PROGRAM_ID + " consumed"
      )
    );
    assert(transaction.logMessages[4] === "Program " + PROGRAM_ID + " success");
    assert(transaction.logMessages.length == 5);
  });
});
```

首先，我们通过 `start` 函数生成一个 `context`，这个 `context` 里会有和 `bankServer` 交互的 `bankClient` 以及 `payer` 账户。

接下来，通过 `TransactionInstruction` 来准备交易的 `Instruction`，发送交易需要对消息进行签名，这里使用 `payer` 来对交易进行签名，将它放在 `keys` 数组里。

```javascript
let ix = new TransactionInstruction({
  keys: [{ pubkey: payer.publicKey, isSigner: true, isWritable: true }],
  programId: PROGRAM_ID,
  data: Buffer.alloc(0), // No data
});
```

创建一个新的交易指令 (`TransactionInstruction`)，`TransactionInstruction` 的定义及参数类型 `TransactionInstructionCtorFields` 如下：

```typescript
/**
 * Transaction Instruction class
 */
declare class TransactionInstruction {
  /**
   * Public keys to include in this transaction
   * Boolean represents whether this pubkey needs to sign the transaction
   */
  keys: Array<AccountMeta>;
  /**
   * Program Id to execute
   */
  programId: PublicKey;
  /**
   * Program input
   */
  data: Buffer;
  constructor(opts: TransactionInstructionCtorFields);
}

/**
 * List of TransactionInstruction object fields that may be initialized at construction
 */
type TransactionInstructionCtorFields = {
  keys: Array<AccountMeta>;
  programId: PublicKey;
  data?: Buffer;
};
```

关于 `TransactionInstructionCtorFields` 的说明：

- `keys`: 需要签名的公钥（支付者的公钥）。
- `programId`: 程序的 ID。
- `data`: 这里没有附加数据。

然后我们准备 `Transaction` 的数据。

首先 `Transaction` 需要最近的区块哈希，这个可以从 `context` 的 `lastBlockHash` 获取。

```javascript
const blockhash = context.lastBlockhash;
```

下面是创建交易的过程。

```javascript
const tx = new Transaction();
tx.recentBlockhash = blockhash;
tx.add(ix).sign(payer);
```

创建一个新的交易 (`Transaction`) 需要如下步骤：

- 设置最近的区块哈希。
- 添加之前定义的指令（`tx.add`），并使用支付者的密钥对交易进行签名(`.sign`)。

`add` 函数通过 Javascript 的 [Rest parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters) 特性将参数转换成数组类型，每个数组类型的是 `Transaction | TransactionInstruction | TransactionInstructionCtorFields` 的联合类型 `Union Type`。

```typescript
declare class Transaction {
  /**
   * Signatures for the transaction.  Typically created by invoking the
   * `sign()` method
   */
  signatures: Array<SignaturePubkeyPair>;
  /**
   * The first (payer) Transaction signature
   *
   * @returns {Buffer | null} Buffer of payer's signature
   */
  get signature(): Buffer | null;
  /**
   * The transaction fee payer
   */
  feePayer?: PublicKey;
  /**
   * The instructions to atomically execute
   */
  instructions: Array<TransactionInstruction>;
  /**
   * Add one or more instructions to this Transaction
   *
   * @param {Array< Transaction | TransactionInstruction | TransactionInstructionCtorFields >} items - Instructions to add to the Transaction
   */
  add(
    ...items: Array<
      Transaction | TransactionInstruction | TransactionInstructionCtorFields
    >
  ): Transaction;
}
```

创建完交易之后，通过 `client.processTransaction` 发送交易并等到结果。

```javascript
let transaction = await client.processTransaction(tx);
```

这里是 `processTransaction` 的定义：

```typescript
/**
 * A client for the ledger state, from the perspective of an arbitrary validator.
 *
 * The client is used to send transactions and query account data, among other things.
 * Use `start()` to initialize a BanksClient.
 */
export declare class BanksClient {
  constructor(inner: BanksClientInner);
  private inner;
  /**
   * Send a transaction and return immediately.
   * @param tx - The transaction to send.
   */
  sendTransaction(tx: Transaction | VersionedTransaction): Promise<void>;
  /**
   * Process a transaction and return the result with metadata.
   * @param tx - The transaction to send.
   * @returns The transaction result and metadata.
   */
  processTransaction(
    tx: Transaction | VersionedTransaction
  ): Promise<BanksTransactionMeta>;
}
```

其 `inner` 是个 `BanksClient`，除了处理交易外，它还能干很多事情，以下是它的定义。

```typescript
export class BanksClient {
  getAccount(address: Uint8Array, commitment?: CommitmentLevel | undefined | null): Promise<Account | null>
  sendLegacyTransaction(txBytes: Uint8Array): Promise<void>
  sendVersionedTransaction(txBytes: Uint8Array): Promise<void>
  processLegacyTransaction(txBytes: Uint8Array): Promise<BanksTransactionMeta>
  processVersionedTransaction(txBytes: Uint8Array): Promise<BanksTransactionMeta>
  tryProcessLegacyTransaction(txBytes: Uint8Array): Promise<BanksTransactionResultWithMeta>
  tryProcessVersionedTransaction(txBytes: Uint8Array): Promise<BanksTransactionResultWithMeta>
  simulateLegacyTransaction(txBytes: Uint8Array, commitment?: CommitmentLevel | undefined | null): Promise<BanksTransactionResultWithMeta>
  simulateVersionedTransaction(txBytes: Uint8Array, commitment?: CommitmentLevel | undefined | null): Promise<BanksTransactionResultWithMeta>
  getTransactionStatus(signature: Uint8Array): Promise<TransactionStatus | null>
  getTransactionStatuses(signatures: Array<Uint8Array>): Promise<Array<TransactionStatus | undefined | null>>
  getSlot(commitment?: CommitmentLevel | undefined | null): Promise<bigint>
  getBlockHeight(commitment?: CommitmentLevel | undefined | null): Promise<bigint>
  getRent(): Promise<Rent>
  getClock(): Promise<Clock>
  getBalance(address: Uint8Array, commitment?: CommitmentLevel | undefined | null): Promise<bigint>
  getLatestBlockhash(commitment?: CommitmentLevel | undefined | null): Promise<BlockhashRes | null>
  getFeeForMessage(messageBytes: Uint8Array, commitment?: CommitmentLevel | undefined | null): Promise<bigint | null>
}

/**
	 * Process a transaction and return the result with metadata.
	 * @param tx - The transaction to send.
	 * @returns The transaction result and metadata.
	 */
	async processTransaction(
		tx: Transaction | VersionedTransaction,
	): Promise<BanksTransactionMeta> {
		const serialized = tx.serialize();
		const internal = this.inner;
		const inner =
			tx instanceof Transaction
				? await internal.processLegacyTransaction(serialized)
				: await internal.processVersionedTransaction(serialized);
		return new BanksTransactionMeta(inner);
	}
```

`processTransaction` 会先通过 `serialize` 对 transaction 进行序列化，判断属于 `LegacyTransaction` 还是 `VersionedTransaction`，分别调用 `processLegacyTransaction` 或 `processVersionedTransaction` 异步方法，并将结果通过 `BanksTransactionMeta` 返回。

而 `BanksTransactionMeta` 包含了 `logMessages` `returnData` 和 `computeUnitsConsumed` 属性。

```typescript
export class TransactionReturnData {
  get programId(): Uint8Array;
  get data(): Uint8Array;
}
export class BanksTransactionMeta {
  get logMessages(): Array<string>;
  get returnData(): TransactionReturnData | null;
  get computeUnitsConsumed(): bigint;
}
```

其中 `logMessages` 是一个字符串数组，用于存储与交易相关的日志消息。我们可以通过这些日志信息，对测试结果进行验证。

比如，可以通过对 `logMessages[0]` 验证 solana program 被调用时，会输出以 `Program ` + `PROGRAM_ID` 开头的内容:

```javascript
assert(transaction.logMessages[0].startsWith("Program " + PROGRAM_ID));
```

一个简单的 `logMessages` 数组的例子：

```json
[
  "Program 11111111111111111111111111111112 invoke [1]",
  "Program log: Hello, world! GM!GN!",
  "Program log: Our program's Program ID: {program_id}",
  "Program 11111111111111111111111111111112 consumed 443 of 200000 compute units",
  "Program 11111111111111111111111111111112 success"
]
```

值得注意的是，在我们的 solana program 里，第一个 `msg!` 输出的日志是 `Hello, world! GM!GN!`，但是发送交易返回的 `logMessages` 数组里它在数组的第二个元素，这是什么原因呢？

```rust
pub fn process_instruction(
    program_id: &Pubkey,
    _accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello, world! GM!GN!");
    // NOTE: You must not use interpolating string like this, as it will not
    // output the string value correctly.
    //
    // You must use placeholder instead.
    //
    // Below is the transaction.logMessages array when using interpolating string
    //
    // [
    //     'Program 11111111111111111111111111111112 invoke [1]',
    //     'Program log: Hello, world! GM!GN!',
    //     "Program log: Our program's Program ID: {program_id}",
    //     'Program 11111111111111111111111111111112 consumed 443 of 200000 compute units',
    //     'Program 11111111111111111111111111111112 success'
    // ]
    // msg!("Our program's Program ID: {program_id}");
    msg!("Our program's Program ID: {}", program_id);
    Ok(())
}
```

其原因是 solana program 执行时 `program runtime` 会通过 `program_invoke` 函数打印被调用的日志，也就是这里的: `Program 11111111111111111111111111111112 invoke [1]`。关于 `program_invoke` 函数的代码可以在 [anza-xyz/agave](https://github.com/anza-xyz/agave/blob/6c6c26eec4317e06e334609ea686b0192a210092/program-runtime/src/stable_log.rs#L20) 这里找到。

````rust
/// Log a program invoke.
///
/// The general form is:
///
/// ```notrust
/// "Program <address> invoke [<depth>]"
/// ```
pub fn program_invoke(
    log_collector: &Option<Rc<RefCell<LogCollector>>>,
    program_id: &Pubkey,
    invoke_depth: usize,
) {
    ic_logger_msg!(
        log_collector,
        "Program {} invoke [{}]",
        program_id,
        invoke_depth
    );
}
````

接下来的检查可以根据具体的业务场景按部就班的进行。

比如，下面检查 solana program 里第一个 `msg!` 打印的内容：

```javascript
const message = "Program log: " + "Hello, world! GM!GN!";
assert(transaction.logMessages[1] === message);
```

接下来，检查 solana program 里第二个 `msg!` 打印的内容：

```javascript
assert(transaction.logMessages[1] === message);
assert(
  transaction.logMessages[2] ===
    "Program log: Our program's Program ID: " + PROGRAM_ID
);
```

再下来，检查其他日志消息的内容和格式，包括程序的成功消息和消耗的计算单位，并确保日志消息的总数为 `5`。

```javascript
assert(
  transaction.logMessages[3].startsWith("Program " + PROGRAM_ID + " consumed")
);
assert(transaction.logMessages[4] === "Program " + PROGRAM_ID + " success");
assert(transaction.logMessages.length == 5);
```

至此，一个简单的通过 `NodeJS` 编写的测试就写好了。

#### All in one test setup script

如果你比较懒，可以直接运行以下脚本到 `setup.sh`，并运行 `bash setup.sh`。

```bash
# 创建测试目录
mkdir hello_world_frontend
cd hello_world_frontend

# 初始化 pnpm 项目
pnpm init

# 安装必要的依赖
pnpm add -D typescript ts-node @types/node chai ts-mocha solana-bankrun
pnpm add @solana/web3.js solana-bankrun

# 创建 TypeScript 配置文件
cat > tsconfig.json << EOF
{
  "compilerOptions": {
    "target": "es2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
EOF

# 创建源代码目录和测试文件
mkdir -p tests
cat > tests/hello_world.test.ts << EOF
import {
    PublicKey,
    Transaction,
    TransactionInstruction,
  } from "@solana/web3.js";
  import { start } from "solana-bankrun";
  import { describe, test } from "node:test";
  import { assert } from "chai";

  describe("hello-solana", async () => {
    // load program in solana-bankrun
    const PROGRAM_ID = PublicKey.unique();
    const context = await start(
      [{ name: "hello_world", programId: PROGRAM_ID }],
      [],
    );
    const client = context.banksClient;
    const payer = context.payer;

    test("Say hello!", async () => {
        const blockhash = context.lastBlockhash;
        // We set up our instruction first.
        let ix = new TransactionInstruction({
          // using payer keypair from context to sign the txn
          keys: [{ pubkey: payer.publicKey, isSigner: true, isWritable: true }],
          programId: PROGRAM_ID,
          data: Buffer.alloc(0), // No data
        });

        const tx = new Transaction();
        tx.recentBlockhash = blockhash;
        // using payer keypair from context to sign the txn
        tx.add(ix).sign(payer);

        // Now we process the transaction
        let transaction = await client.processTransaction(tx);

        assert(transaction.logMessages[0].startsWith("Program " + PROGRAM_ID));
        const message = "Program log: " + "Hello, world! GM!GN!";
        console.log("🌈🌈🌈 ");
        console.log(transaction.logMessages);
        assert(transaction.logMessages[1] === message);
        assert(
          transaction.logMessages[2] ===
            "Program log: Our program's Program ID: " + PROGRAM_ID,
        );
        assert(
          transaction.logMessages[3].startsWith(
            "Program " + PROGRAM_ID + " consumed",
          ),
        );
        assert(transaction.logMessages[4] === "Program " + PROGRAM_ID + " success");
        assert(transaction.logMessages.length == 5);
      });
});
EOF

# 更新 package.json 添加测试脚本
cat > package.json << EOF
{
  "name": "hello_world_frontend",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "pnpm ts-mocha -p ./tsconfig.json -t 1000000 ./tests/hello_world.test.ts"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@types/jest": "^29.5.11",
    "@types/node": "^20.10.5",
    "chai": "^5.1.2",
    "jest": "^29.7.0",
    "solana-bankrun": "^0.4.0",
    "ts-jest": "^29.1.1",
    "ts-mocha": "^10.0.0",
    "ts-node": "^10.9.2",
    "typescript": "^5.3.3"
  },
  "dependencies": {
    "@solana/web3.js": "^1.87.6"
  }
}

# 运行测试
pnpm test
EOF
```

# Frontend

我们有两种方法来开发 solana frontend：

1. 使用 Anchor 框架
2. 不使用 Anchor 框架

我会帮你实现两种方法来开发 Solana frontend。让我们从最基础的开始，逐步构建。

## 1. 不使用 Anchor 框架

首先创建一个新的 Next.js 项目：

```bash
npx create-next-app@latest solana-frontend-nextjs --typescript --tailwind --eslint
cd solana-frontend-nextjs
```

安装必要的依赖：

```bash
pnpm install \
  @solana/web3.js \
  @solana/wallet-adapter-react \
  @solana/wallet-adapter-react-ui \
  @solana/wallet-adapter-base \
  @solana/wallet-adapter-wallets
```

### 1.1 基础设置

首先创建钱包配置文件：

```typescript:src/context/WalletContextProvider.tsx
'use client'

import { FC, ReactNode, useMemo } from "react";
import {
  ConnectionProvider,
  WalletProvider,
} from "@solana/wallet-adapter-react";
import { WalletModalProvider } from "@solana/wallet-adapter-react-ui";
import { clusterApiUrl } from "@solana/web3.js";
import {
  PhantomWalletAdapter,
  SolflareWalletAdapter,
} from "@solana/wallet-adapter-wallets";

require("@solana/wallet-adapter-react-ui/styles.css");

export const WalletContextProvider: FC<{ children: ReactNode }> = ({ children }) => {
  const url = useMemo(() => clusterApiUrl("devnet"), []);
  const wallets = useMemo(
    () => [
      new PhantomWalletAdapter(),
      new SolflareWalletAdapter(),
    ],
    []
  );

  return (
    <ConnectionProvider endpoint={url}>
      <WalletProvider wallets={wallets} autoConnect>
        <WalletModalProvider>{children}</WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
};
```

更新 layout 文件：

```typescript:src/app/layout.tsx
import { WalletContextProvider } from '@/context/WalletContextProvider'
import './globals.css'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        <WalletContextProvider>
          {children}
        </WalletContextProvider>
      </body>
    </html>
  )
}
```

### 1.2 创建主页面组件

注意，要在 `src/app/page.tsx` 文件中，将 `PROGRAM_ID` 替换为你的程序 ID。

```typescript:src/app/page.tsx
'use client'

import { useConnection, useWallet } from '@solana/wallet-adapter-react'
import { WalletMultiButton } from '@solana/wallet-adapter-react-ui'
import { LAMPORTS_PER_SOL, PublicKey, Transaction, TransactionInstruction } from '@solana/web3.js'
import { FC, useState } from 'react'

const Home: FC = () => {
  const { connection } = useConnection()
  const { publicKey, sendTransaction } = useWallet()
  const [loading, setLoading] = useState(false)

  // 替换为你的程序 ID
  const PROGRAM_ID = new PublicKey("3KUbj4gMH77adZnZhatXutJ695qCGzB6G8cmMU1SYMWW")

  const sayHello = async () => {
    if (!publicKey) {
      alert("Please connect your wallet!")
      return
    }

    setLoading(true)
    try {
      const instruction = new TransactionInstruction({
        keys: [
          {
            pubkey: publicKey,
            isSigner: true,
            isWritable: true,
          },
        ],
        programId: PROGRAM_ID,
        data: Buffer.from([]),
      })

      const transaction = new Transaction()
      transaction.add(instruction)

      const signature = await sendTransaction(transaction, connection)
      await connection.confirmTransaction(signature)

      alert("Transaction successful!")
    } catch (error) {
      console.error(error)
      alert(`Error: ${error instanceof Error ? error.message : String(error)}`)
    } finally {
      setLoading(false)
    }
  }

  return (
    <main className="flex min-h-screen flex-col items-center justify-between p-24">
      <div className="z-10 max-w-5xl w-full items-center justify-between font-mono text-sm">
        <div className="flex flex-col items-center gap-8">
          <h1 className="text-4xl font-bold">Solana Hello World</h1>
          <WalletMultiButton />
          {publicKey && (
            <button
              onClick={sayHello}
              disabled={loading}
              className="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
            >
              {loading ? "Processing..." : "Say Hello"}
            </button>
          )}
        </div>
      </div>
    </main>
  )
}

export default Home
```

### 1.3 运行项目

运行:

```bash
pnpm dev
```

点击 `Say Hello` 按钮通过 phantom wallet 发送交易，交易成功之后，可以在 explorer 上看到交易详情：

https://explorer.solana.com/tx/4H3nfuDqaz1s6TDGe3HSL6DsEvq9r3TwcGqqw9kfGGk3c9pjK2HGohmCrfWcZCFXdMJsPobsbcj3UAdmkj2QK8vd?cluster=devnet

## 2. 使用 Anchor 框架

创建新项目：

```bash
npx create-next-app@latest solana-anchor-frontend-nextjs --typescript --tailwind --eslint
cd solana-anchor-frontend-nextjs
```

安装依赖：

```bash
pnpm install \
  @coral-xyz/anchor \
  @solana/web3.js \
  @solana/wallet-adapter-react \
  @solana/wallet-adapter-react-ui \
  @solana/wallet-adapter-base \
  @solana/wallet-adapter-wallets
```

### 2.1 创建 Anchor IDL 类型

```typescript:src/idl/hello_world.ts
export type HelloWorld = {
  "version": "0.1.0",
  "name": "hello_world",
  "instructions": [
    {
      "name": "sayHello",
      "accounts": [],
      "args": []
    }
  ]
};

export const IDL: HelloWorld = {
  "version": "0.1.0",
  "name": "hello_world",
  "instructions": [
    {
      "name": "sayHello",
      "accounts": [],
      "args": []
    }
  ]
};
```

### 2.2 创建 Anchor 工作区提供者

```typescript:src/context/WorkspaceProvider.tsx
"use client";

import { createContext, useContext, ReactNode } from "react"
import { Program, AnchorProvider } from "@coral-xyz/anchor"
import { AnchorWallet, useAnchorWallet, useConnection } from "@solana/wallet-adapter-react"
import { HelloWorld, IDL } from "@/idl/hello_world"
import { PublicKey } from "@solana/web3.js"

const WorkspaceContext = createContext({})

interface Workspace {
  program?: Program<HelloWorld>
}

export const WorkspaceProvider = ({ children }: { children: ReactNode }) => {
  const { connection } = useConnection()
  const wallet = useAnchorWallet()

  const provider = new AnchorProvider(
    connection,
    wallet as AnchorWallet,
    AnchorProvider.defaultOptions()
  )

  const program = new Program(
    IDL,
    new PublicKey("3KUbj4gMH77adZnZhatXutJ695qCGzB6G8cmMU1SYMWW"),
    provider
  )

  const workspace = {
    program,
  }

  return (
    <WorkspaceContext.Provider value={workspace}>
      {children}
    </WorkspaceContext.Provider>
  )
}

export const useWorkspace = (): Workspace => {
  return useContext(WorkspaceContext) as Workspace
}
```

### 2.3 更新布局组件

```typescript:src/app/layout.tsx
import { WalletContextProvider } from '@/context/WalletContextProvider'
import { WorkspaceProvider } from '@/context/WorkspaceProvider'
import './globals.css'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        <WalletContextProvider>
          <WorkspaceProvider>
            {children}
          </WorkspaceProvider>
        </WalletContextProvider>
      </body>
    </html>
  )
}
```

### 2.4 创建主页面组件

```typescript:src/app/page.tsx
'use client'

import { useWallet } from '@solana/wallet-adapter-react'
import { WalletMultiButton } from '@solana/wallet-adapter-react-ui'
import { FC, useState } from 'react'
import { useWorkspace } from '@/context/WorkspaceProvider'

const Home: FC = () => {
  const { publicKey } = useWallet()
  const { program } = useWorkspace()
  const [loading, setLoading] = useState(false)

  const sayHello = async () => {
    if (!publicKey || !program) {
      alert("Please connect your wallet!")
      return
    }

    setLoading(true)
    try {
      const tx = await program.methods
        .sayHello()
        .accounts({})
        .rpc()

      alert(`Transaction successful! Signature: ${tx}`)
    } catch (error) {
      console.error(error)
      alert(`Error: ${error instanceof Error ? error.message : String(error)}`)
    } finally {
      setLoading(false)
    }
  }

  return (
    <main className="flex min-h-screen flex-col items-center justify-between p-24">
      <div className="z-10 max-w-5xl w-full items-center justify-between font-mono text-sm">
        <div className="flex flex-col items-center gap-8">
          <h1 className="text-4xl font-bold">Solana Hello World (Anchor)</h1>
          <WalletMultiButton />
          {publicKey && (
            <button
              onClick={sayHello}
              disabled={loading}
              className="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
            >
              {loading ? "Processing..." : "Say Hello"}
            </button>
          )}
        </div>
      </div>
    </main>
  )
}

export default Home
```

### 2.5 tsconfig.json 配置

为了正确使用 `@` 路径别名，需要配置 `tsconfig.json` 文件：

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./*"],
      "@/idl/*": ["./app/idl/*"],
      "@/context/*": ["./app/context/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

这个配置文件增加了 `@/idl/*` 和 `@/context/*` 的别名，以便在代码中使用这些路径。

### 2.6 运行项目

运行:

```bash
pnpm dev
```

点击 `Say Hello` 按钮通过 phantom wallet 发送交易，交易成功之后，可以在 explorer 上看到交易详情：

https://explorer.solana.com/tx/5dustfzfhSopVKrDiL3CoXAg35jimMBs3oFkxDsiBqM1xQ6t4JnsonbdZirzYdR5i5HGsUKmfhKZb3NQunWDbWiw?cluster=devnet

## 两种方法的主要区别

1. 不使用 Anchor：

- 直接使用 `@solana/web3.js` 创建交易和指令
- 手动构建交易结构
- 更底层的控制

2. 使用 Anchor：

- 使用 Anchor IDL 类型定义
- 更高级的抽象和类型安全
- 更简洁的程序调用方式
- 更好的开发体验

选择哪种方法取决于你的需求：

- 如果需要更多底层控制或项目较小，可以选择不使用 Anchor
- 如果需要更好的开发体验和类型安全，建议使用 Anchor

# 下一步

至此，我们已经完成了一个最基础的 Solana 程序的开发和部署。虽然这个程序只是简单地打印 "Hello, world!"，但它包含了 Solana 程序开发的基本要素：

- 程序入口点的定义
- 基本的参数结构
- 构建和部署流程

在接下来的内容中，我们将学习：

- 如何使用 Anchor 框架开发程序
- 如何处理账户数据
- 如何实现更复杂的指令逻辑
- 如何进行程序测试
- 如何确保程序安全性

敬请期待吧！

# Refs

关于 cargo-build-sbf 解释
https://github.com/solana-labs/solana/issues/34987#issuecomment-1913538260

https://solana.stackexchange.com/questions/16443/error-function-stack-offset-of-7256-exceeded-max-offset-of-4096-by-3160-bytes

安装 solana cli tool suites（注意不要安装 edge 版本，会发现部署不成功问题）
https://solana.com/docs/intro/installation

https://github.com/solana-labs/solana/issues/34987#issuecomment-1914665002
https://github.com/anza-xyz/agave/issues/1572

在 solana 编写一个 helloworld
https://solana.com/developers/guides/getstarted/local-rust-hello-world#create-a-new-rust-library-with-cargo

solana wallet nextjs setup
https://solana.com/developers/guides/wallets/add-solana-wallet-adapter-to-nextjs

https://solana.com/developers/cookbook/wallets/connect-wallet-react
https://www.anza.xyz/blog/solana-web3-js-2-release

https://solana.stackexchange.com/questions/1723/anchor-useanchorwallet-vs-solanas-usewallet

anchor client side development
https://solana.com/developers/courses/onchain-development/intro-to-anchor-frontend
