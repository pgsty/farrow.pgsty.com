# Farrow Documentation

This directory contains the bilingual documentation site for Farrow. English
is served at `/`; Simplified Chinese is served at `/zh/`. The site uses Hugo
Extended and Oink 1.0.0.

Farrow is pre-1.0. Public pages must describe only source repositories,
Releases channels, and package paths that are actually available at the same
checkpoint; a local build or draft is never publication evidence.

## Local development

The published dependency is pinned in `go.mod`. While Oink is developed from a
sibling checkout, use the local replacement target:

```bash
make dev
make check-local
```

The release-resolved paths are:

```bash
make build
make check
```

The production target is `https://farrow.pgsty.com/` and its build is warning-
strict. Pages settings and DNS must authorize that domain before publication.

The source-controlled `wrangler.toml` defines the Pages project and direct-
upload output directory. It does not configure the Git build image: set
`HUGO_VERSION=0.165.0` in Cloudflare Pages **Settings → Environment variables**
for both Production and Preview. Wrangler `[vars]` are Pages Functions runtime
bindings, not build-time variables. Cloudflare's default Hugo can be older than
the minimum required by Oink; `HUGO_SERVER` is not a recognized version selector.

## Content policy

English and Chinese pages live beside each other as `page.md` and
`page.zh.md`. Keep them aligned, concise, and grounded in the current checkout.
Historical Piglet release and evidence records retain their original identity
and must not be presented as post-rename Farrow validation.
