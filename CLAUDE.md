# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal website and blog for Scot J Matkovich, built with [Quarto](https://quarto.org/) and hosted on GitHub Pages. Source lives on `main`; a GitHub Actions workflow re-renders the site and publishes the built HTML to the `gh-pages` branch on every push to `main`.

## Build Commands

```bash
# Render the entire site (from project root, requires Quarto CLI)
quarto render

# Preview the site locally with live reload
quarto preview

# Render a single post (faster, avoids full site re-render)
quarto render posts/2024-09-07_limma_without_empirical_Bayes/index.qmd
```

The project also has an RStudio project file (`sjmatkovich.github.io.Rproj`), so rendering can also be done from within RStudio via the Render button or `quarto` R package.

## Architecture

- **`_quarto.yml`** — Site-wide configuration: navbar, HTML theme (`yeti`), CSS file (output dir is the Quarto default `_site/`, which is gitignored)
- **`index.qmd`** — Home page using the `solana` about template
- **`posts.qmd`** — Blog listing page (grid layout, sorted newest-first, category filtering enabled)
- **`posts/`** — One subdirectory per post, named `YYYY-MM-DD_slug/`, each containing `index.qmd`
- **`posts/_metadata.yml`** — Shared post settings: `freeze: auto` (prevents re-render unless changed), `title-block-banner: true`
- **`_site/`** — Local Quarto render output (gitignored); GitHub Pages serves the `gh-pages` branch that CI publishes, not this directory
- **`.github/workflows/publish.yml`** — CI: on push to `main`, sets up Quarto and runs `quarto publish gh-pages` to render and deploy the site
- **`_freeze/`** — Quarto freeze cache for computed outputs (R code chunks); committed to repo so posts don't re-execute unless their source changes
- **`styles.css`** — Site-wide CSS overrides (currently minimal)

## Post Conventions

Each post's YAML front matter should include:
```yaml
title: ""
description: ""
author:
  name: Scot J Matkovich
  url: https://sjmatkovich.github.io/
  orcid: 0000-0002-7398-6857
date: YYYY-MM-DD
categories: [tag1, tag2, blog]
image: filename.ext
citation:
  author: SJ Matkovich
  url: https://sjmatkovich.github.io/posts/YYYY-MM-DD_slug/
draft: false
```

## Deployment

Deployment is automated by GitHub Actions (`.github/workflows/publish.yml`): every push to `main` re-renders the site and publishes the built HTML to the `gh-pages` branch via `quarto publish gh-pages`. GitHub Pages is configured to serve from `gh-pages` (root). Do **not** commit build output to `main` — just commit source changes and push.

`quarto render` / `quarto preview` are still used locally for previewing (output goes to the gitignored `_site/`).

The site currently has no executable code chunks, so CI needs only Quarto. If posts gain `#| eval: true` R/Python chunks, either (a) add the relevant runtime + dependency-restore steps to the workflow so CI executes them, or (b) keep executing locally and commit the refreshed `_freeze/` so CI publishes without executing (`freeze: auto` skips execution when source matches the freeze).
