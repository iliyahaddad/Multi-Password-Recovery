# Multi Password Recovery - Technical Reference

## Module Map

```
MultiPasswordRecovery.asm    → GUI, message loop, scan thread orchestration
        │
        ├── core.asm          → Item store, crypto, DPAPI, BCrypt, SQLite, ASN.1, encoding
        ├── json.asm          → JSON field parser (Local State, logins.json)
        ├── browsers.asm      → Chromium + Firefox families (all variants)
        ├── winmods.asm       → Credential Manager, Vault, Wi-Fi, RAS/VPN, LSA, PStore
        ├── apps.asm          → Email, FTP, IM, legacy app modules
        ├── pstorage.asm      → IE Protected Storage (legacy)
        ├── opera.asm         → Opera/Opera GX specific handling
        ├── ftpclients.asm    → WS_FTP, CuteFTP
        ├── imclients.asm     → Trillian, Miranda, ICQ, &RQ
        ├── mailclients.asm   → The Bat!, Becky!
        ├── credfiles.asm     → Windows .cred files
        ├── lsasecrets.asm    → LSA Secrets enumeration
        ├── sysvault.asm      → System Vault (%ProgramData%\Microsoft\Vault\)
        └── sqlite3mc.dll     → Runtime SQLite (loaded dynamically)
```

---

## Core Data Structures

### PVITEM (per credential record)
```
szApp[64]     → Source application (e.g., "Chrome", "Credential Manager")
szCat[48]     → Category (e.g., "Browser", "Wi-Fi", "Domain Password")
szUser[384]   → Username / identity
szPass[384]   → Password / secret
szTarget[512] → URL, server, SSID, or target identifier
szPath[264]   → Source file/registry path
dwDate        → Timestamp (reserved)
```
**Max items:** 20,000 (configurable via `MAX_ITEMS` in `pv.inc`)

### PVROW (SQLite row buffer)
```
nCols         → Column count
pCol[32]      → Column text pointers
pBlob[4]      → BLOB column pointers
nBlob[4]      → BLOB sizes
```

---

## Crypto Pipeline (core.asm)

| Function | Algorithm | Use Case |
|----------|-----------|----------|
| `PV_DpapiDecrypt` | DPAPI (`CryptUnprotectData`) | Chromium master key, .cred files |
| `PV_AesGcmDecrypt` | AES-256-GCM | Chrome/Edge v10+ passwords |
| `PV_AesCbcDecrypt` | AES-256-CBC | Legacy Chrome/Firefox |
| `PV_3DesCbcDecrypt` | 3DES-CBC | Firefox 3DES, old Chrome |
| `PV_Pbkdf2Sha256` | PBKDF2-HMAC-SHA256 | Firefox master key |
| `PV_HmacSha1` | HMAC-SHA1 | Mozilla 3DES KDF |
| `PV_Sha1` | SHA-1 | Hash computation |

All BCrypt functions resolved at runtime via `LoadLibrary("bcrypt.dll")` + `GetProcAddress`.

---

## Browser Decryption Flow

### Chromium Family (Chrome, Edge, Brave, Vivaldi, Opera, Yandex...)
```
1. Read %LOCALAPPDATA%\<vendor>\User Data\Local State (JSON)
2. Extract "os_crypt.encrypted_key" (Base64 → DPAPI blob)
3. PV_DpapiDecrypt → 32-byte AES master key
4. For each profile: copy Login Data → temp file
5. SQLite: SELECT origin_url, username_value, password_value
6. For each password_value:
   - v10 (AES-GCM): nonce(12) + ciphertext + tag(16) → PV_AesGcmDecrypt
   - v20 (App-Bound): attempt registry disable → fallback to DPAPI
   - Legacy: DPAPI direct decrypt
```

### Firefox Family (Firefox, Thunderbird, SeaMonkey)
```
1. Locate profiles.ini → profile directories
2. Parse key4.db (ASN.1 NSS DB):
   - Extract salt, iteration count, entry salt
   - PBKDF2-SHA256(master_password, salt, iterations) → AES key
   - Or empty password path (known salt)
3. Parse logins.json (JSON) for encrypted credentials
4. Decrypt via:
   - PBES2-AES-256-CBC (modern)
   - 3DES-CBC (legacy)
```

---

## Windows System Modules (winmods.asm)

| Module | API | Data Source |
|--------|-----|-------------|
| `ScanCredManager` | `CredEnumerateW`, `CredRead`, `CredFree` | Credential Manager |
| `ScanVault` | `VaultEnumerateVaults`, `VaultOpenVault`, `VaultGetItem` | Windows Vault |
| `ScanWlan` | `WlanOpenHandle`, `WlanEnumInterfaces`, `WlanGetProfile` | Wi-Fi profiles |
| `ScanRas` | `RasEnumEntries`, `RasGetEntryDialParams` | RAS/VPN phonebook |
| `ScanLSASecrets` | `LsaOpenPolicy`, `LsaEnumerateSecrets`, `LsaGetSecret` | LSA Secrets |
| `ScanSystemVault` | File enumeration + Vault API | `%ProgramData%\Microsoft\Vault\` |
| `ScanPStore` | `pstorec.dll` COM | Protected Storage (legacy) |

---

## Build System

### Requirements
- **MASM32** at `C:\masm32` (ml.exe, polink.exe, rc.exe, includes, libs)
- **Windows SDK** (for additional headers if needed)

### build.bat Flow
```batch
1. Assemble all .asm → .obj (ml.exe /c /coff)
2. Compile resource → mpr.res (rc.exe)
3. Link with Pelle Polink:
   - All .obj + mpr.res
   - lib32/pf_*.lib (11 Pelle import libraries)
   - masm32.lib
4. Output: Multi Password Recovery.exe
```

### Pelle Import Libraries (lib32/)
Generated from MASM32 `.inc` prototypes via `genlibs.ps1`:
```
pf_kernel32.lib   pf_user32.lib   pf_gdi32.lib
pf_advapi32.lib   pf_comctl32.lib pf_shell32.lib
pf_crypt32.lib    pf_shlwapi.lib  pf_rasapi32.lib
pf_comdlg32.lib   pf_ws2_32.lib
```
**Decoration:** `_Name@n=Name` (critical for loader resolution)

---

## Runtime Dependencies

### Shipped
- `sqlite3mc.dll` — 32-bit SQLite with multiple cipher support (2 MB)

### OS-Provided (Loaded Dynamically)
- `bcrypt.dll` — AES, 3DES, SHA, PBKDF2, HMAC
- `crypt32.dll` — `CryptUnprotectData` (DPAPI)
- `vaultcli.dll` — Windows Vault APIs
- `wlanapi.dll` — Wi-Fi profile APIs
- `advapi32.dll`, `rasapi32.dll`, `shell32.dll`, `shlwapi.dll`, `comctl32.dll`, `comdlg32.dll`, `user32.dll`, `kernel32.dll`, `gdi32.dll`

---

## Key Implementation Details

### Threading Model
- **Main thread**: GUI message loop (WinMain → WndProc)
- **Scan thread**: `ScanThreadProc` → `ScanAll` → all modules → `PostMessage(WM_APP+2)` per batch
- **UI updates**: `WM_APP+2` (flush rows), `WM_APP+11` (scan done)
- **Synchronization**: `CRITICAL_SECTION` around `PV_Add` / item array

### Anti-Locking Strategy
```asm
PV_SqlOpenCopy:
    1. PV_CopyFileToTemp(src, temp_path)  ; CopyFileEx
    2. sqlite3_open_v2(temp_path, ..., SQLITE_OPEN_READONLY)
    3. On close: DeleteFile(temp_path)
```

### JSON Parser (json.asm)
- `PV_JsonFind(json, start, key, out)` — single-purpose, no recursion
- Handles escapes: `\" \\ \n \t \r \uXXXX`
- Returns first match only (optimized for `Local State` / `logins.json`)

### ASN.1 Parser (core.asm)
- `AsnRead(ptr)` → sets `g_asn_tag`, `g_asn_pv`, `g_asn_len`, `g_asn_hdr`
- Used for Firefox `key4.db` NSS database navigation
- Supports SEQUENCE, OCTET STRING, INTEGER, OID tags

---

## Adding New Modules

1. Add `ScanNewModule proto` to `pv.inc`
2. Implement in new `.asm` (or existing apps/winmods)
3. Call `PV_Add(app, cat, user, pass, target, path)` for each credential
4. Add to `ScanAll` in `winmods.asm`
5. Add `.obj` to `build.bat` link line
6. Module must be **failure-tolerant**: check all pointers, return silently on missing data

---

## Export Formats

| Format | Delimiter | Password Handling |
|--------|-----------|-------------------|
| CSV | `;` | Masked unless "Show passwords" active |
| TXT | ` \| ` | Same as CSV |
| JSON | JSON array | Same as CSV |

Passwords in memory are always stored **unmasked**. UI masks via `MaskStr` (replaces chars with `*`). Export writes actual values when user explicitly exports.

---

## Error Handling Philosophy

- **Never crash** on missing file/regkey/DLL
- **Log nothing** to disk (no debug logs in production)
- **Return empty** item set on failure
- **Continue** scanning other modules
- **UI stays alive** regardless of scan errors