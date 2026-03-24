# Ruff — Expert Best Practices

## Mental model

Ruff enforces code quality rules that prevent bugs, improve readability, and modernize Python. The rules in `pyproject.toml` are not arbitrary — each has a reason. Before adding an ignore, understand why the rule exists. Most of the time, fixing the code is better than suppressing the warning.

---

## Expert rules

### Understanding the rule categories (Rafiki's ruleset)

| Prefix | Category | What it catches |
|---|---|---|
| `E` / `W` | pycodestyle | Style errors and warnings |
| `F` | Pyflakes | Unused imports, undefined names |
| `I` | isort | Import ordering |
| `N` | pep8-naming | Naming conventions |
| `UP` | pyupgrade | Outdated Python syntax |
| `B` | flake8-bugbear | Likely bugs and design issues |
| `S` | flake8-bandit | Security issues |
| `DJ` | flake8-django | Django-specific anti-patterns |
| `T20` | flake8-print | `print()` statements left in code |
| `RUF` | Ruff-native | Ruff's own rules |
| `SIM` | flake8-simplify | Unnecessary complexity |
| `TRY` | tryceratops | Exception handling anti-patterns |
| `PERF` | Perflint | Performance anti-patterns |

---

### Key rules to understand deeply

**`B006` — Mutable default argument**

```python
# Bad — the list is shared across all calls
def add_task(name, tags=[]):
    tags.append(name)

# Good
def add_task(name, tags=None):
    if tags is None:
        tags = []
```

**`B007` — Loop variable not used in loop body**

```python
# Ruff flags this — use _ for unused loop var
for i in range(10):
    do_something()  # i never used

for _ in range(10):
    do_something()
```

**`TRY300` — Move return into else block**

```python
# Ruff prefers this pattern
try:
    result = compute()
except ValueError:
    handle_error()
else:
    return result  # TRY300: return belongs in else
```

**`SIM108` — Ternary instead of if-else**

```python
# Bad
if condition:
    x = "yes"
else:
    x = "no"

# Good
x = "yes" if condition else "no"
```

**`UP` rules — Python modernization**

```python
# UP007: Use X | Y instead of Union[X, Y]
# Bad
from typing import Union
def f(x: Union[int, str]) -> None: ...

# Good (Python 3.10+)
def f(x: int | str) -> None: ...
```

**`DJ01` — Avoid `null=True` on string fields**

```python
# Bad — two empty states
name = models.CharField(max_length=100, null=True)

# Good — use blank=True only
name = models.CharField(max_length=100, blank=True, default="")
```

---

### Suppressing rules

**1. Use inline ignores with the specific rule code — never bare `# noqa`.**

```python
print(debug_info)  # noqa: T201  — intentional debug output
```

**2. Use per-file ignores in `pyproject.toml` for entire categories on specific paths.**

```toml
[tool.ruff.lint.per-file-ignores]
"*/tests/*" = ["S101"]       # assert is fine in tests
"*/migrations/*" = ["RUF"]   # already excluded globally
"manage.py" = ["T20"]        # print is fine in management commands
```

**3. Rafiki already ignores `S101` globally (assert in tests) and `RUF012` (mutable ClassVar). Don't duplicate these.**

---

### Running Ruff

```bash
uv run ruff check .           # check only
uv run ruff check . --fix     # auto-fix safe rules
uv run ruff check . --unsafe-fixes  # also fix rules that may change behavior
uv run ruff format .          # format (replaces black)
uv run ruff check --select B  # run only bugbear rules
```

**4. Run `ruff check . --fix` before committing — fixes the majority of issues automatically.**

**5. Use `--exit-non-zero-on-fix` in CI (already set in pre-commit) so auto-fixable issues still fail CI.**

---

### isort (`I` rules)

Ruff handles import sorting. Configuration in `pyproject.toml`:

```toml
[tool.ruff.lint.isort]
force-single-line = true  # one import per line
```

Import order enforced: stdlib → third-party → local. Ruff auto-fixes this with `--fix`.

---

### Integration with mypy

Ruff and mypy complement each other — they catch different issues:

- Ruff: style, anti-patterns, security, modernization
- mypy: type errors, wrong argument types, missing attributes

Run both in pre-commit. A file that passes Ruff but fails mypy (or vice versa) is not clean.

---

## Common mistakes

- Adding `# noqa` without a rule code — suppresses everything, hides future violations
- Ignoring `UP` rules because "it works" — they exist to use modern Python that's faster and cleaner
- Disabling `DJ` rules — they specifically prevent Django bugs
- Not running `--fix` before committing — manually fixing what Ruff can fix automatically

---

## References

- [Ruff rules reference](https://docs.astral.sh/ruff/rules/)
- [Ruff configuration](https://docs.astral.sh/ruff/configuration/)
- [flake8-bugbear rules explained](https://github.com/PyCQA/flake8-bugbear#opinionated-warnings)
