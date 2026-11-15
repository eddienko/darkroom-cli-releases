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

## darkroom login / logout

- `darkroom login [--server URL]`: device sign-in. Prints a code and
  the address of the approval page; waits until it's approved in the browser. The
  approval page must be on the same address (scheme, host and port) as the server you
  gave; if DarkRoom names another, `login` stops without showing it or the code
  (exit 1). While waiting it keeps going through a lost connection, `429` and
  `502`/`503`/`504` (see "Waiting and transient errors"), until the code expires
  (10 minutes by default); it stops at once on any other refusal.
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
