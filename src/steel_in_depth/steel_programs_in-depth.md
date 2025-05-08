# Steel Programs In-Depth

Steel encourages separating your program’s interface (accounts, instructions, errors, events) from its implementation. For example, you might have an **`api`** module or crate defining all account types and instruction formats, and a **`program`** module/crate implementing the instruction handlers and the entrypoint. This division improves readability and scales well as your project grows in complexity.


## Discriminators

**Discriminators** are small identifiers used in Solana programs to distinguish different data types. For example, when you store data in an account or send an instruction, a discriminator tells the program which account structure or instruction variant it represents. Steel streamlines the use of discriminators by letting you define a single enum that lists all your types, rather than manually managing magic numbers. Each variant in the enum gets a unique value, and Steel’s macros will embed these values into account data or instruction data for you, so the program can interpret the bytes correctly.

```rust
use steel::*;

/// Enum for account discriminators – each variant is a unique account type ID.
#[repr(u8)]
#[derive(Clone, Copy, Debug, Eq, PartialEq, IntoPrimitive, TryFromPrimitive)]
pub enum MyAccount {
    Counter = 0,
    Profile = 1,
}

/// Enum for instruction discriminators – each variant identifies an instruction.
#[repr(u8)]
#[derive(Clone, Copy, Debug, Eq, PartialEq, TryFromPrimitive)]
pub enum MyInstruction {
    Initialize = 0,
    Add        = 1,
}
```

*In the code above, `MyAccount` and `MyInstruction` are enums whose variants (like `Counter` or `Add`) act as discriminators. By using `#[repr(u8)]`, we specify that each variant is stored as a single byte. Traits like `IntoPrimitive`/`TryFromPrimitive` make it easy to convert between the enum and its underlying byte value.* When we use Steel macros (as shown later), these enums ensure that each account or instruction type is tagged with the correct ID in its data.

## Entrypoint

In Solana, the entrypoint is the function that the runtime calls to start executing your program. This function must deserialize incoming instruction data and dispatch it to the appropriate handler function. Steel's `entrypoint!` macro streamlines this process by automatically registering your function as the program's entrypoint and reducing boilerplate. You simply implement a `process_instruction` function with the standard Solana signature, and then use `entrypoint!(your_function_name)`. Inside `process_instruction`, you can leverage Steel’s `parse_instruction` helper to decode the raw byte input into your `MyInstruction` enum, making it easy to match against the instruction variants and call the correct logic.

```rust
use steel::*;
use solana_program::{AccountInfo, Pubkey, ProgramResult, ProgramError};

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    data: &[u8],
) -> ProgramResult {
    // Decode the incoming instruction data into our MyInstruction enum (and verify program ID)
    let (instruction, remaining_data) = parse_instruction::<MyInstruction>(&example_api::ID, program_id, data)?;
    // Dispatch to the appropriate handler based on the instruction variant
    match instruction {
        MyInstruction::Initialize => process_initialize(accounts, remaining_data)?,  
        MyInstruction::Add        => process_add(accounts, remaining_data)?,  
    }
    Ok(())
}

// Declare the above function as the program's entrypoint.
entrypoint!(process_instruction);
```

In this example, `parse_instruction::<MyInstruction>` examines the `data` bytes. It uses the first byte (the discriminator) to determine which variant of `MyInstruction` was invoked and returns that variant along with the rest of the data. We then `match` on the instruction enum to call the corresponding function (e.g. `process_add`). The `entrypoint!` macro at the bottom hooks our `process_instruction` into Solana's runtime – this macro expands to the necessary code to set up the entrypoint so that the Solana runtime knows to call it when our program is invoked.

## Errors

Solana programs use `ProgramError` to report failures, and Steel makes it easy to define **custom errors** for your program. Instead of relying on generic error codes, you can create an enum of meaningful error variants with messages, and use Steel’s `error!` macro to integrate them. Each error variant gets a unique numeric code (e.g. an 32-bit integer) and an associated message for clarity. The `error!` macro generates the necessary conversions so that you can use `MyError::Variant` in your code and have it treated as a `ProgramError` when returned from instructions.

```rust
use steel::*;
use thiserror::Error;  // for deriving the Error trait

/// Enum for custom error types (each variant has a message and code).
#[repr(u32)]
#[derive(Debug, Error, Clone, Copy, PartialEq, Eq, IntoPrimitive)]
pub enum MyError {
    #[error("You did something wrong")]
    Dummy = 0,
}

// Register the error type with Steel (allows use of MyError as ProgramError).
error!(MyError);
```

In the code above, we define a `MyError` enum with a variant `Dummy` (code 0) and a human-readable message. The `#[derive(Error)]` attribute (from the `thiserror` crate) lets us easily attach messages to variants. By calling `error!(MyError);`, Steel will implement the boilerplate to use this in the program – for example, you can do `return Err(MyError::Dummy.into())` in your instruction handler, and it will convert into a `ProgramError` with your custom code and message. Custom errors make debugging easier, as you and your users can see exactly what went wrong (in this case, a "You did something wrong" message instead of a generic error).

## Events

Solana doesn't have native "events" like Ethereum, but programs can emit log messages that off-chain clients can monitor. Steel provides an `event!` macro to help you define and emit **structured events** via the Solana logging system. An event in Steel is simply a Rust struct that represents the data you want to log. Using the macro will prepare this struct for logging (by implementing the required traits and possibly an emitter function), so you can easily output it to the program log.

```rust
use steel::*;
use bytemuck::{Pod, Zeroable};  // for Plain Old Data struct support

/// Struct representing an event to log.
#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq, Pod, Zeroable)]
pub struct MyEvent {
    pub value: u64,
}

// Register the event type so it can be emitted in logs.
event!(MyEvent);
```

Here we define `MyEvent` as a simple struct with some data (in this case just a `value`). The `event!(MyEvent)` macro tells Steel to generate the necessary logic to log this event. Under the hood, this uses Solana’s `sol_log_data` instruction to record the event in the transaction log. In practice, you would create an instance of `MyEvent` in your code and call an emit function (provided by the macro expansion) to log it. For example, `MyEvent { value: 42 }` could be logged, and an off-chain client reading the transaction logs would see the data and know an event occurred. Using events is a great way to notify clients (like front-end applications or indexers) about significant actions in your program (such as a value change or an important state transition).

## Validation

Before updating state or calling other programs, a Solana program should **validate** that all inputs (accounts and data) are correct. Steel provides a fluent, chainable API for account validation, which makes these checks more readable and less error-prone. Instead of writing lengthy `if` statements and manual deserialization, you can chain helper methods that confirm properties of accounts (e.g., is signer, is writable, has expected owner, etc.) and convert account data into your Rust structs.

A typical validation flow might look like this in principle:

1. **Check account presence:** Ensure the expected number of accounts are provided (and in the correct order).
2. **Verify signers:** Confirm that any account that is supposed to sign the transaction actually did (`is_signer`).
3. **Deserialize and check ownership:** Convert the account’s data to your program’s struct (e.g. `Counter`) and make sure the account is owned by your program (Steel does this when you call `to_account`/`to_account_mut` with your program’s ID).
4. **Assert business logic conditions:** Verify that the account’s state meets any preconditions for the instruction (for example, a counter value is within an allowed range).

Steel’s validators return `ProgramResult` errors automatically if a check fails, so you can `?` chain them. This leads to concise error-handling without losing clarity. Below is an example of validating two accounts for an `Add` instruction:

```rust
use steel::*;
use solana_program::program_error::ProgramError;

pub fn process_add(accounts: &[AccountInfo<'_>], _data: &[u8]) -> ProgramResult {
    // Extract the expected accounts: a signer and a counter account.
    let [signer_info, counter_info] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    signer_info.is_signer()?;  // The first account must be a signer (transaction signed by this key)

    // Parse and validate the counter account data.
    let counter = counter_info
        .to_account_mut::<Counter>(&example_api::ID)?   // Get a mutable Counter struct (checks ownership & correct discriminator)
        .check_mut(|c| c.value <= 42)?;                 // Ensure the current counter value <= 42 (precondition)

    // At this point, validation passed – it's safe to modify the account.
    counter.value += 1;  // Increment the counter's value

    Ok(())
}
```

In this code, we expect exactly two accounts for the `Add` instruction: one signer (`signer_info`) and one counter state account (`counter_info`). We use Rust’s slice pattern matching (`let [a, b] = accounts else { ... };`) to both extract and validate the number of accounts. Next, `signer_info.is_signer()?` checks that the first account provided has indeed signed the transaction – if not, the program returns an error immediately.

We then parse `counter_info` into our `Counter` struct using `to_account_mut::<Counter>`. This call does multiple things at once: it verifies that `counter_info` is owned by our program (using the provided program ID, here `example_api::ID`), that the account’s data length matches a `Counter` struct, and it gives us a mutable reference to a `Counter` loaded from the account’s data. Immediately after, we chain `.check_mut(|c| c.value <= 42)?` to assert a condition on the data – in this case, that the counter’s value is not greater than 42. If this condition fails, an error is returned (with a default or custom error, e.g. `ProgramError::Custom` if configured with our `MyError`).

By the time we get to modifying `counter.value`, we know that `counter_info` was valid and met all our requirements. We can safely update the account data. This fluent validation pattern makes the code easy to read: each step is clear, and any failure along the way aborts the execution with an appropriate error automatically.

## Accounts

In Solana, **accounts** are the on-chain storage pieces that hold data for your program. Defining account data structures (and ensuring their data is correctly interpreted) can involve a lot of boilerplate. Steel addresses this with the `account!` macro, which ties a Rust struct to an enum discriminator and implements serialization for you. To define an account type in Steel, you follow three steps:

* **Define an enum** that lists all your program’s account types (as we did with `MyAccount` earlier, where each variant corresponds to a distinct account type).
* **Define a struct** for each account type to represent the data you want to store in that account.
* **Use the `account!` macro** to link each struct with the enum. This macro ensures the account’s data begins with the correct discriminator (so the program can identify its type) and provides functions to serialize/deserialize that struct to the account’s byte array.

Let's say our program has two account types: a `Counter` (to store a numeric value) and a `Profile` (to store a user profile id). We would set them up as follows:

```rust
use steel::*;
use bytemuck::{Pod, Zeroable};

/// Enum listing all account types for the program (each variant is a discriminator).
#[repr(u8)]
#[derive(Clone, Copy, Debug, Eq, PartialEq, IntoPrimitive, TryFromPrimitive)]
pub enum MyAccount {
    Counter = 0,
    Profile = 1,
}

/// Data structure stored in a "Counter" account.
#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq, Pod, Zeroable)]
pub struct Counter {
    pub value: u64,
}

/// Data structure stored in a "Profile" account.
#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq, Pod, Zeroable)]
pub struct Profile {
    pub id: u64,
}

// Link each account struct with the enum discriminator and implement serialization logic.
account!(MyAccount, Counter);
account!(MyAccount, Profile);
```

First, `MyAccount` is our enum of account discriminators (here we assign 0 to `Counter` and 1 to `Profile`). Next, we define the structs `Counter` and `Profile` to represent the actual data that will live in those accounts. We've annotated them with `#[repr(C)]` to ensure a stable memory layout and derived `Pod` and `Zeroable` traits (from the bytemuck crate) which mark these structs as plain data that can be safely converted to/from bytes. Finally, we invoke `account!(MyAccount, Counter);` and similarly for `Profile`.

The `account!` macro generates code to glue this all together. It will, for example, write the appropriate discriminator byte into an account when you initialize it, and provide a method to deserialize an account’s data into the `Counter` or `Profile` struct when you call functions like `to_account::<Counter>()`. Essentially, you no longer need to manually write code to pack/unpack these structs or check their discriminator — Steel handles it, reducing the chance of errors and making your account management code much cleaner.

## Instructions

A Solana **instruction** represents a function call or action that a user invokes on your program, often with some input data. In Steel, you define your program’s instructions in a way very similar to accounts: using an enum and some structs, then a macro to tie them together. Each variant of your instruction enum gets a unique discriminator (to identify the instruction), and each instruction that carries data has a corresponding struct defining its fields. The `instruction!` macro connects the enum and structs, ensuring that when an instruction is serialized to bytes, the first byte is the discriminator and the following bytes are the fields of the struct.

Here's how we can define two instructions, `Initialize` (which in this example has no additional data) and `Add` (which carries a `value`):

```rust
use steel::*;
use bytemuck::{Pod, Zeroable};

/// Enum of instruction discriminators for the program.
#[repr(u8)]
#[derive(Clone, Copy, Debug, Eq, PartialEq, TryFromPrimitive)]
pub enum MyInstruction {
    Initialize = 0,
    Add        = 1,
}

/// Data payload for the "Initialize" instruction (empty in this case).
#[repr(C)]
#[derive(Clone, Copy, Debug, Pod, Zeroable)]
pub struct Initialize {}

/// Data payload for the "Add" instruction.
#[repr(C)]
#[derive(Clone, Copy, Debug, Pod, Zeroable)]
pub struct Add {
    pub value: u64,
}

// Link each instruction struct with the enum variant and implement serialization.
instruction!(MyInstruction, Initialize);
instruction!(MyInstruction, Add);
```

We start by defining `MyInstruction` enum with two variants. Much like account discriminators, these are `u8` values (`Initialize = 0`, `Add = 1`) that tag which instruction is being called. We then define a struct for each instruction's arguments: `Initialize` has no fields, and `Add` has a single `u64` field named `value`. After that, the calls to `instruction!(MyInstruction, Initialize);` and `instruction!(MyInstruction, Add);` will generate the necessary code to serialize and deserialize these instructions.

What does that mean in practice? When you create a transaction to call `Add`, Steel will expect the instruction data to start with the byte `1` (the discriminator for `Add`), followed by 8 bytes representing the `value`. Conversely, in your program’s entrypoint, `parse_instruction::<MyInstruction>` (as shown earlier) will examine the first byte of incoming data to figure out which variant it is (0 or 1 in our case) and then parse the rest into either an `Initialize` or `Add` struct. This approach provides a clear structure to your program’s interface: by reading the enum and structs, you know exactly what instructions exist and what data they require. It’s much clearer than dealing with raw byte offsets and manual deserialization.

*(In our entrypoint example above, you saw how `MyInstruction` was used in a `match` to dispatch to `process_initialize` or `process_add`. That dispatch is made possible by this instruction definition setup.)*

## CPIs (Cross-Program Invocations)

Often, your Solana program will need to call into other programs – for example, to transfer SPL tokens or to create a new account using the system program. These are called **Cross-Program Invocations (CPIs)**. CPIs are powerful but can be verbose to implement, as they require constructing instruction data and invoking the external program with the right accounts. Steel provides a set of helper functions to make common CPIs straightforward, so you don't have to manually craft every cross-program call.

Steel’s CPI helpers cover tasks like creating accounts (via the system program), initializing token accounts, minting or transferring tokens, and more. For instance, instead of writing out the entire instruction to call the SPL Token program’s transfer, you can simply call Steel’s `transfer` function. Under the hood, this will perform `invoke` on the token program with the correct accounts and data. These helpers ensure you pass the right accounts and follow best practices, reducing errors in CPI calls.

Let’s look at an example where our program invokes the SPL Token program to transfer tokens from one account to another. We will assume the following accounts are passed in: the user (signer) who authorizes the transfer, a `Counter` account from our program that we might use to gate the transfer, the Token Mint, the sender's token account, the receiver's token account, and the token program itself. We’ll validate all these accounts and then perform the transfer:

```rust
use steel::*;
use solana_program::{account_info::AccountInfo, program_error::ProgramError};
use spl_token::ID as SPL_TOKEN_ID;

pub fn process_transfer(accounts: &[AccountInfo<'_>], _data: &[u8]) -> ProgramResult {
    let [signer_info, counter_info, mint_info, sender_info, receiver_info, token_program] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    // Validate the signer and our program's counter account condition.
    signer_info.is_signer()?;                     // The first account must have signed the transaction
    let counter = counter_info
        .to_account::<Counter>(&example_api::ID)? 
        .check(|c| c.value >= 42)?;               // Require that counter.value >= 42 as a precondition

    // Validate the token mint and token accounts.
    mint_info.to_mint()?;                        // Check that this account is a valid SPL Token Mint
    sender_info
        .is_writable()?                         
        .to_token_account()?                     
        .check(|t| t.owner == *signer_info.key)? 
        .check(|t| t.mint == *mint_info.key)?;    // Sender's token account: must be owned by signer and of the correct mint
    receiver_info
        .is_writable()? 
        .to_token_account()? 
        .check(|t| t.mint == *mint_info.key)?;    // Receiver's token account: must be for the same mint
    token_program.is_program(&SPL_TOKEN_ID)?;     // Ensure the correct token program is provided

    // Invoke the SPL Token program's Transfer instruction via Steel's helper.
    transfer(
        signer_info,    // authority (the signer who approves the transfer)
        sender_info,    // source token account 
        receiver_info,  // destination token account 
        token_program,  // the token program account
        counter.value,  // amount of tokens to transfer (we use the counter's value here)
    )?;
    Ok(())
}
```

In this `process_transfer` handler, we first destructure the `accounts` slice to get the six accounts we expect. We then perform a series of validations similar to earlier examples:

* Check that `signer_info` is a signer (so the user has authorized this operation).
* Convert `counter_info` to a `Counter` struct and ensure `counter.value >= 42`. (In our imagined scenario, perhaps the user must have incremented the counter to 42 or more before they are allowed to transfer tokens – this is just to illustrate using your program’s state as a gate for a CPI.) If this check fails, the function returns an error and the transfer will not proceed.
* Use `to_mint()` on `mint_info` to verify that it’s indeed an SPL Token Mint account. This ensures the bytes in `mint_info` follow the expected format of a Mint.
* Check the sender's token account: we ensure it’s marked writable (`is_writable()`), that it’s actually an SPL Token Account (`to_token_account()` parses its data as a token account structure), and then verify the token account’s owner is our signer and its mint matches the provided mint. These conditions guarantee that the signer owns the tokens we’re about to send, and that those tokens are of the correct type.
* Similarly, check the receiver's token account (writable, a token account, and holding the same mint).
* Finally, confirm that `token_program` is indeed the SPL Token program (by comparing its Pubkey to the known token program ID).

After all the checks, we call `transfer(...)`. This is a Steel-provided function (re-exported from the SPL Token helper module) that performs the token transfer CPI. We pass in the signer (as the authority), the sender and receiver token accounts, the token program account, and the amount to transfer (`counter.value` in this case). Steel’s `transfer` will construct the proper SPL Token Program instruction and call `invoke_signed` or `invoke` for you. Our program doesn’t need to worry about the details of the token program’s instruction format — we just provide the accounts and amount.

By using Steel’s CPI helpers, our code stays concise and focused on the high-level logic (like ensuring the user is allowed to transfer). The framework abstracts the low-level boilerplate of cross-program calls. If you needed to perform other common actions, Steel has similar helpers (for example, creating a new token account might be a single function call rather than many lines of instruction building).
