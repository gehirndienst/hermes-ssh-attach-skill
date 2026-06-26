---
name: ssh-attach
description: "Attach Hermes to a remote machine over SSH, end to end. Trigger by giving an SSH command ('ssh user@host', 'ssh -p 2222 user@host', 'ssh -i key root@ip'), a bare host or ~/.ssh/config alias ('ssh to machine1'), or natural language ('connect to X', 'attach to X', 'work on X'). The skill discovers and tests SSH keys, falls back to password auth (installs sshpass if needed), creates an isolated profile, inherits your model/provider config, plants a remote-aware MEMORY.md, and hands you the launch command. Usage: /skill ssh-attach then name the host"
version: 1.0.0
---

# SSH Attach

Attach Hermes to a remote machine via SSH. The skill handles the entire workflow end-to-end.

## Triggers

- User provides an SSH connection string: `ssh user@host`, `ssh -p 2222 user@host`, etc.
- User names a bare host or `~/.ssh/config` alias: "ssh to machine1", "connect to machine1" (resolved via `ssh -G`)
- User says "connect to X", "attach to X", "work on X"
- User loads the skill with `/skill ssh-attach`

## Authentication Methods

### SSH Keys (preferred)
The skill discovers all available SSH keys and tests each one until one works:

1. List keys: `ls -la ~/.ssh/id_* ~/.ssh/*.pem 2>/dev/null`
2. For each key, test with BatchMode (no passphrase prompt):
   ```bash
   ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new -o IdentitiesOnly=yes -i /PATH/TO/KEY -p PORT USER@HOST echo SSH_OK
   ```
3. Try in priority order: `id_ed25519`, `id_ecdsa`, `id_rsa`, `id_dsa`, then `*.pem`
4. Also check `~/.ssh/config` for host-specific key assignments
5. Report which keys succeeded/failed to the user
6. Use the first working key

### Password Auth (requires sshpass)
If keys aren't set up, password auth works via `sshpass`. The skill:
- Detects when password is needed
- Installs `sshpass` if missing (`brew install sshpass` / `apt install sshpass`)
- Prompts for the password using `clarify`
- Configures `terminal.ssh_password` in the profile

## Workflow — Execute Sequentially

### 1. Parse the SSH Command

Extract `user`, `host`, `port`, `key`, and **working directory** from any common format:

```
ssh user@host
ssh -p 2222 deploy@server.example.com
ssh -i ~/.ssh/custom_key root@10.0.0.1
connect me to ssh user@host and work in /var/www/my-project
```

If the user just says "connect to X" without a full SSH command, use `clarify` to ask:
```
What's the SSH command for X? (e.g., ssh user@X, ssh -p 2222 user@X)
Do you want to work in a specific folder on the remote machine? (default: ~)
```

#### Resolve `~/.ssh/config` aliases first

When the input is a **bare token** with no `user@` and no `ssh` flags — e.g. "ssh to machine1", "connect to machine1" — it may be a `Host` alias in `~/.ssh/config`. Do NOT treat the token as a literal hostname. Expand it with `ssh -G`, which resolves all `Host`/`Match` rules (HostName, User, Port, IdentityFile, ProxyJump, etc.):

```bash
ssh -G machine1 2>/dev/null | grep -iE '^(hostname|user|port|identityfile) '
```

Interpret the output:
- **`hostname` differs from the token**, or `user`/`port`/`identityfile` are set by config → it's a real alias. Use the **resolved** values for everything downstream:
  - `terminal.ssh_host` = resolved `hostname`
  - `terminal.ssh_user` = resolved `user`
  - `terminal.ssh_port` = resolved `port`
  - `terminal.ssh_key` = resolved `identityfile` (first one), if present
  - **Skip Step 2's key-discovery loop** — config already names the key. Just confirm it works once with a BatchMode test.
- **`hostname` equals the token** and nothing else is configured → not an alias. Fall through to normal `user@host` parsing below.

Note: `ssh -G` always prints *some* hostname/port/user (defaults), so compare against the token and look for non-default values rather than mere presence. `ProxyJump`/`ProxyCommand` in the output means the host is reached through a bastion — Hermes connects via the same `~/.ssh/config`, so leave it to SSH; just flag it to the user.

#### Profile name rule

Derive `PROFILE_NAME` from the host, deterministically (so re-running the skill on the same host reuses the same profile):

- Lowercase the host.
- Drop a trailing common domain suffix if it leaves something readable (`server.example.com` → `server`); otherwise keep the full label.
- Replace every non-alphanumeric char with `-`, collapse repeats, trim leading/trailing `-`.
- Examples: `deploy@server.example.com` → `server`; `root@10.0.0.1` → `10-0-0-1`; `user@db.internal.corp` → `db`.

If a different host already owns that name, append the user or a short disambiguator (`server-deploy`, `db-2`).

If the input was an `~/.ssh/config` alias, name the profile after the **alias** (`machine1` → `machine1`), not the resolved hostname — that's what the user types next time, and it keeps re-runs idempotent.

#### Confirm before acting

Echo the parsed values back to the user before configuring anything — one wrong char points the profile at the wrong box:

```
Connecting as USER to HOST:PORT (key/password), workdir DIR, profile PROFILE_NAME. Correct?
```

Skip the confirmation only if the user gave an unambiguous full SSH command and a clear go-ahead.

### 2. Discover and Test Available SSH Keys

**Skip this whole step if Step 1 resolved an `~/.ssh/config` alias** — the config already specifies the key (or none, meaning agent/default). Just run one BatchMode test against the resolved values to confirm, then go to Step 3 only if it fails.

Otherwise: never assume which key to use. Always find and try every available key until one works.

#### 2a. List available private keys

```bash
ls -la ~/.ssh/id_* 2>/dev/null | grep -- '^[^-]' | grep -v '\.pub$'
```

Also check `~/.ssh/config` for host-specific key settings:

```bash
cat ~/.ssh/config 2>/dev/null | head -30
```

#### 2b. Try each key individually (BatchMode = no passphrase prompt)

For every private key found (`id_ed25519`, `id_ecdsa`, `id_rsa`, `id_dsa`, `*.pem`), test with:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new \
    -o IdentitiesOnly=yes -i /PATH/TO/KEY -p PORT USER@HOST echo SSH_OK
```

Try keys in priority order:
1. Keys explicitly configured for the host in `~/.ssh/config`
2. `id_ed25519`
3. `id_ecdsa`
4. `id_rsa`
5. `id_dsa`
6. Any `*.pem` files

Report which keys succeeded/failed. Use the first working key.

#### 2c. Fall back to default agent (no `-i` flag)

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new -p PORT USER@HOST echo SSH_OK
```

This lets the SSH agent pick the key automatically.

#### 2d. Pick the working key

Configure the profile with whichever key returned `SSH_OK`. If no key worked and no agent fallback worked, proceed to Step 3 (password auth).

### 3. Handle Password Auth

#### 3a. Check for sshpass

```bash
which sshpass
```

**If not installed:** Use `clarify`:
```
This machine requires password authentication. I need to install `sshpass` — OK?
```
If yes, run:
- macOS: `brew install sshpass`
- Ubuntu/Debian: `sudo apt install sshpass`
- CentOS/RHEL: `sudo yum install sshpass`

#### 3b. Prompt for password

Use `clarify` (open-ended):
```
Connection to USER@HOST requires a password. What is it?
```

#### 3c. Test password auth

```bash
SSHPASS="PASSWORD" sshpass -e ssh -o StrictHostKeyChecking=accept-new -p PORT USER@HOST echo SSH_OK
```

**If fails:** Tell the user the password was rejected, ask again.
**If succeeds:** Continue to Step 4.

### 4. Check for existing profile

Generate profile name from host (same rules as above). Then check if it already exists:

```bash
hermes profile list
```

**If the profile exists:** Skip to Step 6 (configure/verify). Do NOT ask the user what to do — just reuse it. Only ask if the existing profile points to a *different* host than what the user requested.

**If it does not exist:** Proceed to Step 5 (create it).

### 5. Create Profile (only if missing)

```bash
hermes profile create PROFILE_NAME
```

**Note:** `--clone-all default` does NOT reliably copy model/provider config — always do Step 8 (inherit model/provider config) manually regardless.

### 6. Configure the Profile

```bash
hermes --profile PROFILE_NAME config set terminal.backend ssh
hermes --profile PROFILE_NAME config set terminal.ssh_host HOST
hermes --profile PROFILE_NAME config set terminal.ssh_user USER
hermes --profile PROFILE_NAME config set terminal.ssh_port PORT

# Set remote working directory. Use `workdir` — it controls the shell's
# starting dir in newer Hermes. `terminal.cwd` is legacy; set it too on older versions.
hermes --profile PROFILE_NAME config set workdir /path/to/remote/project
```

**If the user didn't specify a directory, default to the remote home — but QUOTE the tilde:** `hermes --profile PROFILE_NAME config set workdir '~'`. Unquoted `~` is expanded by your *local* shell and writes your local home path into the remote profile. See the tilde pitfall below.

If a key path was specified (`-i`):
```bash
hermes --profile PROFILE_NAME config set terminal.ssh_key KEY_PATH
```

If password auth is being used:
```bash
hermes --profile PROFILE_NAME config set terminal.ssh_password "PASSWORD"
```

**Security:** `ssh_password` is stored in plaintext in the profile's `config.yaml`. Prefer SSH keys whenever possible. Only use password auth when keys aren't an option, and tell the user the password lands on disk unencrypted.

The `config set ... "PASSWORD"` command above also leaks into shell history (`~/.zsh_history`). Avoid that — prefix the command with a leading space (suppresses history when `setopt HIST_IGNORE_SPACE` is on), or read the password into a var first so the literal never appears as a command argument:

```bash
 read -rs PW   # leading space + silent read; nothing echoed or logged
 hermes --profile PROFILE_NAME config set terminal.ssh_password "$PW"
 unset PW
```

### 7. Enforce remote file operations via MEMORY.md

Hermes automatically injects `memories/MEMORY.md` into the agent's context. Create it to permanently remind the agent to use terminal commands for remote files (since `read_file`, `write_file`, etc. are local-only):

```bash
cat > ~/.hermes/profiles/PROFILE_NAME/memories/MEMORY.md << 'EOF'
# SSH Profile Constraints
- terminal.backend is `ssh`. File tools (`read_file`, `write_file`, `search_files`, `patch`) run LOCALLY and CANNOT access the remote filesystem.
- ALWAYS use `terminal` commands for file operations on the remote host:
  - Read: `cat file`, `less file`, `head`, `tail`
  - Write: `echo "content" > file` or `cat > file << 'EOF' ... EOF`
  - Search: `grep -r "pattern" .` or `rg "pattern"`
  - Edit: `sed -i 's/old/new/g' file` or `awk`
  - Move/Copy/Delete: `mv`, `cp`, `rm`, `mkdir`
- Use `terminal` to run external tools, build systems, scripts, and code execution on the remote machine.
- If the user asks to read/write/edit a file, assume they mean on the remote host and use `terminal` commands.
EOF
```

### 8. Inherit Model and Provider Config

**Critical step — new profiles ship with `model: ''` and `providers: {}`**, which causes "Hermes isn't configured yet" on launch. Read the default profile's `~/.hermes/config.yaml` to extract the `model` and `providers` blocks, then patch the new profile's `~/.hermes/profiles/PROFILE_NAME/config.yaml` to replace:

```yaml
model: ''
providers: {}
```

with the actual model/providers block from the default profile. Example for local LLM:

```yaml
model:
  base_url: http://172.17.136.91:8008/v1
  default: Local-Qwen3.6
  provider: custom
providers:
  zelda:
    api: http://172.17.136.91:8008/v1
    name: zelda
    default_model: Local-Qwen3.6
```

For API-key providers, also copy `~/.hermes/.env` to `~/.hermes/profiles/PROFILE_NAME/.env`.

**Run `hermes --profile PROFILE_NAME doctor --fix`** after to migrate config version.

### 9. Create Shell Alias

```bash
hermes profile alias PROFILE_NAME
```

### 10. Verify with Doctor

```bash
hermes --profile PROFILE_NAME doctor
```

### 11. Launch the profile immediately

**IMPORTANT — CLI limitation:** When running inside a CLI `hermes` session, you CANNOT launch a new interactive REPL that the user's keyboard can control. `hermes --profile PROFILE_NAME` spawns a child process, and child processes cannot "hand off" to the parent's TTY. Running it in the background leaves the user with a session they cannot interact with.

**Detect your context** — check the interface before deciding:

```bash
hermes config get display.interface   # 'cli' → CLI session; 'gateway'/'desktop' → app
```

If the key is unset or the command fails, assume **CLI** (the safe default — never auto-launch).

| Context | Action |
|---------|--------|
| **CLI session** (display.interface = cli or `hermes` command-line) | Give the user the exact command to run in a new terminal tab/window. Provide both the full form and the alias (if created). Do NOT attempt to launch it yourself. |
| **Desktop/Gateway app** | Tell the user to switch profiles in the UI. Optionally attempt `hermes --profile PROFILE_NAME` as a background launch. |

**CLI launch instructions (copy-paste ready):**

```
Open a new terminal tab and run:

  hermes --profile PROFILE_NAME

Or use the alias:

  PROFILE_NAME
```

After giving the command, offer to close the current session if the user wants:
```
Run 'exit' or press Ctrl+C to close this session and open the new one.
```

## Pitfalls (learned from real sessions)

### New profiles ship with empty model config
`hermes profile create` creates a profile with `model: ''` and `providers: {}`. The user's local LLM config (base_url, provider, etc.) is in `~/.hermes/config.yaml`, NOT in the new profile. You MUST manually copy the `model` and `providers` blocks — `--clone-all` does not do this reliably.

### ~/.hermes/.env may be a template
For local LLM users, `~/.hermes/.env` is often just a template with masked keys (`***`). The actual model config lives in `~/.hermes/config.yaml` under `model` and `providers`. Do NOT rely on .env for local LLM config.

### SSH key discovery — always try ALL keys
Users may have multiple keys (`id_ed25519`, `id_rsa`, etc.) and only some work for a given host. Never assume which key to use. Test each one with `-o IdentitiesOnly=yes` to isolate which key actually works. Report results.

### BatchMode=yes tests are definitive
A key that works with `BatchMode=yes` works without an SSH agent. A key that fails may have a passphrase or simply not be authorized on the server. The SSH agent (`ssh-add -l`) is often empty — don't rely on it.

### workdir vs terminal.cwd
Use `hermes --profile PROFILE_NAME config set workdir /remote/path` for the remote working directory. The `terminal.cwd` key also exists but `workdir` is the one that controls the shell's starting directory in newer Hermes versions.

### workdir tilde expansion trap
Your local shell expands `~` BEFORE `hermes` sees it, so `hermes config set workdir ~` writes your LOCAL home path (e.g. `/Users/nsmirnov`) into the remote profile instead of the literal `~`. To set the remote home directory:
- Quote it: `hermes --profile PROFILE_NAME config set workdir '~'`
- Or, if it already expanded wrong, patch the profile's `config.yaml` to set `workdir: ~` directly.

### File tools are local-only
`read_file`, `write_file`, `search_files`, and `patch` operate on the local Mac filesystem, NOT the remote host. There is no built-in way to make them run remotely. Use terminal commands instead:
- Reading: `cat file` or `less file`
- Writing: `echo "content" > file` or `cat > file << 'EOF'`
- Searching: `grep -r "pattern" .` or `rg "pattern"`
- Editing: `sed -i 's/old/new/g' file` or `awk`
- Copying/moving/deleting: `cp`, `mv`, `rm`

### Cannot launch interactive REPL from CLI child process
When you run `hermes --profile PROFILE_NAME` from inside an active CLI session, it spawns a child process that the user cannot interact with. The child process gets its own stdin/stdout pipe — not the user's keyboard/terminal. Even running it in the background with PTY leaves it in a separate TTY you cannot hand off. **Do not attempt this.** Instead, give the user the command to run in their own terminal and provide the alias shortcut.

## Additional Commands

### "switch to PROFILE_NAME"
```bash
hermes profile use PROFILE_NAME
```

### "detach" or "go local"
```bash
hermes config set terminal.backend local
```

### "list profiles"
```bash
hermes profile list
```

### "remove" / "delete profile" / "clean up PROFILE_NAME"
Tears down a profile (sessions, memory, config). Confirm with the user first — it's irreversible and deletes the profile's plaintext SSH password along with everything else.
```bash
hermes profile remove PROFILE_NAME
```

## SSH Config Reference

| Config key | Env var | Description |
|-----------|---------|-------------|
| `terminal.backend` | `TERMINAL_ENV` | Set to `ssh` |
| `terminal.ssh_host` | `TERMINAL_SSH_HOST` | Remote hostname or IP |
| `terminal.ssh_user` | `TERMINAL_SSH_USER` | SSH username |
| `terminal.ssh_port` | `TERMINAL_SSH_PORT` | SSH port (default: 22) |
| `terminal.ssh_key` | `TERMINAL_SSH_KEY` | Path to SSH private key |
| `terminal.ssh_password` | `TERMINAL_SSH_PASSWORD` | SSH password (requires sshpass) |

## Notes

- **sshpass is required for password auth** — the skill installs it automatically
- **SSH keys are recommended** for security, but password auth works via sshpass
- **Profiles are isolated** — each machine gets its own sessions, memory, skills, cron jobs
- **Only `terminal` commands run remotely.** File tools (`read_file`, `write_file`, `search_files`, `patch`) and code execution (`execute_code`) are local-only and cannot access the remote filesystem. For remote file operations, use terminal commands: `cat`, `echo >`, `grep`/`rg`, `sed`/`awk`, `cp`, `rm`, etc.
- **Browser tools stay local** — only terminal runs remotely
- **New session required** after changing the backend
- **Long sessions need SSH keepalive** — idle connections drop mid-task. Add `ServerAliveInterval` to `~/.ssh/config` (see troubleshooting)

## Troubleshooting

See `references/troubleshooting.md` for common errors (connection refused, host key mismatch, permission denied, password rejected, etc.) and debug commands.

## Implementation Details

Source code changes that enabled `ssh_password` support: `references/ssh-password-impl.md`
