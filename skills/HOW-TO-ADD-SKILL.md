# How to Add a Custom Skill

Adding a new analysis skill takes 5 minutes. No coding required.

---

## Template

Copy this structure into a new file at `skills/your-skill-name.md`:

```markdown
# Skill: [Your Skill Name]

[One sentence: what does this skill analyze?]

---

## What to check

### 1. [First thing to look for]
[Plain English description of what to look for and why it matters]

Commands to run:
```bash
# example commands
```

### 2. [Second thing to look for]
...

---

## How to analyze

```bash
# The main commands to gather data
```

---

## Findings format

- **Severity**: critical / high / medium / low
- **Category**: [your categories]
- **Finding**: what you found
- **File**: where
- **Fix**: what to do

---

## Example findings

- 🔴 **Critical** `[category]` ...
- 🟡 **Medium** `[category]` ...
```

## Then register it in ANALYZE.md

Open `ANALYZE.md` and add your skill to the "Load your skills" section:

```markdown
### Step 2 — Load your skills
- `skills/code.md`
- `skills/architecture.md`
- `skills/security.md`
- `skills/devops.md`
- `skills/your-skill-name.md`   ← add this line
```

That's it. Next time you run an analysis, Claude Code will use your skill.

---

## Skill ideas to build next

| Skill | What it would check |
|-------|-------------------|
| `accessibility.md` | Missing alt tags, ARIA roles, color contrast issues in HTML/JSX |
| `performance.md` | N+1 query patterns, missing indexes, large bundle signals |
| `api-design.md` | REST conventions, versioning, error response consistency |
| `logging.md` | Structured logging, sensitive data in logs, missing error logging |
| `database.md` | Migration files, ORM usage, raw SQL patterns, connection pooling |
| `i18n.md` | Hardcoded strings, missing translation keys, locale file structure |
| `documentation.md` | JSDoc/docstring coverage, stale comments, README quality |
