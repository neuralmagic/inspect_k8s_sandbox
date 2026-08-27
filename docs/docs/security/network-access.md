# Network Access

!!! danger

    We recommend against allowing internet access, this is a common way for
    agents to violate or bypass other sandbox restrictions.

It is good security practice to prevent your containers from communicating with the
internet by default.

However, some evals may require internet access (e.g. to research topics). The
[built-in Helm chart](../helm/built-in-chart.md) allows you to specify a list of
domains that your containers can access. Use this with care.

??? tip "Internet access for dependencies"

    Allowing broad internet access for installing dependencies is not
    recommended. Where possible, use an internal mirror or artefact storage for
    downloading packages from within your own infrastructure, or include
    critical packages preinstalled in the docker container running the eval.

## Choosing a network policy provider

The built-in Helm chart can render its network policies for two engines, selected with
`networkPolicyProvider`:

* `ovn` (default) — emits plain `networking.k8s.io/v1 NetworkPolicy` resources for
  clusters **without** Cilium, such as
  [OVN-Kubernetes](https://github.com/ovn-org/ovn-kubernetes) / OpenShift.
* `cilium` — emits `CiliumNetworkPolicy` resources. Requires
  [Cilium](https://cilium.io/) and gives the full guarantees described below.

!!! warning "Reduced guarantees with `ovn`"

    Plain Kubernetes `NetworkPolicy` is L3/L4 only and purely additive, so the `ovn`
    provider **cannot** replicate several Cilium-only controls:

    * **`allowDomains` is not domain-pinned.** With `ovn`, a non-empty `allowDomains`
      opens broad egress to *all* IPs on the common web ports (80, 443, 8443, 8000,
      5000) plus any `allowDomainsPorts`. This matches Cilium's `allowDomains: ["*"]`
      behaviour only — traffic is **not** restricted to the listed domains, and there
      is no TLS SNI / HTTP Host identity check. Prefer `allowCIDR` (rendered as an
      `ipBlock`) when you need to restrict egress.
    * **DNS lookups are not filtered.** The anti-exfiltration DNS allow-list is not
      enforced; every query the sandbox makes is forwarded to cluster DNS.
    * **No ICMP rules.** A service that restricts ingress to specific `ports` does not
      get a separate ping (ICMP) allowance; services with no `ports` allow all
      protocols, including ICMP.

    On the `ovn` path, `runtimeClassName` is omitted (rather than defaulting to
    `gvisor`), so the cluster default runtime — e.g. `crun` on OpenShift/cri-o, which
    typically has no gvisor RuntimeClass — is used. Pinning a non-existent RuntimeClass
    otherwise leaves Pods `Pending` until the Helm install times out.

    Same-sandbox isolation, default-deny ingress, network-scoped ingress, `allowCIDR`,
    `allowEntities: world`/`cluster`, and `network_mode: none` (isolated) services are
    all still enforced.

## Cilium

With `networkPolicyProvider: cilium`, the built-in Helm chart uses
[Cilium](https://cilium.io/) Network Policies to restrict network access.

Cilium has tooling to observe network requests, such as
[Hubble](https://github.com/cilium/hubble). Though note from the
[limitations](../design/limitations.md) section that domain names will not be shown when
using the built-in Helm chart due to how DNS resolution is handled.

See the [limitations](../design/limitations.md) section for how Cilium may make certain
Cyber misuse evals harder or impossible to solve.

### DNS Exfiltration

The built-in Helm chart prevents DNS exfiltration attacks. This is where an attacker
uses DNS lookups to an attacker-controlled domain or set of subdomains in order to
exfiltrate data from a system. For example, a malicious agent could make DNS requests
(which go via the kube-dns service if the Core DNS sidecar can't resolve them) to
hostnames like `somedata.attacker.com` and `somedata2.attacker.com` to exfiltrate data.

DNS lookups are restricted to only:

* Services within the Kubernetes namespace
* The domains specified in the `allowDomains` list in the `values.yaml` file.

See the [limitations](../design/limitations.md) section for how this may affect your
evals.

??? danger "I don't want to restrict DNS lookups"

    The list of DNS names that can be queried is controlled by the same allow-list as
    general traffic egress to a domain. In order to allow all DNS lookups without
    restriction (including reverse DNS), you will need to allow all traffic to the
    internet.

    Please consider the security implications: your containers will have unrestricted
    internet access (including the ability to use DNS to exfiltrate data).

    To allow **all** DNS queries and **all** internet access, set
    `allowDomains: ["*"]` or `allowEntities: "all"` in the `values.yaml` file.
