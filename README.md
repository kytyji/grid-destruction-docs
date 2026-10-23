# Grid Destruction — documentation

Source of the documentation site for the Grid Destruction plugin for Unreal
Engine 5: **https://kytyji.github.io/grid-destruction-docs/**

## Build locally

```
pip install -r requirements.txt
mkdocs serve          # http://127.0.0.1:8000, reloads on save
mkdocs build --strict # what CI runs: every link must resolve
```

## Publishing

`.github/workflows/docs.yml` builds every push and pull request and publishes to
the `gh-pages` branch with [mike](https://github.com/jimporter/mike):

| event | publishes |
|---|---|
| push to `main` | version `latest`, the default the site root opens |
| tag `vX.Y.Z` | version `X.Y`, which stays as it is when later versions ship |
| **Actions → docs → Run workflow** | any version by hand, e.g. `1.0` |

GitHub Pages serves the `gh-pages` branch (Settings → Pages → Deploy from a
branch → `gh-pages` / root).

## Editing rules

- Written for buyers: no internal decision numbers outside the Changelog, and no
  references to the plugin repository's internal notes.
- Internal links are relative (`setup.md#anchors-what-holds-a-wall-up`), so they
  work in every version.
