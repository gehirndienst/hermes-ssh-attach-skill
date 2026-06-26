# SSH Connection Troubleshooting

## Common Errors and Fixes

### "sshpass not found" / "sshpass is required"
**Cause:** `sshpass` binary is not installed on the local machine.
**Fix:**
```bash
# macOS
brew install sshpass

# Ubuntu/Debian
sudo apt install sshpass

# CentOS/RHEL
sudo yum install sshpass
```

### "SSH connection failed: Permission denied (publickey,password)"
**Cause:** Key-based auth failed AND password is not configured, or password is wrong.
**Fix:**
1. Test key auth manually: `ssh -o BatchMode=yes user@host echo OK`
2. If it fails, set up keys: `ssh-copy-id user@host`
3. OR configure password: `hermes config set terminal.ssh_password "YOUR_PASSWORD"`

### "SSH connection timed out"
**Cause:** Network unreachable, firewall blocking port, or wrong host/port.
**Fix:**
```bash
# Test basic connectivity
ping HOST
ssh -v -p PORT user@host  # Verbose output shows exactly where it fails
```

### "Host key verification failed"
**Cause:** Known hosts mismatch — the server's host key changed.
**Fix:** Remove the old key and re-accept:
```bash
ssh-keygen -R HOST
ssh user@host  # Will prompt to accept new key
```
Note: Hermes auto-accepts new host keys (`StrictHostKeyChecking=accept-new`), so this only matters if you previously added the host manually.

### "Control socket too long" / macOS socket path limit
**Cause:** macOS enforces a 104-byte limit on Unix domain socket paths. Long hostnames + nested $TMPDIR can exceed this.
**Fix:** Hermes already handles this by hashing the socket path. If you still hit it, set `TMPDIR=/tmp` or use a shorter hostname.

### "password authentication failed" after setting ssh_password
**Cause:** Wrong password, or server requires keyboard-interactive auth (not password auth).
**Fix:**
1. Verify password works manually: `SSHPASS="PASSWORD" sshpass -e ssh user@host echo OK`
2. If manual test fails, the server may require keyboard-interactive auth — sshpass doesn't support this reliably. Use SSH keys instead.
3. Check `sshd_config` on server for `PasswordAuthentication yes` and `KbdInteractiveAuthentication no`

### "Connection refused"
**Cause:** SSH daemon not running, or wrong port.
**Fix:**
```bash
# Check if SSH is listening
ssh -p PORT user@host -v 2>&1 | grep "Connection refused"

# Common alternative ports: 22 (default), 2222, 8022
```

### "Permission denied for key file"
**Cause:** SSH private key has wrong permissions (must be 600).
**Fix:**
```bash
chmod 600 ~/.ssh/your_key_file
```

### "Session froze" / "Write failed: Broken pipe" / "Connection reset" mid-task
**Cause:** Agent sessions are long-lived. An idle SSH connection gets dropped by the server, a NAT, or a firewall after a timeout.
**Fix:** Enable keepalive so SSH sends periodic packets. Hermes' SSH backend honors `~/.ssh/config`, so add a host block:
```
Host HOST
    ServerAliveInterval 30
    ServerAliveCountMax 6
    TCPKeepAlive yes
```
This pings every 30s and only gives up after 6 missed (~3 min). Apply globally with `Host *` if you attach to many machines.

## Debug Commands

When something fails, these commands help diagnose:

```bash
# Verbose SSH test (shows auth methods offered by server)
ssh -vvv user@host echo OK

# Test password auth manually
SSHPASS="PASSWORD" sshpass -e ssh -vvv user@host echo OK

# Check Hermes doctor for the profile
hermes --profile PROFILE_NAME doctor

# Check Hermes status
hermes status
```

## When Keys Are NOT an Option

If you cannot set up SSH keys (e.g., managed hosting, shared servers), password auth via sshpass is the only option. Remember:
- `sshpass` must be installed locally
- Password is stored in `config.yaml` or `.env` in plaintext
- Consider using SSH agent forwarding (`-o ForwardAgent=yes`) as a middle ground

## Profile-Specific Issues

### "Hermes isn't configured yet -- no API keys or providers found"

**Cause:** New profiles ship with `model: ''` and `providers: {}`. The user's model config (local LLM base_url, provider name, etc.) lives in `~/.hermes/config.yaml`, NOT in the new profile.

**Fix:** Copy the `model` and `providers` blocks from `~/.hermes/config.yaml` into `~/.hermes/profiles/PROFILE_NAME/config.yaml`. Example:

```yaml
# Replace:
model: ''
providers: {}

# With:
model:
  base_url: http://YOUR_LOCAL_LLM:PORT/v1
  default: YOUR_MODEL_NAME
  provider: custom
providers:
  your_provider_name:
    api: http://YOUR_LOCAL_LLM:PORT/v1
    name: your_provider_name
    default_model: YOUR_MODEL_NAME
```

Then run: `hermes --profile PROFILE_NAME doctor --fix`

### "~/.hermes/.env is just a template (all keys are *** masked)"

**Cause:** For local LLM users, the .env file is a template with no real keys. The model config lives in config.yaml.

**Fix:** Do NOT rely on `.env` for model config. Read `~/.hermes/config.yaml` for `model` and `providers` blocks. Only copy `.env` if the user has API-key providers (OpenRouter, etc.).

### "Wrong key works, right key fails (ed25519 vs rsa)"

**Cause:** User has multiple SSH keys but only some are authorized on the target server. The key file name (`id_ed25519` vs `id_rsa`) tells you NOTHING about which server accepts it.

**Fix:** Test each key individually:
```bash
for KEY in ~/.ssh/id_ed25519 ~/.ssh/id_ecdsa ~/.ssh/id_rsa ~/.ssh/id_dsa ~/.ssh/*.pem; do
  [ -f "$KEY" ] || continue
  echo -n "  $KEY: "
  ssh -o BatchMode=yes -o ConnectTimeout=5 -o IdentitiesOnly=yes -i "$KEY" user@host echo OK && echo " WORKS" && break
  echo " FAILED"
done
```

Configure the profile with the key that actually returned `SSH_OK`.

### "BatchMode=yes fails but interactive SSH works"

**Cause:** The key has a passphrase, or only the SSH agent has the decrypted key.

**Fix:** Check agent status:
```bash
echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK"
ssh-add -l
```

If agent is empty (`no identities`), the key either has a passphrase or isn't available. Options:
1. Add key to agent: `ssh-add ~/.ssh/id_ed25519`
2. Use a key without a passphrase
3. Configure password auth via sshpass
