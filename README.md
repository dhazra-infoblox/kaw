# kaw - Kubernetes AI Agent Warden

A Kubernetes security tool to verify third-party AI agent identities and protect against malicious domains using DNS threat intelligence.

## What it does

- **Identity Validation**: Verify agents via labels, X.509 certificates, or JWT tokens
- **DNS Threat Protection**: Block malicious domains using Quad9, Cloudflare, or local blocklists
- **Policy Engine**: OPA-based admission control with Rego policies

## Quick Start

```bash
# Install CRDs
kubectl apply -f config/crd/

# Deploy
kubectl apply -f config/deployment/

# Test
kubectl apply -f test/fixtures/trusted-pod.yaml  # should succeed
kubectl apply -f test/fixtures/untrusted-pod.yaml  # should fail
```

## Architecture

- **Admission Webhook**: Validates pod create/update requests
- **Controller**: Reconciles AgentWardenConfig CRD to ConfigMap
- **CoreDNS Plugin**: Runtime DNS monitoring

## Config

Edit `config/samples/warden_v1alpha1_agentwardenconfig.yaml`:

```yaml
spec:
  identity:
    methods: [label, certificate, jwt]
    trustedVendors: [acme, contoso]
  dns:
    providers:
      - name: local-blocklist
        type: local
      - name: quad9
        type: quad9
    chainMode: sequential
```

## Tech Stack

- Go
- Kubernetes (operator-sdk)
- OPA/Rego

## Docs

- [Architecture Diagram](docs/architecture.mermaid)
