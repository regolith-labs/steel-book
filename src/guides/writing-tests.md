# Writing Tests in Steel
Each test in Steel runs inside a fresh, in-memory blockchain using `solana-program-test`. This simulates real-world execution conditions without requiring an actual Solana validator. Learn how to write tests in Steel.

## **Setting Up the Test Environment**

Follow the steps below to set up a Steel testing environment.

### **Adding Dependencies**

To write and run tests, add the following dependencies under `[dev-dependencies]` in `Cargo.toml`:

```toml
[dev-dependencies]
solana-program-test = "1.x"
solana-sdk = "1.x"
tokio = { version = "1.x", features = ["macros", "rt-multi-thread"] }
```

The dependencies above include:

- `solana-program-test` provides the test framework.
- `solana-sdk` includes utilities for creating transactions.
- `tokio` allows running asynchronous tests.

### **Creating a Test Function**

Tests in Steel follow the standard Rust testing framework but use the `#[tokio::test]` macro for asynchronous execution:

```rust
#[tokio::test]
async fn test_example() {
    // Test logic will go here.
}
```

Each test function runs in isolation, ensuring the state does not persist between test cases.

### **Initializing `ProgramTest`**

Before executing transactions, a test environment must be set up using `ProgramTest::new()`:

```rust
use solana_program_test::*;
use solana_sdk::{pubkey::Pubkey, signature::Keypair, transaction::Transaction};

#[tokio::test]
async fn test_initialize() {
    let mut program_test = ProgramTest::new(
        "my_program",                    // The BPF program name
        Pubkey::new_unique(),            // Program ID
        processor!(process_instruction), // Entrypoint function
    );

    let (mut banks_client, payer, recent_blockhash) = program_test.start().await;
}
```

In the code block above:

- `ProgramTest::new()` registers the program and its entry point function.
- `start().await` initializes the test environment and returns:
    - `banks_client`: A simulated Solana bank for processing transactions.
    - `payer`: A keypair preloaded with SOL to cover transaction fees.
    - `recent_blockhash`: Required for signing transactions.

Each test starts with an empty ledger, ensuring that previous tests do not affect new ones.

## Creating Test Transactions

Steel requires manual transaction creation to ensure full control over execution. This involves creating your instructions, bundling them into a single transaction, and signing and submitting the transaction.

### **Creating Instructions**

Each test instruction specifies:

- **The program ID** that will execute it.
- **A list of accounts** involved in the transaction.
- **The instruction data** (parameters for execution).

For example:

```rust
use solana_sdk::{
    instruction::{Instruction, AccountMeta},
    signer::Signer,
};

fn create_instruction(program_id: Pubkey, account: Pubkey) -> Instruction {
    Instruction::new_with_bincode(
        program_id,
        &0u8, // Example instruction data
        vec![AccountMeta::new(account, false)], // Account to modify
    )
}
```

This function above creates an instruction with a **single account**. It uses `bincode` serialization for instruction data and marks the account as writable (not read-only).

### **Signing a Transaction**

Transactions bundle multiple instructions into a single atomic operation:

```rust
let instruction = create_instruction(program_id, some_account);
let mut transaction = Transaction::new_signed_with_payer(
    &[instruction],
    Some(&payer.pubkey()),
    &[&payer],
    recent_blockhash,
);
```

In the code block above `Transaction::new_signed_with_payer()` takes:

- A slice of instructions.
- The fee payer (payer's public key).
- A list of required signers.
- A recent blockhash for signing validity.

### **Submitting Transactions**

Once the transaction is signed, submit it to the test ledger:

```rust
banks_client.process_transaction(transaction).await.unwrap();
```

If an error occurs, the test will fail, highlighting the issue in execution.

## **Verifying State Changes**

After submitting a transaction, the test must confirm that the program executed correctly.

### **Fetching Account Data**

Retrieve account data from the test ledger:

```rust
let account_data = banks_client.get_account(some_account).await.unwrap().unwrap();
```

`get_account(pubkey).await` returns an `Option<Account>`. If the account exists, its data can be inspected.

### **Deserializing Account Data**

Programs store data as raw bytes, so deserialization is required:

```rust
use borsh::BorshDeserialize;

#[derive(BorshDeserialize)]
struct ExampleAccount {
    value: u64,
}

let account_struct = ExampleAccount::try_from_slice(&account_data.data).unwrap();

assert_eq!(account_struct.value, 42);
```

The code block above checks that the stored data was properly written and the expected state change occurred.

### **Handling Errors and Edge Cases**

Steel allows detailed error handling by catching failed transactions:

```rust
let result = banks_client.process_transaction(transaction).await;
assert!(result.is_err()); // Ensure failure if the test expects one
```

This is useful for testing unauthorized transactions, missing or incorrect instruction data, and invalid account permissions.