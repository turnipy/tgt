# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

tgt is a terminal UI client for Telegram, built in Rust using ratatui/crossterm for the TUI and tdlib-rs (TDLib bindings) for the Telegram protocol.

## Build & Development Commands

All Makefile commands use `--no-default-features`, requiring you to specify a TDLib feature flag.

```bash
# Build (default features: download-tdlib + voice-message)
cargo build
# Or with explicit feature:
make build ARGS="--features download-tdlib"

# Run
cargo run
make run ARGS="--features download-tdlib"

# Tests (single-threaded, required for TDLib safety)
cargo test --no-default-features -- --nocapture --test-threads=1
make test ARGS="--features local-tdlib"

# Lint (warnings are errors)
cargo clippy --no-default-features --all-targets --features download-tdlib -- -D warnings
make clippy ARGS="--features download-tdlib"

# Format
cargo fmt --all
cargo fmt --all -- --check

# All checks (fmt + clippy + test)
make all
```

TDLib feature flags (pick one): `download-tdlib` (auto-downloads, recommended), `local-tdlib` (via `LOCAL_TDLIB_PATH` env var), `pkg-config`.

Optional features: `voice-message` (requires CMake), `chafa-dyn`/`chafa-static` (image rendering).

## Architecture

### Event-driven loop (`src/run.rs`)

The app runs a tokio async loop processing three event sources:
1. **TUI backend** — keyboard/mouse input via crossterm
2. **Telegram backend** — TDLib updates (messages, auth state, etc.)
3. **Refresh timer** — ~60 FPS redraw (configurable via `app.toml` `frame_rate`)

Events flow: `Event` → component `handle_events()` → `Action` → component `update()` → `draw()`

### Component system (`src/components/`)

All UI widgets implement the `Component` trait (`component_traits.rs`), which defines `handle_events`, `update`, and `draw`. Components are stored in a `HashMap<ComponentName, Box<dyn Component>>` in `Tui`.

Layout hierarchy:
- `Tui` (`tui.rs`) — top-level manager, owns all components
  - `TitleBar` — app info bar
  - `CoreWindow` — main layout container managing focus between:
    - `ChatListWindow` — left sidebar (resizable)
    - `ChatWindow` — message history
    - `PromptWindow` — message input
    - `ReplyMessage` — reply preview
    - Popups: `CommandGuide`, `ThemeSelector`, `SearchOverlay`, `PhotoViewer`, `FileUploadExplorer`
  - `StatusBar` — connection/message status

### Telegram integration (`src/tg/`)

`TgBackend` wraps tdlib-rs and handles auth, chat loading, message send/receive, file downloads. `TgContext` holds runtime state (chat positions, message history cache). Communication with the UI is via async channels sending `Event` variants.

### Configuration (`src/configs/`)

Two-layer config parsing: raw TOML structs (`configs/raw/`) deserialized first, then converted to processed custom structs (`configs/custom/`). The `ConfigFile` trait handles file search order, loading, and merging with defaults. Configs are versioned and auto-upgraded.

Config search order: `TGT_CONFIG_DIR` env → `./tgt-dev/config` (debug) → `~/.tgt/config` (legacy) → XDG `~/.config/tgt/config`.

### Key types

- `Action` (`action.rs`) — ~50 variant enum representing all user/system actions
- `Event` (`event.rs`) — ~20 variant enum for input and Telegram events
- `AppContext` (`app_context.rs`) — `Arc`-wrapped thread-safe app state shared across tokio tasks
- `MessageEntry` (`tg/message_entry.rs`) — parsed message wrapper
- `ComponentName` (`component_name.rs`) — enum identifying each UI component

### Key crates

- **tdlib-rs** — Telegram TDLib bindings
- **ratatui** + **crossterm** — TUI rendering and terminal I/O
- **tokio** — async runtime
- **clap** (derive) — CLI parsing
- **config** + **toml** + **serde** — configuration
- **tracing** — structured logging
