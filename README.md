# Seamless integration of [pre-commit](https://pre-commit.com/) git hooks with [Nix](https://nixos.org/nix)

![pre-commit.png](pre-commit.png)

The goal is to **manage commit hooks with Nix** and solve the following:

- **Trivial integration for Nix projects** (wires up a few things behind the scenes)

- Provide a low-overhead build of all the tooling available for the hooks to use
   (naive implementation of calling nix-shell does bring some latency when committing)

- **Common hooks for languages** like Python, Haskell, Elm, etc.

- Run hooks **as part of development** and **on your CI**

# Getting started

1. (optional) Use binary caches to avoid compilation:

   ```bash
   nix-env -iA cachix -f https://cachix.org/api/v1/install
   cachix use pre-commit-hooks
   ```

2. Integrate hooks to be built as part of `default.nix`:
   ```nix
    let
      nix-pre-commit-hooks = import (builtins.fetchTarball "https://github.com/cachix/pre-commit-hooks.nix/tarball/master");
    in {
      pre-commit-check = nix-pre-commit-hooks.run {
        src = ./.;
        # If your hooks are intrusive, avoid running on each commit with a default_states like this:
        # default_stages = ["manual" "push"];
        hooks = {
          elm-format.enable = true;
          ormolu.enable = true;
          shellcheck.enable = true;
        };
      };
    }
   ```

   Run `nix-build -A pre-commit-check` to perform the checks as a Nix derivation.

3. Integrate hooks to prepare environment as part of `shell.nix`:
   ```nix
    (import <nixpkgs> {}).mkShell {
       shellHook = ''
        ${(import ./default.nix).pre-commit-check.shellHook}
      '';
    }
   ```

   Add `/.pre-commit-config.yaml` to `.gitignore`.

   Run `nix-shell` to execute `shellHook` which will:
   
   - build the tools and `.pre-commit-config.yaml` config file symlink which
     references the binaries, for speed and safe garbage collection
   - provide the `pre-commit` executable that `git commit` will invoke

## nix flakes

To initialise a project with a `flake.nix` and pre-commit hooks, run:

```bash
nix flake init -t github:cachix/pre-commit-hooks.nix
```

Alternatively, the following snippet shows how pre-commit hooks may be integrated into an already existing `flake.nix`:

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs-channels/nixos-20.03";

  inputs.flake-utils.url = "github:numtide/flake-utils";

  inputs.pre-commit-hooks.url = "github:cachix/pre-commit-hooks.nix/master";

  outputs = { self, nixpkgs, flake-utils, pre-commit-hooks }:
    flake-utils.lib.eachDefaultSystem
      (
        system: rec {
          pre-commit-check = pre-commit-hooks.packages.${system}.run {
            src = ./.;
            # If your hooks are intrusive, avoid running on each commit with a default_states like this:
            # default_stages = ["manual" "push"];
            hooks = {
              elm-format.enable = true;
              ormolu.enable = true;
              shellcheck.enable = true;
            };
          };

          devShell =
            nixpkgs.legacyPackages.${system}.mkShell {
              inherit (pre-commit-check) shellHook;
            };
        }
      );
}
```

Add `/.pre-commit-config.yaml` to the `.gitignore`.

With this in place you may run

```bash
nix build '.#pre-commit-check.{system}' --impure
```

(in which `{system}` is one of `aarch64-linux`, `i686-linux`, `x86_64-darwin`, and `x86_64-linux`)

to perform the checks as a Nix derivation or run

```bash
nix develop
```

to execute the `shellHook` which will:

- build the tools and `.pre-commit-config.yaml` config file symlink which
  references the binaries, for speed and safe garbage collection
- provide the `pre-commit` executable that `git commit` will invoke

## Optional

### Direnv + Lorri

`.envrc`

```
eval "$(lorri direnv)"

# Use system PKI
unset SSL_CERT_FILE
unset NIX_SSL_CERT_FILE

# Install pre-commit hooks
eval "$shellHook"
```

# Hooks

## Nix

- [nixpkgs-fmt](https://github.com/nix-community/nixpkgs-fmt)
- [nixfmt](https://github.com/serokell/nixfmt/)
- [nix-linter](https://github.com/Synthetica9/nix-linter)

## Haskell

- [ormolu](https://github.com/tweag/ormolu)
- [hindent](https://github.com/chrisdone/hindent)
- [hlint](https://github.com/ndmitchell/hlint)
- [cabal-fmt](https://github.com/phadej/cabal-fmt)
- [brittany](https://github.com/lspitzner/brittany)
- [hpack](https://github.com/sol/hpack)

## Elm

- [elm-format](https://github.com/avh4/elm-format)

## Python

- [black](https://github.com/psf/black)

## Rust

- [rustfmt](https://github.com/rust-lang/rustfmt)
- [clippy](https://github.com/rust-lang/rust-clippy)
- cargo-check: Runs `cargo check`

*Warning*: running `clippy` after `cargo check` hides
(https://github.com/rust-lang/rust-clippy/issues/4612) all clippy specific
errors. Clippy subsumes `cargo check` so only one of these two checks should by
used at the same time.

## Shell

- [shellcheck](https://github.com/koalaman/shellcheck)

## Terraform

- `terraform-format`: built-in formatter

# Contributing hooks

Everyone is encouraged to add new hooks.

<!-- TODO generate option docs -->
Have a look at the [existing hooks](modules/hooks.nix) and the [options](modules/pre-commit.nix).

There's no guarantee the hook will be accepted, but the general guidelines are:

- Nix closure of the tool should be small e.g. `< 50MB`
- The tool must not be very specific (e.g. language tooling is OK, but project specific tooling is not)
- The tool needs to live in a separate repository (even if a simple bash script, unless it's a oneliner)
