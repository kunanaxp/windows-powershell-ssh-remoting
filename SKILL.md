---
license: MIT
name: windows-powershell-ssh-remoting
description: Use for remote PowerShell execution on a Windows remote host over SSH, especially when shell quoting or persistent sessions matter.
compatibility: opencode, claude-code, cursor, codex
metadata:
  version: 1.0.0
  author: kunanaxp
  tags: ssh, powershell, remoting, invoke-command, windows, agent
---

# PowerShell SSH Remoting

Prefer `Invoke-Command` for remote PowerShell instead of nested `ssh` + shell command strings. It sends a PowerShell `ScriptBlock` through the SSH PowerShell subsystem, avoiding most remote-shell quoting problems.

## Core principles

* Prefer direct execution when practical.
* Keep temporary scripts local; avoid unnecessary remote files.
* For remote PowerShell, prefer `Invoke-Command` / `New-PSSession`.
* Create or upload a remote script only when the remote file itself is genuinely needed.
* Use a local temporary script only when it simplifies complex automation.

Typical pattern:

```text
Direct shell/tool
    ↓ when remote PowerShell execution is needed
Remote ScriptBlock / persistent session
    ↓ when a local script improves complex automation
Local temporary script
    ↓ only when the remote file itself is genuinely required
Remote script file
```

For interactive terminals, use `ssh` directly.

Avoid nested shell chains such as:

```text
bash → ssh → cmd → pwsh
```

They create unnecessary quoting and escaping problems.

## Basic usage

One-shot:

```powershell
Invoke-Command -HostName <host> -UserName <user> -KeyFilePath <key> `
  -ScriptBlock { Get-Service sshd }
```

Persistent session:

```powershell
$s = New-PSSession -HostName <host> -UserName <user> -KeyFilePath <key>
Invoke-Command -Session $s -ScriptBlock { ... }
Remove-PSSession $s
```

Pass values explicitly:

```powershell
Invoke-Command ... `
  -ScriptBlock { param($p) Get-Process -Name $p } `
  -ArgumentList $processName
```

For genuinely large or reusable remote scripts, `Invoke-Command -FilePath` is available. Do not use it merely to avoid quoting or create temporary remote artifacts.

## Requirements

Local:

* PowerShell 6+ with the SSH parameter set (`SSHHost` / `SSHHostHashParam`).
* `ssh.exe` on `PATH`.

Remote:

* PowerShell 6+.
* OpenSSH `sshd` service running.
* Working SSH authentication.
* `sshd_config` must define a PowerShell subsystem.

Check local support:

```powershell
$PSVersionTable.PSVersion
(Get-Command New-PSSession).ParameterSets.Name
```

## Windows SSH subsystem

Configure `$env:ProgramData\ssh\sshd_config` or `C:\ProgramData\ssh\sshd_config`:

```text
Subsystem powershell C:\PROGRA~1\POWERS~1\7\pwsh.exe -sshs -NoLogo -NoProfile
```

For non-standard installations, replace the pwsh.exe path.

Windows OpenSSH has problems with spaces in subsystem executable paths. Use an 8.3 path or another path without spaces, such as a symbolic link.

After changes:

```powershell
sshd.exe -t
Restart-Service sshd
```

Smoke test:

```powershell
Invoke-Command -HostName <host> -UserName <user> -KeyFilePath <key> `
  -ScriptBlock { $PSVersionTable.PSVersion.ToString() }
```

Use `-NoProfile` to keep profile/startup code from writing unexpected output into the remoting channel.

## Important behavior

* The `ScriptBlock` is parsed locally. This removes the remote shell command-string layer, but not every possible local-shell quoting issue.
* Remote results are usually deserialized objects: properties survive, methods generally do not. Perform method calls inside the remote `ScriptBlock`.
* Native executables such as `nvidia-smi` can be invoked inside a remote `ScriptBlock`.
* For complex local automation, it is fine to create a temporary local script, run it with the appropriate interpreter, then delete it.
* Avoid uploading temporary `.ps1`, `.py`, `.js`, `.sh`, `.cmd`, or other generated scripts to the remote host just for one-off execution.

## Field notes

* `New-PSSession` preserves a PowerShell remoting session, not arbitrary process lifetime.
* `nohup` is a POSIX tool; do not assume it provides Windows process-detachment semantics.
* For work that must survive disconnects, use an appropriate process/service mechanism rather than relying on the remoting session.
* `tmux` / `psmux` are optional tools for persistent interactive terminals. Use them when available and useful.

## Troubleshooting

For:

```text
The SSH transport process has abruptly terminated causing this remote session to break.
```

Check:

1. Windows subsystem executable path, especially spaces.
2. `-sshs -NoProfile -NoLogo`.
3. `sshd.exe -t`.
4. Restart `sshd`.
5. OpenSSH logs:

```powershell
Get-WinEvent -LogName 'OpenSSH/Operational' -MaxEvents 20
```

6. Test the subsystem:

```text
ssh -vvv <host> -s powershell
```

If SSH authentication works but the `powershell` subsystem dies immediately, investigate `pwsh -sshs` startup/configuration rather than basic SSH connectivity.

Running `pwsh -sshs` directly without an SSH subsystem channel is not a valid end-to-end test.
