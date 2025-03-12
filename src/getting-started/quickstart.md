# Quickstart

Steel offers a modular approach, a library of helper functions, and macros for Solana program development. In this tutorial, you will explore Steel's boilerplate program, a counter program.

### Prerequisites

Before you begin, ensure you have the following tools installed in your development environment:

- **Rust**: [Install Rust](installation.html#install-rust).
- **Steel CLI**: [Install Steel CLI](installation.html#install-steel-cli).
- **Solana CLI**: [Install Solana CLI](installation.html#install-the-solana-cli).

Each link will guide you through the installation process. Once you have these tools installed, you're ready to dive in!

## Scaffold boilerplate Steel project

Create a new Steel project by running the command below:

```bash
steel new hello-steel
```

Running the command above creates a `hello-steel` directory with the file tree below:

```bash
.
├── Cargo.lock
├── Cargo.toml
├── README.md
├── api
│   ├── Cargo.toml
│   └── src
│       ├── consts.rs
│       ├── error.rs
│       ├── instruction.rs
│       ├── lib.rs
│       ├── sdk.rs
│       └── state
│           ├── counter.rs
│           └── mod.rs
├── program
│   ├── Cargo.toml
│   ├── src
│   │   ├── add.rs
│   │   ├── initialize.rs
│   │   └── lib.rs
│   └── tests
│       └── test.rs
└── target
```

The file tree contains the following folders:

- **api**: This folder contains types, program states & constants available to the rest of your Solana program.
- **program**: The program folder contains the business logic for your program's instructions.

### Program overview

The generated program maintains a single counter value that can be initialized to zero and incremented by adding values. You can find main program logic is in `program/src/lib.rs`, which handles two instructions: `Initialize` and `Add`.

`Initialize` creates a new counter account initialized to 0:

```rust
//program/src/initialize
use hello_steel_api::prelude::*;
use steel::*;

pub fn process_initialize(accounts: &[AccountInfo<'_>], _data: &[u8]) -> ProgramResult {
    // Load accounts.
    let [signer_info, counter_info, system_program] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    // Validate accounts.
    signer_info.is_signer()?;
    counter_info.is_empty()?.is_writable()?.has_seeds(
        &[COUNTER],
        &hello_steel_api::ID
    )?;
    system_program.is_program(&system_program::ID)?;

    // Initialize counter.
    create_program_account::<Counter>(
        counter_info,
        system_program,
        signer_info,
        &hello_steel_api::ID,
        &[COUNTER],
    )?;

    // Deserialize counter_info and set its `value` field to 0.
    let counter = counter_info.as_account_mut::<Counter>(&hello_steel_api::ID)?;
    counter.value = 0;

    Ok(())
}

```

Steel provides chainable helper methods for account validation:

```rust
// Validate accounts.
signer_info.is_signer()?;
counter_info.is_empty()?.is_writable()?.has_seeds(
    &[COUNTER],
    &hello_steel_api::ID
)?;
system_program.is_program(&system_program::ID)?;
```

Steel simplifies common cross-program invocations with helper functions:

```rust
// Initialize counter.
create_program_account::<Counter>(
    counter_info,
    system_program,
    signer_info,
    &hello_steel_api::ID,
    &[COUNTER],
)?;
```

These methods check account properties (signing authority, writability, PDA seeds, etc.) and return errors if validation fails, creating clean, readable security checks.

`Add` adds a specified amount to the counter, with checks to ensure it stays below 100.

```rust
//program/src/add
use hello_steel_api::prelude::*;
use steel::*;

pub fn process_add(accounts: &[AccountInfo<'_>], data: &[u8]) -> ProgramResult {
    // Parse args.
    let args = Add::try_from_bytes(data)?;
    let amount = u64::from_le_bytes(args.amount);

    // Load accounts.
    let [signer_info, counter_info] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    // Validate accounts.
    signer_info.is_signer()?;
    let counter = counter_info
		.as_account_mut::<Counter>(&hello_steel_api::ID)?
		.assert_mut(|c| c.value < 100)?;

    // Update state
    counter.value += amount;

    Ok(())
}

```

The program tracks the counter's state with a `Counter` struct:

```rust
//api/src/state
#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq, Pod, Zeroable)]
pub struct Counter {
    pub value: u64
}
```

The counter is stored in a PDA (Program Derived Address).

## Running tests

The project includes integration tests that:

- Initializes and verifies the counter's initialization
- Add values to the counter and verifies the addition

You can run these tests by executing the command below:

```bash
steel test
```

## Building and deploying a Steel project

You can build a Steel program by running:

```bash
steel build
```

This command generates a `deploy` folder in your `target` directory.

To deploy the Steel program, run:

```bash
solana program deploy target/deploy/increment_program.so
```
