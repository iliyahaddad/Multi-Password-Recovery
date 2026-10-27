# Multi Password Recovery - Supported Targets

## Web Browsers (Complete Coverage)

### Chromium-Based Browsers
| Browser | Profiles | Encryption | Status |
|---------|----------|------------|--------|
| Google Chrome | ✅ Multi-profile | v10 AES-GCM, v20 App-Bound, Legacy DPAPI | Full |
| Microsoft Edge | ✅ Multi-profile | v10 AES-GCM, v20 App-Bound, Legacy DPAPI | Full |
| Brave Browser | ✅ Multi-profile | v10 AES-GCM | Full |
| Vivaldi | ✅ Multi-profile | v10 AES-GCM | Full |
| Yandex Browser | ✅ Multi-profile | v10 AES-GCM | Full |
| Opera / Opera GX | ✅ Multi-profile | v10 AES-GCM + Opera-specific | Full |
| Chrome Canary | ✅ Multi-profile | v10 AES-GCM | Full |
| Chromium | ✅ Multi-profile | v10 AES-GCM | Full |

**Technical:**
- Master key from `Local State` → `os_crypt.encrypted_key` (Base64 → DPAPI)
- Passwords in `Login Data` SQLite → `password_value` BLOB
- Format detection: `v10` prefix (AES-GCM), `v20` prefix (App-Bound), plain DPAPI
- SQLite copied to temp to avoid lock conflicts

### Firefox Family
| Browser | Key Storage | Encryption | Status |
|---------|-------------|------------|--------|
| Mozilla Firefox | key4.db (NSS) | PBES2-AES-256, 3DES-CBC | Full |
| Mozilla Thunderbird | key4.db (NSS) | PBES2-AES-256, 3DES-CBC | Full |
| SeaMonkey | key4.db (NSS) | PBES2-AES-256, 3DES-CBC | Full |

**Technical:**
- `key4.db` ASN.1 parsing → salt, iterations, entry salts
- PBKDF2-SHA256 with master password (or empty)
- `logins.json` for encrypted credential metadata
- Supports both modern (PBES2) and legacy (3DES) formats

---

## Windows System Credential Stores

| Store | API / Method | Credentials Recovered | Admin Required |
|-------|--------------|----------------------|----------------|
| **Credential Manager** | `CredEnumerateW` | Generic, Domain Password, Visible Password, Certificate | ❌ |
| **Windows Vault** | `vaultcli.dll` (runtime) | IE/Edge saved credentials, Web credentials | ❌ |
| **Wi-Fi Profiles** | `wlanapi.dll` | All saved SSID + passwords (WPA/WPA2/WPA3) | ✅ |
| **RAS / VPN** | `rasapi32.dll` | Dial-up, VPN, PPPoE connection passwords | ❌ |
| **LSA Secrets** | `LsaOpenPolicy` + enumeration | System secrets, cached domain creds, service passwords | ✅ |
| **System Vault** | File scan + Vault API | Machine-wide vault entries | ✅ |
| **Credential Files (.cred)** | File scan + DPAPI | Per-user DPAPI-encrypted credential blobs | ❌ |
| **Protected Storage (PStore)** | `pstorec.dll` COM | Legacy IE 4-6, Outlook Express | ❌ |

---

## Email Clients

| Client | Storage | Method |
|--------|---------|--------|
| The Bat! | Account `.ini` / registry | Config decryption |
| Becky! Internet Mail | Account files | Config parsing |
| Outlook / Outlook Express | PStore / Registry | Legacy (covered by PStore module) |

---

## FTP Clients

| Client | Files | Format |
|--------|-------|--------|
| WS_FTP | `ws_ftp.ini`, `sites.ini` | INI parsing (host/user/pass) |
| CuteFTP | `sm.dat`, `tree.dat`, `smdata.dat` | Binary structure parsing |

---

## Instant Messengers (Legacy)

| Client | Files | Notes |
|--------|-------|-------|
| Trillian | `aim.ini`, `accounts.ini` | INI parsing |
| Miranda IM | `profiles.dat` / database | Database credentials |
| ICQ 2003 | User database files | Legacy format |
| &RQ (amIRC) | Settings files | INI/registry |

---

## Other Applications

| Application | Method |
|-------------|--------|
| FileZilla | `recentservers.xml`, `sitemanager.xml` (Base64 decode) |
| WinSCP | `winscp.ini` / registry (obfuscated password decode) |
| FAR Manager | Registry / config files |
| Total Commander | `wincmd.ini` (FTP passwords) |

---

## Encryption Formats Supported

| Format | Algorithm | Where Used |
|--------|-----------|------------|
| **v10** | AES-256-GCM (nonce + ciphertext + tag) | Chrome 80+, Edge 80+, Brave, Vivaldi, Opera |
| **v20** | App-Bound Encryption (AEAD) + DPAPI fallback | Chrome 80+ with App-Bound Enabled |
| **Legacy DPAPI** | `CryptUnprotectData` | Old Chrome, Edge, .cred files, some app storage |
| **PBES2-AES-256-CBC** | PBKDF2-SHA256 → AES-CBC | Firefox 58+ (key4.db) |
| **3DES-CBC** | Triple DES-CBC | Firefox legacy, old Chrome |
| **AES-CBC** | AES-256-CBC | Firefox intermediate versions |
| **Mozilla 3DES KDF** | HMAC-SHA1 based KDF | Firefox 3DES (empty master password) |

---

## Export Capabilities

| Format | Columns | Password Visibility |
|--------|---------|---------------------|
| **CSV** | Source;Category;Target;Username;Password;Path | Respects "Show passwords" toggle |
| **TXT** | Source \| Category \| Target \| Username \| Password \| Path | Same |
| **JSON** | Array of objects with all fields | Same |

- Real-time filtering across all 6 columns
- Copy single row / username / password to clipboard
- Open containing folder for source file
- Delete individual rows from view

---

## Platform Notes

| Feature | Windows 7 | Windows 8/8.1 | Windows 10 | Windows 11 |
|---------|-----------|---------------|------------|------------|
| Credential Manager | ✅ | ✅ | ✅ | ✅ |
| Windows Vault | ✅ | ✅ | ✅ | ✅ |
| Wi-Fi Profiles | ✅ | ✅ | ✅ | ✅ |
| RAS/VPN | ✅ | ✅ | ✅ | ✅ |
| LSA Secrets | ✅ | ✅ | ✅ | ✅ |
| System Vault | ❌ | ✅ | ✅ | ✅ |
| .cred Files | ✅ | ✅ | ✅ | ✅ |
| Chrome v20 App-Bound | N/A | N/A | ✅ | ✅ |
| Firefox key4.db | ✅ | ✅ | ✅ | ✅ |

**Architecture:** 32-bit x86 (runs on WoW64 on 64-bit Windows)

---

## What Makes It Comprehensive

1. **Single binary covers everything** — no separate tools for browsers vs system vs apps
2. **Modern + Legacy** — latest Chrome v20 App-Bound through ancient ICQ 2003
3. **Real crypto implementation** — not just calling OS APIs; full AES-GCM, PBKDF2, 3DES, ASN.1 parsing
4. **Runtime DLL resolution** — works on any Windows 7-11 without specific SDK versions
5. **Threaded, responsive GUI** — scan doesn't freeze UI; results stream live
6. **Zero dependencies** — portable exe + one DLL
7. **Admin-aware** — gracefully handles elevation requirements per module
8. **Open source** — MIT licensed, fully auditable Assembly source