# GameRoom host agent — downloads

Builds of the GameRoom host agent, and of the service that keeps it running.
Nothing else lives here: the source is in a separate repository, and this one
exists so that a host's machine can download an update without needing an
account.

Each release carries two files and a `manifest.json`.

| File | What it is |
| --- | --- |
| `host-agent-<version>.exe` | The agent. Updates itself, through the service below. |
| `gameroom-service-<version>.exe` | Installs once. It is what performs an update, so it is never updated automatically. |
| `manifest.json` | The version and SHA-256 a host's service checks against. |

## These binaries are not signed

Windows SmartScreen will say **"Windows protected your PC"** on first run —
*More info* → *Run anyway* — and some antivirus will object. That is worth
understanding rather than clicking past: the agent injects keyboard and mouse
input and captures the screen, because that is what streaming a game to
someone else requires. It is also, in outline, what a remote access trojan
does, and nothing about an unsigned binary lets a scanner tell the two apart.

**Check the hash before you run it.** Every release lists a SHA-256, and each
file has a `.sha256` beside it.

```powershell
Get-FileHash .\host-agent-0.1.0.exe -Algorithm SHA256
```

The service checks this hash on every update it applies, and refuses a build
that does not match. That catches a truncated download and a file swapped on
a mirror. It is not a substitute for a signature — whoever can replace a
binary can usually replace the text describing it — which is why checking the
first one by hand is worth the minute.

## Installing

See the host agent documentation for the layout and the one elevated command
that registers the service.

## Reporting something

Issues are open here. For anything that looks like a security problem, please
report it privately rather than in an issue.
