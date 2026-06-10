# MkDocs setup

## Install

```bash
pip install mkdocs-material
```

## Run locally

```bash
cd /Users/tlindberg/bio/design_docs
mkdocs serve
```

Opens at `http://127.0.0.1:8000`. Reloads on save.

## Build static site

```bash
mkdocs build
```

Output goes to `site/`. Add `site/` to `.gitignore`.

## Notes

- Mermaid diagrams render automatically — no extra plugin needed.
- All config is in `mkdocs.yml`.
- Add new pages under `docs/` and register them in the `nav:` section of `mkdocs.yml`.
