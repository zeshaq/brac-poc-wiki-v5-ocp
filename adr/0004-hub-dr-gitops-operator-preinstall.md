# ADR 0004: hub-dr requires OpenShift GitOps Operator OLM-installed in normal ops

## Status

**Superseded by [ADR-0005](0005-defer-hub-dr-spoke-dr-poc-scope.md) on 2026-05-07** (same day).

Was: Accepted — 2026-05-07.

The decision to install OpenShift GitOps Operator on hub-dr was implemented via
`lab-gitops` MR !93 and immediately revealed an OLM-vs-RHACM-`gitopsaddon` conflict
(CSV stuck in `Pending`). MR !94 reverted the change. The follow-up needed to make
the install land cleanly is non-trivial and out of scope for the POC; ADR-0005
defers all hub-dr / spoke-dr work to a future DR-focused track. The Red Hat docs
quoted in this ADR remain accurate — they're the right reference for when DR work
is in scope.

## Context

ADR-0003 (also accepted 2026-05-07) recorded the canonical Argo CD Agent install
path for this fleet via RHACM 2.16's `gitopsaddon`. Its Consequences section
implicitly assumed that ACM Backup/Restore alone covers the hub-dc → hub-dr
activation flow:

> *"hub-dr's Principal arrives via ACM Backup/Restore at activation time."*

Same-day deeper validation (Red Hat docs review explicitly looking for gaps in
prior decisions) surfaced an unambiguous Red Hat prerequisite that contradicts
that assumption:

> *"If the initial hub cluster has any other operators installed, such as
> Ansible Automation Platform, Red Hat OpenShift GitOps, or cert-manager, you
> have to install them before running the restore operation"*
> — [Backup and Restore Hub Clusters with RHACM](https://www.redhat.com/en/blog/backup-and-restore-hub-clusters-with-red-hat-advanced-cluster-management-for-kubernetes)

> *"OpenShift GitOps Operator must be installed"* on **both** active and passive
> hubs.
> — [Argo CD DR strategy using RHACM and OADP](https://www.redhat.com/en/blog/argo-cd-disaster-recovery-strategy-using-red-hat-advanced-cluster-management-and-oadp)

Verified state on hub-dr (2026-05-07):
- RHACM `multiclusterhub` v2.16.0, MultiClusterEngine 2.11.0 — present.
- **No OpenShift GitOps Operator OLM Subscription / CSV.** The only Argo CD
  presence is `acm-openshift-gitops` headless Argo CD CR (RHACM
  `gitopsaddon`-applied via `manifestwork`, label
  `apps.open-cluster-management.io/gitopsaddon: true`). No `argocd-server`,
  no Subscription/CSV path for upgrades.
- All other ADR-0003 prerequisite operators present: cert-manager 1.19.0,
  oadp 1.5.5, ESO 1.1.0.

If a hub-dc → hub-dr activation drill ran today, the RHACM Restore would
encounter a missing prerequisite (no OpenShift GitOps Operator
Subscription/CSV) and fail at the Restore controller's prerequisite check.
The recovery window would not be "Restore Argo CD config" but rather
"OperatorHub subscribe + wait for installPlan + Restore" — an extra step
during the most time-sensitive part of a drill, and one that's avoidable
by pre-installing.

## Decision

**Install OpenShift GitOps Operator on hub-dr via OLM Subscription in normal
operation, not at restore time.** Match hub-dc's channel and version
(`gitops-1.20` channel; auto-upgrade or fixed v1.20.3 per platform policy).

The operator stays **idle** in normal ops:
- No active `ArgoCD` CR managed by it. The `acm-openshift-gitops` headless
  Argo CD CR (delivered by the RHACM `gitopsaddon`) coexists; per the
  `gitopsaddon` source convention, when an OLM Subscription is present the
  addon defers operator installation to OLM rather than deploying its own.
- During a hub-dc → hub-dr activation drill, ACM Restore brings the full
  Argo CD configuration onto hub-dr. The OLM-installed operator on hub-dr
  reconciles the restored CRs immediately — no fresh OLM install on the
  critical path.

**Where the install lives:** add the Subscription to
`lab-gitops/clusters/hub-dr/kustomization.yaml` so the existing
`hub-dr-cluster-config` Argo CD Application reconciles it as part of normal
GitOps. The Subscription itself is owned by lab-gitops platform desired
state, like every other operator install on the fleet.

**Scope clarification — out of scope for this ADR:**
- spoke-dr's Argo CD operator install — handled at DR drill activation time
  per ADR-0001 (spoke-dr is platform standby; app-side activation is
  deliberate). If/when ADR-0001 is revisited for hot-standby, this ADR's
  pattern (OLM-install in normal ops) extends to spoke-dr too.
- spoke-dc's operator state — already correct via the RHACM `gitopsaddon`
  headless CR, which is what the Agent attaches to per ADR-0003.
- hub-dc — already on OLM v1.20.3, no change.

## Consequences

**Accepted**

- One additional idle operator pod (`openshift-gitops-operator-controller-manager`)
  on hub-dr in normal operation. Resource cost negligible — operator without
  managed Argo CD CRs idles at very low CPU/memory.
- Subscription requires a matching OperatorHub channel on hub-dr's CatalogSource
  (cluster-default `redhat-operators` should provide `gitops-1.20`). If the lab
  uses a private mirror or restricted networks, the channel must be reachable —
  follow-up probe required (see "Required follow-up").
- The OpenShift Platform Plus subscription entitlement noted in ADR-0003 must
  cover hub-dr too (it almost certainly does; lab subs typically span all
  clusters). Verifying.
- ACM Backup/Restore at activation time becomes a clean restore — the
  prerequisite is already satisfied. Recovery window measured in
  Restore-controller minutes, not OLM install minutes.

**Avoided**

- The OperatorHub subscribe + installPlan + CSV reconciliation on the critical
  path of a DR drill, with all the network/registry/timeout failure modes that
  introduces.
- A failed Restore at the prerequisite-check stage on first DR drill —
  documented Red Hat behavior.
- A drift between hub-dc (OLM-installed full Argo CD) and hub-dr
  (gitopsaddon-only headless Argo CD) that contradicts the documented
  active-passive hub model.

**Required follow-up**

1. **OperatorHub probe on hub-dr** — confirm `redhat-operators` CatalogSource
   exposes `openshift-gitops-operator` package and the `gitops-1.20` channel
   resolves.
2. **Subscription entitlement check** — confirm hub-dr is covered by the
   OpenShift Platform Plus entitlement that ADR-0003 also requires for spoke-dc.
3. **Platform PR on `lab-gitops`** — add `Subscription/openshift-gitops-operator`
   to `lab-gitops/clusters/hub-dr/kustomization.yaml` (or the appropriate
   per-cluster overlay path). Same shape as hub-dc's existing subscription.
4. **Verify post-install** — confirm the operator deployment lands in
   `openshift-operators` (or the operator's preferred install namespace) on
   hub-dr without a managed `ArgoCD` CR; and confirm the existing
   `acm-openshift-gitops` CR remains stable (the `gitopsaddon` should detect
   the OLM Subscription and defer).
5. **DR drill update** — when the hub-dr activation runbook is written, the
   "verify operator pre-installed" gate is documented as already satisfied.

## Alternatives Considered

- **Install OpenShift GitOps Operator only at hub-dr activation time.**
  Rejected because Red Hat docs explicitly say the operator must be
  pre-installed before Restore runs. Doing it at activation time means a
  failed first attempt or a delayed RTO.

- **Use the RHACM `gitopsaddon`-bundled operator on hub-dr (current state)
  as sufficient.** Rejected because the bundled operator is delivered as a
  raw deployment by `manifestwork`, not via OLM Subscription/CSV. Red Hat's
  Backup/Restore prerequisite specifically references "OpenShift GitOps
  Operator … installed" in the standard OLM sense; the documentation does
  not certify the addon-bundled deployment as equivalent for the prerequisite
  check. Conservative interpretation says use OLM as documented.

- **Disable the `gitopsaddon` on hub-dr entirely; rely on the OLM-installed
  operator for everything.** Rejected because the addon's
  `acm-openshift-gitops` CR is the headless instance the Agent attaches to
  per ADR-0003 — disabling the addon removes that wiring. Coexistence is
  the correct posture: addon for the Agent-attachment shape, OLM operator
  for Backup/Restore prerequisite + idle readiness.

## References

- ADR-0003 (the parent decision this ADR augments): [adr/0003-argocd-agent-via-acm-gitopsaddon.md](0003-argocd-agent-via-acm-gitopsaddon.md)
- ADR-0001 (spoke-dr platform standby — defines the boundary for this ADR's scope): [adr/0001-spoke-dr-platform-standby.md](0001-spoke-dr-platform-standby.md)
- [Backup and Restore Hub Clusters with RHACM](https://www.redhat.com/en/blog/backup-and-restore-hub-clusters-with-red-hat-advanced-cluster-management-for-kubernetes) — *"you have to install them before running the restore operation"*
- [Argo CD DR strategy using RHACM and OADP](https://www.redhat.com/en/blog/argo-cd-disaster-recovery-strategy-using-red-hat-advanced-cluster-management-and-oadp) — *"OpenShift GitOps Operator must be installed"* on both hubs
- [How to Move from Standalone RHACM to an Active/Passive Setup](https://www.redhat.com/en/blog/how-to-move-from-standalone-rhacm-to-an-active/passive-setup) — confirms the active/passive operator pre-install pattern
- [OpenShift GitOps 1.20 install guide](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.20/) — Subscription channel and CatalogSource defaults
- [RHACM 2.16 GitOps documentation (PDF)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/pdf/gitops/Red_Hat_Advanced_Cluster_Management_for_Kubernetes-2.16-GitOps-en-US.pdf) — `GitOpsCluster.spec.gitopsAddon` schema, defer-to-OLM behavior
