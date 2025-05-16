# Installation

### Install Rust

Solana programs are written in the
[Rust programming language](https://www.rust-lang.org/).

The recommended installation method for Rust is
[rustup](https://www.rust-lang.org/tools/install).

Run the following command to install Rust:

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
```

You should see the following message after the installation completes:

```
Rust is installed now. Great!

To get started you may need to restart your current shell.
This would reload your PATH environment variable to include
Cargo's bin directory ($HOME/.cargo/bin).

To configure your current shell, you need to source
the corresponding env file under $HOME/.cargo.

This is usually done by running one of the following (note the leading DOT):
. "$HOME/.cargo/env"            # For sh/bash/zsh/ash/dash/pdksh
source "$HOME/.cargo/env.fish"  # For fish
```

Run the following command to reload your PATH environment variable to include
Cargo's bin directory:

```shell
. "$HOME/.cargo/env"
```

To verify that the installation was successful, check the Rust version:

```shell
rustc --version
```

You should see output similar to the following:

```
rustc 1.80.1 (3f5fd8dd4 2024-08-06)
```
### Install the Solana CLI

The Solana CLI provides all the tools required to build and deploy Solana
programs.

Install the Solana CLI tool suite using the official install command:

```shell
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
```

You can replace `stable` with the release tag matching the software version of
your desired release (i.e. `v2.0.3`), or use one of the three symbolic channel
names: `stable`, `beta`, or `edge`.

If it is your first time installing the Solana CLI, you may see the following
message prompting you to add a PATH environment variable:

```
Close and reopen your terminal to apply the PATH changes or run the following in your existing shell:

export PATH="/Users/test/.local/share/solana/install/active_release/bin:$PATH"
```


### Linux

If you are using a Linux or WSL terminal, you can add the PATH environment
variable to your shell configuration file by running the command logged from the
installation or by restarting your terminal.

```shell
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
```

### Mac

If you're on Mac using `zsh`, running the default `export PATH` command logged
from the installation does not persist once you close your terminal.

Instead, you can add the PATH to your shell configuration file by running the
following command:

```shell
echo 'export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"' >> ~/.zshrc
```

Then run the following command to refresh the terminal session or restart your
terminal.

```shell
source ~/.zshrc
```

To verify that the installation was successful, check the Solana CLI version:

```shell
solana --version
```

You should see output similar to the following:

```
solana-cli 1.18.22 (src:9efdd74b; feat:4215500110, client:Agave)
```

You can view all available versions on the
[Agave Github repo](https://github.com/anza-xyz/agave/releases).

Agave is the validator client from [Anza](https://www.anza.xyz/), formerly known
as Solana Labs validator client.

To later update the Solana CLI to the latest version, you can use the following
command:

```shell
agave-install update
```

### Install Steel Cli

Install the Steel CLI:

```sh
cargo install steel
```

Verify Steel was installed:

```sh
steel --version
```

You should see output similar to the following:

```sh
steel-cli 2.1.1
```
