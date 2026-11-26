# Changelog

All notable changes to this project will be documented in this file.


## v2.0.0-rc.7

This release also contains the changes listed for v2.0.0-rc.6, which was tagged but never published.

`darkroom mcp` stage 2: tools that act, under the rules DarkRoom now enforces for an assistant's sign-in (the CLI
adds none of its own). Eleven new tools: `list_sizes`, `list_environments`, `list_apps` (read-only),
`preview_workflow_run` and `preview_job` (`POST /jobs/preview` for a Workflow run or a custom Job; nothing is created
or counted; read-only), `run_workflow`, `submit_job`
(a custom Job; needs the `agent_actions` permission staff grant), `rerun_job`, `cancel_job`, `start_session` (needs
`agent_actions`) and `stop_session` (only Sessions an assistant started). The tools that act declare `readOnlyHint`
false, and `cancel_job` and `stop_session` `destructiveHint` true. Each tool that creates a Job or Session sends one
`Idempotency-Key` per call (a UUID, or the new `idempotency_key` argument), reuses it for its own retries and returns
it; an unknown outcome says to call again with it, and stays unknown when a later retry fails otherwise. A write
refused because another `darkroom` command renewed the shared sign-in at that moment says to call again once (it is
not resent by itself). A tool that acts never retries a `429` by itself:
`agent.rate_limited` is returned at once with `retry_after_seconds` (DarkRoom's own value, up to an hour; the read
tools no longer retry it either). DarkRoom's refusals are worded for the assistant (`agent.workflow_not_allowed`,
`agent.capability_missing`, `agent.rate_limited`, `auth.agent_forbidden`, validation errors with their pointers,
`limit.*`, entitlements), with the facts as structured content. `get_capabilities` returns `agent_policy` (allowed
actions, the hourly limit, used, remaining); `find_workflows` and `get_workflow` return `agent_runnable`;
`list_sessions` returns `requested_via`. `darkroom mcp --read-only` serves only the eleven read tools of the first
version (a call to any other is refused as an unknown tool): the way to keep an assistant that can only look. The
help, README (registration: `claude mcp add --scope user ... -- /full/path/darkroom mcp`; `/mcp` reconnects) and
the agent skill say what DarkRoom allows an assistant and what staff control, and that what tools return, including
the logs and output names of custom Jobs, goes to the assistant's AI provider.

`darkroom job submit`, `job rerun` and `job cancel` with an agent sign-in word DarkRoom's agent refusals in plain
language (`agent.workflow_not_allowed`, `agent.capability_missing`, `agent.rate_limited` with when to try again,
`auth.agent_forbidden`); `darkroom login --help` describes the rules of an agent sign-in.


`darkroom mcp`: read-only DarkRoom tools for an AI assistant on your own computer (Claude Code, Gemini CLI, OpenAI
Codex and others), as a Model Context Protocol server over standard input and output. It runs only with an agent
sign-in (`darkroom login --agent`; a person's sign-in or none exits 1, and DarkRoom is asked at start which kind it
recorded). Eleven tools, each declared read-only: `list_storage_locations`, `list_storage_entries` (browsable
locations only, through DarkRoom, never with this computer's storage key), `list_jobs`, `get_job`, `get_job_log` (at
most 200 lines and 64 KB, labelled as untrusted output), `list_job_outputs` (names, kinds and sizes), `list_sessions`,
`get_usage`, `get_capabilities`, `find_workflows`, `get_workflow`. No tool takes or returns a token or key. Standard
output carries only protocol messages. It re-reads the stored sign-in before every request, so it shares it safely
with other commands (single-use refresh tokens). What a tool returns is sent to the assistant's AI provider; the
endorsed ones are from Anthropic, Google and OpenAI, and following your project's data rules is your responsibility.
New module: the official MCP Go SDK (`github.com/modelcontextprotocol/go-sdk` v1.8.0, Apache-2.0/MIT) and, through
it, `google/jsonschema-go`, `segmentio/encoding`, `segmentio/asm`, `yosida95/uritemplate`, `golang.org/x/oauth2`,
`x/sync`, `x/sys`, `x/time`. The agent skill and `darkroom login --help` now say what an agent sign-in may not do
(DarkRoom refuses it making or rotating keys and notebook tokens and ending other sign-ins: `auth.agent_forbidden`).

`darkroom storage cp SRC DST [-r] [--force] [--dry-run] [--json] [--quiet] [--strict] [--concurrency N]` and
`darkroom storage rm LOCATION:PATH [-r] [--dry-run] [--yes] [--json] [--quiet]` (stage 2). `cp` copies between this
computer and an object-storage location (upload or download; not between two locations). The whole plan is listed and
checked before anything moves; `--dry-run` shows it; any item that breaks a rule refuses the whole run. An existing
destination is refused without `--force` (the check and the write are separate requests: a small race). Uploads of over
8 MiB are multipart (part size from the file size, at most 9,000 parts, files over 5 TiB refused), 4 parts at a time,
aborted on failure and on Ctrl-C (a second Ctrl-C exits at once); requests are retried with back-off for server errors
and broken connections, never for AccessDenied or a bad key (explained instead); the stored key is re-read once if
another command rotated it. Downloads go to `NAME.part` and are renamed when complete, an interrupted one continues with a
Range request guarded by `If-Match` when the object's ETag is the one recorded beside the part, local symbolic links are
never followed, and keys that would escape the destination, name a file in two ways or differ only in case on macOS and
Windows are refused. `rm` never deletes the location root, needs `-r` and a trailing `/` for a folder, lists first,
and asks you to type `yes` (`--yes` off a terminal); it deletes with one DeleteObject per object and reports every
failure. A source that changes during an upload fails that item (a multipart upload is aborted); a download part without a valid record (`darkroom-etag-v1 ETAG`) is never deleted without `--force`, and a part of the tool's own of an earlier version is replaced with a warning; one run at a time writes a file; `rm -r` skips (never deletes) keys with `.`, `..` or empty segments or a leading `/`, deletes the rest, lists the skipped ones and exits 1 when anything was skipped. No new module: multipart is implemented on `service/s3`. Not part of this: mv, sync, presign, copy between
locations, resuming an upload after the program has ended.

`darkroom storage`: object storage from the command line (stage 1: listing, the key and the setup of other tools).

`darkroom storage ls [LOCATION[:PATH]] [-l] [--human] [--json]`: with no argument, your object-storage locations
(access, `s3://` address, whether a personal key reaches them, used space and quota); with a location and optional
path, the objects and folders in it from an S3 listing (delimiter `/`, every page, always under your own prefix, so
the root of the `users` bucket is never listed). Paths with `..`, empty segments, backslashes or control characters
are refused before anything is sent. The first dependency besides cobra: the AWS SDK for Go v2 (`service/s3`, Apache-2.0),
used with path-style addressing and the default request and response checksums switched off, as the storage system needs;
buckets are never listed, they come from DarkRoom's locations.

`darkroom storage login | status | rotate | revoke`: this computer's own S3 key (one of your two slots; never made by
itself). `login` and `rotate` ask on the terminal for a code from your authenticator app when your sign-in is older than
DarkRoom's step-up window (never from the command line; a run without a terminal and an agent sign-in are refused); they
find and offer to replace a key whose answer was lost, and `rotate` says so before calling DarkRoom when no slot is free.
The secret is kept only in `~/.darkroom/v2/object-key.json` (0600, bound to the server and user), and is never printed
or logged; `revoke` deletes it only after DarkRoom confirms.

`darkroom storage setup env | aws | rclone`: configure other tools. `setup aws` writes the profile `darkroom` with
`credential_process` (the key stays in darkroom's file), path-style addressing and both checksum settings; it never
replaces an existing profile without `--force` and honours `AWS_CONFIG_FILE`. `--write-credentials` (AWS) and `setup rclone`
write a copy of the secret that `rotate` keeps current and `revoke` removes. `setup env` prints export lines. The hidden
`storage credential-process` prints the JSON the AWS tools read, and nothing else, without any network call.

The 0.x `darkroom storage` commands (API v1) are deprecated in favour of these. `darkroom logout` now says when a storage
key is left active. Not yet: `storage cp` and `storage rm`.

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

