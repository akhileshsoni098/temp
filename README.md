# Schema Diff: `database-schema.dbml` (actual project) vs `diagram.dbml` (final/target design)

**database-schema.dbml** = `src/modules/**/entities/*.entity.ts` ke real TypeORM entities se banaya gaya hai — yehi cheez **abhi codebase mein actually chal rahi hai**.

**diagram.dbml** = ek alag, redesign kiya hua "final merged" schema hai (uske apne header ke hisaab se `finaldb.sql` se aaya hai) — ye ek **target/future schema** lagta hai, jo abhi tak implement nahi hua. Isme kayi tables jaan-boojh kar restructure kiye gaye hain current tables ke.

Is file ka use karo ye dekhne ke liye, module-wise: kaunse tables/fields **sirf project mein** hain (implement ho chuke hain par target diagram mein nahi), kaunse **sirf diagram mein** hain (design ho chuke hain par abhi bane nahi), aur kaunse fields **rename/change** hue hain dono ke beech.

Labels jo neeche use kiye hain:
- **PROJECT MEIN EXTRA** — field/table running code mein hai but `diagram.dbml` mein nahi
- **PROJECT MEIN MISSING** — field/table `diagram.dbml` mein hai but code mein abhi tak implement nahi hua
- **RENAME/CHANGE HUA** — same concept hai, bas column ka naam, type, ya constraint alag hai
- **PURA REDESIGN** — table dono jagah hai lekin shape itni alag hai ki practically ye do alag tables hain

---

## IMPORTANT CLARIFICATION — `SoftDeleteEntity` / `AuditableEntity` / `BaseEntity` wala confusion

Aapne `src/common/entities/soft-delete.entity.ts` select kiya tha jisme ye dikh raha hai:

```ts
export abstract class SoftDeleteEntity extends AuditableEntity {
  @Column({ name: 'is_deleted', ... }) isDeleted: boolean;
  @Column({ name: 'deleted_at', ... }) deletedAt: Date;
  @Column({ name: 'deleted_by', ... }) deletedBy: string;
}
```

Aur sawaal ye tha: jab ye class already `is_deleted/deleted_at/deleted_by` define kar rahi hai, toh diff mein kuch tables ke saamne "MISSING" kyun likha hai?

**Maine deeply verify kiya hai — direct har entity file khol ke, sirf summary pe trust nahi kiya.** Pehle ek grep chalaya:

```
grep "extends (SoftDeleteEntity|AuditableEntity|BaseEntity)" src/modules/**/*.entity.ts
→ No matches found
```

Matlab **pure `src/modules/` mein ek bhi entity in teeno base classes ko `extends` nahi karti.** `BaseEntity`, `AuditableEntity`, `SoftDeleteEntity` — teeno abstract classes `src/common/entities/` mein defined toh hain, par **inherit kahin nahi ho rahe — ye dead/unused code hai.**

Uske baad maine in specific entity files ko individually khol ke line-by-line padha (`permission.entity.ts`, `module.entity.ts`, `submodule.entity.ts`, `role-permission.entity.ts`, `user-permission.entity.ts`, `user-session.entity.ts`, `login-challenge.entity.ts`, `login-otp.entity.ts`, `pending-invite.entity.ts`, `activity-log.entity.ts`, `treaty-state.entity.ts`, `treaty-mga.entity.ts`, `treaty-reinsurer.entity.ts`, `journal-entry.entity.ts`) — har ek apne columns **manually** `@Column(...)` decorator se declare karti hai (koi base class extend nahi karti), aur in sab mein `is_deleted`/`deleted_at`/`deleted_by` genuinely **declare hi nahi kiya gaya**.

**Teesra level ka proof — seedha migration files (jo actual DB banati hain) khol ke dekha:**

1. `src/database/migrations/1783223589374-InitialSchema.ts` ka raw `CREATE TABLE` SQL statement har ek table ke liye dekha. Jaise:
   ```sql
   CREATE TABLE "permissions" ("id" uuid ..., "action" character varying ..., "label" character varying ..., "description" character varying, CONSTRAINT ... PRIMARY KEY ("id"))
   ```
   Isme `is_deleted` column hai hi nahi. Same tarah `submodules`, `modules`, `user_permissions`, `pending_invites`, `role_permissions`, `treaty_states`, `treaty_mgas`, `treaty_reinsurers`, `journal_entries`, `user_sessions`, `login_otps`, `login_challenges`, `activity_logs` — in sabke `CREATE TABLE` statements mein bhi `is_deleted`/`deleted_at`/`deleted_by` kahin nahi hai.

2. Ek baad ki migration `src/database/migrations/1783225145073-AddSoftDeleteColumns.ts` hai jiska poora kaam hi ye hai ki purani tables mein `ALTER TABLE ... ADD "is_deleted"` chala ke soft-delete columns add kare — **par ye sirf 12 specific tables pe chalti hai**: `roles`, `mga_master`, `reinsurer_companies`, `lines_of_business`, `cob_master`, `state_master`, `treaties`, `state_documents`, `mga_documents`, `chart_of_accounts`, `gl_mappings`, `chart_of_account_documents`. Flagged 14 tables mein se **ek bhi is list mein nahi hai**.

3. Teesri migration `1783226000000-AddMissingModules.ts` sirf `modules` table mein seed data insert karti hai (`INSERT INTO "modules" ...`), koi column add nahi karti.

**Matlab confirm ho gaya — DB level pe bhi, entity code level pe bhi**: `permissions`, `modules`, `submodules`, `role_permissions`, `user_permissions`, `user_sessions`, `login_challenges`, `login_otps`, `pending_invites`, `activity_logs`, `treaty_states`, `treaty_mgas`, `treaty_reinsurers`, `journal_entries` — in 14 tables mein `is_deleted`/`deleted_at`/`deleted_by` column **kabhi bana hi nahi**, na entity mein na actual Postgres table mein. `soft-delete.entity.ts` in tables se **bilkul connected nahi hai** — na `extends` ke through, na kisi migration ke through.

**Toh project mein `is_deleted` "achieve" kaise ho raha hai (jin tables mein hai unmein)?** `SoftDeleteEntity` base class ko inherit karke **nahi** — har entity file mein manually, alag-alag, copy-paste karke `@Column({ name: 'is_deleted', type: 'boolean', default: false }) isDeleted: boolean;` likha gaya hai, aur DB side pe bhi corresponding manual `ALTER TABLE` migration chalayi gayi. `src/common/entities/base.entity.ts` / `auditable.entity.ts` / `soft-delete.entity.ts` — teeno files ban toh chuki hain (shayad future mein consistency ke liye plan tha), par **poora codebase inhe completely ignore karta hai** — har entity independently, column-by-column manually likhi gayi hai. Isliye ye teeno base classes practically **dead code** hain.

**Practical takeaway**: agar aap chahte ho ki `permissions`, `modules`, `submodules`, `role_permissions`, `user_permissions`, `user_sessions`, `login_challenges`, `login_otps`, `pending_invites`, `activity_logs`, `treaty_states`, `treaty_mgas`, `treaty_reinsurers`, `journal_entries` — in sab mein bhi soft-delete ho, toh do options hain:
1. Har entity mein manually `is_deleted`/`deleted_at`/`deleted_by` columns add karo (jaisa baaki tables mein pattern hai) + ek naya migration likho jo in 14 tables pe `ALTER TABLE ... ADD` chalaye, YA
2. In entities ko actually `extends SoftDeleteEntity` karo taaki wo already-likha hua base class finally kaam mein aaye — is case mein bhi migration generate karni padegi kyunki DB mein columns abhi hain hi nahi, sirf entity class change karne se DB apne aap nahi badlega.

---

## 0. Poore table-level differences

### Sirf project mein hain (`database-schema.dbml`) — `diagram.dbml` mein bilkul nahi
| Table | Module | Note |
|---|---|---|
| `locked_periods` | masters | Accounting period editable hai ya nahi, ye control karta hai. `diagram.dbml` mein iska koi equivalent hi nahi hai. |
| `treaty_lobs` | masters | Junction: treaty ↔ line of business. `diagram.dbml` ne ye poora concept `treaty_products` se replace kar diya (treaty ↔ product, jisme product already LOB+COB dono bundle karta hai). |
| `treaty_lob_cobs` | masters | Doosra-level junction: treaty_lob ↔ class of business. Same reason — `treaty_products` ke favor mein hata diya gaya. |

### Sirf `diagram.dbml` mein hain — project mein implement nahi
Koi nahi. `diagram.dbml` ke har table ka same-naam wala counterpart project mein pehle se implement hai (par kayi reshape/redesign kiye gaye hain — neeche dekho).

### Table count
- `database-schema.dbml` (project, actual): **53 tables**
- `diagram.dbml` (target design): **50 tables**

---

## GROUP 1: Auth & RBAC
*(project modules: `users`, `roles`, `permissions`, `auth`, `activity-logs`)*

### `roles` — koi farak nahi
Dono files mein fields exactly same hain.

### `users` — CHANGED
- **PROJECT MEIN MISSING**: `team_id` — diagram ka note kehta hai ye future team-assignment ke liye placeholder hai, hamesha NULL rehta hai, na koi teams table hai na FK — matlab abhi banane layak kuch nahi hai.
- **PROJECT MEIN EXTRA**: `user_entity_type`, `user_entity_id` — user ko MGA/broker/customer entity se polymorphic link karne ke liye. Diagram ka note saaf kehta hai ye **hataye gaye** kyunki "Southlake internal-only hai... access sirf role/permissions se control hota hai." Project abhi bhi ye dono columns actively use karta hai.

### `permissions` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by` (project ki `permissions` table mein koi soft-delete hai hi nahi).

### `modules` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `submodules` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `role_permissions` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `user_permissions` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `user_sessions` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `login_challenges` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `login_otps` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `pending_invites` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.
- **PROJECT MEIN EXTRA**: `user_entity_type`, `user_entity_id` — `users` wali same story; diagram ne hataye, project mein abhi bhi hain.

### `activity_logs` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.
- **PROJECT MEIN EXTRA**: `changes` (jsonb) — field-level diff track karta hai `[{ field, oldValue, newValue }]`. `diagram.dbml` mein ye field hai hi nahi.
- **RENAME/CHANGE HUA**: `entity_id` project mein `varchar` hai vs diagram mein `uuid` (dono jagah ye polymorphic reference hai bina real FK constraint ke).

> **Pura group ka pattern**: `diagram.dbml` bijli almost har auth/RBAC table (`permissions`, `modules`, `submodules`, `role_permissions`, `user_permissions`, `user_sessions`, `login_challenges`, `login_otps`, `pending_invites`, `activity_logs`) mein `is_deleted/deleted_at/deleted_by` add karta hai, jo project abhi soft-delete NAHI karta. Agar target design ke hisaab se full soft-delete coverage chahiye, toh in 9 tables mein 3 columns add karne padenge + service-layer mein `is_deleted = false` filter aur hard-delete ki jagah soft-delete karna hoga. (Upar wala "IMPORTANT CLARIFICATION" section dekho — `SoftDeleteEntity` base class already ban chuka hai, bas use ho nahi raha.)

---

## GROUP 2: Masters
*(project module: `masters`, plus `document_type_master`/`state_master`/etc.)*

### `document_type_master` — CHANGED
- **RENAME/CHANGE HUA**: project ka `code` ↔ diagram ka `type_code` (same purpose).
- **PROJECT MEIN MISSING**: `created_by`, `updated_by`.
- **PROJECT MEIN EXTRA**: `description`.

### `sequence_prefix_master` — PURA REDESIGN
- **PROJECT MEIN MISSING**: `sequence_type` ('policy'|'claim'), `prefix_connector`, `seq_start`, `suffix`, `suffix_connector`, `created_by`, `updated_by`.
- **PROJECT MEIN EXTRA**: `code`, `next_value` (diagram ka equivalent `next_number` hai), `padding_width`, `description`.
- Pure schema mein sabse zyada reshape hui table yahi hai — diagram ek full "prefix + running number + suffix" numbering scheme banata hai (jaise `POL-000123-CA`); project ka version bahut simple prefix/next-value counter hai jisme zero-padding ke liye `padding_width` hai.

### `state_master` — koi farak nahi

### `state_documents` — CHANGED
- **PROJECT MEIN MISSING**: `document_type_id` (proper FK `document_type_master` ki taraf).
- **PROJECT MEIN EXTRA**: `document_type` (plain varchar, koi FK enforce nahi hai).
- *(Is group ke aakhir mein "document_type vs document_type_id" wala note dekho — ye pattern kayi `*_documents` tables mein repeat hota hai.)*

### `mga_master` — CHANGED
- **PROJECT MEIN MISSING**: kuch naya nahi (diagram ne field hataya hai, add nahi kiya — neeche dekho).
- **PROJECT MEIN EXTRA**: `tax_payable_inhouse` (diagram isse comment-out/remove karta hai), `other_names` (jsonb — diagram ka note saaf kehta hai ye "`mga_other_names` mein normalize ho chuka hai", matlab ye jsonb column redundant hai kyunki `mga_other_names` table project mein pehle se hi maujood hai), `naics_code`, `contact_name`, `contact_email`, `contact_phone` (diagram inhe alag `mga_contact_master` table mein daal deta hai, MGA record pe ek single contact rakhne ki jagah).

### `mga_other_names` — CHANGED
- **PROJECT MEIN MISSING**: `state_id` (diagram har alternate name ko ek specific state se scope karta hai).
- **RENAME/CHANGE HUA**: project ka `name` ↔ diagram ka `display_name`.

### `mga_documents` — CHANGED
- **PROJECT MEIN MISSING**: `document_type_id`.
- **PROJECT MEIN EXTRA**: `document_type` (plain varchar).

### `contact_domain_master` — PURA REDESIGN (same table name, purpose bilkul alag)
- Project ka version: **har MGA ke liye allowed email domains ki whitelist** (`mga_id`, `domain_name`) — customer-portal signup validate karne ke liye.
- Diagram ka version: **functional contact departments ka global lookup** (`domain_code`, `name` — jaise Underwriting, Claims, Accounting, Compliance, IT, General), jise `mga_contact_master.domain_id` reference karta hai.
- **PROJECT MEIN MISSING**: `domain_code`.
- **PROJECT MEIN EXTRA**: `mga_id`, `domain_name`.
- Isko specifically flag karna zaroori hai — ye field tweak nahi hai, same naam ke peeche puri alag table chhupi hai.

### `mga_contact_master` — CHANGED
- **PROJECT MEIN MISSING**: `domain_id` (FK `contact_domain_master` ki taraf — sirf tab meaningful hoga jab upar wala redesign apply ho), `designation`, `mobile`, `notes`.
- **RENAME/CHANGE HUA**: project ka `contact_email`/`contact_phone` ↔ diagram ka `email`/`phone`.

### `carriers` — koi farak nahi

### `carrier_documents` — CHANGED
- **RENAME/CHANGE HUA**: project ka `risk_company_id` ↔ diagram ka `carrier_id` (same FK target, `carriers.id`).
- **PROJECT MEIN EXTRA**: project mein actually DONO hain — `document_type` (varchar) AND `document_type_id` (uuid) — diagram sirf `document_type_id` chahta hai. Bacha hua `document_type` varchar column legacy leftover lag raha hai — code mein check karo ye kahin use ho raha hai ya nahi, phir hataya ja sakta hai.

### `reinsurer_companies` — koi farak nahi

### `broker_master` — PURA REDESIGN (project mein bahut kuch missing hai)
- **PROJECT MEIN MISSING**: `license_number`, `company_id`, `id_name`, `address`, `city`, `state`, `zip`, `commission_pct`, `notes`, `created_by`, `updated_by`.
- **RENAME/CHANGE HUA**: project ka `contact_email`/`contact_phone` ↔ diagram ka `email`/`phone`.
- Masters group ka sabse bada gap yahi hai — diagram ka broker record (address, license, commission %) actual implementation se kaafi zyada rich hai.

### `broker_documents` — field mein koi farak nahi
Dono jagah `document_type_id` use hota hai (project mein FK level pe enforce nahi hai, par column same naam se maujood hai).

### `lines_of_business` — koi farak nahi
### `cob_master` — koi farak nahi

### `product_master` — PURA REDESIGN
- **PROJECT MEIN MISSING**: `created_by`, `updated_by`.
- **RENAME/CHANGE HUA**: project ka `product_id` ↔ diagram ka `product_code`.
- **PROJECT MEIN EXTRA**: `lob_id`, `cob_id` (direct FK columns), `description`. Diagram ka note saaf kehta hai: *"Carrier and MGA are assigned at the treaty level, not here"* aur LOB/COB sirf `product_lobs`/`product_cobs` many-to-many junctions ke through flow karne chahiye. Project mein dono hain — direct single `lob_id`/`cob_id` product pe AND junction tables bhi — ye redundant/conflicting design hai, kisi ek tarike se resolve karna chahiye.

### `product_lobs` / `product_cobs` — CHANGED
- **PROJECT MEIN MISSING**: unique index `(product_id, lob_id)` / `(product_id, cob_id)`. Iske bina project mein same product+LOB (ya product+COB) pair ke duplicate junction rows insert ho sakte hain.

---

## GROUP 3: Treaty
*(project module: `masters`, treaty-related tables)*

### `treaty_type_master` — CHANGED
- **PROJECT MEIN EXTRA**: `description` (diagram mein nahi hai).

### `treaties` — PURA REDESIGN (poore schema mein sabse bada drift yahi hai)
- **PROJECT MEIN MISSING**: `treaty_type_id` (proper FK `treaty_type_master` ki taraf — project ki jagah ek plain `treaty_type` varchar rakhta hai jo default `'Quota Share'` hai, koi FK nahi), `carrier_allocation_type` ('single'|'percentage' — ye flag decide karta hai ki `treaties.risk_company_id` direct use karna hai ya per-state `treaty_state_carriers` dekhna hai), `loss_pick_pct`.
- **PROJECT MEIN EXTRA**: `reinsurer_id` (diagram ne single treaty-level reinsurer column hata diya — reinsurers ab sirf `treaty_reinsurers` mein rehne chahiye), `carrier_retention_pct`, `reinsurer_cession_pct` (diagram ne inhe `treaty_state_carriers.pct` / `treaty_reinsurers.quota_share` mein shift kar diya), `is_active` (diagram isse hata deta hai — treaty ka status `effective_date`/`expiration_date` se derive hona chahiye, stored flag se nahi), `policy_seq_prefix`, `policy_seq_start`, `policy_seq_next`, `claim_seq_prefix`, `claim_seq_start`, `claim_seq_next` (diagram ne saare 6 hata diye — sequence numbering ab poori tarah `treaty_sequence_prefixes` junction + `sequence_prefix_master` ke through flow honi chahiye, treaty row pe duplicate nahi honi chahiye).
- **RENAME/CHANGE HUA**: project ka `treaty_type` (varchar, no FK) wahi concept hai jo diagram ka `treaty_type_id` (uuid FK) hai, bas implementation bilkul alag hai.
- Ye table abhi "purana" design (direct reinsurer/sequence/retention columns) aur "naya" junction-table design — dono ek saath carry kar rahi hai (project mein `treaty_reinsurers`, `treaty_sequence_prefixes`, `treaty_state_carriers` sab already ban chuke hain). Matlab dono patterns ek saath chal rahe hain abhi — confirm karna padega application code actually kaunsa read karta hai.

### `treaty_products` — CHANGED
- **PROJECT MEIN MISSING**: unique index `(treaty_id, product_id)`.

### `treaty_states` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `treaty_state_carriers` — CHANGED
- **RENAME/CHANGE HUA**: project ka `risk_company_id` ↔ diagram ka `carrier_id`; project ka `retention_pct` ↔ diagram ka `pct`.
- **PROJECT MEIN MISSING**: unique index `(treaty_id, state_id, carrier_id)`; saath hi project mein `state_id` nullable hai jabki diagram mein `not null` required hai.
- **PROJECT MEIN EXTRA**: `broker_id` — diagram ke redesigned junction mein koi broker reference hai hi nahi.

### `treaty_mgas` — CHANGED
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.

### `treaty_reinsurers` — CHANGED
- **RENAME/CHANGE HUA**: project ka `cession_pct` ↔ diagram ka `quota_share` (diagram ka note literally kehta hai *"was cession_pct"* — rename confirm hai).
- **PROJECT MEIN MISSING**: `is_deleted`, `deleted_at`, `deleted_by`.
- **PROJECT MEIN EXTRA**: `state_id`, `broker_id` — diagram ka simplified version dono mein se koi nahi rakhta.

### `treaty_sequence_prefixes` — CHANGED
- **PROJECT MEIN MISSING**: unique index `(treaty_id, sequence_prefix_id)`.

### `treaty_lobs`, `treaty_lob_cobs` — Section 0 dekho (sirf project mein hain; diagram ne dono `treaty_products` se replace kar diye)

---

## GROUP 4: Workbook / Excel Import
*(project module: `workbook`)*

### `workbooks` — koi farak nahi
### `state_exhibits` — koi farak nahi

Dono tables field-for-field match karti hain, project-wide camelCase column naming (`monthKey`, `isDeleted`, etc.) samet — jise `diagram.dbml` jaan-boojh kar preserve karta hai (uska comment: *"do NOT convert to snake_case"*).

---

## GROUP 5: Accounting (GL)
*(project modules: `chart-of-accounts`, `gl-mappings`, `journal-entries`, `reports`, plus `cash_settlements` from `workbook`)*

### `chart_of_accounts` — koi farak nahi
### `gl_mappings` — koi farak nahi

### `chart_of_account_documents` — CHANGED
- **PROJECT MEIN MISSING**: `document_type_id`.
- **PROJECT MEIN EXTRA**: `document_type` (plain varchar, no FK).

### `journal_entry_batches` — PURA REDESIGN
- **PROJECT MEIN MISSING**: `approved_at`, `approved_by` (diagram ek explicit approval workflow step track karta hai jo project mein nahi hai).
- **PROJECT MEIN EXTRA**: `month_key`, `state_code`.
- **RENAME/CHANGE HUA**: `workbook_id` project mein `integer` hai (kyunki `workbooks.id` integer PK hai) but diagram mein `uuid` hai — ye diagram ki apni file ke andar hi ek inconsistency lagti hai, kyunki `workbooks.id` waha bhi integer hi hai.
- Default `status` alag hai: project `'posted'` default rakhta hai, diagram `'pending_review'` — do alag workflows dikhate hain (diagram batch post hone se pehle review/approval step expect karta hai; project abhi direct post kar deta hai).
- `treaty_id` aur `workbook_id` project mein plain columns hain bina FK constraint ke; diagram dono ko real foreign keys banata hai (`treaty_id` additionally `not null`).

### `journal_entry_drafts` — PURA REDESIGN (bahut alag shape)
- **PROJECT MEIN MISSING**: `je_number`, `coa_id` (FK), `state_id`, `product_id`, `sub`, `date`, `dp`, `policy`, `memo`, `created_by`, `updated_at`, `updated_by`.
- **PROJECT MEIN EXTRA**: `account_code`, `account_name` (project draft pe raw account code/name strings rakhta hai, `coa_id` FK ki jagah).
- Ye effectively ek alag, bahut simple table hai project mein abhi — diagram ka version poori `journal_entries` shape mirror karta hai (GL account link, state, product, memo fields, etc.) taaki drafts ko approval se pehle edit kiya ja sake; project ka draft table itni saari info carry nahi kar sakta abhi.

### `journal_entries` — CHANGED
- **PROJECT MEIN MISSING**: `state_id`, `product_id` (diagram har GL line ko ek state aur product se link karta hai reporting/filtering ke liye), `is_deleted`, `deleted_at`, `deleted_by` (project ki `journal_entries` mein koi soft-delete columns hai hi nahi — posted GL lines sirf hard-delete ho sakti hain).

### `calculation_report_lines` — PURA REDESIGN (practically same naam ke peeche alag table)
- **PROJECT MEIN MISSING**: `batch_id` (FK), `state_id`, `is_total`, `coa_id` (FK), `line_item`, `amount`, `created_at`, `updated_at`.
- **PROJECT MEIN EXTRA**: `treaty_id`, `line_number`, `line_label`, `formula_expression`, `is_bold`.
- Project ka version ek **configurable formula/label template hai per treaty** (report line kaise compute/display honi chahiye, wo define karta hai). Diagram ka version ek **computed numeric result row hai per batch+state**, direct GL account se linked, Total row ke liye ek boolean flag ke saath. Dono alag problems solve karte hain — ye simple field diff nahi hai, do unrelated designs hain jo ek table naam share kar rahe hain.

### `treaty_itd_totals` — CHANGED
- **PROJECT MEIN MISSING**: unique index `(treaty_id, year, month, state_id)`. Baaki saare numeric metric columns dono files mein exactly match karte hain.
- FKs (`treaty_id`, `state_id`) diagram mein enforce hain but project mein sirf logical hain (koi DB constraint nahi).

### `cash_settlements` — PURA REDESIGN (relationship model hi alag hai, sirf fields nahi)
- **Project**: `workbooks` ke saath one-to-one — `workbookId` `not null` + `unique` hai aur yehi relationship drive karta hai ("ek workbook upload ka ek hi settlement summary"). `batch_id` aur `reinsurer_id` nullable hain, koi FK nahi.
- **Diagram**: har (batch, reinsurer) ke liye ek row — `batch_id` aur `reinsurer_id` `not null` FKs hain, saath mein unique constraint `(batch_id, reinsurer_id)` pe, jabki `workbookId` sirf ek optional "raw upload trace" field hai.
- Matlab diagram expect karta hai ki **ek workbook ke multiple cash-settlement rows ho sakte hain** (treaty ke har reinsurer ke liye alag), jabki project ka actual schema sirf **ek workbook = ek row total** allow karta hai. Agar kisi treaty mein ek se zyada reinsurer hue, toh current schema alag-alag reinsurer ke liye alag settlement represent nahi kar sakta — reserves/financial-reports code mein check karo `cash_settlements` actually kaise likha ja raha hai.

---

## Cross-cutting patterns — code mein zaroor check karo

1. **`document_type` (varchar) vs `document_type_id` (FK)** — ye project ke andar hi inconsistent hai, sirf diagram ke against nahi:
   - Plain `document_type` varchar use karte hain (koi FK nahi): `state_documents`, `mga_documents`, `chart_of_account_documents`
   - `document_type_id` FK-style column already hai: `broker_documents`
   - DONO columns hain: `carrier_documents` (shayad legacy leftover — pehle check karo `document_type` kahin read/write toh nahi ho raha, phir hi hatao)
   - Diagram sabko `document_type_id` pe standardize karta hai, jo `document_type_master` ko reference karta hai.

2. **Soft-delete coverage gap** — Group 1 (Auth & RBAC) ke 9 tables plus `treaty_states`, `treaty_mgas`, `treaty_reinsurers`, aur `journal_entries` mein `is_deleted/deleted_at/deleted_by` hai hi nahi, jabki almost baaki saari tables is convention ko follow karti hain. (Reason upar "IMPORTANT CLARIFICATION" section mein hai — `SoftDeleteEntity` base class bana toh hai par kahin extend nahi hota, isliye ye gap consistent hai.) Confirm karo ki in tables ki rows abhi hard-delete toh nahi ho rahi.

3. **Treaty ka purana vs naya design ek saath chal raha hai** — project ne "naya" junction-table pattern (`treaty_reinsurers`, `treaty_state_carriers`, `treaty_sequence_prefixes`, `treaty_products`) pura bana liya hai jispe diagram design based hai, lekin `treaties` table khud abhi bhi "purane" columns carry kar rahi hai (`reinsurer_id`, `carrier_retention_pct`, `reinsurer_cession_pct`, `policy_seq_prefix/start/next`, `claim_seq_prefix/start/next`, `is_active`) jinhe naye design mein replace hona tha. Confirm karna padega application code actually kaunsa padhta hai.

4. **Missing unique constraints** — `product_lobs`, `product_cobs`, `treaty_products`, `treaty_state_carriers`, `treaty_sequence_prefixes`, `treaty_itd_totals` — in sab mein diagram wale composite unique indexes missing hain, matlab abhi DB level pe duplicate junction/fact rows rukte nahi hain.
