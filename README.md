# GameRoom host agent — downloads

Builds of the GameRoom host agent, and of the service that keeps it running.
Nothing else lives here: the source is in a separate repository, and this one
exists so that a host's machine can download an update without needing an
account.

| File | What it is |
| --- | --- |
| `host-agent-<version>.exe` | The agent. Updates itself, through the service below. |
| `gameroom-service-<version>.exe` | Installs once. It is what performs an update, so it is never updated automatically. |
| `manifest.json` | The version and SHA-256 a host's service checks against. |

---

## Before you start

**Windows 10 version 1903 or newer.** The capture uses Windows Graphics
Capture, which does not exist before that.

**ffmpeg, on your `PATH`.** The agent spawns it to encode video and audio. If
you keep it somewhere unusual, set `FFMPEG_BINARY` to the full path instead.

```powershell
winget install Gyan.FFmpeg
```

**ViGEmBus, if you want controllers to work.** Without it keyboard and mouse
still reach the game and a gamepad does not — the agent says so at startup
rather than failing silently.

```powershell
winget install ViGEm.ViGEmBus
```

**A machine you are willing to hand over.** The current compute provider runs
games on your own desktop with no isolation between a remote player and your
Windows install. The agent warns about this every time it starts. Use a
machine you would not mind reinstalling.

### These binaries are not signed

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

---

## Install

### 1. Lay out the files

Everything lives under `C:\ProgramData\GameRoom`. A version directory holds
**only the executable**; your own configuration sits beside it and is never
touched by an update.

```powershell
$root = "C:\ProgramData\GameRoom"
$version = "0.1.0"

New-Item -ItemType Directory -Force "$root\bin", "$root\agent\$version", "$root\config" | Out-Null

Move-Item .\host-agent-$version.exe       "$root\agent\$version\host-agent.exe"
Move-Item .\gameroom-service-$version.exe "$root\bin\gameroom-service.exe"

# Which build the service should run.
Set-Content "$root\agent\current.txt" $version -NoNewline
```

The result:

```text
  C:\ProgramData\GameRoom\
    bin\gameroom-service.exe      the service; rarely changes
    agent\current.txt             which build to run
    agent\previous.txt            written on the first update; the way back
    agent\0.1.0\host-agent.exe    one build, and nothing else
    config\                       yours, and untouched by any update
    service.log
```

### 2. Pair the machine

Ask your host dashboard for a pairing code, then:

```powershell
cd C:\ProgramData\GameRoom\config
..\agent\0.1.0\host-agent.exe --pair <code>
```

The credential is written to `%LOCALAPPDATA%\GameRoom\agent.json`, not to this
directory — it belongs to the machine, and keeping it out of the install tree
means it does not end up in a backup or a support ticket.

Check it took:

```powershell
..\agent\0.1.0\host-agent.exe --status
```

That prints what the machine is paired to, whether the control plane answers,
and what this PC says it can serve.

### 3. Tell it what you have installed

The agent launches real games, so it needs to know what is on the machine.

```powershell
cd C:\ProgramData\GameRoom\config
..\agent\0.1.0\host-agent.exe --detect-games
```

That scans Steam, Epic and Battle.net and writes a `games.json` proposal.
Read it and correct it — it is your catalogue, and the paths in it are yours.

A `games.json` in `config\` is also what tells the agent to run games rather
than to pretend: with no catalogue it falls back to a mock provider that
reports a session ready and launches nothing, and then refuses to run at all
unless you have explicitly allowed it. If your sessions start and no game
appears, this file is the first thing to check.

### 4. Install the service

**This step needs an elevated shell.** Open PowerShell as Administrator:

```powershell
C:\ProgramData\GameRoom\bin\gameroom-service.exe --install
sc start GameRoomAgent
```

The service starts with Windows, keeps the agent running, and applies updates
when the control plane offers one and no session is in progress. It is what
makes hosting survive a reboot and a logout.

To watch it work without installing anything, run it in the foreground
instead — same code, no Service Control Manager:

```powershell
C:\ProgramData\GameRoom\bin\gameroom-service.exe --console
```

**Never start `host-agent.exe` by hand while the service is running.** It
would be a second agent competing for the same encoder, the same port and the
same session.

---

## Check it works

Each of these answers one question, and none of them needs a player.

```powershell
cd C:\ProgramData\GameRoom\config
$agent = "..\agent\0.1.0\host-agent.exe"

& $agent --status          # paired? reachable? what does this PC offer?
& $agent --list-displays    # which screen a session runs on
& $agent --list-windows     # what the capture can be pointed at
& $agent --probe-audio      # does sound reach a player
& $agent --measure          # what this PC can serve, recorded
```

### `--local-play`: the whole pipeline, without an account

```powershell
& $agent --local-play --window "<part of a window title>"
```

It prints an HTTPS URL. Open it on this machine or on a phone on the same
network, accept the self-signed certificate once, and you are looking at the
real capture, encode and transport path with your own keyboard and mouse
reaching the window. If this works, hosting works.

---

## What a session actually does to your machine

Worth knowing before a stranger is on it.

**It takes the foreground.** The game has to be the active window for the
player's mouse and keyboard to reach it, so the agent raises it whenever
input arrives. You cannot use the machine for something else while a session
runs. Nothing short of a virtual machine changes that on Windows client,
which allows one interactive session at a time.

**A minimised game is restored**, without taking your caret, so a session
does not die because you minimised something. `STREAM_WINDOW_RESTORE=never`
in `config\.env` tells the agent to leave your windows alone and fail the
session instead.

**Sound is the game's process tree only** — on Windows build 20348 or newer.
Below that the API Windows provides accepts the request and then corrupts the
calling process, so the agent refuses it and the session is silent.
`STREAM_AUDIO_SCOPE=system` captures everything the PC plays instead, which
includes your music and your notifications. `host-agent --probe-audio` says
which side of that line you are on.

**It serves 540p60 by default**, and raises that only from measurement. A
host's own panel is also a ceiling: a 1366×768 screen cannot present 1080p
whatever the encoder could do. A virtual display driver lifts both, and lets
a machine host with no monitor attached at all.

---

## Updating

You do not. The service does it: the control plane offers a build, the agent
confirms the machine is idle, and the service downloads it, checks the
SHA-256, installs it beside the current one and restarts into it. Your
`config\` is untouched.

If a new build will not stay up — three failures inside ninety seconds — the
service puts the previous one back on its own. That is why `agent\previous.txt`
and the directory it names are never tidied away.

The service itself is never updated automatically, because it is the thing
performing the update. A new one is a deliberate reinstall: stop the service,
replace `bin\gameroom-service.exe`, start it again.

---

## When something is wrong

**`service.log`** in `C:\ProgramData\GameRoom\` is the first place to look.

**The agent says nothing at all.** If you set `RUST_LOG`, make sure it does
not filter the agent out: `RUST_LOG=gameroom_service=debug` silences the
supervised child completely, so an agent failing to start looks like silence
rather than an error. Use `RUST_LOG=gameroom_service=debug,host_agent=info`.

**Sessions start and no game appears.** Check `config\games.json` exists and
its paths are right. Without it the agent falls back to a provider that runs
nothing.

**The dashboard says the machine stopped reporting.** The agent heartbeats
every fifteen seconds and the platform marks a host offline after sixty. If
the service is running and the machine is on, check that the control plane
address in `--status` answers.

**A controller does nothing.** ViGEmBus is not installed. `Get-Service
ViGEmBus` should say Running.

---

## Reporting something

Issues are open here. For anything that looks like a security problem, please
report it privately rather than in an issue.
