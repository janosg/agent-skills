# Mutable god-object across phases → typestate

**Smell:** one class with a documented "call these in this order, never call X after Y"
rulebook, enforced (if at all) by runtime asserts and boolean flags.

```python
# before: one Client with connect/authenticate/send/close + a page of rules
```

```python
# after: each state is its own type; you can only reach the next via the previous
def connect(address: str) -> ConnectedClient | None: ...

class ConnectedClient:
    def authenticate(self, password: str) -> AuthenticatedClient | None: ...

class AuthenticatedClient:
    def send(self, msg: str) -> None: ...
```

For "cannot use after close", drop `close()` and expose a context manager so the resource is
released at block exit and can't be used afterward:

```python
with connect(addr) as client:      # ConnectedClient/AuthenticatedClient
    client.send("hi")
# closed here; no way to call a closed client
```

**When:** an object moves through mutually exclusive lifecycle states. Splitting them makes
illegal sequences unrepresentable instead of runtime-checked.

**Unless:** the states aren't mutually exclusive, or the object has one or two phases and a
single guard. Then the split adds classes without removing a bug.
