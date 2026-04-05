# Security Policy

## Overview

AdminToolKit runs with Administrator privileges and executes PowerShell scripts
locally and against remote machines via WinRM. This document describes the
security model, known considerations, and how to report vulnerabilities.

---

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.10    | Yes       |
| < 1.10  | No        |

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
- Does not modify the system-wide or user-level execution policy
- Is not persistent across reboots or sessions
- Does not affect other PowerShell sessions running on the machine

---

### Remote Credential Handling

When alternate credentials are provided for remote execution:

- The username and password are held **in memory only** for the duration of
  the session
- Credentials are **never written to disk**, logged, or stored in the registry
- The password is converted to a `PSCredential` object with a `SecureString`
  immediately before the `Invoke-Command` call
- The `RemoteScriptsViewModel` holds the password as a plain `string` property
  while the Remote Target Bar is open — this is a known limitation. Clear the
  credential fields or close the session when done.
- Credentials are not persisted between application restarts

**Recommendation:** Use Windows integrated authentication (Kerberos/Negotiate)
wherever possible. Only use the alternate credentials feature for non-domain
machines or cross-domain scenarios.

---

### WinRM / Remote Execution

Remote scripts run via `Invoke-Command -ComputerName $TargetHost`. The
following WinRM configuration is required and should be understood:

- **TrustedHosts set to `*`** in the setup guides allows connections to any
  hostname without certificate validation. In production environments, restrict
  TrustedHosts to specific IP ranges or use HTTPS with a valid certificate.
- Remote scripts run in the security context of the connecting user (or the
  alternate credential if provided). They do not gain additional privileges on
  the remote machine beyond what that user account has.
- All remote commands are executed over WinRM (port 5985 HTTP by default).
  Consider enabling WinRM over HTTPS (port 5986) in sensitive environments.

---

### Script Execution

AdminToolKit executes `.ps1` scripts stored in the `Scripts\` subfolder of the
installation directory. These scripts are:

- Installed to `C:\Program Files\AdminToolKit\Scripts\` by the installer
- Readable (but not writable) by standard users after installation
- Not signed — they rely on the process-scoped Bypass policy described above

**Risk:** If an attacker can write to the `Scripts\` installation folder, they
could replace scripts with malicious versions that execute with Administrator
privileges when a user runs AdminToolKit. Ensure the installation directory
retains its default ACL (Administrators: Full Control, Users: Read & Execute).

---

### CmRcViewer / Remote Control

The Remote Control feature launches `CmRcViewer.exe` (SCCM Remote Control
Viewer), a proprietary Microsoft/ConfigMgr binary:

- It is passed the target hostname directly as a command-line argument
- AdminToolKit does not intercept, log, or modify the remote session
- The SCCM Remote Tools Operator role is required on the connecting account
- Remote Control sessions are subject to SCCM's own audit logging

---

### Active Directory Operations

AD scripts use the `ActiveDirectory` PowerShell module (RSAT). Operations
performed (password resets, account disables, group modifications) are:

- Executed in the security context of the logged-in user
- Logged by Active Directory's own audit infrastructure (if auditing is enabled)
- **Irreversible in some cases** — Disable Account and Delete Profile operations
  cannot be undone from within AdminToolKit

---

### Sensitive Data in Output

The script output console may display:

- Usernames, email addresses, phone numbers (from AD queries)
- Last logon times, group memberships
- IP addresses, MAC addresses, DNS records
- Installed software names and versions

Use **File → Save Output** with care. Output files are saved in plain text
with no encryption. Do not leave output files containing sensitive user data
in shared or unprotected locations.

**The "Copy to Clipboard" function** copies all current output as plain text.
Be mindful when pasting into ticketing systems, emails, or chat tools.

---

## Known Limitations

| Area | Limitation |
|------|------------|
| Credential storage | Password held as plain `string` in `RemoteScriptsViewModel` while the session is open |
| WinRM TrustedHosts | Setup guides use `*` — restrict in production |
| Script signing | `.ps1` files are not Authenticode-signed |
| No MFA support | Alternate credentials use username/password only |
| No audit log | AdminToolKit itself does not log which scripts were run or by whom |

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

1. **Restrict TrustedHosts** — replace `*` with specific IP ranges or hostnames
2. **Enable WinRM HTTPS** — use port 5986 with a valid certificate to encrypt
   remote traffic
3. **Use dedicated service accounts** — run AdminToolKit under a dedicated admin
   account, not your personal domain admin credentials
4. **Enable AD auditing** — ensure Active Directory audit policies log password
   resets, account lockouts, and group membership changes
5. **Protect the install directory** — verify `C:\Program Files\AdminToolKit\`
   is not writable by standard users
6. **Review scripts before deployment** — read the `.ps1` files before deploying
   in sensitive environments; all scripts are open source and auditable
7. **Use application allowlisting** — add AdminToolKit.exe and its scripts to
   your allowlist rather than disabling execution policy system-wide
