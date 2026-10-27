# Multi Password Recovery

A comprehensive Windows password recovery tool written in x86 Assembly (MASM32) that extracts saved credentials from web browsers, instant messengers, email clients, FTP clients, Windows Vault, Credential Manager, and more.

## Overview

**Multi Password Recovery** is a multi-purpose password recovery utility designed to recover saved passwords and credentials from various applications and Windows system stores. It is built entirely in x86 assembly language using MASM32 and linked with the Polinker.

## Supported Platforms

| Operating System | Support |
|-----------------|---------|
| Windows 7 | Yes |
| Windows 8 / 8.1 | Yes |
| Windows 10 | Yes |
| Windows 11 | Yes |

Architecture: x86 (32-bit)

## Recovered Credentials

### Web Browsers
- **Google Chrome** - Saved logins and credit cards (via Local State master key + AES-GCM/DPAPI)
- **Microsoft Edge** - Saved logins (Chromium-based)
- **Brave Browser** - Saved logins
- **Vivaldi** - Saved logins
- **Chromium** - Saved logins
- **Chrome Canary** - Saved logins
- **Yandex Browser** - Saved logins
- **Opera** - Saved logins (stable and GX)
- **Opera GX** - Saved logins
- **Mozilla Firefox** - Saved logins (key4.db NSS decryption, AES-GCM v10 format, v58+)
- **Mozilla Thunderbird** - Saved logins
- **SeaMonkey** - Saved logins

### Password Storage Formats
- **v10 format** (AES-GCM) - Firefox 58+ / Chrome v80+
- **v20 format** (App-Bound Encryption) - Chrome 80+ with fallback to DPAPI
- **3DES-CBC** - Legacy Firefox / Chrome encryption
- **AES-CBC** - Legacy Firefox encryption
- **DPAPI** - Windows Data Protection API fallback

### Instant Messengers
- **Trillian** - Profile passwords from aim.ini
- **Miranda IM** - Database credentials
- **ICQ 2003** - User database
- **&RQ (amIRC)** - Settings file

### Email Clients
- **The Bat!** - Account configuration files
- **Becky!** - Account configuration files

### FTP Clients
- **WS_FTP** - Site configuration .ini files
- **CuteFTP** - .dat files (sm.dat, tree.dat, smdata.dat)

### Windows System Stores
- **Windows Credential Manager** - Generic, domain, and visible credentials
- **Windows Vault** - IE/Edge vault entries via vaultcli.dll
- **Credential Files (.cred)** - DPAPI-encrypted credential files in %AppData%\Microsoft\Credentials\
- **LSA Secrets** - Local Security Authority secrets (system secrets, cached domain credentials)
- **System Vault** - System-level vault files in %ProgramData%\Microsoft\Vault\
- **Wi-Fi Profiles** - Saved wireless network passwords
- **RAS/VPN** - Dial-up and VPN connection credentials

### Browser Password Storage Details
- Master key extraction from Chrome `Local State` JSON (`os_crypt.encrypted_key`)
- DPAPI decryption of the master key
- AES-GCM decryption of password fields with nonce + ciphertext + tag
- SQLite database copying to temp files to avoid locked database errors
- Profile enumeration via directory scan and `profiles.ini` parsing

## Build Instructions

### Prerequisites
- [MASM32](http://www.masm32.com/) (Microsoft Macro Assembler)
- [Polinker](https://www.pelle.de/en/polink/polink.htm) (Polink linker)
- Windows SDK (for headers/libraries)

### Build
```cmd
cd C:\path\to\Multi\Password\Recovery
build.bat
```

The build script will:
1. Assemble all `.asm` source files with `ml.exe`
2. Compile the resource file with `rc.exe`
3. Link everything with `polink.exe` to produce `Multi Password Recovery.exe`

### Output
- `Multi Password Recovery.exe` - Main executable
- `sqlite3mc.dll` - Required SQLite runtime DLL (must be in same directory as exe)

## Project Structure

```
Multi Password Recovery/
├── Multi Password Recovery.exe  # Compiled executable
├── sqlite3mc.dll                # SQLite runtime library
├── build.bat                    # Main build script
├── LICENSE                      # MIT License
├── README.md                    # This file
├── .gitignore                   # Git ignore rules
├── docs/                        # Documentation (see below)
├── lib32/                       # Import libraries (pf_*.lib)
└── src/                         # Assembly source files
    ├── MultiPasswordRecovery.asm # Main entry point
    ├── core.asm                 # Core functions (DPAPI, AES, PBKDF2, SHA-1, JSON)
    ├── browsers.asm             # Browser extraction (Chrome, Firefox, etc.)
    ├── winmods.asm              # Windows modules (Credential Manager, Vault, Wi-Fi, RAS)
    ├── apps.asm                 # Application modules
    ├── json.asm                 # JSON parser
    ├── pstorage.asm             # IE Protected Storage
    ├── opera.asm                # Opera browser
    ├── ftpclients.asm           # FTP clients (WS_FTP, CuteFTP)
    ├── imclients.asm            # Instant messengers (Trillian, Miranda, ICQ, &RQ)
    ├── mailclients.asm          # Email clients (The Bat!, Becky!)
    ├── credfiles.asm            # Windows Credential Files (.cred)
    ├── lsasecrets.asm           # LSA Secrets extraction
    ├── sysvault.asm             # System Vault extraction
    ├── pv.inc                   # Common includes and prototypes
    └── MultiPasswordRecovery.rc  # Resource file (manifest)
```

## Documentation

| File | Description |
|------|-------------|
| [docs/OVERVIEW.md](docs/OVERVIEW.md) | Program overview, capabilities, why it's powerful |
| [docs/CAPABILITIES.md](docs/CAPABILITIES.md) | Complete list of supported targets and formats |
| [docs/TECHNICAL.md](docs/TECHNICAL.md) | Architecture, module map, crypto pipeline, internals |
| [docs/BUILD.md](docs/BUILD.md) | Build instructions, prerequisites, troubleshooting |

## Techniques Used

### Cryptography
- **DPAPI** (Data Protection API) - Dynamically loaded from `crypt32.dll` to avoid import library issues
- **AES-GCM** - Galois/Counter Mode for modern browser password decryption (v10 format)
- **AES-CBC** - Cipher Block Chaining for legacy encryption
- **3DES-CBC** - Triple DES for legacy Firefox/Chrome encryption
- **PBKDF2-SHA256** - Password-Based Key Derivation Function 2 for Firefox master key
- **HMAC-SHA1** - Hash-based Message Authentication Code for Mozilla 3DES KDF
- **SHA-1** - Secure Hash Algorithm 1

### Windows API
- `CryptUnprotectData` - DPAPI decryption (dynamically loaded)
- `LsaOpenPolicy` / `LsaEnumerateSecrets` / `LsaGetSecret` - LSA secrets
- `VaultEnumerateVaults` / `VaultOpenVault` / `VaultGetItem` - Windows Vault
- `CredEnumerate` / `CredRead` - Credential Manager
- `WlanOpenHandle` / `WlanGetProfile` - Wi-Fi profiles
- `RegReadStr` - Registry value reading
- `GetPrivateProfileString` - INI file parsing

### Anti-Locking
- SQLite databases are copied to temp files before reading to avoid "database is locked" errors when the browser is running

## Limitations

1. **Console vs GUI context**: DPAPI decryption may hang when run from a console application. The GUI version (`Multi Password Recovery.exe`) works correctly as it has a proper message pump.

2. **Chrome v80+ App-Bound Encryption**: The tool attempts to disable App-Bound Encryption by setting the registry key `HKLM\SOFTWARE\Policies\Google\Chrome\ApplicationBoundEncryptionEnabled = 0`. If this fails (requires admin), passwords encrypted with v20 format may not be recoverable.

3. **Firefox v10 (AES-GCM)**: Supported for Firefox 58+ with the master key from `key4.db`.

4. **32-bit only**: Currently compiled as x86 only.

## Legal Disclaimer

This tool is provided for educational and legitimate password recovery purposes only. You should only use it to recover passwords from accounts you own or have explicit authorization to access. The authors are not responsible for any misuse or damage caused by this software.

## License

see LICENSE file for details.

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## Contact

For questions or issues, please open a GitHub issue.
