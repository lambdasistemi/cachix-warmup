---
name: cachix-warmup-guide
description: Guide to the lambdasistemi/cachix-warmup repository — a GitHub Actions-only repo that warms the `paolino` Cachix cache with prebuilt GHC and tests a Homebrew distribution pipeline. Load when working in cachix-warmup, editing `.github/workflows/linux-ghc.yml`, `darwin-ghc.yml`, or `brew-pipeline.yml`, adding or removing a GHC warm-up target repo, debugging a "Warm GHC" or "Homebrew Pipeline Test" run, dealing with the `paolino` cachix cache, `devShells.<system>.default.ghc`, `cachix push`, `CACHIX_AUTH_TOKEN`, `TAP_TOKEN`, the `hello-test-v1` release, the generated `hello-nix.rb` formula, or `lambdasistemi/homebrew-tap`. Keywords: cachix warmup, warm GHC, GHC cache, x86_64-linux, aarch64-linux, aarch64-darwin, substituter, install-nix-action.
---

# cachix-warmup guide

## Repository map

The entire repository is three workflow files; there is no other code.

| Path | Purpose |
| --- | --- |
| `.github/workflows/linux-ghc.yml` | "Linux GHC Cache" — matrix over `x86_64-linux` + `aarch64-linux`; builds and pushes GHC for the Linux target repos. |
| `.github/workflows/darwin-ghc.yml` | "Darwin GHC Cache" — single `aarch64-darwin` runner; builds and pushes GHC for the Darwin target repos. |
| `.github/workflows/brew-pipeline.yml` | "Homebrew Pipeline Test" — bundles `nixpkgs#hello`, releases it, generates a Homebrew formula, pushes it to the tap, and installs it. |
| `README.md` | Human-facing overview, workflow table, cache-consumer config. |
| `AGENTS.md` | Agent entry point. |

## Build, test, run

No build. No test suite. Validation is whether a dispatched run succeeds.

```bash
gh workflow list --repo lambdasistemi/cachix-warmup
gh workflow run "Linux GHC Cache"        --repo lambdasistemi/cachix-warmup
gh workflow run "Darwin GHC Cache"       --repo lambdasistemi/cachix-warmup
gh workflow run "Homebrew Pipeline Test" --repo lambdasistemi/cachix-warmup
gh run watch --repo lambdasistemi/cachix-warmup
gh run view  --repo lambdasistemi/cachix-warmup --log
```

The Homebrew workflow also fires automatically on every push to `main`; the two
GHC workflows are `workflow_dispatch`-only.

## Navigating the code

Both GHC workflows follow the same shape — the logic is an inline bash loop in
the "Build and push GHC" step:

1. Configure Nix with the `paolino` substituter (via
   `cachix/install-nix-action@v30`), then `nix profile install nixpkgs#cachix`.
2. For each target: `git clone --depth 1 --branch <ref>`, then
   `nix build -L <dir>#<attr> --no-link --print-out-paths`, then
   `cachix push paolino` of the resulting store path.

Key difference: `linux-ghc.yml` guards each target with
`nix eval ... .outPath` and **skips** a repo that does not yet expose the
attribute; `darwin-ghc.yml` has no such guard and **fails** if the attribute is
missing.

The cache identity (`CACHIX_CACHE="paolino"`) and the public key
(`paolino.cachix.org-1:ecmgO3CXdgSWA2cHlm4srknd/cLFMLmK3i3NrzeDFaE=`) are
literals in the workflow YAML.

`brew-pipeline.yml` is a linear sequence of named steps (build → bundle →
tarball → release → formula → install); read it top to bottom.

## Using the warm-up targets

Targets are hard-coded bash arrays, one entry per line:

- **Linux** — the `repos` array in `linux-ghc.yml`, entries `"owner/repo ref"`.
  The flake attribute is derived as `devShells.$SYSTEM.default.ghc` where
  `$SYSTEM` is each matrix system. Current targets:
  `lambdasistemi/cardano-tx-tools`, `lambdasistemi/cardano-ledger-rdf`,
  `cardano-foundation/moog` (all on `main`).
- **Darwin** — the `targets` array in `darwin-ghc.yml`, entries
  `"owner/repo ref flake-attr"` (attribute spelled out in full). Current
  targets: `lambdasistemi/agent-daemon`,
  `lambdasistemi/cardano-mpfs-offchain`, `lambdasistemi/amaru-treasury-tx`
  (all on `main`, attr `devShells.aarch64-darwin.default.ghc`).

To add a target, append a line to the relevant array. For a Darwin target the
attribute must already exist on the target repo, or the run fails; for a Linux
target a missing attribute is skipped, so it is safe to add ahead of time.

Secrets a run needs: `CACHIX_AUTH_TOKEN` (both GHC workflows),
`TAP_TOKEN` and `GITHUB_TOKEN` (Homebrew workflow).

## Answering questions

- **"What is this repo for?"** → cache warming: it prebuilds GHC for the listed
  projects so their CI/devs pull it from `paolino.cachix.org` instead of
  compiling. See README "What is this".
- **"How do I add my project to the warm cache?"** → edit the `repos` array
  (Linux) or `targets` array (Darwin); README "Development" and the section
  above.
- **"How do I consume the cache?"** → add the substituter + public key from
  README "Consuming the warmed cache".
- **"What does the Homebrew workflow do / is `hello-nix` a real package?"** →
  it is a *test* pipeline using `nixpkgs#hello`; it proves the bundle → release
  → formula → `brew install` flow against `lambdasistemi/homebrew-tap`. It is
  not a shipped product.
- **"Why didn't my Linux target build?"** → the Linux job skips a repo whose
  `devShells.<system>.default.ghc` attribute is absent; check the run log for
  the "not present … skipping" line.
