# Webhook-Async Documentation Site

This repository contains the official documentation for [`webhook-async`](https://github.com/sunilsankar/webhook-async), deployed automatically to GitHub Pages.

---

## Documentation Pages

- **[Home & Quickstart](index.md)**: Introduction, architecture, and 60-second setup.
- **[Deployment Guide](deployment.md)**: Detailed guides for Linux (systemd), Docker Compose, Kubernetes, macOS (launchd), Windows Server (NSSM), and Reverse Proxies.
- **[Configuration Reference](configuration.md)**: Complete schema for `config.toml`, hook definitions, trigger rules, and template preprocessing.
- **[API Reference](api.md)**: Complete endpoint specifications, authentication, OpenAPI 3.1 schema, SSE stream, and curl recipe book.
- **[Operations & Backups](operations.md)**: Signal handling, zero-downtime hot reloading, online SQLite hot backups (`VACUUM INTO`), logging, and disaster recovery.
- **[Security Architecture](security.md)**: Sandboxing, process group isolation, HMAC signatures, and threat model.
- **[Troubleshooting Matrix](troubleshooting.md)**: Status code lookup and resolutions.

---

## Local Preview

Because this site uses standard GitHub Pages Jekyll, you can preview it locally using Bundler:

```bash
gem install bundler
bundle init
bundle add jekyll jekyll-theme-cayman
bundle exec jekyll serve
```
Open `http://localhost:4000` in your browser.

---

## Deployment to GitHub Pages

This repository automatically builds and publishes to GitHub Pages using the GitHub Actions workflow in `.github/workflows/pages.yml` upon push to `main` or `master`.
