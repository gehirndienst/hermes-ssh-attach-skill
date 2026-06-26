# SSH Password Auth Implementation

Source code changes made to Hermes Agent to support `ssh_password` config option. All changes are in the Hermes source at `~/.hermes/hermes-agent/`.

## Files Modified

### `tools/environments/ssh.py` (core SSH backend)
- Added `_ensure_sshpass_available()` — checks `sshpass` binary exists
- Added `password` parameter to `SSHEnvironment.__init__()` + `self._uses_password_auth` flag
- Modified `_build_ssh_command()` — only adds `BatchMode=yes` when NOT using password auth
- Added `_build_sshpass_prefix()` — returns `["sshpass", "-e"]` when password is set
- Added `_ssh_env()` — sets `SSHPASS` env var for sshpass
- Modified `_establish_connection()` — prepends sshpass prefix and passes env

### `hermes_cli/config.py`
- Added `"ssh_password": "TERMINAL_SSH_PASSWORD"` to `TERMINAL_CONFIG_ENV_MAP`

### `tools/terminal_tool.py`
- Added `"ssh_password": os.getenv("TERMINAL_SSH_PASSWORD", "")` to SSH config dict
- Pass `password=ssh_config.get("password", "")` to `SSHEnvironment()`

### `tools/code_execution_tool.py`
- Added `"password": config.get("ssh_password", "")` to `ssh_config` dict

### `tools/file_tools.py`
- Added `"password": config.get("ssh_password", "")` to `ssh_config` dict

### `hermes_cli/status.py`
- Shows masked password status: `***` or `(not set — key auth)`

### `hermes_cli/doctor.py`
- Detects password auth mode, checks for `sshpass` installation
- Tests password auth using `SSHPASS` env var + `sshpass -e`

## How It Works

1. When `ssh_password` is set in config/env, `_uses_password_auth = True`
2. `BatchMode=yes` is skipped (SSH needs to prompt for password)
3. `sshpass -e` is prepended to the ssh command
4. `SSHPASS` env var carries the password (more secure than `-p` flag)
5. sshpass automatically provides the password when SSH prompts

## Verification

All modified files compile cleanly:
```bash
python3 -m py_compile tools/environments/ssh.py
python3 -m py_compile hermes_cli/config.py
python3 -m py_compile hermes_cli/status.py
python3 -m py_compile hermes_cli/doctor.py
python3 -m py_compile tools/terminal_tool.py
python3 -m py_compile tools/code_execution_tool.py
python3 -m py_compile tools/file_tools.py
```

## Diff Summary

```
 hermes_cli/config.py         |  1 +
 hermes_cli/doctor.py         | 21 +++++++++++++++++++--
 hermes_cli/status.py         |  2 ++
 tools/code_execution_tool.py |  1 +
 tools/environments/ssh.py    | 43 +++++++++++++++++++++++++++++++++++++++++--
 tools/file_tools.py          |  1 +
 tools/terminal_tool.py       |  2 ++
 7 files changed, 67 insertions(+), 4 deletions(-)
```
