# python-best-practices-skill

A picoclaw skill for Python code quality review and best practices enforcement.

## Activation

**Manual invocation only** — say `python-best-practices` or `pbp` to activate.

## Contents

- **SKILL.md** — Main skill file with review checklist, quick actions, and pattern guidance
- **references/ruff-config.toml** — Production-ready Ruff linter/formatter configuration (Python 3.12+)
- **references/security-checklist.md** — Security review patterns with ❌/✅ code examples
- **references/design-patterns.md** — Modern Python patterns (Protocol, dataclass, registry, retry, event bus)

## Covers

- Ruff linting & formatting config
- Security review (SQL injection, command injection, path traversal, XSS, secrets, pickle, passwords)
- Performance profiling workflow (cProfile → line_profiler → py-spy)
- Modern design patterns with practical guidance
- Code review checklist
- Python 3.12+ features worth using

## Source

Enhanced from [jtgsystems/PYTHON-BEST-PRACTICES-2025](https://github.com/jtgsystems/PYTHON-BEST-PRACTICES-2025)

## License

MIT
