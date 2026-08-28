# Farrow Documentation

This directory contains the bilingual documentation site for Farrow. English
is served at `/`; Simplified Chinese is served at `/zh/`. The site uses Hugo
Extended and Oink 0.8.0.

Farrow is pre-1.0. The current checkout is the working distribution path;
there is no public source repository or Releases channel yet. Public pages
must not imply otherwise.

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

The production site is built warning-strict and published at
`https://farrow.pgsty.com/`. Pages settings and DNS must authorize that domain
before deployment.

Cloudflare Pages must define `HUGO_VERSION=0.165.0` in both the Production and
Preview build environments. Cloudflare's default Hugo can be older than the
minimum required by Oink; `HUGO_SERVER` is not a recognized version selector.

## Content policy

English and Chinese pages live beside each other as `page.md` and
`page.zh.md`. Keep them aligned, concise, and grounded in the current checkout.
Historical Piglet release and evidence records retain their original identity
and must not be presented as post-rename Farrow validation.
