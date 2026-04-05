# Contributing to AdminToolKit

Thank you for your interest in contributing! This guide explains how the project
is structured and how to add new scripts, ribbon buttons, or features.

---

## Table of Contents

- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Adding a New Script](#adding-a-new-script)
- [Adding a New Ribbon Group](#adding-a-new-ribbon-group)
- [PowerShell Script Guidelines](#powershell-script-guidelines)
- [Remote Script Guidelines](#remote-script-guidelines)
- [UI / XAML Guidelines](#ui--xaml-guidelines)
- [Submitting Changes](#submitting-changes)
- [Code Style](#code-style)

---

## Getting Started

### Prerequisites

- Windows 10/11 or Windows Server 2016+
- Visual Studio 2019 or 2022 (Community edition is fine)
- .NET Framework 4.8 SDK
- Windows PowerShell 5.1

### Setup

```bash
git clone https://github.com/smanifold123/AdminToolKit.git
cd AdminToolKit
```

Open `AdminToolKit.sln` in Visual Studio. NuGet packages restore automatically
on first build. Alternatively, run `Build.bat` from the solution root.

The app requests UAC elevation on launch — run Visual Studio as Administrator
when debugging, or accept the UAC prompt when launching the built executable.

---

## Project Structure

```
AdminToolKit/
├── AdminToolKit.sln
├── Build.bat                        # Build AdminToolKit.exe
├── Build-Installer.bat              # Build AdminToolKit-Setup.exe (requires NSIS)
├── AdminToolKit_Installer.nsi       # NSIS installer script
├── CmRcViewer/                      # Place CmRcViewer.exe here for installer bundling
│   └── README_FILES.txt
└── AdminToolKit/
    ├── Helpers/
    │   ├── ScriptRegistry.cs        # ★ CENTRAL SCRIPT CATALOGUE — add scripts here
    │   ├── PowerShellRunner.cs      # Executes scripts locally or via WinRM
    │   ├── OutputLine.cs            # Typed output line with colour metadata
    │   ├── ThemeManager.cs          # Light/dark runtime theme switching
    │   └── RelayCommand.cs          # ICommand implementation for MVVM bindings
    ├── Models/
    │   ├── ScriptDefinition.cs      # Describes a script (name, path, params, flags)
    │   ├── ScriptResult.cs          # Result returned after script execution
    │   ├── RemoteTarget.cs          # Holds hostname, username, domain
    │   └── ScriptCategory.cs        # Enum: Local, Remote, ActiveDirectory
    ├── ViewModels/
    │   ├── MainViewModel.cs         # Root VM — output lines, busy state, status text
    │   ├── LocalScriptsViewModel.cs # Handles Local PC script execution
    │   ├── RemoteScriptsViewModel.cs# Handles Remote PC script execution + ping
    │   └── ActiveDirectoryViewModel.cs
    ├── Views/
    │   ├── MainWindow.xaml          # ★ RIBBON LAYOUT — add buttons here
    │   ├── MainWindow.xaml.cs       # Button click handlers, theme, dialogs
    │   ├── Controls/
    │   │   ├── ScriptOutputPanel    # Colour-coded, auto-scrolling output console
    │   │   └── RemoteTargetBar      # Hostname input + Connect button + status dot
    │   └── Dialogs/
    │       ├── ParameterInputDialog # Prompt for script parameters
    │       ├── CredentialDialog     # Alternate username/password input
    │       └── RemoteHostDialog     # (legacy) remote host dialog
    ├── Themes/
    │   ├── DarkTheme.xaml           # Dark colour palette and control styles
    │   └── LightTheme.xaml          # Light colour palette and control styles
    └── Scripts/
        ├── Local/                   # Scripts run on the local machine
        ├── Remote/                  # Scripts run via Invoke-Command on target PC
        └── ActiveDirectory/         # Scripts using the AD PowerShell module
```

**The two files you will touch for 90% of contributions:**

| File | What you do there |
|------|-------------------|
| `Helpers/ScriptRegistry.cs` | Register the script definition (name, path, icon, parameters) |
| `Views/MainWindow.xaml` | Add the ribbon button that triggers it |

---

## Adding a New Script

Adding a script requires changes in exactly **two places** plus a `.ps1` file.

### Step 1 — Create the `.ps1` file

Drop your script in the appropriate folder:

| Tab | Folder |
|-----|--------|
| Local PC | `Scripts\Local\` |
| Remote PC | `Scripts\Remote\` |
| Active Directory | `Scripts\ActiveDirectory\` |

**File naming convention:** `Verb-Noun.ps1` following PowerShell conventions.
Examples: `Get-DiskUsage.ps1`, `Reset-ADPassword.ps1`, `Fix-RemoteWMI.ps1`

### Step 2 — Register in `ScriptRegistry.cs`

Find the appropriate list (`LocalScripts`, `RemoteScripts`, or
`ActiveDirectoryScripts`) and add an entry:

```csharp
new ScriptDefinition
{
    Name               = "My New Script",       // Must match the ribbon button Tag exactly
    GroupName          = "My Group",            // Ribbon group header this belongs to
    Description        = "What this script does.",
    IconKey            = "IconNetwork",         // Key from Window.Resources in MainWindow.xaml
    ScriptPath         = @"Scripts\Local\My-NewScript.ps1",

    // Optional: declare parameters the user will be prompted for
    Parameters = new Dictionary<string, string>
    {
        { "ParameterName", "Friendly prompt text shown to the user" }
    },

    // Remote scripts only:
    RequiresRemoteHost = true,
}
```

**Available icon keys:**

| Key | Appearance |
|-----|------------|
| `IconSystemInfo` | Monitor / computer |
| `IconDisk` | Hard drive |
| `IconProcess` | CPU / chip |
| `IconService` | Gear / settings |
| `IconNetwork` | Globe / network |
| `IconSecurity` | Shield |
| `IconUser` | Person silhouette |
| `IconGroup` | Multiple people |
| `IconFix` | Wrench |
| `IconClean` | Trash / clear |
| `IconClipboard` | Clipboard |
| `IconPrint` | Printer |
| `IconRemoteControl` | Monitor with cursor |

To add a new icon, add a `DrawingImage` with a unique `x:Key` to
`Window.Resources` in `MainWindow.xaml` using SVG path geometry.

### Step 3 — Add the ribbon button in `MainWindow.xaml`

Find the appropriate `RibbonGroupBox` (or create a new one — see below) and
add a button:

```xml
<fluent:Button Header="My New Script"
               LargeIcon="{StaticResource IconNetwork}"
               ToolTip="What this script does."
               Tag="My New Script"
               Click="LocalScript_Click" />
```

> **Important:** The `Tag` value must **exactly match** `ScriptDefinition.Name`
> in `ScriptRegistry.cs`. The click handler uses `Tag` to look up the script.

**Click handlers by tab:**

| Tab | Handler |
|-----|---------|
| Local PC | `LocalScript_Click` |
| Remote PC | `RemoteScript_Click` |
| Active Directory | `AdScript_Click` |

---

## Adding a New Ribbon Group

Add a `RibbonGroupBox` inside the appropriate `RibbonTabItem`:

```xml
<fluent:RibbonGroupBox Header="My New Group"
                       Background="{DynamicResource BackgroundPanelBrush}">
    <fluent:Button Header="My Script"
                   LargeIcon="{StaticResource IconNetwork}"
                   ToolTip="What this script does."
                   Tag="My Script"
                   Click="LocalScript_Click" />
</fluent:RibbonGroupBox>
```

Always use `{DynamicResource ...}` for colours — static colours break when
the theme is switched at runtime.

---

## PowerShell Script Guidelines

Follow these rules for all scripts to ensure output displays correctly in the
console panel.

### Output — the most important rule

**Always use `Write-Output`. Never use `Write-Host`, `Format-Table`, or `Format-List`.**

```powershell
# CORRECT
Write-Output "  Disk usage collected."
Write-Output ("  {0,-20} : {1}" -f "OS Name", $os.Caption)

# WRONG — causes blank output or FormatEntryData type names
Write-Host "Disk usage collected."
Get-Process | Format-Table -AutoSize
$obj | Format-List
```

`Format-Table` and `Format-List` produce internal formatting objects that
`ToString()` as type names like
`Microsoft.PowerShell.Commands.Internal.Format.FormatEntryData` rather than
readable text when collected via `ps.Invoke()`.

### Column formatting

Use PowerShell's `-f` string operator for aligned output:

```powershell
# Key-value pairs
Write-Output ("  {0,-20} : {1}" -f "Computer Name", $env:COMPUTERNAME)

# Table columns
Write-Output ("  {0,-25} {1,-15} {2}" -f "Name", "Status", "StartType")
Write-Output ("  {0,-25} {1,-15} {2}" -f "----", "------", "---------")
foreach ($svc in $services) {
    Write-Output ("  {0,-25} {1,-15} {2}" -f $svc.Name, $svc.Status, $svc.StartType)
}
```

### Section headers

Use a consistent divider style:

```powershell
Write-Output ""
Write-Output "  -- SECTION TITLE ------------------------------------------"
```

### Error handling

Always wrap the main logic in `try/catch` and output errors via `Write-Output`:

```powershell
try {
    # main logic
    Write-Output "  Operation completed."
} catch {
    Write-Output "  ERROR: $($_.Exception.Message)"
}
```

### Success/failure indicators

End scripts with a clear result line:

```powershell
Write-Output "  Operation completed successfully."
# or
Write-Output "  ERROR: could not connect to $TargetHost"
```

### Parameters

Parameters declared in `ScriptRegistry.cs` are injected as PowerShell variables
at the top of the script by `PowerShellRunner.InjectParameters()`:

```csharp
// In ScriptRegistry.cs:
Parameters = new Dictionary<string, string>
{
    { "Username", "SAM account name (e.g. jsmith)" }
}
```

```powershell
# In the .ps1 file — $Username is available automatically:
if (-not $Username) { Write-Output "  No username provided."; return }
Write-Output "  Processing: $Username"
```

Do not add `param()` blocks — parameter injection prepends variable assignments
as plain PowerShell statements before your script text.

---

## Remote Script Guidelines

Remote scripts have additional constraints because they run inside
`Invoke-Command -ScriptBlock {}` over a PSSession.

### Credential preamble

All remote scripts must include this block at the top to support optional
alternate credentials:

```powershell
$__credParams = @{}
if ($__cred -ne $null) {
    $__credParams['Credential']     = $__cred
    $__credParams['Authentication'] = 'Negotiate'
}
```

Then use `@__credParams` splatting on `Invoke-Command`:

```powershell
Invoke-Command -ComputerName $TargetHost @__credParams -ScriptBlock {
    # your script here
}
```

`$__cred` and `$TargetHost` are injected automatically by `PowerShellRunner`
before execution. Do not declare them yourself.

### Write-Output inside Invoke-Command — critical

Inside `Invoke-Command -ScriptBlock {}`, **`Write-Host` writes to the remote
machine's console** and is never returned to the caller. Always use
`Write-Output`:

```powershell
Invoke-Command -ComputerName $TargetHost @__credParams -ScriptBlock {
    Write-Output "This comes back to AdminToolKit."  # CORRECT
    Write-Host   "This goes to the remote console."  # WRONG — never visible
}
```

### No Format cmdlets inside scriptblocks

`Format-Table` and `Format-List` cannot serialize across a PSSession boundary:

```powershell
Invoke-Command ... -ScriptBlock {
    # WRONG
    Get-Service | Format-Table -AutoSize

    # CORRECT
    Get-Service | ForEach-Object {
        Write-Output ("  {0,-30} {1}" -f $_.Name, $_.Status)
    }
}
```

### Keep lines outside the scriptblock for preamble output

```powershell
# These run locally (correct — output captured normally)
Write-Output "  Remote Disk Usage: $TargetHost"
Write-Output "  Queried at: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"

Invoke-Command -ComputerName $TargetHost @__credParams -ScriptBlock {
    # Everything here runs remotely
    Write-Output "  Remote output line."
}
```

---

## UI / XAML Guidelines

- Always use `{DynamicResource BrushName}` for colours — never hardcode hex
  values in the XAML so themes work correctly
- Use `{StaticResource IconKey}` for `LargeIcon` on ribbon buttons
- Ribbon group headers use Title Case: `"Remote System"`, `"User Management"`
- Button headers use Title Case with no trailing punctuation: `"System Info"`,
  `"Reset Password"`
- Keep `ToolTip` text to one sentence — it appears in a small hover tooltip
- All interactive controls must have a `ToolTip`

**Common dynamic resources:**

| Resource Key | Used for |
|---|---|
| `BackgroundPanelBrush` | `RibbonGroupBox` background |
| `BackgroundBrush` | Window/control background |
| `TextPrimaryBrush` | Primary text foreground |
| `AccentBrush` | Highlighted/active elements |

---

## Submitting Changes

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-new-script`
3. Make your changes following the guidelines above
4. Test your script manually — verify output renders correctly in the console
   panel (no `FormatEntryData` type names, no blank output)
5. For remote scripts: test both with and without alternate credentials,
   and verify the `00000409` subfolder is not accidentally included
6. Commit with a descriptive message:
   ```
   Add Get-BitLockerStatus script to Local PC > Security group
   ```
7. Push and open a Pull Request with:
   - What the script does
   - Which tab and ribbon group it appears in
   - Any prerequisites (e.g. requires RSAT, requires PSWindowsUpdate module)
   - Screenshot of the output if possible

---

## Code Style

### C#

- 4-space indentation
- Opening braces on the same line (`{`) for methods and properties
- `_camelCase` for private fields, `PascalCase` for properties and methods
- `var` for local variables where the type is obvious from the right-hand side
- XML doc comments (`///`) on public members in the Helpers and Models layers

### PowerShell

- 4-space indentation
- Use approved verbs (`Get-`, `Set-`, `Invoke-`, `Remove-`, `Start-`, `Stop-`)
- Comment sections with `# ── SECTION ───` style headers
- Keep scripts focused — one script, one job
- Avoid aliases (`%`, `?`, `gci`) — use full cmdlet names for readability

### XAML

- 4-space indentation
- One attribute per line for elements with more than two attributes
- Group related attributes: Name/Key first, layout properties, visual properties, events last
- Comments use `<!-- ── Description ── -->` style

---

## Questions

Open a GitHub Discussion or Issue if you're unsure about anything before
writing code. Happy to help with script ideas, icon choices, or architecture
questions.
