# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**k8s-agent-warden** - Kubernetes AI Agent Warden

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

### 3. Scalability

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
└─────────────────────────────────────────────────────────────────────┘
```

## Data Flow

```
1. Agent attempts outbound connection to domain X
2. Request intercepted by DNS validation pipeline
3. Check local blocklist/cache first
4. Query chained external providers via unified interface
5. Verify domain against trusted vendor list (ConfigMap/CRD/Certificate)
6. Allow or block based on results
7. Log and alert as needed
```

## Tech Stack

- **Language**: Go
- **Kubernetes**: operator-sdk / kubebuilder
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
```

## Project Structure

```
.
├── api/            # Kubernetes API definitions (CRDs)
├── config/         # Kubernetes manifests (CRD, RBAC, deployments)
├── controllers/    # Reconciliation logic
├── internal/       # Internal packages
│   ├── dns/        # DNS validation logic
│   │   ├── provider/    # Unified provider interface
│   │   │   └── interface.go
│   │   ├── chain/      # Provider chaining logic
│   │   ├── cache/      # Local cache
│   │   └── local/      # Local DNS / blocklist
│   ├── identity/   # Agent identity validation
│   └── providers/  # External DNS provider implementations
├── pkg/            # Shared packages
└── Makefile        # Build automation
```
