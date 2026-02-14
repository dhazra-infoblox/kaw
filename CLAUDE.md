# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**k8s-agent-warden** - Kubernetes AI Agent Warden

A security tool to verify third-party AI agent identities in Kubernetes clusters and protect against hijacked/malicious domains using DNS threat intelligence.

## Problem Statement

When deploying third-party AI agents in Kubernetes, there is no reliable way to verify:
- **Identity**: That an agent is actually from the claimed vendor (e.g., "Mastercard")
- **Integrity**: That the agent hasn't been compromised or tampered with
- **Safety**: That the agent's outbound connections aren't to hijacked or malicious domains

## Requirements

### 1. DNS Threat Validation (Core Feature)

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

### 2. Agent Identity Validation

**Data Sources (all supported):**
- ConfigMap / CRD - manually defined list of trusted vendor domains
- Dynamic discovery - automatically discover vendor domains from cluster resources
- Certificate-based - extract domains from agent X.509 certificates

**Identity Verification Methods:**
- mTLS certificates (X.509)
- JWT/OIDC tokens

### 3. Policy Engine (OPA/Gatekeeper)

**Policy Support:**
- OPA (Open Policy Agent) integration for declarative policy enforcement
- Gatekeeper as admission controller for Kubernetes-native policy enforcement
- Rego policy language for writing agent identity and domain validation policies

**Policy Use Cases:**
- Allow only trusted vendor agents (whitelist by vendor label)
- Block agents connecting to known malicious domains
- Enforce required labels/annotations on agent pods
- Validate agent certificates at deployment time
- Rate limiting for agent API calls

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

## Architecture

```
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
│  │                    k8s-agent-warden                         │   │
│  │  • Validates certificates/tokens                            │   │
│  │  • Checks domains against DNS threat feeds                  │   │
│  │  • Blocks compromised agents                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              DNS Validation Pipeline                         │   │
│  │                                                              │   │
│  │   ┌──────────────────────────────────────────────────────┐ │   │
│  │   │         Unified DNS Provider Interface                 │ │   │
│  │   │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │ │   │
│  │   │  │ Quad9  │  │Cloudflare│ │Umbrella│  │ Custom │    │ │   │
│  │   │  └────────┘  └────────┘  └────────┘  └────────┘    │ │   │
│  │   └──────────────────────────────────────────────────────┘ │   │
│  │                              │                               │   │
│  │   ┌──────────┐   ┌──────────┐   ┌──────────┐              │   │
│  │   │  Local  │ → │ Chained  │ → │  Cache   │ → Allowed    │   │
│  │   │Blocklist│   │  Check   │   │          │              │   │
│  │   └──────────┘   └──────────┘   └──────────┘              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    OPA Policy Engine                          │   │
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
│   └── policies/   # OPA/Gatekeeper policy definitions
├── controllers/    # Reconciliation logic
├── internal/       # Internal packages
│   ├── dns/        # DNS validation logic
│   │   ├── provider/    # Unified provider interface
│   │   │   └── interface.go
│   │   ├── chain/      # Provider chaining logic
│   │   ├── cache/      # Local cache
│   │   └── local/      # Local DNS / blocklist
│   ├── identity/   # Agent identity validation
│   ├── policy/     # OPA policy evaluation
│   └── providers/  # External DNS provider implementations
├── pkg/            # Shared packages
└── Makefile        # Build automation
```
