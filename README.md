# Email Message ID Generator

Generates RFC 5322 compliant `Message-ID` headers of the form `<local-part@domain>`, with a local part derived from a SHA-256 digest so it is globally unique and unguessable.

## Usage

```python
from email_message_id_generator import MessageIdGenerator, generate_message_id

# One-off message id
mid = generate_message_id("mail.example.org")
assert mid.startswith("<") and mid.endswith(">")

# Reusable generator (preferred for throughput)
class FakeClock:
    def __init__(self):
        self.t = 0
    def __call__(self):
        return self.t

clock = FakeClock()
gen = MessageIdGenerator("mail.example.org", clock=clock)
first = gen.generate()
second = gen.generate()
assert first != second
```

Both exported names live in `email_message_id_generator`. `MessageIdGenerator(domain, clock=None)` exposes `.generate() -> str`. `generate_message_id(domain, clock=None) -> str` is a convenience wrapper for a single message.

## Why this exists

Mail systems rely on globally unique `Message-ID` values to thread and deduplicate messages. A common failure is to use `uuid.uuid4()` plus a domain, but UUIDs include characters that some legacy MTAs mishandle, and the local-part length can drift past RFC 5321's 64-octet limit. This library produces a 64-character lowercase-hex local part — well inside the limit — by hashing `domain | pid | tick | counter`.

The trade-off: uniqueness is probabilistic in the hash, not information-theoretic. In practice the counter plus monotonic clock makes collisions astronomically unlikely; for environments that require ordered or semantic ids (e.g. `<YYYYMMDD.thread@domain>`), use something else.

## The awkward edge you will hit

The clock must be monotonic non-decreasing. If you supply a clock that moves backwards (e.g. a misconfigured NTP source feeding `time.time`), `generate()` raises `ClockMovedBackwards` rather than silently risking a duplicate. Catch it at the call site and pick a policy — reset, retry, or fail loud. If you pass `clock=None`, the default uses `time.monotonic_ns`, which never goes backwards.

Two generators at the same tick with the same domain and counter value will produce identical ids. The digest inputs differ only by `domain | pid | tick | counter`; if all four match, the id matches. This is why high-throughput callers should keep one generator instance alive so the counter advances, rather than instantiating per message.
