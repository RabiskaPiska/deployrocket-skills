---
name: py-quality-gate
description: Python code quality and speed optimization skill — linters, formatters, type checkers, profilers, testing, security scanning. Part of the rabicli-merged-master skill system.
---

# /rabicli-python — Python Code Quality & Speed Optimization

This skill optimizes Python code quality and speed using top GitHub tools from the Oct 2026 code-quality & speed report.

## INSTALLATION (run once per machine)

```bash
pip install ruff mypy pyrefly bandit safety pip-audit
pip install py-spy memray scalene pyinstrument
pip install pytest pytest-benchmark hypothesis mutmut coverage
pip install uv # 10-100x faster than pip
```

Run the gate through `uv run` against the project's lockfile (`uv run ruff check`, `uv run ty check`), not a second copy installed with global pip.

## PHASES

### Phase 1 — Lint & format (Ruff)

Ruff replaces flake8, pylint, isort, pyupgrade, pydocstyle. ~48,000 stars, 10-100x faster.

```bash
ruff check <path>              # lint
ruff format --check <path>     # format check
ruff format <path>             # auto-format
ruff check --select E,F,W,C,I,N,UP <path>
```

### Phase 2 — Type checking (mypy / pyrefly / ty)

```bash
mypy --strict <file>
pyrefly check <file>
ty check <file>
```

### Phase 3 — Security scanning (bandit, safety, pip-audit)

```bash
bandit -r <path>
safety check
pip-audit
```

### Phase 4 — Profiling (py-spy, memray, scalene, pyinstrument)

```bash
py-spy record -o profile.svg -- python script.py
memray run -o output.bin script.py
scalene --cli script.py
pyinstrument script.py
```

### Phase 5 — Testing (pytest, Hypothesis, mutmut, pytest-benchmark)

```bash
python -m pytest <path> -q
python -m pytest --cov=<pkg> --cov-report=term-missing --cov-fail-under=80
python -m pytest --benchmark <path>
mutmut run
```

### Phase 6 — Speed optimization with uv

```bash
uv pip install -r requirements.txt
uv run python script.py
```

## EXIT CODES

| Code | Meaning |
|------|---------|
| 0 | PASS — all checks green, all tests passed |
| 1 | FAIL — real defect found |
| 2 | COULD-NOT-MEASURE — no tests collected, missing SDK, build failure |

## RULES

1. Evidence or it did not happen — exact command, exit code, counts from output.
2. Never weaken a test to get green.
3. Never pause to ask permission for testable facts.
4. Report to `C:\Users\Administrator\Desktop\reports\<timestamp>\<task-name>.md`

## EXAMPLE

```
/rabicli-python all C:\Users\Administrator\Desktop\Projects\Releases\new\rabicli\rabicli.py
```