---
name: python-best-practices
description: "Python code quality review and best practices enforcement. ONLY activate when user explicitly says 'python-best-practices'. Does NOT auto-trigger. Covers: Ruff linting config, security review, performance profiling, design patterns, anti-patterns."
customized: true
customized_at: 2026-05-23T15:40:00Z
source: https://github.com/jtgsystems/PYTHON-BEST-PRACTICES-2025
environment: picoclaw workspace-default (Python 3.12, linux amd64)
---

# Python Best Practices (2025)

**Manual invocation only** — do not activate unless user explicitly says "python-best-practices".

Covers: linting config, code review, security hardening, performance profiling, design patterns.

## When to Use

- User says "python-best-practices" or "pbp" explicitly
- Code review requests for Python projects
- Setting up linting/formatting for a new Python project
- Security audit of Python code
- Performance investigation of Python code

## Quick Actions

### Set up Ruff for a project
```bash
# Copy our reference config
cp skills/python-best-practices/references/ruff-config.toml PROJECT/pyproject.toml
# Or merge into existing pyproject.toml — see references/ruff-config.toml

# Run
ruff check . --fix
ruff format .
```

### Security review
```bash
# Run bandit
bandit -r src/ -f json 2>/dev/null | python3 -c "
import json,sys; d=json.load(sys.stdin)
issues=d.get('results',[])
print(f'Issues: {len(issues)}')
for i in issues[:10]: print(f\"  {i['issue_severity']} {i['issue_text']} @ {i['filename']}:{i['line_number']}\")
"
```

### Performance profiling
```bash
# Quick profile
python3 -m cProfile -s cumtime script.py 2>&1 | head -30

# Line-level (install: pip install line_profiler)
kernprof -l -v script.py

# Live process (install: pip install py-spy)
py-spy top --pid PID
```

## Code Review Checklist

When reviewing Python code, check these (only flag genuine issues):

### Security (critical)
- [ ] No f-string/format in SQL queries → use parameterized `?` placeholders
- [ ] No `eval()`/`exec()` on user input
- [ ] No `pickle.loads()` on untrusted data → use JSON
- [ ] No hardcoded secrets/passwords/API keys
- [ ] `subprocess` calls use `shell=False` (default) + list args
- [ ] File operations validate paths (no traversal via `../`)
- [ ] HTML output escaped (`html.escape()` or template auto-escape)
- [ ] Password hashing uses `bcrypt`/`argon2`, not `md5`/`sha1`
- [ ] `secrets` module for tokens, not `random`
- [ ] CORS/CSRF configured for web apps

### Anti-patterns (high impact)
- [ ] No mutable default args (`def f(x=[])` → `def f(x=None)`)
- [ ] No bare `except:` → catch specific exceptions
- [ ] No `from module import *`
- [ ] No global mutable state → encapsulate in class
- [ ] No deep nesting (>3 levels) → use early returns/guard clauses
- [ ] No God classes (>20 public methods) → split responsibilities
- [ ] No circular imports → restructure dependencies
- [ ] No `type()` for type checks → use `isinstance()`

### Performance (when relevant)
- [ ] List comprehensions over manual loops for simple transforms
- [ ] `defaultdict`/`Counter` over manual dict counting
- [ ] Generators for large sequences (don't materialize full lists)
- [ ] `join()` for string concatenation in loops
- [ ] Batch DB operations (`executemany` vs loop of `execute`)
- [ ] `__slots__` for classes with many instances
- [ ] Context managers for resource cleanup (`with` statements)

### Type hints (modern Python)
- [ ] Use `list[str]` not `List[str]` (Python 3.9+)
- [ ] Use `X | Y` not `Union[X, Y]` (Python 3.10+)
- [ ] Use `Protocol` for structural typing, not ABC where duck typing suffices
- [ ] Use `dataclass`/`NamedTuple` for data containers, not manual `__init__`

## Design Patterns — When to Use

Skip the textbook stuff. These are the patterns that actually matter in our codebase:

### Protocol (structural typing)
```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class HasClose(Protocol):
    def close(self) -> None: ...

# Any object with .close() satisfies this — no inheritance needed
def cleanup(resource: HasClose) -> None:
    resource.close()
```
**Use when**: multiple unrelated classes share an interface but don't share a base class.

### Dataclass + `__post_init__`
```python
from dataclasses import dataclass

@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
    
    def __post_init__(self):
        if not 1 <= self.port <= 65535:
            raise ValueError(f"Invalid port: {self.port}")
```
**Use when**: data containers that need validation on init.

### Context manager for cleanup
```python
from contextlib import contextmanager

@contextmanager
def temp_directory():
    import tempfile, shutil
    d = tempfile.mkdtemp()
    try:
        yield d
    finally:
        shutil.rmtree(d)
```
**Use when**: any resource that needs guaranteed cleanup (DB connections, temp files, locks).

### Retry decorator
```python
import functools, time

def retry(max_attempts=3, delay=1.0, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (2 ** attempt))  # exponential backoff
        return wrapper
    return decorator
```
**Use when**: calling external APIs, network services, or flaky dependencies.

### Factory for pluggable backends
```python
_REGISTRY: dict[str, type] = {}

def register(name: str):
    def decorator(cls):
        _REGISTRY[name] = cls
        return cls
    return decorator

def create(name: str, **kwargs):
    cls = _REGISTRY.get(name)
    if not cls:
        raise ValueError(f"Unknown: {name}. Available: {list(_REGISTRY)}")
    return cls(**kwargs)
```
**Use when**: plugin systems, multiple backend implementations, CLI tools with subcommands.

## Python 3.12+ Features Worth Using

- **Type aliases** (`type Point = tuple[float, float]`) — cleaner than `TypeAlias`
- **`**kwargs` typing** (`def f(**kwargs: Unpack[SomeTypedDict])`)
- **`override` decorator** — catches signature mismatches at type-check time
- **Exception groups** (`except*`) — for concurrent error handling
- **`pathlib` improvements** — `Path.walk()`, `PurePath.is_relative_to()`

## References

- `references/ruff-config.toml` — Production-ready Ruff config (copy into `pyproject.toml`)
- `references/security-checklist.md` — Detailed security review patterns with code examples
- `references/design-patterns.md` — Full pattern library with modern Python examples
