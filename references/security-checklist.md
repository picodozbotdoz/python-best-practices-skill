# Security Review Patterns — Python

Concrete patterns to check during code review. Each section: ❌ bad → ✅ fixed.

## SQL Injection

```python
# ❌ BAD: f-string in query
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# ❌ BAD: string concatenation
cursor.execute("SELECT * FROM users WHERE name = '" + name + "'")

# ✅ GOOD: parameterized query
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))

# ✅ GOOD: ORM (SQLAlchemy)
session.query(User).filter(User.id == user_id).first()
```

## Command Injection

```python
# ❌ BAD: shell=True with user input
subprocess.run(f"ping {host}", shell=True)

# ❌ BAD: os.system
os.system(f"rm {path}")

# ✅ GOOD: list args, no shell
subprocess.run(["ping", "-c", "1", host], capture_output=True)

# ✅ GOOD: validate input first
import shlex
if not re.match(r'^[a-zA-Z0-9._-]+$', host):
    raise ValueError(f"Invalid hostname: {host}")
subprocess.run(["ping", "-c", "1", host])
```

## Path Traversal

```python
# ❌ BAD: direct join with user input
filepath = os.path.join(base_dir, user_filename)

# ✅ GOOD: resolve and verify containment
from pathlib import Path
filepath = (Path(base_dir) / user_filename).resolve()
if not filepath.is_relative_to(Path(base_dir).resolve()):
    raise ValueError("Path traversal detected")
```

## Pickle Deserialization

```python
# ❌ NEVER: pickle on untrusted data
data = pickle.loads(untrusted_bytes)  # arbitrary code execution

# ✅ GOOD: use JSON for data exchange
import json
data = json.loads(untrusted_bytes)

# ✅ GOOD: if pickle absolutely needed, use RestrictedUnpickler
import pickle, io
class RestrictedUnpickler(pickle.Unpickler):
    ALLOWED = {"builtins": {"dict", "list", "tuple", "set", "str", "int", "float", "bool"}}
    def find_class(self, module, name):
        if module in self.ALLOWED and name in self.ALLOWED[module]:
            return getattr(__import__(module), name)
        raise pickle.UnpicklingError(f"Forbidden: {module}.{name}")
data = RestrictedUnpickler(io.BytesIO(untrusted_bytes)).load()
```

## Secret Handling

```python
# ❌ BAD: hardcoded secrets
API_KEY = "sk-1234567890abcdef"
DB_PASSWORD = "hunter2"

# ❌ BAD: secrets in logs
logger.info(f"Connecting with password={password}")

# ✅ GOOD: environment variables
import os
API_KEY = os.environ["API_KEY"]  # KeyError if missing — fail fast

# ✅ GOOD: masked logging
logger.info(f"Connecting as {user} to {host}:{port}")
```

## XSS Prevention

```python
# ❌ BAD: unescaped user input in HTML
html = f"<div>{user_input}</div>"

# ✅ GOOD: escape
import html
safe = f"<div>{html.escape(user_input)}</div>"

# ✅ GOOD: template auto-escape (Jinja2)
# {{ user_input }} is auto-escaped by default
```

## Random for Security

```python
# ❌ BAD: predictable randomness for tokens
import random
token = ''.join(random.choices(string.ascii_letters, k=32))

# ✅ GOOD: cryptographically secure
import secrets
token = secrets.token_urlsafe(32)
session_id = secrets.token_hex(32)
```

## Password Hashing

```python
# ❌ BAD: weak hashing
import hashlib
hashed = hashlib.sha256(password.encode()).hexdigest()

# ✅ GOOD: bcrypt with work factor
import bcrypt
salt = bcrypt.gensalt(rounds=12)
hashed = bcrypt.hashpw(password.encode(), salt)

# ✅ GOOD: argon2 (preferred if available)
from argon2 import PasswordHasher
ph = PasswordHasher()
hashed = ph.hash(password)
```

## eval/exec

```python
# ❌ NEVER: eval on user input
result = eval(user_expression)

# ❌ NEVER: exec on user input
exec(user_code)

# ✅ GOOD: ast.literal_eval for safe literal parsing
import ast
result = ast.literal_eval(user_string)  # only literals: dict, list, str, int, etc.

# ✅ GOOD: for math expressions, use a safe parser
import ast, operator
OPS = {ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul}
def safe_eval(expr):
    tree = ast.parse(expr, mode='eval')
    # walk tree and evaluate only numeric operations
```

## Dependency Security

```bash
# Check for known vulnerabilities
pip install safety && safety check

# Audit code for security issues
pip install bandit && bandit -r src/

# Pin dependencies with hashes
pip install pip-tools && pip-compile --generate-hashes requirements.in
```

## Subprocess Safety

```python
# ❌ BAD: shell=True
subprocess.call("ls " + user_dir, shell=True)

# ✅ GOOD: list form
subprocess.run(["ls", user_dir], check=True, capture_output=True)

# ✅ GOOD: with timeout
subprocess.run(["ls", user_dir], check=True, timeout=30)
```

## Temporary Files

```python
# ❌ BAD: predictable temp file
tmp = "/tmp/myapp_data.json"

# ✅ GOOD: secure temp file
import tempfile
fd, path = tempfile.mkstemp(suffix=".json")
try:
    with os.fdopen(fd, 'w') as f:
        json.dump(data, f)
finally:
    os.unlink(path)

# ✅ GOOD: context manager
with tempfile.NamedTemporaryFile(mode='w', suffix=".json", delete=True) as f:
    json.dump(data, f)
    # file auto-deleted when context exits
```
