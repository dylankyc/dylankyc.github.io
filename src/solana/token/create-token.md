# Create solana token

<!-- toc -->

## Create token(Token Mint)

Use `spl-token create-token` to create a new token. See https://solana.com/docs/core/tokens

```
Creating token 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY under program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA

Address:  23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
Decimals:  9

Signature: 3oGeH3iY45PwcBet5A6x2oGUkpN3EK3cGXKsrTg9ui587ucH2BX6uLQhG1z4nCHUn5RiMKGc4maG7VEA2phPNStL
```

Remember the `Address`: `23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY`, which is the address of the new token.

Below is the help message of `spl-token create-token`.

```bash
spl-token create-token --help
spl-token-create-token
Create a new token

USAGE:
    spl-token create-token [FLAGS] [OPTIONS] [TOKEN_KEYPAIR]

FLAGS:
        --enable-close                 Enable the mint authority to close this mint
        --enable-freeze                Enable the mint authority to freeze token accounts for this mint
        --enable-group                 Enables group configurations in the mint. The mint authority must initialize the
                                       group.
        --enable-member                Enables group member configurations in the mint. The mint authority must
                                       initialize the member.
        --enable-metadata              Enables metadata in the mint. The mint authority must initialize the metadata.
        --enable-non-transferable      Permanently force tokens to be non-transferable. They may still be burned.
        --enable-permanent-delegate    Enable the mint authority to be permanent delegate for this mint
    -h, --help                         Prints help information
    -V, --version                      Prints version information
    -v, --verbose                      Show additional information

OPTIONS:
        --with-compute-unit-limit <COMPUTE-UNIT-LIMIT>        Set compute unit limit for transaction, in compute units.
        --with-compute-unit-price <COMPUTE-UNIT-PRICE>
            Set compute unit price for transaction, in increments of 0.000001 lamports per compute unit.

    -C, --config <PATH>                                       Configuration file to use
        --decimals <DECIMALS>
            Number of base 10 digits to the right of the decimal place [default: 9]

        --default-account-state <default_account_state>
            Specify that accounts have a default state. Note: specifying "initialized" adds an extension, which gives
            the option of specifying default frozen accounts in the future. This behavior
```

### Create token with custom keypair

First, let's create a custom keypair for the token.

```bash
solana-keygen new -o mint-token-keypair.json
```

Output:

```
Generating a new keypair

For added security, enter a BIP39 passphrase

NOTE! This passphrase improves security of the recovery seed phrase NOT the
keypair file itself, which is stored as insecure plain text

BIP39 Passphrase (empty for none):

Wrote new keypair to mint-token-keypair.json
===========================================================================
pubkey: 7QtePN3WrDHK3q4bvwb3Qf6JhCSWbh9orRz397TFTV3z
===========================================================================
Save this seed phrase and your BIP39 passphrase to recover your new keypair:
key pool shallow divide limit derive explain boring brief merge include fox
===========================================================================
```

The content of the keypair is:

```bash
> cat mint-token-keypair.json
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[92,210,46,10,118,80,241,238,26,161,95,110,122,34,166,248,130,235,232,105,62,135,225,239,163,245,199,37,76,92,227,133,95,68,223,248,3,243,16,22,78,44,214,186,120,128,176,204,180,172,17,84,137,168,2
19,33,214,163,251,37,71,236,79,77]
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Next, let's create the token with the custom keypair.

```bash
> spl-token create-token mint-token-keypair.json
Creating token 7QtePN3WrDHK3q4bvwb3Qf6JhCSWbh9orRz397TFTV3z under program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA

Address:  7QtePN3WrDHK3q4bvwb3Qf6JhCSWbh9orRz397TFTV3z
Decimals:  9

Signature: 45sfrobQ8koMYTdABqeZTLfdU6WS3Qc4Kpi8dkP1oNrYr8BfwvUs6ZHa5DMxZ6eB27bBgSN3CnTc2GTjJZgZ93Gt
```

Notice, the address of the token is `7QtePN3WrDHK3q4bvwb3Qf6JhCSWbh9orRz397TFTV3z`, which is the address of the custom keypair.

Check the transaction on the explorer:

https://explorer.solana.com/tx/45sfrobQ8koMYTdABqeZTLfdU6WS3Qc4Kpi8dkP1oNrYr8BfwvUs6ZHa5DMxZ6eB27bBgSN3CnTc2GTjJZgZ93Gt?cluster=devnet

From the Program Instruction Logs we can see that Token Program(`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`) is invoked and `InitializeMint` instruction is called to create the mint token.

```json
[
  "Program 11111111111111111111111111111111 invoke [1]",
  "Program 11111111111111111111111111111111 success",
  "Program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA invoke [1]",
  "Program log: Instruction: InitializeMint",
  "Program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA consumed 2919 of 3069 compute units",
  "Program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA success",
  "Program ComputeBudget111111111111111111111111111111 invoke [1]",
  "Program ComputeBudget111111111111111111111111111111 success"
]
```

## Check the supply

New tokens initially have no supply. Let's check the supply.

```bash
spl-token supply 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
```

The output is:

```
0
```

## Decode Account - MintAccount

### Decode Token Account using `spl-token display`

You can use `spl-token display` to decode the mint account. It will query the details of an SPL token mint account.

Here is the help message of `spl-token display`.

```bash
spl-token display -h
spl-token-display
Query details of an SPL Token mint, account, or multisig by address

USAGE:
    spl-token display [FLAGS] [OPTIONS] <TOKEN_ADDRESS>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information
    -v, --verbose    Show additional information

OPTIONS:
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
        --output <FORMAT>
            Return information in specified output format [possible values: json, json-compact]

    -p, --program-id <ADDRESS>                            SPL Token program id

ARGS:
    <TOKEN_ADDRESS>    The address of the SPL Token mint, account, or multisig to query
```

Let's decode the mint account `23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY`.

```bash
spl-token display 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY

SPL Token Mint
  Address: 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
  Program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
  Supply: 0
  Decimals: 9
  Mint authority: 7HjK9uvhowd7JZyq2fH5LAhCzELTuBq5oWHEBtB9SMwn
  Freeze authority: (not set)
```

From the output, we can see that the mint account is initialized with the following fields:

- `Address`: `23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY`
- `Program`: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`
- `Supply`: `0`
- `Decimals`: `9`
- `Mint authority`: `7HjK9uvhowd7JZyq2fH5LAhCzELTuBq5oWHEBtB9SMwn`
- `Freeze authority`: `(not set)`

### Decode Token Account using `MintLayout.decode`

We can use `MintLayout.decode` to decode the mint account.

````typescript
import { Connection, PublicKey } from "@solana/web3.js";
import bs58 from "bs58";
import { AccountLayout, MintLayout, TOKEN_PROGRAM_ID } from "@solana/spl-token";
import yargs from "yargs";

const NETWORK_URLS = {
  devnet: "https://api.devnet.solana.com",
  mainnet: "https://api.mainnet-beta.solana.com",
  localnet: "http://127.0.0.1:8899",
} as const;

type NetworkType = keyof typeof NETWORK_URLS;

const argv = yargs
  .option("account", {
    alias: "a",
    description: "Account address to query",
    type: "string",
    demandOption: true,
  })
  .option("network", {
    alias: "n",
    description: "Solana network to use",
    choices: ["devnet", "mainnet", "localnet"] as const,
    type: "string",
    default: "devnet",
  })
  .help()
  .parseSync();

async function decodeAccount(accountAddress: string, network: NetworkType) {
  try {
    console.log(`Network: ${network}`);
    console.log(`Account Address: ${accountAddress}`);
    const connection = new Connection(NETWORK_URLS[network]);
    const publicKey = new PublicKey(accountAddress);
    const accountInfo = await connection.getAccountInfo(publicKey);

    if (!accountInfo) {
      console.log("❌ Account not found");
      return;
    }

    console.log("\nAccount Basic Info:");
    console.log("------------------");
    console.log(`Owner: ${accountInfo.owner.toString()}`);
    console.log(`Data length: ${accountInfo.data.length} bytes`);
    console.log(`Executable: ${accountInfo.executable}`);
    console.log(`Lamports: ${accountInfo.lamports / 1e9} SOL`);

    // Check if it's an executable program
    if (accountInfo.executable) {
      console.log("\n✅ This is an Executable Program Account");
      // Decode first 8 bytes of program data (often contains a discriminator)
      const programData = accountInfo.data.slice(0, 8);
      console.log(`Program Discriminator: ${bs58.encode(programData)}`);
      return;
    }

    // Check if it's a Token Program account
    if (accountInfo.owner.equals(TOKEN_PROGRAM_ID)) {
      // Token Mint Account (82 bytes)
      if (accountInfo.data.length === MintLayout.span) {
        console.log("\n✅ This is a Token Mint Account");
        const mintInfo = MintLayout.decode(accountInfo.data);
        console.log("\nDecoded Mint Info:", {
          mintAuthority: mintInfo.mintAuthority?.toString(),
          supply: mintInfo.supply.toString(),
          decimals: mintInfo.decimals,
          isInitialized: mintInfo.isInitialized,
          freezeAuthority: mintInfo.freezeAuthority?.toString(),
        });
        return;
      }

      // Token Account (165 bytes)
      if (accountInfo.data.length === AccountLayout.span) {
        console.log("\n✅ This is a Token Account");
        const tokenInfo = AccountLayout.decode(accountInfo.data);
        console.log("\nDecoded Token Info:", {
          mint: tokenInfo.mint.toString(),
          owner: tokenInfo.owner.toString(),
          amount: tokenInfo.amount.toString(),
          delegateOption: tokenInfo.delegateOption,
          delegate: tokenInfo.delegate.toString(),
          state: tokenInfo.state,
          isNativeOption: tokenInfo.isNativeOption,
          isNative: tokenInfo.isNative,
          delegatedAmount: tokenInfo.delegatedAmount.toString(),
          closeAuthorityOption: tokenInfo.closeAuthorityOption,
          closeAuthority: tokenInfo.closeAuthority.toString(),
        });
        return;
      }
    }

    // Regular account with data
    if (accountInfo.data.length > 0) {
      console.log(
        "\n✅ This is a Program-Owned Account (PDA) or Custom Account"
      );
      console.log("\nRaw Data:");
      console.log("Base58:", bs58.encode(accountInfo.data));
      console.log("Hex:", Buffer.from(accountInfo.data).toString("hex"));

      // Try to decode as UTF-8 in case it contains readable text
      try {
        const textDecoder = new TextDecoder();
        const text = textDecoder.decode(accountInfo.data);
        if (text.match(/^[\x00-\x7F]*$/)) {
          // Check if ASCII
          console.log("UTF-8:", text);
        }
      } catch (e) {
        // Ignore UTF-8 decode errors
      }
      return;
    }

    // System account (just lamports, no data)
    console.log("\n✅ This is a System Account (only holds SOL)");
  } catch (error) {
    console.error("Error decoding account:", error;

## Create token account(Token Account)

Let's create a token account to hold units of the token specified in the create-account command.

From `spl-token create-account --help` command, to create a token account, we need to provide the token mint address `<TOKEN_MINT_ADDRESS>`(created in the previous step via `spl-token create-token`) and the account keypair.

```bash
spl-token-create-account
Create a new token account

USAGE:
    spl-token create-account [FLAGS] [OPTIONS] <TOKEN_MINT_ADDRESS> [ACCOUNT_KEYPAIR]

ARGS:
    <TOKEN_MINT_ADDRESS>    The token that the account will hold
    <ACCOUNT_KEYPAIR>       Specify the account keypair. This may be a keypair file or the ASK
                            keyword. [default: associated token account for --owner]
````

Let's create a token account for the token mint address `23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY`.

```bash
spl-token create-account 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
```

The output:

```bash
Creating account 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP

Signature: 4HJ2uZBV32CLaxAvML6hZe49qhcXatzUTnN7r7t1fYrYYSQDb4kwh4kVdvhfBmRVLU3wN37ETjm9XF1nKKrtKhQD
```

NOTE: You cannot create a token account if it's already exists.

```bash
spl-token create-account 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
Creating account 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP
Error: "Error: Account already exists: 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP"
```

You can pass `create-account --owner <owner address>` to create token account for specified user.

## Decode Account - Token Account

### Decode Token Account using `spl-token display`

Let's run `spl-token display` to decode this token account.

```bash
spl-token di
splay 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP

SPL Token Account
  Address: 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP
  Program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
  Balance: 0
  Decimals: 9
  Mint: 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
  Owner: 7HjK9uvhowd7JZyq2fH5LAhCzELTuBq5oWHEBtB9SMwn
  State: Initialized
  Delegation: (not set)
  Close authority: (not set)
```

### Decode Token Account using `AccountLayout.decode`

Let's use `AccountLayout.decode` to decode this token account.

```bash
npx ts-node
decode-account.ts -a 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP
Network: devnet
Account Address: 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP
(node:34568) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)

Account Basic Info:
------------------
Owner: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
Data length: 165 bytes
Executable: false
Lamports: 0.00203928 SOL

✅ This is a Token Account

Decoded Token Info: {
  mint: '23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY',
  owner: '7HjK9uvhowd7JZyq2fH5LAhCzELTuBq5oWHEBtB9SMwn',
  amount: '0',
  delegateOption: 0,
  delegate: '11111111111111111111111111111111',
  state: 1,
  isNativeOption: 0,
  isNative: 0n,
  delegatedAmount: '0',
  closeAuthorityOption: 0,
  closeAuthority: '11111111111111111111111111111111'
}
```

As there's no balance in new token account, the balance in decoded token info is `0`.

## Mint token

To mint tokens, use `spl-token mint <token address> <amount>`.

```bash
spl-token mint 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY 10000000
```

Output:

```
Minting 10000000 tokens
  Token: 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
  Recipient: 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP
```

Recipient is the address of your wallet's token account.
You can check the balance at https://solana.fm/address/7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP/tokens?cluster=devnet-alpha

Also note that minting tokens will fail if the recipient token account has not been created first. You must create a token account before you can mint tokens to it.

```bash
spl-token mint 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY 10000000
Error: "Account 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP not found"
```

## Transfer

First, let's mint token for another wallet `Cw8N9C5eWfxJe6nNoYYL6Q4xNfH5BpCpFuPU6hPPkv4C`.

```bash
spl-token create-account --owner Cw8N9C5eWfxJe6nNoYYL6Q4xNfH5BpCpFuPU6hPPkv4C 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY --fee-payer ~/.config/solana/id.json
```

NOTE: `--fee-payer` must be provided.

Output:

```bash
Creating account 6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd

Signature: 67GJs35AWa2zqBo5kqJmae7x4s9vZCa8hinxizZz3Ac7ZwWUeamEdsjTbUskhVpgTBd1cDPdsCRYFoUJeCwauCL1
```

The account `6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd` is the token account for another wallet `Cw8N9C5eWfxJe6nNoYYL6Q4xNfH5BpCpFuPU6hPPkv4C`.

Next, we'll mint token for another token account `6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd`.

```
spl-token mint 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY 987654321 -- 6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd
```

```
Minting 987654321 tokens
  Token: 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY
  Recipient: 6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd

Signature: 4KreanW6ftnE85JXhXf1phS9KhFoL7Dhru9JE8cP3ui3sfCtJ5oa3cTstkguBpbqs9843owCZsU91HoHk4rWZRzG
```

💰 Check the balance at https://solana.fm/address/6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd/tokens?cluster=devnet-alpha

Finally, let's transfer 777777 tokens to another token account `6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd`.

```bash
spl-token transfer 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY 777777 6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd
```

Output:

```
spl-token transfer 23b9PTjuFNRrobSTmydcf4bWrei2bRdLfpwRAR8RtAUY 777777 6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd
Transfer 777777 tokens
  Sender: 7HSevHURThjhH4BeygaaBxyT7BSYruysmcRjoUPsUzLP
  Recipient: 6XVWmgPnpvLom7MSjvgvuUhNh1yzSdktpK8PTapRYkXd

Signature: 5Eni3GJGBiLFJYAigPYd5hTk4zXMHDLpVseBLVNQF2SVyunUfq3Wewpo5FoKWLdJYHK45D49m7K2K5iPYqHE5yQ4
```

The transaction is here https://solana.fm/tx/4Md3ycC6a2rU2gKXV9rY1eRdL36xULBN36aNkK6TFqKjDPKTa6GoaBzQfjMVe955HdZpRGPizhNdBEEsmc2GmZs6?cluster=devnet-alpha%3Fcluster%3Ddevnet-alpha

## Transfer if token account is not exist

If the token account is not existed, we can specify `--fund-recipient` option to fund the receiver's associated token account, at the sender's expense.

Let's create a new keypair using `solana-keygen`. The public key is `7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx`.

```bash
solana-keygen new -o account-1.json
Generating a new keypair

For added security, enter a BIP39 passphrase

NOTE! This passphrase improves security of the recovery seed phrase NOT the
keypair file itself, which is stored as insecure plain text

BIP39 Passphrase (empty for none):

Wrote new keypair to account-1.json
=============================================================================
pubkey: 7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx
=============================================================================
Save this seed phrase and your BIP39 passphrase to recover your new keypair:
odor talk orchard cable real assault common artefact example castle idle wall
=============================================================================
```

Let's create a mint token and a token account to hold the token.

```bash
spl-token create-token
Creating token J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm under program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA

Address:  J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm
Decimals:  9

Signature: 2xb99CCpA7nCxjYLjGrY7aYjV4VUk4h9ox7qSnTamN4Cm4QbW6xA9PwDXoWKUFQpPicY495DvoSXNdFv8fs28yWo
```

Create a token account to hold mint token `J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm`.

```bash
spl-token create-account J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm
Creating account AB7baqsAgkfjwQtekTuvzT5amuM1xrRSUxRpqdnkWngJ

Signature: 5YYX9XWdUwCanBzUc1sbX7XH7fqwcckrFCZQNL1AZXvsN9kjuTUCeG1uP9hWuftT5PFn4mSWNFKBEwyTBjU6LHvV
```

We'll mint `100` tokens to token account `AB7baqsAgkfjwQtekTuvzT5amuM1xrRSUxRpqdnkWngJ`.

```bash
spl-token mint J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm 100
Minting 100 tokens
  Token: J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm
  Recipient: AB7baqsAgkfjwQtekTuvzT5amuM1xrRSUxRpqdnkWngJ

Signature: 2AQNdA8yzLRwwNHMhdgk4nJEgqXZKnrHCNuXqGgUYBhTomq6rSwMUuk8RARC16eRSoBZDkS2Xnyc5a6CkH3tEMFG
```

Now, let's transfer `20` tokens to account `7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx` (created by `solana-keygen`).

```bash
spl-token transfer J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm 20 7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx
Transfer 20 tokens
  Sender: AB7baqsAgkfjwQtekTuvzT5amuM1xrRSUxRpqdnkWngJ
  Recipient: 7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx
Error: "Error: The recipient address is not funded. Add `--allow-unfunded-recipient` to complete the transfer."
```

We encounter an error. Let's pass `--allow-unfunded-recipient` option to `transfer` subcommand.

```bash
spl-token transfer --allow-unfunded-recipient J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm 20 7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx
Transfer 20 tokens
  Sender: AB7baqsAgkfjwQtekTuvzT5amuM1xrRSUxRpqdnkWngJ
  Recipient: 7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx
  Recipient associated token account: 7Q3GEiV3E9ccjP8Wyo6kyLi68QaSev1LfpRsyeQFQnyk
Error: "Error: Recipient's associated token account does not exist. Add `--fund-recipient` to fund their account"
```

Another error occurred. Let's pass `--fund-recipient` option to `transfer` subcommand.

Let's transfer tokens using `--fund-recipient` option.

```bash
spl-token transfer --allow-unfunded-recipient --fund-recipient J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm 20 7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx
Transfer 20 tokens
  Sender: AB7baqsAgkfjwQtekTuvzT5amuM1xrRSUxRpqdnkWngJ
  Recipient: 7VNq6ixbLWTz85vQLqwuUUSV66LYhhhCUMKZ13uQmuhx
  Recipient associated token account: 7Q3GEiV3E9ccjP8Wyo6kyLi68QaSev1LfpRsyeQFQnyk
  Funding recipient: 7Q3GEiV3E9ccjP8Wyo6kyLi68QaSev1LfpRsyeQFQnyk

Signature: 2YvAX3J256pWVb2GqguqbAepLaGxXyGHST2yUZk4z7ojNcGkyLNMMAE7UdGt3vmkUzj8WcCSHUWsBvgYCM1oJtpm
```

We'll see we have 80 tokens in balance, with 20 tokens transfered.

```bash
spl-token accounts
Token                                         Balance
--------------------------------------------------------
So11111111111111111111111111111111111111112   0.99796072
J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm  80
```

We can check token account balance for `7Q3GEiV3E9ccjP8Wyo6kyLi68QaSev1LfpRsyeQFQnyk`(sender funded token account through `--fund-recipient` option).

```bash
spl-token balance --address 7Q3GEiV3E9ccjP8Wyo6kyLi68QaSev1LfpRsyeQFQnyk
20
```

## Create token metadata

To create a token with metadata, you can pass the `--enable-metadata` option.

If you don't use the token 2022 program ID, you'll get a `IncorrectProgramId` error.

```bash
spl-token create-token --enable-metadata
Creating token ATsFvKCKSibNTsUNodjgpsFw52DTvf3qFtXaWtnVp35Z under program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
Error: Program(IncorrectProgramId)
```

To avoid this error, use the token 2022 program ID:

```bash
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb --enable-metadata
```

NOTE: `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` is a token-2022 program address in spl github repo.

See https://spl.solana.com/token-2022 for more details.

The source code is located in https://github.com/solana-labs/solana-program-library/tree/master/token/program-2022

Output:

```
Creating token xpJkcpUuL4gxmqfZXBXnH15sUUNRtFgrB41rSYu5mnF under program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb
To initialize metadata inside the mint, please run `spl-token initialize-metadata xpJkcpUuL4gxmqfZXBXnH15sUUNRtFgrB41rSYu5mnF <YOUR_TOKEN_NAME> <YOUR_TOKEN_SYMBOL> <YOUR_TOKEN_URI>`, and sign with the mint authority.

Address:  xpJkcpUuL4gxmqfZXBXnH15sUUNRtFgrB41rSYu5mnF
Decimals:  9

Signature: 5QErL4ZfNfgoRuc2JyS76TYPw9W8vmj6EM6mcy2KAMToGRaHHJ7nHijqoSgN8ZRFfwKXBQUs6BGoy1W3bcHoTgDj
```

Now, a new token is created, we can initialize token with metadata.

```bash
spl-token initialize-metadata <TOKEN_MINT_ADDRESS> <YOUR_TOKEN_NAME> <YOUR_TOKEN_SYMBOL> <YOUR_TOKEN_URI>
```

Run:

```bash
spl-token initialize-metadata xpJkcpUuL4gxmqfZXBXnH15sUUNRtFgrB41rSYu5mnF "Helloworld" "HelloWorld" "https://raw.githubusercontent.com/solana-developers/opos-asset/main/assets/DeveloperPortal/metadata.json"

Signature: 4mT6j4Smy6tnrEEBdmKpWvgbVoP9nc8q3uPY1rtn2LyTw5EepYdp5ctdmcL7vKSw22LC9h9J3xp8aRXd2RLqH95S
```

We use metadata in this url:
https://raw.githubusercontent.com/solana-developers/opos-asset/main/assets/DeveloperPortal/metadata.json

```json
{
  "name": "OPOS",
  "symbol": "OPOS",
  "description": "Only Possible On Solana",
  "image": "https://raw.githubusercontent.com/solana-developers/opos-asset/main/assets/DeveloperPortal/image.png",
  "attributes": [
    {
      "trait_type": "Item",
      "value": "Developer Portal"
    }
  ]
}
```

Check token metadata:
https://solana.fm/address/xpJkcpUuL4gxmqfZXBXnH15sUUNRtFgrB41rSYu5mnF/tokens?cluster=devnet-alpha

There are two transactions involved:

- https://explorer.solana.com/tx/5QErL4ZfNfgoRuc2JyS76TYPw9W8vmj6EM6mcy2KAMToGRaHHJ7nHijqoSgN8ZRFfwKXBQUs6BGoy1W3bcHoTgDj?cluster=devnet

- https://explorer.solana.com/tx/4mT6j4Smy6tnrEEBdmKpWvgbVoP9nc8q3uPY1rtn2LyTw5EepYdp5ctdmcL7vKSw22LC9h9J3xp8aRXd2RLqH95S?cluster=devnet

Transaction beginning with `5QEr...`: called `MetadataPointerInstruction::Initialize` and `Instruction: InitializeMint` to create a token mint account with metadata.

```
Program 11111111111111111111111111111111 invoke [1] Program 11111111111111111111111111111111 success Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb invoke [1] Program log: MetadataPointerInstruction::Initialize Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb consumed 2441 of 6591 compute units Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb success Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb invoke [1] Program log: Instruction: InitializeMint Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb consumed 4000 of 4150 compute units Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb success Program ComputeBudget111111111111111111111111111111 invoke [1] Program ComputeBudget111111111111111111111111111111 success
```

Transaction beginning with `4mT6...` called `TokenMetadataInstruction: Initialize` to initialize the metadata.

```
Program 11111111111111111111111111111111 invoke [1] Program 11111111111111111111111111111111 success Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb invoke [1] Program log: TokenMetadataInstruction: Initialize Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb consumed 9669 of 9819 compute units Program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb success Program ComputeBudget111111111111111111111111111111 invoke [1] Program ComputeBudget111111111111111111111111111111 success
```

You can examine the source code in detail to understand what happens. DYOR :)

# spl-token cli source code

## spl-token cli main function structure

The source code of `spl-token` is located in https://github.com/solana-labs/solana-program-library/blob/2dff12b0e5b08393999d843f52733098f398ce44/token/cli/src/main.rs#L8

The `main` function is the entry point of the `spl-token` command.

It uses the `clap` crate to parse the command line arguments and subcommands.

```rust
use {
    solana_sdk::signer::Signer,
    spl_token_cli::{clap_app::*, command::process_command, config::Config},
    std::{str::FromStr, sync::Arc},
};

#[tokio::main]
async fn main() -> Result<(), Error> {
    let default_decimals = format!("{}", spl_token_2022::native_mint::DECIMALS);
    let minimum_signers_help = minimum_signers_help_string();
    let multisig_member_help = multisig_member_help_string();
    let app_matches = app(
        &default_decimals,
        &minimum_signers_help,
        &multisig_member_help,
    )
    .get_matches();

    let mut wallet_manager = None;
    let mut bulk_signers: Vec<Arc<dyn Signer>> = Vec::new();

    let (sub_command, matches) = app_matches.subcommand().unwrap();
    let sub_command = CommandName::from_str(sub_command).unwrap();

    let mut multisigner_ids = Vec::new();
    let config = Config::new(
        matches,
        &mut wallet_manager,
        &mut bulk_signers,
        &mut multisigner_ids,
    )
    .await;

    solana_logger::setup_with_default("solana=info");
    let result =
        process_command(&sub_command, matches, &config, wallet_manager, bulk_signers).await?;
    println!("{}", result);
    Ok(())
}
```

The `process_command` function is the entry point of the `spl-token` command. Basically, it matches the subcommand and then calls the corresponding function to handle the command. In this case, it matches the `CommandName::CreateToken` subcommand and then calls the `command_create_token` function to handle the command.

```rust
pub async fn process_command<'a>(
    sub_command: &CommandName,
    sub_matches: &ArgMatches,
    config: &Config<'a>,
    mut wallet_manager: Option<Rc<RemoteWalletManager>>,
    mut bulk_signers: Vec<Arc<dyn Signer>>,
) -> CommandResult {
    match (sub_command, sub_matches) {
        (CommandName::Bench, arg_matches) => {}
        (CommandName::CreateToken, arg_matches) => {}
        (CommandName::SetInterestRate, arg_matches) => {}
        (CommandName::SetTransferHook, arg_matches) => {}
        (CommandName::InitializeMetadata, arg_matches) => {}
        (CommandName::UpdateMetadata, arg_matches) => {}
        (CommandName::InitializeGroup, arg_matches) => {}
        (CommandName::UpdateGroupMaxSize, arg_matches) => {}
        (CommandName::InitializeMember, arg_matches) => {}
        (CommandName::CreateAccount, arg_matches) => {}
        (CommandName::CreateMultisig, arg_matches) => {}
        (CommandName::Authorize, arg_matches) => {}
        (CommandName::Transfer, arg_matches) => {}
        (CommandName::Burn, arg_matches) => {}
        (CommandName::Mint, arg_matches) => {}
        (CommandName::Freeze, arg_matches) => {}
        (CommandName::Thaw, arg_matches) => {}
        (CommandName::Wrap, arg_matches) => {}
        (CommandName::Unwrap, arg_matches) => {}
        (CommandName::Approve, arg_matches) => {}
        (CommandName::Revoke, arg_matches) => {}
        (CommandName::Close, arg_matches) => {}
        (CommandName::CloseMint, arg_matches) => {}
        (CommandName::Balance, arg_matches) => {}
        (CommandName::Supply, arg_matches) => {}
        (CommandName::Accounts, arg_matches) => {}
        (CommandName::Address, arg_matches) => {}
        (CommandName::AccountInfo, arg_matches) => {}
        (CommandName::MultisigInfo, arg_matches) => {}
        (CommandName::Display, arg_matches) => {}
        (CommandName::Gc, arg_matches) => {}
        (CommandName::SyncNative, arg_matches) => {}
        (CommandName::EnableRequiredTransferMemos, arg_matches) => {}
        (CommandName::DisableRequiredTransferMemos, arg_matches) => {}
        (CommandName::EnableCpiGuard, arg_matches) => {}
        (CommandName::DisableCpiGuard, arg_matches) => {}
        (CommandName::UpdateDefaultAccountState, arg_matches) => {}
        (CommandName::UpdateMetadataAddress, arg_matches) => {}
        (CommandName::UpdateGroupAddress, arg_matches) => {}
        (CommandName::UpdateMemberAddress, arg_matches) => {}
        (CommandName::WithdrawWithheldTokens, arg_matches) => {}
        (CommandName::SetTransferFee, arg_matches) => {}
        (CommandName::WithdrawExcessLamports, arg_matches) => {}
        (CommandName::UpdateConfidentialTransferSettings, arg_matches) => {}
        (CommandName::ConfigureConfidentialTransferAccount, arg_matches) => {}
        (c @ CommandName::EnableConfidentialCredits, arg_matches)
        | (c @ CommandName::DisableConfidentialCredits, arg_matches)
        | (c @ CommandName::EnableNonConfidentialCredits, arg_matches)
        | (c @ CommandName::DisableNonConfidentialCredits, arg_matches) => {}
        (c @ CommandName::DepositConfidentialTokens, arg_matches)
        | (c @ CommandName::WithdrawConfidentialTokens, arg_matches) => {}
        (CommandName::ApplyPendingBalance, arg_matches) => {}
    }
}
```

## CommandName::CreateToken

Let's examine the `CommandName::CreateToken` case. In this match arm, the code extracts several arguments from `arg_matches`:

- `decimals`: The number of decimal places for the token (u8)
- `mint_authority`: The public key that will have permission to mint new tokens
- `memo`: An optional string memo
- `rate_bps`: An optional interest rate in basis points (i16)
- `metadata_address`: An optional Pubkey for token metadata
- `group_address`: An optional Pubkey for token group
- `member_address`: An optional Pubkey for token member

It also handles transfer fee configuration in two ways:

1. Via deprecated `transfer_fee` argument that takes two values:
   - Transfer fee basis points
   - Maximum fee amount
2. Via newer separate arguments:
   - `transfer_fee_basis_points`: Fee rate in basis points
   - `transfer_fee_maximum_fee`: Maximum fee amount with UI parsing

Finally, it sets up the token signer and public key, either from provided keypair or generates new ones, before calling `command_create_token` to create the token.

```rust
pub async fn process_command<'a>(
    sub_command: &CommandName,
    sub_matches: &ArgMatches,
    config: &Config<'a>,
    mut wallet_manager: Option<Rc<RemoteWalletManager>>,
    mut bulk_signers: Vec<Arc<dyn Signer>>,
) -> CommandResult {
    match (sub_command, sub_matches) {
        // ...
        (CommandName::CreateToken, arg_matches) => {
            let decimals = *arg_matches.get_one::<u8>("decimals").unwrap();
            let mint_authority =
                config.pubkey_or_default(arg_matches, "mint_authority", &mut wallet_manager)?;
            let memo = value_t!(arg_matches, "memo", String).ok();
            let rate_bps = value_t!(arg_matches, "interest_rate", i16).ok();
            let metadata_address = value_t!(arg_matches, "metadata_address", Pubkey).ok();
            let group_address = value_t!(arg_matches, "group_address", Pubkey).ok();
            let member_address = value_t!(arg_matches, "member_address", Pubkey).ok();

            let transfer_fee = arg_matches.values_of("transfer_fee").map(|mut v| {
                println_display(config,"transfer-fee has been deprecated and will be removed in a future release. Please specify --transfer-fee-basis-points and --transfer-fee-maximum-fee with a UI amount".to_string());
                (
                    v.next()
                        .unwrap()
                        .parse::<u16>()
                        .unwrap_or_else(print_error_and_exit),
                    v.next()
                        .unwrap()
                        .parse::<u64>()
                        .unwrap_or_else(print_error_and_exit),
                )
            });

            let transfer_fee_basis_point = arg_matches.get_one::<u16>("transfer_fee_basis_points");
            let transfer_fee_maximum_fee = arg_matches
                .get_one::<Amount>("transfer_fee_maximum_fee")
                .map(|v| amount_to_raw_amount(*v, decimals, None, "MAXIMUM_FEE"));
            let transfer_fee = transfer_fee_basis_point
                .map(|v| (*v, transfer_fee_maximum_fee.unwrap()))
                .or(transfer_fee);

            let (token_signer, token) =
                get_signer(arg_matches, "token_keypair", &mut wallet_manager)
                    .unwrap_or_else(new_throwaway_signer);
            push_signer_with_dedup(token_signer, &mut bulk_signers);
            let default_account_state =
                arg_matches
                    .value_of("default_account_state")
                    .map(|s| match s {
                        "initialized" => AccountState::Initialized,
                        "frozen" => AccountState::Frozen,
                        _ => unreachable!(),
                    });
            let transfer_hook_program_id =
                pubkey_of_signer(arg_matches, "transfer_hook", &mut wallet_manager).unwrap();

            let confidential_transfer_auto_approve = arg_matches
                .value_of("enable_confidential_transfers")
                .map(|b| b == "auto");

            // The code above prepares all the necessary variables and parameters that will be passed to
            // the command_create_token function, including:
            // - Token decimals and authority
            // - Various enable flags (freeze, close, non-transferable, etc.)
            // - Optional parameters like memo, metadata address, group/member addresses
            // - Transfer fee configuration
            // - Account state and confidential transfer settings


            // 👀 👀 👀 👀 👀 👀 👀 👀
            // This is the dirty work.
            // 👀 👀 👀 👀 👀 👀 👀 👀
            command_create_token(
                config,
                decimals,
                token,
                mint_authority,
                arg_matches.is_present("enable_freeze"),
                arg_matches.is_present("enable_close"),
                arg_matches.is_present("enable_non_transferable"),
                arg_matches.is_present("enable_permanent_delegate"),
                memo,
                metadata_address,
                group_address,
                member_address,
                rate_bps,
                default_account_state,
                transfer_fee,
                confidential_transfer_auto_approve,
                transfer_hook_program_id,
                arg_matches.is_present("enable_metadata"),
                arg_matches.is_present("enable_group"),
                arg_matches.is_present("enable_member"),
                bulk_signers,
            )
            .await
        }
    }
}
```

Let's focus on the `command_create_token` function, which will do the dirty work. It uses `#[allow(clippy::too_many_arguments)]` to suppress the warning about having too many arguments. While this is not ideal from a code design perspective, it's sometimes necessary when dealing with complex token creation parameters.

```rust
#[allow(clippy::too_many_arguments)]
async fn command_create_token(
    config: &Config<'_>,
    decimals: u8,
    token_pubkey: Pubkey,
    authority: Pubkey,
    enable_freeze: bool,
    enable_close: bool,
    enable_non_transferable: bool,
    enable_permanent_delegate: bool,
    memo: Option<String>,
    metadata_address: Option<Pubkey>,
    group_address: Option<Pubkey>,
    member_address: Option<Pubkey>,
    rate_bps: Option<i16>,
    default_account_state: Option<AccountState>,
    transfer_fee: Option<(u16, u64)>,
    confidential_transfer_auto_approve: Option<bool>,
    transfer_hook_program_id: Option<Pubkey>,
    enable_metadata: bool,
    enable_group: bool,
    enable_member: bool,
    bulk_signers: Vec<Arc<dyn Signer>>,
) -> CommandResult {
    println_display(
        config,
        format!(
            "Creating token {} under program {}",
            token_pubkey, config.program_id
        ),
    );

    let token = token_client_from_config(config, &token_pubkey, Some(decimals))?;

    let freeze_authority = if enable_freeze { Some(authority) } else { None };

    let mut extensions = vec![];

    if enable_close {
        extensions.push(ExtensionInitializationParams::MintCloseAuthority {
            close_authority: Some(authority),
        });
    }

    if enable_permanent_delegate {
        extensions.push(ExtensionInitializationParams::PermanentDelegate {
            delegate: authority,
        });
    }

    if let Some(rate_bps) = rate_bps {
        extensions.push(ExtensionInitializationParams::InterestBearingConfig {
            rate_authority: Some(authority),
            rate: rate_bps,
        })
    }

    if enable_non_transferable {
        extensions.push(ExtensionInitializationParams::NonTransferable);
    }

    if let Some(state) = default_account_state {
        assert!(
            enable_freeze,
            "Token requires a freeze authority to default to frozen accounts"
        );
        extensions.push(ExtensionInitializationParams::DefaultAccountState { state })
    }

    if let Some((transfer_fee_basis_points, maximum_fee)) = transfer_fee {
        extensions.push(ExtensionInitializationParams::TransferFeeConfig {
            transfer_fee_config_authority: Some(authority),
            withdraw_withheld_authority: Some(authority),
            transfer_fee_basis_points,
            maximum_fee,
        });
    }

    if let Some(auto_approve) = confidential_transfer_auto_approve {
        extensions.push(ExtensionInitializationParams::ConfidentialTransferMint {
            authority: Some(authority),
            auto_approve_new_accounts: auto_approve,
            auditor_elgamal_pubkey: None,
        });
        if transfer_fee.is_some() {
            // Deriving ElGamal key from default signer. Custom ElGamal keys
            // will be supported in the future once upgrading to clap-v3.
            //
            // NOTE: Seed bytes are hardcoded to be empty bytes for now. They
            // will be updated once custom ElGamal keys are supported.
            let elgamal_keypair =
                ElGamalKeypair::new_from_signer(config.default_signer()?.as_ref(), b"").unwrap();
            extensions.push(
                ExtensionInitializationParams::ConfidentialTransferFeeConfig {
                    authority: Some(authority),
                    withdraw_withheld_authority_elgamal_pubkey: (*elgamal_keypair.pubkey()).into(),
                },
            );
        }
    }

    if let Some(program_id) = transfer_hook_program_id {
        extensions.push(ExtensionInitializationParams::TransferHook {
            authority: Some(authority),
            program_id: Some(program_id),
        });
    }

    if let Some(text) = memo {
        token.with_memo(text, vec![config.default_signer()?.pubkey()]);
    }

    // CLI checks that only one is set
    if metadata_address.is_some() || enable_metadata {
        let metadata_address = if enable_metadata {
            Some(token_pubkey)
        } else {
            metadata_address
        };
        extensions.push(ExtensionInitializationParams::MetadataPointer {
            authority: Some(authority),
            metadata_address,
        });
    }

    if group_address.is_some() || enable_group {
        let group_address = if enable_group {
            Some(token_pubkey)
        } else {
            group_address
        };
        extensions.push(ExtensionInitializationParams::GroupPointer {
            authority: Some(authority),
            group_address,
        });
    }

    if member_address.is_some() || enable_member {
        let member_address = if enable_member {
            Some(token_pubkey)
        } else {
            member_address
        };
        extensions.push(ExtensionInitializationParams::GroupMemberPointer {
            authority: Some(authority),
            member_address,
        });
    }

    let res = token
        .create_mint(
            &authority,
            freeze_authority.as_ref(),
            extensions,
            &bulk_signers,
        )
        .await?;

    let tx_return = finish_tx(config, &res, false).await?;

    if enable_metadata {
        println_display(
            config,
            format!(
                "To initialize metadata inside the mint, please run \
                `spl-token initialize-metadata {token_pubkey} <YOUR_TOKEN_NAME> <YOUR_TOKEN_SYMBOL> <YOUR_TOKEN_URI>`, \
                and sign with the mint authority.",
            ),
        );
    }

    if enable_group {
        println_display(
            config,
            format!(
                "To initialize group configurations inside the mint, please run `spl-token initialize-group {token_pubkey} <MAX_SIZE>`, and sign with the mint authority.",
            ),
        );
    }

    if enable_member {
        println_display(
            config,
            format!(
                "To initialize group member configurations inside the mint, please run `spl-token initialize-member {token_pubkey}`, and sign with the mint authority and the group's update authority.",
            ),
        );
    }

    Ok(match tx_return {
        TransactionReturnData::CliSignature(cli_signature) => format_output(
            CliCreateToken {
                address: token_pubkey.to_string(),
                decimals,
                transaction_data: cli_signature,
            },
            &CommandName::CreateToken,
            config,
        ),
        TransactionReturnData::CliSignOnlyData(cli_sign_only_data) => {
            format_output(cli_sign_only_data, &CommandName::CreateToken, config)
        }
    })
}
```

This function creates a new SPL Token with the specified parameters. It first initializes a token client with the given token public key and decimals. Then it calls `create_mint` to create the actual token on-chain, passing in:

- The mint authority that can mint new tokens
- An optional freeze authority that can freeze token accounts
- Any token extensions enabled (like metadata, groups, etc)
- The bulk signers needed to authorize the transaction

The `create_mint` call returns a transaction signature that is processed and returned to the caller.

We can briefly summarize the `command_create_token` function as follows:

```rust
// 1) create the token client
let token = token_client_from_config(config, &token_pubkey, Some(decimals))?;

// 2) create the mint
let res = token
        .create_mint(
            &authority,
            freeze_authority.as_ref(),
            extensions,
            &bulk_signers,
        )
        .await?;

// 3) return the transaction signature
let tx_return = finish_tx(config, &res, false).await?;
```

Let's take a look at the first part: creating token client via `token_client_from_config` function.

The `token_client_from_config` function is used to create a token client with the given configuration. It first calls `base_token_client` to create a basic token client, then calls `config_token_client` to apply any additional configuration settings from the `config` object.

```rust
fn token_client_from_config(
    config: &Config<'_>,
    token_pubkey: &Pubkey,
    decimals: Option<u8>,
) -> Result<Token<ProgramRpcClientSendTransaction>, Error> {
    let token = base_token_client(config, token_pubkey, decimals)?;
    config_token_client(token, config)
}
```

The `base_token_client` function is used to create a basic token client. It takes the `config`, `token_pubkey`, and `decimals` as arguments and returns a `Result<Token<ProgramRpcClientSendTransaction>, Error>`.

```rust
fn base_token_client(
    config: &Config<'_>,
    token_pubkey: &Pubkey,
    decimals: Option<u8>,
) -> Result<Token<ProgramRpcClientSendTransaction>, Error> {
    Ok(Token::new(
        config.program_client.clone(),
        &config.program_id,
        token_pubkey,
        decimals,
        config.fee_payer()?.clone(),
    ))
}
```

After token client is created, the `config_token_client` function is used to apply any additional configuration settings from the `config` object to the token client. It sets the compute unit limit, compute unit price, and nonce account if they are provided in the `config` object.

```rust
fn config_token_client(
    token: Token<ProgramRpcClientSendTransaction>,
    config: &Config<'_>,
) -> Result<Token<ProgramRpcClientSendTransaction>, Error> {
    let token = token.with_compute_unit_limit(config.compute_unit_limit.clone());

    let token = if let Some(compute_unit_price) = config.compute_unit_price {
        token.with_compute_unit_price(compute_unit_price)
    } else {
        token
    };

    if let (Some(nonce_account), Some(nonce_authority), Some(nonce_blockhash)) = (
        config.nonce_account,
        &config.nonce_authority,
        config.nonce_blockhash,
    ) {
        Ok(token.with_nonce(
            &nonce_account,
            Arc::clone(nonce_authority),
            &nonce_blockhash,
        ))
    } else {
        Ok(token)
    }
}
```

The `Token::new` function is used to create a new `Token` instance. It takes the `client`, `program_id`, `address`, `decimals`, and `payer` as arguments and returns a `Token` instance.

```rust
impl<T> Token<T>
where
    T: SendTransaction + SimulateTransaction,
{
    pub fn new(
        client: Arc<dyn ProgramClient<T>>,
        program_id: &Pubkey,
        address: &Pubkey,
        decimals: Option<u8>,
        payer: Arc<dyn Signer>,
    ) -> Self {
        Token {
            client,
            pubkey: *address,
            decimals,
            payer,
            program_id: *program_id,
            nonce_account: None,
            nonce_authority: None,
            nonce_blockhash: None,
            memo: Arc::new(RwLock::new(None)),
            transfer_hook_accounts: None,
            compute_unit_price: None,
            compute_unit_limit: ComputeUnitLimit::Default,
        }
    }
}
```

The `Token` struct is defined as follows:

```rust
pub struct Token<T> {
    client: Arc<dyn ProgramClient<T>>,
    pubkey: Pubkey, /* token mint */
    decimals: Option<u8>,
    payer: Arc<dyn Signer>,
    program_id: Pubkey,
    nonce_account: Option<Pubkey>,
    nonce_authority: Option<Arc<dyn Signer>>,
    nonce_blockhash: Option<Hash>,
    memo: Arc<RwLock<Option<TokenMemo>>>,
    transfer_hook_accounts: Option<Vec<AccountMeta>>,
    compute_unit_price: Option<u64>,
    compute_unit_limit: ComputeUnitLimit,
}
```

It is a generic struct that represents an SPL Token. It contains various fields that store information about the token, such as its `client`, `pubkey`, `decimals`, `payer`, `program_id`, `nonce_account`, `nonce_authority`, `nonce_blockhash`, `memo`, `transfer_hook_accounts`, `compute_unit_price`, and `compute_unit_limit`.

You may notice that the `client` field is an `Arc<dyn ProgramClient<T>>`. This is a generic type that implements the `ProgramClient` trait. It's used to send transactions to the network.

It's actually an instance of `ProgramRpcClient` that implements `ProgramClient` trait. It's created in `Config::new` function in `fn main`. It wraps `RpcClient` and provides additional features for programmatic use. We'll talk about it later.

```rust
impl<'a> Config<'a> {
    pub async fn new(
        matches: &ArgMatches,
        wallet_manager: &mut Option<Rc<RemoteWalletManager>>,
        bulk_signers: &mut Vec<Arc<dyn Signer>>,
        multisigner_ids: &'a mut Vec<Pubkey>,
    ) -> Config<'a> {
         let rpc_client = Arc::new(RpcClient::new_with_timeouts_and_commitment(
            json_rpc_url,
            DEFAULT_RPC_TIMEOUT,
            commitment_config,
            DEFAULT_CONFIRM_TX_TIMEOUT,
        ));
        Arc::new(ProgramRpcClient::new(
            rpc_client.clone(),
            ProgramRpcClientSendTransaction,
        ))
    }
}
```

### create_mint

The `create_mint` function is a public method for `Token` struct that creates a new SPL Token. It takes the `mint_authority`, `freeze_authority`, `extension_initialization_params`, and `signing_keypairs` as arguments and returns a `TokenResult<T::Output>`.

Let's how `create_mint` function works:

```rust
impl<T> Token<T>
where
    T: SendTransaction + SimulateTransaction,
{
    #[allow(clippy::too_many_arguments)]
    pub async fn create_mint<'a, S: Signers>(
        &self,
        mint_authority: &'a Pubkey,
        freeze_authority: Option<&'a Pubkey>,
        extension_initialization_params: Vec<ExtensionInitializationParams>,
        signing_keypairs: &S,
    ) -> TokenResult<T::Output> {
        let decimals = self.decimals.ok_or(TokenError::MissingDecimals)?;

        let extension_types = extension_initialization_params
            .iter()
            .map(|e| e.extension())
            .collect::<Vec<_>>();
        let space = ExtensionType::try_calculate_account_len::<Mint>(&extension_types)?;

        let mut instructions = vec![system_instruction::create_account(
            &self.payer.pubkey(),
            &self.pubkey,
            self.client
                .get_minimum_balance_for_rent_exemption(space)
                .await
                .map_err(TokenError::Client)?,
            space as u64,
            &self.program_id,
        )];

        for params in extension_initialization_params {
            instructions.push(params.instruction(&self.program_id, &self.pubkey)?);
        }

        instructions.push(instruction::initialize_mint(
            &self.program_id,
            &self.pubkey,
            mint_authority,
            freeze_authority,
            decimals,
        )?);

        self.process_ixs(&instructions, signing_keypairs).await
    }
}
```

Let's see how these arguments are used in the `process_command` function.

First, `mint_authority` is set to the pubkey of the signer. It is defaulted to the pubkey of the signer if not provided.

```rust
pub async fn process_command<'a>(
    sub_command: &CommandName,
    sub_matches: &ArgMatches,
    config: &Config<'a>,
    mut wallet_manager: Option<Rc<RemoteWalletManager>>,
    mut bulk_signers: Vec<Arc<dyn Signer>>,
) -> CommandResult {
    match (sub_command, sub_matches) {
        (CommandName::CreateToken, arg_matches) => {
            let mint_authority =
                config.pubkey_or_default(arg_matches, "mint_authority", &mut wallet_manager)?;
        }
    }
}
```

The `pubkey_or_default` function is used to get the pubkey of the signer. It takes the `arg_matches`, `address_name`, and `wallet_manager` as arguments and returns a `Result<Pubkey, Error>`. If an explicit address is provided, it returns the address. Otherwise, it returns the default address: `self.default_signer()?.pubkey()`.

```rust
    // Checks if an explicit address was provided, otherwise return the default
    // address if there is one
    pub(crate) fn pubkey_or_default(
        &self,
        arg_matches: &ArgMatches,
        address_name: &str,
        wallet_manager: &mut Option<Rc<RemoteWalletManager>>,
    ) -> Result<Pubkey, Error> {
        if let Some(address) = pubkey_of_signer(arg_matches, address_name, wallet_manager)
            .map_err(|e| -> Error { e.to_string().into() })?
        {
            return Ok(address);
        }

        Ok(self.default_signer()?.pubkey())
    }
```

The `default_signer` function will load the default signer from the config file. It is defaulted to `~/.config/solana/cli/config.yml`.

```rust
impl<'a> Config<'a> {
    // Returns Ok(default signer), or Err if there is no default signer configured
    pub(crate) fn default_signer(&self) -> Result<Arc<dyn Signer>, Error> {
        if let Some(default_signer) = &self.default_signer {
            Ok(default_signer.clone())
        } else {
            Err("default signer is required, please specify a valid default signer by identifying a \
                 valid configuration file using the --config argument, or by creating a valid config \
                 at the default location of ~/.config/solana/cli/config.yml using the solana config \
                 command".to_string().into())
        }
    }
    pub async fn new_with_clients_and_ws_url(
        matches: &ArgMatches,
        wallet_manager: &mut Option<Rc<RemoteWalletManager>>,
        bulk_signers: &mut Vec<Arc<dyn Signer>>,
        multisigner_ids: &'a mut Vec<Pubkey>,
        rpc_client: Arc<RpcClient>,
        program_client: Arc<dyn ProgramClient<ProgramRpcClientSendTransaction>>,
        websocket_url: String,
    ) -> Config<'a> {
        let cli_config = if let Some(config_file) = matches.value_of("config_file") {
            solana_cli_config::Config::load(config_file).unwrap_or_else(|_| {
                eprintln!("error: Could not find config file `{}`", config_file);
                exit(1);
            })
        } else if let Some(config_file) = &*solana_cli_config::CONFIG_FILE {
            solana_cli_config::Config::load(config_file).unwrap_or_default()
        } else {
            solana_cli_config::Config::default()
        };
        let multisigner_pubkeys =
            Self::extract_multisig_signers(matches, wallet_manager, bulk_signers, multisigner_ids);

        let config = SignerFromPathConfig {
            allow_null_signer: !multisigner_pubkeys.is_empty(),
        };

        let default_keypair = cli_config.keypair_path.clone();

        let default_signer: Option<Arc<dyn Signer>> = {
            if let Some(owner_path) = matches.try_get_one::<String>("owner").ok().flatten() {
                signer_from_path_with_config(matches, owner_path, "owner", wallet_manager, &config)
                    .ok()
            } else {
                signer_from_path_with_config(
                    matches,
                    &default_keypair,
                    "default",
                    wallet_manager,
                    &config,
                )
                .map_err(|e| {
                    if std::fs::metadata(&default_keypair).is_ok() {
                        eprintln!("error: {}", e);
                        exit(1);
                    } else {
                        e
                    }
                })
                .ok()
            }
        }
        .map(Arc::from);
    }
}
```

Let's go back to the `process_command` function.

`freeze_authority` is set to `authority` if `enable_freeze` is true, otherwise it is set to `None`. This means that if freezing is enabled, the authority (signer) will have the ability to freeze token accounts, preventing any transfers of tokens from those accounts. If freezing is disabled, no one will have the authority to freeze token accounts.

```rust
#[allow(clippy::too_many_arguments)]
async fn command_create_token(
    config: &Config<'_>,
    decimals: u8,
    token_pubkey: Pubkey,
    authority: Pubkey,
    enable_freeze: bool,
    // omit...
) -> CommandResult {
    let token = token_client_from_config(config, &token_pubkey, Some(decimals))?;

    let freeze_authority = if enable_freeze { Some(authority) } else { None };
    // omit...
}
```

Then, we'll prepare instructions for creating the token. There are three instructions involved:

- `system_instruction::create_account`
- extension initialization instructions(optional)
- `instruction::initialize_mint`

The `system_instruction::create_account` instruction is used to create a new account with the given parameters.

```rust
let mut instructions = vec![system_instruction::create_account(
    &self.payer.pubkey(),
    &self.pubkey,
    self.client
        .get_minimum_balance_for_rent_exemption(space)
        .await
        .map_err(TokenError::Client)?,
    space as u64,
    &self.program_id,
)];
```

If extension parameters exist, we'll push their corresponding initialization instructions to the `instructions` vector. These extensions can add additional functionality to the token, such as transfer fees or interest-bearing capabilities.

```rust
for params in extension_initialization_params {
    instructions.push(params.instruction(&self.program_id, &self.pubkey)?);
}
```

Next, we'll push the `instruction::initialize_mint` instruction to the `instructions` vector.

```rust
instructions.push(instruction::initialize_mint(
    &self.program_id,
    &self.pubkey,
    mint_authority,
    freeze_authority,
    decimals,
));
```

Finally, we'll call `process_ixs` to process the instructions.

```rust
self.process_ixs(&instructions, signing_keypairs).await
```

Below is the implementation of `process_ixs` function for `Token` struct.

```rust
impl<T> Token<T>
where
    T: SendTransaction + SimulateTransaction,
{
    pub async fn process_ixs<S: Signers>(
        &self,
        token_instructions: &[Instruction],
        signing_keypairs: &S,
    ) -> TokenResult<T::Output> {
        let transaction = self
            .construct_tx(token_instructions, signing_keypairs)
            .await?;

        self.client
            .send_transaction(&transaction)
            .await
            .map_err(TokenError::Client)
    }
}
```

The `process_ixs` function builds the transaction using `construct_tx` and submits it to the Solana network via the client's `send_transaction` method.

If you are interested in the signing part, you can check the `construct_tx` function.

```rust
async fn construct_tx<S: Signers>(
        &self,
        token_instructions: &[Instruction],
        signing_keypairs: &S,
    ) -> TokenResult<Transaction> {
        let mut instructions = vec![];
        let payer_key = self.payer.pubkey();
        let fee_payer = Some(&payer_key);

        {
            let mut w_memo = self.memo.write().unwrap();
            if let Some(memo) = w_memo.take() {
                let signing_pubkeys = signing_keypairs.pubkeys();
                if !memo
                    .signers
                    .iter()
                    .all(|signer| signing_pubkeys.contains(signer))
                {
                    return Err(TokenError::MissingMemoSigner);
                }

                instructions.push(memo.to_instruction());
            }
        }

        instructions.extend_from_slice(token_instructions);

        let blockhash = if let (Some(nonce_account), Some(nonce_authority), Some(nonce_blockhash)) = (
            self.nonce_account,
            &self.nonce_authority,
            self.nonce_blockhash,
        ) {
            let nonce_instruction = system_instruction::advance_nonce_account(
                &nonce_account,
                &nonce_authority.pubkey(),
            );
            instructions.insert(0, nonce_instruction);
            nonce_blockhash
        } else {
            self.client
                .get_latest_blockhash()
                .await
                .map_err(TokenError::Client)?
        };

        if let Some(compute_unit_price) = self.compute_unit_price {
            instructions.push(ComputeBudgetInstruction::set_compute_unit_price(
                compute_unit_price,
            ));
        }

        // The simulation to find out the compute unit usage must be run after
        // all instructions have been added to the transaction, so be sure to
        // keep this instruction as the last one before creating and sending the
        // transaction.
        match self.compute_unit_limit {
            ComputeUnitLimit::Default => {}
            ComputeUnitLimit::Simulated => {
                self.add_compute_unit_limit_from_simulation(&mut instructions, &blockhash)
                    .await?;
            }
            ComputeUnitLimit::Static(compute_unit_limit) => {
                instructions.push(ComputeBudgetInstruction::set_compute_unit_limit(
                    compute_unit_limit,
                ));
            }
        }

        let message = Message::new_with_blockhash(&instructions, fee_payer, &blockhash);
        let mut transaction = Transaction::new_unsigned(message);
        let signing_pubkeys = signing_keypairs.pubkeys();

        if !signing_pubkeys.contains(&self.payer.pubkey()) {
            transaction
                .try_partial_sign(&vec![self.payer.clone()], blockhash)
                .map_err(|error| TokenError::Client(error.into()))?;
        }
        if let Some(nonce_authority) = &self.nonce_authority {
            let nonce_authority_pubkey = nonce_authority.pubkey();
            if nonce_authority_pubkey != self.payer.pubkey()
                && !signing_pubkeys.contains(&nonce_authority_pubkey)
            {
                transaction
                    .try_partial_sign(&vec![nonce_authority.clone()], blockhash)
                    .map_err(|error| TokenError::Client(error.into()))?;
            }
        }
        transaction
            .try_partial_sign(signing_keypairs, blockhash)
            .map_err(|error| TokenError::Client(error.into()))?;

        Ok(transaction)
    }
}
```

The transaction is sent through the `send_transaction` method via `ProgramRpcClient`, which is a wrapper around `RpcClient`.

As the documentation of `RpcClient` says, it communicates with a Solana node over JSON-RPC protocol to submit transactions to the network.

> `RpcClient` communicates with a Solana node over [JSON-RPC], with the
> [Solana JSON-RPC protocol][jsonprot]. It is the primary Rust interface for
> querying and transacting with the network from external programs.

```rust
/// A client of a remote Solana node.
///
/// `RpcClient` communicates with a Solana node over [JSON-RPC], with the
/// [Solana JSON-RPC protocol][jsonprot]. It is the primary Rust interface for
/// querying and transacting with the network from external programs.
///
/// This type builds on the underlying RPC protocol, adding extra features such
/// as timeout handling, retries, and waiting on transaction [commitment levels][cl].
/// Some methods simply pass through to the underlying RPC protocol. Not all RPC
/// methods are encapsulated by this type, but `RpcClient` does expose a generic
/// [`send`](RpcClient::send) method for making any [`RpcRequest`].
///
/// The documentation for most `RpcClient` methods contains an "RPC Reference"
/// section that links to the documentation for the underlying JSON-RPC method.
/// The documentation for `RpcClient` does not reproduce the documentation for
/// the underlying JSON-RPC methods. Thus reading both is necessary for complete
/// understanding.
///
/// `RpcClient`s generally communicate over HTTP on port 8899, a typical server
/// URL being "http://localhost:8899".
///
/// Methods that query information from recent [slots], including those that
/// confirm transactions, decide the most recent slot to query based on a
/// [commitment level][cl], which determines how committed or finalized a slot
/// must be to be considered for the query. Unless specified otherwise, the
/// commitment level is [`Finalized`], meaning the slot is definitely
/// permanently committed. The default commitment level can be configured by
/// creating `RpcClient` with an explicit [`CommitmentConfig`], and that default
/// configured commitment level can be overridden by calling the various
/// `_with_commitment` methods, like
/// [`RpcClient::confirm_transaction_with_commitment`]. In some cases the
/// configured commitment level is ignored and `Finalized` is used instead, as
/// in [`RpcClient::get_blocks`], where it would be invalid to use the
/// [`Processed`] commitment level. These exceptions are noted in the method
/// documentation.
pub struct RpcClient {
    sender: Box<dyn RpcSender + Send + Sync + 'static>,
    config: RpcClientConfig,
}
```

The `ProgramRpcClient` is created in the `new` function of `Config` struct, which wraps `RpcClient` and provides additional features for programmatic use.

```rust
impl<'a> Config<'a> {
    pub async fn new(
        matches: &ArgMatches,
        wallet_manager: &mut Option<Rc<RemoteWalletManager>>,
        bulk_signers: &mut Vec<Arc<dyn Signer>>,
        multisigner_ids: &'a mut Vec<Pubkey>,
    ) -> Config<'a> {
         let rpc_client = Arc::new(RpcClient::new_with_timeouts_and_commitment(
            json_rpc_url,
            DEFAULT_RPC_TIMEOUT,
            commitment_config,
            DEFAULT_CONFIRM_TX_TIMEOUT,
        ));
        let program_client: Arc<dyn ProgramClient<ProgramRpcClientSendTransaction>> = if sign_only {
            let blockhash = matches
                .get_one::<Hash>(BLOCKHASH_ARG.name)
                .copied()
                .unwrap_or_default();
            Arc::new(ProgramOfflineClient::new(
                blockhash,
                ProgramRpcClientSendTransaction,
            ))
        } else {
            Arc::new(ProgramRpcClient::new(
                rpc_client.clone(),
                ProgramRpcClientSendTransaction,
            ))
        };
    }
}
```

Once the transaction is created and signed, it is sent to the client through the `send_transaction` method.

```rust
impl<T> Token<T>
where
    T: SendTransaction + SimulateTransaction,
{
    pub async fn process_ixs<S: Signers>(
        &self,
        token_instructions: &[Instruction],
        signing_keypairs: &S,
    ) -> TokenResult<T::Output> {
        // build the transaction
        let transaction = self
            .construct_tx(token_instructions, signing_keypairs)
            .await?;

        // submit the transaction
        self.client
            .send_transaction(&transaction)
            .await
            .map_err(TokenError::Client)
    }
}
```

Notice we initialized a `ProgramRpcClient` in the `new` function of `Config` struct.

```rust
impl<'a> Config<'a> {
    pub async fn new(
        matches: &ArgMatches,
        wallet_manager: &mut Option<Rc<RemoteWalletManager>>,
        bulk_signers: &mut Vec<Arc<dyn Signer>>,
        multisigner_ids: &'a mut Vec<Pubkey>,
    ) -> Config<'a> {
        // Create an RpcClient with the given parameters
        // This is the client that will be used to send the transaction to the network
        let rpc_client = Arc::new(RpcClient::new_with_timeouts_and_commitment(
            json_rpc_url,
            DEFAULT_RPC_TIMEOUT,
            commitment_config,
            DEFAULT_CONFIRM_TX_TIMEOUT,
        ));

        // Create a ProgramRpcClient with the RpcClient and the ProgramRpcClientSendTransaction instance
        let program_client: Arc<dyn ProgramClient<ProgramRpcClientSendTransaction>> = if sign_only {
            let blockhash = matches
                .get_one::<Hash>(BLOCKHASH_ARG.name)
                .copied()
                .unwrap_or_default();
            Arc::new(ProgramOfflineClient::new(
                blockhash,
                ProgramRpcClientSendTransaction,
            ))
        } else {
            // Create a ProgramRpcClient with the RpcClient and the ProgramRpcClientSendTransaction instance
            Arc::new(ProgramRpcClient::new(
                rpc_client.clone(),
                ProgramRpcClientSendTransaction,
            ))
        };
    }
}
```

And the `ProgramRpcClient` implements the `ProgramClient` trait. The `SendTransactionRpc` trait contains the `send` method.

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
```

In the implementation of `ProgramRpcClient`, the `send_transaction` method will call the `send` method of the `SendTransactionRpc` trait, as there is a trait bound `ST: SendTransactionRpc` in the `ProgramClient` trait, meaning that `ST` must implement the `SendTransactionRpc` trait, so that we can call the `send` method.

In other words, the `send_transaction` method is delegated to the `send` method of the `SendTransactionRpc` trait. Because `ProgramRpcClientSendTransaction` struct implements the `SendTransactionRpc` trait, so the `send_transaction` method is delegated to the `send` method of the `ProgramRpcClientSendTransaction` struct.

```rust
#[async_trait]
impl<ST> ProgramClient<ST> for ProgramRpcClient<ST>
where
    ST: SendTransactionRpc + SimulateTransactionRpc + Send + Sync,
{
    // Delegate the send_transaction business logic to the send method of the
    // SendTransactionRpc trait
    // The send method is implemented in the ProgramRpcClientSendTransaction struct,
    // which implements the SendTransactionRpc trait
    async fn send_transaction(&self, transaction: &Transaction) -> ProgramClientResult<ST::Output> {
        self.send.send(&self.client, transaction).await
    }
}
```

Below is the definition of the `ProgramRpcClientSendTransaction` struct, which implements the `SendTransaction` trait. This struct is the instance of the generic type `ST` in the generic type `ProgramRpcClient`.

```rust
#[derive(Debug, Clone, Copy, Default)]
pub struct ProgramRpcClientSendTransaction;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum RpcClientResponse {
    Signature(Signature),
    Transaction(Transaction),
    Simulation(RpcSimulateTransactionResult),
}

// Because the trait bound `ST: SendTransactionRpc + SimulateTransactionRpc + Send + Sync` is required in the `ProgramClient` trait, and SendTransactionRpc trait extends SendTransaction trait,
// so the `ProgramRpcClientSendTransaction` struct implements the `SendTransaction` and `SendTransactionRpc` traits.
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

            // Notice this client is saved in the `ProgramRpcClient` struct
            client
                .send_and_confirm_transaction(transaction)
                .await
                .map(RpcClientResponse::Signature)
                .map_err(Into::into)
        })
    }
}
```

The interesting part is that the `send` method of the `ProgramRpcClientSendTransaction` struct is implemented to call the `send_and_confirm_transaction` method of the `RpcClient` struct. This is the method that actually sends the transaction to the network.

Conclusion:

The `ProgramRpcClient` is a client that sends transactions to the network using the `RpcClient` struct. The `ProgramRpcClientSendTransaction` struct is the instance of the generic type `ST` in the generic type `ProgramRpcClient`. The `send` method of the `ProgramRpcClientSendTransaction` struct is implemented to call the `send_and_confirm_transaction` method of the `RpcClient` struct. This is the method that actually sends the transaction to the network.

We can create different clients in different environments, such as `ProgramRpcClient` for production environment, `ProgramBanksClient` for testing environment, and `ProgramOfflineClient` for offline environment. That's why we have three different clients defination in the client module.

## CommandName::CreateAccount

Let's examine the `CommandName::CreateAccount` case. In this match arm, the code extracts several arguments from `arg_matches`:

- `token`: The public key of the token to create an associated token account for
- `owner`: The public key of the owner of the associated token account
- `immutable`: A boolean flag indicating if the associated token account is immutable
- `bulk_signers`: A vector of signers to use for the transaction

```rust
pub async fn process_command<'a>(
    sub_command: &CommandName,
    sub_matches: &ArgMatches,
    config: &Config<'a>,
    mut wallet_manager: Option<Rc<RemoteWalletManager>>,
    mut bulk_signers: Vec<Arc<dyn Signer>>,
) -> CommandResult {
    match (sub_command, sub_matches) {
        // ...
        (CommandName::CreateAccount, arg_matches) => {
            let token = pubkey_of_signer(arg_matches, "token", &mut wallet_manager)
                .unwrap()
                .unwrap();

            // No need to add a signer when creating an associated token account
            let account = get_signer(arg_matches, "account_keypair", &mut wallet_manager).map(
                |(signer, account)| {
                    push_signer_with_dedup(signer, &mut bulk_signers);
                    account
                },
            );

            let owner = config.pubkey_or_default(arg_matches, "owner", &mut wallet_manager)?;
            command_create_account(
                config,
                token,
                owner,
                account,
                arg_matches.is_present("immutable"),
                bulk_signers,
            )
            .await
        }
    }
}
```

Let's look at the `command_create_account` function.

```rust
async fn command_create_account(
    config: &Config<'_>,
    token_pubkey: Pubkey,
    owner: Pubkey,
    maybe_account: Option<Pubkey>,
    immutable_owner: bool,
    bulk_signers: Vec<Arc<dyn Signer>>,
) -> CommandResult {
    let token = token_client_from_config(config, &token_pubkey, None)?;
    let mut extensions = vec![];

    let (account, is_associated) = if let Some(account) = maybe_account {
        (
            account,
            token.get_associated_token_address(&owner) == account,
        )
    } else {
        (token.get_associated_token_address(&owner), true)
    };

    println_display(config, format!("Creating account {}", account));

    if !config.sign_only {
        if let Some(account_data) = config.program_client.get_account(account).await? {
            if account_data.owner != system_program::id() || !is_associated {
                return Err(format!("Error: Account already exists: {}", account).into());
            }
        }
    }

    if immutable_owner {
        if config.program_id == spl_token::id() {
            return Err(format!(
                "Specified --immutable, but token program {} does not support the extension",
                config.program_id
            )
            .into());
        } else if is_associated {
            println_display(
                config,
                "Note: --immutable specified, but Token-2022 ATAs are always immutable, ignoring"
                    .to_string(),
            );
        } else {
            extensions.push(ExtensionType::ImmutableOwner);
        }
    }

    let res = if is_associated {
        println!("🌈🌈🌈 is_associated");
        token.create_associated_token_account(&owner).await
    } else {
        let signer = bulk_signers
            .iter()
            .find(|signer| signer.pubkey() == account)
            .unwrap_or_else(|| panic!("No signer provided for account {}", account));

        token
            .create_auxiliary_token_account_with_extension_space(&**signer, &owner, extensions)
            .await
    }?;

    let tx_return = finish_tx(config, &res, false).await?;
    Ok(match tx_return {
        TransactionReturnData::CliSignature(signature) => {
            config.output_format.formatted_string(&signature)
        }
        TransactionReturnData::CliSignOnlyData(sign_only_data) => {
            config.output_format.formatted_string(&sign_only_data)
        }
    })
}
```

This function is the core logic of the `CommandName::CreateAccount` command. It creates an associated token account for the given token and owner.

The `token.create_associated_token_account(&owner).await` method is the method that actually creates the associated token account.

Let's look at the `create_associated_token_account` method of the `Token` struct. It is implemented as follows:

```rust
impl<T> Token<T>
where
    T: SendTransaction + SimulateTransaction,
{
    /// Create and initialize the associated account.
    pub async fn create_associated_token_account(&self, owner: &Pubkey) -> TokenResult<T::Output> {
        self.process_ixs::<[&dyn Signer; 0]>(
            &[create_associated_token_account(
                &self.payer.pubkey(),
                owner,
                &self.pubkey,
                &self.program_id,
            )],
            &[],
        )
        .await
    }
}
```

It uses `create_associated_token_account` function to create an `instruction` to create an associated token account. This function will call `get_associated_token_address_and_bump_seed_internal`, which is an internal function to generate PDA(See `Pubkey::find_program_address` and `Pubkey::try_find_program_address` method).

```rust
/// Creates Create instruction
pub fn create_associated_token_account(
    funding_address: &Pubkey,
    wallet_address: &Pubkey,
    token_mint_address: &Pubkey,
    token_program_id: &Pubkey,
) -> Instruction {
    build_associated_token_account_instruction(
        funding_address,
        wallet_address,
        token_mint_address,
        token_program_id,
        0, // AssociatedTokenAccountInstruction::Create
    )
}

fn build_associated_token_account_instruction(
    funding_address: &Pubkey,
    wallet_address: &Pubkey,
    token_mint_address: &Pubkey,
    token_program_id: &Pubkey,
    instruction: u8,
) -> Instruction {
    let associated_account_address = get_associated_token_address_with_program_id(
        wallet_address,
        token_mint_address,
        token_program_id,
    );
    // safety check, assert if not a creation instruction, which is only 0 or 1
    assert!(instruction <= 1);
    Instruction {
        program_id: id(),
        accounts: vec![
            AccountMeta::new(*funding_address, true),
            AccountMeta::new(associated_account_address, false),
            AccountMeta::new_readonly(*wallet_address, false),
            AccountMeta::new_readonly(*token_mint_address, false),
            AccountMeta::new_readonly(SYSTEM_PROGRAM_ID, false),
            AccountMeta::new_readonly(*token_program_id, false),
        ],
        data: vec![instruction],
    }
}

/// Derives the associated token account address for the given wallet address,
/// token mint and token program id
pub fn get_associated_token_address_with_program_id(
    wallet_address: &Pubkey,
    token_mint_address: &Pubkey,
    token_program_id: &Pubkey,
) -> Pubkey {
    get_associated_token_address_and_bump_seed(
        wallet_address,
        token_mint_address,
        // NOTICE: This is the SPL Associated token program id:
        // ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
        &crate::program::id(),
        token_program_id,
    )
    .0
}

/// Derives the associated token account address and bump seed
/// for the given wallet address, token mint and token program id
pub fn get_associated_token_address_and_bump_seed(
    wallet_address: &Pubkey,
    token_mint_address: &Pubkey,
    program_id: &Pubkey,
    token_program_id: &Pubkey,
) -> (Pubkey, u8) {
    get_associated_token_address_and_bump_seed_internal(
        wallet_address,
        token_mint_address,
        program_id,
        token_program_id,
    )
}

/// For internal use only.
#[doc(hidden)]
pub fn get_associated_token_address_and_bump_seed_internal(
    wallet_address: &Pubkey,
    token_mint_address: &Pubkey,
    program_id: &Pubkey,
    token_program_id: &Pubkey,
) -> (Pubkey, u8) {
    Pubkey::find_program_address(
        &[
            &wallet_address.to_bytes(),
            &token_program_id.to_bytes(),
            &token_mint_address.to_bytes(),
        ],
        program_id,
    )
}

impl Pubkey {
    /// Find a valid [program derived address][pda] and its corresponding bump seed.
    ///
    /// [pda]: https://solana.com/docs/core/cpi#program-derived-addresses
    ///
    /// Program derived addresses (PDAs) are account keys that only the program,
    /// `program_id`, has the authority to sign. The address is of the same form
    /// as a Solana `Pubkey`, except they are ensured to not be on the ed25519
    /// curve and thus have no associated private key. When performing
    /// cross-program invocations the program can "sign" for the key by calling
    /// [`invoke_signed`] and passing the same seeds used to generate the
    /// address, along with the calculated _bump seed_, which this function
    /// returns as the second tuple element. The runtime will verify that the
    /// program associated with this address is the caller and thus authorized
    /// to be the signer.
    #[cfg(any(target_os = "solana", feature = "curve25519"))]
    pub fn find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> (Pubkey, u8) {
        Self::try_find_program_address(seeds, program_id)
            .unwrap_or_else(|| panic!("Unable to find a viable program address bump seed"))
    }

    /// Find a valid [program derived address][pda] and its corresponding bump seed.
    ///
    /// [pda]: https://solana.com/docs/core/cpi#program-derived-addresses
    ///
    /// The only difference between this method and [`find_program_address`]
    /// is that this one returns `None` in the statistically improbable event
    /// that a bump seed cannot be found; or if any of `find_program_address`'s
    /// preconditions are violated.
    ///
    /// See the documentation for [`find_program_address`] for a full description.
    ///
    /// [`find_program_address`]: Pubkey::find_program_address
    // If target_os = "solana", then the function will use
    // syscalls which bring no dependencies.
    // When target_os != "solana", this should be opt-in so users
    // don't need the curve25519 dependency.
    #[cfg(any(target_os = "solana", feature = "curve25519"))]
    #[allow(clippy::same_item_push)]
    pub fn try_find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> Option<(Pubkey, u8)> {
        // Perform the calculation inline, calling this from within a program is
        // not supported
        #[cfg(not(target_os = "solana"))]
        {
            let mut bump_seed = [u8::MAX];
            for _ in 0..u8::MAX {
                {
                    let mut seeds_with_bump = seeds.to_vec();
                    seeds_with_bump.push(&bump_seed);
                    match Self::create_program_address(&seeds_with_bump, program_id) {
                        Ok(address) => return Some((address, bump_seed[0])),
                        Err(PubkeyError::InvalidSeeds) => (),
                        _ => break,
                    }
                }
                bump_seed[0] -= 1;
            }
            None
        }
        // Call via a system call to perform the calculation
        #[cfg(target_os = "solana")]
        {
            let mut bytes = [0; 32];
            let mut bump_seed = u8::MAX;
            let result = unsafe {
                crate::syscalls::sol_try_find_program_address(
                    seeds as *const _ as *const u8,
                    seeds.len() as u64,
                    program_id as *const _ as *const u8,
                    &mut bytes as *mut _ as *mut u8,
                    &mut bump_seed as *mut _ as *mut u8,
                )
            };
            match result {
                SUCCESS => Some((Pubkey::from(bytes), bump_seed)),
                _ => None,
            }
        }
    }
}
```

Now we know an ATA is a PDA. We can verify it using `PublicKey.findProgramAddress` in typescript.

```typescript
import {
  TOKEN_PROGRAM_ID,
  ASSOCIATED_TOKEN_PROGRAM_ID,
} from "@solana/spl-token";
import { PublicKey } from "@solana/web3.js";
import yargs from "yargs";

const argv = yargs
  .option("account", {
    alias: "a",
    description: "Account address to query",
    type: "string",
    demandOption: true,
  })
  .option("token", {
    alias: "t",
    description: "Token address to use",
    type: "string",
    demandOption: true,
  })
  .help()
  .parseSync();

//PubKey of Associated Token Program https://explorer.solana.com/address/ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
let SPL_ASSOCIATED_TOKEN_ACCOUNT_PROGRAM_ID = ASSOCIATED_TOKEN_PROGRAM_ID,
  YOUR_DEFAULT_SOL_ADDRESS = argv.account,
  TOKEN_ADDRESS = argv.token;

// The rust code for the function is:
//
// /// For internal use only.
// #[doc(hidden)]
// pub fn get_associated_token_address_and_bump_seed_internal(
//     wallet_address: &Pubkey,
//     token_mint_address: &Pubkey,
//     program_id: &Pubkey,
//     token_program_id: &Pubkey,
// ) -> (Pubkey, u8) {
//     Pubkey::find_program_address(
//         &[
//             &wallet_address.to_bytes(),
//             &token_program_id.to_bytes(),
//             &token_mint_address.to_bytes(),
//         ],
//         program_id,
//     )
// }
async function findAssociatedTokenAddress(
  address: string,
  tokenAddress: string
) {
  // NOTE: This async version is deprecated, use sync version
  const result = await PublicKey.findProgramAddress(
    [
      // wallet address, your default Solana address to transform to Associated token address(ATA)
      new PublicKey(address).toBuffer(),
      // address of SPL token program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
      TOKEN_PROGRAM_ID.toBuffer(),
      // token mint address
      new PublicKey(tokenAddress).toBuffer(),
    ],
    // address of SPL Associated Token Account program: ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
    SPL_ASSOCIATED_TOKEN_ACCOUNT_PROGRAM_ID
  ).then((result) => result[0]);

  return result;
}

function findAssociatedTokenAddressSync(address: string, tokenAddress: string) {
  const result = PublicKey.findProgramAddressSync(
    [
      // wallet address, your default Solana address to transform to Associated token address(ATA)
      new PublicKey(address).toBuffer(),
      // address of SPL token program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
      TOKEN_PROGRAM_ID.toBuffer(),
      // token mint address
      new PublicKey(tokenAddress).toBuffer(),
    ],
    // address of SPL Associated Token Account program: ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
    SPL_ASSOCIATED_TOKEN_ACCOUNT_PROGRAM_ID
  );

  return result;
}

async function main() {
  // const result = await findAssociatedTokenAddress(
  //     YOUR_DEFAULT_SOL_ADDRESS,
  //     TOKEN_ADDRESS
  // );
  // console.log(result);

  const result = findAssociatedTokenAddressSync(
    YOUR_DEFAULT_SOL_ADDRESS,
    TOKEN_ADDRESS
  );
  console.log(result);
}

main();
```

Run `get-ata.ts` to generate ATA.

```bash
npx ts-node get-ata.ts -a FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH -t J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm
[
  PublicKey [PublicKey(AB7baqsAgkfjwQtekTuvzT5amuM1xrRSUxRpqdnkWngJ)] {
    _bn: <BN: 885072e7e485040c628343c0a78ce7297c66741518186e76b894ae7ede4f4993>
  },
  253
]
```

Notice the seeds for the ATA(PDA) is:

- Wallet address: `FCxBXdduz9HqTEPvEBSuFLLAjbVYh9a5ZgEnZwKyN2ZH`
- TOKEN PROGRAM ID: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`
- Token mint address: `J4qKh7nmJ5a9xWY13jdyEjdYo7HoSWU5H1pRanyu3dJm`

And as we're generating PDA for ATA Program, whose program id is `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL`.

That's why we call `findProgramAddressSync` like this:

```typescript
const result = PublicKey.findProgramAddressSync(
  [
    // wallet address, your default Solana address to transform to Associated token address(ATA)
    new PublicKey(address).toBuffer(),
    // address of SPL token program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
    TOKEN_PROGRAM_ID.toBuffer(),
    // token mint address
    new PublicKey(tokenAddress).toBuffer(),
  ],
  // address of SPL Associated Token Account program: ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
  SPL_ASSOCIATED_TOKEN_ACCOUNT_PROGRAM_ID
);
```

Also notice beside pass `TOKEN_PROGRAM_ID` to `findProgramAddressSync`, we can also pass `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` to create a token 2022 associated token account. This address is also exported in `constant.ts` from `spl-token`.

```typescript
import { PublicKey } from "@solana/web3.js";

/** Address of the SPL Token program */
export const TOKEN_PROGRAM_ID = new PublicKey(
  "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"
);

/** Address of the SPL Token 2022 program */
export const TOKEN_2022_PROGRAM_ID = new PublicKey(
  "TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb"
);

/** Address of the SPL Associated Token Account program */
export const ASSOCIATED_TOKEN_PROGRAM_ID = new PublicKey(
  "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
);

/** Address of the special mint for wrapped native SOL in spl-token */
export const NATIVE_MINT = new PublicKey(
  "So11111111111111111111111111111111111111112"
);

/** Address of the special mint for wrapped native SOL in spl-token-2022 */
export const NATIVE_MINT_2022 = new PublicKey(
  "9pan9bMn5HatX4EJdBwg9VgCa7Uz5HL8N1m5D3NdXejP"
);

/** Check that the token program provided is not `Tokenkeg...`, useful when using extensions */
export function programSupportsExtensions(programId: PublicKey): boolean {
  if (programId.equals(TOKEN_PROGRAM_ID)) {
    return false;
  } else {
    return true;
  }
}
```

# Refs

https://solana.com/docs/core/tokens#create-token-metadata
