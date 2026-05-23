# Modern Python Design Patterns

Patterns that actually get used. Skip the textbook Gang-of-Four stuff.

## Structural Typing with Protocol

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Serializable(Protocol):
    def to_dict(self) -> dict: ...
    def from_dict(cls, data: dict) -> "Serializable": ...

# Any class with matching methods satisfies this — no inheritance needed
class Config:
    def to_dict(self) -> dict:
        return {"host": self.host, "port": self.port}
    @classmethod
    def from_dict(cls, data: dict) -> "Config":
        return cls(host=data["host"], port=data["port"])

def save(obj: Serializable, path: str):
    with open(path, "w") as f:
        json.dump(obj.to_dict(), f)

# Runtime check works too:
assert isinstance(Config(), Serializable)  # True
```

## Dataclass Patterns

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass
class ServerConfig:
    host: str = "0.0.0.0"
    port: int = 8080
    tags: list[str] = field(default_factory=list)
    _cache: dict = field(default_factory=dict, repr=False)

    def __post_init__(self):
        if not 1 <= self.port <= 65535:
            raise ValueError(f"Port out of range: {self.port}")

# Frozen for immutability
@dataclass(frozen=True)
class Point:
    x: float
    y: float

# Slots for memory efficiency (Python 3.10+)
@dataclass(slots=True)
class HighVolumeObject:
    id: int
    value: str
```

## Registry / Plugin Pattern

```python
_REGISTRY: dict[str, type] = {}

def register(name: str):
    """Decorator to register a class in the plugin registry."""
    def decorator(cls):
        _REGISTRY[name] = cls
        return cls
    return decorator

def create(name: str, **kwargs):
    cls = _REGISTRY.get(name)
    if not cls:
        raise ValueError(f"Unknown: {name}. Available: {list(_REGISTRY)}")
    return cls(**kwargs)

def list_available() -> list[str]:
    return sorted(_REGISTRY.keys())

# Usage:
@register("sqlite")
class SQLiteDatabase:
    def __init__(self, path: str): ...

@register("postgres")
class PostgresDatabase:
    def __init__(self, dsn: str): ...

db = create("sqlite", path=":memory:")
```

## Retry with Exponential Backoff

```python
import functools, time, logging

def retry(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exceptions: tuple[type[Exception], ...] = (Exception,),
    logger: logging.Logger | None = None,
):
    """Retry decorator with exponential backoff and jitter."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    delay = min(base_delay * (2 ** attempt), max_delay)
                    # Add jitter: ±25%
                    import random
                    delay *= 0.75 + random.random() * 0.5
                    if logger:
                        logger.warning(f"{func.__name__} attempt {attempt+1} failed: {e}, retry in {delay:.1f}s")
                    time.sleep(delay)
        return wrapper
    return decorator

# Usage:
@retry(max_attempts=3, exceptions=(ConnectionError, TimeoutError))
def fetch_data(url: str) -> dict:
    ...
```

## Context Manager Patterns

```python
from contextlib import contextmanager, asynccontextmanager
import tempfile, shutil

# Resource lifecycle
@contextmanager
def db_connection(dsn: str):
    conn = sqlite3.connect(dsn)
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# Temporary directory with cleanup
@contextmanager
def temp_dir(prefix: str = "work_"):
    d = tempfile.mkdtemp(prefix=prefix)
    try:
        yield d
    finally:
        shutil.rmtree(d, ignore_errors=True)

# Suppressed errors
@contextmanager
def suppress(*exceptions):
    try:
        yield
    except exceptions:
        pass

# Async version
@asynccontextmanager
async def async_pool():
    pool = await aiohttp.ClientSession()
    try:
        yield pool
    finally:
        await pool.close()
```

## Observer / Event Bus

```python
from typing import Any, Callable
from collections import defaultdict

class EventBus:
    def __init__(self):
        self._handlers: dict[str, list[Callable]] = defaultdict(list)

    def on(self, event: str, handler: Callable):
        self._handlers[event].append(handler)

    def off(self, event: str, handler: Callable):
        self._handlers[event].remove(handler)

    def emit(self, event: str, data: Any = None):
        for handler in self._handlers[event]:
            handler(data)

# Usage:
bus = EventBus()
bus.on("user.created", lambda u: send_welcome_email(u))
bus.on("user.created", lambda u: log_audit("user_created", u))
bus.emit("user.created", {"email": "alice@example.com"})
```

## Configuration Builder

```python
from dataclasses import dataclass, field

@dataclass
class AppConfig:
    db_host: str = "localhost"
    db_port: int = 5432
    db_name: str = ""
    log_level: str = "INFO"
    workers: int = 4
    features: dict[str, bool] = field(default_factory=dict)

class ConfigBuilder:
    def __init__(self):
        self._config = AppConfig()

    def db(self, host: str, port: int = 5432, name: str = "") -> "ConfigBuilder":
        self._config.db_host = host
        self._config.db_port = port
        self._config.db_name = name
        return self

    def log(self, level: str) -> "ConfigBuilder":
        self._config.log_level = level
        return self

    def workers(self, n: int) -> "ConfigBuilder":
        self._config.workers = n
        return self

    def feature(self, name: str, enabled: bool = True) -> "ConfigBuilder":
        self._config.features[name] = enabled
        return self

    def build(self) -> AppConfig:
        if not self._config.db_name:
            raise ValueError("db_name is required")
        return self._config

# Usage:
config = (ConfigBuilder()
    .db("prod-db.internal", name="myapp")
    .log("WARNING")
    .workers(8)
    .feature("dark_mode")
    .build())
```

## Sentinel / Null Object

```python
# Sentinel for distinguishing "not provided" from None
_MISSING = object()

def get_config(key: str, default=_MISSING):
    value = _store.get(key, default)
    if value is __MISSING:
        raise KeyError(f"Required config missing: {key}")
    return value

# Null object pattern
class NullLogger:
    def debug(self, *a, **kw): pass
    def info(self, *a, **kw): pass
    def warning(self, *a, **kw): pass
    def error(self, *a, **kw): pass

def process(data, logger=None):
    logger = logger or NullLogger()  # no more "if logger:" everywhere
    logger.info("Processing...")
```

## Cached Property (stdlib)

```python
from functools import cached_property

class DataProcessor:
    def __init__(self, path: str):
        self.path = path

    @cached_property
    def data(self) -> list[dict]:
        """Loaded once, then cached on instance."""
        with open(self.path) as f:
            return json.load(f)

    @cached_property
    def summary(self) -> dict:
        """Also cached — computed from cached data."""
        return {"count": len(self.data), "keys": list(self.data[0].keys())}

# Python 3.12+ also has:
# @cached_property on dataclass fields via __slots__
```
