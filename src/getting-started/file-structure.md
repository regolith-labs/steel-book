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

## File Descriptions:

### Workspace Configuration

- **`Cargo.toml (workspace)`**: The workspace configuration file for managing dependencies and building multiple packages (API and program).

### API Module (`api/`)

- **`api/`**: This contains the program's API logic, including constants, error handling, events, instructions, and state management. The api folder serves as the contract interface.
- **`api/Cargo.toml`**: The Cargo configuration file for the API package, managing dependencies for the API part of the program.
- **`api/src/consts.rs`**: This File Defines constants used across the program to ensure consistency and readability in the codebase.
- **`api/src/error.rs`**: Contains error types for handling specific error cases in the program.
- **`api/src/event.rs`**: Contains event definitions for the program to emit events to the blockchain.
- **`api/src/instruction.rs`**: This handles the program's instructions, defining what actions can be performed by the contract.
- **`api/src/lib.rs`**: The main entry point for the API logic, connecting all components together.
- **`api/src/loaders.rs`**: Helper functions for loading accounts or data.
- **`api/src/sdk.rs`**: SDK-specific logic for interacting with the Solana blockchain.
- **`api/src/state/`**: Contains files defining the state structures of different accounts or entities in the program.
  - **`state/mod.rs`**: The entry point for managing the program state, organizing all state-related logic.
  - **`state/account_1.rs`**: Defines the structure and logic of Account 1.
  - **`state/account_2.rs`**: Defines the structure and logic of Account 2.

### Program Module (`program/`)

- **`program/`**: Contains the implementation of the program's core logic and instructions.|
- program/Cargo.toml: The Cargo configuration file for the program package.
- **`program/src/lib.rs`**: This is the main entry point for the program logic, It is responsible for instruction processing and interaction with the blockchain.
- **`program/src/instruction_1.rs`**: Defines the logic for a particular instrcuction(Instruction 1 in this case) in the program.
- **`program/src/instruction_2.rs`**: Defines the logic for another instruction (Instruction 2 in this case) in the program.
