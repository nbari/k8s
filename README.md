# Kubernetes Platform Notes

Public, secret-free documentation for the `talos-48` Kubernetes platform.

This repository is intentionally documentation-only. It summarizes the platform
architecture, design decisions, and implementation status without storing
secrets, kubeconfigs, recovery keys, private tokens, or raw machine
configuration.

## Goal

Build a production-style Kubernetes platform for private infrastructure using
immutable Talos nodes, GitOps, Cilium eBPF networking, BGP-advertised ingress,
Vault-backed secret management, persistent storage, and a complete
observability stack.

The platform is designed to replace an older Rancher/RKE2 cluster gradually
without forcing every application to be migrated at once.

## talos-48

- Domain: `48.network`
- Kubernetes API VIP: `10.246.0.80`
- Control plane nodes: `10.246.0.81-10.246.0.83`
- Worker nodes: `10.246.0.84-10.246.0.87`
- CNI: Cilium with eBPF kube-proxy replacement
- Router/BGP peer: VyOS `10.246.0.1`
- LoadBalancer pool: `10.246.2.2-10.246.2.14`
- Active Talos Gateway VIP: `10.246.2.3`
- Public Vault endpoint: `https://vault.48.network`
- Private platform endpoints: `https://grafana.48.network`, `https://logs.48.network`, `https://metrics.48.network`, `https://alertmanager.48.network`, `https://hubble.48.network`

`10.246.2.2` remains assigned to the older RKE2 cluster during migration. The
Talos cluster uses `10.246.2.3` to avoid both clusters advertising the same
BGP route.

## GitOps Layout

- `talos-48`: Talos machine configuration and encrypted recovery material.
- `talos-platform-gitops`: Flux-managed platform components.
- `talos-apps-gitops`: Application workloads after the platform is stable.
- `talos-48-vault`: Terraform-managed Vault API configuration (auth methods, PKI, policies, secret engines).
- `k8s`: public documentation and diagrams only.

## Implemented Capabilities

- Immutable Talos control plane with three control-plane nodes and four workers.
- Static node addressing, API VIP, and split DNS under `48.network`.
- Cilium eBPF kube-proxy replacement with no flannel and no kube-proxy pods.
- Cilium BGP peering with VyOS for Kubernetes LoadBalancer/Gateway VIPs.
- Hubble network observability UI with relay for live traffic flow inspection.
- Envoy Gateway as the private ingress layer for platform services.
- cert-manager for certificate lifecycle automation.
- Let's Encrypt DNS-01 for browser-trusted private routes.
- Vault HA Raft with AWS KMS auto-unseal and persistent Ceph RBD volumes.
- Vault API configuration (auth methods, PKI mounts, policies, KV engine) managed via Terraform in `talos-48-vault`.
- External Secrets Operator reading Vault over HTTPS.
- Vault PKI integrated with cert-manager for internal certificate issuance.
- Cloudflare external-dns with an explicit `external-dns=public` label gate.
- VictoriaMetrics, VictoriaLogs, Grafana, and Alertmanager for observability.
- Grafana GitHub OAuth and PostgreSQL-backed state.
- Alertmanager Slack routing through a Vault-managed webhook.
- Talos control-plane, kubelet, etcd, Cilium, node-exporter, and platform metrics scraping.

## TLS and PKI

cert-manager handles all certificate automation. Two issuers are in use:

**Let's Encrypt DNS-01** (`letsencrypt-cloudflare-production-talos`) — all
browser-facing endpoints regardless of whether they are public or private. This
avoids distributing an internal CA to client devices while keeping a consistent
green-padlock experience on every endpoint.

**Vault PKI** (`vault-internal`) — internal service-to-service TLS only. Used
for certificates consumed by Kubernetes workloads rather than browsers, such as
the Vault backend certificate validated by Envoy's `BackendTLSPolicy`.

```text
Browser-facing endpoint (e.g. hubble.48.network)
        |
        v
cert-manager Certificate → letsencrypt-cloudflare-production-talos
        |
        v
Let's Encrypt DNS-01 via Cloudflare API
        |
        v
cert-manager writes TLS Secret → gateway-system namespace
        |
        v
Envoy Gateway terminates TLS

Internal service-to-service TLS (e.g. Vault backend)
        |
        v
cert-manager Certificate → vault-internal ClusterIssuer
        |
        v
Vault PKI (pki_int/roles/kubernetes) signs the certificate
        |
        v
cert-manager writes TLS Secret → consumed by workload or BackendTLSPolicy
```

## Current Platform

The Talos cluster currently has the full platform online:

- Flux GitOps
- Cilium with eBPF kube-proxy replacement
- Cilium BGP and LB IPAM
- Hubble relay and UI at `https://hubble.48.network`
- Envoy Gateway on `10.246.2.3`
- cert-manager with Let's Encrypt DNS-01
- Ceph CSI RBD storage
- Vault HA Raft with AWS KMS auto-unseal
- Vault API configuration managed via Terraform (`talos-48-vault`)
- External Secrets Operator reading Vault over HTTPS
- Vault PKI exposed to cert-manager through the `vault-internal` ClusterIssuer
- external-dns for explicitly labeled public Cloudflare records
- VictoriaMetrics and VictoriaLogs for metrics and logs
- Grafana, Alertmanager

## Security Model

- Raw Talos machine configs, `kubeconfig`, and `talosconfig` are kept out of
  public documentation and private Git in plaintext.
- Durable Talos bootstrap material is stored in the private `talos-48`
  repository with SOPS + age encryption.
- Vault recovery keys and root token are stored outside Git (1Password).
- Vault API configuration is managed via Terraform with remote state in S3.
  The `terraform-operator` policy token is used for CI; the root token is
  break-glass only.
- Kubernetes consumes application and platform secrets through External Secrets
  Operator rather than committing Kubernetes `Secret` manifests with plaintext
  values.
- Public DNS automation is opt-in per HTTPRoute through the
  `external-dns=public` label.
- Private platform routes use split DNS and are not published by external-dns.

Vault traffic is encrypted end to end:

```text
client -> HTTPS vault.48.network:443 -> Envoy Gateway -> HTTPS vault-active.vault.svc:8200 -> Vault
```

external-dns is installed but intentionally conservative: it watches Gateway
HTTPRoutes for `48.network` and only manages routes labeled
`external-dns=public`.

Platform endpoints are private-only through split DNS to `10.246.2.3`:

- Grafana: `https://grafana.48.network`
- VictoriaLogs: `https://logs.48.network`
- VictoriaMetrics: `https://metrics.48.network`
- Alertmanager: `https://alertmanager.48.network`
- Hubble: `https://hubble.48.network`

Current validation:

- All seven Kubernetes nodes are `Ready`.
- Flux and all platform Helm releases are `Ready`.
- No pods are pending or crash-looping.
- VictoriaMetrics reports zero down scrape targets including etcd.
- Grafana login redirects to GitHub OAuth.
- Alertmanager loads the Slack receiver configuration from Vault.
- Hubble shows live pod network flows.
- `terraform plan` on `talos-48-vault` reports no changes.

## Operations Roadmap

The base platform is complete. Remaining work:

- Add backup and restore runbooks for Vault, Grafana PostgreSQL state,
  VictoriaMetrics, VictoriaLogs, and Ceph-backed platform volumes (Velero).
- Add upgrade runbooks for Talos, Kubernetes, Cilium, Flux, and platform Helm
  charts.
- Add CiliumNetworkPolicy resources for namespace isolation.
- Bootstrap `talos-apps-gitops` when application migration starts.

## Architecture

```mermaid
flowchart LR
  user[LAN / Tailscale client] --> dns[Split DNS / Cloudflare DNS]
  dns --> vip[Gateway VIP 10.246.2.3]
  vip --> envoy[Envoy Gateway]
  envoy --> vault[Vault HA Raft]
  envoy --> grafana[Grafana]
  envoy --> metrics[VictoriaMetrics]
  envoy --> logs[VictoriaLogs]
  envoy --> alerts[Alertmanager]
  envoy --> hubble[Hubble UI]
  envoy --> apps[Future apps]

  subgraph talos48[talos-48 Kubernetes]
    cilium[Cilium eBPF + BGP + Hubble]
    envoy
    cert[cert-manager]
    eso[External Secrets Operator]
    vault
    grafana
    metrics
    logs
    alerts
    hubble
    ceph[Ceph CSI RBD]
  end

  vyos[VyOS 10.246.0.1] <--> cilium
  cert --> letsencrypt[Let's Encrypt DNS-01]
  cert --> vaultpki[Vault PKI]
  envoy --> edns[external-dns]
  edns --> cloudflare[Cloudflare DNS]
  eso --> vault
  vaultpki --> vault
  vault --> kms[AWS KMS auto-unseal]
  vault --> ceph
  metrics --> ceph
  logs --> ceph
  grafana --> ceph
  alerts --> ceph
  terraform[Terraform talos-48-vault] --> vault
  terraform --> s3[S3 state eu-north-1]
```
