# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About This Repository

A personal collection of Go interview preparation cheatsheets in Russian, sourced from real interviews (Avito, Kaspersky, Uzum, etc.). This is a pure markdown documentation repo — no build system, no tests, no Go source files.

## Content Organization

Files use a numbered prefix to indicate topic order within a category. The README groups them into logical sections:

- **go-internals**: `01_slices.md`, `02_map.md`, `03_interfaces.md`, `04_defer_panic_recover.md`
- **concurrency**: `01_goroutines_scheduler.md`, `02_channels.md`, `03_patterns.md`, `04_sync_primitives.md`
- **databases**: `01_indexes_transactions.md`
- **networks**: `01_http_grpc_tcp.md`

Files live in their respective subdirectories matching the structure above.

## Content Style

- Written in Russian
- Each file is a self-contained cheatsheet on its topic
- Typically includes: internal implementation details, gotchas/traps, code snippets, and interview-style Q&A
- Priority topics (most frequently asked in real interviews): goroutines/GMP scheduler, channels, DB indexes, select, ACID/transactions, slices, defer, nil interface trap
