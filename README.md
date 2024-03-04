# remote_power_control

Shell script to remotely power on/off computers

## Usage

`remote_power_control <type> <...>`

## Supported types

- `homeassistant`
- `pikvm`
- `wakeonlan` (only supports 'on' action)

## Supported actions

- `on`
- `off`

### Post power on sleep

By default, after a power on action, there is a sleep for 4 minutes to wait for the computer to power on. This can be configured by the `REMOTE_POWER_CONTROL_POST_POWER_ON_SLEEP_IN_SECONDS` environment variable.

## Type usages

### homeassistant

`remote_power_control homeassistant <action> <switch_entity_id> [<homeassistant_base_url>] [<homeassistant_access_token>]`

### pikvm
`remote_power_control pikvm <action> [<pikvm_url>] [<pikvm_username>] [<pikvm_password>]`

### wakeonlan
`remote_power_control wakeonlan <mac_address>`

# Environment variables

ENV variables (optionally set in /`etc/remote_power_control.env`) can also be set instead of optional parameters

- `REMOTE_POWER_CONTROL_PIKVM_URL`
- `REMOTE_POWER_CONTROL_PIKVM_USERNAME`
- `REMOTE_POWER_CONTROL_PIKVM_PASSWORD`
- `REMOTE_POWER_CONTROL_HOMEASSISTANT_URL`
- `REMOTE_POWER_CONTROL_HOMEASSISTANT_ACCESS_TOKEN`
- `REMOTE_POWER_CONTROL_POST_POWER_ON_SLEEP_IN_SECONDS`
