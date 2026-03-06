# dubbo-admin-ai-wiki

Standalone MkDocs repository for the Dubbo Admin AI documentation site.

## Local development

```bash
pip install -r docs/requirements-docs.txt
mkdocs serve
```

## Build

```bash
mkdocs build --clean
```

## GitHub Pages

Update `site_url` in [mkdocs.yml](/Users/liwener/programming/dubbo-admin-ai-wiki/mkdocs.yml) by replacing `<github-username>` with the target GitHub username, then enable Pages with `Build and deployment -> GitHub Actions`.
