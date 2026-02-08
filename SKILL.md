---
name: smoke-first-structured-builder
description: A phased builder with three validation gates - smoke, contract, and integration - that generates skills using structured output templates.
version: 0.1.0
license: Apache-2.0
---

# Builder Goal

Generate agent skills through a strict three-gate validation pipeline. Each gate must pass before the next phase begins. Skills are built around structured output templates that enforce consistent file organization.

# Inputs

1. idea_prompt - The core idea or task description for the skill.
2. idea_context (optional) - Additional constraints, examples, or domain-specific context.
3. target output directory `.soup/skills/<skill-name>/`

# Structural Template

Every generated skill follows this exact directory layout:

```
<skill-name>/
  SKILL.md              # Skill documentation with YAML frontmatter
  scripts/
    run.sh              # Entry point (executable, with shebang)
    test.sh             # Unified test file (smoke + contract + integration)
  references/
    examples.md         # Usage examples with expected output
```

No other directories are created. All logic lives in `scripts/`. All documentation lives in `references/`. The `SKILL.md` is the single source of truth for what the skill does.

# Three-Gate Pipeline

## Gate 1: Smoke (must pass before writing full implementation)

Write a minimal `scripts/test.sh` containing exactly ONE test case that exercises the skill's primary happy path. Then write just enough `scripts/run.sh` to make it pass.

Smoke test pattern:
```bash
echo "Gate 1: Smoke"
RESULT=$(echo "sample input" | "$SCRIPT_DIR/run.sh")
if echo "$RESULT" | grep -qF -- "expected output"; then
  pass "smoke test"
else
  fail "smoke test" "unexpected output"
fi
```

Rules:
- Must complete in under 5 seconds
- Must use only bash and python3 stdlib
- If this fails, fix `run.sh` before proceeding — never skip Gate 1

## Gate 2: Contract (validates input/output boundaries)

Append contract tests to `scripts/test.sh` that verify:
- Missing or empty input produces a nonzero exit code and stderr message
- Invalid input types are rejected with a clear error
- Output format matches the documented contract (e.g., valid JSON, expected line format)
- Exit codes follow convention: 0 for success, 1 for handled errors, 2 for usage errors

Contract test pattern:
```bash
echo "Gate 2: Contract"
assert_exit_code "no args fails" 1 "$SCRIPT_DIR/run.sh"
assert_exit_code "invalid input fails" 1 "$SCRIPT_DIR/run.sh" --bad-option
RESULT=$(echo "valid input" | "$SCRIPT_DIR/run.sh")
echo "$RESULT" | python3 -c "import sys,json; json.load(sys.stdin)" 2>/dev/null
assert_exit_code "output is valid JSON" 0 echo "$RESULT" | python3 -c "import sys,json; json.load(sys.stdin)"
```

## Gate 3: Integration (edge cases and real-world scenarios)

Append integration tests that cover:
- Large inputs (100+ lines if applicable)
- Special characters (quotes, newlines, unicode)
- Multiple invocations produce consistent results (idempotency)
- At least one realistic end-to-end scenario matching the idea prompt

# Test Harness

All tests use this shared harness at the top of `scripts/test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PASS=0; FAIL=0; TOTAL=0

pass() { ((PASS++)); ((TOTAL++)); echo "  PASS: $1"; }
fail() { ((FAIL++)); ((TOTAL++)); echo "  FAIL: $1 -- $2"; }

assert_eq() {
  local desc="$1" expected="$2" actual="$3"
  [ "$expected" = "$actual" ] && pass "$desc" || fail "$desc" "expected '$expected', got '$actual'"
}

assert_contains() {
  local desc="$1" needle="$2" haystack="$3"
  echo "$haystack" | grep -qF -- "$needle" && pass "$desc" || fail "$desc" "missing '$needle'"
}

assert_exit_code() {
  local desc="$1" code="$2"; shift 2
  set +e; "$@" >/dev/null 2>&1; local rc=$?; set -e
  [ "$code" -eq "$rc" ] && pass "$desc" || fail "$desc" "expected exit $code, got $rc"
}
```

# Generation Sequence

1. **Derive name**: Extract kebab-case name (3-50 chars) from idea_prompt.
2. **Write Gate 1 test**: Create `scripts/test.sh` with harness + one smoke test.
3. **Write run.sh**: Implement just enough to pass Gate 1. Use bash or Python stdlib only.
4. **Run Gate 1**: Execute the smoke test. Fix until it passes.
5. **Write Gate 2 tests**: Append contract tests to `scripts/test.sh`.
6. **Harden run.sh**: Add input validation, error messages, exit codes to pass Gate 2.
7. **Run Gate 2**: Execute. Fix until it passes.
8. **Write Gate 3 tests**: Append integration tests to `scripts/test.sh`.
9. **Complete run.sh**: Handle edge cases to pass Gate 3.
10. **Run Gate 3**: Execute full test suite. All gates must pass.
11. **Write SKILL.md**: Document the skill with valid YAML frontmatter.
12. **Write references/examples.md**: Include concrete usage examples with expected output.

# Zero-Dependency Rules

- Shell: use POSIX utilities (`bash`, `grep`, `sed`, `awk`, `jq`, `curl`, `cat`, `sort`, `wc`)
- Python: use only stdlib (`json`, `os`, `sys`, `pathlib`, `re`, `argparse`, `csv`, `subprocess`)
- Never assume package managers have been run
- If an external dep is unavoidable, add `requirements.txt` with justification comments

# Output Contract

- `SKILL.md` is mandatory with valid YAML frontmatter (`name`, `description`, `version`, `license`)
- `scripts/run.sh` is the single entry point (executable, with `#!/usr/bin/env bash`)
- `scripts/test.sh` contains all three gates in one file (executable)
- `references/examples.md` has at least two usage examples with expected output
- No absolute paths or `..` in any file paths
- No file exceeds 100KB

# Quality Gates Summary

| Gate | Focus | Must Pass Before |
|------|-------|-----------------|
| Smoke | Happy path works | Writing contract tests |
| Contract | I/O boundaries correct | Writing integration tests |
| Integration | Edge cases handled | Publishing skill |

All three gates must pass. The builder never publishes a skill with failing tests.
