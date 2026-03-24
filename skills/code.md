# Skill: Code Quality Analysis

Analyze the repository for code quality, maintainability, and test coverage.

---

## What to measure

### 1. Language & size
- Identify all programming languages present (count files per extension)
- Count total files and total lines of code
- Identify the primary language (most files/lines)
- Note any unusual or unexpected languages present

### 2. Test coverage
- Find all test files (patterns: `*.test.*`, `*.spec.*`, `__tests__/`, `test_*.py`, `*_test.go`)
- Calculate: test files ÷ total source files = test ratio
- Estimate coverage level:
  - Below 10% → Critical: almost no tests
  - 10–30% → Low: significant gaps
  - 30–60% → Moderate: reasonable but improvable
  - Above 60% → Good coverage

### 3. Code complexity signals
- Files over 500 lines → flag each one
- Functions over 80 lines → flag if visible in a scan
- Deeply nested code (4+ levels of indentation) → flag files
- Count TODO / FIXME / HACK / XXX comments across all files

### 4. Code duplication signals
- Look for repeated blocks of 10+ identical or near-identical lines
- Note files with very similar names that may be duplicates
- Flag copy-paste patterns

**In the report, for each duplication signal found, include:**
- The specific files (or file pairs) where the duplication was detected
- An estimate of the duplicated line count or percentage of the file affected
- The nature of the duplication (e.g., identical utility functions, copy-pasted config blocks, near-identical classes)
- The recommended action (e.g., extract to a shared module, replace with a parameterized function, consolidate into a single source of truth)

**Severity guide for duplication findings:**
- 🔴 **Critical** — core business logic duplicated across 3+ files with diverging implementations (risk of bugs from inconsistent changes)
- 🟠 **High** — 50+ lines duplicated across 2 files, or entire files that are near-identical copies
- 🟡 **Medium** — repeated utility/helper code (10–50 lines) that could be extracted into a shared module
- 🔵 **Low** — minor copy-paste (< 10 lines) or cosmetic repetition with no logic risk

**Example findings:**
- 🟠 **High** `[duplication]` `src/utils/dateHelper.js` and `src/lib/dateUtils.js` contain ~80 identical lines of date-formatting logic. Consolidate into a single shared module.
- 🟡 **Medium** `[duplication]` Input validation block repeated in `controllers/userController.js` (lines 45–67) and `controllers/adminController.js` (lines 112–134). Extract to a shared `validateInput()` helper.
- 🔵 **Low** `[duplication]` Database connection setup copy-pasted across 4 route files. Consider a shared `db.js` initializer.

### 5. Documentation
- Is there a README.md? Is it substantive (>20 lines) or empty?
- Are there inline comments explaining *why* (not just *what*)?
- Is there an API reference or docs folder?

---

### 6. Error handling patterns

Look for anti-patterns across all languages:

```bash
# Empty catch blocks (swallowed exceptions)
grep -rn "catch.*{" --include="*.cs" --include="*.java" --include="*.ts" --include="*.js" . \
  --exclude-dir={node_modules,.git,bin,obj,target} -A1 | grep -B1 "^\s*}"

# Catching generic exceptions
grep -rn "catch (Exception\b\|catch (Error\b\|except Exception\|except:\s*$" \
  --include="*.cs" --include="*.java" --include="*.py" --include="*.ts" . \
  --exclude-dir={node_modules,.git,bin,obj,target}

# Console.log / print statements left in production code (not test files)
grep -rn "console.log\|Console.WriteLine\|System.out.println\|print(" \
  --include="*.ts" --include="*.js" --include="*.cs" --include="*.java" --include="*.py" . \
  --exclude-dir={node_modules,.git,bin,obj,target} | grep -v test | grep -v Test | grep -v spec
```

### 7. Code style and consistency

- Check for mixed indentation (tabs vs spaces) in the same file
- Check for inconsistent naming conventions (camelCase mixed with snake_case)
- Look for commented-out code blocks (dead code)

```bash
# Commented-out code (large blocks)
grep -rn "^\s*//.*=\|^\s*#.*def \|^\s*//.*function\|^\s*//.*class " \
  --include="*.cs" --include="*.java" --include="*.ts" --include="*.js" --include="*.py" . \
  --exclude-dir={node_modules,.git,bin,obj,target} | head -20
```

---

## How to analyze

Run these commands to gather data. **Scan ALL source files**, not just a subset:

```bash
# File and language counts
find . -type f | grep -v node_modules | grep -v .git | grep -v bin/ | grep -v obj/ | grep -v target/ | grep -v dist/ | grep -v build/ | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -30

# Total lines of code (all languages)
find . -type f \( -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.jsx" \
  -o -name "*.py" -o -name "*.go" -o -name "*.java" -o -name "*.kt" \
  -o -name "*.cs" -o -name "*.rb" -o -name "*.php" -o -name "*.rs" \
  -o -name "*.swift" -o -name "*.scala" -o -name "*.dart" \) \
  | grep -v node_modules | grep -v .git | grep -v bin/ | grep -v obj/ | grep -v target/ \
  | xargs wc -l 2>/dev/null | tail -1

# Test files (all languages)
find . -type f | grep -v node_modules | grep -v .git | grep -v bin/ | grep -v obj/ | \
  grep -iE "(test|spec|__tests__|\.test\.|\.spec\.|_test\.|Tests/|Test/)" | wc -l

# TODO/FIXME count (all languages)
grep -r "TODO\|FIXME\|HACK\|XXX\|WORKAROUND\|TEMP\|KLUDGE" \
  --include="*.js" --include="*.ts" --include="*.py" --include="*.cs" --include="*.java" \
  --include="*.go" --include="*.rb" --include="*.rs" --include="*.kt" \
  -l . | grep -v node_modules | grep -v bin/ | grep -v obj/ | wc -l

# Long files (over 500 lines) — ALL languages
find . -type f \( -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.py" \
  -o -name "*.cs" -o -name "*.java" -o -name "*.go" -o -name "*.rb" -o -name "*.rs" \) \
  | grep -v node_modules | grep -v bin/ | grep -v obj/ | grep -v target/ \
  | xargs wc -l 2>/dev/null | awk '$1 > 500' | sort -rn

# Very long files (over 1000 lines) — potential god classes
find . -type f \( -name "*.js" -o -name "*.ts" -o -name "*.py" -o -name "*.cs" -o -name "*.java" \) \
  | grep -v node_modules | grep -v bin/ | grep -v obj/ | grep -v target/ \
  | xargs wc -l 2>/dev/null | awk '$1 > 1000' | sort -rn
```

---

## Findings format

For each issue found, record:
- **Severity**: critical / high / medium / low
- **Category**: testing / complexity / duplication / documentation
- **Finding**: what the problem is
- **File**: specific file (if applicable)
- **Fix**: what to do about it

---

## Example findings

- 🔴 **Critical** `[testing]` Zero test files found in a 200-file codebase. Add tests before any major refactoring.
- 🟠 **High** `[complexity]` `src/payments/processor.js` is 847 lines. Split into smaller single-responsibility modules.
- 🟡 **Medium** `[testing]` Test ratio is 8% (12 test files / 150 source files). Target 30%+.
- 🔵 **Low** `[docs]` 47 TODO comments across 23 files. Consider triaging these into GitHub issues.
