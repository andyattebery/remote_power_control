# remote_power_control

POSIX-shell script to power remote hosts on/off via Home Assistant smart
switches, PiKVM ATX, Wake-on-LAN, or SSH.

See [`SPEC.md`](SPEC.md) for the full behavioral specification and
[`remote/`](remote/) for the server-side wrapper used by the `ssh` type.

## Usage

```
remote_power_control [<flags>] <host> <type> [<action>] [<type-specific args>...]
```

### Types

| Type            | Example |
|-----------------|---------|
| `homeassistant` | `remote_power_control my-nas homeassistant on switch.nas_plug` |
| `pikvm`         | `remote_power_control my-server pikvm on` *(or with a PiKVM Switch port: `... pikvm on 0`)* |
| `wakeonlan`     | `remote_power_control my-pc wakeonlan AA:BB:CC:DD:EE:FF` |
| `ssh`           | `remote_power_control my-server ssh reboot` |

### Actions

- `on`, `off` — `homeassistant`, `pikvm`
- `on` (implicit) — `wakeonlan`
- `reboot`, `shutdown` — `ssh`
- `status` — all types (prints `on` or `off` based on host reachability)

### Flags

- `-n`, `--dry-run` — print what would run, don't execute
- `-f`, `--force` — skip the "already on" pre-check
- `-v`, `--verbose` — enable shell tracing (debug only — leaks tokens)
- `-q`, `--quiet` — suppress informational output
- `-h`, `--help` — print help

## Behavior

Before any `on` action, the script pings the host. If it's already responding,
the script exits 0 without issuing the power command. After a successful
`on`/`reboot`/`wakeonlan`, it polls until the host is reachable again, up to a
configurable max wait (default 180 s). If the host doesn't respond in time, it
exits 3.

Every invocation appends a line to `/var/log/remote_power_control.log` (or
syslog as fallback) with the host, type, action, exit code, and duration.

## Configuration

URLs and credentials are supplied via environment variables — set them
inline (`FOO=bar remote_power_control ...`) or in
`/etc/remote_power_control/env` (path overridable via `ENV_FILE_PATH`),
which is sourced if it exists. The file should be `chmod 600` since it
holds API tokens — the script warns if it's world-readable.

| Variable | Purpose |
|---|---|
| `REMOTE_POWER_CONTROL_HOMEASSISTANT_URL` | HA base URL with scheme and port (e.g. `http://ha.local:8123`) |
| `REMOTE_POWER_CONTROL_HOMEASSISTANT_ACCESS_TOKEN` | HA bearer token |
| `REMOTE_POWER_CONTROL_PIKVM_URL` | PiKVM URL |
| `REMOTE_POWER_CONTROL_PIKVM_USERNAME` | PiKVM user |
| `REMOTE_POWER_CONTROL_PIKVM_PASSWORD` | PiKVM password |
| `REMOTE_POWER_CONTROL_SSH_USERNAME` | SSH user for `ssh` type |
| `REMOTE_POWER_CONTROL_SSH_PRIVATE_KEY_PATH` | SSH key for `ssh` type |
| `REMOTE_POWER_CONTROL_POST_POWER_ON_MAX_WAIT_IN_SECONDS` | Max wait after power-on (default 180) |
| `REMOTE_POWER_CONTROL_POST_REBOOT_MAX_WAIT_IN_SECONDS` | Max wait after `ssh reboot` (default 180) |

The older `*_SLEEP_IN_SECONDS` names are still honored as fallbacks for
existing deployments.

## Server-side setup

The `ssh` type expects the target host to run [`remote/power_command_target`](remote/power_command_target)
as a restricted SSH `ForceCommand`. See [`remote/README.md`](remote/README.md)
for setup.

## Dependencies

- `curl` ≥ 7.76, `ssh`, `wakeonlan`, `ping` — required
- `sudo`, `logger`, `stat`, `date` — standard

## Resources

- [Home Assistant REST API](https://developers.home-assistant.io/docs/api/rest/)
- [PiKVM API](https://docs.pikvm.org/api/)
