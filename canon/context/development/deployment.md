---
title: Deployment
description: Cloudflare Pages Direct Upload, production and preview deploys, custom domains, the API token, and edge analytics
---

# Deployment

## Overview

Owns how a built site reaches erclx.com. The `deploy` job in `.github/workflows/verify.yml` uploads the `dist/` artifact `build-verify` produced to the Direct Upload Pages project `erclx-dev`, so Cloudflare never builds. A push to `main` deploys to production, and a dispatch from any other ref deploys to a `<hash>.erclx-dev.pages.dev` preview.

The apex and `www` are custom domains on that project, and a Redirect Rule on the zone sends `www` to the apex with a 301. Cloudflare Web Analytics runs from the zone, so no beacon code lives in this repository. A deploy by hand reads `CLOUDFLARE_ACCOUNT_ID` and `CLOUDFLARE_API_TOKEN` from `.env`.

```bash
bun run build
bunx wrangler pages deploy ./dist --project-name=erclx-dev --branch=main
```

## Layout

### `public/`

Holds files served verbatim at the domain root, so a file placed here is a public URL the moment it ships.

- `resume.pdf` serves at `erclx.com/resume.pdf`, and readers reach it through the `/resume` route, whose tab carries the site's icon where the bare PDF cannot
- The résumé is an exported copy, so it goes stale silently when the source changes
- The route draws the PDF's pages from images in `src/assets/resume/`, so a sync reruns `bun run resume-pages`, which needs `pdftoppm` from `poppler-utils`
- The rendered suite fails on a PDF whose hash differs from the one that run recorded
- One résumé ships, in English, since the page is English and the closing ask holds one résumé control

### `public/_redirects`

Decides what a request with no file behind it receives, together with `src/pages/404.astro`.

- It sends `/aitk` to `/canon`, and `/projects/<name>` to `/<name>`, so a shared `/projects/canon` link reaches the route
- A bare `/projects` goes to the landing page, and so does `/projects/`, whose splat is empty
- A redirect to a name with no route lands on the 404 page after one hop, which is the intended result
- `404.astro` builds `dist/404.html`, and Pages serves it with a 404 for any other unknown path
- A build with no top-level `404.html` makes Pages assume a single-page app and answer every unknown path with the landing page and a 200

## Gotchas

- The API token carries `Account → Cloudflare Pages → Edit` alone, since a leaked global key rewrites DNS for the zone. Rotate it whenever it leaves the machine.
- A dispatch preview never retires on its own, since Direct Upload holds no git ref. Delete it under Workers & Pages, then erclx-dev, then Deployments.
- Wrangler's OAuth callback cannot reach WSL from the Windows browser, so deploy by hand with the two variables in `.env`
- A fetch from Sweden carries no analytics beacon by design, since EU visitors are excluded, and `/cdn-cgi/trace` reporting `loc=SE` tells the two apart
- A change to redirects, headers, the 404 page, or DNS needs a live preview check, since the e2e suite runs against `astro preview` and never reaches the edge
