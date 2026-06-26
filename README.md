# hermes-ssh-attach-skill

A skill for the [Hermes agent](https://hermes-agent.nousresearch.com/) that attaches your agent to a remote machine over SSH - end to end. Formulate your intent in natural language mentioning the host (or alias from the ssh config), user, and working directory, and the skill handles the rest: key discovery, password prompting, hermes profile creation and configuration.

## Which problem does it solve

To work with the local Hermes agent(s) on the remote machine, you manually create a profile, set SSH backend (cli), pick the right key, copy model config, set tools remote propagation etc.

With this skill: You say "connect to my_machine.com using myuser" and it does everything e2e - discovers keys, tests them, creates/reuses the profile, configures SSH + model + memory, runs doctor, gives you the launch command

## What it does

- **Parses any SSH form** - `ssh user@host`, `ssh -p 2222 user@host`, `ssh -i key root@ip`, or a bare `~/.ssh/config` alias (`ssh to machine1`, resolved via `ssh -G`)
- **Discovers and tests SSH keys** - tries every key (`id_ed25519`, `id_rsa`, `*.pem`, …) with `BatchMode` until one works. No guessing
- **Falls back to password auth** via `sshpass` - installs it if missing, prompts for the password, stores it in the profile (plaintext, see Security notes)
- **Creates an isolated profile per machine** - separate sessions, memory, skills, and cron jobs
- **Inherits your model/provider config** so the new profile isn't born unconfigured
- **Plants a `MEMORY.md`** reminding the agent that file tools are local-only and remote work goes through `terminal`
- **Launches** - in a desktop mode or, inside a CLI session, hands you the exact command to run in a new terminal

## Installation

Clone into your Hermes skills directory like so:

```bash
git clone https://github.com/gehirndienst/hermes-ssh-attach-skill.git ~/.hermes/skills/software-development/ssh-attach
```

Hermes discovers the skill on next session start

## Usage

Load it and provide a connection:

```
/skill ssh-attach
ssh deploy@server.example.com
```

Or trigger it conversationally:

- `connect to ssh user@host`
- `ssh to machine1` (a `~/.ssh/config` alias)
- `attach to 10.0.0.1 and work in /srv/app`

### Other commands the skill understands

| Say | Does |
|-----|------|
| `switch to PROFILE` | `hermes profile use PROFILE` |
| `detach` / `go local` | resets backend to local |
| `list profiles` | `hermes profile list` |
| `remove profile PROFILE` | tears it down (irreversible - confirms first) |

## How it works

The SSH backend routes **only `terminal` commands** to the remote host. File tools (`read_file`, `write_file`, `search_files`, `patch`), code execution, and browser tools all run **locally**. The skill plants a [`MEMORY.md`](references/memory-md-template.md) in the profile so the agent always reaches for `terminal` (`cat`, `echo >`, `grep`/`rg`, `sed`) when touching remote files.

### SSH config reference

| Config key | Env var | Description |
|-----------|---------|-------------|
| `terminal.backend` | `TERMINAL_ENV` | Set to `ssh` |
| `terminal.ssh_host` | `TERMINAL_SSH_HOST` | Remote hostname or IP |
| `terminal.ssh_user` | `TERMINAL_SSH_USER` | SSH username |
| `terminal.ssh_port` | `TERMINAL_SSH_PORT` | SSH port (default 22) |
| `terminal.ssh_key` | `TERMINAL_SSH_KEY` | Path to private key |
| `terminal.ssh_password` | `TERMINAL_SSH_PASSWORD` | Password (requires `sshpass`) |

## Security notes

- `ssh_password` is stored **in plaintext** in the profile's `config.yaml`. Prefer SSH keys whenever possible
- Setting it via `config set` also lands in your shell history - the skill uses a silent `read -rs` instead
- `profile remove` deletes the stored profile data along with everything else

## License

MIT

# Author

[Nikita Smirnov](https://github.com/gehirndienst)