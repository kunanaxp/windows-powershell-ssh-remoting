# windows-powershell-ssh-remoting

A skill for AI coding agents (opencode, Claude Code, Cursor, Codex, etc.) that
teaches **PowerShell Remoting over SSH** — running commands on a remote Windows
host by passing PowerShell `ScriptBlock`s instead of brittle `ssh "..."` command
strings.

> **The problem it solves**: When an agent runs remote commands via `ssh user@host
> "some command"`, the string passes through multiple parsers (local shell → ssh →
> remote shell). Every `$`, `;`, `"`, and backtick is a chance for corruption.
> Powershell Remoting over SSH replaces the string with an object stream, so the
> remote-shell layer **disappears entirely**.

## Why this is better than plain `ssh`

| | Plain `ssh "cmd"` | PowerShell SSH Remoting |
|---|---|---|
| Transport | string | serialized PowerShell `ScriptBlock` |
| Quoting risk | high (multi-layer escaping) | mostly eliminated |
| Return value | text you must parse | structured objects (properties survive) |
| Persistent sessions | reconnect every time | `New-PSSession` keeps state |
| Long scripts | heredoc / temp upload | `-FilePath` (no remote upload needed) |

## Installation

As an opencode skill, place this folder under
`~/.config/opencode/skills/windows-powershell-ssh-remoting/` (global) or
`<project>/.opencode/skills/...` (project), then restart opencode.

For other agents (Claude Code, Cursor, Codex), copy the `SKILL.md` into their
agent/skills directory and register it per their docs. The skill assumes the
agent can already run a local `pwsh`.

## Prerequisites

**Local** (client):
- PowerShell 6+ with the SSH parameter set — check:
  ```powershell
  (Get-Command New-PSSession).ParameterSets.Name   # must include SSHHost
  ```
- `ssh.exe` on `PATH`

**Remote** (Windows host):
- PowerShell 6+ installed
- OpenSSH `sshd` service running
- Working SSH authentication (key or password)
- `sshd_config` defines a PowerShell subsystem (see below)

## One-time remote setup

On the remote Windows host, edit `C:\ProgramData\ssh\sshd_config` and add:

```text
Subsystem powershell C:\PROGRA~1\POWERS~1\7\pwsh.exe -sshs -NoLogo -NoProfile
```

Then validate and restart:

```powershell
sshd.exe -t
Restart-Service sshd
```

**Two gotchas** (both cost real debugging time):
1. **8.3 short path is required** — OpenSSH for Windows breaks on subsystem
   paths containing spaces (`C:\Program Files\...`). Use `C:\PROGRA~1\POWERS~1\7\`
   or a symlink. For non-standard PowerShell installs, adapt the path.
2. **`-NoProfile` is required** — profile/startup output pollutes the remoting
   channel and kills the session.

## Usage examples

One-shot:

```powershell
Invoke-Command -HostName <host> -UserName <user> -KeyFilePath <key> `
  -ScriptBlock { Get-Service sshd }
```

Persistent session:

```powershell
$s = New-PSSession -HostName <host> -UserName <user> -KeyFilePath <key>
Invoke-Command -Session $s -ScriptBlock { Get-Process pwsh }
Remove-PSSession $s
```

Local script without upload:

```powershell
Invoke-Command -HostName <host> -UserName <user> -KeyFilePath <key> `
  -FilePath .\deploy.ps1
```

## Compatible agents

Tested/deployed in **opencode**. The `Invoke-Command` patterns are
agent-agnostic and work from any environment that can invoke a local `pwsh`.

- opencode (primary)
- Claude Code, Cursor, Codex (via manual skill registration)

## Contributing

Improvements welcome via pull request. Keep the SKILL.md self-contained so the
skill stays a single-file drop-in.

## License

[MIT](LICENSE)