# Schema Diff: `database-schema.dbml` (actual project) vs `diagram.dbml` (final/target design)

**database-schema.dbml** = reverse-engineered from the real TypeORM entities in `src/modules/**/entities/*.entity.ts` — this is what is **actually running in the codebase today**.

**diagram.dbml** = a separate, redesigned "final merged" schema (per its own header, sourced from `finaldb.sql`) — this looks like a **target/future schema**, not what's implemented yet. Several tables in it are intentionally restructured versions of the current tables.

Use this file to see, module by module: which tables/fields exist **only in the project** (implemented but not in the target diagram), which exist **only in the diagram** (designed but not yet built), and which fields were **renamed/changed** between the two.

Labels used below:
- **EXTRA IN PROJECT** — field/table is in the running code but NOT in `diagram.dbml`
- **MISSING IN PROJECT** — field/table is in `diagram.dbml` but NOT implemented in code yet
- **RENAMED/CHANGED** — same concept, different column name, type, or constraint
- **REDESIGNED** — the table exists in both but the shape is different enough that it's effectively a different table

---

## 0. Whole-table differences

### Tables that exist ONLY in the project (`database-schema.dbml`) — not in `diagram.dbml` at all
| Table | Module | Notes |
|---|---|---|
| `locked_periods` | masters | Controls whether an accounting period is editable. No equivalent anywhere in `diagram.dbml`. |
| `treaty_lobs` | masters | Junction: treaty ↔ line of business. `diagram.dbml` replaced this whole concept with `treaty_products` (treaty ↔ product, where a product already bundles LOB+COB). |
| `treaty_lob_cobs` | masters | Second-level junction: treaty_lob ↔ class of business. Same reason as above — removed in favor of `treaty_products`. |

### Tables that exist ONLY in `diagram.dbml` — not implemented in the project
None. Every table in `diagram.dbml` has a same-named counterpart already implemented in the project (though several are reshaped — see below).

### Table counts
- `database-schema.dbml` (project, actual): **53 tables**
- `diagram.dbml` (target design): **50 tables**

---

## GROUP 1: Auth & RBAC
*(project modules: `users`, `roles`, `permissions`, `auth`, `activity-logs`)*

### `roles` — no differences
Identical field set in both files.

### `users` — CHANGED
- **MISSING IN PROJECT**: `team_id` — diagram note says it's a placeholder for future team assignment, always NULL, no FK/table exists yet anyway, so nothing to actually build.
- **EXTRA IN PROJECT**: `user_entity_type`, `user_entity_id` — polymorphic link from a user to an MGA/broker/customer entity. Diagram's note explicitly says these were **removed** because "Southlake is internal-only... access is controlled purely by role/permissions." The project still actively uses these two columns.

### `permissions` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by` (project's `permissions` table has no soft-delete at all).

### `modules` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `submodules` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `role_permissions` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `user_permissions` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `user_sessions` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `login_challenges` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `login_otps` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `pending_invites` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.
- **EXTRA IN PROJECT**: `user_entity_type`, `user_entity_id` — same story as `users` above; diagram removed these, project still has them.

### `activity_logs` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.
- **EXTRA IN PROJECT**: `changes` (jsonb) — field-level diff tracking `[{ field, oldValue, newValue }]`. Not present in `diagram.dbml` at all.
- **RENAMED/CHANGED**: `entity_id` is `varchar` in the project vs `uuid` in the diagram (both are documented as a polymorphic reference with no real FK constraint).

> **Pattern across this whole group**: `diagram.dbml` adds `is_deleted/deleted_at/deleted_by` to almost every auth/RBAC table (`permissions`, `modules`, `submodules`, `role_permissions`, `user_permissions`, `user_sessions`, `login_challenges`, `login_otps`, `pending_invites`, `activity_logs`) that the project currently does NOT soft-delete. If you want full soft-delete coverage to match the target design, these 9 tables need the 3 columns added + service-layer changes to filter `is_deleted = false` and to soft-delete instead of hard-delete.

---

## GROUP 2: Masters
*(project module: `masters`, plus `document_type_master`/`state_master`/etc.)*

### `document_type_master` — CHANGED
- **RENAMED/CHANGED**: project's `code` ↔ diagram's `type_code` (same purpose).
- **MISSING IN PROJECT**: `created_by`, `updated_by`.
- **EXTRA IN PROJECT**: `description`.

### `sequence_prefix_master` — REDESIGNED
- **MISSING IN PROJECT**: `sequence_type` ('policy'|'claim'), `prefix_connector`, `seq_start`, `suffix`, `suffix_connector`, `created_by`, `updated_by`.
- **EXTRA IN PROJECT**: `code`, `next_value` (diagram's equivalent is `next_number`), `padding_width`, `description`.
- This table is one of the most reshaped in the whole schema — diagram builds a full "prefix + running number + suffix" numbering scheme (e.g. `POL-000123-CA`); the project's version is a much simpler prefix/next-value counter with a `padding_width` for zero-padding instead.

### `state_master` — no differences

### `state_documents` — CHANGED
- **MISSING IN PROJECT**: `document_type_id` (proper FK to `document_type_master`).
- **EXTRA IN PROJECT**: `document_type` (plain varchar, no FK enforced).
- *(See the "document_type vs document_type_id" note at the end of this group — this pattern repeats across several `*_documents` tables.)*

### `mga_master` — CHANGED
- **MISSING IN PROJECT**: nothing new (diagram removed a field, doesn't add one — see below).
- **EXTRA IN PROJECT**: `tax_payable_inhouse` (diagram comments this field out / removes it), `other_names` (jsonb — diagram's note explicitly says this was "normalized out to `mga_other_names`", i.e. this jsonb column is redundant now that the `mga_other_names` table already exists in the project too), `naics_code`, `contact_name`, `contact_email`, `contact_phone` (diagram moves these into the dedicated `mga_contact_master` table instead of storing a single contact on the MGA record itself).

### `mga_other_names` — CHANGED
- **MISSING IN PROJECT**: `state_id` (diagram scopes each alternate name to a specific state).
- **RENAMED/CHANGED**: project's `name` ↔ diagram's `display_name`.

### `mga_documents` — CHANGED
- **MISSING IN PROJECT**: `document_type_id`.
- **EXTRA IN PROJECT**: `document_type` (plain varchar).

### `contact_domain_master` — REDESIGNED (same table name, different purpose entirely)
- Project's version: **per-MGA whitelist of email domains** (`mga_id`, `domain_name`) used to validate customer-portal signups.
- Diagram's version: **global lookup of functional contact departments** (`domain_code`, `name` — e.g. Underwriting, Claims, Accounting, Compliance, IT, General), referenced by `mga_contact_master.domain_id`.
- **MISSING IN PROJECT**: `domain_code`.
- **EXTRA IN PROJECT**: `mga_id`, `domain_name`.
- Flag this one specifically — it's not a field tweak, it's a completely different table hiding behind the same name.

### `mga_contact_master` — CHANGED
- **MISSING IN PROJECT**: `domain_id` (FK to `contact_domain_master`, only meaningful once that table's redesign above is applied), `designation`, `mobile`, `notes`.
- **RENAMED/CHANGED**: project's `contact_email`/`contact_phone` ↔ diagram's `email`/`phone`.

### `carriers` — no differences

### `carrier_documents` — CHANGED
- **RENAMED/CHANGED**: project's `risk_company_id` ↔ diagram's `carrier_id` (same FK target, `carriers.id`).
- **EXTRA IN PROJECT**: project actually has BOTH `document_type` (varchar) AND `document_type_id` (uuid) — the diagram only wants `document_type_id`. The leftover `document_type` varchar column looks like dead/legacy data worth checking for usage.

### `reinsurer_companies` — no differences

### `broker_master` — REDESIGNED (project is missing a lot)
- **MISSING IN PROJECT**: `license_number`, `company_id`, `id_name`, `address`, `city`, `state`, `zip`, `commission_pct`, `notes`, `created_by`, `updated_by`.
- **RENAMED/CHANGED**: project's `contact_email`/`contact_phone` ↔ diagram's `email`/`phone`.
- This is one of the biggest gaps in the Masters group — the diagram's broker record is much richer (address, license, commission %) than what's actually implemented.

### `broker_documents` — no field differences
Both use `document_type_id` (FK not enforced at the DB level in the project, but the column exists and is named the same).

### `lines_of_business` — no differences
### `cob_master` — no differences

### `product_master` — REDESIGNED
- **MISSING IN PROJECT**: `created_by`, `updated_by`.
- **RENAMED/CHANGED**: project's `product_id` ↔ diagram's `product_code`.
- **EXTRA IN PROJECT**: `lob_id`, `cob_id` (direct FK columns), `description`. Diagram's note is explicit: *"Carrier and MGA are assigned at the treaty level, not here"* and LOB/COB are meant to flow ONLY through the `product_lobs`/`product_cobs` many-to-many junctions. The project keeps both — a direct single `lob_id`/`cob_id` on the product AND the junction tables — which is a redundant/conflicting design worth resolving one way or the other.

### `product_lobs` / `product_cobs` — CHANGED
- **MISSING IN PROJECT**: unique index `(product_id, lob_id)` / `(product_id, cob_id)`. Without it, the project can insert duplicate junction rows for the same product+LOB (or product+COB) pair.

---

## GROUP 3: Treaty
*(project module: `masters`, treaty-related tables)*

### `treaty_type_master` — CHANGED
- **EXTRA IN PROJECT**: `description` (not in diagram).

### `treaties` — REDESIGNED (the single biggest area of drift in the whole schema)
- **MISSING IN PROJECT**: `treaty_type_id` (proper FK to `treaty_type_master` — the project instead stores a plain `treaty_type` varchar defaulting to `'Quota Share'` with no FK at all), `carrier_allocation_type` ('single'|'percentage' — the flag that decides whether to use `treaties.risk_company_id` directly or look up `treaty_state_carriers` per state), `loss_pick_pct`.
- **EXTRA IN PROJECT**: `reinsurer_id` (diagram removed the single treaty-level reinsurer column — reinsurers are meant to live ONLY in `treaty_reinsurers`), `carrier_retention_pct`, `reinsurer_cession_pct` (diagram moved these to `treaty_state_carriers.pct` / `treaty_reinsurers.quota_share`), `is_active` (diagram removes this — treaty status is meant to be derived from `effective_date`/`expiration_date` instead of a stored flag), `policy_seq_prefix`, `policy_seq_start`, `policy_seq_next`, `claim_seq_prefix`, `claim_seq_start`, `claim_seq_next` (diagram removed all six — sequence numbering is meant to flow entirely through the `treaty_sequence_prefixes` junction + `sequence_prefix_master`, not be duplicated onto the treaty row).
- **RENAMED/CHANGED**: project's `treaty_type` (varchar, no FK) is the same concept as diagram's `treaty_type_id` (uuid FK) but implemented completely differently.
- This table currently mixes the "old" design (direct reinsurer/sequence/retention columns) with the "new" junction-table design (which the project also has fully built: `treaty_reinsurers`, `treaty_sequence_prefixes`, `treaty_state_carriers` all exist) — meaning the project is running both patterns side by side right now.

### `treaty_products` — CHANGED
- **MISSING IN PROJECT**: unique index `(treaty_id, product_id)`.

### `treaty_states` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `treaty_state_carriers` — CHANGED
- **RENAMED/CHANGED**: project's `risk_company_id` ↔ diagram's `carrier_id`; project's `retention_pct` ↔ diagram's `pct`.
- **MISSING IN PROJECT**: unique index `(treaty_id, state_id, carrier_id)`; also project's `state_id` is nullable while diagram requires it `not null`.
- **EXTRA IN PROJECT**: `broker_id` — diagram's redesigned junction has no broker reference here at all.

### `treaty_mgas` — CHANGED
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.

### `treaty_reinsurers` — CHANGED
- **RENAMED/CHANGED**: project's `cession_pct` ↔ diagram's `quota_share` (diagram's note literally says *"was cession_pct"* — confirmed rename).
- **MISSING IN PROJECT**: `is_deleted`, `deleted_at`, `deleted_by`.
- **EXTRA IN PROJECT**: `state_id`, `broker_id` — diagram's simplified version has neither.

### `treaty_sequence_prefixes` — CHANGED
- **MISSING IN PROJECT**: unique index `(treaty_id, sequence_prefix_id)`.

### `treaty_lobs`, `treaty_lob_cobs` — see Section 0 (only exist in the project; diagram replaced both with `treaty_products`)

---

## GROUP 4: Workbook / Excel Import
*(project module: `workbook`)*

### `workbooks` — no differences
### `state_exhibits` — no differences

Both tables match field-for-field, including the project-wide-inconsistent camelCase column naming (`monthKey`, `isDeleted`, etc.), which `diagram.dbml` explicitly preserves on purpose (see its comment: *"do NOT convert to snake_case"*).

---

## GROUP 5: Accounting (GL)
*(project modules: `chart-of-accounts`, `gl-mappings`, `journal-entries`, `reports`, plus `cash_settlements` from `workbook`)*

### `chart_of_accounts` — no differences
### `gl_mappings` — no differences

### `chart_of_account_documents` — CHANGED
- **MISSING IN PROJECT**: `document_type_id`.
- **EXTRA IN PROJECT**: `document_type` (plain varchar, no FK).

### `journal_entry_batches` — REDESIGNED
- **MISSING IN PROJECT**: `approved_at`, `approved_by` (diagram tracks an explicit approval workflow step that the project doesn't have).
- **EXTRA IN PROJECT**: `month_key`, `state_code`.
- **RENAMED/CHANGED**: `workbook_id` is `integer` in the project (matches `workbooks.id`, which is an integer PK) but `uuid` in the diagram — likely an oversight in the diagram, since `workbooks.id` is an integer everywhere else in that same file.
- Default `status` differs: project defaults to `'posted'`, diagram defaults to `'pending_review'` — this reflects two different assumed workflows (diagram expects a review/approval step before a batch is posted; the project currently posts directly).
- `treaty_id` and `workbook_id` are plain columns with no FK constraint in the project; diagram enforces both as real foreign keys (`treaty_id` additionally `not null`).

### `journal_entry_drafts` — REDESIGNED (very different shape)
- **MISSING IN PROJECT**: `je_number`, `coa_id` (FK), `state_id`, `product_id`, `sub`, `date`, `dp`, `policy`, `memo`, `created_by`, `updated_at`, `updated_by`.
- **EXTRA IN PROJECT**: `account_code`, `account_name` (the project stores raw account code/name strings on the draft instead of a `coa_id` FK).
- This is effectively a different, much simpler table in the project today — the diagram's version mirrors the full `journal_entries` shape (GL account link, state, product, memo fields, etc.) so that drafts can be edited in place before being copied into `journal_entries` on approval; the project's draft table can't currently carry that much information.

### `journal_entries` — CHANGED
- **MISSING IN PROJECT**: `state_id`, `product_id` (diagram links every GL line to a state and a product for reporting/filtering), `is_deleted`, `deleted_at`, `deleted_by` (the project's `journal_entries` has no soft-delete columns at all — posted GL lines can only be hard-deleted).

### `calculation_report_lines` — REDESIGNED (essentially a different table under the same name)
- **MISSING IN PROJECT**: `batch_id` (FK), `state_id`, `is_total`, `coa_id` (FK), `line_item`, `amount`, `created_at`, `updated_at`.
- **EXTRA IN PROJECT**: `treaty_id`, `line_number`, `line_label`, `formula_expression`, `is_bold`.
- Project's version is a **configurable formula/label template per treaty** (used to define how a report line should be computed/displayed). Diagram's version is a **computed numeric result row per batch+state**, directly linked to a GL account, with a boolean flag for the Total row. These solve different problems — this isn't a simple field diff, it's two unrelated designs sharing a table name.

### `treaty_itd_totals` — CHANGED
- **MISSING IN PROJECT**: unique index `(treaty_id, year, month, state_id)`. All the numeric metric columns themselves match exactly between both files.
- FKs (`treaty_id`, `state_id`) are enforced in the diagram but only logical (no DB constraint) in the project.

### `cash_settlements` — REDESIGNED (different relationship model, not just fields)
- **Project**: one-to-one with `workbooks` — `workbookId` is `not null` + `unique` and is what drives the relationship ("one settlement summary per workbook upload"). `batch_id` and `reinsurer_id` are nullable with no FK.
- **Diagram**: one row per (batch, reinsurer) — `batch_id` and `reinsurer_id` are `not null` FKs with a unique constraint on `(batch_id, reinsurer_id)`, while `workbookId` is just an optional "raw upload trace" field.
- This means the diagram expects **multiple cash-settlement rows per workbook** (one per reinsurer on that treaty), while the project's actual schema only allows **one row per workbook total**. If a treaty ever has more than one reinsurer, the current schema cannot represent a separate settlement per reinsurer — worth checking against how `cash_settlements` is actually being written in the reserves/financial-reports code.

---

## Cross-cutting patterns worth checking in code

1. **`document_type` (varchar) vs `document_type_id` (FK)** — inconsistent across the project itself, not just vs. the diagram:
   - Uses plain `document_type` varchar (no FK): `state_documents`, `mga_documents`, `chart_of_account_documents`
   - Uses `document_type_id` FK-style column already: `broker_documents`
   - Uses BOTH columns: `carrier_documents` (likely legacy leftover — check if `document_type` is still read/written anywhere before removing it)
   - The diagram standardizes everything on `document_type_id` referencing `document_type_master`.

2. **Soft-delete coverage gap** — 9 tables in Group 1 (Auth & RBAC) plus `treaty_states`, `treaty_mgas`, `treaty_reinsurers`, and `journal_entries` have no `is_deleted/deleted_at/deleted_by` in the project even though nearly every other table follows that convention. Confirm whether rows in these tables are ever hard-deleted today.

3. **Old vs. new treaty design running side by side** — the project has fully built the "new" junction-table pattern (`treaty_reinsurers`, `treaty_state_carriers`, `treaty_sequence_prefixes`, `treaty_products`) that the diagram is designed around, but `treaties` itself still carries the "old" columns (`reinsurer_id`, `carrier_retention_pct`, `reinsurer_cession_pct`, `policy_seq_prefix/start/next`, `claim_seq_prefix/start/next`, `is_active`) that the new design was supposed to replace. Worth confirming which one the application code actually reads from today.

4. **Missing unique constraints** — `product_lobs`, `product_cobs`, `treaty_products`, `treaty_state_carriers`, `treaty_sequence_prefixes`, `treaty_itd_totals` are all missing composite unique indexes that the diagram defines, meaning duplicate junction/fact rows are not currently prevented at the DB level.
