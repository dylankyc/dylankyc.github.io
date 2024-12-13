# Create Solana Account Using TypeScript, Rust and Golang

<!-- toc -->

# Introduction

We'll walk through the process of creating Solana accounts using TypeScript, Golang, and Rust.

# Creating Solana Accounts with TypeScript

## Introduction

In this tutorial, we'll walk through the process of creating Solana accounts using TypeScript. We'll cover everything from setting up your project to creating and funding accounts on the Solana blockchain.

## Prerequisites

Before we begin, make sure you have the following prerequisites installed and set up on your system:

- Node.js (version 16 or later)
- npm or yarn
- Solana CLI installed
- Basic understanding of TypeScript and blockchain concepts

## Project Setup

### 1. Initialize a New TypeScript Project

```bash
# Create a new directory
mkdir solana-account-creation
cd solana-account-creation

# Initialize npm project
pnpm init -y

# Install TypeScript and Solana Web3.js
pnpm install typescript @solana/web3.js ts-node @types/node
```

We need to configure the `tsconfig.json` file with the following content to run the code with `ts-node`:

```bash
cat <<EOF > tsconfig.json
{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "types": ["node"],
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
EOF
```

### 2. Create a TypeScript File to Create Solana Accounts

```bash
touch create-account.ts
```

Add the following code to `create-account.ts`:

```typescript
import {
  SystemProgram,
  Keypair,
  PublicKey,
  Transaction,
  sendAndConfirmTransaction,
  Connection,
  clusterApiUrl,
  LAMPORTS_PER_SOL,
} from "@solana/web3.js";

import { readFileSync } from "fs";
// NOTE: types field in compilerOption in `tsconfig.json` should be node
import { homedir } from "os";

async function getBalance(publicKey: PublicKey) {
  const connection = new Connection(clusterApiUrl("devnet"), "confirmed");
  const balance = await connection.getBalance(publicKey);
  // console.log(`Payer's balance: ${balance / LAMPORTS_PER_SOL} SOL`);
  return balance;
}

// Output: /Users/dylankyc
// console.log(process.env.HOME);

const payerFilePath = `${homedir()}/.config/solana/id.json`;
const payerSecretKey = Uint8Array.from(
  JSON.parse(readFileSync(payerFilePath, "utf-8"))
);
console.log(`🌈 payerFilePath: ${payerFilePath}`);

const payer = Keypair.fromSecretKey(payerSecretKey);

const connection = new Connection(clusterApiUrl("devnet"), "confirmed");
// NOTE: Instead of generate keypair using Keypair::generate function,
// we'll use payer's public key
const fromPubkey = payer;
// const fromPubkey = Keypair.generate();

console.log("🌈 🌈 🌈 Create acount 🌈 🌈 🌈 ");
console.log(fromPubkey);

async function main() {
  const balanceBefore = await getBalance(fromPubkey.publicKey);
  // Airdrop SOL for transferring lamports to the created account
  //
  // NOTE: We have enough SOLs in local solana wallet, so skip airdrop
  //
  // const airdropSignature = await connection.requestAirdrop(
  //   fromPubkey.publicKey,
  //   LAMPORTS_PER_SOL
  // );
  // const tx = await connection.confirmTransaction(airdropSignature);
  // // output tranasction info
  // console.log('Transaction confirmed for airdrop:', tx);

  // amount of space to reserve for the account
  const space = 0;

  // Seed the created account with lamports for rent exemption
  const rentExemptionAmount =
    await connection.getMinimumBalanceForRentExemption(space);

  console.log(`🌈 rentExemptionAmount is : ${rentExemptionAmount}`);

  const newAccountPubkey = Keypair.generate();
  console.log("🌈 newAccountPubkey is generated");
  console.log(
    `🌈 new account address is : ${newAccountPubkey.publicKey.toBase58()}`
  );
  console.log(newAccountPubkey);

  const createAccountParams = {
    fromPubkey: fromPubkey.publicKey,
    newAccountPubkey: newAccountPubkey.publicKey,
    lamports: rentExemptionAmount,
    space,
    programId: SystemProgram.programId,
  };

  const createAccountTransaction = new Transaction().add(
    SystemProgram.createAccount(createAccountParams)
  );

  const createAccountTx = await sendAndConfirmTransaction(
    connection,
    createAccountTransaction,
    [fromPubkey, newAccountPubkey]
  );
  console.log("Transaction confirmed for account creation:", createAccountTx);

  const balanceAfter = await getBalance(fromPubkey.publicKey);

  console.log(`🌈 Balance before: ${balanceBefore}`);
  console.log(`🌈 Balance after : ${balanceAfter}`);

  // See transaction in devnet:
  //
  // https://explorer.solana.com/tx/5WqZGH4w3W65HvPhaqpKYQqn599By9wj1CWNM4XSp6VedB1C4dtNbv3zM3krd3nSLSLSsFLwGDD1UraxbCS1Vo4U?cluster=devnet
  //
  // https://explorer.solana.com/tx/3yio4oQfQ2WxNcYegQezpcriJZTguTffoPKjESeAb9W6NCjG4GBYAvqRdcnNz4AX3jqUofZyy9ugRhcqJVeWnmng?cluster=devnet
}

main();
```

Run the code:

```bash
npx ts-node create-account.ts
```

Output:

```bash
npx ts-node create-account.ts
🌈 payerFilePath: /Users/dylankyc/.config/solana/id.json
🌈 🌈 🌈 Create acount 🌈 🌈 🌈
Keypair {
  _keypair: {
    publicKey: Uint8Array(32) [
      // omit
    ],
    secretKey: Uint8Array(64) [
      // omit
    ]
  }
}
(node:87514) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
🌈 rentExemptionAmount is : 890880
🌈 newAccountPubkey is generated
🌈 new account address is : BN6yM69dUR98M3kxFeWQT8xRyeNE3cuREjSojJsHLZy7
Keypair {
  _keypair: {
    publicKey: Uint8Array(32) [
      // omit
    ],
    secretKey: Uint8Array(64) [
      // omit
    ]
  }
}
Transaction confirmed for account creation: 2yt5YKyYbQiW9WtEDDDG4QXfz6b2M1QhvrXpUG7mY9cWAKVQqeFoLxX25kWqjWFcR2Dj1QQWgu6FofUwrziqNNe3
🌈 Balance before: 90859560959
🌈 Balance after : 90858660079
```

# Creating Solana Accounts with Rust

## Project Setup

### 1. Initialize a New Rust Project using `cargo`

```bash
# Initialize a new Rust project
cargo new solana-rust-example
cd solana-rust-example
```

### 2. Add dependencies

```bash
# Add dependencies
cargo add anyhow bs58 dirs rand solana-rpc-client solana-sdk
```

Here is the `Cargo.toml` file:

```toml
name = "solana-rust-example"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1.0.93"
bs58 = "0.5.1"
dirs = "5.0.1"
rand = "0.8.5"
solana-rpc-client = "2.1.4"
solana-sdk = "2.1.4"
```

### 3. Add `src/main.rs`

```rust
use anyhow::{Context, Result};
use rand::prelude::*;
use solana_rpc_client::rpc_client::RpcClient;
use solana_sdk::{
    pubkey::Pubkey,
    signature::{read_keypair_file, Keypair, Signer},
    system_instruction, system_program,
    transaction::Transaction,
};
use std::path::PathBuf;

fn create_account(
    client: &RpcClient,
    payer: &Keypair,
    new_account: &Keypair,
    space: u64,
) -> Result<()> {
    // Get minimum balance for rent exemption
    let rent = client.get_minimum_balance_for_rent_exemption(space.try_into()?)?;

    // Create account instruction
    let instr = system_instruction::create_account(
        &payer.pubkey(),
        &new_account.pubkey(),
        rent,
        space,
        &system_program::ID,
    );

    // Get latest blockhash
    let blockhash = client.get_latest_blockhash()?;

    // Create transaction
    let tx = Transaction::new_signed_with_payer(
        &[instr],
        Some(&payer.pubkey()),
        &[payer, new_account],
        blockhash,
    );

    // Send transaction and confirm
    let sig = client.send_and_confirm_transaction(&tx)?;

    println!("Transaction confirmed with signature: {}", sig);

    Ok(())
}

fn main() -> Result<()> {
    // Replace with your Solana cluster URL (e.g., "https://api.mainnet-beta.solana.com")
    let rpc_url = "https://api.devnet.solana.com";
    let client = RpcClient::new(rpc_url);
    println!("🌈 client connected");

    // Load payer keypair from the default file location (~/.config/solana/id.json)
    let payer_path = dirs::home_dir()
        .expect("Could not find home directory")
        .join(".config/solana/id.json");
    let payer = read_keypair_file(payer_path)
        .expect("Failed to read keypair file. Ensure the file exists and is valid.");
    println!("✅ Payer keypair loaded: {}", payer.pubkey());

    // Generate a new account keypair
    let new_account = Keypair::new();

    // Specify space for the new account (e.g., 0 for a simple account)
    let space = 0;

    // Call the create_account function
    match create_account(&client, &payer, &new_account, space) {
        Ok(_) => println!("Account created: {}", new_account.pubkey()),
        Err(e) => eprintln!("Error creating account: {:?}", e),
    }

    // ✅ Create account OK
    // See transaction:
    // https://explorer.solana.com/tx/2Wc9vs5RiqF2VQVAfVREYFg2xpddJJdkiywvBQ3ywxtdP7XPcgwa3eTdJLwnUghEwYDRuDXWtzhR1hFEguc3xyzn?cluster=devnet

    Ok(())
}
```

### 4. Explanation

The code is straightforward and easy to understand.

Let's break down what this code does:

1. First, we set up the RPC client to connect to Solana's devnet:

```rust
let rpc_url = "https://api.devnet.solana.com";
let client = RpcClient::new(rpc_url);
println!("🌈 client connected");
```

2. Next, we load the payer's keypair from the default file location (~/.config/solana/id.json):

```rust
let payer_path = dirs::home_dir()
    .expect("Could not find home directory")
    .join(".config/solana/id.json");
let payer = read_keypair_file(payer_path)
    .expect("Failed to read keypair file. Ensure the file exists and is valid.");
println!("✅ Payer keypair loaded: {}", payer.pubkey());
```

3. We generate a new account keypair:

```rust
let new_account = Keypair::new();
```

4. We specify the space for the new account (e.g., 0 for a simple account):

```rust
let space = 0;
```

5. Finally, we call the `create_account` function to create the new account:

```rust
match create_account(&client, &payer, &new_account, space) {
    Ok(_) => println!("Account created: {}", new_account.pubkey()),
    Err(e) => eprintln!("Error creating account: {:?}", e),
}
```

This function sends the transaction to create the new account and prints the transaction signature.

### 5. Run the code:

```bash
cargo run
```

Output:

```bash
🌈 client connected
✅ Payer keypair loaded: FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH
Transaction confirmed with signature: 4KtfYuuMVYcXbMydzUAMVjpDNeWdQELDnXLokYwksJbfN2sEVuMHn4XLsB13nF8RpRVmN7BEHVwzmCh8uPDeYmWc
Account created: EiQLqEn3pgmrfVVakztzRAoe9FHQtnyR7kHyUXUjn8p2
```

# Creating Solana Accounts with Golang

## Project Setup

### 1. Initialize a New Golang Project

```bash
mkdir solana-golang-example
cd solana-golang-example
go mod init solana-golang-example
```

### 2. Add Dependencies

```bash
go get github.com/gagliardetto/solana-go
go get github.com/gagliardetto/solana-go/rpc
```

Output:

```bash
go: downloading github.com/gagliardetto/solana-go v1.12.0
go: downloading filippo.io/edwards25519 v1.0.0-rc.1
go: downloading github.com/gagliardetto/binary v0.8.0
go: downloading github.com/davecgh/go-spew v1.1.1
go: downloading github.com/gagliardetto/treeout v0.1.4
go: downloading github.com/mr-tron/base58 v1.2.0
go: downloading github.com/mostynb/zstdpool-freelist v0.0.0-20201229113212-927304c0c3b1
go: downloading github.com/streamingfast/logging v0.0.0-20230608130331-f22c91403091
go: downloading go.mongodb.org/mongo-driver v1.12.2
go: downloading go.uber.org/zap v1.21.0
go: downloading github.com/fatih/color v1.9.0
go: downloading github.com/klauspost/compress v1.13.6
go: downloading github.com/blendle/zapdriver v1.3.1
go: downloading github.com/logrusorgru/aurora v2.0.3+incompatible
go: downloading golang.org/x/crypto v0.0.0-20220622213112-05595931fe9d
go: downloading go.uber.org/atomic v1.7.0
go: downloading go.uber.org/multierr v1.6.0
go: downloading github.com/mattn/go-colorable v0.1.4
go: downloading github.com/mattn/go-isatty v0.0.11
go: downloading golang.org/x/term v0.0.0-20210927222741-03fcf44c2211
go: downloading golang.org/x/sys v0.0.0-20220722155257-8c9f86f7a55f
go: added filippo.io/edwards25519 v1.0.0-rc.1
go: added github.com/blendle/zapdriver v1.3.1
go: added github.com/davecgh/go-spew v1.1.1
go: added github.com/fatih/color v1.9.0
go: added github.com/gagliardetto/binary v0.8.0
go: added github.com/gagliardetto/solana-go v1.12.0
go: added github.com/gagliardetto/treeout v0.1.4
go: added github.com/json-iterator/go v1.1.12
go: added github.com/klauspost/compress v1.13.6
go: added github.com/logrusorgru/aurora v2.0.3+incompatible
go: added github.com/mattn/go-colorable v0.1.4
go: added github.com/mattn/go-isatty v0.0.11
go: added github.com/mitchellh/go-testing-interface v1.14.1
go: added github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd
go: added github.com/modern-go/reflect2 v1.0.2
go: added github.com/mostynb/zstdpool-freelist v0.0.0-20201229113212-927304c0c3b1
go: added github.com/mr-tron/base58 v1.2.0
go: added github.com/streamingfast/logging v0.0.0-20230608130331-f22c91403091
go: added go.mongodb.org/mongo-driver v1.12.2
go: added go.uber.org/atomic v1.7.0
go: added go.uber.org/multierr v1.6.0
go: added go.uber.org/zap v1.21.0
go: added golang.org/x/crypto v0.0.0-20220622213112-05595931fe9d
go: added golang.org/x/sys v0.0.0-20220722155257-8c9f86f7a55f
go: added golang.org/x/term v0.0.0-20210927222741-03fcf44c2211
```

### 3. Add `main.go`

Although the code is simple, it's a good example to understand how to create a new account on Solana.

```go
package main

import (
  "context"
  "fmt"

  "github.com/gagliardetto/solana-go"
  "github.com/gagliardetto/solana-go/rpc"
)

func main() {
  // Create a new account:
  account := solana.NewWallet()
  fmt.Println("account private key:", account.PrivateKey)
  fmt.Println("account public key:", account.PublicKey())

  // Create a new RPC client:
  // client := rpc.New(rpc.TestNet_RPC)
  // Devnet RPC
  client := rpc.New(rpc.DevNet_RPC)

  // Airdrop 1 SOL to the new account:
  out, err := client.RequestAirdrop(
    context.TODO(),
    account.PublicKey(),
    solana.LAMPORTS_PER_SOL*1,
    rpc.CommitmentFinalized,
  )
  if err != nil {
    panic(err)
  }
  fmt.Println("airdrop transaction signature:", out)
}
```

### 4. Add `create-account-with-payer.go`

In order to fully understand how to create a new account on Solana, we need to add a `create-account-with-payer.go` file. This file will create a new account with a payer.

```go
package main

import (
	"context"
	"fmt"
	"os"
	"path/filepath"

	"github.com/gagliardetto/solana-go"
	"github.com/gagliardetto/solana-go/programs/system"
	"github.com/gagliardetto/solana-go/rpc"
)

func main() {
	// Load payer keypair from ~/.config/solana/id.json
	homeDir, err := os.UserHomeDir()
	if err != nil {
		panic(fmt.Errorf("failed to get home directory: %w", err))
	}

	payer, err := solana.PrivateKeyFromSolanaKeygenFile(filepath.Join(homeDir, ".config", "solana", "id.json"))
	if err != nil {
		panic(fmt.Errorf("failed to load payer keypair: %w", err))
	}

	// Create a new account
	newAccount := solana.NewWallet()
	fmt.Println("New account private key:", newAccount.PrivateKey)
	fmt.Println("New account public key:", newAccount.PublicKey())
	fmt.Println("Payer public key:", payer.PublicKey())

	// Create RPC client
	client := rpc.New(rpc.DevNet_RPC)

	// Get recent blockhash
	recent, err := client.GetLatestBlockhash(context.Background(), rpc.CommitmentFinalized)
	if err != nil {
		panic(fmt.Errorf("failed to get recent blockhash: %w", err))
	}

	// Create transaction to create account
	tx, err := solana.NewTransaction(
		[]solana.Instruction{
	    // NOTE: The order of the instructions is important.
      //
			// Here is the signature of NewCreateAccountInstruction:
      //
			// NewCreateAccountInstruction declares a new CreateAccount instruction with the provided parameters and accounts.
			// func NewCreateAccountInstruction(
			// 	// Parameters:
			// 	lamports uint64,
			// 	space uint64,
			// 	owner ag_solanago.PublicKey,
			// 	// Accounts:
			// 	fundingAccount ag_solanago.PublicKey,
			// 	newAccount ag_solanago.PublicKey) *CreateAccount {
			// 	return NewCreateAccountInstructionBuilder().
			// 		SetLamports(lamports).
			// 		SetSpace(space).
			// 		SetOwner(owner).
			// 		SetFundingAccount(fundingAccount).
			// 		SetNewAccount(newAccount)
			// }
			system.NewCreateAccountInstruction(
				// solana.LAMPORTS_PER_SOL*1, // lamports
				solana.LAMPORTS_PER_SOL/5, // lamports (0.2 SOL = 1/5 SOL)
				0,                         // space
				solana.SystemProgramID,    // owner
				payer.PublicKey(),         // from
				newAccount.PublicKey(),    // new account (to)
			).Build(),
		},
		recent.Value.Blockhash,
		solana.TransactionPayer(payer.PublicKey()),
	)
	if err != nil {
		panic(fmt.Errorf("failed to create transaction: %w", err))
	}

	// Sign transaction
	_, err = tx.Sign(
		func(key solana.PublicKey) *solana.PrivateKey {
			if payer.PublicKey().Equals(key) {
				return &payer
			}
			if newAccount.PublicKey().Equals(key) {
				return &newAccount.PrivateKey
			}
			return nil
		},
	)
	if err != nil {
		panic(fmt.Errorf("unable to sign transaction: %w", err))
	}

	// Send transaction
	sig, err := client.SendTransaction(context.Background(), tx)
	if err != nil {
		panic(fmt.Errorf("failed to send transaction: %w", err))
	}

	fmt.Println("Transaction signature:", sig)
}
```

### 5. Explanation

The code is straightforward and easy to understand.

Let's break down what this code does:

1. First, we load the payer's keypair from the default file location (~/.config/solana/id.json):

```go
payer, err := solana.PrivateKeyFromSolanaKeygenFile(filepath.Join(homeDir, ".config", "solana", "id.json"))
```

2. Next, we create a new account:

```go
newAccount := solana.NewWallet()
```

3. We create a new RPC client:

```go
client := rpc.New(rpc.DevNet_RPC)
```

4. We get the recent blockhash:

```go
recent, err := client.GetLatestBlockhash(context.Background(), rpc.CommitmentFinalized)
```

5. We create a transaction to create the new account:

```go
tx, err := solana.NewTransaction(
```

6. We sign the transaction:

```go
_, err = tx.Sign(
  func(key solana.PublicKey) *solana.PrivateKey {
    if payer.PublicKey().Equals(key) {
      return &payer
    }
    if newAccount.PublicKey().Equals(key) {
      return &newAccount.PrivateKey
    }
    return nil
  },
)
```

The reason for this is that the payer needs to sign the transaction to approve spending SOL, and the new account needs to sign to prove ownership of the private key.

The `Sign` function is a bit complex, but it's necessary to ensure that the transaction is valid and can be executed on the Solana blockchain.

7. We send the transaction:

```go
sig, err := client.SendTransaction(context.Background(), tx)
```

Finally, we print the transaction signature:

```go
fmt.Println("Transaction signature:", sig)
```

### 6. Run the code:

```bash
go run create-account-with-payer.go
```

Output:

```bash
go run create-account-with-payer.go
New account private key: 3YoYuoiBYMh7Quuo8sZTTLUGPpzEcgB7iipgV4B7JL1FS7Y7SaTLDSdvwkc1ometgLqyYkufayG3QW8qxnCFWM1F
New account public key: 7XJwrhf7AVpUsx3qAinUKbruxH9SVVfrR3sjvUk74vx5
Payer public key: FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH
Transaction signature: 5nbe1TfU9Pi7k6oaELuCfPtqGQAASn7Q9rP46r3vugC5jgjnxXCf9o5epFMWxn9kq4SnQiS2Tqdgd9UCrwSMkhbU
```
