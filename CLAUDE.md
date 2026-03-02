# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal website and blog for Scot J Matkovich, built with [Quarto](https://quarto.org/) and hosted on GitHub Pages. The rendered output lives in the `docs/` directory (configured as the GitHub Pages source).

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

- **`_quarto.yml`** — Site-wide configuration: output dir (`docs/`), navbar, HTML theme (`yeti`), CSS file
- **`index.qmd`** — Home page using the `solana` about template
- **`posts.qmd`** — Blog listing page (grid layout, sorted newest-first, category filtering enabled)
- **`posts/`** — One subdirectory per post, named `YYYY-MM-DD_slug/`, each containing `index.qmd`
- **`posts/_metadata.yml`** — Shared post settings: `freeze: auto` (prevents re-render unless changed), `title-block-banner: true`
- **`docs/`** — Rendered HTML output committed to the repo; this is what GitHub Pages serves
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

After rendering, commit both `docs/` and `_freeze/` changes. GitHub Pages serves from `docs/` on the `main` branch. There is no CI pipeline — rendering and committing is done manually.
