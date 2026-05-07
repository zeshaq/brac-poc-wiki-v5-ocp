# ADR 0006: POC uses traditional push-based GitOps; Argo CD Agent deferred

## Status

Accepted — 2026-05-07. **Supersedes [ADR-0003](0003-argocd-agent-via-acm-gitopsaddon.md)**
on the install path and operating mode. ADR-0001 (spoke-dr platform standby) and
ADR-0005 (defer hub-dr/spoke-dr work) remain in force unchanged.

## Context

ADR-0003 (accepted 2026-05-07 morning) picked **Argo CD Agent in managed mode via the
RHACM 2.16 `gitopsaddon`** as the multi-cluster app-delivery mechanism. Two
same-day developments overturn that choice for the POC:

### 1. Live install attempt revealed an addon/OLM conflict

`lab-gitops` MR !93 (the ADR-0004 install) added an OLM `Subscription` for OpenShift
GitOps Operator on hub-dr. The CSV stalled in `Pending` indefinitely. Diagnosis (see
ADR-0005): the addon-bundled installation already owns the operator's cluster-scoped
resources (CRDs, ClusterRoles, webhooks). The `gitopsaddon` source documents an
`olmSubscription.enabled` deferral switch, but it is a **deliberate** switch that
must be set on the relevant `GitOpsCluster` CR — not auto-detection. Plus hub-dr is
its own RHACM hub, so the `GitOpsCluster` to patch is hub-dr's own, not hub-dc's.
MR !93 was reverted via !94. ADR-0005 deferred hub-dr/spoke-dr work; this ADR
extends that scope reduction.

### 2. Red Hat's 1.20 Argo CD Agent **architecture** doc explicitly recommends
**not** using the Agent for new adoptions

From *Red Hat OpenShift GitOps 1.20 / Argo CD Agent architecture* (last updated
2026-03-26):

> *"The Argo CD Agent configuration is designed for advanced users who already
> understand Argo CD and Red Hat OpenShift GitOps concepts. If you are new to
> Argo CD, start with the traditional push-based hub-and-spoke model before
> adopting the Agent-based approach."* — §1.1

The same document calls out, in §1.8 Known Limitations:
> - *"Partial support for ApplicationSets"*
> - *"Limited App-of-Apps pattern support in managed mode"*
> - *"The principal Agent does not support high availability"*
> - (Maturity row in the comparison table) *"Is an emerging configuration; not all
>   features are currently supported."*

The lab's existing GitOps model (per `gitops.html`) is built on a Root app + AppSets
pattern (App-of-Apps style) and the workload delivery plan in ADR-0003 was a
new `workloads-agent-managed.yaml` `ApplicationSet`. Both patterns hit Red Hat's
explicitly-documented limitations *in managed mode*. Switching to Agent in the
**autonomous** mode would resolve the App-of-Apps gap but moves Application
authoring onto the spoke — contradicting the
"`lab-gitops` platform-only, apps in `lab-workloads`" boundary
(`operating-rules.html`). No mode of the Agent today is a clean fit.

### Why this matters at POC scale

The POC's primary objective is "demonstrate Open Liberty + NGINX on OpenShift via
canonical GitOps." The Agent is the *future* canonical pattern (ADR-0003's research
backing remains accurate); the *current-best-supported* canonical pattern for new
adoptions is what Red Hat's own architecture doc points to: traditional push-based
hub-and-spoke. The security trade-off the Agent is designed to mitigate (centralized
cluster credentials on the hub) is real but not material at POC scale (one workload,
one spoke; RHACM cluster-proxy already brokers hub→spoke API access).

## Decision

**Adopt traditional push-based hub-and-spoke for the POC.** Concretely:

- **Hub Argo CD on hub-dc** is the only Argo CD that authors and reconciles workload
  Applications. The full OLM-installed `openshift-gitops` instance (v1.20.3) is the
  one source of truth.
- **spoke-dc is registered as an Argo CD destination cluster** on hub-dc's Argo CD.
  The cluster registration already exists implicitly via RHACM's GitOps add-on
  (`gitopsCluster/gitops-managed`) and the cluster-proxy bridge — no new
  networking work required.
- **Hub-side ApplicationSet** is the existing `workload-config.yaml` in
  `lab-gitops/argocd/applicationsets/`. Its cluster selector is **extended** to
  match `spoke-dc` in addition to its current `name: hub-dc` scope. A small selector
  edit is the install PR — no `workloads-agent-managed.yaml` AppSet is authored.
- **Workload tree** lives where it already lives:
  `lab-workloads.git/components/apps/brac-poc-openliberty-v5/` referenced from
  `clusters/spoke-dc/kustomization.yaml`. No change.
- **No Argo CD Agent install. No Principal CR. No PKI bootstrap. No `gitopsAddon.argoCDAgent` flag.**
- **The headless `acm-openshift-gitops` Argo CD on spoke-dc (delivered by the
  RHACM `gitopsaddon`) keeps doing what it does today**: it reconciles the
  ACM-pulled `spoke-dc-cluster-config` Application (cluster-config / platform
  layer). It does **not** participate in workload delivery in the push-mode POC.

Workload delivery flow under this ADR:

```
lab-workloads.git/main
       │  (commit: workload manifests)
       ▼
hub-dc Argo CD: workload-config ApplicationSet
       │  (generator: clusters; selector: gitops-managed=true, name in [hub-dc, spoke-dc])
       ▼
hub-dc Argo CD: Application/spoke-dc-workloads
       │  (push: hub Argo writes to spoke-dc Kubernetes API
       │   via RHACM cluster-proxy + the cluster credential in
       │   openshift-gitops on hub-dc)
       ▼
spoke-dc: namespace brac-poc-openliberty-v5
       │  (Argo creates/updates: OpenLibertyApplication CRs,
       │   nginx Deployment, Services, Route, ESO objects, etc.)
       ▼
brac-poc-openliberty-v5 reconciled by Argo on hub-dc
```

## Consequences

**Accepted**

- Hub Argo on hub-dc holds the spoke-dc cluster credential (already does, via
  RHACM cluster-proxy + the `gitops-managed` `GitOpsCluster` CR's downstream
  effect). The Agent's "no centralized cluster credentials" benefit is not
  realised in the POC.
- A hub-dc Argo CD outage stops workload reconciliation on spoke-dc — same
  trade-off the architecture doc documents for traditional push-based
  hub-and-spoke. Acceptable for POC scale.
- The Agent migration is preserved as a *future* path: ADR-0003 stays in the
  ADR repo (Status: Superseded by 0006) as the recorded plan. When Argo CD Agent
  matures past "emerging configuration" and the gitopsaddon path stabilises, the
  fleet has a documented migration target.
- The existing `cluster-config-managed-pull` ApplicationSet (cluster-config delivery
  via ACM `manifestwork`) is **unchanged**. Cluster-config and workload-config use
  different mechanisms — that asymmetry is acceptable and is the same as how the
  lab operates today.

**Avoided**

- The "Partial ApplicationSet support" risk on the workload AppSet (the lab's
  push-mode ApplicationSet works fully because it's not under the Agent).
- The "Limited App-of-Apps in managed mode" risk for the existing Root-app pattern.
- The "Principal does not support HA" architectural ceiling (no Principal in this
  topology).
- The gitopsaddon ↔ OLM Subscription conflict that MR !93 hit on hub-dr.
- A POC that depends on a Tech Preview / "emerging configuration" feature the doc
  explicitly recommends against for new adoptions.

**Required follow-up** (the platform PR scope under this ADR)

1. **Edit `lab-gitops/argocd/applicationsets/workload-config.yaml`**: extend the
   `clusters` generator to match `spoke-dc` in addition to (or replacing) the
   current `name: hub-dc` scope. Verify the AppSet template's destination /
   namespace fields generate a `spoke-dc-workloads` Application that pushes to
   spoke-dc cleanly.
2. **Verify spoke-dc cluster registration on hub-dc Argo CD**: an Argo CD
   `Cluster` Secret (label `argocd.argoproj.io/secret-type: cluster`) should
   already exist in `openshift-gitops` on hub-dc as a side-effect of RHACM's
   GitOps cluster setup. If not, the platform PR adds it.
3. **No additional changes to `lab-workloads`**. The Option-C tree from MR !2 is
   the workload artifact; the hub Argo CD reads it directly.
4. **Smoke**: trigger the AppSet, confirm `spoke-dc-workloads` Application reaches
   `Synced/Healthy`, confirm the brac-poc-openliberty-v5 namespace on spoke-dc
   shows `OpenLibertyApplication` CRs reconciling (replacing the ADR-0003
   direct-exception state).
5. **Decommission the direct exception** on spoke-dc (BuildConfigs, ImageStreams,
   the original `oc apply`-ed manifests) once Argo's reconciliation is steady.

## Alternatives Considered

### P2 — Agent in autonomous mode

Rejected. Autonomous mode places workload Application authoring on the spoke
cluster (`.spec` lives on spoke-dc). That contradicts
`operating-rules.html`'s rule that `lab-gitops` is platform desired state and
`lab-workloads` is non-platform workload desired state — autonomous mode would
require Application yamls to live somewhere outside the canonical repos
(typically a per-spoke Argo on the spoke itself), which is a meaningful
boundary violation. The autonomous-mode advantage (App-of-Apps support) doesn't
help our specific use case (a single ApplicationSet generating a single
Application per cluster).

### P3 — Stay on ADR-0003 (Agent in managed mode via gitopsaddon)

Rejected for the reasons in Context: documented limitations against ApplicationSets
and App-of-Apps in managed mode, "emerging configuration" maturity caveat, and the
live conflict observed on hub-dr. The Agent can be reconsidered when those settle.

### Continue indefinitely on the ADR-0003 direct-exception state on spoke-dc

Rejected. The brac-poc workload runs today via `oc apply` from
`brac-poc-openliberty-v5/deploy/openshift/brac-poc-openliberty-v5.yaml` (per
ADR-0003 in this project repo's `docs/adr/`). That's a documented exception, not
a steady state. The POC objective is to demonstrate canonical GitOps, so the
workload must be reconciled by Argo CD eventually. P1 is the lowest-friction way
to get there.

### Use the existing `cluster-config-managed-pull` ACM-pull pattern for workloads too

Rejected because the user explicitly stated "openshift operation is supposed to
be fully GitOps driven, no manual oc command" and "spoke clusters use the
Argo-based agent pull model" — meaning workload delivery should be Argo's job,
not ACM's `manifestwork`. ACM-pull is correct for cluster-config (operators,
platform services) and stays. Workload delivery uses the canonical Argo CD
mechanism the architecture doc recommends.

## References

- *Red Hat OpenShift GitOps 1.20 — Argo CD Agent architecture* (PDF in this
  project repo at
  [`Red_Hat_OpenShift_GitOps-1.20-Argo_CD_Agent_architecture-en-US.pdf`](https://github.com/zeshaq/brac-poc-openliberty-v5/blob/main/Red_Hat_OpenShift_GitOps-1.20-Argo_CD_Agent_architecture-en-US.pdf),
  last updated 2026-03-26 per the doc cover) — §1.1 IMPORTANT, §1.8 Known
  Limitations, §1.5 Modes
- ADR-0003 (superseded by this one): [adr/0003-argocd-agent-via-acm-gitopsaddon.md](0003-argocd-agent-via-acm-gitopsaddon.md)
- ADR-0005 (the same-day defer-DR decision; complements this ADR): [adr/0005-defer-hub-dr-spoke-dr-poc-scope.md](0005-defer-hub-dr-spoke-dr-poc-scope.md)
- ADR-0001 (spoke-dr platform standby — unchanged): [adr/0001-spoke-dr-platform-standby.md](0001-spoke-dr-platform-standby.md)
- Project ADR-0007 (no per-app Application yaml — still in force, AppSet is the only path): <https://github.com/zeshaq/brac-poc-openliberty-v5/blob/main/docs/adr/0007-no-per-app-application-yaml.md>
- lab-gitops MR !93 (the install attempt): https://gitlab.apps.sub.comptech-lab.com/gitops/lab-gitops/-/merge_requests/93
- lab-gitops MR !94 (the revert): https://gitlab.apps.sub.comptech-lab.com/gitops/lab-gitops/-/merge_requests/94
