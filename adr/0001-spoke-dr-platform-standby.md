# ADR 0001: spoke-dr is platform standby, not app-ready hot standby

## Status

Accepted — 2026-05-07

## Context

The four-cluster OCP fleet (`hub-dc`, `hub-dr`, `spoke-dc`, `spoke-dr`) was
provisioned with deliberately asymmetric roles: hub-dc is the active management
hub, hub-dr is a passive restore hub, spoke-dc is the active workload cluster,
and spoke-dr is the regional standby for spoke-dc. Up to this decision, the
recorded operating model left the **standby semantics for spoke-dr explicitly
open** — see the prior wording in `topology.html` ("Define whether standby
becomes app-ready hot mirror"), `spoke-dr.html` ("the recorded operating model
does not yet treat it as a full hot mirror"), and `roadmap.html` row "spoke-dr
semantics — Choose platform standby or app-ready hot standby."

The choice has real cost downstream:

1. **Cross-cluster session/state replication.** Hot standby implies the
   workload's stateful components are replicated across spokes (database
   streaming replication, Redis cross-cluster, application-level distributed
   cache). For non-trivial workloads this is operationally heavy and a frequent
   source of drift.
2. **Image promotion path.** Hot standby implies images are pulled and warm on
   both spokes simultaneously, which doubles registry traffic and cache
   pressure unless explicitly mirrored.
3. **GitOps surface.** Hot standby implies the workload `ApplicationSet`
   matches both spokes, which means `clusters/spoke-dr/kustomization.yaml` in
   `lab-workloads` mirrors `clusters/spoke-dc/kustomization.yaml` continuously.
   Per-spoke variation (replica counts, environment labels) creates a coupling
   that the platform-standby model avoids.
4. **External traffic routing.** Hot standby makes sense paired with a
   global LB / GTM / latency-aware DNS that fails over automatically. Without
   that, the second cluster runs idle pods that consume capacity without
   carrying user traffic.
5. **Recovery ergonomics vs operational simplicity.** Hot standby gives the
   shortest possible recovery time (seconds-to-minutes via LB health-check
   failover). Platform standby gives a longer recovery time (minutes to ~hour
   for runbook-driven activation) in exchange for *one* GitOps source of truth
   and *one* live workload to debug at any time.

The lab is a POC. The first workload onboarding (`brac-poc-openliberty-v5`) is
demo-grade and stateless — its DR posture should not drive a cluster-wide hot-
standby commitment.

## Decision

**spoke-dr is platform standby.**

- Operators, OSSM 3, ESO + Vault wiring, OADP backup, storage, and namespace-
  level platform configuration are kept Ready on spoke-dr. The
  `spoke-dr-cluster-config` Application is `Synced/Healthy`.
- **No application workloads run on spoke-dr in normal operation.** The
  workload `ApplicationSet` selector matches `spoke-dc` only.
  `lab-workloads/clusters/spoke-dr/kustomization.yaml` stays at
  `resources: []`.
- App-side DR is a **deliberate, runbook-driven activation**, not an
  automatic mirror. The activation procedure is recorded in the `Backup and DR`
  page ("Spoke regional DR" section).
- The Argo CD Agent is installed on spoke-dc only in the steady state.
  spoke-dr keeps its existing ACM-pull-mode `cluster-config` reconciliation;
  an Agent is added at activation time if needed.

## Consequences

**Accepted**

- Recovery time during a regional spoke-dc loss is non-zero; a runbook
  activation step (workload PR to `lab-workloads`, Argo sync, DNS flip) sits in
  the critical path before traffic resumes on spoke-dr. Drill capture defines
  the acceptable RTO.
- `spoke-dr.html` and `topology.html` describe spoke-dr as platform standby,
  not "a full hot mirror" — external commitments referencing this fleet must
  match.
- The first DR drill produces evidence of the activation runbook, not of
  automatic failover.

**Avoided**

- Cross-cluster session replication, dual image promotion, and split-brain
  handling are out of scope until a workload requires them.
- Per-spoke divergence in `lab-workloads/clusters/spoke-{dc,dr}/` is avoided —
  one source of truth (`spoke-dc`) until activation.
- The workload `ApplicationSet` stays simple: a single cluster generator
  matching `spoke-dc` until the activation PR opts spoke-dr in.

**Required follow-up**

- The `Regional DR` row in `roadmap.html` is reframed: drill the runbook-driven
  activation, capture image pre-pull warmth and Vault role readiness, measure
  RTO. (Done in same PR as the wiki updates that accompanied this ADR.)
- The activation procedure on `spoke-dr.html` and `backup-dr.html` is the
  authoritative runbook source. ADR-recorded but the operational steps are
  there, not duplicated here.

## Alternatives Considered

- **Active-active hot standby with external LB failover.** Both spokes run the
  workload at full capacity; a global LB (Cloudflare, F5 GTM, Route53 health
  checks) routes to the healthy spoke. Rejected on grounds of operational
  complexity for a POC — split-brain handling, session replication, and dual
  image promotion are a multi-quarter discipline, not a POC bring-up activity.
- **Cold standby (spoke-dr powered down).** Rejected because OpenShift is not
  cleanly "powered down" — operators, secrets, mesh, and CRDs need to stay
  reconciled to be ready for activation. Platform standby is the operationally
  honest version of "cold."
- **Active-passive with replicas: 0 on spoke-dr.** Rejected because the
  workload manifests would need to live in `clusters/spoke-dr/kustomization.yaml`
  in a near-but-not-quite-mirrored state (replicas patched to 0). The
  platform-standby model keeps that file empty, which is materially less
  surface area to maintain.

## References

- Topology: <https://brac-poc-wiki-v5-ocp.pages.dev/topology>
- spoke-dr cluster page (decision recorded inline): <https://brac-poc-wiki-v5-ocp.pages.dev/spoke-dr>
- DR activation runbook (8-step procedure): <https://brac-poc-wiki-v5-ocp.pages.dev/backup-dr> ("Spoke regional DR" section)
- Roadmap entry for Regional DR drill: <https://brac-poc-wiki-v5-ocp.pages.dev/roadmap>
- GitOps model (workload AppSet matches spoke-dc only): <https://brac-poc-wiki-v5-ocp.pages.dev/gitops>
