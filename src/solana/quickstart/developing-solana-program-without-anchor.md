# Developing Solana Program Without Anchor

<!-- toc -->

# Initialize Project

## Initialize Project with Cargo

We can use cargo to initialize the project.

```bash
cargo init hello_world --lib
```

# Write Code

## Program Entrypoint

We can use `entrypoint` macro to write the program entrypoint.

`entrypoint` macro needs a function parameter, which is the entrypoint function of the solana program.

```rust
pub fn process_instruction() -> ProgramResult {
    msg!("Hello, world!");
    Ok(())
}
```

If the function signature passed to the `entrypoint` macro does not meet the requirements, the compilation will fail with an error:

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

## Modify the signature of the process_instruction function

Add three parameters to the `process_instruction` function:

- `program_id`: `&Pubkey` type, representing the public key address of the current program
- `accounts`: `&[AccountInfo]` type, representing the reference of an array of AccountInfo, which contains all the account information involved in the transaction
- `instruction_data`: `&[u8]` type, representing the input data of the instruction, passed in as a byte array

These three parameters are the basic elements of the Solana program execution:

- `program_id` is used to verify the program identity and permissions
- `accounts` contains all the account data that the program needs to read or modify
- `instruction_data` carries the specific instruction data passed in when calling the program

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

Note that the parameter names are prefixed with an underscore (`_`) here, because we don't need to use these parameters in this simple example, which avoids compiler warnings about unused variables. In actual development, these parameters are very important, and we will explain how to use them in detail in subsequent examples.

We can also refer to the [documentation of the `solana_program_entrypoint` crate](https://docs.rs/solana-program-entrypoint/latest/solana_program_entrypoint/macro.entrypoint.html) for the function signature:

```rust
/// fn process_instruction(
///     program_id: &Pubkey,      // Public key of the account the program was loaded into
///     accounts: &[AccountInfo], // All accounts required to process the instruction
///     instruction_data: &[u8],  // Serialized instruction-specific data
/// ) -> ProgramResult;
```

# Build Program

## Build Program with cargo build-sbf

To build the solana program, we need to use the `cargo build-sbf` program.

```bash
cargo build-sbf
```

The build failed, and here is the error message.

```
dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf
error: package `solana-program v2.1.4` cannot be built because it requires rustc 1.79.0 or newer, while the currently active rustc version is 1.75.0-dev
Either upgrade to rustc 1.79.0 or newer, or use
cargo update solana-program@2.1.4 --precise ver
where `ver` is the latest version of `solana-program` supporting rustc 1.75.0-dev
```

We can check the `rustc` version information by using the `--version` parameter.

```bash
cargo-build-sbf --version
```

Output:

```
solana-cargo-build-sbf 1.18.25
platform-tools v1.41
rustc 1.75.0
```

Regarding the version mismatch between the system Rust compiler and the Rust compiler used by build-sbf, you can refer to this issue.
https://github.com/solana-labs/solana/issues/34987

## Solve the build-sbf compilation failure problem

One way is to use an old version of `solana-program`, such as the `=1.17.0` version.

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

But after running `cargo build-sbf`, another error occurred.

```bash
error: failed to parse lock file at: /Users/dylan/Code/solana/projects/hello_world/Cargo.lock

Caused by:
  lock file version 4 requires `-Znext-lockfile-bump`
```

It seems that the `cargo` version used by `build-sbf` does not support the `Cargo.lock` file version = 4, which is automatically generated by the editor (vscode/cursor).

Install the `stable` version of the solana cli tool chain: `sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"`, but it still failed to compile, and the error is as follows:

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

When compiling with `cargo build-sbf`, you need to download the corresponding version of `platform-tools`, because the `v1.42` version of `platform-tools` for Mac (Intel) has not been released yet, so the above command failed to run.

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

Using the `beta` version of the solana cli tool suites, it could compile, but it encountered this error:

`Exceeding the maximum stack offset may cause undefined behavior during execution.`

```
   Compiling bincode v1.3.3
Error: Function _ZN112_$LT$solana_program..instruction..InstructionError$u20$as$u20$solana_frozen_abi..abi_example..AbiEnumVisitor$GT$13visit_for_abi17hc69c00f4c61717f8E Stack offset of 6640 exceeded max offset of 4096 by 2544 bytes, please minimize large stack variables. Estimated function frame size: 6680 bytes. Exceeding the maximum stack offset may cause undefined behavior during execution.
```

The specific reason is still the version problem, and the analysis can be referred to:
https://solana.stackexchange.com/questions/16443/error-function-stack-offset-of-7256-exceeded-max-offset-of-4096-by-3160-bytes

After updating the `solana-program` version to `2.1.4` (run `sh -c "$(curl -sSfL https://release.anza.xyz/v2.1.4/install)"`), use the following tool chain to compile:

```bash
> cargo build-sbf --version
solana-cargo-build-sbf 2.1.4
platform-tools v1.43
rustc 1.79.0

# solana-cargo-build-sbf 2.2.0
# platform-tools v1.43
# rustc 1.79.0
```

Run `cargo build-sbf`:

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

Finally, the compilation succeeded, let's celebrate with a bottle of champagne!

Here is the `Cargo.toml` file:

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

## Build Product

`cargo build-sbf` is a special build command provided by Solana, which compiles a Rust program into BPF (Berkeley Packet Filter) bytecode that can be executed in the Solana runtime environment. This command does the following:

1. Use a specific Rust toolchain to compile the code

   - Use a Rust compiler optimized for Solana
   - Use the `bpfel-unknown-unknown` target platform
   - Enable release mode optimization

2. Generate necessary deployment files

   - Compile the `.so` file (shared object file)
   - Generate the program keypair (if it does not exist)
   - Optimize and compress the final binary file

3. Verify the compilation result
   - Check if the program size is within the limit
   - Verify if the program format is correct

The command execution process:

1. First, check and download the necessary toolchain
2. Use cargo to compile the project
3. Post-process the compilation result (e.g., strip debugging information)
4. Place the final file in the `target/deploy` directory

This command replaces the earlier `cargo build-bpf`, providing a better build experience and more modern toolchain support.

Let's see what files are generated. After running the `cargo build-sbf` command, two important files will be generated in the `target/deploy` directory:

- `hello_world.so`：The compiled program file, which is an executable file in BPF (Berkeley Packet Filter) format
- `hello_world-keypair.json`：The program keypair file, used for program deployment and upgrade

If you see output similar to the following, it indicates a successful build:

```bash
BPF SDK: /Users/username/.local/share/solana/install/releases/1.14.x/solana-release/bin/sdk/bpf
cargo-build-sbf child: rustup toolchain list -v
cargo-build-sbf child: cargo +bpf build --target bpfel-unknown-unknown --release
    Finished release [optimized] target(s) in 0.20s
cargo-build-sbf child: /Users/username/.local/share/solana/install/releases/1.14.x/solana-release/bin/sdk/bpf/scripts/strip.sh /Users/username/projects/hello_world/target/bpfel-unknown-unknown/release/hello_world.so /Users/username/projects/hello_world/target/deploy/hello_world.so
```

# Deploy

Now we can deploy the compiled program to the Solana network. In the development stage, we usually use the local testnet (localhost) or the devnet for testing.

First, ensure your Solana CLI is configured to the correct cluster:

```bash
# 切换到开发网
solana config set --url devnet
# 切换到本地测试网
solana config set --url localnet

# 查看当前配置
solana config get
```

Then use the following command to deploy the program:

```bash
solana program deploy target/deploy/hello_world.so
```

After successful deployment, you will see the program ID (public key address). Please save this address, as it will be needed for future interactions with the program.

But when we deploy the program by running the `solana program deploy` command, the deployment failed.

```bash
dylan@smalltown ~/Code/solana/projects/helloworld (master)> solana program deploy ./target/deploy/helloworld.so
⠁   0.0% | Sending 1/173 transactions               [block height 2957; re-sign in 150 blocks]
    thread 'main' panicked at quic-client/src/nonblocking/quic_client.rs:142:14:
QuicLazyInitializedEndpoint::create_endpoint bind_in_range: Os { code: 55, kind: Uncategorized, message: "No buffer space available" }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

So what does this `No buffer space available` mean?

After a long time of troubleshooting, I finally found that the problem was probably caused by **version** issues. Since the program created by the `Anchor` project can be deployed normally.

Here is the version information of the `solana` command:

```bash
> solana --version
solana-cli 2.2.0 (src:67704836; feat:1081947060, client:Agave)
```

## Back to the Anchor project to verify that the deployment failure is due to version issues

We can create a new project by running `anchor init helloworld` and deploy the program by running `anchor build` and `anchor deploy`.

```bash
anchor init helloworld
cd helloworld
anchor build
anchor deploy
```

From the error message, it is found that the newly generated anchor project also encounters the same error when deploying: `No buffer space available`

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

Check the anchor version:

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

Check the solana version:

```bash
> solana --version
solana-cli 2.2.0 (src:67704836; feat:1081947060, client:Agave)
```

This `2.2.0` version looks strange, and I suddenly thought that I installed the edge version of solana cli to compile the solana program, and the version of the solana cli it carries is `2.2.0`:

```bash
sh -c "$(curl -sSfL https://release.anza.xyz/edge/install)"
```

So I switched back to the `stable` version:

```bash
> sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
downloading stable installer
  ✨ stable commit fbead11 initialized
```

The stable version of solana is `2.0.19`.

```bash
> solana --version
solana-cli 2.0.19 (src:fbead118; feat:607245837, client:Agave)
```

Before re-deploying the program, let's clean up the `buffers` of the previously failed program, which is buffer accounts. For what is a buffer account, please refer to Tips 3.

- Check all buffer accounts: `solana program show --buffers`
- Close all buffer accounts: `solana program close --buffers`
  - Closing buffer accounts can reclaim SOL stored in buffer accounts

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

After cleaning up the buffer accounts, we also switched back to the `stable` version of the solana cli, and we tried to deploy the program again:

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

Success 🎉, let's celebrate with a bottle of champagne!

This further deepened our suspicion that the version issue was causing the program to fail to deploy.

## Back to deploy our hello_world project

After verifying that the deployment failure was not due to the project type (anchor project or cargo project), we returned to the `cargo init` created project: `hello_world`.

We can deploy the program through the `solana` subcommand: run `solana program deploy ./target/deploy/helloworld.so` to deploy the program.

We will deploy on both `localnet` and `devnet`.

### localnet deployment

First, let's deploy on `localnet`.

Switch to localnet:

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

Deploy the program:

```bash
dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana program deploy ./target/deploy/hello_world.so
Program Id: DhQr1KGGQcf8BeU5uQvR35p2kgKqEinD45PRTDDRqx7z

Signature: 3WVEWN4NUodsb8ZDjbjrTWXLikZ7wbWCuzuRZtSBmyKL4kVvESSeLwKZ3cJo1At4vDcaBs5iEcHhdteyXCwqwmDw
```

### devnet deployment

Here is the `devnet` deployment.

Switch to localnet:

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

We can check the balance before and after deployment through `solana balance`.

```bash
# Balance before deployment
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana balance
75.153619879 SOL

# Balance after deployment
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> solana balance
75.152378439 SOL
```

The version at this time:

```bash
dylan@smalltown ~/Code/solana/projects/helloworld (master)> solana --version
solana-cli 2.0.19 (src:fbead118; feat:607245837, client:Agave)
```

It can be seen that it is not recommended to use the latest version (solana-cli `2.2.0`), otherwise it will be counterproductive.

# Tips

## Tip 1: Keep the version of solana cli consistent with the version in Cargo.toml

In the [solana official tutorial](https://solana.com/developers/guides/getstarted/local-rust-hello-world#create-a-new-rust-library-with-cargo) it mentions this Tip:

> It is highly recommended to keep your solana-program and other Solana Rust dependencies in-line with your installed version of the Solana CLI. For example, if you are running Solana CLI 2.0.3, you can instead run:

```bash
cargo add solana-program@"=2.0.3"
```

> This will ensure your crate uses only 2.0.3 and nothing else. If you experience compatibility issues with Solana dependencies, check out the

## Tip 2: Do not add solana-sdk to dependencies, as it is only used offchain

Refer to the explanation here:
https://solana.stackexchange.com/questions/9109/cargo-build-bpf-failed

> I have identified the issue. The solana-sdk is designed for off-chain use only, so it should be removed from the dependencies.

Error when adding `solana-sdk` to dependencies:

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

## Tip 3: About buffer accounts

In Solana, buffer accounts are used as temporary accounts during the program deployment process. They are an important mechanism for deploying programs on Solana. Due to the transaction size limit of `1232` bytes, deploying a program usually requires multiple transaction steps. In this process, the buffer account serves as a storage for the program's bytecode until the deployment is complete.

Key points about buffer accounts:

- Temporary storage: Buffer accounts are used to store the program's bytecode, ensuring that large programs can be processed during deployment.
- Automatic closure: Once the program is successfully deployed, the associated buffer accounts are automatically closed, releasing the allocated resources.
- Failure handling: If the deployment fails, the buffer accounts are not automatically deleted. Users can choose:
  - Continue using the existing buffer account to complete the deployment.
  - Close buffer accounts to reclaim the allocated SOL (rent).
- Check buffer accounts: You can check if there are any unclosed buffer accounts using the command `solana program show --buffers`.
- Close buffer accounts: You can close buffer accounts using the command `solana program close --buffers`.

For an explanation of the process of deploying a Solana program, you can refer to the official documentation: https://solana.com/docs/programs/deploying#program-deployment-process

# Redeploy

Redeploying only requires editing the code and running `cargo build-sbf` to compile the code, then deploying through `solana program deply ./target/deploy/hello_world.so`.

```bash
cargo build-sbf
solana program deploy ./target/deploy/hello_world.so
```

You can verify that the new version of the program is running by running tests and client scripts.

```bash
# Run tests
cargo test-sbf
# Run client script
cargo run --example client
```

For example, if I modify the `msg!` input content to `Hello, world! GM!GN!`, running tests and client scripts will see this output in the log.

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

Run tests:

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

# Best practices

## Best practice for installing solana-cli

The best way is to install a specific version of solana cli, such as using the following method to install the `2.0.3` version:

```bash
# Installing stable and beta are not recommended
# sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
# sh -c "$(curl -sSfL https://release.anza.xyz/beta/install)"
# Recommended to install a specific version
sh -c "$(curl -sSfL https://release.anza.xyz/v2.0.3/install)"
```

Output:

```
downloading v2.0.3 installer
  ✨ 2.0.3 initialized
```

Run `cargo build-sbf --version` to check the version of `cargo build-sbf`:

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> cargo build-sbf --version
solana-cargo-build-sbf 2.0.3
platform-tools v1.41
rustc 1.75.0
```

It can be seen that the rustc version here is `1.75.0`, which is relatively old, and the compilation must include the `-Znext-lockfile-bump` parameter, otherwise it will compile with an error:

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master)> cargo build-sbf
info: uninstalling toolchain 'solana'
info: toolchain 'solana' uninstalled
error: failed to parse lock file at: /Users/dylan/Code/solana/projects/hello_world/Cargo.lock

Caused by:
  lock file version 4 requires `-Znext-lockfile-bump`
```

Here is the complete compilation process after passing the `-Znext-lockfile-bump` parameter:

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

It is worth noting that both installing the stable version and the beta version will cause compilation failures. Running `cargo build-sbf` on the stable version will download the platform-tools for the `x86_64` architecture from the github release page, but the official release does not provide platform-tools for this version. Here is the error information:

```bash
(base) dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> cargo build-sbf --version
solana-cargo-build-sbf 2.0.19
platform-tools v1.42

(base) dylan@smalltown ~/Code/solana/projects/hello_world (master) [1]> cargo build-sbf
[2024-12-05T06:17:30.547088000Z ERROR cargo_build_sbf] Failed to install platform-tools: HTTP status client error (404 Not Found) for url (https://github.com/anza-xyz/platform-tools/releases/download/v1.42/platform-tools-osx-x86_64.tar.bz2)
```

It is found that if `--tools-version` is specified as `v1.43`, it will also fail to compile.

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

So it is best to install a specific version of the solana cli.

# How to view deployed program

We can view the deployed program by accessing the following address.

https://explorer.solana.com/?cluster=custom

It will automatically use the local localhost:8899 as the rpc endpoint, and search for the program id in the search bar to view the transaction details.

# Client call

## Client call program (Rust) (invoke solana program)

First, create the `examples` directory, and then create the `client.rs` file in the `examples` directory.

```bash
mkdir -p examples
touch examples/client.rs
```

Add the following content to `Cargo.toml`:

```toml
[[example]]
name = "client"
path = "examples/client.rs"
```

Add `solana-client` dependency:

```bash
cargo add solana-client@1.18.26 --dev
```

Add the following code to `examples/client.rs`, note to replace the program ID you deployed:

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

This simple script can invoke the deployed solana program, which mainly does the following:

- Connect to the local RPC
- Create a new account
- Airdrop 1 SOL to the new account
- Create the instruction (Instruction) required for the hello_world program
- Send the transaction (through `send_and_confirm_transaction`)

About the program ID, we can get the program ID through the `solana address -k <program keypair>.json` command:

```bash
solana address -k ./target/deploy/hello_world-keypair.json
```

The `-k` parameter receives the keypair file, and can obtain the PublicKey.

Run client:

```bash
cargo run --example client
```

Run client code output:

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

## Client call (TypeScript)

We can send transactions by creating a nodejs project:

```bash
mkdir -p helloworld
npm init -y
npm install --save-dev typescript
npm install @solana/web3.js@1 @solana-developers/helpers@2
```

Create the `tsconfig.json` configuration file:

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

Create the `hello-world-client.ts` file, note to modify the `PublicKey` parameter to the programID you deployed:

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

Run:

```bash
npx ts-node hello-world-client.ts
```

Output:

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

# Some experiments

## Which versions can successfully compile and test

First, check the versions of `build-sbf` and `test-sbf` we installed:

```bash
# build-sbf version
> cargo build-sbf --version
solana-cargo-build-sbf 2.1.4
platform-tools v1.43
rustc 1.79.0

# test-sbf version
> cargo test-sbf --version
solana-cargo-test-sbf 2.1.4
```

We test which versions can compile and test correctly through this command: `rm -rf target Cargo.lock && cargo build-sbf && cargo test-sbf`

| version   | DevDependencies & Dependencies                                                                             | NOTE           |
| --------- | ---------------------------------------------------------------------------------------------------------- | -------------- |
| ✅2.1.4   | `cargo add solana-sdk@2.1.4 solana-program-test@2.1.4 tokio --dev && cargo add solana-program@2.1.4`       | latest version |
| ✅2.0.18  | `cargo add solana-sdk@2.0.18 solana-program-test@2.0.18 tokio --dev && cargo add solana-program@2.0.18`    | latest version |
| ✅2.0.3   | `cargo add solana-sdk@2.0.3 solana-program-test@2.0.3 tokio --dev && cargo add solana-program@2.0.3`       |                |
| ✅1.18.26 | `cargo add solana-sdk@1.18.26 solana-program-test@1.18.26 tokio --dev && cargo add solana-program@1.18.26` |                |

Here is an example of `Cargo.toml` (corresponding to version `2.0.3`):

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

# Test

About the test of solana program, we usually use:

bankrun is a lightweight framework for testing Solana programs in Node.js. Compared to the traditional solana-test-validator, it provides higher speed and convenience. It can achieve some functions that solana-test-validator cannot, such as time rollback and dynamic setting of account data.

It starts a lightweight BanksServer, which is similar to an RPC node but faster, and creates a BanksClient to communicate with the server.

Main features:

- High efficiency: Much faster than solana-test-validator.
- Flexibility: Supports time rollback and dynamic account data setting.
- solana-bankrun is based on solana-program-test, using a lightweight BanksServer and BanksClient.

Next, let's see how to write test cases using Rust (`solana-program-test`) and NodeJS (`solana-bankrun`).

## Test (Rust)

First, let's use Rust code to test.

First, install the dependencies required for testing:

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

We have tested that versions `2.1.4`, `2.0.18`, `2.0.3`, and `1.18.26` can successfully compile and test, so we only selected version `1.18.26` for demonstration.

Test result output:

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

## Test (NodeJS)

Next, let's use NodeJS to write test cases.

First, use pnpm to create a new project.

```bash
mkdir hello_world_frontend
cd hello_world_frontend

# Initialize pnpm project
pnpm init
```

Next, install the dependencies:

```bash
# Install necessary dependencies
pnpm add -D typescript ts-node @types/node chai ts-mocha solana-bankrun
pnpm add @solana/web3.js solana-bankrun
```

Next, write the test program:

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

First, we generate a `context` through the `start` function, which will have the `bankClient` and `payer` account for interacting with the `bankServer`.

Next, we prepare the transaction `Instruction` through the `TransactionInstruction` class, and send the transaction by signing the message with the `payer` account.

```javascript
let ix = new TransactionInstruction({
  keys: [{ pubkey: payer.publicKey, isSigner: true, isWritable: true }],
  programId: PROGRAM_ID,
  data: Buffer.alloc(0), // No data
});
```

Create a new transaction instruction (`TransactionInstruction`), the definition and parameter type `TransactionInstructionCtorFields` of `TransactionInstruction` are as follows:

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

About the `TransactionInstructionCtorFields`:

- `keys`: The public key (payer's public key) to sign.
- `programId`: The ID of the program.
- `data`: There is no additional data.

Next, we prepare the data of `Transaction`.

First, `Transaction` needs the latest block hash, which can be obtained from `context.lastBlockhash`.

```javascript
const blockhash = context.lastBlockhash;
```

Next, we create the transaction.

```javascript
const tx = new Transaction();
tx.recentBlockhash = blockhash;
tx.add(ix).sign(payer);
```

Creating a new transaction (`Transaction`) requires the following steps:

- Set the latest block hash.
- Add the previously defined instruction (`tx.add`), and sign the transaction with the payer's key (`.sign`).

The `add` function converts parameters to an array type using Javascript's [Rest parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters) feature, and each array type is a union type of `Transaction | TransactionInstruction | TransactionInstructionCtorFields`.

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

After creating the transaction, send it through `client.processTransaction` and wait for the result.

```javascript
let transaction = await client.processTransaction(tx);
```

Here is the definition of `processTransaction`:

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

Its `inner` is a `BanksClient`, which can do many things besides processing transactions, as shown below.

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

`processTransaction` first serializes the transaction through `serialize`, determines whether it belongs to `LegacyTransaction` or `VersionedTransaction`, and then calls the asynchronous methods `processLegacyTransaction` or `processVersionedTransaction` respectively, and returns the result through `BanksTransactionMeta`.

And `BanksTransactionMeta` contains the `logMessages`, `returnData`, and `computeUnitsConsumed` properties.

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

Among them, `logMessages` is a string array used to store log messages related to the transaction. We can verify the test results through these log messages.

For example, you can verify that the solana program is invoked by checking the `logMessages[0]` to start with `Program ` + `PROGRAM_ID`:

```javascript
assert(transaction.logMessages[0].startsWith("Program " + PROGRAM_ID));
```

An example of a simple `logMessages` array:

```json
[
  "Program 11111111111111111111111111111112 invoke [1]",
  "Program log: Hello, world! GM!GN!",
  "Program log: Our program's Program ID: {program_id}",
  "Program 11111111111111111111111111111112 consumed 443 of 200000 compute units",
  "Program 11111111111111111111111111111112 success"
]
```

Note that in our solana program, the first `msg!` output is `Hello, world! GM!GN!`, but in the `logMessages` array returned by sending the transaction, it is the second element. What is the reason for this?

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

The reason is that when the solana program is executed, the `program runtime` prints the invoked log through the `program_invoke` function, which is here: `Program 11111111111111111111111111111112 invoke [1]`. You can find the code of the `program_invoke` function in [anza-xyz/agave](https://github.com/anza-xyz/agave/blob/6c6c26eec4317e06e334609ea686b0192a210092/program-runtime/src/stable_log.rs#L20).

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

The subsequent checks can be performed step by step according to the specific business scenario.

For example, the following checks the content of the first `msg!` print in the solana program:

```javascript
const message = "Program log: " + "Hello, world! GM!GN!";
assert(transaction.logMessages[1] === message);
```

Next, check the content of the second `msg!` print in the solana program:

```javascript
assert(transaction.logMessages[1] === message);
assert(
  transaction.logMessages[2] ===
    "Program log: Our program's Program ID: " + PROGRAM_ID
);
```

Next, check the content and format of other log messages, including the success message of the program and the consumed compute units, and ensure the total number of log messages is `5`.

```javascript
assert(
  transaction.logMessages[3].startsWith("Program " + PROGRAM_ID + " consumed")
);
assert(transaction.logMessages[4] === "Program " + PROGRAM_ID + " success");
assert(transaction.logMessages.length == 5);
```

So far, a simple test written through `NodeJS` is ready.

#### All in one test setup script

If you are lazy, you can directly run the following script to `setup.sh`, and run `bash setup.sh`.

```bash
# Create test directory
mkdir hello_world_frontend
cd hello_world_frontend

# Initialize pnpm project
pnpm init

# Install necessary dependencies
pnpm add -D typescript ts-node @types/node chai ts-mocha solana-bankrun
pnpm add @solana/web3.js solana-bankrun

# Create TypeScript configuration file
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

# Create source code directory and test file
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

# Update package.json to add test script
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

# Run test
pnpm test
EOF
```

# Frontend

There are two methods to develop solana frontend:

1. Using Anchor framework
2. Not using Anchor framework

I will help you implement both methods to develop Solana frontend. Let's start from the most basic one, and build step by step.

## 1. Not using Anchor framework

First, create a new Next.js project:

```bash
npx create-next-app@latest solana-frontend-nextjs --typescript --tailwind --eslint
cd solana-frontend-nextjs
```

Install necessary dependencies:

```bash
pnpm install \
  @solana/web3.js \
  @solana/wallet-adapter-react \
  @solana/wallet-adapter-react-ui \
  @solana/wallet-adapter-base \
  @solana/wallet-adapter-wallets
```

### 1.1 Basic setup

First, create a wallet configuration file:

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

Update the layout file:

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

### 1.2 Create main page component

Note that you need to replace the `PROGRAM_ID` in the `src/app/page.tsx` file with your program ID.

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

  // Replace with your program ID
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

### 1.3 Run project

Run:

```bash
pnpm dev
```

Click the `Say Hello` button to send a transaction through phantom wallet, and you can see the transaction details on the explorer:

https://explorer.solana.com/tx/4H3nfuDqaz1s6TDGe3HSL6DsEvq9r3TwcGqqw9kfGGk3c9pjK2HGohmCrfWcZCFXdMJsPobsbcj3UAdmkj2QK8vd?cluster=devnet

## 2. Using Anchor framework

Create a new project:

```bash
npx create-next-app@latest solana-anchor-frontend-nextjs --typescript --tailwind --eslint
cd solana-anchor-frontend-nextjs
```

Install dependencies:

```bash
pnpm install \
  @project-serum/anchor \
  @solana/web3.js \
  @solana/wallet-adapter-react \
  @solana/wallet-adapter-react-ui \
  @solana/wallet-adapter-base \
  @solana/wallet-adapter-wallets
```

### 2.1 Create Anchor IDL type

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

### 2.2 Create Anchor workspace provider

```typescript:src/context/WorkspaceProvider.tsx
"use client";

import { createContext, useContext, ReactNode } from "react"
import { Program, AnchorProvider } from "@project-serum/anchor"
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

### 2.3 Update layout component

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

### 2.4 Create main page component

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

### 2.5 tsconfig.json configuration

To correctly use `@` path aliases, you need to configure the `tsconfig.json` file:

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

This configuration file adds aliases for `@/idl/*` and `@/context/*` to use these paths in the code.

### 2.6 Run project

Run:

```bash
pnpm dev
```

Click the `Say Hello` button to send a transaction through phantom wallet, and you can see the transaction details on the explorer:

https://explorer.solana.com/tx/5dustfzfhSopVKrDiL3CoXAg35jimMBs3oFkxDsiBqM1xQ6t4JnsonbdZirzYdR5i5HGsUKmfhKZb3NQunWDbWiw?cluster=devnet

## Key Differences Between Two Methods

1. Without Anchor:

- Directly use `@solana/web3.js` to create transactions and instructions
- Manually construct transaction structures
- Lower-level control

2. With Anchor:

- Use Anchor IDL type definitions
- Higher-level abstractions and type safety
- More concise program invocation
- Better development experience

Which method to choose depends on your needs:

- If you need more low-level control or have a smaller project, you can choose not to use Anchor
- If you need better development experience and type safety, it's recommended to use Anchor

# Next Steps

At this point, we have completed the development and deployment of a basic Solana program. Although this program simply prints "Hello, world!", it contains the fundamental elements of Solana program development:

- Program entry point definition
- Basic parameter structure
- Build and deployment process

In the upcoming content, we will learn:

- How to develop programs using the Anchor framework
- How to handle account data
- How to implement more complex instruction logic
- How to test programs
- How to ensure program security

Stay tuned!

# Refs

Explanation about cargo-build-sbf
https://github.com/solana-labs/solana/issues/34987#issuecomment-1913538260

https://solana.stackexchange.com/questions/16443/error-function-stack-offset-of-7256-exceeded-max-offset-of-4096-by-3160-bytes

Installing Solana CLI tool suites (Note: Don't install edge version as it may cause deployment issues)
https://solana.com/docs/intro/installation

https://github.com/solana-labs/solana/issues/34987#issuecomment-1914665002
https://github.com/anza-xyz/agave/issues/1572

Writing a Hello World program on Solana
https://solana.com/developers/guides/getstarted/local-rust-hello-world#create-a-new-rust-library-with-cargo

solana wallet nextjs setup
https://solana.com/developers/guides/wallets/add-solana-wallet-adapter-to-nextjs

https://solana.com/developers/cookbook/wallets/connect-wallet-react
https://www.anza.xyz/blog/solana-web3-js-2-release

https://solana.stackexchange.com/questions/1723/anchor-useanchorwallet-vs-solanas-usewallet

anchor client side development
https://solana.com/developers/courses/onchain-development/intro-to-anchor-frontend
