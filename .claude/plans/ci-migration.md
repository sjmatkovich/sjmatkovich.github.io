# Plan: Migrate Quarto site to CI-based render & publish

## Context

Today the site is rendered **manually** (`quarto render`) and the built HTML in
`docs/` is committed to `main`; GitHub Pages serves from `main` / `docs`. This
is error-prone (easy to forget to render, or to commit stale/partial output) and
couples source and build artifacts in the same branch.

We will replace this with a **GitHub Actions** pipeline that re-renders and
re-publishes the site automatically on every push to `main`, pushing built HTML
to a dedicated `gh-pages` branch via Quarto's official `quarto publish gh-pages`
flow. Source stays clean on `main`; built output lives only on `gh-pages`.

**Key enabling fact from exploration:** the site has **no executable code**
(the only R chunks, in `posts/2024-09-07_limma_without_empirical_Bayes/index.qmd`,
are `#| eval: false`) and **no R/Python/package dependencies** (no `renv.lock`,
`requirements.txt`, etc.). So CI only needs Quarto itself — no language runtimes,
no package restore, no freeze-cache concerns.

## Decisions (confirmed with user)

- **Publish method:** `gh-pages` branch via `quarto-dev/quarto-actions/publish@v2`.
- **Existing output:** stop tracking `docs/` and gitignore it; keep `_freeze/`
  tracked (harmless now, useful if computed code is added later).

## Changes

### 1. New workflow — `.github/workflows/publish.yml`

```yaml
name: Render and Publish

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write   # needed to push the built site to gh-pages

concurrency:
  group: pages-publish
  cancel-in-progress: true

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Quarto
        uses: quarto-dev/quarto-actions/setup@v2
        # Optional: pin for reproducibility, e.g.
        #   with:
        #     version: 1.5.57

      - name: Render and publish to gh-pages
        uses: quarto-dev/quarto-actions/publish@v2
        with:
          target: gh-pages
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Notes:
- No `r-lib/actions/setup-r` or Python/Jupyter setup steps — not needed (no
  executable content). If computed code is added later, insert the appropriate
  setup step before the publish step; `_freeze/` is already retained.
- The `publish` action renders **and** pushes; it also writes `.nojekyll` to
  `gh-pages` automatically so GitHub doesn't run Jekyll over Quarto output
  (important for `site_libs/` and underscore-prefixed paths).
- `GITHUB_TOKEN` is auto-provided; `permissions: contents: write` lets it push
  the `gh-pages` branch.

### 2. `_quarto.yml` — revert output dir to Quarto default

Remove the `output-dir: docs` line so the project renders to the default
`_site/` (the conventional target for the `gh-pages` flow). The `docs` name was
only meaningful for "serve from main/docs", which we're abandoning.

```yaml
project:
  type: website
  # output-dir: docs   <-- delete this line (defaults to _site)
```

### 3. `.gitignore` — ignore build output

Add the new local render target so it's never committed:

```
/_site/
```

(`/.quarto/` is already ignored. `_freeze/` stays tracked deliberately.)

### 4. Stop tracking `docs/`

```bash
git rm -r --cached docs/   # untrack, then delete the working-tree copy
rm -r docs/
```

`docs/` becomes obsolete once Pages serves from `gh-pages`; leaving it tracked
would only let it drift from the live site.

### 5. Update `CLAUDE.md`

The **Deployment** and **Architecture** sections currently describe manual
render + committing `docs/`. Update them to describe:
- CI auto-renders and publishes on push to `main`.
- Built output lives on `gh-pages` (not `docs/`); output-dir is now `_site` and
  is gitignored.
- Local `quarto render` / `quarto preview` still work for previewing; no manual
  commit of build output is needed.

### 6. One-time GitHub repo setting (manual, in the GitHub UI)

After the **first** successful workflow run creates the `gh-pages` branch:
**Settings → Pages → Build and deployment → Source = "Deploy from a branch",
Branch = `gh-pages`, Folder = `/ (root)`.**

The custom domain / `sjmatkovich.github.io` URL is unaffected.

## Forward-looking: when posts use executable code (`#| eval: true` R/Python)

The base workflow above needs **only Quarto** because the site currently
executes nothing. As soon as a post runs R or Python chunks, CI must also
reproduce the compute environment (runtime **+** exact packages). Two strategies,
with `freeze: auto` as the hinge:

- **Strategy A — CI executes (full reproducibility; matches the "CI owns
  rendering" goal).** Add runtime + dependency-restore steps before the publish
  step; commit a dependency manifest. Posts whose source changed re-execute in
  CI.
- **Strategy B — Execute locally, CI publishes only.** Keep rendering changed
  posts locally, commit the refreshed `_freeze/`; `freeze` short-circuits
  execution so the **minimal Quarto-only workflow keeps working unchanged**.
  Cheaper but relies on the author re-rendering; less reproducible.

### Strategy A — steps to insert between "Set up Quarto" and the publish step

R (knitr engine) — requires a committed `renv.lock`
(`renv::init()` + `renv::snapshot()` locally; must include `knitr`/`rmarkdown`):
```yaml
      - uses: r-lib/actions/setup-r@v2
        with:
          use-public-rspm: true
      - uses: r-lib/actions/setup-renv@v2   # restore + cache from renv.lock
```
(`setup-renv`'s pak-based restore handles compiled-package system libs.
Alternative: a `DESCRIPTION` + `r-lib/actions/setup-r-dependencies@v2`.)

Python (Jupyter engine) — requires a committed `requirements.txt` that
**includes `jupyter` + `nbclient`/`nbformat`** plus analysis packages:
```yaml
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements.txt
```

### Strategy B — local-render workflow (concrete Quarto commands)

The mechanism: `freeze` stores **raw executed results** in `_freeze/`. CI's
`quarto publish gh-pages` re-renders, but with `freeze: auto` it only *executes*
a post when that post's source differs from its stored freeze. If you always
re-render locally before committing, the freeze matches the source, so CI reuses
results and **never executes code** — meaning CI needs no R/Python and no
package manifest at all. Package versions only matter on your local machine.

Per-edit loop (run locally, where R/Python + packages are installed):

```bash
# 1. Edit the post that has executable (#| eval: true) chunks, then re-execute
#    just that post to refresh its freeze cache:
quarto render posts/<YYYY-MM-DD_slug>/index.qmd
#    (source changed ⇒ freeze: auto re-executes ⇒ rewrites
#     _freeze/posts/<slug>/.../execute-results/html.json)

# 2. Stage BOTH the source and the refreshed freeze, then commit + push to main:
git add posts/<YYYY-MM-DD_slug>/ _freeze/posts/<YYYY-MM-DD_slug>/
git commit -m "Update <slug> (refresh freeze)"
git push            # CI publishes using the committed freeze; no execution
```

Forcing re-execution when the *source* didn't change (e.g. an external data file
changed, or to deliberately refresh): the surest way is to clear that post's
freeze cache and re-render —

```bash
rm -rf _freeze/posts/<YYYY-MM-DD_slug>/
quarto render posts/<YYYY-MM-DD_slug>/index.qmd
```

(`quarto preview` also writes freeze for any doc it renders, so previewing then
committing the resulting `_freeze/` changes works too.)

**The one failure mode to guard against:** if you edit a post's source but forget
to re-render locally, the committed source no longer matches its freeze, so CI
(`freeze: auto`) will *try* to execute it and **fail the publish** (no runtime in
CI). Two ways to handle:
- **Keep `freeze: auto` (recommended):** the build fails loudly, telling you to
  re-render — no silently-stale pages.
- **Set `freeze: true`** in `posts/_metadata.yml`: CI never attempts execution,
  so a forgotten local render serves *stale* output instead of failing. Quieter,
  but can drift unnoticed.

A `renv.lock` / `requirements.txt` is **optional** under Strategy B (only for
your own local reproducibility), unlike Strategy A where it is mandatory.

### Knock-on effects to apply at that time
- **Pin the Quarto version** in `setup@v2` (`with: version: <x.y.z>`) for
  reproducible renders.
- `_freeze/` stays committed (already is): under A it lets unchanged posts skip
  re-execution; under B it is the publish mechanism.
- CI build time grows (dependency restore) — mitigated by the actions' caching.
- A failed package install now fails the publish; a green local render no longer
  guarantees a green deploy unless the manifest is complete.
- The gh-pages publish + untracked-`docs/` parts of this plan are **unchanged**;
  executable code only adds setup steps and a manifest file.

## Files touched

- `.github/workflows/publish.yml` (new)
- `_quarto.yml` (1-line removal)
- `.gitignore` (add `/_site/`)
- `docs/` (untracked + deleted)
- `CLAUDE.md` (docs update)

## Verification (end-to-end)

1. Merge `CI_revisions` → `main` (or push these changes to `main`).
2. Open the repo **Actions** tab; confirm the "Render and Publish" run succeeds.
3. Confirm a `gh-pages` branch now exists containing rendered HTML + `.nojekyll`.
4. Do the one-time Pages source setting (step 6), then load
   `https://sjmatkovich.github.io/` and confirm it renders the latest content.
5. **Regression test the automation:** make a trivial content edit (e.g. a word
   in `index.qmd`), commit to `main`, push, and confirm the workflow re-runs and
   the live site reflects the change within ~1–2 minutes.
6. Confirm `docs/` no longer appears in `git status` / the tree on `main`.

## Rollback

If CI publishing misbehaves, revert by: re-adding `output-dir: docs` to
`_quarto.yml`, restoring `docs/` (`quarto render`), re-committing it to `main`,
and switching Pages source back to `main` / `docs`. The workflow file can be
deleted or disabled independently.
