# Skill: Architecture Analysis

Analyze the repository's structure, design patterns, dependencies, and coupling.

---

## What to detect

### 1. Project type
Identify what kind of project this is by looking for these files:

| File found | Project type |
|------------|-------------|
| `package.json` + `tsconfig.json` | TypeScript / Node.js |
| `package.json` + React in deps | React app |
| `next.config.js` | Next.js |
| `requirements.txt` or `pyproject.toml` | Python |
| `pom.xml` or `build.gradle` | Java / Kotlin |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `Gemfile` | Ruby / Rails |
| `composer.json` | PHP |

### 2. Directory structure & layering
- List top-level directories (excluding `node_modules`, `.git`, `dist`, `build`)
- Identify if there is a clear layered structure:
  - Good patterns: `src/`, `lib/`, `api/`, `services/`, `models/`, `controllers/`, `middleware/`, `utils/`
  - Flag if everything is flat at the root
- Detect monorepo: look for `packages/`, `apps/`, `pnpm-workspace.yaml`, `lerna.json`, `turbo.json`

### 3. Dependency analysis
**For Node.js — read `package.json`:**
- Count direct dependencies vs devDependencies
- Flag if direct deps > 50 (bloat risk)
- Flag exact version pins (e.g. `"lodash": "4.17.21"` vs `"^4.17.21"`) — exact pins block security patches
- Look for obviously duplicated functionality (e.g. both `moment` and `date-fns`)
- Check if a lock file exists (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`)

**For Python — read `requirements.txt` or `pyproject.toml`:**
- Count dependencies
- Flag unpinned dependencies (no version specified)

### 4. Circular dependency signals (JS/TS)
Scan import statements for obvious cycles:
```bash
# Find all import/require statements
grep -r "from '\.\." src/ --include="*.ts" --include="*.js" | head -50
grep -r "require('\.\." src/ --include="*.js" | head -50
```
Flag if modules in the same layer import each other (e.g. service A imports service B imports service A).

### 5. Design pattern signals
Look for evidence of:
- **Dependency injection** (constructor parameters, `@Injectable`, DI containers)
- **Repository pattern** (files named `*Repository`, `*Store`, `*DAO`)
- **Factory pattern** (files named `*Factory`, `*Builder`, `*Creator`)
- **Event-driven** (EventEmitter, message queues, pub/sub)
- **God objects** — single files/classes doing too many unrelated things

---

## How to analyze

```bash
# Top-level structure
ls -la | grep "^d"

# Dependency count (Node.js)
cat package.json | python3 -c "import json,sys; p=json.load(sys.stdin); print('direct:', len(p.get('dependencies',{})), 'dev:', len(p.get('devDependencies',{})))"

# Check for lock file
ls package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null

# Find largest directories (complexity hotspots)
du -sh */ 2>/dev/null | sort -rh | head -10

# Count files per top-level directory
for dir in */; do echo "$dir: $(find $dir -type f | grep -v node_modules | wc -l) files"; done
```

---

## Findings format

- **Severity**: critical / high / medium / low
- **Category**: structure / dependencies / coupling / patterns
- **Finding**: specific observation
- **Location**: file or directory
- **Fix**: concrete recommendation

---

## Example findings

- 🟠 **High** `[dependencies]` 73 direct dependencies in `package.json`. Run `npx depcheck` to identify unused ones.
- 🟠 **High** `[coupling]` No lock file found. `npm install` may produce different results across environments.
- 🟡 **Medium** `[structure]` All 140 source files are in a flat `/src` directory. Group by feature or layer (services/, models/, routes/).
- 🟡 **Medium** `[dependencies]` Both `moment` (legacy) and `date-fns` present. Pick one and remove the other.
- 🔵 **Low** `[structure]` No monorepo tooling detected despite having 3 distinct apps. Consider Turborepo or nx.
