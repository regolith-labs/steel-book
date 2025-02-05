# steel-cli

A CLI is provided to support building and managing an Steel workspace. For a
comprehensive list of commands and options, run `steel -h` on any of the
following subcommands.

```sh
A command line interface for building Solana programs.

Usage: steel [OPTIONS] <COMMAND>

Commands:
  new    Create a new Solana program
  build  Compile a program and all of its dependencies
  test   Execute all unit and integration tests
  clean  Remove artifacts cargo has generated in the past
  help   Print this message or the help of the given subcommand(s)

Options:
  -k, --keypair <KEYPAIR>        Sets the primary signer on any subcommands that require a signature. Defaults to the signer in Solana CLI's YAML config file which is usually located at `~/.config/solana/cli/config.yml`. This arg is parsed identically to the vanilla Solana CLI and supports `usb://` and `prompt://` URI schemes as well as filepaths to keypair JSON files [env: KEYPAIR=]
  -u, --url <URL>                Sets the Solana RPC URL. Defaults to the `rpc_url` in Solana CLI's YAML config file which is usually located at `~/.config/solana/cli/config.yml` [env: RPC_URL=]
      --commitment <COMMITMENT>  Set the default commitment level of any RPC client requests [env: COMMITMENT=] [possible values: processed, confirmed, finalized]
  -h, --help                     Print help
  -V, --version                  Print version
```

## Global Options

| Option | Description | Environment Variable |
|--------|-------------|---------------------|
| `-k, --keypair <KEYPAIR>` | Sets the primary signer. Defaults to signer in `~/.config/solana/cli/config.yml`. Supports `usb://`, `prompt://` URIs and keypair JSON filepaths | `KEYPAIR` |
| `-u, --url <URL>` | Sets the Solana RPC URL. Defaults to `rpc_url` in `~/.config/solana/cli/config.yml` | `RPC_URL` |
| `--commitment <COMMITMENT>` | Sets default commitment level for RPC client requests. Values: `processed`, `confirmed`, `finalized` | `COMMITMENT` |
| `-h, --help` | Prints help information | - |
| `-V, --version` | Prints version information | - |

## Commands

### New
```sh
steel new <NAME> [--no-git]
```

Creates a new Solana program project.

Options:

- --no-git: Creates project without git initialization or .gitignore

### Build

```sh
steel build
```

Compiles the program and all of its dependencies.

### Test

```sh
steel test
```

Executes all unit and integration tests for the program.

### Clean

```sh
steel clean
```

Removes artifacts that cargo has generated in previous builds.

### Help

```sh
steel help [SUBCOMMAND]
```

Prints help information. When a subcommand is specified, prints detailed help for that subcommand.

## Usage Examples

1. Create a new project:
```sh
steel new my_solana_program
```

2. Create a project without git:
```sh
steel new my_solana_program --no-git
```

3. Build with custom RPC URL:
```sh
steel build --url https://api.mainnet-beta.solana.com
```

4. Test with specific keypair:
```sh
steel test --keypair ~/my-keypair.json
```

5. Build with specific commitment level:
```sh
steel build --commitment finalized
```
