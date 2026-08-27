# Versioned documentation

The documentation site is published as one static site managed by Zensical's
[Mike fork](https://github.com/squidfunk/mike):

- `/` redirects to `/current/`, which is built from `main` and replaced on each
  successful Current deployment.
- `/X.Y.Z/` is an immutable snapshot built from tag `X.Y.Z` while importing
  the exact released DSPy wheel. `/X.Y/` redirects to the newest imported
  patch in that minor line.

Mike owns `versions.json`, the default redirect, minor aliases, and the
selector rendered by Material or Zensical. The official fork is pinned to an
exact commit because it is currently distributed from GitHub rather than
PyPI. `releases.json` remains the source of truth only for the one-time
historical bootstrap. Prereleases are never published as stable documentation
routes.

The one-time bootstrap and Current review happen on the generated
`versioned-docs` branch. At cutover, that exact reviewed tree is committed onto
the existing deployment repository's `master` branch. Vercel remains on
`master`; subsequent Current and release jobs update that branch through Mike.
Every version remains in one deployment branch, not a separate hosting project
or deployment.

## Production cutover and rollback

Cutover is a one-time operator action, not a permanent workflow. In a fresh
clone of `krypticmouse/dspy-docs`, first verify the reviewed
`versioned-docs` branch, then preserve and promote it:

```bash
git fetch origin master versioned-docs
test -z "$(git ls-remote --heads origin legacy-material-backup)"
git push origin origin/master:refs/heads/legacy-material-backup

git switch -C promote-versioned-docs origin/master
git config user.name github-actions
git config user.email github-actions@github.com
git read-tree --reset -u origin/versioned-docs
git commit -m "Promote versioned documentation"
test "$(git rev-parse HEAD^{tree})" = "$(git rev-parse origin/versioned-docs^{tree})"
git push origin HEAD:master
```

The backup push intentionally fails if `legacy-material-backup` already exists,
so a later cutover cannot silently replace the rollback point. To roll back,
restore its tree with another normal commit:

```bash
git fetch origin master legacy-material-backup
git switch -C rollback-material-docs origin/master
git read-tree --reset -u origin/legacy-material-backup
git commit -m "Restore Material documentation"
git push origin HEAD:master
```

Current and release publishing fail closed unless `master` already contains
Mike's `versions.json`. Merging the automation before promotion therefore
cannot silently perform the cutover.

## Historical fidelity

Versions 3.0 through 3.3 use their tagged Material for MkDocs configuration.
Their requirements were not fully pinned and referred to DSPy on the moving
`main` branch. Bootstrap replaces that dependency with the release wheel and
resolves the remaining requirements no later than the tag's commit time.

Every release tag contains `uv.lock`, but those locks describe DSPy's project
and development dependencies. The separate documentation toolchain is defined
only in `docs/requirements.txt`; MkDocs, Material, Jupyter, redirects,
mkdocstrings, and llmstxt are absent from the locks. The locks therefore cannot
reproduce the historical documentation environment.

Each snapshot contains `_meta/build.json` with:

- source tag and commit;
- renderer and renderer version;
- DSPy artifact source and SHA-256;
- the complete resolved Python package set;
- known renderer differences.

PyPI never published `dspy==3.1.1`. That snapshot is the sole exception: its
wheel is built from tag `3.1.1` using the same metadata substitutions as the
release workflow and is marked `tag-built-wheel`.

Generated HTML is expected to differ byte-for-byte from the historical
deployment because the original build environment and output were not
archived. The compatibility contract is URLs, redirects, anchors, content,
notebooks, API symbols from the matching artifact, search behavior, social
metadata, navigation, and user-facing interactions. Visual review catches
accidental regressions, but exact fonts, spacing, line wrapping, card styling,
and pixel output are not part of the contract. Mike's version selector, build
metadata, HTML minification, and omission of production source maps are
intentional additions.

## Historical corrections

Release snapshots are immutable to automated publication, not permanently
write-once storage. A retry with byte-identical output is a no-op; a retry with
different output fails before Mike can replace `/X.Y.Z/`. This prevents routine
release automation from silently changing the meaning of an existing version
URL.

If an old snapshot needs a deliberate correction, make a reviewed pull request
directly against `krypticmouse/dspy-docs`. Limit the change to the affected
version directory unless its picker metadata or aliases genuinely need to
change. The deployment repository's pull request and Git history are the audit
trail and rollback mechanism, so no separate replacement workflow is needed.

## Output size

Production builds use Mike's redirect aliases so `/X.Y/` contains redirect
HTML rather than another asset copy. They also remove source maps and minify
HTML with whitespace-sensitive elements preserved. Both renderers pass the
same static and browser parity gates after this optimization. Git deduplicates
byte-identical files in the deployment repository; no content-addressed asset
rewriter or separate snapshot store is introduced.

## Zensical parity gate

Current and future snapshots may change from `material` to `zensical` only
after the parity checks cover every row below. Record any accepted difference
in the snapshot metadata rather than allowing silent omissions.

| Capability | Material pipeline | Zensical 0.0.57 status |
| --- | --- | --- |
| Markdown, navigation, CSS and JavaScript | Native | Native or Zensical compatibility support; feature checks pass |
| Template overrides | Jinja | MiniJinja compatibility; the incompatible custom tabs partial is replaced by Zensical's built-in tabs |
| API reference | `mkdocstrings` | Same plugin and matching public symbols |
| Notebooks | `mkdocs-jupyter` | Pre-render with `nbconvert` into a temporary source tree |
| Redirects | `mkdocs-redirects` | Generate equivalent static redirects after rendering |
| Social cards and metadata | Material `social` | Generate cards and inject equivalent metadata after rendering |
| `llms.txt` | `mkdocs-llmstxt` | Generate from the same source inventory after rendering |
| Build-time statistics | MkDocs hook | Run the existing fetcher before rendering and inject `extra.stats` |
| Search | Material search | Disco; indexed routes and representative browser discoverability pass, while result order may differ |
| Version deployment | Mike branch and selector | Official Zensical Mike fork using the same branch contract |

Update this table and the parity fixtures whenever Zensical or a replacement
extension changes one of these outcomes.

The executable gate is split into two layers:

```bash
python docs/scripts/zensical_parity.py \
  --material-output site-material \
  --zensical-output site-zensical \
  --report zensical-parity.json
python docs/scripts/zensical_browser_parity.py \
  --material site-material \
  --zensical site-zensical \
  --report zensical-browser-parity.json \
  --screenshots zensical-parity-screenshots
```

Both layers use the same `build_zensical_site` implementation as the production
entry point. The static layer fails closed on route, content, API, notebook,
redirect, metadata, social-card availability, `llms.txt`, search-index, asset,
template, statistics, sitemap, navigation, and version-selector regressions.
The browser layer fails closed on Mike's picker, search discoverability, desktop
and mobile navigation, dark-mode persistence, announcements, homepage and
tutorial interactions, footer navigation, and custom scripts. Screenshot and
social-card pixel comparisons are retained as informational review artifacts.

A fresh local run passes both layers. The `Zensical docs parity` workflow runs
on affected pull requests and manual dispatches, uploads reports and
screenshots even when a gate fails, and then enforces both reports. Current and
future release workflows explicitly select `zensical`. The release workflow
builds only the new tag, gives its minor alias to the newest patch, and pushes
the updated production branch. Historical snapshots remain immutable and are
never rebuilt during or after the cutover.
