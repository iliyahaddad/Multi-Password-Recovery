# Multi Password Recovery - Overview

## What It Is

**Multi Password Recovery** is a comprehensive, native Windows x86 password recovery utility written entirely in Assembly (MASM32). It extracts saved credentials from **every major credential store on Windows** — web browsers, Windows system vaults, email clients, FTP clients, instant messengers, and legacy applications — all in a single portable executable.

## Why It's Powerful

### 1. Unmatched Browser Coverage
- **Chromium Family**: Chrome, Edge, Brave, Vivaldi, Yandex, Opera/Opera GX, Chrome Canary, Chromium — extracts logins via `Local State` master key + AES-GCM/DPAPI decryption
- **Firefox Family**: Firefox, Thunderbird, SeaMonkey — full NSS key4.db ASN.1 parsing with PBKDF2-SHA256, PBES2-AES, and 3DES-CBC decryption
- Supports **all modern encryption formats**: v10 (AES-GCM), v20 (App-Bound Encryption with fallback), legacy 3DES/AES-CBC

### 2. Complete Windows System Integration
- **Credential Manager**: Generic, domain, and visible credentials via `CredEnumerateW`
- **Windows Vault**: IE/Edge vault entries via `vaultcli.dll` runtime resolution
- **Wi-Fi Profiles**: All saved wireless network passwords via `wlanapi.dll`
- **RAS/VPN**: Dial-up and VPN connection credentials from phonebook files
- **LSA Secrets**: Local Security Authority secrets (system-level, cached domain credentials)
- **System Vault**: `%ProgramData%\Microsoft\Vault\` system-level entries
- **Credential Files (.cred)**: DPAPI-encrypted files in `%AppData%\Microsoft\Credentials\`
- **Protected Storage (PStore)**: Legacy IE/Outlook Express credentials

### 3. Application & Legacy Support
- **Email Clients**: The Bat!, Becky! — account configuration decryption
- **FTP Clients**: WS_FTP (.ini), CuteFTP (.dat files)
- **Instant Messengers**: Trillian, Miranda IM, ICQ 2003, &RQ (amIRC)
- **Other Apps**: Outlook/Express, FileZilla, WinSCP, FAR Manager, Total Commander

### 4. Advanced Cryptography Implementation
- **DPAPI** (Data Protection API) — dynamic `crypt32.dll` loading, `CryptUnprotectData`
- **AES-GCM** — Galois/Counter Mode for modern browser passwords (v10 format)
- **AES-CBC** — Cipher Block Chaining for legacy encryption
- **3DES-CBC** — Triple DES for Firefox/Chrome legacy formats
- **PBKDF2-SHA256** — Firefox master key derivation
- **HMAC-SHA1** — Mozilla 3DES key derivation function
- **SHA-1** — Hash computation
- All crypto via **BCrypt** (`bcrypt.dll`) with runtime function resolution

### 5. Anti-Locking & Reliability
- SQLite databases **copied to temp files** before reading — avoids "database is locked" errors when browsers are running
- **Graceful degradation** — missing DLLs/files/registry keys don't crash; modules simply return no items
- **Threaded scanning** — GUI remains responsive; results stream into ListView in real-time
- **Critical section protected** item store — thread-safe concurrent access

### 6. Export & Usability
- **Real-time filtering** — instant search across all columns
- **Show/Hide passwords** toggle
- **Export to CSV, TXT, JSON** — with password masking control
- **Copy row / username / password** to clipboard
- **Open profile folder** context menu
- **Portable** — single `.exe` + `sqlite3mc.dll`, no installation

---

## Architecture Highlights

| Component | Technology | Purpose |
|-----------|------------|---------|
| **GUI** | Win32 API (ListView, StatusBar, Toolbar) | Native, lightweight, no runtime dependencies |
| **Assembly** | MASM32 x86 (32-bit) | Maximum performance, minimal size (~600 KB) |
| **Linker** | Pelle Polink | Optimized PE generation |
| **SQLite** | sqlite3mc.dll (runtime loaded) | Chrome/Firefox database access |
| **Crypto** | BCrypt (runtime) + DPAPI | Modern + legacy decryption |
| **Manifest** | Common Controls 6.0 | Visual styles on Windows 7-11 |

---

## Supported Platforms

| OS | Architecture | Status |
|----|-------------|--------|
| Windows 7 | x86 (WoW64) | ✅ |
| Windows 8/8.1 | x86 (WoW64) | ✅ |
| Windows 10 | x86 (WoW64) | ✅ |
| Windows 11 | x86 (WoW64) | ✅ |

**Note:** 32-bit only. Requires Administrator for LSA Secrets, System Vault, Wi-Fi, and some RAS/VPN credentials.

---

## Design Philosophy

> **Local-only. No network. No persistence. No stealth. Your credentials, your machine, your control.**

- Zero telemetry, zero external connections
- All decryption happens in-process on your machine
- Exported reports contain only what you explicitly choose to export
- Open source (MIT) — audit the code yourself

---

## Quick Start

```cmd
# Build
build.bat

# Run (sqlite3mc.dll must be in same directory)
Multi Password Recovery.exe
```

The scan starts automatically on launch. Results appear in real-time. Use the filter box to narrow results. Toggle "Show passwords" to reveal/hide. Export via Save CSV/TXT/JSON buttons.