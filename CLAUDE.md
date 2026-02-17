# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**KAW** - Kubernetes Agent Warden

A security tool to verify third-party AI agent identities in Kubernetes clusters using an external CA service for identity verification and DNS threat intelligence.

## Problem Statement

When deploying third-party AI agents in Kubernetes, there is no reliable way to verify:
- **Identity**: That an agent is actually from the claimed vendor (e.g., "Mastercard")
- **Integrity**: That the agent hasn't been compromised or tampered with
- **Safety**: That the agent's outbound connections aren't to hijacked or malicious domains

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Two-Project System                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      CA as SaaS (External)                       │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              CA Service (https://ca.example.com)        │   │
│   │                                                          │   │
│   │   • Vendor registration                                 │   │
│   │   • X.509 certificate issuance                         │   │
│   │   • REST API: GET /certificates/{vendor}              │   │
│   │   • Returns: cert + signature                         │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                │ Returns cert + signature
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     KAW (In Customer Cluster)                    │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  1. Receive pod create request                          │   │
│   │  2. Extract vendor from pod labels                     │   │
│   │  3. Query CA: GET /certificates/{vendor}              │   │
│   │  4. Validate:                                          │   │
│   │     • Signature (using CA public key)                  │   │
│   │     • Not expired                                      │   │
│   │     • Vendor matches                                    │   │
│   │  5. DNS validation (optional)                          │   │
│   │  6. OPA policy (optional)                              │   │
│   │  7. Allow/Deny                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Vendor Registration (CA Service - Separate Project)

```
┌──────────┐    ┌─────────────┐    ┌──────────────┐
│  Vendor  │───→│    CA      │───→│  Certificate │
│  applies │    │   Service  │    │   issued     │
│          │    │ (validates)│    │              │
└──────────┘    └─────────────┘    └──────────────┘
```

### 2. Agent Deployment (KAW Flow)

```
1. User deploys agent (with vendor labels, NO certificate):
   apiVersion: v1
   kind: Pod
   metadata:
     labels:
       vendor: acme-corp
       agent: acme-agent-v1
   spec:
     containers:
       - name: agent
         image: acme/agent:latest

2. KAW intercepts pod create request

3. KAW extracts vendor from labels:
   - vendor: acme-corp
   - agent: acme-agent-v1

4. KAW queries CA Service:
   GET https://ca.example.com/certificates/acme-corp/acme-agent-v1

5. CA returns:
   {
     "vendor": "acme-corp",
     "agent_id": "acme-agent-v1",
     "certificate": "-----BEGIN CERTIFICATE-----...",
     "signature": "...",
     "issued_at": "2024-01-01T00:00:00Z",
     "expires_at": "2025-01-01T00:00:00Z"
   }

6. KAW validates:
   - vendor label matches cert.vendor
   - agent label matches cert.agent_id
   - Signature (using trusted CA public key)
   - Not expired

7. (Optional) DNS validation
8. (Optional) OPA policy evaluation

9. Allow or Deny
```

## Requirements

### 1. Agent Identity Validation (via External CA)

**CA Service Integration:**
- KAW queries external CA service to get vendor + agent certificates
- Similar to how DigiCert issues SSL certificates for websites
- CA Service is a separate SaaS (not part of KAW)
- KAW trusts CA via public key in ConfigMap
- Each agent version gets its own certificate (per-agent)

**CA Service API:**
```
GET /certificates/{vendor}/{agent_id}

Response:
{
  "vendor": "acme-corp",
  "agent_id": "acme-agent-v1",
  "certificate": "-----BEGIN CERTIFICATE-----...",
  "signature": "...",
  "issued_at": "2024-01-01T00:00:00Z",
  "expires_at": "2025-01-01T00:00:00Z"
}
```

**KAW Flow:**
1. User deploys agent pod with labels:
   - `vendor: acme-corp`
   - `agent: acme-agent-v1`
2. KAW intercepts pod create request
3. KAW queries CA Service: `GET /certificates/acme-corp/acme-agent-v1`
4. KAW validates:
   - vendor label matches cert.vendor
   - agent label matches cert.agent_id
   - signature valid
   - not expired
5. Allow or Deny

**User Experience:** User just deploys agent - no certificate management needed.

**Registration Flow:**
1. Vendor registers with CA (company, domain, contact)
2. Vendor registers their agents (agent_id, version, metadata)
3. CA issues certificate per agent
4. Certificate contains: vendor + agent_id + signature

### 2. DNS Threat Validation (Optional)

**Unified Provider Interface:**
- Common interface for all external DNS providers
- Pluggable architecture - easily add new providers
- Support for:
  - Quad9 (free, public)
  - Cloudflare (API required)
  - Cisco Umbrella (API required)
  - Custom blocklists

**Provider Chaining Modes:**
- Sequential (fail-safe) - check provider A, then B, then C
- Parallel (faster) - check all providers simultaneously
- Weighted - each provider has a trust score, aggregate results
- Fallback - use provider A, if unavailable use B

**Local DNS Capabilities:**
- Local blocklist file (static list of blocked domains)
- Local DNS resolver (run a local DNS server that queries upstream)
- Local cache (cache results from external providers for performance)

### 3. Policy Engine (Optional)

**Policy Support:**
- OPA (Open Policy Agent) integration for declarative policy enforcement
- Embedded OPA (no extra pods)
- Rego policy language for writing agent identity and domain validation policies

**Policy Use Cases:**
- Allow only trusted vendor agents (whitelist by vendor label)
- Block agents connecting to known malicious domains
- Enforce required labels/annotations on agent pods
- Validate agent certificates at deployment time

**Example Policies (Rego):**
```rego
# Allow only trusted vendor agents
allow[msg] {
  input.request.kind.kind == "Pod"
  vendor := input.request.object.metadata.labels.vendor
  trusted_vendors[vendor]
  msg := ""
}

# Block agents connecting to malicious domains
deny[msg] {
  input.request.kind.kind == "Pod"
  domain := input.request.object.spec.containers[_].env[_].value
  malicious_domains[domain]
  msg := sprintf("Blocked malicious domain: %v", [domain])
}
```

### 4. Scalability

- Single cluster deployment
- Multiple clusters support (shared threat data)
- Global distribution capability

### 5. Configuration (KawConfig CRD)

KAW is configured via the `KawConfig` CRD:

```yaml
apiVersion: warden.k8s.io/v1alpha1
kind: KawConfig
metadata:
  name: default
spec:
  # CA Service configuration
  caService:
    endpoint: https://ca.example.com
    publicKey: |
      -----BEGIN PUBLIC KEY-----
      ...
      -----END PUBLIC KEY-----
    insecureSkipVerify: false
    timeout: 10s

  # Identity Validation
  identity:
    enabled: true
    vendorLabel: vendor        # Label to extract vendor name
    agentLabel: agent          # Label to extract agent name

  # DNS Validation (optional)
  dns:
    enabled: false
    providers:
      - name: local-blocklist
        type: local
      - name: quad9
        type: quad9
    chainMode: sequential

  # Policy Engine (optional)
  policy:
    enabled: false
    bundleName: kaw-policies
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     External CA Service (SaaS)                        │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  CA Service (https://ca.example.com)                       │   │
│   │  • Vendor registration                                      │   │
│   │  • X.509 certificate issuance                               │   │
│   │  • REST API: GET /certificates/{vendor}                   │   │
│   └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Returns cert + signature
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                              │
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │
│  │   Agent A   │    │   Agent B   │    │   Agent C   │            │
│  │ (vendor: Acme) │  │ (vendor: Contoso) │ │ (vendor:未知) │            │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘            │
│         │                  │                  │                    │
│         ▼                  ▼                  ▼                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                         KAW                                 │   │
│  │  • Queries CA Service for certificates                      │   │
│  │  • Validates: signature, expiry, vendor match             │   │
│  │  • Checks domains against DNS threat feeds (optional)     │   │
│  │  • Blocks unverified agents                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              DNS Validation Pipeline (Optional)               │   │
│  │                                                              │   │
│  │   ┌──────────────────────────────────────────────────────┐ │   │
│  │   │         Unified DNS Provider Interface                 │ │   │
│  │   │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │ │   │
│  │   │  │ Quad9  │  │Cloudflare│ │Umbrella│  │ Custom │    │ │   │
│  │   │  └────────┘  └────────┘  └────────┘  └────────┘    │ │   │
│  │   └──────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    OPA Policy Engine (Optional)               │   │
│  │  ┌─────────────────┐    ┌─────────────────────────────┐    │   │
│  │  │  Gatekeeper     │ ←  │  Rego Policies              │    │   │
│  │  │  (Admission)    │    │  • Vendor whitelist          │    │   │
│  │  │                 │    │  • Domain blocklist          │    │   │
│  │  │                 │    │  • Required labels          │    │   │
│  │  └─────────────────┘    └─────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Flow

```
1. Agent deployment request arrives at Kubernetes API
2. Gatekeeper intercepts via admission webhook
3. OPA evaluates Rego policies:
   - Check vendor is trusted
   - Check domains against DNS validation pipeline
   - Check required labels/annotations
4. Policy decision: Allow or Deny
5. If allowed: Agent pod created
6. Runtime: DNS validation pipeline validates outbound connections
7. Log and alert as needed
```

## Tech Stack

- **Language**: Go
- **Kubernetes**: operator-sdk / kubebuilder
- **Policy Engine**: OPA + Gatekeeper
- **Policy Language**: Rego
- **Deployment**: Kubernetes operator + admission webhook
- **DNS Resolution**: CoreDNS integration or custom DNS server

## Build Commands

```bash
# Generate operator code
make generate
make manifests

# Build and deploy
make docker-build
make docker-push
make deploy

# Run locally (without cluster)
make run

# Run tests
make test

# Apply OPA policies
make apply-policies
```

## Project Structure

```
.
├── api/            # Kubernetes API definitions (CRDs)
├── config/         # Kubernetes manifests (CRD, RBAC, deployments)
│   └── policies/   # OPA policy definitions
├── controllers/    # Reconciliation logic
├── internal/       # Internal packages
│   ├── webhook/   # Admission webhook handlers
│   ├── identity/  # Agent identity validation
│   │   ├── client.go   # CA Service HTTP client
│   │   └── validator.go # Certificate validation
│   ├── dns/       # DNS validation logic (optional)
│   │   ├── provider/    # Unified provider interface
│   │   ├── chain/      # Provider chaining logic
│   │   └── cache/      # Local cache
│   └── policy/    # OPA policy evaluation (optional)
├── helm/kaw/      # Helm chart
└── Makefile       # Build automation
├── pkg/            # Shared packages
└── Makefile        # Build automation
```
