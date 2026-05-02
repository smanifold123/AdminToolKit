# AdminToolKit

**A WPF + Fluent.Ribbon PowerShell administration suite for Windows sysadmins.**

Built with .NET Framework 4.8, Windows PowerShell 5.1, and Fluent.Ribbon 9.x.
Industrial dark UI with light/dark theme switching. Three ribbon tabs. 57 built-in scripts. Easy to extend.

---

## Features

- **Local PC tab** — system info, maintenance, network diagnostics, security auditing, and repair tools
- **Remote PC tab** — run the same maintenance and fix scripts against any WinRM-enabled machine
- **Active Directory tab** — user/group/computer management and reporting tools
- **Theme switching** — light and dark themes switchable at runtime via File menu
- **Script output panel** — colour-coded, auto-scrolling console with Save Output and Copy to Clipboard
- **Parameterised scripts** — buttons that need input prompt the user before running
- **Remote credentials** — optional alternate credential support for non-domain remote machines
- **SCCM Remote Control** — one-click Remote Control viewer (CmRcViewer.exe) for the target PC via SCCM/ConfigMgr
- **Remote Desktop (RDP)** — launch an RDP session to the target PC with optional credential pass-through via Windows Credential Manager
- **Multiple remote targets** — run any remote script against a comma-separated list of PCs in one click
- **Saved target profiles** — save and reload frequently-used hostnames and credentials across sessions
- **Script timeout & cancel** — configurable timeout (default 30s) with a Cancel button to stop hung scripts
- **Output search/filter** — real-time filter box above the console to search long output
- **HTML export** — save colour-coded output as an HTML file for sharing and reporting

---

## Prerequisites

| Requirement               | Version / Notes                                    |
|---------------------------|----------------------------------------------------|
| Windows OS                | Windows 10 / Windows 11 / Windows Server 2016+     |
| .NET Framework            | 4.8 (pre-installed on Win10 1903+)                 |
| Windows PowerShell        | 5.1 (pre-installed on all modern Windows)          |
| Visual Studio             | 2019 or 2022 (Community edition is fine)           |
| Fluent.Ribbon NuGet       | 9.0.3 (restored automatically on build)            |
| RSAT AD Tools *(AD tab)*  | Optional — needed for Active Directory scripts     |
| WinRM *(Remote tab)*      | Enabled on target PCs for remote execution         |
| Visual C++ 2013 x86 runtime | Required by CmRcViewer.exe — installed automatically by the AdminToolKit installer (`vcredist_x86.exe`). Download: https://aka.ms/highdpimfc2013x86enu |
| SCCM Admin Console *(Remote Control)* | CmRcViewer.exe from ConfigMgr Admin Console — installed to `C:\Program Files (x86)\Microsoft Configuration Manager\AdminConsole\bin\i386\` |

---

## Getting Started

### 1. Clone / download the project

```
AdminToolKit/
├── AdminToolKit.sln
└── AdminToolKit/
    ├── AdminToolKit.csproj
    ├── admintoolkit.ico
    ├── packages.config
    ├── app.manifest
    ├── App.xaml / App.xaml.cs
    ├── Models/
    ├── ViewModels/
    ├── Views/
    │   ├── MainWindow.xaml / .cs
    │   ├── Controls/
    │   │   ├── ScriptOutputPanel.xaml / .cs
    │   │   └── RemoteTargetBar.xaml / .cs
    │   └── Dialogs/
    ├── Helpers/
    ├── Themes/
    │   ├── LightTheme.xaml
    │   └── DarkTheme.xaml
    └── Scripts/
        ├── Local/
        ├── Remote/
        └── ActiveDirectory/
```

### 2. Restore NuGet packages

Keep the `packages\` folder alongside the solution, or restore via:

```powershell
nuget restore AdminToolKit.sln
```

### 3. Build and run

Run `Build.bat`, or open in Visual Studio and press **F5**.
The app requests elevation via UAC — required for security event log access, service management, and repair tools.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      MainWindow.xaml                        │  ← Fluent Ribbon UI
│   Tab 1: Local PC  │  Tab 2: Remote PC  │  Tab 3: AD        │
└──────────────────────┬──────────────────────────────────────┘
                       │ button click → script name (Tag)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│         LocalVM / RemoteVM / ActiveDirectoryVM              │  ← ViewModels
│         Look up ScriptDefinition in ScriptRegistry          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   PowerShellRunner                          │  ← Execution engine
│   RunLocalAsync()  /  RunRemoteAsync()                      │
│   All output collected post-Invoke, delivered in one batch  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│               ScriptOutputPanel (console)                   │  ← Auto-scrolling output
│   Colour-coded: Info / Success / Warning / Error            │
└─────────────────────────────────────────────────────────────┘
```

### Key files

| File | Purpose |
|------|---------|
| `Helpers/ScriptRegistry.cs` | **Central script catalogue** — add new scripts here |
| `Helpers/PowerShellRunner.cs` | Executes scripts locally or via WinRM |
| `Helpers/OutputLine.cs` | Typed output lines with colour metadata |
| `Helpers/ThemeManager.cs` | Runtime light/dark theme switching |
| `Models/ScriptDefinition.cs` | Describes a script (name, path, parameters, flags) |
| `Views/MainWindow.xaml` | Ribbon layout (tabs, groups, buttons) |
| `Views/Controls/RemoteTargetBar.xaml` | Hostname input and connection status bar |
| `Themes/DarkTheme.xaml` | Dark colour palette and control styles |
| `Themes/LightTheme.xaml` | Light colour palette and control styles |

---

## How to Add a New Script

Adding a script requires changes in **two places only**.

### Step 1 — Register the script (`ScriptRegistry.cs`)

```csharp
new ScriptDefinition
{
    Name        = "My New Script",
    GroupName   = "My Group",
    Description = "What this script does.",
    IconKey     = "IconNetwork",
    ScriptPath  = @"Scripts\Local\My-Script.ps1",
    Parameters  = new Dictionary<string, string>
    {
        { "ComputerName", "Enter computer name" }
    }
}
```

### Step 2 — Add the ribbon button (`MainWindow.xaml`)

```xml
<fluent:Button Header="My New Script"
               LargeIcon="{StaticResource IconNetwork}"
               ToolTip="What this script does."
               Click="LocalScript_Click"
               Tag="My New Script" />
```

> The `Tag` value must exactly match `ScriptDefinition.Name`.

### Step 3 — Create the `.ps1` file

Drop your script at `Scripts\Local\My-Script.ps1`. Parameters declared in `ScriptRegistry.cs` are injected as PowerShell variables automatically:

```powershell
# $ComputerName is available here — injected by the runner
Write-Host "Running on: $ComputerName"
```

> **Important:** All scripts must use `Write-Output` only. Never use `Format-Table` or `Format-List` — these produce `FormatEntryData` objects that display as type names in the output panel. For remote scripts (inside `Invoke-Command` scriptblocks), `Write-Host` sends output to the remote console and is never returned to the caller — use `Write-Output` exclusively.

---

## Remote PC Tab — WinRM Setup

WinRM must be enabled on each target machine before remote scripts will work.

**On each target PC (run as administrator):**

```powershell
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
```

**On your admin PC:**

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
```

Use **alternate credentials** in the Remote Host dialog for non-domain machines.

---

## Active Directory Tab — RSAT Setup

AD scripts require the Active Directory PowerShell module (RSAT).

**Windows 10/11 via Settings → Optional Features:**
Install *RSAT: Active Directory Domain Services and Lightweight Directory Tools*

**Or via PowerShell:**

```powershell
Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
```

---

## Built-in Scripts Reference

### 💻 Local PC Tab

#### System Information

| Button | Description |
|--------|-------------|
| System Info | Detailed hardware, OS, BIOS, RAM, and uptime |
| Disk Usage | Free/used space for all local drives |
| Installed Software | Full application inventory from the registry (32-bit and 64-bit) |

#### Maintenance

| Button | Parameters | Description |
|--------|-----------|-------------|
| Flush DNS | — | Flush the local DNS resolver cache |
| Clear Temp | — | Report disk space usage across Temp, caches, SoftwareDistribution, browser caches, CCM cache, and Recycle Bin |
| Restart Spooler | — | Stop and restart the Print Spooler service to fix stuck print jobs |
| Intune Sync | — | Trigger an Intune policy sync via the PushLaunch scheduled task |
| Delete Old Users | Days threshold | Delete local user profiles (and registry entries) not used in N days |
| Add Printer | Printer IP/hostname, Display name | Add a network printer via TCP/IP port using Microsoft IPP Class Driver |
| Health Check | — | Run `sfc /scannow` to verify and repair protected Windows system files |
| Remove Office | — | Remove all versions of Microsoft Office, Visio, and Project (C2R, MSI, SaRA fallback, registry/file cleanup) |

#### Network

| Button | Parameters | Description |
|--------|-----------|-------------|
| Network Config | — | IP address, MAC, gateway, and DNS for all adapters |
| Test Connectivity | Target address | Ping a hostname or IP and test DNS resolution |
| Open Ports | — | All TCP ports currently listening on this machine |

#### Security

| Button | Description |
|--------|-------------|
| Local Admins | All members of the local Administrators group (uses ADSI for compatibility with all Windows versions) |
| Last Logins | Last 20 interactive logon events from the Security event log (Event ID 4624, types 2 and 10) |

#### Fixes

| Button | Description |
|--------|-------------|
| Repair Windows | Run `DISM /Online /Cleanup-Image /RestoreHealth` to repair the Windows image |
| Fix WMI | Rebuild the WMI repository — stops service, clears repo, re-registers all DLLs and recompiles MOF files |
| Reset File Ext | Clear user-set file associations from the registry and restart Explorer |
| Fix Win Update | Reset Windows Update components — renames SoftwareDistribution and Catroot2 folders, restarts services |
| Clean SCCM | Remove the SCCM/ConfigMgr client — stops services, removes WMI namespaces, registry keys, and all folders |

#### Tools

| Button | Description |
|--------|-------------|
| Clear Output | Clear the output console |
| Copy to Clipboard | Copy all current output text to the Windows clipboard |

---

### 🖧 Remote PC Tab

> All remote scripts use `Invoke-Command -ComputerName $TargetHost`. Enter the target hostname in the Remote Target Bar and click **Connect** before running scripts.
>
> **Multiple targets:** expand the "Multiple targets" section in the Remote Target Bar and enter a comma-separated list of hostnames to run a script against all of them sequentially.
>
> **Saved profiles:** click 💾 to save the current hostname and credentials as a profile, then select it from the dropdown on future sessions.
>
> **Cancel:** a Cancel button appears while a script is running. Scripts also time out automatically after the configured timeout (default: 30 seconds).

#### Remote System

| Button | Description |
|--------|-------------|
| System Info | Hardware and OS info from the remote PC |
| Disk Usage | Drive space on the remote PC |
| Running Processes | Top CPU-consuming processes on the remote PC |

#### Remote Maintenance

| Button | Parameters | Description |
|--------|-----------|-------------|
| Restart PC | — | Gracefully restart the remote computer |
| Flush DNS | — | Flush DNS cache on the remote PC |
| Install Updates | — | Trigger Windows Update scan and install (requires PSWindowsUpdate on target) |
| Intune Sync | — | Trigger Intune policy sync via PushLaunch on the remote PC |
| Clear Temp | — | Report disk space usage of temp/cache locations on the remote PC |
| Delete Old Users | Days threshold | Delete old user profiles (and registry entries) on the remote PC |
| Add Printer | Printer IP/hostname, Display name | Add a network printer on the remote PC |
| Remove Office | — | Remove all versions of Microsoft Office, Visio, and Project from the remote PC (C2R and MSI; SaRA not available remotely) |

#### Remote Fixes

| Button | Description |
|--------|-------------|
| Repair Windows | Run DISM RestoreHealth on the remote PC |
| Fix WMI | Rebuild the WMI repository on the remote PC |
| Reset File Ext | Reset default file associations on the remote PC |
| Fix Win Update | Reset Windows Update components on the remote PC |
| Clean SCCM | Remove the SCCM/ConfigMgr client from the remote PC |

#### Remote Services

| Button | Parameters | Description |
|--------|-----------|-------------|
| List Services | — | All services on the remote PC with their status |
| Restart Service | Service name | Restart a named service on the remote PC |

#### Support

| Button | Description |
|--------|-------------|
| Remote Control | Launch the SCCM Remote Control viewer (CmRcViewer.exe) for the target PC. Connects without any interactive prompts. Path resolved from registry with hard-coded fallback. Requires CmRcViewer.exe at `C:\Program Files (x86)\Microsoft Configuration Manager\AdminConsole\bin\i386\`. |
| Remote Desktop | Open an RDP session (`mstsc.exe`) to the target PC. If alternate credentials are configured, they are temporarily saved to Windows Credential Manager via `cmdkey` and auto-removed after 15 seconds. |

#### Tools

| Button | Description |
|--------|-------------|
| Clear Output | Clear the output console |
| Copy to Clipboard | Copy all current output text to the Windows clipboard |

---

### 🏢 Active Directory Tab

#### User Management

| Button | Parameters | Description |
|--------|-----------|-------------|
| Get User Info | Username | Full AD details for a user account |
| Unlock Account | Username | Unlock a locked-out AD user account |
| Reset Password | Username, New password | Reset a user's password and force change at next logon |
| Disable Account | Username | Disable an AD user account (e.g. for leavers) |
| Locked Out Users | — | All currently locked-out user accounts in AD |

#### Group Management

| Button | Parameters | Description |
|--------|-----------|-------------|
| Group Members | Group name | All members of an AD security group |
| User Groups | Username | All AD groups a user belongs to |
| Add to Group | Username, Group name | Add a user to a security group |
| Clone User | Source username, Target username | Copy all group memberships from source user to target user |

#### Computer Management

| Button | Parameters | Description |
|--------|-----------|-------------|
| Computer Info | Computer name | AD computer object details and last logon |
| Inactive Computers | — | Computer accounts with no logon in the last 90+ days |

#### Reports

| Button | Parameters | Description |
|--------|-----------|-------------|
| Password Expiry | — | Users whose passwords expire within the next 14 days |
| Inactive Users | — | Enabled user accounts with no logon in the last 60 days |
| Export OU Objects | OU Distinguished Name | Export all objects in the OU (with description) to `C:\temp\<OUName>.csv` |
| Export Group Members | Group name | Export all group members (username, display name, email, description) to `C:\temp\<GroupName>.csv` |

#### Tools

| Button | Description |
|--------|-------------|
| Clear Output | Clear the output console |
| Copy to Clipboard | Copy all current output text to the Windows clipboard |

---

## Output Console

The console at the bottom of the window shows colour-coded output:

| Colour | Meaning |
|--------|---------|
| Cyan | Section headers |
| Light grey | Standard output |
| Green | Success / completion |
| Amber | Warnings |
| Red | Errors |

**File → Save Output** saves the full console to a `.txt` file.
**Tools → Copy to Clipboard** copies all output to the Windows clipboard.

---

## Theme Switching

Switch between dark and light themes at any time via **File → Switch to Light/Dark Theme**.
The preference is saved across sessions.

---

## Security Notes

- The app requests **administrator elevation** via the app manifest.
- Execution policy is set to **Bypass** for the current process only — not system-wide.
- Credentials entered in the Remote dialog are held **in memory only** — never written to disk.
- The AD Reset Password script stores the password as a `SecureString` before calling `Set-ADAccountPassword`.
- The Local Admins script uses ADSI (`WinNT://./Administrators`) for compatibility with all Windows versions.
- Repair Windows, Fix WMI, Clean SCCM, and Delete Old Users all require the app to be run as Administrator.

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| `The term 'Get-ADUser' is not recognized` | Install RSAT AD Tools |
| `Access is denied` on remote scripts | Run `Enable-PSRemoting` on target; check firewall rules |
| `WinRM cannot complete the operation` | Add target IP/hostname to TrustedHosts list |
| Security event log empty or access denied | Run AdminToolKit as Administrator |
| Fluent.Ribbon missing on build | Keep the `packages\` folder next to the solution and restore NuGet |
| Output shows `FormatEntryData` type names | Script is using `Format-Table` or `Write-Output` — change to `Write-Host` |
| App does not request admin elevation | Check that `app.manifest` is referenced in project properties |
| Delete Old Users or Export scripts prompt for input | Expected — enter the requested value and click OK |
| Remote scripts fail immediately | Enter hostname in Remote Target Bar and click Connect first |
| Add Printer fails | Check the printer is reachable on the network; run as Administrator |
| Export CSV has spaces in filename | Group/OU names with spaces are automatically converted to underscores in the filename |
| Remote Control `0xc000007b` error | Visual C++ 2013 x86 runtime is missing or the wrong bitness (64-bit). The AdminToolKit installer installs `vcredist_x86.exe` automatically. Reinstall AdminToolKit or run `vcredist_x86.exe` manually from https://aka.ms/highdpimfc2013x86enu |
| Remote Control button does nothing or errors | Ensure CmRcViewer.exe is present at `C:\Program Files (x86)\Microsoft Configuration Manager\AdminConsole\bin\i386\`. The installer offers to copy it from the bundled CmRcViewer folder. |
| Remote Control connects to wrong machine | Check the hostname in the Remote Target Bar is correct — CmRcViewer uses the hostname directly without further prompts. |

---

## Extending — Adding a New Ribbon Group

```xml
<fluent:RibbonGroupBox Header="My New Group"
                       Background="{DynamicResource BackgroundPanelBrush}">
    <fluent:Button Header="My Script"
                   LargeIcon="{StaticResource IconNetwork}"
                   ToolTip="What this script does."
                   Click="LocalScript_Click"
                   Tag="My Script" />
</fluent:RibbonGroupBox>
```

Use `{DynamicResource ...}` for all colour references so they update correctly when the theme is switched at runtime.

---

## License

AdminToolKit is licensed under the **Microsoft Public License (MS-PL)**.

See [LICENSE.txt](LICENSE.txt) for the full license text.

Key points:
- No warranty is provided — you use the software at your own risk
- Attribution required if redistributing: *"Based on AdminToolKit"*
- The AdminToolKit name and logo may not be used to endorse modified versions
