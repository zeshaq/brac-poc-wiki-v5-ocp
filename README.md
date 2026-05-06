# brac-poc-wiki-v5-ocp

OpenShift-only project wiki for the BRAC POC.

This site tracks the OpenShift lab fleet, setup notes, current state, and
roadmap items for the four OCP clusters:

- `hub-dc`
- `hub-dr`
- `spoke-dc`
- `spoke-dr`

## Published site

Cloudflare Pages project:

<https://brac-poc-wiki-v5-ocp.pages.dev>

## Source layout

- `.github/ISSUE_TEMPLATE/` - issue forms for OpenShift operational tasks,
  decision records, and DR drills.
- `index.html` - overview and landing page.
- `topology.html` - cluster roles and placement model.
- `diagrams.html` - diagram index with clickable rows.
- `diagram-fleet-topology.html` - full-page rendered topology diagram and
  reusable source snippet.
- `gitops.html` - desired-state repository boundaries and Argo CD flow.
- `hub-dc.html`, `hub-dr.html`, `spoke-dc.html`, `spoke-dr.html` - per-cluster
  operating notes.
- `platform-services.html` - platform capability and cleanup status.
- `secrets-vault.html` - external Vault, ESO smoke wiring,
  and application secret onboarding guardrails.
- `backup-dr.html` - OADP and ACM activation drill gates.
- `service-mesh.html` - OSSM 3 state and onboarding checklist.
- `app-onboarding.html` - Java/JBoss app path on OpenShift.
- `roadmap.html` - current execution sequence and done signals.
- `operating-rules.html` - safety and session-memory rules.
- `assets/diagrams/` - static diagram assets.
- `styles.css` - responsive, static styling.
- `404.html` - static not-found page.
- `robots.txt` and `sitemap.xml` - crawler hints.
- `_headers` - Cloudflare Pages response headers.

There is no build step.

## Update flow

Edit the static files, commit to `main`, and deploy with:

```bash
wrangler pages deploy . --project-name=brac-poc-wiki-v5-ocp --branch main
```

## Tracking model

GitHub Issues and Projects in this repository track the OpenShift operations
queue. Issues are for bounded work items, risks, decisions, and DR drills.
Milestones group larger goals such as hub DR, image mirroring, ODF Regional-DR,
and app onboarding. The GitHub Project gives the board view.

The tracker is not the desired-state source of truth. Platform desired state
stays in `lab-gitops-full/`, workload desired state stays in `lab-workloads/`,
and operational session memory stays in `/home/ze/codex-opp-agent`.

Do not put secrets, kubeconfigs, kubeadmin passwords, pull secrets, PAT values,
or full Secret manifests in issues, project fields, comments, or wiki pages.
