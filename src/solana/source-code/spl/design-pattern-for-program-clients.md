# Design Pattern for Program Clients

<!-- toc -->

# Understanding Solana's Client Architecture: A Deep Dive into Rust Type System Usage

## Introduction

Solana's client architecture provides an excellent example of leveraging Rust's type system to create a flexible, type-safe, and maintainable design.

In this post, we'll explore how Solana implements different client types for various use cases while maintaining a unified interface.

## The Three Client Types

The solana-program-library crate implements three main client types:

- `ProgramRpcClient`: For production RPC interactions
- `ProgramBanksClient`: For testing and simulation
- `ProgramOfflineClient`: For offline transaction signing

Let's take a look at each of these client types.

### 1. ProgramBanksClient

`ProgramBanksClient` has two fields:

- `context`: A `ProgramBanksClientContext` enum that can be either `Client` or `Context`
- `send`: This is of generic type `ST`, which implements the `SendTransactionBanksClient` and `SimulateTransactionBanksClient` traits to send transactions(see `send` method) and simulate transactions(see `simulate` method).

```rust
pub struct ProgramBanksClient<ST> {
    context: ProgramBanksClientContext,
    send: ST,
}

enum ProgramBanksClientContext {
    Client(Arc<Mutex<BanksClient>>),
    Context(Arc<Mutex<ProgramTestContext>>),
}

#[async_trait]
impl<ST> ProgramClient<ST> for ProgramBanksClient<ST>
where
    ST: SendTransactionBanksClient + SimulateTransactionBanksClient + Send + Sync,
{}

/// Extends basic `SendTransaction` trait with function `send` where client is
/// `&mut BanksClient`. Required for `ProgramBanksClient`.
pub trait SendTransactionBanksClient: SendTransaction {
    fn send<'a>(
        &self,
        client: &'a mut BanksClient,
        transaction: Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::Output>>;
}

/// Extends basic `SimulateTransaction` trait with function `simulation` where
/// client is `&mut BanksClient`. Required for `ProgramBanksClient`.
pub trait SimulateTransactionBanksClient: SimulateTransaction {
    fn simulate<'a>(
        &self,
        client: &'a mut BanksClient,
        transaction: Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::SimulationOutput>>;
}

/// Basic trait for sending transactions to validator.
pub trait SendTransaction {
    type Output;
}
```

`ProgramBanksClient` is primarily used for:

- Testing Solana Programs: It enables developers to simulate and send transactions to a Solana program while running tests, allowing for verification of program behavior without the need for an actual on-chain deployment.
- Encapsulation of Client Logic: The client encapsulates the logic required to send and simulate transactions, making it easier to manage and modify as testing needs evolve.

There are two variants of `ProgramBanksClientContext`:

- **`Client(Arc<Mutex<BanksClient>>)`**: Provides streamlined access for basic transaction processing and simple unit tests. It's lightweight and ideal for single transaction validation and basic program interaction testing.
- **`Context(Arc<Mutex<ProgramTestContext>>)`**: Offers a comprehensive testing environment with advanced features like custom genesis configuration, block management, fee payer accounts, and program deployment capabilities. This makes it suitable for integration testing, multi-transaction scenarios, and complex state management testing where full lifecycle control is needed.

According to the documentation, `BanksClient` is a client for the ledger state, from the perspective of an arbitrary validator. It serves as a client interface for interacting with the Solana blockchain's in-memory state during testing.

### 2. ProgramRpcClient

The `ProgramRpcClient<ST>` struct is designed to serve as a client for interacting with the Solana blockchain through the `RpcClient`.

```rust
pub struct ProgramRpcClient<ST> {
    client: Arc<RpcClient>,
    send: ST,
}

#[async_trait]
impl<ST> ProgramClient<ST> for ProgramRpcClient<ST>
where
    ST: SendTransactionRpc + SimulateTransactionRpc + Send + Sync,
{}

/// Extends basic `SendTransaction` trait with function `send` where client is
/// `&RpcClient`. Required for `ProgramRpcClient`.
pub trait SendTransactionRpc: SendTransaction {
    fn send<'a>(
        &self,
        client: &'a RpcClient,
        transaction: &'a Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::Output>>;
}

/// Extends basic `SimulateTransaction` trait with function `simulate` where
/// client is `&RpcClient`. Required for `ProgramRpcClient`.
pub trait SimulateTransactionRpc: SimulateTransaction {
    fn simulate<'a>(
        &self,
        client: &'a RpcClient,
        transaction: &'a Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::SimulationOutput>>;
}
```

- Used for regular RPC interactions with Solana nodes
- Wraps the standard `RpcClient` from solana-client
- Communicates with Solana nodes over JSON-RPC protocol
- Suitable for standard production environments

### 3. ProgramOfflineClient

```rust
pub struct ProgramOfflineClient<ST> {
    blockhash: Hash,
    _send: ST,
}

#[async_trait]
impl<ST> ProgramClient<ST> for ProgramOfflineClient<ST>
where
    ST: SendTransaction<Output = RpcClientResponse>
        + SimulateTransaction<SimulationOutput = RpcClientResponse>
        + Send
        + Sync,
{}
```

- Designed for offline transaction signing
- Doesn't require network connection
- Limited functionality (can't fetch accounts or rent)
- Useful for cold wallet scenarios

## Type System Design

### 1. Unified Interface Through Traits

The `ProgramClient` trait serves as the foundational interface that all client implementations must adhere to, defining the core functionality for interacting with Solana programs:

```rust
#[async_trait]
pub trait ProgramClient<ST>
where
    ST: SendTransaction + SimulateTransaction,
{
    async fn get_minimum_balance_for_rent_exemption(&self, data_len: usize) -> ProgramClientResult<u64>;
    async fn get_latest_blockhash(&self) -> ProgramClientResult<Hash>;
    async fn send_transaction(&self, transaction: &Transaction) -> ProgramClientResult<ST::Output>;
    async fn get_account(&self, address: Pubkey) -> ProgramClientResult<Option<Account>>;
    async fn simulate_transaction(&self, transaction: &Transaction) -> ProgramClientResult<ST::SimulationOutput>;
}
```

It contains 5 methods:

- `get_minimum_balance_for_rent_exemption`: Get the minimum balance required for rent exemption.
- `get_latest_blockhash`: Get the latest blockhash.
- `send_transaction`: Send a transaction.
- `get_account`: Get an account.
- `simulate_transaction`: Simulate a transaction.

### 2. Transaction Handling Traits

In Solana's client architecture, there are base traits (`SendTransaction` and `SimulateTransaction`) that define what operations like sending and simulating transactions are possible.

```rust
/// Basic trait for sending transactions to validator.
pub trait SendTransaction {
    type Output;
}

/// Basic trait for simulating transactions in a validator.
pub trait SimulateTransaction {
    type SimulationOutput: SimulationResult;
}

/// Trait for the output of a simulation
pub trait SimulationResult {
    fn get_compute_units_consumed(&self) -> ProgramClientResult<u64>;
}
```

However, there are different client types, i.e. `Banks`(`ProgramBanksClient`), `RPC`(`ProgramRpcClient`), `Offline`(`ProgramOfflineClient`). How do they implement these operations differently?

We can define different traits to extend the basic `SendTransaction` and `SimulateTransaction` traits and add trait bounds to the generic parameter `ST` of `ProgramClient` to ensure that the concrete type `ST` implements these operations differently.

Here are the extended traits:

- `SendTransactionBanksClient`: Extends `SendTransaction` with a `send` method that takes a `&mut BanksClient` and a `Transaction`.
- `SimulateTransactionBanksClient`: Extends `SimulateTransaction` with a `simulate` method that takes a `&mut BanksClient` and a `Transaction`.
- `SendTransactionRpc`: Extends `SendTransaction` with a `send` method that takes a `&RpcClient` and a `&Transaction`.
- `SimulateTransactionRpc`: Extends `SimulateTransaction` with a `simulate` method that takes a `&RpcClient` and a `&Transaction`.

```rust
/// Extends basic `SendTransaction` trait with function `send` where client is
/// `&RpcClient`. Required for `ProgramRpcClient`.
pub trait SendTransactionRpc: SendTransaction {
    fn send<'a>(
        &self,
        client: &'a RpcClient,
        transaction: &'a Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::Output>>;
}

// Extends basic `SendTransaction` trait with function `send` where client is
/// `&mut BanksClient`. Required for `ProgramBanksClient`.
pub trait SendTransactionBanksClient: SendTransaction {
    fn send<'a>(
        &self,
        client: &'a mut BanksClient,
        transaction: Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::Output>>;
}

/// Extends basic `SimulateTransaction` trait with function `simulate` where
/// client is `&RpcClient`. Required for `ProgramRpcClient`.
pub trait SimulateTransactionRpc: SimulateTransaction {
    fn simulate<'a>(
        &self,
        client: &'a RpcClient,
        transaction: &'a Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::SimulationOutput>>;
}

/// Extends basic `SimulateTransaction` trait with function `simulation` where
/// client is `&mut BanksClient`. Required for `ProgramBanksClient`.
pub trait SimulateTransactionBanksClient: SimulateTransaction {
    fn simulate<'a>(
        &self,
        client: &'a mut BanksClient,
        transaction: Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::SimulationOutput>>;
}
```

So when implementing `ProgramClient` for `ProgramRpcClient`, we add trait bounds to the generic parameter `ST` to ensure that the concrete type `ST` implements the `SendTransactionRpc` and `SimulateTransactionRpc` traits.

```rust
impl<ST> ProgramClient<ST> for ProgramRpcClient<ST>
where
    ST: SendTransactionRpc + SimulateTransactionRpc + Send + Sync,
{}
```

Similarly, when implementing `ProgramClient` for `ProgramBanksClient`, we add trait bounds to the generic parameter `ST` to ensure that the concrete type `ST` implements the `SendTransactionBanksClient` and `SimulateTransactionBanksClient` traits.

```rust
impl<ST> ProgramClient<ST> for ProgramBanksClient<ST>
where
    ST: SendTransactionBanksClient + SimulateTransactionBanksClient + Send + Sync,
{}
```

And when implementing `ProgramClient` for `ProgramOfflineClient`, we add trait bounds to the generic parameter `ST` to ensure that the concrete type `ST` implements the `SendTransaction` and `SimulateTransaction` traits.

```rust
impl<ST> ProgramClient<ST> for ProgramOfflineClient<ST>
where
    ST: SendTransaction<Output = RpcClientResponse>
        + SimulateTransaction<SimulationOutput = RpcClientResponse>
        + Send
        + Sync,
{}
```

So how to initialize these client types?

For `ProgramRpcClient`, we can initialize it like this:

```rust
let rpc_client = Arc::new(RpcClient::new("http://api.mainnet-beta.solana.com"));
let program_client = ProgramRpcClient::new(
    rpc_client,
    ProgramRpcClientSendTransaction
);
```

`ProgramRpcClientSendTransaction` is a concrete type that implements the `SendTransactionRpc` trait.

```rust
#[derive(Debug, Clone, Copy, Default)]
pub struct ProgramRpcClientSendTransaction;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum RpcClientResponse {
    Signature(Signature),
    Transaction(Transaction),
    Simulation(RpcSimulateTransactionResult),
}

impl SendTransaction for ProgramRpcClientSendTransaction {
    type Output = RpcClientResponse;
}

impl SendTransactionRpc for ProgramRpcClientSendTransaction {
    fn send<'a>(
        &self,
        client: &'a RpcClient,
        transaction: &'a Transaction,
    ) -> BoxFuture<'a, ProgramClientResult<Self::Output>> {
        Box::pin(async move {
            if !transaction.is_signed() {
                return Err("Cannot send transaction: not fully signed".into());
            }

            client
                .send_and_confirm_transaction(transaction)
                .await
                .map(RpcClientResponse::Signature)
                .map_err(Into::into)
        })
    }
}
```

We can also define another concrete type like `ProgramRpcClientSendTransactionCustom` that implements both `SendTransaction` and `SimulateTransaction` traits, and use it to initialize `ProgramRpcClient`. This helps us to customize the send behavior of `ProgramRpcClient`.

In other words, when you instantiate `ProgramRpcClient`( `ProgramBanksClient` or `ProgramOfflineClient`), you provide a concrete type for `ST` that implements the required traits. The `send` field will then be used in methods like `send_transaction` and `simulate_transaction` to invoke the appropriate logic for handling those operations.

```rust
let program_client = ProgramRpcClient::new(
    rpc_client,
    ProgramRpcClientSendTransactionCustom
);
```

Implementing traits for different concrete types and using them to initialize `ProgramRpcClient` is a good example of the strategy pattern.

## Design Patterns

Now, let's take a look at the design patterns used in the `ProgramClient` trait.

### 1. Strategy Pattern

The generic parameter `ST` implements the strategy pattern:

```rust
pub struct ProgramRpcClient<ST> {
    client: Arc<RpcClient>,
    send: ST,  // Strategy for sending transactions
}
```

### 2. Adapter Pattern

Each client adapts a specific underlying implementation:

```rust
#[async_trait]
impl<ST> ProgramClient<ST> for ProgramRpcClient<ST>
where
    ST: SendTransactionRpc + SimulateTransactionRpc + Send + Sync,
{
    async fn send_transaction(&self, transaction: &Transaction) -> ProgramClientResult<ST::Output> {
        self.send.send(&self.client, transaction).await
    }
}

#[async_trait]
impl<ST> ProgramClient<ST> for ProgramBanksClient<ST>
where
    ST: SendTransactionBanksClient + SimulateTransactionBanksClient + Send + Sync,
{
    async fn send_transaction(&self, transaction: &Transaction) -> ProgramClientResult<ST::Output> {
        self.run_in_lock(|client| {
            let transaction = transaction.clone();
            self.send.send(client, transaction)
        })
        .await
    }
}

#[async_trait]
impl<ST> ProgramClient<ST> for ProgramOfflineClient<ST>
where
    ST: SendTransaction<Output = RpcClientResponse>
        + SimulateTransaction<SimulationOutput = RpcClientResponse>
        + Send
        + Sync,
{
    async fn send_transaction(&self, transaction: &Transaction) -> ProgramClientResult<ST::Output> {
        Ok(RpcClientResponse::Transaction(transaction.clone()))
    }
}
```

### 3. Composition Pattern

The design uses composition over inheritance:

```rust
/// Program client for `BanksClient` from crate `solana-program-test`.
pub struct ProgramBanksClient<ST> {
    context: ProgramBanksClientContext,
    send: ST,
}

/// Program client for `RpcClient` from crate `solana-client`.
pub struct ProgramRpcClient<ST> {
    client: Arc<RpcClient>,
    send: ST,
}

/// Program client for offline signing.
pub struct ProgramOfflineClient<ST> {
    blockhash: Hash,
    _send: ST,
}
```

The `ProgramBanksClient` struct composes of a `context` and a `send` field. The `context` field is an enum that can be either `Client` or `Context`. The `send` field is a generic type `ST` that implements the `SendTransactionBanksClient` and `SimulateTransactionBanksClient` traits.

The `ProgramRpcClient` struct composes of a `client` and a `send` field. The `client` field is an `Arc<RpcClient>`. The `send` field is a generic type `ST` that implements the `SendTransactionRpc` and `SimulateTransactionRpc` traits.

The `ProgramOfflineClient` struct composes of a `blockhash` and a `_send` field. The `blockhash` field is a `Hash`. The `_send` field is a generic type `ST` that implements the `SendTransaction` and `SimulateTransaction` traits.

## Usage Examples

### Banks Client for High Performance

```rust
let banks_client = Arc::new(Mutex::new(BanksClient::new(...)));
let program_client = ProgramBanksClient::new_from_client(
    banks_client,
    ProgramBanksClientProcessTransaction
);
```

### RPC Client for Standard Usage

```rust
let rpc_client = Arc::new(RpcClient::new("http://api.mainnet-beta.solana.com"));
let program_client = ProgramRpcClient::new(
    rpc_client,
    ProgramRpcClientSendTransaction
);
```

### Offline Client for Cold Wallets

```rust
let program_client = ProgramOfflineClient::new(
    blockhash,
    ProgramOfflineTransaction
);
```

## Benefits of This Design

1. **Type Safety**

   - Compile-time guarantees
   - No runtime type errors
   - Clear interface contracts

2. **Flexibility**

   - Easy to add new client types
   - Customizable transaction handling
   - Pluggable components

3. **Performance**

   - Zero-cost abstractions
   - Direct banking stage access when needed
   - RPC interface when appropriate

4. **Maintainability**
   - Clear separation of concerns
   - Modular design
   - Easy to test

## Conclusion

Solana's client architecture demonstrates sophisticated use of Rust's type system to:

- Support different performance requirements
- Ensure type safety and correct usage
- Allow flexibility in transaction handling
- Maintain a consistent interface across implementations

The design shows how to leverage Rust's type system to create a robust and flexible architecture that can handle different use cases while maintaining type safety and a clean interface.
