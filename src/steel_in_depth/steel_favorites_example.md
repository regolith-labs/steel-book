# Steel Project – Favorites Example

> [Program Code](https://github.com/Perelyn-sama/building-with-steel/tree/master/favorites)

In this tutorial, we'll build a simple Solana program using [Steel](https://github.com/Perelyn-sama/building-with-steel) that allows users to store their "favorites"—a number, a color, and a list of hobbies—into a single on-chain account.

---

## 🧱 Setting Up the Workspace

Create a new project using Steel:

```sh
steel new favorites-example
```

---

## 🪤 Clean Up Boilerplate

From `api/src/`, delete:

* `add.rs`
* `initialize.rs`

From `program/src/`, delete:

* `add.rs`
* `initialize.rs`

---

## 📄 API Instructions

### `api/src/instruction.rs`

We define a single instruction—`SetFavorites`—that accepts a custom struct:

```rust
use crate::state::Favorites;
use steel::*;

#[repr(u8)]
#[derive(Clone, Copy, Debug, Eq, PartialEq, TryFromPrimitive)]
pub enum FavoritesInstruction {
    SetFavorites = 0,
}

#[repr(C)]
#[derive(Clone, Copy, Debug, Pod, Zeroable)]
pub struct SetFavorites {
    pub data: Favorites,
}

instruction!(FavoritesInstruction, SetFavorites);
```

**💡 Expectation**: `Favorites` must be Pod/Zeroable so it can be easily serialized into bytes for cross-program invocation and account storage.

---

### `api/src/sdk.rs`

Build the `Instruction` client-side:

```rust
use steel::*;
use crate::prelude::*;

pub fn set_favorites(signer: Pubkey, data: Favorites) -> Instruction {
    Instruction {
        program_id: crate::ID,
        accounts: vec![
            AccountMeta::new(signer, true),
            AccountMeta::new(favorites_pda().0, false),
            AccountMeta::new_readonly(system_program::ID, false),
        ],
        data: SetFavorites { data }.to_bytes(),
    }
}
```

**💡 Expectation**: You can pass complex structured data (like `Favorites`) inside instructions as long as it satisfies Pod (plain old data) constraints.

---

## 🧠 State Management

### `api/src/state/favorites.rs`

This is where your account data layout is defined.

```rust
use steel::*;
use super::FavoritesAccount;

#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq, Pod, Zeroable)]
pub struct Favorites {
    pub number: u8,
    pub color: [u8; 16],
    pub hobbies: [[u8; 16]; 3],
}

account!(FavoritesAccount, Favorites);
```

**📌 Key Concepts**:

* `#[repr(C)]` ensures predictable memory layout for account serialization.
* The `account!` macro tags the struct with a logical `FavoritesAccount` enum variant, useful for deserialization and identifying account types.
* All fields use fixed-size arrays for compatibility with `Pod`.

### `api/src/state/mod.rs`

```rust
mod favorites;
pub use favorites::*;
use steel::*;
use crate::consts::*;

#[repr(u8)]
#[derive(Clone, Copy, Debug, Eq, PartialEq, IntoPrimitive, TryFromPrimitive)]
pub enum FavoritesAccount {
    Favorites = 0,
}

pub fn favorites_pda() -> (Pubkey, u8) {
    Pubkey::find_program_address(&[FAVORITES], &crate::id())
}
```

**🔑 Expectation**:

* PDAs are derived consistently using the same seed (`FAVORITES`) so both client and program agree on the account address.
* The `FavoritesAccount` enum is used internally by Steel to distinguish account types when reading from state.

---

## ⚙️ Program Logic

### `program/src/set_favorites.rs`

```rust
use favorites_api::prelude::*;
use steel::*;

pub fn process_set_favorites(accounts: &[AccountInfo<'_>], data: &[u8]) -> ProgramResult {
    let [signer_info, favorites_info, system_program] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    signer_info.is_signer()?;
    favorites_info
        .is_empty()?  // Account must not exist yet
        .is_writable()?  // PDA will be initialized here
        .has_seeds(&[FAVORITES], &favorites_api::ID)?;
    system_program.is_program(&system_program::ID)?;

    let args = SetFavorites::try_from_bytes(data)?;

    // Allocate and initialize the PDA account
    create_program_account::<Favorites>(
        favorites_info,
        system_program,
        signer_info,
        &favorites_api::ID,
        &[FAVORITES],
    )?;

    // Write the user-supplied data
    let favorites = favorites_info.as_account_mut::<SetFavorites>(&favorites_api::ID)?;
    favorites.data = args.data;

    Ok(())
}
```

**💡 Expectation**:

* Ensures account is only initialized once via `is_empty`.
* PDA validation ensures correct derivation using `has_seeds`.
* The use of `create_program_account` abstracts rent-exemption and serialization into a clean helper.

---

### `program/src/lib.rs`

```rust
#![allow(unexpected_cfgs)]
mod set_favorites;
use set_favorites::*;
use favorites_api::prelude::*;
use steel::*;

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    data: &[u8],
) -> ProgramResult {
    let (ix, data) = parse_instruction(&favorites_api::ID, program_id, data)?;

    match ix {
        FavoritesInstruction::SetFavorites => process_set_favorites(accounts, data)?,
    }

    Ok(())
}

entrypoint!(process_instruction);
```

---

## ✅ Integration Test

### `program/tests/test.rs`

```rust
use favorites_api::prelude::*;
use solana_program::hash::Hash;
use solana_program_test::{processor, BanksClient, ProgramTest};
use solana_sdk::{signature::Keypair, signer::Signer, transaction::Transaction};
use steel::*;

async fn setup() -> (BanksClient, Keypair, Hash) {
    let mut program_test = ProgramTest::new(
        "favorites_program",
        favorites_api::ID,
        processor!(favorites_program::process_instruction),
    );
    program_test.prefer_bpf(true);
    program_test.start().await
}

#[tokio::test]
async fn run_test() {
    let (banks, payer, blockhash) = setup().await;

    let number = 17;
    let color = string_to_bytes::<16>("blue").unwrap();
    let hobbies = [
        string_to_bytes::<16>("reading").unwrap(),
        string_to_bytes::<16>("coding").unwrap(),
        string_to_bytes::<16>("gaming").unwrap(),
    ];

    let data = Favorites {
        number,
        color,
        hobbies,
    };

    let ix = set_favorites(payer.pubkey(), data);
    let tx = Transaction::new_signed_with_payer(&[ix], Some(&payer.pubkey()), &[&payer], blockhash);
    let res = banks.process_transaction(tx).await;
    assert!(res.is_ok());

    // Validate the PDA and the stored data
    let address = favorites_pda().0;
    let account = banks.get_account(address).await.unwrap().unwrap();
    let favorites = Favorites::try_from_bytes(&account.data).unwrap();

    assert_eq!(account.owner, favorites_api::ID);
    assert_eq!(favorites.number, number);
    assert_eq!(favorites.color, color);
    assert_eq!(favorites.hobbies, hobbies);
}
```

**🧪 Expectation**:

* This test validates full integration: instruction decoding, PDA creation, account serialization, and state unpacking.
* The use of `.try_from_bytes()` matches `#[derive(Pod)]` to deserialize raw account data.
