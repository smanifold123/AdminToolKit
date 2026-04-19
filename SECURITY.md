# Security Policy

## Overview

AdminToolKit runs with Administrator privileges and executes PowerShell scripts
locally and against remote machines via WinRM. This document describes the
security model, known considerations, and how to report vulnerabilities.

A formal CISO-level security review was conducted prior to v1.12. All findings
were remediated. See [CISO Review Findings (v1.12)](#ciso-review-findings-v112)
for the full record.

---

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.13    | Yes       |
| 1.12    | Yes       |
| < 1.12  | No        |

---

## Security Model

### Privilege Elevation

AdminToolKit requests elevation via UAC on launch (`app.manifest` requires
`requireAdministrator`). This is necessary for:

- Reading the Security event log (Last Logins)
- Managing Windows services (Restart Spooler, Fix WMI)
- Modifying system files and registry (Fix Windows Update, Clean SCCM)
- Running DISM and SFC repairs

**The app does not bypass UAC.** If the user declines the UAC prompt,
the application will not start.

---

### PowerShell Execution Policy

AdminToolKit sets `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`
for the **current process only** before running each script. This change:

- Applies only to the PowerShell runspace created for that script execution
- Does **not** modify the system-wide or user-level execution policy
- Is not persistent across reboots or sessions
- Does not affect other PowerShell sessions running on the machine

For remote executions, a dedicated `Runspace` is opened via
`RunspaceFactory.CreateRunspace()` and the bypass is applied to that runspace
only before the user script is loaded. The runspace is disposed after execution.

---

### Remote Credential Handling

When alternate credentials are provided for remote execution:

- The password is stored as `System.Security.SecureString` in
  `RemoteScriptsViewModel` — never as a plain `string`
- The `PasswordBox` in the UI syncs to `SecureString` via `SetPassword()` on
  every keystroke; the plain password is never accessible via a property
- Credentials are passed to the PowerShell engine via
  `Runspace.SessionStateProxy.SetVariable("__cred", psCred)` — a
  `PSCredential` object is constructed from the `SecureString` directly and
  injected into the runspace. **The password never appears as a string literal
  in script text at any point.**
- The `SecureString` is cleared and disposed immediately after each script run
- Credentials are **never written to disk**, logged, or stored in the registry
- Credentials are not persisted between application restarts

**Recommendation:** Use Windows integrated authentication (Kerberos/Negotiate)
wherever possible. Only use the alternate credentials feature for non-domain
machines or cross-domain scenarios.

---

### Input Validation

All user-supplied input is validated and sanitised before use in script
execution:

**Hostname validation** (`ValidateHostname()` in `PowerShellRunner.cs`):
- Strict allowlist: letters, digits, hyphens, dots, and underscores only
- Maximum 253 characters (RFC 1035 DNS limit)
- Any hostname failing validation blocks script execution before it starts
- An error line is written to the output panel with no script executed

**Script parameter injection** (`InjectParameters()` in `PowerShellRunner.cs`):
- Parameter values are single-quote escaped (`'` → `''`) before injection
- Parameter names (variable names) are stripped of all non-alphanumeric and
  non-underscore characters via `EscapeVarName()`

---

### Audit Logging

Every script execution is written to an append-only audit log at:

```
%ProgramData%\AdminToolKit\audit.log
```

Each log entry contains:

| Field | Description |
|-------|-------------|
| Timestamp | ISO-8601 UTC (`yyyy-MM-ddTHH:mm:ssZ`) |
| User | Windows identity (`DOMAIN\username`) |
| Machine | Source machine hostname |
| Category | `Local` / `Remote` / `ActiveDirectory` |
| Script | Script name |
| Target | Target hostname, or `LOCAL` for local scripts |
| Result | `SUCCESS` or `FAILURE` |
| Duration | Execution time in seconds |
| Error | First error message if result is FAILURE |

The log uses pipe (`|`) as a field delimiter. Pipe characters in field values
are replaced with `/` to prevent log injection. Log writes use a
`SemaphoreSlim` lock to prevent race conditions on concurrent executions.
Logging failures are silently swallowed — they can never crash the application.

The log is accessible via **File → View Audit Log** in the application menu.

---

### WinRM / Remote Execution

Remote scripts run via `Invoke-Command -ComputerName $TargetHost`. The
following WinRM configuration is required and should be understood:

- **TrustedHosts set to `*`** in the lab setup guides is convenient but not
  recommended for production — it permits connections to any hostname without
  certificate validation. Restrict TrustedHosts to specific IP ranges in
  production environments (see [Hardening Recommendations](#hardening-recommendations)).
- Remote scripts run in the security context of the connecting user (or the
  alternate credential if provided). They do not gain additional privileges on
  the remote machine beyond what that account has.
- All remote commands are executed over WinRM (port 5985 HTTP by default).
  Enable WinRM over HTTPS (port 5986) in sensitive environments.

---

### Script Execution

AdminToolKit executes `.ps1` scripts stored in the `Scripts\` subfolder of the
installation directory. These scripts are:

- Installed to `C:\Program Files\AdminToolKit\Scripts\` by the installer
- Readable (but not writable) by standard users after installation
- Not Authenticode-signed — they rely on the process-scoped Bypass policy
  described above

**Risk:** If an attacker can write to the `Scripts\` installation folder, they
could replace scripts with malicious versions that execute with Administrator
privileges. Ensure the installation directory retains its default ACL
(Administrators: Full Control, Users: Read & Execute).

`Build-Installer.bat` supports optional Authenticode signing of the installer
itself via the `SIGN_CERT` variable — see [Code Signing](#code-signing).

---

### CmRcViewer / Remote Control

The Remote Control feature launches `CmRcViewer.exe` (SCCM Remote Control
Viewer), a proprietary Microsoft/ConfigMgr binary:

- The target hostname is resolved from registry
  (`HKLM\SOFTWARE\WOW6432Node\Microsoft\ConfigMgr10\AdminUI`) before falling
  back to the hard-coded default path — it is not constructed from user input
- AdminToolKit does not intercept, log, or modify the remote session
- The SCCM Remote Tools Operator role is required on the connecting account
- Remote Control sessions are subject to SCCM's own audit logging

---

### Remote Desktop (RDP)

The Remote Desktop feature launches `mstsc.exe` with the validated target
hostname. If alternate credentials are configured:

- Credentials are temporarily stored in Windows Credential Manager via
  `cmdkey /generic:"TERMSRV/hostname"` to allow mstsc to connect without
  prompting
- A background PowerShell job automatically removes the stored credentials
  from Credential Manager after 15 seconds
- The plain-text password is extracted from `SecureString` only within the
  local PowerShell process for the `cmdkey` call, and the reference is
  immediately zeroed after use

---

### Active Directory Operations

AD scripts use the `ActiveDirectory` PowerShell module (RSAT). Operations
performed (password resets, account disables, group modifications) are:

- Executed in the security context of the logged-in user
- Logged by Active Directory's own audit infrastructure (if auditing is enabled)
- Logged by AdminToolKit's own `AuditLogger` with user, target, and result
- **Irreversible in some cases** — Disable Account and Delete Profile operations
  cannot be undone from within AdminToolKit

---

### Sensitive Data in Output

The script output console may display:

- Usernames, email addresses, phone numbers (from AD queries)
- Last logon times, group memberships
- IP addresses, MAC addresses, DNS records
- Installed software names and versions

Use **File → Save Output** and **Export HTML** with care. Output files are
saved in plain text or HTML with no encryption. Do not leave output files
containing sensitive user data in shared or unprotected locations.

The **Copy to Clipboard** function copies all current output as plain text.
Be mindful when pasting into ticketing systems, emails, or chat tools.

---

## CISO Review Findings (v1.12)

A formal security review was conducted against the codebase before internal
deployment approval. Six findings were identified and all were remediated in
v1.12. The findings and resolutions are summarised below.

| # | Finding | Severity | Status |
|---|---------|----------|--------|
| 1 | Plain-text password embedded as string literal in PowerShell script text | **HIGH** | ✅ Remediated |
| 2 | No audit trail of administrative actions | **HIGH** | ✅ Remediated |
| 3 | Hostname injection attack surface (insufficient input validation) | **MEDIUM** | ✅ Remediated |
| 4 | Script parameter variable names not sanitised | **MEDIUM** | ✅ Remediated |
| 5 | Execution policy bypass not clearly scoped/documented | **LOW** | ✅ Remediated |
| 6 | WinRM TrustedHosts = `*` in setup guidance | **LOW** | ✅ Documented |

### Finding 1 — Plain-text password in script text (HIGH)

**Before:** `PowerShellRunner.ExecuteRemote()` built the credential injection
block as a string literal:
```powershell
$__pass = ConvertTo-SecureString 'PLAINTEXT_PASSWORD' -AsPlainText -Force
```
This meant the password existed verbatim in managed heap memory as part of
the script string and would appear in any heap dump or crash report.

**After:** A `Runspace` is created via `RunspaceFactory.CreateRunspace()`. A
`PSCredential` is constructed directly from the `SecureString` and injected
via `Runspace.SessionStateProxy.SetVariable("__cred", psCred)`. The password
never appears in script text. File: `Helpers\PowerShellRunner.cs`.

---

### Finding 2 — No audit trail (HIGH)

**Before:** No logging of any kind. Impossible to answer "who ran what, against
which machine, and when" in a GDPR data subject request or ISO 27001 audit.

**After:** `AuditLogger.cs` writes an append-only structured log to
`%ProgramData%\AdminToolKit\audit.log` on every script execution. All fields
are sanitised to prevent log injection. Files: `Helpers\AuditLogger.cs`,
`Helpers\PowerShellRunner.cs`, `Views\MainWindow.xaml`, `Views\MainWindow.xaml.cs`.

---

### Finding 3 — Hostname injection (MEDIUM)

**Before:** Target hostname was injected into script text with only
single-quote escaping, leaving PowerShell metacharacters as an attack surface.

**After:** `ValidateHostname()` applies a strict allowlist (letters, digits,
hyphens, dots, underscores only, max 253 chars). Invalid input blocks
execution before any script text is constructed. File: `Helpers\PowerShellRunner.cs`.

---

### Finding 4 — Parameter name sanitisation (MEDIUM)

**Before:** Script parameter names were used as PowerShell variable names
without sanitisation — only values were escaped.

**After:** `EscapeVarName()` strips all non-alphanumeric and non-underscore
characters from parameter names before injection. File: `Helpers\PowerShellRunner.cs`.

---

### Finding 5 — Execution policy bypass scope (LOW)

**Before:** Bypass was applied via `ps.AddScript()` with no documentation of
scope.

**After:** `SetBypassPolicyOnRunspace()` applies the bypass to the dedicated
runspace only, with explicit documentation that this is `Scope Process` and
does not affect system-wide policy. File: `Helpers\PowerShellRunner.cs`.

---

### Finding 6 — WinRM TrustedHosts = `*` (LOW)

**Before:** Setup documentation recommended `TrustedHosts = *` without
production caveats.

**After:** SECURITY.md and setup guides updated with explicit production
hardening recommendation to restrict TrustedHosts to specific subnets and
to use WinRM over HTTPS. File: `SECURITY.md`.

---

## Known Limitations

| Area | Limitation |
|------|------------|
| Audit log integrity | Log file is append-only but not cryptographically signed — a local Administrator could modify it |
| No MFA support | Alternate credentials use username/password only; no FIDO2 or smart card support |
| Script signing | `.ps1` files are not Authenticode-signed; integrity relies on install directory ACL |
| RDP credential window | Credentials exist in Credential Manager for up to 15 seconds during RDP session launch |
| WinRM transport | HTTP (port 5985) used by default — enable HTTPS in production |

---

## Code Signing

The official `AdminToolKit-Setup.exe` installer is unsigned by default.
Enterprise environments may encounter a Windows SmartScreen warning
("Unknown publisher") on first run.

To sign the installer with your organisation's code-signing certificate:

1. Set `SIGN_CERT=YOUR_CERT_THUMBPRINT` in `Build-Installer.bat`
2. Set `SIGN_TSA` to your timestamp authority URL (e.g. `http://timestamp.digicert.com`)
3. Run `Build-Installer.bat` — `signtool.exe` runs automatically after NSIS

Obtaining a trusted certificate requires purchasing one from a Certificate
Authority (DigiCert, Sectigo, GlobalSign). Self-signed certificates do not
eliminate the SmartScreen warning.

---

## Reporting a Vulnerability

If you discover a security vulnerability in AdminToolKit, please **do not open
a public GitHub issue**.

Report vulnerabilities by:

1. Opening a **GitHub Security Advisory** (Repository → Security → Advisories →
   New draft security advisory)
2. Or emailing the repository owner directly via the contact on their GitHub profile

Please include:

- A description of the vulnerability and its potential impact
- Steps to reproduce
- Affected version(s)
- Any suggested mitigation or fix

You can expect an initial response within **5 business days**. We will work
with you to understand and address the issue before any public disclosure.

---

## Hardening Recommendations

For organisations deploying AdminToolKit in production environments:

1. **Restrict TrustedHosts** — replace `*` with specific IP ranges or
   hostnames, e.g. `Set-Item WSMan:\localhost\Client\TrustedHosts -Value "10.0.1.*"`
2. **Enable WinRM HTTPS** — use port 5986 with a valid certificate to encrypt
   all remote traffic; prevents credential interception on untrusted networks
3. **Sign the installer** — use `Build-Installer.bat` with `SIGN_CERT` set to
   eliminate SmartScreen warnings and verify installer integrity
4. **Use dedicated service accounts** — run AdminToolKit under a dedicated
   admin account, not personal domain admin credentials; limits blast radius
   if credentials are compromised
5. **Enable AD auditing** — ensure Active Directory audit policies log password
   resets, account lockouts, and group membership changes independently of
   AdminToolKit's own log
6. **Protect the install directory** — verify `C:\Program Files\AdminToolKit\`
   is not writable by standard users; prevents script replacement attacks
7. **Review the audit log regularly** — `%ProgramData%\AdminToolKit\audit.log`
   should be ingested into your SIEM or reviewed periodically for anomalous
   activity (unexpected targets, failure spikes, out-of-hours usage)
8. **Use application allowlisting** — add `AdminToolKit.exe` to your allowlist
   rather than disabling execution policy system-wide; enforce via AppLocker
   or WDAC
9. **Rotate credentials** — if alternate credentials are used, treat them as
   privileged service account credentials and rotate per your PAM policy
