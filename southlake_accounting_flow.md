# Southlake Insurance App — Kaise Kaam Karta Hai (Application Flow)

Yeh document sirf **samajhne ke liye** hai — koi technical jaankari nahi, bas ek aam user jaise is system ko use karta hai waise hi yahan bataya gaya hai.

---

## 1. Sabse pehla kadam — Login

```
   ┌─────────────┐      ┌───────────────┐      ┌───────────────────┐
   │  Email +    │ ───▶ │  OTP aata hai │ ───▶ │  App khul jata hai │
   │  Password   │      │ (verify OTP)  │      │ (Dashboard dikhta) │
   └─────────────┘      └───────────────┘      └───────────────────┘
```

- User apna **email + password** daalta hai.
- Sahi hone par ek **OTP (One Time Password)** aata hai — usko verify karna hota hai.
- OTP sahi nikla toh user andar aa jata hai aur usko **Dashboard** dikhta hai.
- Agar pehle se kisi aur device/browser me login hai, toh ek confirmation poocha jata hai ("kya purana session hata kar yahan login karna hai?").

---

## 2. Sabse zaroori concept — Role aur Permission

Yahi poore app ka dil hai. Isko achhe se samajh lena kaafi hai.

```
   ┌────────┐        ┌───────────────┐        ┌────────┐
   │  Role  │ ─────▶ │  Permissions  │ ─────▶ │  User  │
   │ (jaise │  di    │ (View/Create/ │  milti │ (jaise │
   │ Admin) │  jati  │  Edit/Delete) │  hain  │ Rohit) │
   └────────┘  hain  └───────────────┘        └────────┘
```

- Har **User** ko ek **Role** milta hai (jaise: Superadmin, Admin, ya koi custom role — jaise "Accountant", "Data Entry").
- Har **Role** me yeh tay hota hai ki woh kin-kin cheezon ko:
  - sirf **dekh (View)** sakta hai
  - **naya bana (Create)** sakta hai
  - **badlaav (Edit)** kar sakta hai
  - **mita (Delete)** sakta hai
- Yeh setting **Roles & Permissions** page se Admin/Superadmin control karta hai.
- Zaroorat pade toh kisi ek user ko uske role se alag hatkar bhi koi extra permission di ja sakti hai, ya koi permission chheeni ja sakti hai (individual override).

**Sabse zaroori baat:** agar kisi role/user ke paas kisi kaam ki permission nahi hai, toh:
- woh button/option usko **sidebar aur screen par dikhega hi nahi**.
- agar koi zabardasti try bhi kare, toh **server khud us request ko reject kar dega** — yaani sirf button chhupana nahi, balki asli security bhi hai.

---

## 3. Login ke baad — Sidebar apne aap badalta hai

```
   User login karta hai
          │
          ▼
   Uska Role check hota hai
          │
          ▼
   Uski Permissions pata chalti hain
          │
          ▼
   Sirf wahi Menu/Options sidebar me dikhte hain
   jinki usko ijaazat hai
```

Udaharan:
- Jis user ke paas sirf **"MGA dekhne"** ki permission hai, usko sidebar me sirf MGA wala option dikhega — Treaties, Roles, Accounting kuch nahi dikhega.
- Jiske paas **Superadmin** role hai, usko poora app dikhega — koi pabandi nahi.
- Agar kal kisi role ki permission badal di jaaye (kuch naya diya ya hataya gaya), toh agli baar jab woh user login karega, sidebar apne aap update ho jayega — koi alag se setting nahi karni padti.

---

## 4. App ke mukhya hisse (Modules)

```
                         ┌─────────────────────┐
                         │      Dashboard       │  (sabko dikhta hai)
                         └─────────────────────┘
                                    │
      ┌─────────────────┬──────────┴──────────┬─────────────────┐
      ▼                 ▼                     ▼                 ▼
┌───────────┐   ┌───────────────┐   ┌──────────────────┐  ┌─────────────┐
│  Masters   │   │  Accounting   │   │ User Management   │  │  Reports    │
│ (Treaties, │   │ (Chart of     │   │ (Users, Roles,     │  │ (Balance    │
│ MGA, LOB,  │   │ Accounts,     │   │ Activity Logs)     │  │ Sheet, P&L, │
│ States...) │   │ Journal       │   │                    │  │ Test        │
│            │   │ Entries...)   │   │                    │  │ Balance)    │
└───────────┘   └───────────────┘   └──────────────────┘  └─────────────┘
```

- **Dashboard** — login ke baad pehli screen, overview dikhata hai.
- **Masters** — company ki base jaankari rakhne ki jagah: Treaties, MGA, Reinsurers, Brokers, States, LOB/COB, Products, GL Mapping waghera.
- **Accounting** — Chart of Accounts, Journal Entries, Reinsurance Calculations, Test Balance.
- **User Management** — kaun-kaun is app ko use karta hai (Users), unke Roles & Permissions, aur sabki Activity Logs (kisne kab kya kiya).
- **Reports** — Balance Sheet, Profit & Loss jaisi summary reports.

Har ek module alag-alag permission se control hota hai — yaani kisi ko sirf Masters dikh sakta hai, kisi ko sirf Accounting.

---

## 5. Roz ka kaam — Create / Edit / Delete kaise hota hai

```
   User kisi cheez ko Save/Update/Delete karne ki koshish karta hai
                         │
                         ▼
        Kya uske paas us kaam ki permission hai?
                    /            \
                 Haan             Nahi
                  │                 │
                  ▼                 ▼
         ✅ Kaam ho jata hai   ❌ Error aata hai
         (data save/update/     ("Aapke paas yeh
          delete ho jata hai)    karne ki permission
                                  nahi hai")
```

- Har form me jo bhi field validation lagi hai (jaise sahi email hona, sahi phone number hona), woh pehle check hoti hai — agar kuch galat bhara hai toh laal rang me turant bata diya jata hai ki kya theek karna hai.
- Sab kuch sahi hone par hi data server par jata hai, wahan phir se permission check hokar save hota hai.

---

## 6. Treaty kya hota hai (Business ka core setup)

Treaty ek **contract/agreement** hota hai — jisme yeh tay hota hai ki kaunsi Insurance Company (Carrier), kaunsa MGA, aur kaunse Reinsurer(s) mil kar kis business (States, Products) par kaam karenge, aur profit/loss kis ratio (%) me baatega.

```
   Treaty banate waqt yeh sab jodna padta hai:

   ┌────────┐   ┌─────────┐   ┌───────────────┐   ┌──────────┐   ┌──────────┐
   │  MGA   │ + │ Carrier │ + │ Reinsurer(s)  │ + │ States   │ + │ Products │
   │ (kaam  │   │ (risk   │   │ (risk-sharing │   │ (kahan   │   │ (kaunsa  │
   │ karne  │   │ uthane  │   │  partner, %   │   │  business│   │ business:│
   │ wala)  │   │  wala)  │   │  share tay)   │   │  chalega)│   │ LOB+COB) │
   └────────┘   └─────────┘   └───────────────┘   └──────────┘   └──────────┘
                          │
                          ▼
                  Ek complete "Treaty" ban jata hai
                  (apna code, effective date, %
                   ratios — Quota Share, Commission,
                   Ceding Fee, etc. — sab is treaty
                   ke andar set hote hain)
```

- Yeh sab **Masters** se pehle se bane hue records hote hain (MGA Masters me MGA, Reinsurer Masters me Reinsurer, waghera) — Treaty banate waqt bas unko select/jodna hota hai.
- Ek baar Treaty ban jaye, toh usi ke against har mahine ka accounting/reinsurance calculation hota hai (neeche section 7).

---

## 7. Har Mahine ka Kaam — Excel Upload se Final Entry tak

Yeh is poore application ka **sabse important, roz/mahine ka real kaam** hai — isi ke liye sab kuch bana hai.

```
 1. Excel Workbook Upload
    (state-wise premium, claims, reserves ka data)
             │
             ▼
 2. System khud calculate karta hai
    (Har state ke liye numbers nikalta hai — kitna
     premium, kitna commission, kitna claim, etc.)
             │
             ▼
 3. Draft ban jata hai (abhi FINAL nahi hai)
    ┌─────────────────────┐   ┌───────────────────────┐
    │ Journal Entry Draft │   │ Cash Settlement Draft  │
    │ (accounting entries)│   │ (Reinsurer ko kitna    │
    │                      │   │  dena/lena hai)        │
    └─────────────────────┘   └───────────────────────┘
             │
             ▼
 4. Authorized person check karta hai, chahe toh
    galti sudhar sakta hai (status: "Pending Review")
             │
             ▼
 5. Approve karta hai
             │
             ▼
 6. FINAL ho jata hai — ab yeh:
    - Journal Entries list me sthayi (permanent) dikhega
    - Cash Settlement report me confirm ho jayega
    - Financial Reports (Balance Sheet, P&L, Test Balance)
      me count hoga
```

- Jab tak "Pending Review" hai, tab tak edit/correct kiya ja sakta hai.
- **Approve karne ke baad woh entry badli nahi ja sakti** — asli accounting record ban jati hai, permanent record ke liye rakhi jati hai.
- Yeh poora calculation Treaty ke andar set kiye gaye % ratios (Quota Share, Commission, Ceding Fee, etc.) ke hisab se hota hai.

---

## 8. Ek nazar me — Poora Flow (Login se Final Report tak)

```
Login (Email + OTP)
      │
      ▼
Role pata chalta hai ──▶ Permissions pata chalti hain ──▶ Sidebar sajta hai
      │
      ▼
User apne module me jata hai (Masters / Accounting / User Management...)
      │
      ▼
Masters me MGA, Carrier, Reinsurer, States, Products ready kiye jaate hain
      │
      ▼
Inko jod kar ek Treaty banti hai (% ratios ke saath)
      │
      ▼
Har mahine Excel Workbook upload hota hai us Treaty ke liye
      │
      ▼
System Draft Journal Entry + Cash Settlement bana deta hai
      │
      ▼
Authorized user review/correct karta hai (Pending Review)
      │
      ▼
Approve hone par FINAL Journal Entry + Cash Settlement ban jata hai
      │
      ▼
Financial Reports (Balance Sheet, P&L, Test Balance) me yeh sab dikhta hai
      │
      ▼
Har action Activity Log me record ho jata hai (kisne, kab, kya kiya)
```

---

## 9. Technical Flow — Jab bhi koi API hit hoti hai, backend me kya hota hai

Yeh section thoda technical hai — batata hai ki jab user koi bhi button dabata hai (Save, Delete, page load, waghera), tab frontend se backend tak request kaise jaati hai, aur backend usko kin-kin steps se guzaarta hai pehle response wapas bhejne se.

```
 FRONTEND (Browser)
      │
      │  User "Save" dabata hai
      ▼
 HTTP Request bheji jaati hai
 (URL + Method jaise POST/PATCH/DELETE + data
  + Header me: "Authorization: Bearer <session_token>")
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│                     BACKEND (Server)                     │
│                                                           │
│  STEP 1 — AuthGuard (Login check)                        │
│  ├─ Token hai bhi ya nahi?                                │
│  ├─ Token valid/expired toh nahi?                         │
│  ├─ User ka account Active hai?                           │
│  └─ Uska Role Active hai?                                 │
│         │ Fail ──▶ 401 Unauthorized ("Login required")    │
│         │ Pass                                            │
│         ▼                                                 │
│  STEP 2 — PermissionsGuard (Permission check)             │
│  ├─ Is action ke liye kaunsi permission chahiye           │
│  │   (jaise "treaty.edit", "user.delete")?                │
│  ├─ Kya is user/role ke paas woh permission hai?          │
│  │   (Role ki permission + user ki khud ki extra/         │
│  │    hati hui permission dono milakar check hota hai)    │
│  └─ Superadmin ho toh sab kuch allow                       │
│         │ Fail ──▶ 403 Forbidden ("Aapke paas access nahi")│
│         │ Pass                                            │
│         ▼                                                 │
│  STEP 3 — Data Validation                                 │
│  ├─ Bheja gaya data sahi format me hai?                    │
│  │   (jaise email sahi email jaisa dikhta hai, koi         │
│  │    zaroori field khali toh nahi)                        │
│         │ Fail ──▶ 400 Bad Request ("Yeh field galat hai") │
│         │ Pass                                            │
│         ▼                                                 │
│  STEP 4 — Business Logic (Service)                        │
│  ├─ Extra rules check hote hain, jaise:                    │
│  │   "khud ko deactivate nahi kar sakte"                   │
│  │   "last superadmin ko delete nahi kar sakte"            │
│  │   "system role delete nahi ho sakta"                    │
│         │ Fail ──▶ 400/403 (reason ke saath)               │
│         │ Pass                                            │
│         ▼                                                 │
│  STEP 5 — Database                                         │
│  └─ Data save/update/delete hota hai                       │
│         │                                                  │
│         ▼                                                  │
│  STEP 6 — Activity Log                                     │
│  └─ "Kisne, kab, kya kiya" record ho jata hai              │
└─────────────────────────────────────────────────────────┘
      │
      ▼
 Response wapas frontend ko jaati hai
 (Success → data return hota hai, toast "Saved!" dikhta hai)
 (Error → jo bhi step fail hua uska reason dikhta hai)
```

**Zaroori baatein:**
- Yeh **saare 6 steps HAR ek request** par hote hain — chahe wo list dekhna ho, naya record banana ho, ya delete karna ho. Koi shortcut nahi hai.
- Sirf frontend par button chhupana kaafi nahi hai — is diagram ka STEP 2 (`PermissionsGuard`) server par bhi zaroor check hota hai. Isliye koi bhi seedha URL/API ko directly hit karke bhi permission bypass nahi kar sakta.
- STEP 1 aur STEP 2 ka result thodi der ke liye "yaad" (cache) rakha jata hai taaki har request par baar-baar poora database na khangalna pade — lekin agar permission badal jaaye, toh agli login/refresh par yeh cache apne aap saaf ho jata hai.

---

## Sankshep me (Summary)

| Kya | Kaise |
|---|---|
| App me ghusna | Email + Password + OTP |
| Kisko kya dikhega | Uske Role aur Permission par nirbhar |
| Koi permission badle toh | Agli login par automatically sidebar update |
| Bina permission kuch karne ki koshish | Server khud rok deta hai |
| Treaty kya hai | MGA + Carrier + Reinsurer + States + Products ka combined agreement, apne % ratios ke saath |
| Mahine ka data kaise aata hai | Excel Workbook upload se |
| Accounting entry kab "pakki" hoti hai | Sirf Approve karne ke baad — usse pehle "Pending Review" me correct ho sakti hai |
| Sab kuch track hota hai | Activity Logs me har action save hota hai |
