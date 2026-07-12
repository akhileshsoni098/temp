# Migration Step-by-Step Guide (Southlake_service)

Reference: `npm run migration:*` scripts defined in `package.json`, backed by TypeORM CLI (`typeorm/cli`) + `src/database/data-source.ts`.

`synchronize: false` hai is project me (dekh lo `data-source.ts`) — iska matlab DB kabhi apne aap change nahi hoga entity edit karne se. Har change ke liye ek migration file chahiye hi hogi, phir usko run karna padega.

---

## SCENARIO A — Naya Entity/Table banate waqt (New Entity Creation)

**Step 1 — Entity file banao**
`src/modules/<module>/entities/<name>.entity.ts` me naya `@Entity('table_name')` class likho (existing entities jaisa pattern follow karo — id uuid pk, columns, soft-delete cols agar chahiye).

**Step 2 — Module me register karo**
Naya entity `<module>.module.ts` ke `TypeOrmModule.forFeature([...])` array me add karo. **Ye step miss mat karna** — agar register nahi kiya to app me `@InjectRepository()` fail hoga, chahe migration chal bhi jaye.
> Note: `src/database/data-source.ts` glob pattern (`modules/**/entities/*.entity.ts`) se migration-generate automatically naya entity file pick kar leta hai, chahe module me register ho ya na ho — isliye generate to chalega, lekin app runtime me tabhi kaam karega jab module registration ho.

**Step 3 — Migration generate karo**
```
npm run migration:generate -- src/database/migrations/Create<EntityName>Table
```
Example: naya `Tpa` entity banaya to:
```
npm run migration:generate -- src/database/migrations/CreateTpaEdgeclaimTable
```
Ye command current entities vs current DB schema compare karke `CREATE TABLE ...` wali migration file khud bana degi.

**Step 4 — Generated file review karo**
`src/database/migrations/<timestamp>-Create<EntityName>Table.ts` khol ke dekho — CREATE TABLE, FK constraints, indexes sab sahi hai ya nahi.

**Step 5 — Migration run karo**
```
npm run migration:run
```

**File naming convention (is project me already jo follow ho raha hai):**
```
src/database/migrations/<epoch-timestamp>-<PascalCaseDescriptiveName>.ts
```
Example: `1783570000000-AddBrokerDetailsAndLicensedStates.ts`
Class name andar: `<PascalCaseDescriptiveName><epoch-timestamp>` → `AddBrokerDetailsAndLicensedStates1783570000000`

---

## SCENARIO B — Existing Entity/Table modify karte waqt (add/rename/drop column, add FK, etc.)

**Step 1 — Entity file edit karo**
Jis `.entity.ts` file me change chahiye (column add/remove/rename, type change, nullable change, naya relation) — wahi edit karo.

**Step 2 — Migration generate karo**
```
npm run migration:generate -- src/database/migrations/<VerbDescribingChange>
```
Examples:
```
npm run migration:generate -- src/database/migrations/AddCommissionPctToBroker
npm run migration:generate -- src/database/migrations/RenameCarrierIdToCarrierCode
npm run migration:generate -- src/database/migrations/DropContactNameFromBroker
```

**Step 3 — ZAROOR review karo (ye step modification me sabse important hai)**
Rename ke case me TypeORM kabhi kabhi samajh nahi paata ki "rename hua hai" — usko lagta hai **column drop + naya column add** hua hai. Agar aisa hua:
- Manually generated file me `ALTER TABLE ... RENAME COLUMN "old" TO "new"` likh do (jaisa purani migration `1783570000000-AddBrokerDetailsAndLicensedStates.ts` me pattern hai), DROP+ADD ki jagah — warna existing data loss ho jayega.

**Step 4 — Migration run karo**
```
npm run migration:run
```

**File naming same convention:**
```
<timestamp>-<PascalCaseDescriptiveName>.ts
```

---

## SCENARIO C — Table ka naam rename karte waqt (e.g. `broker_master` → `master_broker`)

⚠️ **Ye sabse risky scenario hai — yahan `migration:generate` bilkul use MAT karo.**

**Kyun risky hai:** Column-rename me TypeORM confuse hoke DROP+ADD kar deta hai (Scenario B dekho), lekin table-rename me ye problem aur bada ho jaata hai — `migration:generate` ko ye samajh hi nahi aata ki rename hua hai. Wo seedha ye generate kar dega:
```sql
DROP TABLE "broker_master";          -- SAB DATA + saari rows permanently delete
CREATE TABLE "master_broker" (...);  -- bilkul khaali naya table
```
Isse existing broker data chala jayega, aur jo bhi dusre tables (`broker_licensed_states`, `treaty_state_carriers`, `treaty_reinsurers`) is table ko FK se reference karte hain, unke constraints bhi toot jayenge.

**Step 1 — Entity me table name badlo**
```ts
@Entity('broker_master')   // pehle
```
→
```ts
@Entity('master_broker')   // isme badlo
```

**Step 2 — Manually migration likho** (auto-generate ki jagah empty stub banao)
```
npm run migration:create -- src/database/migrations/RenameBrokerMasterToMasterBroker
```
Andar khud likho:
```ts
import { MigrationInterface, QueryRunner } from 'typeorm';

export class RenameBrokerMasterToMasterBroker1783xxxxxxxxx implements MigrationInterface {
  name = 'RenameBrokerMasterToMasterBroker1783xxxxxxxxx';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "broker_master" RENAME TO "master_broker"`);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "master_broker" RENAME TO "broker_master"`);
  }
}
```
`ALTER TABLE ... RENAME TO ...` se Postgres andar hi andar sab kuch (data, indexes, aur `broker_licensed_states`/`treaty_state_carriers`/`treaty_reinsurers` jaise tables ke FK constraints jo `broker_master(id)` ko point karte hain) automatically naye naam pe transfer kar deta hai — **kuch bhi tootega nahi, koi data loss nahi.**

**Step 3 — Run karo**
```
npm run migration:run
```

**Step 4 — Extra check jo sirf table-rename me zaroori hai**
Repo me kahin agar koi raw SQL string me `"broker_master"` hardcoded hai (seed scripts, custom `.query()` calls, kisi report/service me) — wahan bhi manually `master_broker` karna padega, kyunki wo ORM ke through nahi jaate. Normal `@InjectRepository(Broker)` wale saare service/DAO calls automatically theek rahenge (wo entity class ke through jaate hain, table-name string se nahi) — sirf raw SQL check karna kaafi hai.

---

## Har scenario ke baad — verification steps (dono case me common)

1. **Build check:**
   ```
   npm run build
   ```
2. **DB me confirm karo columns/tables sahi bane:** (jaisa broker `updated_by` bug debug karte waqt kiya tha) — `information_schema.columns` query se check kar sakte ho, ya seedha psql/DB client se `\d table_name`.
3. **Sync check (optional but recommended):**
   ```
   npm run migration:generate -- src/database/migrations/_check
   ```
   Agar output "No changes in database schema were found" aaye → entities aur DB perfectly sync hain. Agar koi file ban jaye to usko delete kar do (matlab kuch mismatch reh gaya tha).

---

## Mistake ho gaya to (Undo)

```
npm run migration:revert
```
Sirf sabse **last applied** migration revert hogi. Dobara chalao to usse pehle wali bhi revert hogi (ek-ek karke peeche jaata hai).

---

## Quick command cheat-sheet

| Situation | Command |
|---|---|
| Naya entity/table | `npm run migration:generate -- src/database/migrations/Create<Name>Table` |
| Existing entity me column/field change | `npm run migration:generate -- src/database/migrations/<VerbChangeName>` |
| Khud se SQL likh ke migration banani ho (auto-generate ke bina) | `npm run migration:create -- src/database/migrations/<Name>` (empty up/down stub banega, khud fill karo) |
| Migration apply karna | `npm run migration:run` |
| Last migration undo | `npm run migration:revert` |
| Sync-check (no pending changes confirm) | `npm run migration:generate -- src/database/migrations/_check` |

---

## Golden rules (in DB ko manually badalte waqt zaroor follow karo)

1. **Kabhi bhi DB table ko directly psql/GUI se edit mat karo.** Hamesha entity change karo → migration generate/run karo. Warna entity aur DB out-of-sync ho jayenge (jaisa broker `updated_by` wala bug hua tha — ek migration file empty thi lekin DB me already "run" mark ho chuki thi).
2. **Generated migration hamesha review karo** — khaas kar rename/drop wale case me, kyunki auto-generate kabhi galat DROP+CREATE bana sakta hai.
3. **Ek migration file jo already `migration:run` se apply ho chuki hai, usko edit mat karo.** Agar mistake mila to naya migration file banao jo usko fix kare (jaisa humne `AddUpdatedByToBrokerMasterFix` naya migration banaya tha, purani empty stub ko edit karne ki jagah).
4. **`db:reset` / `db:refresh` sirf tab chalao jab poora DB wipe karke fresh seed chahiye** — normal entity change/modify ke liye kabhi zaroorat nahi.
5. Har entity change ke baad `npm run build` chala ke confirm karo TypeScript errors nahi hain, phir migration run karo.
