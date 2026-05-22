# Plan: Migrate debt_collection Module from UUID + ID to UUID-as-Primary-Key

## Context

The debt_collection module currently uses dual identifiers: an auto-increment `id` primary key (implicit Django BigAutoField) and a separate explicit `uuid` field on 10 models. This plan transitions those 10 models to use `uuid` as the primary key, eliminating the redundant integer ID and simplifying the data model.

**Why:** Standardizing on UUID as primary key across the codebase improves data portability, reduces storage (no duplicate identifiers), and aligns with emerging architectural preferences for distributed systems.

**Models affected (11 total — 10 with existing uuid + LinkedAccount):**
- Primary entities: `Clients`, `Creditor`, `DebtorAccounts`, `Debtors`
- Status/detail models: `DebtorAccountStatus`, `RepresentedByAttorney`
- Legal chain: `LegalDetails`, `ComplaintInformation`, `JudgementInformation`, `GarnishmentInformation`
- Judgment: `AccountJudgment`
- Relationships: `LinkedAccount` (add uuid field as part of Phase 1.1)

**Scope:** This plan covers ONLY the debt_collection module. Contact, Company, User, and other cross-module models are explicitly OUT OF SCOPE.

---

## Migration Strategy Overview

Due to complex inter-model relationships (OneToOne cascades, self-referential FKs, GenericForeignKey polymorphism), the migration will proceed in **4 phases** rather than all at once:

### Phase 1: Prepare — Add UUID PKs alongside existing IDs
- Add new `<model>_uuid_pk` fields as UUIDField with `primary_key=True` on all 10 models
- Keep old `id` and `uuid` fields intact (no data loss)
- Populate new UUID PK fields from existing `uuid` column via data migration

### Phase 2: Migrate Relationships — Retarget all FKs to UUID PKs
- Add new UUID-based FK columns on all models that reference the 10 target models
- Copy FK data from old integer IDs to new UUID FKs via data migration
- Drop old integer FK columns

### Phase 3: Drop Legacy Identifiers — Remove old IDs and redundant UUIDs
- Drop the old `id` (auto-increment PK) from all 10 models
- Drop the old separate `uuid` field (now redundant with UUID PK)
- Rename `<model>_uuid_pk` to `id` to maintain Django's naming convention

### Phase 4: Test & Verify
- Run `python manage.py check` on updated models
- Verify all migrations apply cleanly with `python manage.py migrate_schemas`
- Test affected serializers, views, and endpoints
- Confirm GenericForeignKey references still resolve (they use object_id, not actual FKs)

---

## Detailed Breakdown

### Phase 1: Prepare — Add UUID Primary Key Fields

**1.1 Create Model Field Addition Migrations**

**Migration 0014a:** `debt_collection/migrations/0014_add_uuid_field_to_linkedaccount.py` (auto-generated)
- Add `uuid` field to LinkedAccount (it's the only model in this migration that doesn't have one):
```python
uuid = models.UUIDField(unique=True, default=uuid.uuid4, editable=False)
```

**Migration 0014b:** `debt_collection/migrations/0014b_add_uuid_pk_fields.py` (auto-generated)
- Add `<model>_uuid_pk` field to each of the 11 models:
```python
<model>_uuid_pk = models.UUIDField(primary_key=False, null=True, editable=False, unique=True)
```

**Models to update (in order — dependencies matter):**
1. Clients (no dependencies)
2. Creditor (FK to Clients)
3. DebtorAccounts (FK to Clients, Creditor)
4. AccountJudgment (OneToOne to DebtorAccounts)
5. LegalDetails (OneToOne to DebtorAccounts)
6. ComplaintInformation (OneToOne to LegalDetails)
7. JudgementInformation (OneToOne to LegalDetails)
8. GarnishmentInformation (OneToOne to LegalDetails)
9. DebtorAccountStatus (FK to DebtorAccounts)
10. RepresentedByAttorney (FK to DebtorAccounts)
11. Debtors (FK to DebtorAccounts)
12. LinkedAccount (FK to DebtorAccounts — self-referential)

**1.2 Create Data Migrations — Populate UUIDs**

File A: `debt_collection/migrations/0015_populate_linkedaccount_uuid.py` (manually written)
```python
def populate_linkedaccount_uuids(apps, schema_editor):
    # Generate UUIDs for LinkedAccount records that don't have one yet
    LinkedAccount = apps.get_model('debt_collection', 'LinkedAccount')
    for link in LinkedAccount.objects.filter(uuid__isnull=True):
        link.uuid = uuid.uuid4()
        link.save(update_fields=['uuid'])
```

File B: `debt_collection/migrations/0016_populate_uuid_pks.py` (manually written)
```python
def populate_uuids(apps, schema_editor):
    # Copy uuid values to _uuid_pk for each of the 11 models
    Clients = apps.get_model('debt_collection', 'Clients')
    for client in Clients.objects.all():
        if client.uuid:
            client.clients_uuid_pk = client.uuid
            client.save(update_fields=['clients_uuid_pk'])
    # Repeat for remaining 10 models (Creditor, DebtorAccounts, ..., LinkedAccount)
```

**Files to modify:**
- `debt_collection/models.py` — add `uuid` field to LinkedAccount; add `<model>_uuid_pk` fields to all 11 models
- Create new migration file `0014_add_uuid_field_to_linkedaccount.py` (auto)
- Create new migration file `0014b_add_uuid_pk_fields.py` (auto)
- Create new migration file `0015_populate_linkedaccount_uuid.py` (manual)
- Create new migration file `0016_populate_uuid_pks.py` (manual)

---

### Phase 2: Migrate Relationships — Update All Foreign Keys

**2.1 Add New UUID-Based FK Columns**

For each model that FKs to one of the 11 target models, add a `<target>_uuid` FK field.

**FKs to update (15 cross-model relationships, including LinkedAccount self-reference):**

| Source | Source File | Current FK | New UUID FK | Target |
|--------|-------------|-----------|------------|--------|
| ClientSettings | models.py:143 | `clients` (int FK) | `clients_uuid` | Clients |
| TransactionAllocations | models.py:227 | `clients` | `clients_uuid` | Clients |
| TransactionAllocations | models.py:228 | `debtor_accounts` (string ref) | `debtor_accounts_uuid` | DebtorAccounts |
| Creditor | models.py:259 | `clients` | `clients_uuid` | Clients |
| DebtorAccounts | models.py:350 | `clients` | `clients_uuid` | Clients |
| DebtorAccounts | models.py:351 | `creditor` | `creditor_uuid` | Creditor |
| RepresentedByAttorney | models.py:748 | `debtor_accounts` | `debtor_accounts_uuid` | DebtorAccounts |
| ClaimDetails | models.py:778 | `debtor_accounts` | `debtor_accounts_uuid` | DebtorAccounts |
| DebtorAccountStatus | models.py:??? | `debtor_accounts` | `debtor_accounts_uuid` | DebtorAccounts |
| ClientActionCodes | models.py:1095 | `clients` | `clients_uuid` | Clients |
| ClientFinanceLedger | models.py:1236 | `clients` | `clients_uuid` | Clients |
| DebtorsNextKin | models.py:1412 | `debtors` | `debtors_uuid` | Debtors |
| DebtorFinancials | models.py:1518 | `debtors` | `debtors_uuid` | Debtors |
| LinkedAccount (account) | models.py:1611 | `debtor_accounts` (ForeignKey, account) | `debtor_accounts_uuid` | DebtorAccounts |
| LinkedAccount (linked) | models.py:1613 | `debtor_accounts` (ForeignKey, linked_account) | `linked_to_debtor_accounts_uuid` | DebtorAccounts |

**OneToOne relationships (CASCADE — keep old, add new):**
- DebtorAccountSettings → DebtorAccounts (add `debtor_accounts_uuid`)
- LegalDetails → DebtorAccounts (add `debtor_accounts_uuid`)
- ComplaintInformation → LegalDetails (add `legal_details_uuid`)
- JudgementInformation → LegalDetails (add `legal_details_uuid`)
- GarnishmentInformation → LegalDetails (add `legal_details_uuid`)
- AccountJudgment → DebtorAccounts (add `debtor_accounts_uuid`)
- PreJudgmentComponent → AccountJudgment (add `account_judgment_uuid`)
- PostJudgmentInterestStep → AccountJudgment (add `account_judgment_uuid`)

**Create migration:**
File: `debt_collection/migrations/0017_add_uuid_fk_fields.py` (auto-generated)

**2.2 Data Migration — Copy FK Data from Integer to UUID**

File: `debt_collection/migrations/0018_copy_fk_data_to_uuid.py` (manual)

```python
def copy_fk_data(apps, schema_editor):
    # For each FK relationship, copy old int ID to new UUID FK
    # Example: ClientSettings.clients → clients_uuid
    ClientSettings = apps.get_model('debt_collection', 'ClientSettings')
    for setting in ClientSettings.objects.all():
        if setting.clients_id:
            Clients = apps.get_model('debt_collection', 'Clients')
            client = Clients.objects.get(id=setting.clients_id)
            setting.clients_uuid = client.clients_uuid_pk
            setting.save(update_fields=['clients_uuid'])
    # Repeat for all 15+ FK relationships
```

**Files to modify:**
- `debt_collection/models.py` — add `<target>_uuid` FK fields to all models that reference the 11 target models
- Create migration `0017_add_uuid_fk_fields.py` (auto)
- Create migration `0018_copy_fk_data_to_uuid.py` (manual)

---

### Phase 3: Drop Legacy Identifiers — Remove Old IDs and Redundant UUIDs

**3.1 Drop Old Integer PKs and Old UUID Fields**

For the 11 target models, remove:
- The implicit `id` (auto-increment PK)
- The old explicit `uuid` field (now redundant)

File: `debt_collection/migrations/0019_remove_legacy_ids_uuids.py`

```python
# Remove old id and uuid fields from all 11 models
RemoveField('Clients', 'id'),
RemoveField('Clients', 'uuid'),
RemoveField('Creditor', 'id'),
RemoveField('Creditor', 'uuid'),
# ... and so on for all 11 models including LinkedAccount
```

**3.2 Drop Old Integer FKs from Models That Reference the 11 Target Models**

File: `debt_collection/migrations/0020_remove_legacy_fks.py`

```python
# For each model that has an old integer FK, remove it
RemoveField('ClientSettings', 'clients'),
RemoveField('TransactionAllocations', 'clients'),
# ... repeat for all 15+ old FKs
```

**3.3 Rename UUID PK Fields to Standard `id`**

File: `debt_collection/migrations/0021_rename_uuid_pk_to_id.py`

```python
RenameField('Clients', 'clients_uuid_pk', 'id'),
RenameField('Creditor', 'creditor_uuid_pk', 'id'),
# ... repeat for all 11 models
```

**3.4 Rename UUID FK Fields to Match Old FK Names**

File: `debt_collection/migrations/0022_rename_uuid_fks.py`

For clarity, rename the new UUID FKs to match the old FK names where they exist:
```python
# Example: ClientSettings.clients_uuid → clients
RenameField('ClientSettings', 'clients_uuid', 'clients'),
RenameField('TransactionAllocations', 'creditor_uuid', 'creditor'),
# ... repeat for all UUID FKs
```

**Files to modify:**
- `debt_collection/models.py` — update model field definitions (remove old `id`/`uuid`, rename `_uuid_pk` to `id`, rename `_uuid` FKs to old names)
- Create migrations `0019_remove_legacy_ids_uuids.py` through `0022_rename_uuid_fks.py`

---

### Phase 4: Test & Verify

**4.1 Run Django Checks**
```bash
python manage.py check
```
Verify no errors in model definitions, FKs, or relationships.

**4.2 Test Migrations**
```bash
python manage.py migrate_schemas
```
Ensure all 4 new migrations apply cleanly to all tenant schemas (company, public, admin).

**4.3 Run Pytest Suite**
```bash
pytest debt_collection/tests/ -v
```
Verify all existing tests pass with UUID-based PKs. If any tests assume integer IDs, update them to accept UUIDs.

**4.4 Verify Serializers and Views**
- Check `debt_collection/serializers.py` — if any serializers hardcode `id` field access or assume integer IDs, update them to work with UUID strings
- Test endpoints that return or consume Clients, Creditor, DebtorAccounts, etc. via Postman or curl
- Verify UUID format in API responses (should be string, not integer)

**4.5 Test GenericForeignKey Polymorphism**
- Verify that Contact and Note models can still look up debt_collection objects via GenericForeignKey
- GenericForeignKey uses `object_id` (the pk value), so it should work if UUID becomes the new pk
- Test creating and retrieving contacts linked to a Clients or DebtorAccounts instance

**4.6 Test External Module References**
- Verify `notes.ClientNotes` FK to Clients still works
- Verify `tiger_functions.TimeTrackerLog` FK to Clients still works
- Both will need their FKs updated in Phase 2, but test the integration

**Critical edge cases to test:**
- Orphaned records (SET_NULL FKs should handle gracefully)
- Self-referential LinkedAccount (DebtorAccounts → DebtorAccounts)
- OneToOne cascade deletions (LegalDetails cascade deletion chain)
- Bulk operations and migrations on large datasets

---

## Dependencies & Constraints

**Models that CANNOT be included in this Phase 1 migration (out of scope):**
- Any model not in the 11-model list + LinkedAccount (Contact, Company, User, etc. in other modules)

**Note on LinkedAccount:** Will be migrated as part of Phase 1 (user clarification: include it for consistency across debt_collection). Currently has no `uuid` field, so Phase 1.1 will add one before populating the UUID PK.

**Ordering Constraints (must migrate in this order due to FKs):**
1. Clients (no dependencies)
2. Creditor (depends on Clients)
3. DebtorAccounts (depends on Clients, Creditor)
4. DebtorAccountStatus, RepresentedByAttorney, LegalDetails (depend on DebtorAccounts)
5. ComplaintInformation, JudgementInformation, GarnishmentInformation (depend on LegalDetails)
6. AccountJudgment (depends on DebtorAccounts)
7. Debtors (depends on DebtorAccounts)
8. All downstream models (DebtorsNextKin, DebtorFinancials, etc.)

**Data Loss Risks:**
- **None** — all migrations preserve existing data, just reorganize PKs and FKs
- Rollback is possible by reversing migrations (removing new fields first)

---

## Files to Create/Modify

**Django Model Definitions:**
- `debt_collection/models.py` — add/rename/remove fields on 10 models + all models with FKs to them

**Migration Files (11 new):**
1. `debt_collection/migrations/0014_add_uuid_field_to_linkedaccount.py` (auto-generated)
2. `debt_collection/migrations/0014b_add_uuid_pk_fields.py` (auto-generated)
3. `debt_collection/migrations/0015_populate_linkedaccount_uuid.py` (manual)
4. `debt_collection/migrations/0016_populate_uuid_pks.py` (manual)
5. `debt_collection/migrations/0017_add_uuid_fk_fields.py` (auto-generated)
6. `debt_collection/migrations/0018_copy_fk_data_to_uuid.py` (manual)
7. `debt_collection/migrations/0019_remove_legacy_ids_uuids.py` (auto-generated)
8. `debt_collection/migrations/0020_remove_legacy_fks.py` (auto-generated)
9. `debt_collection/migrations/0021_rename_uuid_pk_to_id.py` (auto-generated)
10. `debt_collection/migrations/0022_rename_uuid_fks.py` (auto-generated or manual)

**Possible Serializer Updates:**
- `debt_collection/serializers.py` — if any serializers assume integer IDs, update them to handle UUID strings

---

## Verification Strategy

| Step | Command | Expected Result | Owner |
|------|---------|-----------------|-------|
| Model syntax check | `python manage.py check` | No errors | Claude |
| Migration creation | `python manage.py makemigrations debt_collection` | No unmigrated changes | Claude |
| Migration apply (test DB) | `python manage.py migrate_schemas` | All 10 migrations apply, no errors | Claude |
| Full pytest suite | `pytest debt_collection/tests/ -v` | All tests pass with UUID PKs | Claude |
| API endpoint smoke test | curl against a Clients/DebtorAccounts endpoint | Status 200, UUID in response | Claude + User verification |
| GenericForeignKey test | Create Contact linked to Clients, retrieve it | Contact resolves correctly | Claude |
| Rollback test | Reverse one migration | Migration rolls back cleanly | User (optional) |

---

## User Clarifications & Decisions (Finalized)

✅ **LinkedAccount Migration:** INCLUDE — add uuid field and migrate to UUID-PK for consistency across debt_collection (user clarification: "Migrate LinkedAccount too")

✅ **Testing:** Pytest tests exist and WILL RUN automatically during Phase 4 verification (user clarification: "Yes, run them automatically")

✅ **External Consumers:** NONE — No external services depend on debt_collection integer IDs; safe to change format without coordination (user clarification: "No external consumers")

---

## Rollback Plan

If something goes wrong during migration:

1. **Before Phase 1 commit:** Create a database backup
2. **If Phase 1-2 fail:** Reverse migrations one at a time using `python manage.py migrate debt_collection 0013` (the pre-migration state)
3. **If Phase 3-4 fail:** Full database restore from backup, then investigate the root cause before re-attempting
4. **Post-Phase 3 rollback:** More difficult because old integer ID columns are dropped. Recommend backup restoration.

---

## Timeline & Effort Estimate

| Phase | Migrations | Manual Data Work | Testing | Estimated Time |
|-------|-----------|------------------|---------|-----------------|
| 1 (Prepare) | 4 files | 2 data migrations | Model check | 2-3 hours |
| 2 (Relationships) | 2 files | 1 data migration | FK validation | 2-3 hours |
| 3 (Drop Legacy) | 4 files | Field renames | Model syntax | 1-2 hours |
| 4 (Test) | 0 files | 0 | Pytest + API + GenericFK | 2-3 hours |
| **Total** | **10 files** | **3 data migrations** | **End-to-end** | **7-11 hours** |

---

## Next Steps

This plan is ready for implementation. When approved:

1. **Start Phase 1** — Add UUID fields and UUID PK fields to all 11 models (including LinkedAccount)
2. **Populate UUIDs** — Run data migrations to populate _uuid_pk fields
3. **Continue Phases 2-4** — Migrate relationships, drop legacy fields, and run full test suite
4. **Final validation** — Confirm all pytest tests pass, API endpoints return UUIDs, and GenericForeignKey polymorphism works

**No external coordination needed** — no external consumers depend on integer IDs.
