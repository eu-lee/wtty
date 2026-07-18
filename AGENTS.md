# Agent Development Guide

A file for [guiding coding agents](https://agents.md/).

## Git Remotes — IMPORTANT

This repository is a private fork. All git and GitHub activity MUST stay on
the fork, never the original upstream project:

- **ONLY push to `eu-lee/ghostty`** (the `origin` remote). NEVER push to
  `ghostty-org/ghostty`, `mitchellh/ghostty`, or any other remote.
- Do not add an upstream remote. If one exists, do not push or fetch-and-PR
  against it.
- Open pull requests against `eu-lee/ghostty` only. `gh` on a fork can
  default the PR base to the upstream repository — always pass
  `--repo eu-lee/ghostty` to `gh pr create` (a `gh repo set-default
  eu-lee/ghostty` is configured, but do not rely on it).
- Never open issues, comments, or PRs on the upstream repository.

## Commands

- **Build:** `zig build`
  - If you're on macOS and don't need to build the macOS app, use
    `-Demit-macos-app=false` to skip building the app bundle and speed up
    compilation.
- **Test (Zig):** `zig build test`
  - Prefer to run targeted tests with `-Dtest-filter` because the full
    test suite is slow to run.
- **Test filter (Zig)**: `zig build test -Dtest-filter=<test name>`
- **Formatting (Zig)**: `zig fmt .`
- **Formatting (Swift)**: `swiftlint lint --strict --fix`
- **Formatting (other)**: `prettier -w .`

## libghostty-vt

- Build: `zig build -Demit-lib-vt`
- Build WASM: `zig build -Demit-lib-vt -Dtarget=wasm32-freestanding -Doptimize=ReleaseSmall`
- Test: `zig build test-lib-vt -Dtest-filter=<filter>`
  - Prefer this when the change is in a libghostty-vt file
- All C enums in `include/ghostty/vt/` must have a `_MAX_VALUE = GHOSTTY_ENUM_MAX_VALUE`
  sentinel as the last entry to force int enum sizing (pre-C23 portability).

## Directory Structure

- Shared Zig core: `src/`
- macOS app: `macos/`
- GTK (Linux and FreeBSD) app: `src/apprt/gtk`

