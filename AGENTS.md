# Repository Agent Guide

## What this repo is

`cachix-warmup` is a GitHub Actions–only repository. It contains no source
code — just three workflows under `.github/workflows/`. Two of them pre-build
the GHC toolchain used by a set of Haskell/Nix projects and push it to the
[`paolino`](https://app.cachix.org/cache/paolino) Cachix cache (one for Linux,
one for macOS). The third is a proof-of-concept for distributing a Nix-built
binary through Homebrew (`lambdasistemi/homebrew-tap`).

## How to work here

There is no build or test step — every change is an edit to YAML under
`.github/workflows/`.

- List the workflows: `gh workflow list --repo lambdasistemi/cachix-warmup`
- Trigger a warm-up (manual dispatch):
  `gh workflow run "Linux GHC Cache" --repo lambdasistemi/cachix-warmup`
  (also `"Darwin GHC Cache"`, `"Homebrew Pipeline Test"`)
- Watch a run: `gh run watch --repo lambdasistemi/cachix-warmup`
- Inspect a run's logs: `gh run view --repo lambdasistemi/cachix-warmup --log`

The warm-up targets are hard-coded bash arrays inside the workflow scripts:
the `repos` array in `linux-ghc.yml` and the `targets` array in
`darwin-ghc.yml`. Adding a project means appending a line there, not changing
any code.

## Skills

Activatable procedures live under `skills/`. Load the one whose description
matches your task.

- `skills/cachix-warmup-guide/` — repository map, what each workflow does, the
  Cachix cache details, and how to add or change a warm-up target.
