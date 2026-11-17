# Changelog

All notable changes to this project will be documented in this file.


## v2.0.0-rc.5

`darkroom session cp`: copy files and directories between this computer and a running Session's
file browser (its home), like `cp` and `scp`: `darkroom session cp results.csv ses_1a2b3c:data/`,
`darkroom session cp -r ses_1a2b3c:data ./backup`. For small amounts of data (large data: the S3
keys, or `rclone`). Safe by default: an existing destination file is refused unless `--force` (the
whole plan is checked first); remote paths stay inside the home (absolute, `..` and hidden names
are refused before anything is sent); larger uploads go in pieces with a check that stops a
repeated piece from duplicating data, and a failed upload deletes its partial file; downloads are
written to `DEST.part` and renamed when complete (`--resume` continues one); a directory download
is extracted safely (nothing outside the destination, no symbolic links or devices). Uploading a
directory skips hidden entries and symbolic links and says so (`--strict` makes that a failure).
`--json` writes one summary to stdout; `--verbose` says which Session and address are used and
which step failed (never the token); progress is shown on stderr only in a terminal (`--quiet`
turns it off). It uses the notebook token exactly as `session desktop` does, and sends it only to
the Session's own address.

`darkroom session rm [--dry-run] [--yes] SESSION:PATH...`: deletes files and EMPTY folders in a running
Session's file browser. **Deletion is permanent** as far as `darkroom` can promise. There is no recursive
delete (`-r` is refused): a folder can contain a link to another folder and the Session's file interface may
not say which entries are links, so delete the files first, then the empty folder; a folder that is not empty
is refused with its entry count. The paths are literal (no wildcard expansion; a warning if one looks like a
pattern); the file browser root, `..`, absolute and hidden paths and a target named twice are refused before
anything is sent; a plan is printed first, `--dry-run` deletes nothing, and without a terminal the command
refuses unless `--yes` is given (an interrupt or the end of input at the question deletes nothing). An entry
the Session says is already gone is reported as such, not as deleted.

`darkroom session ls [-l] [--human] [--json] [SESSION][:PATH]`: lists a folder, or describes one file, in a
running Session's file browser, to see what is there before `session cp`. Names are sorted without regard to
case with `/` marking directories; `-l` is a table (type, size, time, name); `--json` writes every entry exactly
as the Session gives it. Control characters in names are shown as `?` in the text output. It reads a file's
description only, never its content. Same Session choice, notebook token and path rules as `session cp`.

`darkroom session list`: your Sessions (ID, label, App, state), by default those that are not
finished (`--state` chooses others, `--json` is for scripts and agents), to find a Session id for
`session desktop` and `session cp`.

The update notice and the minimum version apply to the new commands as to every other (no notice
with `--json`). The agent skill covers both commands.

## v2.0.0-rc.4

Update check. `darkroom` reads the oldest supported version and the newest announced version from
DarkRoom (`GET /config`; needs a DarkRoom that sends them, an older one changes nothing). Below the
oldest, commands that use DarkRoom stop with exit status 1 and say where to download a newer one
(`version`, `help`, `logout` and `completion` still work; `DARKROOM_SKIP_VERSION_CHECK=1` bypasses
it). Behind the newest, one plain notice on stderr at most once a day, only in a terminal and never
with `--json`; `DARKROOM_NO_UPDATE_CHECK=1` turns only the notice off (the minimum still applies). The download address is built in. What it
learns is kept in `update-check.json` in the configuration directory. `darkroom` never updates
itself.

## v2.0.0-rc.3

`darkroom login --agent`: the human signs in for an AI agent. The approval page says an agent is
asking, and the jobs that sign-in submits are labelled `agent` (not `cli`) in DarkRoom. It has
exactly the rights of any other sign-in and is approved only by the human, in the browser. A
computer keeps one sign-in, so signing in as the other kind (`--agent` over a plain `login`, or the
reverse) now asks for `--force`. Plain `login` sends nothing new. Needs a DarkRoom that knows the
`agent` kind.

An agent skill: `SKILL.md`, a short guide for AI agents that use `darkroom` (sign-in, the
safe job workflow, the exit statuses, what never to do, when to ask the human). It is
published at the root of the releases repository and attached to each release as
`darkroom-SKILL.md`, listed in `checksums.txt`. No change to any command.

## v2.0.0-rc.2

The same program as 2.0.0-rc.1: no change to any command. This release adds the user documentation to
the releases repository (a README with the install and verification steps, the command reference
`USAGE.md` and this changelog) and a short release page.

## v2.0.0-rc.1

darkroom 2.x is the client of the DarkRoom API v2 (0.x remains the API v1 client).

- `darkroom login` / `logout`: device sign-in, approved in the browser with a second factor;
  opaque tokens refreshed automatically; stored in `~/.darkroom/v2`.
- `darkroom session desktop [SESSION]`: a local bridge so a native VNC viewer can use a
  running Session's desktop: listens on `127.0.0.1` only, accepts one viewer (or one at a
  time with `--keep`) and carries it over a WebSocket to the Session, signed with the
  notebook token (`--token-file` or `$DARKROOM_NOTEBOOK_TOKEN`, never a flag value; the
  token goes only to the Session's own host). The WebSocket client is part of darkroom
  (no new dependency). Only the TurboVNC viewer has been tried. With `--url` and no
  SESSION, the Session id is taken from the `--url` path (a differing SESSION is an error).
- `darkroom job submit` for custom jobs through the DarkRoom API: Environment or permitted
  image, command and arguments after `--` passed verbatim (no shell), resources or a size,
  inputs, output location, `--dry-run`, `--wait`, idempotent retries. With neither
  `--environment` nor `--image` the job runs in the platform's default Environment for
  jobs (as it already uses the default size), so `darkroom job submit --output home -- echo ok`
  is valid.
- `darkroom job submit --shell 'LINE'`: shell mode, sends `["/bin/bash", "-c"]` and the line as
  an ordinary command (at most 4096 characters; the image needs bash). The job also gets
  `$DARKROOM_OUTPUT_DIR`, `$DARKROOM_INPUT_<NAME>` and `$DARKROOM_JOB_NUMBER` from the backend
  version that carries them (an older backend simply lacks them).
- `darkroom job get | list | cancel | wait | result`, all with `--json`. Cancelling a job
  that had already finished says so and exits 0.
- `darkroom job logs NUMBER`: a job's log, running or finished. The whole log is streamed;
  `--level`, `--query`, `--tail N`, `--follow` (until the job ends, with the exit status
  of `job wait`; interrupting never changes the job) and `--json` (one entry per line)
  read it a page at a time. On a terminal, control characters in the log are shown as
  escapes instead of being sent to it (`--raw` turns that off); an archive that may be
  incomplete is noted on stderr.
- `darkroom job rerun`: a failed or cancelled job again as a new job with the same
  specification, checked against current access and limits; `--wait`, idempotent retries.
- Removed the API v1 commands (kubeconfig/UserJob job submission, logs, shell,
  port-forward, storage and config): they stay in 0.x.
- Security: the DarkRoom address (`--server`, `$DARKROOM_URL`, or the one stored by
  `login`) must be `https://`; `http://` only for `localhost`, `127.0.0.1` and `[::1]`.
  Nothing is sent to any other address, a stored sign-in for one is not used (sign in
  again), and `login` refuses a sign-in page on a different origin than the server. A
  redirect is followed only within the server's own scheme, host and port; any other is
  refused before a request body or token is sent again. When `login` replaces a stored
  sign-in for an address that is not secure, it tells you the old one was not revoked and
  should be revoked in DarkRoom.
- `job wait`, `submit --wait`, `rerun --wait` and the polling of `login` survive a lost
  connection, `429` (honouring `Retry-After`) and `502`/`503`/`504` until `--timeout` or
  the expiry of the code, instead of ending at the first one; `401`, `403` and `404`
  still end them. Without `--timeout` (always the case for `submit --wait` and
  `rerun --wait`), a wait gives up with exit status 1 after 30 minutes of failures in a
  row, saying how long DarkRoom was unavailable; any answer starts the count again, and
  with `--timeout` the timeout alone decides (exit 124).
- Built with Go 1.27.2 (`go.mod` now has `go 1.27.0` and `toolchain go1.27.2`; no
  dependency changed), and the release workflow runs `govulncheck` before building.


## v0.2.17

- Fix issues around refresh token.


## v0.2.16

- Reverted GUI capabilities as it moves to a different package.
- Add flag to request GPU.
- Add refresh token to login.


## v0.2.15

Small release to add GUI capabilites with `darkroom job submit --gui`
on supported platforms.


## v0.2.8

- Improved job support:
  - add extra information to job status and list useful for debugging
  - job list accepts flags to filter jobs by status
  - job shell behaves like a normal shell now
- Support backend changes:
  - job status and job list now use database information
  - job submit now appends a unique job id to the job name
  - job status returns job logs if these exist


## v0.2.7

- Use GitHub secrets to add secure encryption and server keys
- Add validation of jobs resource requests
- Improved job shell command
- Improve error handling and reporting


## v0.2.5

- First attempt at colouring error messages and other terminal outputs
- Check that a file exists in the remote storage before deleting
- Initial support for a key to be sent in all request headers

### New commands

- `darkroom config set` - add a new command to set variables directly in the configuration file.
- `darkroom storage mb` - add a new command to create a bucket

## v0.2.4

Initial release of Darkroom CLI for storage management and authentication.

### Authentication
- `darkroom login` – authenticate with the remote API

### Storage Management
- `darkroom storage ls <path>` – list buckets or folder contents
- `darkroom storage cp <local> <remote>` – copy files to/from remote storage
- `darkroom storage rm <path>` – delete remote file or directory (`--recursive` supported)
- `darkroom storage stat <path>` – show metadata for remote object

### Utilities
- `darkroom version` – show Darkroom version, Git commit, and build date
- Configuration stored in `~/.darkroom/config.yaml` (encrypted)

