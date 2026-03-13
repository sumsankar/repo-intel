# Contributing to repo-intel

Thank you for your interest in contributing! This guide covers how to add new skills, run analysis, and open pull requests.

## Adding a New Skill

1. Read [skills/HOW-TO-ADD-SKILL.md](skills/HOW-TO-ADD-SKILL.md) for the full skill specification format.
2. Create a new file under `skills/` following the naming convention `<skill-name>.md`.
3. Register the skill in [ANALYZE.md](ANALYZE.md) under the appropriate section.
4. Add a usage example to `examples/` if the skill produces output.

## Pull Request Workflow

1. Fork the repository and create a branch from `main`:
   ```bash
   git checkout -b feat/my-new-skill
   ```
2. Make your changes. Keep each PR focused on a single skill or fix.
3. Ensure all Markdown files pass lint checks (see [CI](#ci) below).
4. Open a pull request against `main` with a clear description of what was changed and why.

## CI

The CI pipeline runs automatically on every push and PR:

- **Markdown lint** — via `markdownlint-cli2`. Fix lint errors before merging.
- **Link check** — via `markdown-link-check`. Ensure all links in `.md` files are reachable.

To run checks locally:
```bash
npm install -g markdownlint-cli2 markdown-link-check
markdownlint-cli2 "**/*.md" "#node_modules"
find . -name "*.md" -not -path "./node_modules/*" | xargs -I {} markdown-link-check {}
```

## Code of Conduct

Be respectful and constructive. We follow the [Contributor Covenant](https://www.contributor-covenant.org/) code of conduct.

## Questions?

Open a [GitHub Discussion](https://github.com/sumsankar/repo-intel/discussions) for questions or ideas before opening an issue or PR.
