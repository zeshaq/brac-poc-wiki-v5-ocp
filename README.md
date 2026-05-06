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

- `index.html` - single-page OpenShift operations wiki.
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

