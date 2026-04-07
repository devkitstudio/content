---
title: New Features
order: 1
icon: zap
---

### Document Utilities

The `0.4.0` update introduces a brand new suite of document tools designed to help you preview and interact with different formats right from the studio:

- **Markdown Viewer**: A robust Markdown preview tool equipped with syntax highlighting, an integrated VSCode-style error pane for precise Markdown linting, and a real-time responsive UI.
- **PDF Viewer**: Added native PDF rendering support. Now you can securely preview and navigate through your `.pdf` documents entirely on the client-side.

### Data Utilities

- **UUID Generator**: A robust, zero-dependency utility for generating v1 (time-based), v4 (random), and v7 (time-ordered) UUIDs in bulk. It comes with real-time formatting toggles and ensures 100% native fallback stability for legacy browsers.

### String Utilities

- **URL Parser**: A comprehensive breakdown tool to deeply process any URI / URL structures. Designed with smart scheme identification to rapidly recognize Connection Types (Web, Databases, Websockets, Telephony) and auto-extract implied ports and Query parameter segments.
- **Text/Code Diff**: A specialized comparison utility decoupled from JSON constraints. Compare plain text, YAML, SQL, or arbitrary scripts with a dynamic language-highlighting toggle dropdown. Includes all staple Diff view features: Split/Inline modes, Minimap, Word Wrap, and Export summaries.
- **Hash Generator**: A robust multi-algorithm generator that effortlessly computes MD5, SHA-1, SHA-256, and SHA-512 hashes in perfectly synchronized real-time. Also includes powerful Bcrypt calculation featuring a customizable synchronous Salt Round selector and instant Re-roll tools.
- **Regex Tester**: A split-pane Regex101-style workspace equipped to parse JS Regular Expressions and highlight matches natively. Showcases advanced array destructuring to granularly isolate overlapping capture groups (e.g. Group 1, Group 2) along with exact string indices.

### Date Utilities

- **Cron Expression**: A dual-purpose Cron editor that acts as both a visual builder and a sharp parser. Provides a clickable interface to configure time segments without touching syntax, while instantly rendering the execution timeline and human-readable translation.

### Tool Registry Reorganization

- Refactored the internal Tool Registry. The **CSV Viewer** has been successfully moved from an isolated category into the **Data** group alongside the DB Diagram tool to create a more unified workflow.

### Cheatsheets & Documentation

- **Regex ReDoS Vulnerability Advisory**: Shipped a new internal cheatsheet module focused unconditionally on educating developers about Catastrophic Backtracking overhead on Main Threads, aiming towards proactive Regex optimization standards.
