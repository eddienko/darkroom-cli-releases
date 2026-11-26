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
  `agent` in DarkRoom. DarkRoom refuses it every security-sensitive account operation
  (making or rotating keys and notebook tokens, ending other sign-ins: `auth.agent_forbidden`,
  final, never a request for a code), and applies an assistant's rules to it (see
  `darkroom mcp`): only Workflows staff marked for assistants, custom jobs and Sessions only
  with `agent_actions`, a limit of jobs and Sessions per hour, stopping only Sessions an
  assistant started. `job submit`, `job rerun` and `job cancel` say so in plain words when
  DarkRoom refuses one of these (`agent.workflow_not_allowed`, `agent.capability_missing`,
  `agent.rate_limited` with when to try again, `auth.agent_forbidden`; exit 1). It still needs the
  human to approve in the browser; nothing in `darkroom` can approve it. A computer
  keeps one sign-in (`~/.darkroom/v2`, or `$DARKROOM_CONFIG_DIR`), so signing in as the
  other kind than the stored one (`--agent` over a plain sign-in, or the reverse) is
  refused unless you add `--force` (it then replaces and ends the old one) or log out
  first; use a separate `DARKROOM_CONFIG_DIR` to keep both.
- `darkroom logout`: ends this computer's sign-in on the server and removes it locally. A storage key (`darkroom
  storage login`) is not ended: stderr says so; run `darkroom storage revoke` before logging out.

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

## darkroom storage

Object storage: your locations, the objects in them, copying to and from them, deleting, and the S3 key this
computer uses. The commands are for API v2 (the 0.x
`darkroom storage ls | cp | rm | stat | mb` of the old client belong to API v1 and are not part of 2.x: use
these instead, and keep a 0.x binary only for what 2.x does not do yet).

**The key.** Object storage is reached with a personal, expiring S3 key (DarkRoom: Account > Credentials). You
have two key slots in all; `darkroom` owns at most one of them, named `cli-<host name>`, and the web app may hold
the other. `darkroom` never creates a key by itself: a missing or expired key is reported (`run darkroom storage
login`). The secret is returned once by DarkRoom and kept in `~/.darkroom/v2/object-key.json` (or
`$DARKROOM_CONFIG_DIR`), written atomically with mode 0600 in a 0700 directory, refused on Linux and macOS if
others can read it, and bound to the DarkRoom address and user name it was made for (another is never used). It is
never printed, logged, put in a URL or an error, or sent to DarkRoom. A file that can't be used (readable by others, not
a regular file, damaged, or holding a value that isn't plausible, such as a region with a line break) is never used and its
content never shown: delete it and run `darkroom storage login` (which finds the key DarkRoom still holds for this computer,
by name, and offers to replace it; `login` and `revoke` offer to ignore or delete such a file on a terminal). Ctrl-C at a
prompt ends the command with nothing changed. On Windows no permissions are set (the same
posture as the sign-in file): the file is as private as your profile directory. Anyone who can run programs as you
can read it, which is why an AI agent's sign-in (`login --agent`) is refused here. Expiry (default 30 days, 1 hour
to 1 year as DarkRoom sets it) is enforced by DarkRoom's sweeper shortly after the time, not at the instant; from
the expiry on this computer no longer uses the key.

### darkroom storage ls [flags] [LOCATION[:PATH]]

With no argument: your object-storage locations, `LOCATION  ACCESS  URI  PERSONAL KEY  USED  NAME` (`PERSONAL KEY` is what DarkRoom says: a personal key can reach the
location, not that this computer has one; with no usable key here one line on stderr says so; `USED` is the project's used space and quota where DarkRoom knows them, not only
yours). `--json`: an array of `{location, id, name, access, uri, endpoint, bucket, prefix, personal_credentials,
used_bytes, quota_bytes, usage_as_of}`. No key is needed for this list.

With a location: the objects and folders directly under `PATH`, from an S3 listing (delimiter `/`, every page, always
with your own prefix in that location, so the root of the `users` bucket, which the storage system refuses, is never
listed). `LOCATION` is a name such as `processed` or an id such as `loc_processed`; the locations come from DarkRoom,
never from a bucket list (a plain `aws s3 ls` reaches only one of the two storage systems). `PATH` is relative to your
prefix in that location; one leading `/` is the root (`users:/data` is `data`). `..` and `.` segments, empty segments
(`a//b`), backslashes, control characters, invalid text encoding and keys over 1024 bytes are refused with exit status 2
before anything is sent. A `PATH` ending in `/` is a folder; without it, a folder of that name is listed if there is
one, else the single object is described. Names are in the order S3 returns them (byte order), `/` after a folder,
control characters shown as `?`.

| Flag | |
|---|---|
| `-l`, `--long` | A table `TYPE SIZE MODIFIED NAME` (`dir` or `file`; size in bytes, `-` for a folder; time in your zone). |
| `--human` | With `-l`, sizes in KiB, MiB, GiB. |
| `--json` | One array on stdout and nothing else, streamed: `{name, path, type: "file"\|"dir", size, last_modified}` (`path` relative to your prefix; `size` and `last_modified` are `null` for a folder). If the listing fails part way the array is closed after the entries already written (valid JSON, possibly incomplete), the error is on stderr and the exit status is 1; a failure before listing starts leaves stdout empty. |

An empty location prints nothing (`[]` with `--json`), exit 0; a path with nothing under it is exit 1. The request
goes only to the location's endpoint, which must be `https://` (as for DarkRoom itself, `http://` only for localhost)
and the same as the key's, with path-style addressing and the SDK's default checksums switched off (the storage
system refuses them). A refusal is explained after the fact: from the S3 error (`RequestTimeTooSkewed`: your clock;
`AccessDenied`, `InvalidAccessKeyId`, `SignatureDoesNotMatch`) and one read of your credentials, which says whether the
key was revoked or rotated elsewhere (for example in the web app), expired, or does not reach that location. A
warning on stderr says when the key has under two days left. Exit status: 0; 1 a failure; 2 a wrong command line.

### darkroom storage cp [flags] SRC DST

Copies one file or (with `-r`) one folder between this computer and a location: upload (`SRC` local, `DST`
`LOCATION:PATH`) or download. Copies between two locations or between two local paths are refused (exit 2). `PATH` is
as for `storage ls`; if `DST` is an existing folder or ends in `/` the source keeps its name inside it, otherwise `DST`
is the new name. A remote folder is written with a trailing `/` (`-r` without it, on something that is not an object, is
refused with "did you mean data/").

| Flag | Meaning |
|---|---|
| `-r` | copy a folder |
| `--force` | replace destinations that already exist (otherwise each one is refused) |
| `--dry-run` | print the plan (every item, sizes, total, what replaces, what is skipped) and stop; only listing and HEAD requests are made |
| `--strict` | refuse the run if anything would be skipped |
| `--concurrency N` | requests in flight for an upload, 1 to 16 (default 4) |
| `--quiet` | no progress, no summary |
| `--json` | ONE document on stdout: `{command, direction, dry_run, files, bytes, skipped, refused, failed, error, results: [{path, local, bytes, status, error, note}]}`; valid also when the run fails (`error` says why); a failure before the plan (wrong path, no key, read-only location) writes nothing to stdout |

**Plan first.** Everything is listed and checked before the first byte moves; one item that breaks a rule refuses the
whole run (nothing is copied, exit 1). An existing destination is such an item unless `--force`: upload, the key exists;
download, the file exists. Nothing conditional is sent (no `If-None-Match`): the check and the write are separate
requests, so a writer that appears in between can still be overwritten (a small race). A location whose access is `read`
refuses an upload before any request, with DarkRoom's explanation. Large listings are streamed: the plan is built twice
(once to check, once to transfer) and holds only a hash per name (and per folder), never the items; with `--json` it also
holds at most 10,000 results and at most 10,000 failures (`truncated: true` says a list was cut; the counters are exact; `--json` `path`, `local`, `error` and `note` are
sanitised, control characters shown as `?`, so a `local` path is not byte-exact),
and an upload holds the names of one folder at a time (a folder with millions of entries needs memory for them). Pass 2 leaves
alone anything the plan did not contain.

**Upload.** A file of up to 8 MiB is one PutObject; a larger one is a multipart upload with parts of `max(8 MiB,
size/9000)` rounded up to a MiB (so never more than 9,000 of the 10,000 parts S3 allows); above 5 TiB it is refused. Parts
go `--concurrency` at a time. A failed upload and Ctrl-C (or SIGTERM) abort the multipart upload this run created; a
second Ctrl-C ends the program at once and may leave parts behind. The CLI cannot list unfinished uploads in the `users`
location (the storage system does not grant `ListBucketMultipartUploads` there), so it only aborts what it started in the
same run; leftover parts are for the administrator. Server errors (5xx), throttling and broken or stalled connections are
retried three times with a growing pause; `AccessDenied`, `InvalidAccessKeyId` and `SignatureDoesNotMatch` never are
(they are explained with one read of your credentials and the run stops). If another command rotated the key meanwhile
it is read once more (under the key lock) and the request repeated. A key that is expired or has under 5 minutes left is
refused before anything is sent; under 2 days left is warned about; a key that expires during the run stops it. Symbolic
links and special files inside a folder are skipped and listed (never followed); `--strict` refuses the run for them. A name
that cannot be a key (control characters, a backslash, invalid UTF-8) is refused. A source that cannot be opened is refused in the
plan. A file whose size or modification time changes between the plan and the end of its read **fails that item**: a
multipart upload is aborted and nothing is published; a small file is sent in one request, which may already have
published the object, and the message then says "the object may hold a partial copy: run the command again with
`--force`". A file that appears after the plan is skipped with a warning (run again). Three failures **of the storage** in
a row stop the run; failures of this computer's own files (an unreadable file, a full disk) never count toward that. There
is no resume of an upload after the program ended: run it again (with `--force` for what is already there).

**Download.** Each file is written to `NAME.part` (and `NAME.part.etag`, the object's ETag when the part was started)
and renamed when complete. Running the command again continues an interrupted part: a HEAD gives the current ETag, and
only when it equals the recorded one is the rest fetched with `Range` and `If-Match`; otherwise (or when the server
ignores the range) the file starts again. The record is the single line `darkroom-etag-v1 ETAG` (anything else is not a valid record). A part with a valid record of
**another version** of the object is this tool's own and is replaced, with a `warning: replaced NAME.part: ...` line on stderr (the plan says so too). A
`NAME.part` (or a stray `NAME.part.etag`) **without** a valid record is taken for your own file: it is
refused in the plan (and in `--dry-run`, which marks a continued part `RESUMES the part` and a forced replacement
`REPLACES NAME.part`) and is replaced only with `--force`: **a part without a valid record is never deleted without `--force`**. A part, a record or a
`NAME.part.lock` that is a link, a pipe or a folder is refused by name and never opened or removed (an old lock is taken over only if it is a regular file); the record is created exclusively
and read without following a link. One run at a time writes a file: `NAME.part.lock` is created exclusively, refreshed
while held, removed on exit, and taken over when nobody refreshed it for 2 minutes (a second run says "another darkroom run is
writing this file ... if no other darkroom is running, the lock is taken over after 2 minutes"). The part is written at its offset (not appended) and must have exactly the object's size before
it is renamed. A file written earlier in the same run is never replaced, even with `--force`; two keys that macOS sees as one
name through Unicode normalisation (NFC/NFD) are only caught when the second is written, and fail that item (the plan cannot
tell without a normalisation library). Downloaded files are 0644 in 0755 folders (your umask applies; use `umask 077` for
private data), nothing is fsynced before the rename, and no checksum of the content is checked (a multipart ETag is not a
content hash). Local symbolic links are never followed, in the destination's folders or as the file or its part. The plan refuses: keys with `..`, an absolute path, empty segments, a backslash, control characters or
invalid UTF-8; names Windows does not allow (reserved names, `<>:"|?*`, a trailing dot or space) on Windows; a key that
is both an object and a folder (named, with its partner, on stderr); keys that differ only in case on macOS and Windows;
a key that is the temporary name (`x.part`, `x.part.etag`) of another. Folder marker objects (keys ending in `/`) are
skipped. The listing asks for URL-encoded keys and decodes them only when the storage system says it did so.

Exit status: 0; 1 a failure or a refusal; 2 a wrong command line or path.

### darkroom storage rm [flags] LOCATION:PATH

Deletes objects. **Permanent: no undo, no trash.** Never deleted, whatever the flags: the location's root (your whole
prefix in `users`, or the whole bucket). A path ending in `/` is a folder and needs `-r`; with `-r` the path must end in
`/` (so `data` can never reach `data2/`). The objects are listed first (streamed) and you see the count, the total size and
the first names (control characters shown as `?`), then must type the word `yes`; without a terminal `--yes` is
required (exit 2); `--dry-run` shows the same and deletes nothing. Deleting is one DeleteObject per object, 8 at a
time (the batch call is not used); a failure does not stop the others and all are reported (exit 1); an object already gone
counts as deleted; each object is deleted by its exact listed key. A read-only location is refused with DarkRoom's
explanation. `--quiet`, `--json` as for `cp` (`{command, dry_run, files, bytes, skipped_unsafe, failed, error, results}`, real sizes from the listing; skipped keys are in
`results` with status `refused` and a note; the lists hold at most 10,000 entries, `truncated` says so, the counters are exact).
The second listing may not exceed the count of deletable keys you confirmed (the run stops).

**Unsafe keys (owner decision).** A key with a `.`, `..` or empty segment or a leading `/` could be read as another key by a proxy that
normalises paths, so it is never deleted by this command, whatever the flags (there is no flag to delete it; use another tool).
With `-r` such keys are **skipped**: the rest of the folder is deleted as planned, the skipped names are listed (sanitised) with the
reason `unsafe key (.. / empty segment / leading slash)`, and the exit status is **1 whenever anything was skipped**, also for
`--dry-run` (0 when nothing would be skipped), so scripts notice. The plan and the question show both numbers: "delete N objects (X) under
LOC:PATH; skip M whose names cannot be deleted safely"; `--yes` skips the question but the line is still printed on stderr. The final line
reads "Deleted N; M skipped (unsafe names: delete them with another tool); F failed". Unsafe keys are never deleted in the second listing
either; if their number there differs from the first listing the run says so once ("the listing now has K unsafe key(s) (K0 in the first
listing)"; no key is named), sets `unsafe_changed: true` in `--json`, exits 1 and keeps deleting the safe keys (the first listing's rows and
`skipped_unsafe` stay the reported set; an equal number swapped for other keys is not noticed; a growing count of safe keys stops the run). A single key addressed
directly (without `-r`) with such a path is refused (exit 2). A folder of only unsafe keys deletes nothing (exit 1). Keys whose
names `PATH` rejects (control characters, backslash) are reachable only through `-r` of the parent folder. Three failures of the storage in
a row stop the run.

### darkroom storage login | status | rotate | revoke

- `storage login [--expires-in 90d] [--json]` makes this computer's key (`POST /credentials`). DarkRoom asks you to
  confirm it is you when your sign-in is older than its step-up window (10 minutes by default; a CLI token refresh does
  not renew it): `darkroom` then asks on the terminal for a code from your authenticator app (6 digits) or a recovery
  code, and sends it only in the request body (never from the command line or the environment; it is shown as you
  type). Three wrong codes stop it; DarkRoom's own limit on wrong codes (`429`) is reported with its wait. A
  non-interactive run (no terminal) is refused with exit status 2: a script or CI cannot create a key. Under an
  agent sign-in it is refused (exit 1, checked on this computer only). If this computer already has an active key it
  stops; if the stored key is no longer active in DarkRoom it makes a new one. **A lost answer:** creation and rotation
  have no idempotency key, so an answer that is lost may leave a key holding a slot with no secret here; `login` says
  so, and the next `login` finds an active key with this computer's name that nothing here can use and, after you
  confirm, revokes it (a short time with no key) and makes a new one. No free slot (two keys held) is reported with
  the keys' names, before anything is created. When the storage system or its expiry check is not answering (either of
  its two gateways may be down) nothing is created and the message says so.
- `storage status [--json]`: the stored key's key id (never the secret), expiry, what DarkRoom says about it (active,
  revoked, expired, or no record: the web app can change it), the slots in use and the locations it reaches with their
  access. `--json`: `{state, server, username, credential_id, key_id, endpoint, region, expires_at, locations,
  keys_active, keys_max, files}`, `state` being `none`, `active`, `expired`, `revoked` or `unknown`. Exit 0 whenever it
  could report.
- `storage rotate [--expires-in 90d] [--json]`: DarkRoom's rotation (the new key is made in the free slot, then the old
  one revoked, within about a minute). The same terminal, step-up, agent and lost-answer rules as `login`. A rotation
  needs a free slot: with two keys held it says so before calling DarkRoom and, on a terminal, offers to revoke this
  key first and make a new one instead, with a warning about the gap (anything using the key fails meanwhile, and a
  failure to make the new key leaves you without one); the alternative is to revoke the other key and rotate. An expired
  or revoked key cannot be rotated (`login` instead). Files that `storage setup` wrote with a copy of the secret are
  updated, but only where they still hold the old one.
- `storage revoke`: revokes the key and, **only after DarkRoom answers 200**, deletes the local file and the copies
  `storage setup` wrote (where still the old secret). A gateway that is down (`503`) leaves the key active and the file
  in place: run it again. A revoked key may keep working for a minute or two (the gateways cache accounts). Nothing
  to revoke is exit 0. `darkroom logout` does not revoke the key (it says so on stderr): revoke before logging out.

Exit status of all four: 0; 1 a failure or a refusal; 2 a wrong command line or no terminal.

### darkroom storage setup env | aws | rclone

**Environment variables win.** Exported `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` or `AWS_SESSION_TOKEN` beat
`AWS_PROFILE`, so they override the darkroom profile: unset them if `aws` reports `SignatureDoesNotMatch`. After
`storage rotate` the key id stays the slot's id but the secret changes, so variables exported by `setup env` go stale.
`storage status` warns (stderr, and `warnings` in `--json`; never the values) when any of the three is set.

All need a usable key and read only your files. They never write the secret anywhere except where stated.

- `setup env`: `export` lines (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`,
  `AWS_ENDPOINT_URL_S3`, `AWS_REQUEST_CHECKSUM_CALCULATION` and `AWS_RESPONSE_CHECKSUM_VALIDATION` as `when_required`)
  for `eval "$(darkroom storage setup env)"`. Writes nothing. **The secret is on stdout**; path-style addressing cannot
  be set by environment variable.
- `setup aws [--print] [--force] [--write-credentials]`: the profile `[profile darkroom]` in `$AWS_CONFIG_FILE`, else
  `~/.aws/config` (a symbolic link is followed and kept; other lines and profiles are untouched; the file is replaced
  atomically): `region` (DarkRoom's, else `us-east-1`), `endpoint_url`, `s3 = addressing_style = path`, both checksum
  settings `when_required`, and `credential_process = "/absolute/path/to/darkroom" storage credential-process
  [--config-dir "DIR" --]` (the directory only when `$DARKROOM_CONFIG_DIR` is set). The AWS CLI runs it for each call:
  the secret stays in darkroom's file and nothing goes stale on rotation. `--write-credentials` copies the key into
  `$AWS_SHARED_CREDENTIALS_FILE`, else `~/.aws/credentials` (mode 0600) instead; `rotate` keeps that copy current and
  `revoke` removes it. An existing, different `[profile darkroom]` is never replaced unless `--force`; a file with two sections of that name is
not edited (remove the extra one); a CRLF file keeps its line endings; `--print` shows
  what would be written (never the secret) and changes nothing. A warning is printed if the AWS CLI on the `PATH` is
  v1 or older than 2.13. Use it with the whole address: `AWS_PROFILE=darkroom aws s3 ls s3://space-users/jdoe/` (a
  bare `aws s3 ls` lists the buckets of only one of the two storage systems).
- `setup rclone [--print] [--force] [--aws-profile]`: the remote `[darkroom]` in `$RCLONE_CONFIG`, else
  `$XDG_CONFIG_HOME/rclone/rclone.conf` or `~/.config/rclone/rclone.conf` (`%APPDATA%\rclone\rclone.conf` on Windows),
  mode 0600, an S3 remote with `provider = Other`, `force_path_style = true`, the endpoint, a region, and the key (an
  extra copy of the secret, kept current by `rotate`, removed by `revoke`). `--aws-profile` writes the remote without a
  secret (and forgets any record of a secret copy at that path, so `rotate` and `revoke` say nothing about it) (`env_auth = true`, `profile = darkroom`): rclone then runs the AWS profile's `credential_process`, so
  `setup aws` is needed as well (tried with rclone 1.76 against a stand-in only). An encrypted `rclone.conf` is not
  edited. Use `rclone lsf darkroom:space-users/jdoe/`; do not list the remote's buckets.
- `storage credential-process` (hidden; run by other programs): stdout is `{"Version":1,"AccessKeyId":…,
  "SecretAccessKey":…,"Expiration":…}` and nothing else. It makes no request, refreshes no token and prints no
  notice; when the key is missing, expired, unusable or is not for the stored sign-in it exits 1 with one line on
  stderr ending `run darkroom storage login`. It needs the stored sign-in only to check the key belongs to it.

## darkroom mcp [--read-only]

Serves DarkRoom tools to an AI assistant on this computer over the Model Context Protocol
(MCP), on standard input and output. The assistant starts it (register the full path of
`darkroom` and the argument `mcp` with it; see the README, for Claude Code with
`claude mcp add --scope user`); you don't run it yourself.

- **Agent sign-in only.** It runs only with a sign-in made by `darkroom login --agent` in the
  configuration directory it uses (`~/.darkroom/v2`, or `$DARKROOM_CONFIG_DIR`). With a
  person's sign-in, or none, it sends no token (only the usual version check) and exits 1, saying to run `darkroom login
  --agent`. At start it also asks DarkRoom which kind it recorded for the sign-in and stops
  (exit 1) if DarkRoom holds it as a person's, or if the sign-in has ended; if DarkRoom can't
  be reached it says so on stderr and starts anyway.
- **The shared sign-in.** It reads the stored sign-in again before every request, so a
  refresh made meanwhile by another `darkroom` command is used, not repeated (refresh tokens
  are single-use; refreshes are serialised by a lock in the configuration directory). If the
  stored sign-in is removed, replaced by a person's sign-in or moved to another DarkRoom while
  it runs, the tools stop with a message saying to run `darkroom login --agent`; a person's
  token is never sent.
- **Read tools and tools that act.** The read tools only read, through DarkRoom, as you, and
  are marked read-only (`readOnlyHint`). The tools that act are marked not read-only, and
  `cancel_job` and `stop_session` destructive (`destructiveHint`). No tool uses this
  computer's storage key (`darkroom storage login`), and no tool takes or returns a token,
  key or notebook token.
- **`--read-only`** serves only the eleven read tools of the first version (`list_storage_locations`
  to `get_workflow` below): no other tool is listed, and a call to one is refused as an
  unknown tool. Use it to keep an assistant that can only look.
- **Idempotency keys.** Each call of a tool that creates a Job or Session (`run_workflow`,
  `submit_job`, `rerun_job`, `start_session`) sends one `Idempotency-Key` (a new UUID, or the
  `idempotency_key` argument), uses the same key for its own retries, and returns it. After an
  unknown outcome (`outcome_unknown: true`) the assistant calls again with that key: DarkRoom
  returns the same Job or Session if it was created. Once one attempt's outcome is unknown,
  the call reports `outcome_unknown` with its key even if a later retry fails otherwise. A
  write refused `401 auth.unauthenticated` because another `darkroom` command renewed the
  shared sign-in at that moment is not sent again by itself: the tool says nothing was done
  and to call it again once.
- **Output.** Standard output carries only protocol messages; notices (the update notice,
  as for every command), warnings and errors go to standard error. A read tool tries a
  transient failure (network, `429`, `502`/`503`/`504`) again twice, briefly. A tool that acts
  tries the network, `502`, `503` and `504` again twice, briefly, with the same key, and
  **never a `429`**: `agent.rate_limited` (and any other `429`) is returned at once with
  `retry_after_seconds`.

| Tool | Arguments | Returns |
|---|---|---|
| `list_storage_locations` | none | your storage locations: id, name, kind, access, `browsable`, quota and usage |
| `list_storage_entries` | `location` (id, id without `loc_`, or exact name), `path`, `cursor`, `limit` (1-200, default 100) | one directory: names, kinds, sizes, modification times, `next_cursor`. Only locations with `browsable` true (object storage the deployment lets DarkRoom list); others are refused with an explanation |
| `list_jobs` | `state` (comma separated), `limit` (1-100, default 20), `cursor` | your Jobs, newest first: number, name, kind, state, workflow, times, failure |
| `get_job` | `number` | one Job: state, failure, workflow and parameters or command (each at most 16 KB), inputs, output location, resources, times |
| `get_job_log` | `number`, `tail` (1-200, default 100) | the last lines of the log, at most 200 lines and 64 KB (each line at most 4 KB), newest kept, labelled as untrusted output of the job |
| `list_job_outputs` | `number` | what the Job wrote at the top of its output directory: names, kinds, sizes, where; never contents |
| `list_sessions` | `state` (comma separated) | your Sessions (by default the ones not finished and those started in the last 7 days), at most 100, with `requested_via` (`agent`: started by an assistant) |
| `get_usage` | none | the compute you hold now against your limits |
| `get_capabilities` | none | the features, the actions you are entitled to, your limits, and `agent_policy`: `allowed_actions`, `agent_actions`, `submissions_per_hour`, `submissions_used`, `submissions_remaining`, `retry_after_seconds` |
| `find_workflows` | `query`, `limit` (1-100, default 25) | Workflows you may run: slug, name, summary, tags, current version, `agent_runnable` |
| `get_workflow` | `slug`, `version` (default: current) | one version: documentation (at most 8 KB), parameter schema (at most 32 KB), inputs, declared outputs, resources, `runnable`, `agent_runnable` |
| `list_sizes` | `usable_for` (`job`, default, or `session`), `app_id` | the Sizes you may use: id, name, resources, `default_for` |
| `list_environments` | `usable_for` (`job`, default, or `session`), `app_id` | the Environments you may use: id, name, version, status, `default_for` (description at most 1 KB) |
| `list_apps` | none | the interactive Apps you may start: id, name, availability, the Environments and Sizes each allows, its defaults |
| `preview_workflow_run` | as `run_workflow`, without `idempotency_key` | what the run would be (`POST /jobs/preview`): name, resources, Environment, parameters, inputs (with where they appear in the Job), mounts, output location, `warnings` (one says when the hourly limit is used up); nothing is created or counted. Read-only |
| `preview_job` | as `submit_job`, without `idempotency_key` | what the custom Job would be (`POST /jobs/preview`): name, resources, Environment or image, command, inputs (with where they appear in the Job), mounts, output location, `warnings`; refused as `submit_job` would be (for example without `agent_actions`); nothing is created or counted. Read-only |
| `run_workflow` | `slug`, `version`, `name`, `parameters` (an object), `inputs` (`[{name, location, path}` or `{name, uri}]`, at most 20), `output_location` (a location id), `size_id` or `resources` (`{cpu, memory_bytes, gpu}`), `idempotency_key` | the new Job (number, state, workflow, resources, inputs, output location) and `idempotency_key` |
| `submit_job` | `command` (1-16 strings), `args` (0-256), `environment_id` or `image`, `name`, `inputs`, `output_location` (required), `size_id` or `resources`, `idempotency_key` | the new custom Job and `idempotency_key`. The command runs as given, with no shell |
| `rerun_job` | `number`, `idempotency_key` | the new Job (`rerun_of` set) and `idempotency_key` |
| `cancel_job` | `number` | the Job's state (`cancelling`), or `already_finished` with its final state. Destructive |
| `start_session` | `app_id`, `environment_id`, `size_id` (defaults: the App's), `label` (at most 80 characters), `idempotency_key` | the Session (id, state `starting`, `requested_via`) and `idempotency_key` |
| `stop_session` | `id` | the Session's state (`stopping`), or `already_stopped`. Destructive |

Ids may be given with or without their prefix (`loc_`, `env_`, `siz_`, `app_`, `ses_`).

**What DarkRoom allows an assistant** (DarkRoom enforces it for every
request of an assistant's sign-in, whatever the program, and `darkroom mcp` adds no rule of
its own; your DarkRoom's staff set it):

- Workflows only if staff marked them `agent_runnable` (else `agent.workflow_not_allowed`);
- custom Jobs (also the rerun of a custom Job) and Sessions only if staff granted your account
  `agent_actions` (else `agent.capability_missing`), on top of your own rights;
- at most `submissions_per_hour` Jobs and Sessions created by assistants' sign-ins per hour
  (30 by default; else `429 agent.rate_limited` with the seconds until one more fits);
  previews, cancels and stops are not counted;
- cancelling any of your Jobs; stopping only Sessions an assistant started (else
  `auth.agent_forbidden`).

A tool that is refused answers a tool error that says why and what to do, with the facts as
structured content: `code`, `status`, `retry_after_seconds`, `limit`, field `errors` (JSON
pointers into the request), `current_version`, `outcome_unknown`, `idempotency_key`. These
rules bind the assistant's sign-in, not the program: an assistant with a shell can use a person's
sign-in on the same computer. They guard against accidents and injected instructions.

It cannot copy or delete anything, read a file's contents, list storage areas that are not
browsable, or make or show credentials.

**Your data.** What a tool returns (file and job names, parameters, commands, log lines and
output names, including those of custom Jobs) is sent to the AI provider behind your
assistant. The endorsed assistants are from Anthropic, Google and OpenAI. Following your
project's rules about which data may be sent to them is your responsibility.

Exit status: 0 when the assistant closes the connection (or on Ctrl-C), 1 a failure (not
signed in, not an agent sign-in, the sign-in ended, a protocol failure).

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
