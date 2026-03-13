# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- `.gitignore` to exclude local dev config, generated reports, and secrets
- `.github/SECURITY.md` for responsible vulnerability disclosure process
- `.github/workflows/ci.yml` — CI pipeline with Markdown lint and link checking
- `.github/dependabot.yml` — auto-updates for GitHub Actions dependencies
- `CONTRIBUTING.md` — contributor guide covering skill authoring and PR workflow
- `CHANGELOG.md` — this file

### Fixed
- Default PostgreSQL credentials in `docs/deployment-architecture.md` replaced with environment variable references

### Removed
- Stopped tracking `.claude/settings.local.json` (machine-local developer config)
- Stopped tracking `repo-intel-report.html` and `repo-intel-report.md` (generated output)
