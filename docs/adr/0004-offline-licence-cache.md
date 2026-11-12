# 0004. Machine-bound licence keys with a verified offline cache

- Status: Accepted
- Date: February to March 2026 (own key system replacing a third-party licensing service)
- Deciders: Melchior Vila (CTO)

## Context

Marble (Windows) and Br'eye sell content packs and full access through licence keys. Our users are often in schools where the network is filtered or unavailable, and on personal computers that go offline for days. The software must keep working offline, but a licence must not be copied freely between machines or extended by changing the clock. The previous system also had to check the key before the software could start, which made every start slow.

## Decision

1. **Own key system** on our server (replacing a third-party licensing service): a key is activated for x **machine code**, and the server answers with the access it grants (full, base, web packs) and a `valid_until` date.
2. **Local cache** of the granted access, signed with a **salted SHA-256 hash** over the check date, `valid_until`, the machine code and the access fields.
3. On start, the cache is **rejected** if the hash does not match (tampering or copy to another machine), if the last check is in the future (clock moved back) or if `valid_until` has passed. A rejected cache falls back to the free/demo access, never to a crash.
4. **Start reads only the local cache**, so it is fast and needs no network. The online key check is no longer on the start-up path and runs whenever it suits us: when online, keys are re-checked and the cache refreshed; access is also synced to the server so rights stay correct across devices.
5. **Content downloaded from the web** (teachers' quizzes and grids) carries its own hash over its title and access IDs, checked before it is opened, so access rights cannot be edited locally.

## Alternatives considered

- Third-party licensing service: used before 03/2026; removed to control the data, the offline behaviour and the cost.
- Online check at every start: simplest, but it slows every start and is unusable on school networks.

## Consequences

- Faster start: no key check or network round-trip before the software is usable.
- Easier to optimize: since the check is decoupled from start-up, we can run it whenever we want (in the background, when connected, at a quiet moment).

## Validation

- Manual tests: offline start, clock moved back, cache file edited, cache copied to another machine, several keys on one machine.
- Rejected caches are logged with a category code in the error log, which is synced to the server, to spot problems in the field.

## References

- `Source/APIs/Marble/licensing.py`, `Source/FilesHandling/license_cache_manager.py`
- History: "feat(start): keys works for 30 days now, faster start !" (2025-06-25), "feat(web): Marble get the access from the web" (2026-03-01), "feat(keys): our own marble keys" (2026-03-02), "feat(key): now check / works on offline" (2026-03-02), third-party licensing service removed (2026-03-04), "feat(MarbleKeys): we now have multiple keys" (2026-03-18), "feat(hash): grid now have Hash to safe their access" (2026-04-01), "feat(SyncKey): now checks keys online when marble's connected" (2026-04-09)
