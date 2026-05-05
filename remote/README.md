# remote/

Server-side companion to `remote_power_control`.

`power_command_target` is a tiny SSH `ForceCommand` wrapper deployed on a host
that should accept restricted `reboot` and `shutdown` commands over SSH —
typically from a control node running `remote_power_control <host> ssh reboot`
(or from any other client with a key authorized to invoke it).

## Deployment

On the target host:

1. Copy `power_command_target` to `/usr/local/bin/power_command_target` (mode 0755).
2. Create a dedicated user (e.g. `remote-power-control`) that the controller
   will SSH in as.
3. Add the controller's public key to that user's `~/.ssh/authorized_keys`,
   restricted to the wrapper:

   ```
   command="/usr/local/bin/power_command_target",no-pty,no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 AAAA... controller@host
   ```

4. Allow the user to run `reboot` and `shutdown` without a password via sudoers
   (drop in `/etc/sudoers.d/remote-power-control`):

   ```
   remote-power-control ALL=(root) NOPASSWD: /usr/sbin/reboot, /usr/sbin/shutdown
   ```

## Supported `SSH_ORIGINAL_COMMAND` values

| Command    | Effect                       |
|------------|------------------------------|
| `reboot`   | `sudo /usr/sbin/reboot`      |
| `shutdown` | `sudo /usr/sbin/shutdown now` |

Anything else exits 2 with `"<cmd> not supported"` on stderr — the calling
client (e.g. `remote_power_control`) propagates the non-zero exit.

## Why absolute paths

The script runs under a restricted SSH session with a minimal `PATH`. Calling
`/usr/sbin/reboot` and `/usr/sbin/shutdown` directly avoids depending on the
restricted user's `PATH` setup.
