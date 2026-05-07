# ADR 0005: Defer hub-dr / spoke-dr work for the POC; scope to hub-dc + spoke-dc

## Status

Accepted — 2026-05-07. **Supersedes [ADR-0004](0004-hub-dr-gitops-operator-preinstall.md)** (the hub-dr OpenShift GitOps Operator pre-install no longer applies — that work is deferred).

## Context

ADR-0004 (accepted earlier 2026-05-07) called for installing OpenShift GitOps Operator on
hub-dr via OLM, ahead of any RHACM Backup/Restore drill. The corresponding lab-gitops
change (MR !93) was merged and Argo CD on hub-dr applied the Subscription. The install
**did not complete**:

- Subscription resolved (`installedCSV: openshift-gitops-operator.v1.20.3`).
- InstallPlan was approved (`Installing`).
- CSV stuck in `Pending` indefinitely.
- The new OLM-deployed operator deployment never appeared.
- The RHACM `gitopsaddon`-bundled `openshift-gitops-operator-controller-manager`
  (in `openshift-gitops-operator` namespace, label
  `apps.open-cluster-management.io/gitopsaddon: true`) remained the only operator
  deployment running.

Root cause: the addon-bundled installation already owns the operator's cluster-scoped
resources (CRDs, ClusterRoles, ValidatingWebhookConfiguration). OLM stalls when those
collide. The fix that ADR-0004 implicitly assumed — "the gitopsaddon defers to OLM
automatically when a Subscription exists" — was wrong. The deferral is actually a
**deliberate switch**: `gitopsAddon.olmSubscription.enabled: true` on the relevant
`GitOpsCluster` CR. And hub-dr is its **own** RHACM hub (its own
`multiclusterhub` 2.16.0, `local-cluster-klusterlet` AppliedManifestWork on itself), so
it has its own GitOpsCluster CR — not hub-dc's.

Making this work cleanly on hub-dr would require:
1. Patching hub-dr's own GitOpsCluster CR with `olmSubscription.enabled: true`.
2. Manually cleaning up addon-bundled cluster-scoped resources before retrying.
3. Verifying the headless `acm-openshift-gitops` CR continues to function with the
   OLM-installed operator instead of the addon-bundled one.

That's substantially more work than the brac-poc-openliberty-v5 demo objective
needs. The POC is about demonstrating Open Liberty + NGINX on OpenShift via the
canonical GitOps + Argo CD Agent pattern. **DR drills, hub failover, and regional
spoke failover are explicitly outside the POC scope.**

ADR-0001 already established that spoke-dr is platform standby with runbook-driven
DR activation. This ADR extends that posture to hub-dr: hub-dr stays as it is
(RHACM-managed passive hub with addon-bundled headless Argo CD), no further
investment in the GitOps integration there until DR work is in scope.

## Decision

**Defer all hub-dr and spoke-dr work for the POC.** The POC ships on hub-dc + spoke-dc
only. Concretely:

- **hub-dr**: stays as-is. RHACM `multiclusterhub` 2.16.0, RHACM `gitopsaddon`-bundled
  headless Argo CD, no OLM-installed OpenShift GitOps Operator. Self-managed via local
  `hub-dr-cluster-config` Argo Application. ACM Backup/Restore prerequisites are
  acknowledged as gaps to address when DR drill is in scope; not a blocker for the
  POC demo.
- **spoke-dr**: stays platform standby per ADR-0001. No application workloads in normal
  ops. App-side activation is deferred until DR drill is in scope.
- **hub-dc**: continues to be the active management hub with full OLM-installed
  OpenShift GitOps Operator v1.20.3 and the Argo CD Agent enablement (per ADR-0003).
- **spoke-dc**: continues to be the active workload cluster with the brac-poc-openliberty-v5
  workload, RHACM-bundled headless Argo CD ready for the Agent attachment.

**ADR-0004 is superseded.** The hub-dr OLM pre-install is no longer required because
hub-dr DR drills are not in POC scope. The lab-gitops Subscription manifest added by
MR !93 was reverted in MR !94 (lab-gitops main). The orphaned CSV record on hub-dr
is benign and will be GC'd by OLM (or manually cleaned if needed).

**ADR-0003 (Argo CD Agent via gitopsaddon) remains in force** but its scope narrows:
the workload `ApplicationSet` selector matches `spoke-dc` only, the GitOpsCluster CR
patch applies to spoke-dc only, no Agent provisioning targets hub-dr or spoke-dr.

## Consequences

**Accepted**

- Hub failover drill is **not validated** during the POC. If hub-dc is lost while the
  POC is live, recovery is more involved than ADR-0004 anticipated (OLM install on
  hub-dr + addon resolution + ACM Restore, not just ACM Restore). Documented as
  accepted risk for the POC.
- The brac-poc-wiki-v5-ocp wiki retains hub-dr and spoke-dr pages, but their
  "Recommended next action" framing changes from "drive toward DR readiness" to
  "deferred — see ADR-0005."
- Future DR work picks up from this ADR and the orphaned CSV state on hub-dr — not a
  clean slate.

**Avoided**

- The RHACM `gitopsaddon` vs OLM Subscription conflict on hub-dr (CSV Pending forever).
- The follow-up cleanup (cluster-scoped resource deletions, addon-bundled deployment
  removal, GitOpsCluster CR patches on hub-dr's own ACM hub) — non-trivial work that
  doesn't move the POC demo forward.
- A "DR ready in 30 minutes" claim that wouldn't survive the first real drill.

**Required follow-up** (when DR work is in scope, after the POC)

- Re-evaluate hub-dr's gitopsaddon vs OLM strategy. Two coherent paths: (a) patch
  hub-dr's GitOpsCluster CR with `olmSubscription.enabled: true` and let the addon
  hand off; (b) disable the gitopsaddon on hub-dr entirely and OLM-install fresh.
- Re-evaluate spoke-dr's posture. ADR-0001 already documents the runbook-driven
  activation path; if hot-standby ever becomes the goal, both that ADR and this one
  are revisited.
- Validate the ACM Backup/Restore drill end-to-end with the chosen hub-dr posture.

## Alternatives Considered

### Stay on ADR-0004's path: investigate the CSV-Pending state, fix in place

Rejected because the fix path involves hub-dr's own RHACM hub (separate from hub-dc's),
patching its `GitOpsCluster` CR, and managing the addon-vs-OLM trust-handoff. That's
a substantial DR-focused investment that doesn't advance the POC demo objective. The
POC's primary risk now is "the demo workload doesn't run on spoke-dc" — not "hub-dr
isn't DR-ready."

### Disable RHACM `gitopsaddon` on hub-dr entirely; rely solely on OLM-installed operator

Rejected for the same scope reason. The current state (addon-bundled operator + Argo
managed CR) is functional for hub-dr-local Argo CD reconciliation today. Tearing it
out and rebuilding via OLM is a real platform shift that belongs to the DR work, not
the POC.

### Treat the orphaned CSV as a real problem and immediately clean it up

Approved on a small scale (once owner authorizes), but not blocking the POC. The CSV
is `Pending`, no deployment ever came up — it's an inert record. OLM may GC it; if not
a single `oc delete csv` clears it.

## References

- ADR-0001 (spoke-dr platform standby): [adr/0001-spoke-dr-platform-standby.md](0001-spoke-dr-platform-standby.md) — established the runbook-driven DR posture this ADR extends to hub-dr
- ADR-0003 (Argo CD Agent install via gitopsaddon — still in force): [adr/0003-argocd-agent-via-acm-gitopsaddon.md](0003-argocd-agent-via-acm-gitopsaddon.md)
- ADR-0004 (superseded by this one): [adr/0004-hub-dr-gitops-operator-preinstall.md](0004-hub-dr-gitops-operator-preinstall.md)
- lab-gitops MR !93 (the install attempt that revealed the conflict): https://gitlab.apps.sub.comptech-lab.com/gitops/lab-gitops/-/merge_requests/93
- lab-gitops MR !94 (the revert, merged): https://gitlab.apps.sub.comptech-lab.com/gitops/lab-gitops/-/merge_requests/94
- Red Hat: [Backup and Restore Hub Clusters with RHACM](https://www.redhat.com/en/blog/backup-and-restore-hub-clusters-with-red-hat-advanced-cluster-management-for-kubernetes) — the docs that informed ADR-0004; still authoritative for when DR is in scope
- stolostron/multicloud-integrations (gitopsaddon source — `olmSubscription.enabled` is the deferral switch, not auto-detection): https://github.com/stolostron/multicloud-integrations
