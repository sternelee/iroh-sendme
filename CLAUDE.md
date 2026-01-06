# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Sendme is a Rust CLI tool for sending files and directories over the internet using the [iroh](https://crates.io/crates/iroh) networking library. It provides P2P file transfer with NAT hole punching, blake3 verified streaming, and resumable downloads.

## Development Commands

### Building
- `cargo build` - Build the project
- `cargo build --release` - Build optimized release binary

### Testing
- `cargo test` - Run all tests
- `cargo test --test cli` - Run CLI integration tests specifically
- Tests are located in `tests/cli.rs` and use the `duct` crate for subprocess testing

### Linting and Formatting
- `cargo fmt --all -- --check` - Check code formatting
- `cargo clippy --locked --workspace --all-targets --all-features` - Run Clippy lints
- `cargo fmt` - Format code

### Other
- `cargo run -- --help` - Show CLI usage
- `cargo install --path .` - Install locally

## Architecture

### Core Structure

The entire application is contained in `src/main.rs` (a single-binary CLI). Key architectural components:

#### Commands
Two main subcommands defined in the `Commands` enum:
- **Send** (`sendme send <path>`): Hosts a file/directory for transfer
- **Receive** (`sendme receive <ticket>`): Downloads data using a ticket

#### Send Flow (`send` function)
1. Creates/loads a secret key (from `IROH_SECRET` env var or generates one)
2. Builds an iroh `Endpoint` with relay and discovery configuration
3. Creates a temporary `.sendme-send-*` directory for blob storage
4. Imports the file/directory into an iroh-blobs `FsStore` via `import()`
   - Uses `WalkDir` to traverse directories
   - Parallelizes import using `num_cpus` workers with `futures_buffered`
   - Creates a `Collection` containing all file hashes
5. Creates and starts a `BlobsProtocol` provider with progress events
6. Generates a `BlobTicket` containing endpoint address and collection hash
7. Waits for connections, serves data via iroh protocol router
8. On shutdown, cleans up temp directory

#### Receive Flow (`receive` function)
1. Parses the ticket to get endpoint address and data hash
2. Creates an iroh `Endpoint` for connecting
3. Creates temporary `.sendme-recv-*` directory for blob storage
4. Loads or downloads the collection:
   - Checks local store for existing data via `local()`
   - If incomplete: connects to sender, downloads via `execute_get()`
   - Progress shown via multi-progress bars (`indicatif` crate)
5. Exports the collection to current directory via `export()`
6. Cleans up temp directory

#### Key Data Structures
- `Collection`: Represents a set of files (hash + name pairs)
- `BlobTicket`: Encodes endpoint address + hash for sharing
- `Store`/`FsStore`: iroh-blobs storage backend for content-addressed data
- `Endpoint`: iroh networking endpoint with hole-punching and relay support

#### Progress Reporting
Uses `indicatif` `MultiProgress` for concurrent progress bars:
- Import/export progress per file
- Overall operation progress
- Download progress with throughput display
- Send-side connection/request progress via provider events

#### Path Handling
- `canonicalized_path_to_string()`: Converts paths to platform-agnostic strings
- Validates path components to prevent directory traversal
- All temp directories use `.sendme-*` prefix

### Dependencies
- `iroh` / `iroh-blobs`: Core networking and content-addressed storage
- `clap`: CLI argument parsing with derive macros
- `tokio`: Async runtime
- `indicatif`: Progress bars
- `walkdir`: Directory traversal
- `futures-buffered`: Buffered concurrent futures processing
- `crossterm` (optional): Clipboard support and keyboard handling

### MSRV
Minimum Supported Rust Version: **1.81** (must also update in `.github/workflows/ci.yml` if changed)
