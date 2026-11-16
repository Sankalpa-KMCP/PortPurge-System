# PortPurge

[![Latest Release](https://img.shields.io/github/v/release/Sankalpa-KMCP/PortPurge-System?style=flat-square&color=238636)](https://github.com/Sankalpa-KMCP/PortPurge-System/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Built with Tauri](https://img.shields.io/badge/Built_with-Tauri_v2-24C8DB?style=flat-square&logo=tauri&logoColor=white)](https://tauri.app/)
[![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey?style=flat-square)](https://github.com/Sankalpa-KMCP/PortPurge-System/releases/latest)

A lightweight, cross-platform desktop utility designed for developers to inspect, troubleshoot, and terminate localhost-bound network processes without command-line overhead.

Built with **Tauri v2**, **Rust**, **React 19**, **TypeScript**, and **Tailwind CSS v4**.

---

## Overview

When developing microservices, web applications, or local servers, encountering "port already in use" errors often forces developers to manually run shell commands like `netstat`, `lsof`, or `taskkill`.

PortPurge automates localhost socket discovery, identifies owning processes, presents deep diagnostic details (such as memory usage, user context, start time, and CLI arguments), and terminates stubborn processes safely with one click.

---

## Downloads and Installation

Pre-compiled binary packages are available directly from the [Releases](https://github.com/Sankalpa-KMCP/PortPurge-System/releases/latest) page.

| Platform | Format | Description |
|---|---|---|
| **Windows** | `.exe` | Standard NSIS Setup installer (`PortPurge_x.x.x_x64-setup.exe`) |
| **Windows** | `.msi` | Enterprise-compatible Windows Installer package (`PortPurge_x.x.x_x64_en-US.msi`) |
| **All Platforms** | Source | Clone and build locally using Cargo and Node.js |

### In-App Auto-Updates

PortPurge integrates the official Tauri auto-updater. On launch, the client checks GitHub Releases for new signed releases. When an update is detected, users can download and restart into the latest version with one click. Release artifacts are cryptographically signed using Minisign keys.

---

## Core Capabilities

### Localhost Monitoring

- **Focused discovery** — Surfaces active sockets bound strictly to `127.0.0.1`, `::1`, and `localhost` (TCP listeners and UDP sockets). External interfaces are filtered out to keep the workspace clean.
- **Asynchronous non-blocking I/O** — System commands execute in Rust worker threads via `spawn_blocking`, ensuring the UI never stutters during scans.
- **Automated polling and manual refresh** — Refreshes port states every 3 seconds while providing instant manual refresh and timestamp tracking.
- **Fuzzy search and protocol filtering** — Instantly filter by port number, PID, process name, or protocol (ALL / TCP / UDP).
- **Process grouping** — Collapse multiple port allocations under shared process names; terminate all associated PIDs in a single action.

### Deep Process Inspection

- **Detailed process metrics** — Inspect any active port to view executable path, full command-line arguments, resident memory usage (RSS), owning user, and process creation timestamp.
- **Graceful ACL handling** — If operating system security policies restrict access to elevated process details, the modal surfaces available fields and displays an elevation advisory.
- **Direct purge from inspector** — Transition directly from the inspection modal into the process termination flow.

### Global Summon Shortcut

- **System-wide hotkey** — Press `Alt+Shift+P` (Windows/Linux) or `Option+Shift+P` (macOS) from any active application to summon PortPurge instantly.
- **Foreground focus and search auto-select** — Restores the application from the background or system tray and automatically focuses the fuzzy search input.
- **Graceful registration** — Uses `tauri-plugin-global-shortcut`. Conflicts with existing operating system shortcuts are logged without crashing the application.

### Smart Protect (Safe Mode)

- **Protected process denylist** — Prevents accidental termination of critical system processes across the UI, batch actions, and tray shortcuts.
- **Backend enforcement** — All terminations pass through the `ledger::kill_and_record` gate, which validates targets against `config.json` before executing operating system kill routines.
- **Visual indicators** — Protected processes feature a Shield indicator, reduced opacity, and disabled termination controls.
- **Standard seeds** — Default system seeds are generated automatically on first startup:
  - **Windows**: `System`, `smss.exe`, `csrss.exe`, `wininit.exe`, `services.exe`, `lsass.exe`, `svchost.exe`, `explorer.exe`
  - **macOS**: `launchd`, `kernel_task`, `WindowServer`, `loginwindow`, `syspolicyd`
  - **Linux**: `systemd`, `systemd-journal`, `sshd`

### Purge Ledger (Audit Log)

- **Persistent history** — Every termination attempt (successful or blocked) is written to a local JSON log (`purge-ledger.json`), retaining the last 100 actions.
- **Sliding drawer UI** — View chronological records detailing process name, PID, port, protocol, invocation source, timestamp, and termination status.
- **Real-time event synchronization** — Tauri IPC events update the history drawer instantly when actions occur from either the main window or system tray.

### System Tray Integration

- **Quick-kill slots** — Right-click the system tray icon to view up to 5 recently discovered localhost ports for instant termination.
- **Port load status** — Tray icon color dynamically reflects current localhost socket volume (normal, elevated, high load).
- **Background operation** — Closing the application window minimizes to the system tray rather than terminating the process.

---

## Architecture

```
+--------------------------------------------------------------+
|                     React Frontend                           |
|  App.tsx (state, IPC, polling, ledger, summon focus)         |
|  LedgerDrawer | PortTable | SearchFilters | Modals           |
+------------------------------+-------------------------------+
                               |  Tauri IPC + Events
                               v
+--------------------------------------------------------------+
|                Tauri Rust Backend (lib.rs)                   |
|        IPC commands | summon | config | ledger | tray        |
+------------------------------+-------------------------------+
                               |
         +---------------------+---------------------+
         v                     v                     v
+--------------+      +-----------------+      +---------------+
| summon/mod.rs|      |  ledger/mod.rs  |      |  sys/ (native)|
| Alt+Shift+P  |      |  kill_and_record|      |  port scan    |
| window focus |      |  ledger persist |      |  kill & inspect|
+--------------+      +-----------------+      +---------------+
```

### Module Responsibilities

| Path | Subsystem | Responsibility |
|---|---|---|
| `src/App.tsx` | Frontend | State orchestration, IPC polling, event listeners, search auto-focus |
| `src/components/PortTable.tsx` | Frontend | Sortable, searchable port listings with grouping and action triggers |
| `src/components/LedgerDrawer.tsx` | Frontend | Audit log drawer surfacing recent process termination events |
| `src-tauri/src/summon/mod.rs` | Backend | Global OS hotkey registration, window focus restoration, summon events |
| `src-tauri/src/config/mod.rs` | Backend | Smart Protect configuration loader, seed provisioning, denylist check |
| `src-tauri/src/ledger/mod.rs` | Backend | Append-only audit logging, disk persistence, UI synchronization |
| `src-tauri/src/tray/mod.rs` | Backend | System tray menu generation, quick-kill dispatch, load indicators |
| `src-tauri/src/sys/windows.rs` | Backend | Windows `netstat` parsing, PowerShell CIM process inspection, taskkill |
| `src-tauri/src/sys/unix.rs` | Backend | Unix `lsof` listener parsing, `ps` process inspection, SIGKILL routing |

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Desktop Framework** | Tauri v2 | Native system bindings, minimal binary footprint, secure IPC |
| **Core Runtime** | Rust | High-performance socket inspection and process lifecycle control |
| **User Interface** | React 19, TypeScript | Reactive interface state and type-safe component logic |
| **Styling** | Tailwind CSS v4 | Hardware-accelerated glassmorphism design system |
| **Icons** | Lucide React | Clean, scalable vector interface icons |
| **Build Tooling** | Vite 7, Cargo | Optimized bundling and native compilation |

---

## Configuration Paths

PortPurge maintains its settings and audit logs within standard operating system application directories:

| Platform | Configuration Directory | Files |
|---|---|---|
| **Windows** | `%APPDATA%\com.portpurge.app` | `config.json`, `purge-ledger.json` |
| **macOS** | `~/Library/Application Support/com.portpurge.app` | `config.json`, `purge-ledger.json` |
| **Linux** | `~/.local/share/com.portpurge.app` | `config.json`, `purge-ledger.json` |

### Customizing Protected Processes

To protect custom development services (e.g., local database daemons or custom background workers), add process names to `config.json`:

```json
{
  "protectedProcessNames": [
    "postgres",
    "mysqld",
    "docker",
    "redis-server"
  ]
}
```

Matching rules:
1. Whitespace is trimmed automatically.
2. Comparison is case-insensitive.
3. Trailing `.exe` suffixes are stripped during evaluation (`node.exe` matches `node`).

---

## Local Development

### Prerequisites

- **Node.js**: v20 or higher (v24+ recommended)
- **Rust & Cargo**: v1.80 or higher
- **Native OS Tools**:
  - Windows: `netstat`, `tasklist`, `taskkill`, PowerShell
  - macOS / Linux: `lsof`, `ps`, `kill`

### Setup Instructions

1. **Clone the repository:**
   ```bash
   git clone https://github.com/Sankalpa-KMCP/PortPurge-System.git
   cd PortPurge-System
   ```

2. **Install frontend dependencies:**
   ```bash
   npm install
   ```

3. **Start development server:**
   ```bash
   npm run tauri dev
   ```

4. **Run test suites:**
   ```bash
   # Rust backend tests
   cargo test --manifest-path src-tauri/Cargo.toml

   # Frontend component tests
   npm run test
   ```

5. **Build release installer:**
   ```bash
   npm run tauri build
   ```

---

## Security and Privileges

- **Least Privilege Execution** — PortPurge runs without elevation by default. To inspect or terminate services owned by other users or the operating system, launch PortPurge as Administrator (Windows) or via `sudo` (macOS/Linux).
- **Input Sanitization** — All IPC commands enforce strict numeric types (`u32` PIDs, `u16` ports). Process identifiers are never passed through raw shell string concatenation.
- **Argument Caution** — The Process Details modal inspects full command-line arguments. Developers should exercise standard caution when inspecting processes initialized with inline secrets or tokens.

---

## License

This project is licensed under the [MIT License](LICENSE).
