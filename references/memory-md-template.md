# MEMORY.md Template for SSH Profiles

Hermes automatically injects `memories/MEMORY.md` into every session's context.
Use it to permanently remind the agent how to work with the remote filesystem.

## How it works

1. Create `~/.hermes/profiles/PROFILE_NAME/memories/MEMORY.md`
2. Hermes loads it at the start of every session with that profile
3. The agent follows the constraints listed there

## Template

```markdown
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
```

## Why this is needed

The SSH backend only routes `terminal` commands to the remote host. File tools
(`read_file`, `write_file`, `search_files`, `patch`) and code execution
(`execute_code`) always run on the local Mac. Without this MEMORY.md, the agent
will try to use local file tools and silently operate on the wrong filesystem.

## Creating it programmatically

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
