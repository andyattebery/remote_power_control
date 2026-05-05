# remote_power_control — Specification

This document defines the canonical behavior of `remote_power_control` (client)
and `remote/power_command_target` (server-side wrapper). Anything that drifts
from this spec is a bug.

## Overview

`remote_power_control` is a POSIX-shell script that powers a remote host
on/off, queries its state, or triggers a graceful reboot/shutdown over SSH.
Each method (Home Assistant smart switch, PiKVM ATX, Wake-on-LAN, SSH) is a
"type". The script handles pre-checks, retries, timeouts, and post-action
waits so callers chaining commands get a usable host when it returns.

`remote/power_command_target` is the server-side wrapper that accepts
restricted `reboot` and `shutdown` commands when invoked as an SSH
`ForceCommand`.

## CLI

```
remote_power_control [<flags>] <host> <type> [<action>] [<type-specific args>...]
```

Positional args:
- `<host>` — hostname or IP, used for the pre-check and the post-action wait.
- `<type>` — `homeassistant`, `pikvm`, `wakeonlan`, or `ssh`.
- `<action>` — type-dependent (see below). For `wakeonlan`, action defaults to
  `on` and may be omitted; pass `status` explicitly to query state.

Global flags (parsed before positional args):

| Flag                  | Effect |
|-----------------------|--------|
| `-n`, `--dry-run`     | Print sanitized commands instead of executing curl/ssh/wakeonlan. Pre-checks still run. |
| `-f`, `--force`       | Skip the "already on" pre-check. |
| `-v`, `--verbose`     | Enable shell tracing (`set -x`). Tokens may leak — debug only. |
| `-q`, `--quiet`       | Suppress informational stdout. Errors still go to stderr. |
| `-h`, `--help`        | Print usage and exit 0. |

## Types and arguments

| Type            | Positional args after `<host> <type>`                                                       |
|-----------------|---------------------------------------------------------------------------------------------|
| `homeassistant` | `<action> <switch_entity_id> [<homeassistant_url>] [<homeassistant_access_token>]`          |
| `pikvm`         | `<action> [<pikvm_url>] [<pikvm_username>] [<pikvm_password>]`                              |
| `wakeonlan`     | `<mac_address>`  *(action implicit `on`)* — or `status`                                     |
| `ssh`           | `<action> [<ssh_username>] [<ssh_private_key_path>]`                                        |

## Actions per type

| Type            | Supported actions                |
|-----------------|----------------------------------|
| `homeassistant` | `on`, `off`, `status`            |
| `pikvm`         | `on`, `off`, `status`            |
| `wakeonlan`     | `on` (implicit), `status`        |
| `ssh`           | `reboot`, `shutdown`, `status`   |

`status` is type-agnostic in implementation: it pings `<host>` and prints `on`
or `off`. Type-specific APIs (HA `/api/states/<entity>`, PiKVM `/api/atx`) are
*not* queried — see "Caveats" below.

## Environment variables

Optionally set in `$ENV_FILE_PATH` (default `/etc/remote_power_control/env`,
overridable via env). The file is sourced if it exists; the script warns to
stderr if it is world-readable.

| Variable | Used for |
|---|---|
| `REMOTE_POWER_CONTROL_HOMEASSISTANT_URL` | Default HA URL, including scheme and port (e.g. `http://ha.local:8123`) |
| `REMOTE_POWER_CONTROL_HOMEASSISTANT_ACCESS_TOKEN` | Default HA bearer token |
| `REMOTE_POWER_CONTROL_PIKVM_URL` | Default PiKVM URL |
| `REMOTE_POWER_CONTROL_PIKVM_USERNAME` | Default PiKVM user |
| `REMOTE_POWER_CONTROL_PIKVM_PASSWORD` | Default PiKVM password |
| `REMOTE_POWER_CONTROL_SSH_USERNAME` | Default SSH user for `ssh` type |
| `REMOTE_POWER_CONTROL_SSH_PRIVATE_KEY_PATH` | Default SSH key for `ssh` type |
| `REMOTE_POWER_CONTROL_POST_POWER_ON_MAX_WAIT_IN_SECONDS` | Max wait after a power-on. Default `180`. |
| `REMOTE_POWER_CONTROL_POST_REBOOT_MAX_WAIT_IN_SECONDS`   | Max wait after `ssh reboot`. Default `180`. |

**Backward compatibility.** The older names
`REMOTE_POWER_CONTROL_POST_POWER_ON_SLEEP_IN_SECONDS` and
`REMOTE_POWER_CONTROL_POST_REBOOT_SLEEP_IN_SECONDS` are honored as fallbacks
when the new `_MAX_WAIT_IN_SECONDS` form is unset. Existing ansible-role env
files keep working without change. The semantic has changed — these now control
*max wait until ready*, not *fixed sleep duration*.

## Pre-action check

Before any `on` action, the script calls `host_is_responding "$host"` (single
ping, 2 s timeout). If the host responds, the script exits 0 without issuing
the power command. `--force` bypasses this. The pre-check does not run for
`off`, `reboot`, `shutdown`, or `status`.

## Post-action wait

After a successful action (and only when not `--dry-run`), the script polls
`host_is_responding` until success or a timeout, then exits 0 (success) or 3
(timeout).

| Action context             | Behavior |
|----------------------------|----------|
| `homeassistant on`, `pikvm on`, `wakeonlan` | Poll until responding, max `POST_POWER_ON_MAX_WAIT_IN_SECONDS`. |
| `ssh reboot`               | Wait up to 60 s for host to *stop* responding, then poll until it responds again, max `POST_REBOOT_MAX_WAIT_IN_SECONDS`. If the host never drops, exit 3. |
| All `off` / `shutdown`     | No wait. |
| `status`                   | No wait (the check is the action). |

**Timeout is a failure.** If the wait exceeds the budget, the script exits 3.
The power command itself may have succeeded, but from the script's contract
perspective the host is unusable. Callers that don't care can `|| true` the
invocation.

## Exit codes

| Code | Meaning |
|------|---------|
| `0`  | Success |
| `1`  | Generic error (fallback) |
| `2`  | Usage error: missing args, unknown type, invalid action |
| `3`  | Host unreachable / `wait_for_host_ready` timeout |
| `4`  | Authentication failure (HTTP 401/403, ssh auth) |
| `5`  | API error (HTTP 4xx other than auth, malformed response) |
| `6`  | Transient error after retries (HTTP 5xx, network) |

curl exits 22 on HTTP errors with `--fail-with-body`. The script propagates
that as 1 unless future work maps the HTTP status to 4/5/6 specifically. (See
"Future work" below.)

## Audit log

On every invocation, a single line is appended on EXIT:

```
<UTC ISO timestamp> host=<host> type=<type> action=<action> exit=<rc> duration_s=<n>
```

Written to `/var/log/remote_power_control.log` if writable; otherwise to
syslog via `logger -t remote_power_control`. Log write failure does not fail
the script.

## Strict mode and quoting

The script enables `set -u` after env-file sourcing and flag parsing.
Function-internal variables use a leading underscore (e.g. `_host`, `_action`)
to avoid shadowing main-script globals. Maybe-unset reads use `${VAR-}`.

## Dependencies

- `curl` ≥ 7.76 (required for `--fail-with-body`)
- `ssh`
- `wakeonlan`
- `ping` (Linux iputils or BSD/macOS variant — `host_is_responding` branches
  on `uname` so the client works from either Linux or macOS control nodes)
- `sudo`, `logger`, `stat`, `date` (standard)

## Server-side wrapper: `remote/power_command_target`

Deployed on hosts that accept `ssh reboot` / `ssh shutdown` from
`remote_power_control`. See `remote/README.md` for setup. Supports exactly:

| `SSH_ORIGINAL_COMMAND` | Effect                        |
|------------------------|-------------------------------|
| `reboot`               | `sudo /usr/sbin/reboot`       |
| `shutdown`             | `sudo /usr/sbin/shutdown now` |

Anything else exits 2.

## Caveats

- **`status` is host-reachability, not plug-state.** `status` pings the host;
  it does not query the HA smart-plug or PiKVM ATX power LED.
  - PiKVM ATX `leds.power` is unreliable: many installs only wire the
    momentary power button, not the power-LED leads.
  - HA reports the smart-plug state, which can differ from the host state
    (plug energized but host hasn't booted; host on a different outlet).
  - For homelab use, "is the host reachable" matches what users actually
    mean. Users who genuinely need plug/PDU state can `curl` the HA or
    PiKVM API directly.
- **Ping responding ≠ sshd ready.** A host can answer ICMP while sshd is
  still starting. `wait_for_host_ready` returns when ping succeeds; users
  needing strict ssh-ready can chain `until ssh ... true; do sleep 5; done`.
- **Verbose tracing leaks tokens.** `-v` enables `set -x` with no
  redaction. Debug-grade only — do not enable on shared screens or in CI
  logs.
- **`--dry-run` skips network calls and waits.** Pre-check ping still runs
  (read-only). Audit log entry still records the (no-op) invocation.
- **`--dry-run` secret redaction is regex-based** (`Bearer [^ ]*`,
  `X-KVMD-Passwd: [^ ]*`). Passwords containing spaces would be only
  partially redacted. Use strong tokens without spaces.

## Future work

- Map curl HTTP status to specific exit codes (4 for 401/403, 5 for other
  4xx, 6 for 5xx after retries).
- Optional deprecation hint when the old `_SLEEP_IN_SECONDS` env vars are
  read.
- Soft-off (try `ssh shutdown` before `pikvm off` / `homeassistant off`).
- Token redaction in `-v` tracing output.
