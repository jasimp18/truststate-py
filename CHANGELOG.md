# Changelog

## [1.1.0] — 2026-04-03

### Breaking Changes

- **`actor_id` is now required** for all writes. Previously the SDK silently fell back to `"sdk-writer"` when no actor ID was provided. This fallback has been removed.
- Any call to `check()`, `check_batch()`, or `check_with_evidence()` that does not supply an `actor_id` (per-call or via `default_actor_id` on the client) will now raise a `TrustStateError` **before** any network request is made.
- Any `actor_id` that is not registered as an active Data Source in the TrustState tenant will be rejected by the server with `403 UNKNOWN_SOURCE`.

### Why This Changed

TrustState now enforces a **registered Data Source** model. Every integration must be declared in the TrustState dashboard under **Manage → Data Sources** before it can submit writes. This gives compliance teams:

- A clear registry of every system and feed writing to the ledger
- Per-source schema and policy bindings
- The ability to pause or revoke a source without changing API keys
- Audit trails tied to a named, managed entity rather than a free-text string

### Migration Guide

**Before (v1.0.x):**
```python
# actor_id was optional — SDK fell back to "sdk-writer"
client = TrustStateClient(api_key="ts_your_api_key")
result = await client.check("KYCRecord", data)
```

**After (v1.1.0):**
```python
# 1. Register your source in the TrustState dashboard first:
#    Manage → Data Sources → New Source
#    Set Source ID to e.g. "my-service-001"

# 2. Pass it as default_actor_id (applies to all calls):
client = TrustStateClient(
    api_key="ts_your_api_key",
    default_actor_id="my-service-001",   # ← required
)
result = await client.check("KYCRecord", data)

# Or pass per-call:
result = await client.check(
    entity_type="KYCRecord",
    data=data,
    actor_id="my-service-001",
)
```

### What Happens on Error

| Scenario | Where caught | Error |
|---|---|---|
| `actor_id` not provided | SDK (client-side) | `TrustStateError` before HTTP call |
| `actor_id` not registered in dashboard | API (server-side) | `403 UNKNOWN_SOURCE` |
| Source registered but status `inactive`/`paused` | API (server-side) | `403 UNKNOWN_SOURCE` |

### Other Changes

- `default_actor_id` constructor parameter is now documented as **required** in the config reference.
- README updated with Data Sources section, updated quickstart, and updated batch example.
- Per-item `actor_id` still overrides `default_actor_id` — unchanged behaviour, now explicitly documented.

---

## [1.0.0] — 2026-03-01

Initial release.
