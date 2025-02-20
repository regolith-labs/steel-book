# First Steps

This section introduces the steel command-line tool. We will walk through creating a new project, compiling it, and running tests.

To start a new project with Steel, use the [`steel`](../reference/steel/steel-new.md) command:

```sh
steel new hello-steel
```

Now, let’s explore the structure that `steel` has generated for us:

```sh
cd hello-steel

tree . -d -L 1
.
├── api
└── program
```

Compile your program using the Solana toolchain:
```sh
steel build
```

Test your program using the Solana toolchain:
```sh
steel test
```

> 💡 **Tip**
>
> You can always view detailed help for any command or subcommand by appending `--help` to it.

## File structure

While not strictly enforced, we recommend organizing your Solana program with the following file structure. We have found this pattern to improve code readability, separating the contract interface from its implementation. It scales well for complex contracts.

```
Cargo.toml (workspace)
⌙ api
  ⌙ Cargo.toml
  ⌙ src
    ⌙ consts.rs
    ⌙ error.rs
    ⌙ event.rs
    ⌙ instruction.rs
    ⌙ lib.rs
    ⌙ loaders.rs
    ⌙ sdk.rs
    ⌙ state
      ⌙ mod.rs
      ⌙ account_1.rs
      ⌙ account_2.rs
⌙ program
  ⌙ Cargo.toml
  ⌙ src
    ⌙ lib.rs
    ⌙ instruction_1.rs
    ⌙ instruction_2.rs
```
