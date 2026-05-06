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

- `index.html` - overview and landing page.
- `topology.html` - cluster roles and placement model.
- `diagrams.html` - diagram index with clickable rows.
- `diagram-fleet-topology.html` - full-page rendered topology diagram and
  reusable source snippet.
- `gitops.html` - desired-state repository boundaries and Argo CD flow.
- `hub-dc.html`, `hub-dr.html`, `spoke-dc.html`, `spoke-dr.html` - per-cluster
  operating notes.
- `platform-services.html` - platform capability and cleanup status.
- `secrets-vault.html` - RKE2 Vault, ESO smoke wiring, scoped export token,
  and application secret onboarding guardrails.
- `backup-dr.html` - OADP and ACM activation drill gates.
- `service-mesh.html` - OSSM 3 state and onboarding checklist.
- `apps-kafka.html` - Java/JBoss app path and RKE2 Kafka readiness.
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
