# Change Token Mint Owner

<!-- toc -->

# Introduction

In this guide, we will learn how to change the mint authority for a token mint account using the `spl-token authorize` command.

Use `spl-token authorize` to change the mint authority for a token mint account.

```bash
# display the current mint authority for mint token
(base) dylankyc@smoltown ~/Documents/solana> spl-token display J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm

SPL Token Mint
  Address: J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm
  Program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
  Supply: 100000000000
  Decimals: 9
  Mint authority: FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH
  Freeze authority: (not set)

# Current wallet address
(base) dylankyc@smoltown ~/Documents/solana> solana address
FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH

# Show another wallet address
(base) dylankyc@smoltown ~/Documents/solana [1]> solana address -k account-2.json
DyfxCDkNAWWmPxqmeZnA6K3bwanrPLZAofD64HXdfvYf

# Change the mint authority
(base) dylankyc@smoltown ~/Documents/solana> spl-token authorize J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm mint DyfxCDkNAWWmPxqmeZnA6K3bwanrPLZAofD64HXdfvYf
Updating J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm
  Current mint: FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH
  New mint: DyfxCDkNAWWmPxqmeZnA6K3bwanrPLZAofD64HXdfvYf

Signature: 4KHkfxTn6f6PX45iDRaEyew2EGDYbTZYyy8WcZDtD3fABFig8nVeTZGNCMo1PMtchJ9AgHaincnwVbqz4gNmxGn1

# Display the mint authority after change
# Notice the mint authority has changed to DyfxCDkNAWWmPxqmeZnA6K3bwanrPLZAofD64HXdfvYf
(base) dylankyc@smoltown ~/Documents/solana> spl-token display J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm

SPL Token Mint
  Address: J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm
  Program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
  Supply: 100000000000
  Decimals: 9
  Mint authority: DyfxCDkNAWWmPxqmeZnA6K3bwanrPLZAofD64HXdfvYf
  Freeze authority: (not set)
```

# spl-token authorize help

For a full list of options, use `spl-token authorize -h`.

```bash
(base) dylankyc@smoltown ~/Documents/solana [1]> spl-token authorize J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm -h
spl-token-authorize
Authorize a new signing keypair to a token or token account

USAGE:
    spl-token authorize [FLAGS] [OPTIONS] <TOKEN_ADDRESS> <AUTHORITY_TYPE> <AUTHORITY_ADDRESS>

FLAGS:
        --disable                     Disable mint, freeze, or close functionality by setting authority to None.
        --dump-transaction-message    Display the base64 encoded binary transaction message in sign-only mode
    -h, --help                        Prints help information
        --sign-only                   Sign the transaction offline
    -V, --version                     Prints version information
    -v, --verbose                     Show additional information

OPTIONS:
        --authority <KEYPAIR>
            Specify the current authority keypair. Defaults to the client keypair.

        --blockhash <BLOCKHASH>                           Use the supplied blockhash
        --with-compute-unit-limit <COMPUTE-UNIT-LIMIT>    Set compute unit limit for transaction, in compute units.
        --with-compute-unit-price <COMPUTE-UNIT-PRICE>
            Set compute unit price for transaction, in increments of 0.000001 lamports per compute unit.

    -C, --config <PATH>                                   Configuration file to use
        --fee-payer <KEYPAIR>
            Specify the fee-payer account. This may be a keypair file, the ASK keyword
            or the pubkey of an offline signer, provided an appropriate --signer argument
            is also passed. Defaults to the client keypair.
    -u, --url <URL_OR_MONIKER>
            URL for Solana's JSON RPC or moniker (or their first letter): [mainnet-beta, testnet, devnet, localhost]
            Default from the configuration file.
        --multisig-signer <MULTISIG_SIGNER>...            Member signer of a multisig account
        --nonce <PUBKEY>
            Provide the nonce account to use when creating a nonced
            transaction. Nonced transactions are useful when a transaction
            requires a lengthy signing process. Learn more about nonced
            transactions at https://docs.solanalabs.com/cli/examples/durable-nonce
        --nonce-authority <KEYPAIR>
            Provide the nonce authority keypair to use when signing a nonced transaction

        --output <FORMAT>
            Return information in specified output format [possible values: json, json-compact]

    -p, --program-id <ADDRESS>                            SPL Token program id
        --signer <PUBKEY=SIGNATURE>...                    Provide a public-key/signature pair for the transaction

ARGS:
    <TOKEN_ADDRESS>        The address of the token mint or account
    <AUTHORITY_TYPE>       The new authority type. Token mints support `mint`, `freeze`, and mint extension
                           authorities; Token accounts support `owner`, `close`, and account extension authorities.
                           [possible values: mint, freeze, owner, close, close-mint, transfer-fee-config, withheld-
                           withdraw, interest-rate, permanent-delegate, confidential-transfer-mint,
                           transfer-hook-program-id, confidential-transfer-fee, metadata-pointer, metadata, group-
                           pointer, group-member-pointer, group]
    <AUTHORITY_ADDRESS>    The address of the new authority
(base) dylankyc@smoltown ~/Documents/solana> spl-token authorize J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm owner
error: The following required arguments were not provided:
    <AUTHORITY_ADDRESS>

USAGE:
    spl-token authorize [FLAGS] [OPTIONS] <TOKEN_ADDRESS> <AUTHORITY_TYPE> <AUTHORITY_ADDRESS>

For more information try --help
```
