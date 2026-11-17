# darkroom 2.x command reference

`--server URL` (or `$DARKROOM_URL`) chooses the DarkRoom deployment for `login`; other
commands use the deployment you signed in to.

The address must be `https://`. `http://` is accepted only for `localhost`, `127.0.0.1`
and `[::1]` (development and the local stack). Any other address is refused before
anything is sent, with exit status 2 (usage), because sign-in tokens would travel
unprotected. The address must not contain a user name or password. Every command checks
the address it uses, including the one stored by `login`: a stored sign-in for an
address that is not secure (for example one made by an earlier darkroom against
`http://`) is not used and nothing is sent to it. The command says so and exits 1;
sign in again with `darkroom login --server https://…`. `darkroom logout` still
removes such a sign-in from this computer, without sending it, and exits 1. When
`login --server https://…` replaces such a sign-in, it can't end the old one either: it
says so on stderr, and the old sign-in should be revoked in DarkRoom.

A redirect from DarkRoom is followed only if it stays on the same scheme, host and port
as the address in use (and so still obeys the rule above). Any other redirect is refused
before the request, its body or the `Authorization` header is sent again: the command
fails with an error that names where DarkRoom tried to send it (the host, never a token).

An agent skill for these commands is `SKILL.md` in the releases repository (also each
release's `darkroom-SKILL.md`; `skills/darkroom/SKILL.md` in the source).

## Updates: the notice and the minimum version

DarkRoom tells `darkroom` (in `GET /config`: `api.min_client_version` and
`api.latest_client_version`) the oldest version the deployment supports and the newest the
operator announces. `darkroom` never updates itself, downloads or runs anything.

- **Below the minimum:** every command that talks to DarkRoom (all but `version`, `help`,
  `--help`, `logout` and `completion`; `login` is refused too) stops before sending anything
  else, with exit status 1 and a message naming this version, the minimum, the releases
  address and the advice to verify the download against `checksums.txt`. For support,
  `DARKROOM_SKIP_VERSION_CHECK=1` lets it run (it says so on stderr).
- **Behind the newest:** one plain notice on stderr, `A newer darkroom (X) is available; you
  have Y. It can be downloaded from <address> (verify it against checksums.txt).`, at most once every 24 hours, only when stderr is a terminal
  and the command is not using `--json`. `DARKROOM_NO_UPDATE_CHECK=1` turns the notice off only: the
  minimum is still enforced and the once-a-day refresh for it still happens.
- A cache older than seven days is not used to refuse a command (a fresh fetch is still tried);
  the notice may still use older data.
- A build with the default development version `2.0.0-dev` counts as a prerelease below
  `2.0.0`, so it is refused when the deployment's minimum is `2.0.0` or higher unless
  `DARKROOM_SKIP_VERSION_CHECK=1` is set (developers). The bypass line is not printed with
  `--json`.
- The releases address is built into `darkroom` (`https://github.com/eddienko/darkroom-cli-releases/releases`);
  nothing a server sends changes it, and a version from the server is shown only if it is a
  plain `MAJOR.MINOR.PATCH[-PRERELEASE]`.
- What was learned is kept in `update-check.json` in the configuration directory
  (`~/.darkroom/v2`, or `$DARKROOM_CONFIG_DIR`; readable only by you). When it is older than
  24 hours or for another server, a command makes one `GET /config` to your DarkRoom with a
  3 second limit before it runs; if that fails nothing is said, the command goes on (with what was last known, if
  anything) and no new attempt is made for 10 minutes. A development build whose version is not
  `MAJOR.MINOR.PATCH[-PRERELEASE]` checks nothing.

## darkroom login / logout

- `darkroom login [--server URL]`: device sign-in. Prints a code and
  the address of the approval page; waits until it's approved in the browser. The
  approval page must be on the same address (scheme, host and port) as the server you
  gave; if DarkRoom names another, `login` stops without showing it or the code
  (exit 1). While waiting it keeps going through a lost connection, `429` and
  `502`/`503`/`504` (see "Waiting and transient errors"), until the code expires
  (10 minutes by default); it stops at once on any other refusal.
- `darkroom login --agent`: the same, for an AI agent: the human runs it. The approval
  page shows that an agent is asking, and the jobs this sign-in submits are labelled
  `agent` in DarkRoom (a label only: the same rights as any sign-in). It still needs the
  human to approve in the browser; nothing in `darkroom` can approve it. A computer
  keeps one sign-in (`~/.darkroom/v2`, or `$DARKROOM_CONFIG_DIR`), so signing in as the
  other kind than the stored one (`--agent` over a plain sign-in, or the reverse) is
  refused unless you add `--force` (it then replaces and ends the old one) or log out
  first; use a separate `DARKROOM_CONFIG_DIR` to keep both.
- `darkroom logout`: ends this computer's sign-in on the server and removes it locally.

## darkroom job submit [flags] (-- COMMAND [ARGUMENT...] | --shell LINE)

| Flag | Meaning |
|---|---|
| `--environment ID` | The Environment to run in (`env_…` or its slug). With neither `--environment` nor `--image`, the platform's default Environment for jobs (the server refuses, naming `/environment_id`, when there is none) |
| `--image REF` | Or your own image (not with `--environment`) (needs custom image access; the deployment decides which registries are allowed) |
| `--size ID` | A size for jobs (default: the platform's default job size) |
| `--cpu N --memory M [--gpu G]` | Or explicit resources (`8Gi`, `512Mi`, `8G`) |
| `--input [NAME=]LOCATION:PATH` | An input (repeatable). `NAME` keys it in the job's configuration |
| `--input-uri [NAME=]s3://…` | An input by URI (repeatable) |
| `--output LOCATION` | Where outputs go (required); the job's own directory is allocated |
| `--shell LINE` | Run the line with `/bin/bash -c` instead of a command after `--` (see Shell mode) |
| `--name TEXT` | A name for the job |
| `--dry-run` | Check and resolve without submitting (shows where inputs appear inside the job) |
| `--wait` | Then wait until it finishes (as `job wait`) |
| `--idempotency-key KEY` | Repeat an earlier submission safely: the same key returns the same job |

The command and its arguments are an argument vector: no shell, no variable expansion.
Inside the job, `$DARKROOM_JOB_CONFIG` names a JSON file with the job's inputs (by name)
and output directory. The same paths are plain environment variables:
`$DARKROOM_OUTPUT_DIR` (the output directory), `$DARKROOM_INPUT_<NAME>` (each input's
path; `NAME` is the input's name in upper case, so `input_1` is `$DARKROOM_INPUT_INPUT_1`;
an input given by URI has none) and `$DARKROOM_JOB_NUMBER`. They exist from the backend
version that carries them; a job on an older backend simply lacks them.

**Shell mode.** `--shell 'LINE'` sends the ordinary command `["/bin/bash", "-c"]` with the
line as its one argument (the platform itself still runs no shell), so pipes, redirects,
`&&`, `$VARIABLES` and `*` work:
`darkroom job submit --output home --shell 'cd data && python run.py > out.txt'`. The image
needs `/bin/bash`. The line is at most 4096 characters, without NUL and not blank; longer
scripts belong in a file the job runs. Nothing is added (no `-e`): start the line with
`set -euo pipefail` to stop at the first error. `--shell` with a command after `--`, or
with an invalid line, is a usage error (exit 2).

If the answer to a submission is lost, `darkroom` says so, never retries by itself, and
prints the idempotency key to retry with safely.

Exit status: 0 created, 1 refused, 2 usage, 3 outcome unknown (the answer was lost; the
message "couldn't confirm the submission" prints the key to retry with). With `--wait`, once
the job exists, as for `wait` (0 succeeded, 3 failed, 4 cancelled, 124 timed out, 1 other
errors), so then 3 means the job failed.

## darkroom job get | list | cancel | wait | result

- `get NUMBER`: the job's state, failure, environment or image, command, resources,
  inputs, output location and times.
- `list [--state S] [--kind workflow|custom] [--search TEXT] [--limit N]`
- `cancel NUMBER`: asks DarkRoom to stop it. Cancelling again is safe. A job that had
  already finished (succeeded, failed or cancelled) is not an error: it prints
  `Job #N had already finished (STATE); nothing to cancel.` and exits 0 (with `--json`,
  the job as it is now; if the state cannot be read, the message omits it and the exit
  status is still 0 in text mode). It does not use the `wait`/`result` exit statuses.
  Exit status otherwise: 1 for other failures (no such job, not signed in, not
  permitted), 2 usage, 3 outcome unknown (cancelling again is safe).
- `wait NUMBER [--timeout 30m]`: polls at the interval DarkRoom suggests. Exit status 0
  succeeded, 3 failed, 4 cancelled, 124 timed out, 1 other errors. See "Waiting and
  transient errors" below.
- `result NUMBER`: how a finished job ended and its output location (same exit statuses).

All accept `--json`.

## darkroom job logs NUMBER [flags]

Shows a job's log: what the job printed, one entry per line as
`timestamp LEVEL message` (the level in capitals, padded to 8 characters). It works
while the job runs (what it has printed so far) and after it has finished (the log
DarkRoom kept). Only your own jobs; another's is "no such job" (exit 1).

Without options the whole log is **streamed** to the standard output (the API's
download), never held in memory, so a very large log is fine. The options below read
the log through the paged API instead (and print the same lines):

| Option | Meaning |
|---|---|
| `--level LIST` | Only these levels, comma-separated: `debug`, `info`, `warning`, `error`, `critical`. An unknown name is refused (exit 2) before anything is sent. |
| `-q`, `--query TEXT` | Only entries containing the text (not case-sensitive). |
| `--tail N` | Only the last N entries (1 to 10000), printed oldest first. Read backward from the newest a page at a time. The 10000 limit is because these entries are held in memory to print them in order; without `--tail` the whole log is streamed. 0, a negative or non-numeric N is exit 2. |
| `-f`, `--follow` | Print what exists, then keep printing new entries until the job finishes, then exit with its outcome (below). With `--tail`: the last N, then follow. |
| `--timeout D` | With `--follow` only: stop following after D (for example `30m`), exit 124. Using it without `--follow` is exit 2. |
| `--json` | One JSON object per entry per line (`cursor`, `timestamp`, `level`, `source`, `message`, `details`), as the API sends it, and nothing else on the standard output. With `--follow`, line by line. |
| `--raw` | On a terminal, don't escape control characters (below). |

`--level`, `--query` and `--json` without `--tail` or `--follow` print the whole matching
log.

**Following.** The log is read from the cursor of the last entry printed, at the
interval DarkRoom suggests (never faster than every 2 seconds), while the job runs.
When the job has finished, the rest is read and then the log DarkRoom archived, which
continues from the same position, so no entry is printed twice or missed; if the archive
hasn't appeared 2 minutes after the job finished, the command says so on the standard
error and stops with what it has. `--follow` of a job that has already finished just
prints its log. Exit status: 0 succeeded, 3 failed, 4 cancelled, 124 `--timeout`
reached, 1 other errors (not signed in, no such job, the log can't be read, DarkRoom
unavailable for 30 minutes without `--timeout`), 2 usage. The job's progress
(`job #N: running`) goes to the standard error. Following only reads: **interrupting it
(Ctrl-C) never changes the job.** A lost connection, `429` (`Retry-After` is honoured)
and `502`/`503`/`504` don't end it: they are retried as for `job wait` (see "Waiting and
transient errors" below). Without `--follow` nothing is retried: a failed request is an
error (exit 1).

**Notes and errors.** If the kept log may be incomplete (DarkRoom says it reached its
size limit, may have been rotated, or the job was cancelled), a line such as
`note: this log may be incomplete (truncated)` is written to the **standard error**, after
the log, never to the standard output; the paged options and `--follow` read that
state (the whole-log stream has no such mark). If the job's log can't be read at all
(DarkRoom answers `503 dependency.unavailable` for `logs`), nothing is printed to the
standard output, the message goes to the standard error and the exit status is 1.

**A job's log is untrusted text.** It is whatever the job printed. When the standard
output is a terminal, darkroom does not send control characters to it: ESC and the other
control characters (C0, DEL and C1) except tab and newline, and the invisible
text-direction and line-separator characters (U+200E, U+200F, U+061C, U+202A to U+202E,
U+2066 to U+2069, U+2028, U+2029, which could make a line read differently from what it
holds), are written as visible escapes (`\x1b`, `\u009b`, `\u202e`; bytes that aren't valid UTF-8 as `\xNN`), so a job can't move the
cursor, retitle the window or type into your shell. With `--json` the escapes are JSON's
(`\u001b`). The `job #N: STATE` progress line on the standard error is escaped the same way
when the standard error or output is a terminal. When the output is a file or a pipe the bytes are passed on unchanged;
`--raw` turns the escaping off on a terminal.

## darkroom job rerun NUMBER [--wait] [--idempotency-key KEY]

Runs a failed or cancelled job again as a **new** job with the same specification (the
same workflow version and parameters, or environment or image, command and arguments;
the same inputs, resources and output location), with its own output directory. The
original doesn't change; the new job shows `Rerun of #NUMBER`. There is no prompt: the
command is the confirmation.

DarkRoom checks the rerun against your access and limits as they are now, and changes
nothing to make it fit. A workflow version that can no longer run is refused, with the
current version and a link to run it from the web app (a new run, not a rerun).

Exit status: 0 created, 1 refused, 2 usage, 3 outcome unknown: the key to retry with is
printed, and a retry with it returns the same new job. With `--wait`, as for `wait`.
Accepts `--json`.

## darkroom session desktop [SESSION] [flags]

Lets a native VNC viewer on this computer use a running Session's desktop (a Session that
uses a desktop Environment). darkroom listens on `127.0.0.1` and, for the viewer that
connects, carries the VNC connection to the Session over a secure WebSocket
(`wss://<Hub host>/user/<user>/<session id>/desktop-websockify/`), signed with your
notebook token.

`SESSION` is a Session id (`ses_...`). Left out, it is the one Session you have that is
running; with none or several, they are listed (id, label, app) and the exit status is 1.
If `--url` is given and `SESSION` is not, the Session id is taken from the `--url` path (the
one path segment that is a `ses_...` id; none, or several different, and the rule above
applies). If both name an id and they differ, the command exits 2 before any request.
The Session must be `running` (it is not waited for); the address comes from DarkRoom's
`open_url` for it.

| Flag | |
|---|---|
| `--port N` | Local port on `127.0.0.1` (default 5901; `0` = any free port). The address is printed on stderr. A busy port is an error (try `--port 0`). |
| `--token-file PATH` | File whose first line is the notebook token. On Linux and macOS a file readable by group or others is refused (`chmod 600`), as `ssh` does for keys. |
| `--url URL` | The Session's base address instead of DarkRoom's. It must be `https://`, contain the Session id as a path segment (which then also names the Session when `SESSION` is left out), and be on the same host (and port) as the Session's `open_url`. |
| `--keep` | After the first viewer, keep accepting viewers, one at a time, until Ctrl-C. |

The notebook token is the one made in the web app (Account > Credentials, shown once).
Creating one from the CLI is not available. It is never a command-line value (the process
list and shell history would show it); it is read from, in this order, `--token-file`, then
`$DARKROOM_NOTEBOOK_TOKEN`. There is no hidden prompt (the Go standard library cannot turn
off echo portably, and darkroom takes no extra dependency for it); without either source
the command exits 2 naming both. The token is sent only to the Session's own address
(the host of its `open_url`), never to the DarkRoom server and never to a redirect target:
a redirect on the WebSocket upgrade is an error. It is never printed.

**One connection by default, and why.** The desktop's VNC server has no password: the
notebook token is the only protection, and the bridge adds it. Whoever can connect to the
local port while the bridge runs gets the desktop. So by default the bridge accepts one
connection, stops listening, and ends when that viewer disconnects. With `--keep`, anyone
on this computer can use the port while it runs (still one at a time: a second connection
while one is open is closed at once). The note is printed on stderr at start.

Nothing is written to stdout. Ctrl-C (or SIGTERM) closes both sides and exits 0, as does
the viewer or the Session ending the connection normally. Exit status: 1 for a failure
(the Session isn't running, a refused upgrade: 401/403 token wrong, expired or not for
this Session, 404 the Session has no desktop, a TLS error, a lost connection or a close
with an error status), 2 for a wrong command line or token source. TLS certificates are
always verified; there is no option to skip that. The connection to the Session is direct: proxy settings (`HTTPS_PROXY` and the like) are not used.

```bash
export DARKROOM_NOTEBOOK_TOKEN=...        # or: --token-file ~/.darkroom-notebook-token
darkroom session desktop ses_1a2b3c       # prints: Listening on 127.0.0.1:5901 ...
vncviewer localhost:5901                  # in another terminal
```

Only the TurboVNC viewer has been tried (with `websocat` as the bridge; this command's
own use with it has not been tried against a real Session yet).

## darkroom session list [flags]

Lists your Sessions, so that a Session id (`ses_...`) can be found from the command line for
`session desktop` and `session cp`. Table columns: `ID`, `LABEL` (truncated to 40 characters),
`APP`, `STATE` (with the progress, `phase: message`, while the Session is `starting`), one
Session per line. The address a Session is opened at is not in the table.

| Flag | |
|---|---|
| `--state STATE[,STATE...]` | Only Sessions in these states: `starting`, `running`, `stopping`, `stopped`, `failed`. Default: `starting,running,stopping` (the Sessions that are not finished). An unknown state exits 2 before any request. |
| `--json` | One JSON array on stdout and nothing else: for each Session `id`, `label`, `app` (`{id, name}` or `null`), `state`, `progress` (`{phase, message}` or `null`) and, while it is running, `open_url`. The full label is only here. |

At most 500 Sessions are fetched (following `next_cursor`); if there are more, `(and more)` is
printed on stderr. With none, the table is empty, `you have no Sessions` goes to stderr, and
`--json` writes `[]`. Exit status: 0 (also when there are none); 1 a failure (not signed in,
DarkRoom unavailable); 2 a wrong command line.

## darkroom session cp [flags] SOURCE... DEST

Copies files and directories between this computer and a running Session's file browser (the
one JupyterLab shows, whose root is your home in the Session), like `cp` and `scp`. It talks
to the Session's own JupyterLab interface at the address DarkRoom gives for the Session (the
host of its `open_url`, https only), signed with your notebook token: no DarkRoom API and no
other host is involved. **It is for small amounts of data.** For large data use the S3 keys
for `incoming` and `processed`, or `rclone`.

Exactly one side is remote, written `SESSION:PATH`. `SESSION` is a Session id (`ses_...`); empty
(`:PATH`) it is the one running Session you have, chosen as `session desktop` chooses it (with
none or several, they are listed and the exit status is 1). With `--url`, the Session id is taken
from the pasted address as in `session desktop`. A path whose part before the first colon is not
empty or `ses_...` (for example `C:\Users\me\f.txt`, `./a:b`) is local; write `./` before a local
name that would otherwise read as `SESSION:`.

```bash
darkroom session cp results.csv ses_1a2b3c:data/results.csv      # upload one file
darkroom session cp -r ./analysis :                              # a directory into your home (the only running Session)
darkroom session cp ses_1a2b3c:data/out.png ./                   # download into the current directory
darkroom session cp -r ses_1a2b3c:data ./backup                  # download a directory
```

**Paths.** `PATH` is relative to the file browser root (your home). An absolute path, any `..`
segment, a name that starts with a dot (the Session refuses hidden names), a backslash, a
control character (NUL included) or a path over 4096 bytes is refused with exit status 2 before
anything is sent. An empty path is the root: a directory as a destination, and not allowed as a
source. **Destination rules, as `cp`:** if `DEST` is an existing directory, or ends with `/`, each
source keeps its own name inside it; otherwise (one source) `DEST` is the new name; several sources
need a directory. Missing parent directories are created, on the Session and on this computer. A
directory source needs `-r` (exit status 2 otherwise).

**No silent overwrite.** An existing destination file is refused (exit status 1, nothing copied)
unless `--force`. For a directory the whole plan is checked before anything is sent or written, so a
refusal leaves nothing half-copied; a directory where a file belongs (or the reverse) is always refused.

**Upload.** A file that fits in one piece is sent in one request; a larger file goes in pieces
(`--chunk-size`, default 8 MiB, from 1 MiB to 32 MiB; the file is read as a stream, so memory is
bounded by one piece). The Session appends each piece, so a piece is never sent twice blindly: if a
piece's request fails in a way that leaves it unknown whether the Session got it (a lost connection,
a timeout, a 5xx), the file's size in the Session is read: unchanged, the piece is sent again (up to
three times); already including the piece, it is carried on; anything else, the upload stops. A failed
upload deletes its partial file in the Session; if that fails the message names the path. With
`--force` over an existing file sent in pieces, the old remote file is replaced: it is deleted first (the
Session would append otherwise), so **if the upload then fails, neither the old nor the new file
remains** (this is what `cp -f` does too). The cleanup of a failed upload deletes the partial file only if
its size is one this upload could have written; otherwise it leaves it and says so. If the local file
grows or shrinks while it is uploaded, a warning is printed on stderr and listed in `--json` as
`warnings`. Uploading a directory (`-r`) creates the directories and skips, and reports, hidden
entries (the Session refuses them), symbolic links (not followed) and other special files; the exit
status is still 0 for those skips unless `--strict`.

**Download.** A file is written to `DEST.part` (created exclusively; a symbolic link there is never
followed) and renamed when complete, so a half file is never under the final name. An existing
`DEST.part` is refused unless `--resume` (continue it) or `--force` (start over) is given. An
interrupted download keeps `DEST.part`; `--resume` continues it with a range request, checking that
the file's length in the Session (the total in the answer) is still what it was when the command
started, and that the final size matches: if the file changed length, it stops with a message (a
Session that ignores the range sends the whole file again, which replaces the part).
A directory (`-r`) is received as one zip archive and extracted: an entry that is absolute, contains
`..`, a backslash, a drive letter, a control character, is not valid UTF-8 or has a name over 255 bytes
stops the whole extraction before anything is written; symbolic links and devices are not created
(skipped, and reported); files get mode 0644 and directories 0755; more than 1 000 000 entries is
refused, and so is an archive that would expand (declared sizes added up) to more than
`--max-extract-size` (default 8 GiB) or, where the platform can say (Linux, macOS), to more than the
free disk space; the archive's own download is cut off at that size too, and the bytes written are
held to it. Names that differ only in case (on Windows and macOS), and on Windows reserved names
(`CON`, `PRN`, `AUX`, `NUL`, `CONIN$`, `CONOUT$`, `COM0`-`COM9`, `LPT0`-`LPT9` and the superscript forms, with or without an extension) and names ending
in a dot or a space (also `COM0`, `LPT0`, the superscript forms `COM¹`-`COM³`/`LPT¹`-`LPT³`, `CONIN$` and `CONOUT$`), stop the extraction before anything is written. The temporary archive is removed on every path. A failed extraction removes the files it wrote and the empty directories it created; with `--force`, files it had already replaced are not restored, and directories that already existed are left as they are. A name that turns out to be the same file as one already written from the same archive (for example Unicode-normalised names on macOS) stops the extraction. The archive is expected to contain the
directory itself as its first level (as JupyterLab's own "download as archive" does); if it does not,
its entries are extracted under `DEST` as they are.

| Flag | |
|---|---|
| `-r`, `--recursive` | Copy directories. |
| `--force` | Replace files that already exist at the destination. |
| `--strict` | Exit 1 if anything was skipped (hidden entries, symbolic links, special files). |
| `--resume` | Continue an interrupted download (`DEST.part`). |
| `--max-extract-size SIZE` | Refuse a directory download that would expand to more than this (default 8GiB; `KiB`, `MiB`, `GiB`). |
| `--chunk-size SIZE` | Piece size for larger uploads: bytes, or with `K`, `KiB`, `M`, `MiB`; 1 MiB to 32 MiB. |
| `--token-file PATH`, `--url URL` | As for `session desktop`. |
| `-q`, `--quiet` | No progress and no summary on stderr (errors are still printed). |
| `--json` | One JSON object on stdout and nothing else (below). |
| `-v`, `--verbose` | On stderr, before transferring: the DarkRoom server, the Session id and state and the Session's address (host and path); then one line per kind of request as it first happens (`GET contents`, `PUT contents`, `DELETE contents`, `GET files`, `GET directories`); and a failure names its step (for example `upload chunk 2 of 3`). Never the notebook token, the archive token or file contents. |

The notebook token is read exactly as for `session desktop` (`--token-file PATH`, else
`$DARKROOM_NOTEBOOK_TOKEN`; a file readable by group or others is refused on Linux and macOS; no
prompt; without either, exit status 2). It is sent only in the `Authorization: token` header to the
Session's own https address, never in a URL, argument, message or log, never to DarkRoom, and
never to a redirect target (a redirect is an error). The connection is direct (proxy settings are
not used) and certificates are always verified. A directory download sends a random `archiveToken`
that the Session's archive service requires; it is not the notebook token.

**Progress** (bytes, rate, elapsed, per file and total) goes to stderr only when stderr is a
terminal and `--quiet` is not given; at the end a summary of what was copied, skipped and failed is
printed on stderr. **`--json`** writes one object to stdout and nothing else:
`{"direction": "upload"|"download", "session", "destination", "copied": [{"source", "destination",
"bytes"}], "skipped": [{"path", "reason"}], "failed": [{"path", "error"}], "warnings": ["..."], "bytes", "seconds"}`; it is
written when a transfer was reached, also when some items failed (exit status 1), and not when the
command line, the token or the Session was refused before that.

**Limits.** Only what the file browser root contains is reachable; hidden files are skipped (the
Session refuses them); modes, owners and times are not preserved; symbolic links are not followed;
transfers are one file at a time, and an interrupted upload cannot be resumed (it is deleted and
started again); speed is whatever the connection gives (about 1.8 MB/s was measured on one link).

Exit status: 0 everything requested was copied (skips of hidden entries and symbolic links are still
0 without `--strict`); 1 a failure (the Session is not running, 401/403 the token is wrong, expired
or not for this Session, a path the Session refused, a refused overwrite, a TLS error or a lost
connection, or any item that failed); 2 a wrong command line, path or token source. Ctrl-C stops the
transfer, deletes a partial upload and exits 1.

## darkroom session ls [flags] [SESSION][:PATH]

Lists a folder, or describes one file, in a running Session's file browser (the one JupyterLab shows;
its root is your home), so that you can see what is there before `session cp`. The remote side is
written as for `session cp`: `ses_1a2b3c:data`, `ses_1a2b3c:` or just `ses_1a2b3c` for the root, `:data`
for the one running Session (chosen as `session desktop` chooses it), none at all for the root of that
Session; `--url`, `--token-file` and the token rules are those of `session cp`. A path that is absolute,
has a `..` segment, a hidden (dot) name, a backslash or a control character is refused with exit status 2
before anything is sent.

The command first asks what the path is (`GET /api/contents/<path>?content=0`, which never carries a
file's data); only for a directory does it then read the list (`content=1`). A file's data is never
fetched.

| Flag | |
|---|---|
| `-l`, `--long` | A table `TYPE SIZE MODIFIED NAME` (type `dir`, `file` or `nb` for a notebook; size in bytes, `-` for a directory; time in your local zone). |
| `--human` | With `-l`, sizes in KiB, MiB, GiB. |
| `--json` | One array of `{name, path, type, size, last_modified}` on stdout and nothing else: the exact strings the Session gave, every entry (`type` is `directory`, `file` or `notebook`; `size` is `null` for a directory). |
| `-v`, `--verbose`, `--token-file PATH`, `--url URL` | As for `session cp`. |

The default output is one name per line, sorted by name without regard to case, `/` after a directory.
**Hidden entries (names starting with a dot) are not listed:** the Session does not return them. In the
text output control characters in a name (and line, paragraph and bidirectional controls) are shown as
`?`, so that a name cannot move the cursor or send terminal escapes; `--json` keeps the exact name, so
treat it as data. An empty folder prints nothing (exit 0). Exit status: 0; 1 a failure (the Session is not
running, 401/403 the token is refused, no such file or directory, a TLS error or a lost connection); 2 a
wrong command line, path or token source.

## darkroom session rm [flags] SESSION:PATH...

Deletes files and **empty** folders in a running Session's file browser (its root is your home).
**Deletion is permanent** as far as `darkroom` can promise: it does not know whether the Session keeps a
trash and does not check. Look first with `darkroom session ls`, or use `--dry-run`.

**Recursive delete is not available** (`-r` is accepted only to be refused, with exit status 2 and no request
sent). A folder can contain a link to another folder, and the Session's file interface may not say which entries are
links (it may list a link as an ordinary folder), so `darkroom` cannot be sure: descending could delete files
outside the folder you named. Delete the files first (see them with `darkroom session ls`), then the empty folder. A folder
target is listed once (no descent): if it is empty it is deleted, otherwise that target is refused with its
entry count (`folder is not empty (N entries): delete its contents first`), counted as failed, and the other
targets go on. A folder that holds only hidden files may look empty (the Session may not list them), and the Session may then
refuse to delete it; that is reported as a failure.

Each target is `SESSION:PATH` (or `:PATH` for the one running Session); several targets are allowed, on one
Session. **There is no wildcard expansion:** the path is literal. A path with `*`, `?` or `[` is allowed (a
file may really be named so) but a warning says nothing is expanded; a `*` is normally expanded by your shell
against *local* files before `darkroom` runs. These are refused with exit status 2 before anything is sent: no
target, the file browser root (an empty path, `.`, `ses_1a2b3c:`), any `..`, an absolute path, a backslash, a
hidden (dot) name, a control character, a path that is not remote (`C:\x` is a local path), the same path
named twice. One refused target refuses the whole command.

| Flag | |
|---|---|
| `-r`, `--recursive` | Refused (exit 2): see above. |
| `--dry-run` | Print the plan and delete nothing (no `DELETE` is sent); it never asks. With `--json` the plan is an array of `{path, type, error}` (`type`: `file`, `empty_folder`, `refused_folder`, `missing`). |
| `--yes` | Skip the confirmation question only. The plan is still printed and every refusal still applies. |
| `--json` | One JSON object on stdout and nothing else: `{deleted, already_gone, failed: [{path, reason}], dry_run}`. |
| `-v`, `--verbose`, `--token-file PATH`, `--url URL` | As for `session cp`. |

**The plan** is printed on stderr: each target as a file, an empty folder, or refused with the reason (a missing
target says `no such file or directory`, counts as failed and the others still go on). **Confirmation:** unless
`--yes`, `Delete these? [y/N]` is asked on the terminal; anything but `y` or `yes`, the end of the input, and
Ctrl-C delete nothing and exit 1. Without a terminal and without `--yes` (and without `--dry-run`) the command
refuses with exit status 2, before sending anything.

**Deleting.** One `DELETE` per target, in order; one is never repeated. After a lost connection or a 5xx the
entry is looked at once: if it is gone the answer was lost, so it counts as deleted, otherwise it is reported as
failed. A refused token (401, 403), a TLS error or a lost connection ends the whole command. If the Session
answers 404 to the `DELETE` the entry was already gone: it is reported as **already gone**, in the text and in
`already_gone` of `--json`, and is never counted as deleted. If the Session refuses to delete a folder at its turn (it may not be empty now: something was
added) it is not forced. Names are shown with control characters as `?`.

Exit status: 0 every target was deleted or was already gone (or `--dry-run` with nothing wrong); 1 anything
failed or you answered no; 2 a wrong command line, `-r`, a refused target, or a non-interactive run without `--yes`.

## Waiting and transient errors

`job wait`, `job submit --wait`, `job rerun --wait`, `job logs --follow` and the polling of
`login` keep waiting through failures that say nothing about the job or the sign-in:

- the network failing (no connection, a dropped connection, an answer cut off, no answer
  within 60 seconds);
- `429`: DarkRoom asks to slow down. darkroom waits for the larger of the `Retry-After`
  it sent (at most 10 minutes) and the polling interval;
- `502`, `503`, `504`.

Each is tried again after the polling interval (at least 2 seconds for `job wait`),
doubled for each failure in a row up to 30 seconds (or the interval, if that is
longer), until the job finishes, `--timeout` is reached (exit 124; if no answer was ever
received the message says the state is unknown) or the sign-in code expires. The first
failure of a run is reported on stderr (`darkroom: …; will keep trying`), so a wait that
is struggling is not silent. Nothing is changed by retrying: waiting only reads.

Without `--timeout` (`job wait` without it, and always `submit --wait` and `rerun
--wait`, which have none), a wait gives up with exit status 1 when every attempt has
failed for 30 minutes in a row. The message says how long DarkRoom was unavailable and
that the job itself is not affected (`darkroom job get NUMBER` shows it), and the stderr
notice says "will keep trying for up to 30 minutes". Any answer from DarkRoom starts the
count again, so an intermittent outage never ends a wait. With `--timeout`, the timeout
alone decides (exit 124). `login` is limited by the life of its code instead.

Any other refusal ends the wait at once with exit status 1, for example `401` (signed
out), `403`, `404` (no such job) or `500`.
