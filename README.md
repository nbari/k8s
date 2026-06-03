# Kubernetes Platform Notes

Public, secret-free documentation for the `talos-48` Kubernetes platform.

## talos-48

- Domain: `48.network`
- Kubernetes API VIP: `10.246.0.80`
- Control plane nodes: `10.246.0.81-10.246.0.83`
- Worker nodes: `10.246.0.84-10.246.0.87`
- CNI: Cilium with eBPF kube-proxy replacement
- Router/BGP peer: VyOS `10.246.0.1`
- LoadBalancer pool: `10.246.2.2-10.246.2.14`
- Active Talos Gateway VIP: `10.246.2.3`

`10.246.2.2` remains assigned to the older RKE2 cluster during migration. The
Talos cluster uses `10.246.2.3` to avoid both clusters advertising the same
BGP route.

## GitOps Layout

- `talos-48`: Talos machine configuration and encrypted recovery material.
- `talos-platform-gitops`: Flux-managed platform components.
- `talos-apps-gitops`: Application workloads after the platform is stable.
- `k8s`: public documentation and diagrams only.

## TLS and PKI

Use cert-manager for Kubernetes certificate automation and Vault PKI as the
internal certificate authority.

cert-manager handles:

- watching `Certificate` resources
- requesting and renewing certificates
- storing TLS material as Kubernetes `Secret` objects
- making certificates easy for Gateway API and applications to consume

Vault PKI handles:

- internal CA and intermediate CA management
- certificate signing policy
- trust roots for private services
- future workload identity and mTLS use cases

Recommended model:

```text
Gateway or app needs TLS
        |
        v
cert-manager Certificate
        |
        v
Vault-backed Issuer or ClusterIssuer
        |
        v
Vault PKI signs the certificate
        |
        v
cert-manager writes a Kubernetes TLS Secret
        |
        v
Envoy Gateway or the app consumes the Secret
```

Use Vault PKI for internal `48.network` services. Use public ACME, such as
Let's Encrypt with Cloudflare DNS-01, only for names that require public browser
trust without installing the internal CA.
