# Title

How would you structure ingestion of polymorphic third-party payloads in Django/DRF with a service layer?

# Body

I'm refactoring a Django/DRF module that ingests skip-trace data from multiple external bureaus (Experian, Equifax, IDI, TLO). Each provider returns a completely different JSON structure. I store the raw payload in a `JSONField` for audit, and normalize parts of it into relational models (`DebtorContact`, `SkipTraceVehicle`, `SkipTraceProperty`) for querying.

I've moved from ViewSets to explicit `APIView` classes plus a service layer that owns the atomic write:

```python
# services.py
class DebtorContactService:
    @transaction.atomic
    def register_bureau_payload(self, debtor, bureau_type, raw_data):
        parsed = self._parse(bureau_type, raw_data)
        # bulk_create / bulk_update of normalized records
        return contact_record
```

The payload shape depends on a `bureau_type` request parameter. Simplified examples:

```json
// bureau_type = "experian"
{"consumer": {"names": [...], "tradelines": [...]}}

// bureau_type = "tlo"
{"Person": {"Vehicles": [...], "Addresses": [...]}}
```

## Constraints

1. Atomic writes — a payload either fully normalizes or nothing persists (raw audit record excepted).
2. Bulk write performance (`bulk_create` / `bulk_update`) — payloads can contain hundreds of records.
3. Upstream schema changes must never require a DB migration to keep ingesting.

## Question 1 — Where should payload validation live?

- **A. Per-bureau DRF serializers**, selected at runtime in the view. Standard DRF errors and OpenAPI schema, but deeply nested serializers for 4+ evolving schemas get painful.
- **B. Service-layer validation** with pydantic/dataclasses — view passes raw JSON through. Validation sits next to the parsers, but bypasses DRF conventions and error formatting.
- **C. Hybrid** — thin DRF serializer validates the envelope (`bureau_type`, debtor ref), service validates the payload body.

I'm leaning toward C. Has anyone run this split in production — does keeping two validation layers coherent become its own problem?

## Question 2 — Failure model for bulk normalization

My current plan: persist the raw payload first with a status field (`received` → `normalized` / `failed`), then run normalization all-or-nothing inside `@transaction.atomic`, so failed payloads are re-processable from stored JSON. The alternative — per-section savepoints (keep vehicles even if properties fail) — seems to make dedup state ambiguous.

Is all-or-nothing the right call here, or have you found partial persistence worth the complexity?

---

Stack: Django 5.x, DRF 3.15, PostgreSQL. Happy to share more of the actual parser/service code if useful.
