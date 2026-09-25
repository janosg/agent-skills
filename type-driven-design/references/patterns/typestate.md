# Mutable god-object across phases → typestate

**Smell:** one class with a documented "call these in this order, never call X after Y"
rulebook, enforced (if at all) by runtime asserts and boolean flags.

```python
# before: one Client with connect/authenticate/send/close + a page of rules
```

```python
# after: each state is its own type; you can only reach the next via the previous
from collections.abc import Iterator
from contextlib import contextmanager

@contextmanager
def connect(address: str) -> Iterator[ConnectedClient]: ...   # raises if it can't connect

class ConnectedClient:
    def authenticate(self, password: str) -> AuthenticatedClient: ...   # raises on failure

class AuthenticatedClient:
    def send(self, msg: str) -> None: ...

with connect(addr) as conn:
    client = conn.authenticate(password)
    client.send("hi")
# released here
```

Failure raises instead of returning `| None`, so every state stays non-optional. The context
manager replaces `close()`: release happens at block exit, and there is no `close()` to call
twice. It guarantees release, not unusability: a reference that outlives the block still
type-checks.

**When:** an object moves through mutually exclusive lifecycle states. Splitting them makes
illegal sequences unrepresentable instead of runtime-checked.

**Unless:** the states aren't mutually exclusive, or the object has one or two phases and a
single guard. Then the split adds classes without removing a bug.
