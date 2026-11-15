# darkroom

`darkroom` is the command-line client of DarkRoom. It signs you in to a DarkRoom
deployment, submits and follows jobs, and lets a native VNC viewer use the desktop of a
running Session. It is one program with no installer and nothing else to install.
`darkroom` 2.x speaks DarkRoom API v2.

This repository only carries the programs (see [Releases](../../releases)) and these
documents: [USAGE.md](USAGE.md) is the command reference, [CHANGELOG.md](CHANGELOG.md)
the history of changes.

## Install

The files are on the Releases page: `darkroom-linux-amd64` (Linux, 64-bit),
`darkroom-darwin-arm64` (macOS, Apple silicon), `darkroom-darwin-amd64` (macOS, Intel) and
`darkroom-windows-amd64.exe` (Windows, 64-bit), with `checksums.txt` listing their SHA-256
digests. There is no Linux ARM or Windows ARM file. Download your file and `checksums.txt`
from the same release into one folder, either in the browser or in a terminal (a plain
download; never pipe anything to a shell). In a terminal use the explicit version: the
`latest` link skips prereleases, so name the version.

macOS, Apple silicon (for Intel use `darkroom-darwin-amd64`):

```bash
VERSION=<the version shown on the Releases page, for example 2.0.0>   # replace this whole value
curl -fLO https://github.com/eddienko/darkroom-cli-releases/releases/download/v${VERSION}/darkroom-darwin-arm64
curl -fLO https://github.com/eddienko/darkroom-cli-releases/releases/download/v${VERSION}/checksums.txt
grep ' darkroom-darwin-arm64$' checksums.txt | shasum -a 256 -c -     # must say OK
chmod +x darkroom-darwin-arm64
mkdir -p ~/.local/bin && mv darkroom-darwin-arm64 ~/.local/bin/darkroom
```

Linux: the same with `darkroom-linux-amd64` and `sha256sum -c -` instead of
`shasum -a 256 -c -`. Make sure `~/.local/bin` is on your `PATH`.

Windows (PowerShell): download both files, print the digest and compare it with the line
for your file; the two must be identical (if not, delete the file and do not run it):

```powershell
$v = "<the version shown on the Releases page>"; $u = "https://github.com/eddienko/darkroom-cli-releases/releases/download/v$v"
Invoke-WebRequest "$u/darkroom-windows-amd64.exe" -OutFile darkroom-windows-amd64.exe
Invoke-WebRequest "$u/checksums.txt" -OutFile checksums.txt
(Get-FileHash .\darkroom-windows-amd64.exe -Algorithm SHA256).Hash.ToLower()
Select-String darkroom-windows-amd64.exe checksums.txt
```

Then put the file in a folder of your own, rename it `darkroom.exe`, add the folder to your
`PATH` and open a new terminal. Check the install with `darkroom version`: it prints
`darkroom X.Y.Z (DarkRoom API v2)`, the git commit and the build date.

A checksum shows the file was not damaged or changed on its way to you; it is not a
signature and does not prove who published it. These download commands were written from
standard tool behaviour and not run by us against the published release; only the
`grep ... | shasum` line was checked, locally, against the `checksums.txt` the release
script produces.

**The programs are not signed or notarised by Apple** (nor signed for Windows); this is
not done for these releases.

macOS (observed): a binary downloaded with the browser and then run is blocked by a window
saying it cannot be opened because Apple cannot verify it is free of malware, offering to
move it to the Bin (the wording may differ between macOS versions). Two ways round it,
standard macOS behaviour that we have not tried. Do either only for a file whose checksum
you verified:

1. In Terminal, before the first run, remove the browser's quarantine mark (this does
   nothing else): `xattr -d com.apple.quarantine ./darkroom-darwin-arm64` (use the name of
   your file).
2. After the first blocked attempt, open System Settings > Privacy & Security and choose
   "Open Anyway".

A file downloaded with `curl` or `gh` is not marked by the browser (not tried by us), so
the quarantine step does not apply to it.

Windows (SmartScreen), documented behaviour, not tried: it may show "Windows protected your
PC"; "More info", "Run anyway", or `Unblock-File .\darkroom.exe` in PowerShell. Antivirus
products may also flag unsigned programs.

## Quick start

```bash
darkroom login --server https://YOUR-DARKROOM-ADDRESS    # prints a code and an address; approve in the browser
darkroom job submit --output home --dry-run -- echo ok   # check and resolve, send nothing
darkroom job submit --output home -- echo ok             # submit
darkroom job get NUMBER                                  # state, failure, times
export DARKROOM_NOTEBOOK_TOKEN=...                       # or: --token-file PATH (made in the web app)
darkroom session desktop SESSION                         # then point a VNC viewer at the printed port
```

Everything else is in [USAGE.md](USAGE.md).

## For AI agents

[SKILL.md](SKILL.md) is a short skill file for an AI agent that runs `darkroom` for you: what
each command is for, the safe patterns, what never to do and when to ask you. Each release
also has it as the file `darkroom-SKILL.md` (listed in `checksums.txt`). Copy it to the place
your agent reads skills or instructions from, or point your agent at its address. We have not
checked which agent products read it.
