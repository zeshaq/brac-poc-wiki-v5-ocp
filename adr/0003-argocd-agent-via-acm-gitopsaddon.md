# ADR 0003: Argo CD Agent install via RHACM 2.16 `gitopsaddon` (supersedes ADR-0002)

## Status

Accepted — 2026-05-07. **Supersedes [ADR-0002](0002-argocd-agent-topology.md)** on
the PKI strategy and install path.

## Context

ADR-0002 captured three sub-decisions for the Argo CD Agent topology
(operating mode, Principal HA, PKI). It was accepted on 2026-05-07 morning
based on the assumption that the Agent would be installed manually via custom
Principal/Agent CR manifests in `lab-gitops/components/platform/argocd-agent-{principal,spoke}/`,
with a separately-managed `argocd-agentctl` CA persisted in Vault and projected
via ESO.

Same-day deeper research (and a cluster probe) revealed three facts that
overturn parts of ADR-0002:

1. **RHACM 2.15+ ships the Argo CD Agent as a Tech Preview** via the existing
   `gitopsaddon` mechanism, controlled by a flag on the `GitOpsCluster` CR.
   Quoted from the RHACM 2.15 release notes: "*You can choose to enable the
   OpenShift GitOps add-on with or without the ArgoCD agent. The Argo CD agent
   (available as a technology preview)…*"
   ([release notes](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html-single/release_notes/index)).
2. **Hub-dc is on RHACM 2.16.1** (verified 2026-05-07 via `oc get
   multiclusterhub`). The `GitOpsCluster` CRD on the live cluster includes
   `spec.gitopsAddon.argoCDAgent` as a real schema field with `enabled`, `mode`,
   `propagateHubCA`, `serverAddress`, and `serverPort`.
3. **An existing `GitOpsCluster` CR named `gitops-managed`** lives in
   `openshift-gitops` on hub-dc, 7d old. The whole cluster-side install
   reduces to **patching that existing CR** with the agent flag — the addon
   controller takes over from there.

The "headless" Argo CD CR `acm-openshift-gitops` on spoke-dc that I noted as
suspicious in the cluster probe is in fact intentional: the addon's CR layout
(`application-controller` + `repo-server` + `redis`, no `argocd-server`) is
purpose-built to attach a Principal/Agent. From Christian Stark's stderr.at
walkthrough (2026-01-14), confirmed by the RHACM 2.16.1 source on
`stolostron/multicloud-integrations`:
> "*the Principal component cannot be installed alongside an existing Argo CD
> instance where the application controller is enabled. You must create a
> separate Argo CD instance with the controller disabled.*"

The CR layout I observed on spoke-dc is the addon's own answer to that
constraint.

ADR-0002's Operating-mode decision (Managed) and Principal-HA decision (Single
Principal on hub-dc) are unaffected by this — both translate cleanly to the
`gitopsaddon` path (`mode: managed`, no warm-standby Principal on hub-dr in
normal ops; ACM Backup/Restore covers hub-dr activation).

## Decision

### Operating mode — Managed (unchanged from ADR-0002)

`spec.gitopsAddon.argoCDAgent.mode: managed`. Hub Argo CD on hub-dc owns the
workload `Application`/`ApplicationSet` resources; the Agent on spoke-dc
executes; status flows back to hub via the resource-proxy. Same as ADR-0002.

### Principal HA — Single Principal on hub-dc (unchanged from ADR-0002)

One Principal on hub-dc in normal operation. hub-dr's Principal arrives via
ACM Backup/Restore at activation time. No warm standby in normal ops. Same as
ADR-0002.

### PKI — ACM-propagated hub CA (**replaces** ADR-0002's `argocd-agentctl` + Vault)

`spec.gitopsAddon.argoCDAgent.propagateHubCA: true`. The hub controller
generates and rotates the Agent CA; ACM's existing `manifestwork` trust
mechanism propagates it to the managed cluster. Vault is not in the loop for
Agent PKI material.

This supersedes ADR-0002's `argocd-agentctl` + Vault choice. ADR-0004
(Vault-first secret policy) still applies to *application* secrets and CI
credentials; the Agent CA is operator-domain trust material, conceptually
distinct, and the canonical Red Hat install does not parameterise it through
Vault.

### Install path — patch the existing `GitOpsCluster` CR (**replaces** the manual install)

The cluster-side install reduces to a single patch on the existing CR:

```yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-managed
  namespace: openshift-gitops
spec:
  gitopsAddon:
    enabled: true
    argoCDAgent:
      enabled: true
      mode: managed
      propagateHubCA: true
```

No `lab-gitops/components/platform/argocd-agent-{principal,spoke}/` directories.
No separate Principal CR yaml. No separate Agent CR yaml. No `argocd-agentctl
init` step. The hub controller swaps the cluster's `AddOnTemplate` to
`gitops-addon-{ns}-{name}` (with cert registration), ships Principal config to
hub-dc, and reconfigures the existing managed-cluster operator deployment
(already labelled `apps.open-cluster-management.io/gitopsaddon: true`) to
attach the Agent.

The workload `ApplicationSet` (`workloads-agent-managed.yaml` in
`lab-gitops/argocd/applicationsets/`) remains as ADR-0002 described — that
piece is unaffected.

## Consequences

**Accepted**

- The Agent is **Tech Preview** in RHACM 2.15+, not GA. Production-grade
  workloads should be cautious; for the BRAC POC this is acceptable.
  GA timeline is not publicly committed by Red Hat.
- An OpenShift Platform Plus subscription is documented as a prerequisite
  per the Red Hat blog "Manage clusters and applications at scale with Argo
  CD Agent." Subscription entitlement on the lab clusters needs confirmation
  before the install lands. Tracked as a follow-up.
- The Agent CA is owned by ACM, not Vault. Rotation, revocation, and
  emergency re-issuance flow through ACM's hub controller, not the
  Vault+ESO path that other secrets in the lab use. This is a deliberate
  divergence — Agent PKI is operator infrastructure, not application
  secret material.
- Removing the addon (e.g., for migration to a different multi-cluster
  pattern in the future) requires unsetting `gitopsAddon.enabled` on the
  `GitOpsCluster`, which the addon controller handles. The
  `acm-openshift-gitops` ArgoCD CR persists; in-flight Applications it
  reconciled remain on the cluster.

**Avoided** (vs ADR-0002's superseded plan)

- Hand-rolling `argocd-agentctl init`, the Vault KV path, the ESO
  `ExternalSecret` projecting the CA into `openshift-gitops`, and the
  Principal CR boilerplate. The addon does this end-to-end.
- A second operator install on spoke-dc to get Agent-capable CRDs. The
  RHACM-bundled operator already there is what the addon configures.
- Migration of in-flight ACM-pulled Applications. The addon's CR
  (`acm-openshift-gitops`) is preserved; the Agent attaches to it.

**Required follow-up** (the platform PR scope)

- **Entitlement check.** Confirm OpenShift Platform Plus subscription on
  hub-dc and spoke-dc before the install. (Owner-managed, not a code
  change.)
- **`lab-gitops` PR**: patch the `gitops-managed` GitOpsCluster CR with the
  `gitopsAddon.argoCDAgent` block. Single small change.
- **`lab-gitops` PR**: add
  `argocd/applicationsets/workloads-agent-managed.yaml` ApplicationSet
  generating `<cluster>-workloads` Applications matching `spoke-dc` only.
  Retire the existing push-mode `workload-config.yaml` once the agent path
  is verified.
- **Verification**: confirm the Agent attaches to `acm-openshift-gitops`
  on spoke-dc (operator deployment reconciles to a new image with Agent
  enabled; agent registers with the Principal on hub-dc; client cert is
  provisioned by the addon).
- **Smoke**: verify a workload `Application` generated by the new
  `ApplicationSet` reconciles on spoke-dc through the Agent path, end-to-end.

## Alternatives Considered

### Stay on ADR-0002's path: manual `argocd-agentctl` + Vault + custom Principal/Agent CRs

Rejected because the canonical Red Hat path is documented and shipped in
RHACM 2.16, and it's strictly less work, less code in `lab-gitops`, less
custom PKI surface to maintain, and less risk of drift from Red Hat's
supported configuration. ADR-0002's plan was correct in October 2025 and
becomes obsolete the moment ACM 2.15 ships, which it has.

### Coexistence: keep the RHACM-bundled operator AND OLM-install OpenShift GitOps Operator on spoke-dc

Rejected. The `gitopsaddon` source explicitly handles the *opposite* — its
`olmSubscription.enabled` flag forces the addon to use OLM rather than its
embedded chart, and the addon detects existing OLM installs to skip its own.
Two independently-managed operators in the same namespace would conflict.

### Disable the `gitopsaddon` and OLM-install OpenShift GitOps Operator instead

Rejected. Loses the existing ACM-pull `spoke-dc-cluster-config` reconciliation
(currently Synced/Healthy via `manifestwork`); Application would need to
move to Argo CD Agent managed-mode along with workloads, which is more
churn than the canonical path needs. Reasonable if the lab decides to drop
ACM entirely from the GitOps story, but that's a much bigger architectural
shift than ADR-0001 + ADR-0002 contemplated.

## References

- [Manage clusters and applications at scale with Argo CD Agent (Red Hat blog, 2026-01-12)](https://www.redhat.com/en/blog/manage-clusters-and-applications-scale-argo-cd-agent-red-hat-openshift-gitops)
- [Multi-cluster GitOps with the Argo CD Agent Tech Preview (Red Hat blog, 2025-11-05)](https://www.redhat.com/en/blog/multi-cluster-gitops-argo-cd-agent-openshift-gitops)
- [What's New in OpenShift GitOps 1.19 (2026-01-12)](https://developers.redhat.com/blog/2026/01/12/whats-new-openshift-gitops-119)
- [What's New in OpenShift GitOps 1.20 (2026-03-26)](https://developers.redhat.com/blog/2026/03/26/whats-new-openshift-gitops-120)
- [Integrate Red Hat Advanced Cluster Management with Argo CD (2026-03-24)](https://developers.redhat.com/articles/2026/03/24/integrate-red-hat-advanced-cluster-management-argo-cd)
- [Using the Argo CD Agent with OpenShift GitOps — manual path, 2025-10-06 (the path ADR-0002 followed)](https://developers.redhat.com/blog/2025/10/06/using-argo-cd-agent-openshift-gitops)
- [stderr.at GitOps Collection Ep. 15 — Argo CD Agent (2026-01-14)](https://blog.stderr.at/gitopscollection/2026-01-14-argocd-agent/)
- [RHACM 2.15 Release Notes](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html-single/release_notes/index)
- [New efficiency upgrades in RHACM 2.15 (Red Hat blog)](https://www.redhat.com/en/blog/new-efficiency-upgrades-red-hat-advanced-cluster-management-kubernetes-215)
- [stolostron/multicloud-integrations (gitopsaddon source, COMPONENT_VERSION 2.17.0)](https://github.com/stolostron/multicloud-integrations)
- [argoproj-labs/argocd-agent](https://github.com/argoproj-labs/argocd-agent)
- ADR-0001 (spoke-dr platform standby): [adr/0001-spoke-dr-platform-standby.md](0001-spoke-dr-platform-standby.md) — informs the "single Principal" decision since spoke-dr is platform standby
- ADR-0002 (superseded by this one): [adr/0002-argocd-agent-topology.md](0002-argocd-agent-topology.md)
- Project ADR-0007 (no per-app Application yaml, AppSet-only): <https://github.com/zeshaq/brac-poc-openliberty-v5/blob/main/docs/adr/0007-no-per-app-application-yaml.md>
