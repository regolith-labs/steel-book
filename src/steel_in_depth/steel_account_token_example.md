# Steel Project - Token Example

> [Program Code](https://github.com/Perelyn-sama/building-with-steel/tree/master/token)

> [Video Tutorial](https://www.youtube.com/watch?v=_TMPF79CgTg&t=39s)

In this tutorial, we’ll build a minimal token creation and minting program using [Steel](https://github.com/Perelyn-sama/building-with-steel). This includes creating a custom SPL-compatible token and minting it to a user's associated token account.

---

## 🧱 Setting Up the Workspace

Start by creating a new Steel project:

```sh
steel new counter-example
```

---

## 📆 Add Dependencies

In the root-level `Cargo.toml`, add the following dependencies. These are necessary for working with SPL tokens and metadata:

```toml
[dependencies]
spl-token = { version = "8.0.0", features = ["no-entrypoint"] }
spl-associated-token-account = { version = "6.0.0", features = ["no-entrypoint"] }
mpl-token-metadata = { version = "1.7.0" }
```

These are the core crates needed to work with token minting and metadata. The `no-entrypoint` feature ensures that we can safely use these as dependencies without conflict.

---

## 🔧 Configure `api/Cargo.toml`

Inside the `api` crate's `Cargo.toml`, ensure the workspace dependencies are declared so they're shared from the root workspace:

```toml
mpl-token-metadata.workspace = true
spl-token.workspace = true
spl-associated-token-account.workspace = true
```

---

## 🪤 Clean Up Unused Files

Since this project is focused solely on token creation and minting, remove the following boilerplate files from `api/src/`:

* `state.rs`
* `errors.rs`
* `consts.rs`

They are unnecessary for this simplified token program.

---

## 📄 API Instructions

### `api/src/instruction.rs`

Here we define the actual instructions that can be executed by our program. These include:

* `Create`: Initializes a new token mint and metadata
* `Mint`: Mints tokens to a given address

```rust
use steel::*;

#[repr(u8)]
#[derive(Clone, Copy, Debug, Eq, PartialEq, TryFromPrimitive)]
pub enum TokenInstruction {
    Create = 0, // Create a new token mint and metadata
    Mint = 1,   // Mint tokens to an account
}

// Data structure passed with the Create instruction
#[repr(C)]
#[derive(Clone, Copy, Debug, Pod, Zeroable)]
pub struct Create {
    pub name: [u8; 32],    // Token name (e.g. "Bitcoin")
    pub symbol: [u8; 8],   // Token symbol (e.g. "BTC")
    pub uri: [u8; 128],    // Metadata URI
    pub decimals: u8,      // Token precision
}

// Data structure passed with the Mint instruction
#[repr(C)]
#[derive(Clone, Copy, Debug, Pod, Zeroable)]
pub struct Mint {
    pub amount: [u8; 8], // Amount to mint (little-endian u64)
}

// Generate instruction decoding logic
instruction!(TokenInstruction, Create);
instruction!(TokenInstruction, Mint);
```

---

### `api/src/sdk.rs`

This module contains utility functions that return the necessary instructions to invoke from a client or test. You should remove `initialize` and `add` functions from the bpilerplate code.

The `AccountMeta::new` function allows the declartion of an account, it acceps both the account and a boolean indicating whether it's a signer or not.
We use `AccountMeta::new_readonly` for read-only system accounts.

```rust
use spl_associated_token_account::get_associated_token_address;
use steel::*;
use crate::prelude::*;

// Constructs the instruction to create a token mint and its metadata
pub fn create(
    user: Pubkey,           // The user creating the mint
    mint: Pubkey,           // The mint account to be created
    name: [u8; 32],         // Token name
    symbol: [u8; 8],        // Token symbol
    uri: [u8; 128],         // Metadata URI
    decimals: u8,           // Number of decimals
) -> Instruction {
    // Derive the metadata account address using a PDA
    let metadata = Pubkey::find_program_address(
        &[
            "metadata".as_bytes(),
            mpl_token_metadata::ID.as_ref(),
            mint.as_ref(),
        ],
        &mpl_token_metadata::ID,
    )
    .0;

    // Construct the Instruction
    Instruction {
        program_id: crate::ID,
        accounts: vec![
            AccountMeta::new(user, true),                     // payer and authority
            AccountMeta::new(mint, true),                     // mint account
            AccountMeta::new(metadata, false),                // derived metadata
            AccountMeta::new_readonly(spl_token::ID, false),
            AccountMeta::new_readonly(mpl_token_metadata::ID, false),
            AccountMeta::new_readonly(system_program::ID, false),
            AccountMeta::new_readonly(sysvar::rent::ID, false),
        ],
        data: Create {
            name,
            symbol,
            uri,
            decimals,
        }
        .to_bytes(),
    }
}

// Constructs the instruction to mint tokens to the recipient's associated token account
pub fn mint(
    mint_authority: Pubkey, // signer who owns the mint
    recipient: Pubkey,      // user receiving tokens
    mint: Pubkey,           // token mint
    amount: u64,            // amount to mint
) -> Instruction {
    let recipient_ata = get_associated_token_address(&recipient, &mint);

    Instruction {
        program_id: crate::ID,
        accounts: vec![
            AccountMeta::new(mint_authority, true),
            AccountMeta::new(recipient, false),
            AccountMeta::new(recipient_ata, false),
            AccountMeta::new(mint, false),
            AccountMeta::new_readonly(spl_token::ID, false),
            AccountMeta::new_readonly(spl_associated_token_account::ID, false),
            AccountMeta::new_readonly(system_program::ID, false),
        ],
        data: Mint {
            amount: amount.to_le_bytes(),
        }
        .to_bytes(),
    }
}
```

This setup ensures your frontend or test scripts can programmatically generate the correct instructions to interact with your program.

### `api/src/lib.rs`

Clean up the imports and expose the instruction and SDK modules:

```rust
pub mod instruction;
pub mod sdk;

pub mod prelude {
    pub use crate::instruction::*;
    pub use crate::sdk::*;
}

use steel::*;

// TODO Set program id
declare_id!("z7msBPQHDJjTvdQRoEcKyENgXDhSRYeHieN1ZMTqo35");
```

## 🧠 Program Logic

Delete `program/src/add.rs` and `program/src/initialize.rs`

### `program/src/create.rs`

This handler sets up the mint account and metadata. Note that args are unpacked into the same sizes decleared in `instruction.rs`.

When initializing the mint we use `Some(user_info.key)` as the freeze authority, it could also be None.

```rust
use solana_program::msg;
use solana_program::program_pack::Pack;
use steel::*;
use token_api::prelude::*;

pub fn process_create(accounts: &[AccountInfo<'_>], data: &[u8]) -> ProgramResult {
    let [user_info, mint_info, metadata_info, token_program, token_metadata_program, system_program, rent_sysvar] =
        accounts
    else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    // validate
    user_info.is_signer()?;
    mint_info.is_empty()?.is_signer()?;
    metadata_info.is_empty()?.is_writable()?;
    token_program.is_program(&spl_token::ID)?;
    token_metadata_program.is_program(&mpl_token_metadata::ID)?;
    system_program.is_program(&system_program::ID)?;
    rent_sysvar.is_sysvar(&sysvar::rent::ID)?;

    // create mint account
    create_account(
        user_info,
        mint_info,
        system_program,
        spl_token::state::Mint::LEN,
        &token_program.key,
    )?;

    let args = Create::try_from_bytes(data)?;
    let name = bytes_to_string::<32>(&args.name)?;
    let symbol = bytes_to_string::<8>(&args.symbol)?;
    let uri = bytes_to_string::<128>(&args.uri)?;
    let decimals = args.decimals;

    let ix = spl_token::instruction::initialize_mint(
        token_program.key,
        mint_info.key,
        user_info.key,
        Some(user_info.key),
        decimals,
    )?;
    let account_infos = &[
        user_info.clone(),
        mint_info.clone(),
        token_program.clone(),
        rent_sysvar.clone(),
    ];
    solana_program::program::invoke(&ix, account_infos)?;

    mpl_token_metadata::instructions::CreateMetadataAccountV3Cpi {
        __program: token_metadata_program,
        metadata: metadata_info,
        mint: mint_info,
        mint_authority: user_info,
        payer: user_info,
        update_authority: (user_info, true),
        system_program,
        rent: Some(rent_sysvar),
        __args: mpl_token_metadata::instructions::CreateMetadataAccountV3InstructionArgs {
            data: mpl_token_metadata::types::DataV2 {
                name,
                symbol,
                uri,
                seller_fee_basis_points: 0,
                creators: None,
                collection: None,
                uses: None,
            },
            is_mutable: true,
            collection_details: None,
        },
    }
    .invoke()?;

    Ok(())
}
```

---

### `program/src/mint.rs`

This handler ensures the recipient has an ATA, and mints tokens into it.

```rust
use solana_program::msg;
use solana_program::program_pack::Pack;
use steel::*;
use token_api::prelude::*;

pub fn process_mint(accounts: &[AccountInfo<'_>], data: &[u8]) -> ProgramResult {
    let [mint_authority_info, recipient_info, recipient_ata_info, mint_info, token_program, ata_program, system_program] =
        accounts
    else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    mint_authority_info.is_signer()?;
    recipient_info.is_writable()?;
    recipient_ata_info.is_empty()?.is_writable()?;
    mint_info.as_mint()?;
    token_program.is_program(&spl_token::ID)?;
    ata_program.is_program(&spl_associated_token_account::ID)?;
    system_program.is_program(&system_program::ID)?;

    create_associated_token_account(
        mint_authority_info,
        recipient_info,
        recipient_ata_info,
        mint_info,
        system_program,
        token_program,
        ata_program,
    )?;

    let args = Mint::try_from_bytes(data)?;
    let amount = u64::from_le_bytes(args.amount);

    let instruction = spl_token::instruction::mint_to(
        token_program.key,
        mint_info.key,
        recipient_ata_info.key,
        mint_authority_info.key,
        &[&mint_authority_info.key],
        amount,
    )?;

    let account_infos = &[
        token_program.clone(),
        mint_info.clone(),
        mint_authority_info.clone(),
        recipient_ata_info.clone(),
    ];
    solana_program::program::invoke(&instruction, account_infos)?;

    Ok(())
}
```

---

### `program/src/lib.rs`

Wire up the instruction dispatcher and map TokenInstruction to the appropriate handler.

```rust
mod create;
mod mint;

pub use create::*;
pub use mint::*;

use steel::*;
use token_api::prelude::*;

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    data: &[u8],
) -> ProgramResult {
    let (ix, data) = parse_instruction(&token_api::ID, program_id, data)?;

    match ix {
        TokenInstruction::Create => process_create(accounts, data)?,
        TokenInstruction::Mint => process_mint(accounts, data)?,
    }

    Ok(())
}

entrypoint!(process_instruction);
```

---

### `program/tests/test.rs`

Add the below to the setup function to ensure the token_metadata program is loaded:

`program_test.add_program("token_metadata", mpl_token_metadata::ID, None);`

This integration test validates both creation and minting, Note that

```rust
use solana_program::hash::Hash;
use solana_program_test::{processor, BanksClient, ProgramTest};
use solana_sdk::{
    program_pack::Pack, signature::Keypair, signer::Signer, transaction::Transaction,
};
use steel::*;
use token_api::prelude::*;

async fn setup() -> (BanksClient, Keypair, Hash) {
    let mut program_test = ProgramTest::new(
        "token_program",
        token_api::ID,
        processor!(token_program::process_instruction),
    );

    program_test.add_program("token_metadata", mpl_token_metadata::ID, None);

    program_test.prefer_bpf(true);
    program_test.start().await
}

#[tokio::test]
async fn run_test() {
    // Setup test
    let (mut banks, payer, blockhash) = setup().await;
    let mint_keypair = Keypair::new();

    let name = string_to_bytes::<32>("ANATOLY").unwrap();
    let symbol = string_to_bytes::<8>("MERT").unwrap();
    let uri = string_to_bytes::<128>("blah blah blah").unwrap();
    let decimals = 9;

    // Submit create transaction.
    let ix = create(
        payer.pubkey(),
        mint_keypair.pubkey(),
        name,
        symbol,
        uri,
        decimals,
    );
    let tx = Transaction::new_signed_with_payer(
        &[ix],
        Some(&payer.pubkey()),
        &[&payer, &mint_keypair],
        blockhash,
    );
    let res = banks.process_transaction(tx).await;
    assert!(res.is_ok());

    let serialized_mint_data = banks
        .get_account(mint_keypair.pubkey())
        .await
        .unwrap()
        .unwrap()
        .data;

    let mint_data = spl_token::state::Mint::unpack(&serialized_mint_data).unwrap();
    assert!(mint_data.is_initialized);
    assert_eq!(mint_data.mint_authority.unwrap(), payer.pubkey());
    assert_eq!(mint_data.decimals, decimals);

    // Submit initialize transaction.
    let ix = mint(payer.pubkey(), payer.pubkey(), mint_keypair.pubkey(), 100);
    let tx = Transaction::new_signed_with_payer(&[ix], Some(&payer.pubkey()), &[&payer], blockhash);
    let res = banks.process_transaction(tx).await;
    assert!(res.is_ok());

    let ata = spl_associated_token_account::get_associated_token_address(
        &payer.pubkey(),
        &mint_keypair.pubkey(),
    );

    let serialized_ata_info = banks.get_account(ata).await.unwrap().unwrap().data;
    let ata_info = spl_token::state::Account::unpack(&serialized_ata_info).unwrap();
    assert_eq!(ata_info.amount, 100);

    dbg!(ata_info);
    assert!(false);
}
```

---

Let me know if you want a version of this with full inline comments or diagrams!
