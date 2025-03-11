Testing in Steel ensures that Solana programs function correctly before deployment. Steel builds on Solana’s `program-test` framework, allowing you to run on-chain programs in a simulated environment. This ensures program correctness, prevents costly bugs, and efficiently verifies state changes.

## How Steel Facilitates Testing

Steel enforces a structured approach to testing by:

- Using `solana-program-test` to execute program logic in an isolated environment.
- Providing a way to manually construct transactions, ensuring you understand how their instructions interact with the blockchain.
- Allowing direct access to account data for precise validation of expected state changes.

Since Steel does not abstract transaction creation, you have full control over signing, submitting, and verifying transactions. This is particularly useful for testing scenarios where specific accounts must sign transactions, multiple instructions need to be executed in sequence, or fine-tuned error handling is required.

## Key Concepts

### **Program Test Framework**

Steel runs tests inside a local, in-memory Solana ledger created using `ProgramTest`. This simulates real-world execution, including account ownership rules, instruction processing, and transaction fee calculations. Because the test environment is isolated, each test starts with a clean slate, preventing unintended interactions between test cases.

### **BanksClient**

The `BanksClient` provides a way to interact with the test ledger by submitting transactions, querying account states, and retrieving logs. Direct access to `BanksClient` allows you to:

- Fetch account data before and after transactions to confirm expected changes.
- Handle errors at the transaction level and inspect their causes.
- Manually check program logs to debug failures.

This level of control is useful when testing behaviors that involve multiple accounts, cross-program invocations, or conditional logic based on transaction results.

### **Transactions & Instructions**

Steel requires tests to construct transactions explicitly. You must:

1. Create individual instructions specifying which accounts are involved.
2. Bundle these instructions into a transaction.
3. Sign the transaction with the necessary keys.
4. Submit the transaction to the test ledger and check for success or failure.

This process ensures that you fully understand how their program interacts with the blockchain rather than relying on automated transaction handling. It also makes testing edge cases, such as missing signatures, incorrect instruction data, or invalid account ownership easier.