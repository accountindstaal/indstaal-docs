# Indstaal Application Flow – Roles & Actions by Module

A complete guide to how the Indstaal application works, including all roles, modules, actions, and workflows.

---

## Roles Overview

| Role           | Description                                                                                                                              |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **SuperAdmin** | Full access to all modules; can perform any action; manages users and settings                                                           |
| **Sales**      | Creates inquiries, QRFs; publishes QRFs; auto-assigns designer; updates Steel BOQ category when PENDING_SALES (assigned); publishes estimations; generates proposals |
| **Designer**   | Edits assigned QRFs; creates estimations; submits estimations to Admin; can reassign QRF internally                                      |
| **Purchase**   | Manages static price config: Rate (₹/MT) only; adds items and subcategories; does NOT see QRF or Estimation                              |
| **Admin**      | Same as SuperAdmin (publish estimation, manage users)                                                                                    |

---

## 1. DASHBOARD

| Role           | Access                               | Actions                                  |
| -------------- | ------------------------------------ | ---------------------------------------- |
| **SuperAdmin** | All inquiries, all QRFs, total users | View stats, recent items, role breakdown, Active Users |
| **Sales**      | Own inquiries, own/assigned QRFs     | View stats for their data (no Active Users) |
| **Designer**   | Own/assigned QRFs                    | View stats for their data (no Active Users) |
| **Purchase**   | Price Config only                    | View **Price Config** card only; no Active Users, no inquiry/QRF stats |

**Notes:**

- **Purchase**: Dashboard shows only the Price Config card; no inquiry/QRF statistics or Active Users.
- **Sales/Designer**: See inquiry/QRF stats; no Active Users.
- **SuperAdmin**: Full dashboard including Active Users, Users by role, and all stats.

---

## 2. INQUIRY MODULE

**Flow:** Create Inquiry → (Optional) Create QRF from Inquiry

| Role           | List | Create | Update | Delete | Create QRF from Inquiry |
| -------------- | ---- | ------ | ------ | ------ | ----------------------- |
| **SuperAdmin** | All  | ✓      | All    | ✓      | ✓                       |
| **Sales**      | Own  | ✓      | Own    | ✗      | ✓ (own inquiries)       |
| **Designer**   | ✗    | ✗      | ✗      | ✗      | ✗                       |
| **Purchase**   | ✗    | ✗      | ✗      | ✗      | ✗                       |

**Notes:**

- Inquiry list: SuperAdmin sees all; others see only their own (`created_by`)
- Inquiry create: Sales only (`IsSalesUser`)
- Inquiry delete: SuperAdmin only; not allowed if QRF exists
- QRF creation from Inquiry: Any authenticated user with access to the inquiry (SuperAdmin: any; others: own)

---

## 3. QRF MODULE

**Flow:** Create QRF (from Inquiry) → Edit → Publish → **Auto-assign to default Designer** → Create Revision

### 3.1 QRF List & Detail

| Role           | List              | Detail           |
| -------------- | ----------------- | ---------------- |
| **SuperAdmin** | All QRFs          | All              |
| **Sales**      | Own + assigned    | Own + assigned   |
| **Designer**   | Assigned to them  | Assigned to them |
| **Purchase**   | ✗ (no QRF access) | ✗                |

### 3.2 QRF Actions

| Action                       | SuperAdmin | Sales           | Designer                         | Purchase |
| ---------------------------- | ---------- | --------------- | -------------------------------- | -------- |
| **Create QRF from Inquiry**  | ✓          | ✓ (own inquiry) | ✗                                | ✗        |
| **Edit QRF**                 | ✓          | ✓               | ✓ (assigned)                     | ✗        |
| **Publish QRF**              | ✓          | ✓               | ✗                                | ✗        |
| **Assign/Reassign Designer** | ✓          | ✓               | ✓ (reassign only, when assigned) | ✗        |
| **Create Revision**          | ✓          | ✓               | ✗                                | ✗        |
| **Delete QRF**               | ✓          | ✗               | ✗                                | ✗        |
| **Generate QRF PDF**         | ✓          | ✓               | ✓ (assigned)                     | ✗        |
| **View Documents**           | ✓          | ✓               | ✓ (assigned)                     | ✗        |

**QRF Status Flow:** DRAFT → PUBLISHED

### 3.3 Auto-Assign on Publish

When Sales or SuperAdmin **publishes** a QRF:

- If no designer is assigned, the QRF is **automatically assigned** to the default designer
- Default designer: configured via `DEFAULT_DESIGNER_EMAIL` (env, default: `designer@example.com`)
- Must be an active user in the Designer group

### 3.4 Designer Reassignment

- **Sales/SuperAdmin**: Can assign or reassign any Designer at any time
- **Designer**: Can **reassign** the QRF to another Designer **only when they are the current assignee**
- Target must be in the Designer role

### 3.5 QRF Documents

- Documents can be uploaded per QRF (PDF, Excel, Word, Images, max 2GB via multipart upload)
- View documents from QRF Revisions page (Documents icon)
- Upload allowed when QRF is DRAFT and user can edit

---

## 4. ESTIMATION MODULE

**Flow:** Create Estimation (rates from Price Config) → Designer submits to Admin → Admin adds conversion & submits to Sales → Sales publishes

### 4.1 Estimation Status Flow

```
DRAFT → PENDING_ADMIN → PENDING_SALES → PUBLISHED
  (Designer submit)    (Admin submit)   (Sales/Admin publish)
```

- **DRAFT**: Designer creates and edits; rates from Price Config
- **PENDING_ADMIN**: Designer submitted; Admin sees rates & conversion, can update conversion
- **PENDING_SALES**: Admin submitted to Sales; assigned to QRF's Inquiry sales person; Sales can publish
- **PUBLISHED**: Sales (or Admin) published; proposal can be generated

### 4.2 Estimation List & Detail

| Role           | List                                                   | Detail             |
| -------------- | ------------------------------------------------------ | ------------------ |
| **SuperAdmin** | All                                                    | All                |
| **Sales**      | PUBLISHED (own QRF) + PENDING_SALES (assigned to them) | Same               |
| **Designer**   | Own + assigned QRF                                     | Own + assigned QRF |
| **Purchase**   | ✗ (no Estimation access)                               | ✗                  |

### 4.3 Estimation Actions by Status

| Action                        | Who                                       | When                                                                      |
| ----------------------------- | ----------------------------------------- | ------------------------------------------------------------------------- |
| **Create Estimation**         | SuperAdmin, Designer                      | From QRF Revisions (published revision); rates come from Price Config     |
| **Submit to Admin**           | SuperAdmin, Designer                      | DRAFT only                                                                |
| **Add/Edit Conversion Rates** | SuperAdmin (Admin)                        | PENDING_ADMIN                                                             |
| **Submit to Sales**           | SuperAdmin (Admin)                        | PENDING_ADMIN; assigns to QRF's Inquiry sales person (no selection popup) |
| **Update Steel BOQ Category** | SuperAdmin, Sales (assigned only)         | PENDING_SALES; pencil icon opens Update Category popup; select category/subcategory from Price Config |
| **Publish**                   | SuperAdmin (Admin), Sales (assigned only) | PENDING_SALES; discount popup shown before publish (optional discount %)  |
| **Delete**                    | SuperAdmin                                | Not PUBLISHED                                                             |
| **Generate Proposal**         | SuperAdmin, Sales                         | PUBLISHED only                                                            |

### 4.4 Steel BOQ View by Role

| Role                                                | Rate (₹/MT)             | Conversion (₹/MT) | Rate × Qty | Conversion × Qty | Total Price         |
| --------------------------------------------------- | ----------------------- | ----------------- | ---------- | ---------------- | ------------------- |
| **SuperAdmin** (editing PENDING_ADMIN)              | Read-only (from config) | Editable          | ✓          | ✓                | —                   |
| **SuperAdmin** (viewing PENDING_SALES or PUBLISHED) | ✓                       | ✓ (read-only)     | ✓          | ✓                | —                   |
| **Sales** (PENDING_SALES or PUBLISHED)              | ✓                       | —                 | —          | —                | ✓ (Rate × Qty only) |
| **Designer**                                        | —                       | —                 | —          | —                | —                   |

**Notes:**

- **Sales**: Sees only Rate (₹/MT) and one Total Price column (Quantity × Rate). No conversion rate.
- **SuperAdmin**: Sees both Rate and Conversion. When editing PENDING_ADMIN: conversion is editable; when viewing PENDING_SALES or PUBLISHED: conversion is read-only. Two separate totals: Rate × Qty and Conversion × Qty.

### 4.5 Steel BOQ Creation & Update Category (PENDING_SALES)

**At Estimation Creation:**

- Steel BOQ is populated from Price Config with **categories only** (no subcategories).
- Categories = items where `parent_code` is null or empty (e.g. code "1", "2", "3").

**Update Category (PENDING_SALES):**

- **Who can use**: SuperAdmin and Sales (assigned to the estimation) when status is PENDING_SALES.
- **How**: Pencil icon in the last column opens an **Update Category** popup.
- **Popup**: Lists category and subcategories from Price Config; user selects one to update that row only (code, name, rate, quantity preserved).
- **Admin view**: SuperAdmin sees **Rate** and **Conversion** separately (e.g. "Rate ₹12.00 + Conversion ₹22.00 = ₹34.00/MT"). On selection, both `rate_per_mt` and `conversion_rate` are updated from Price Config.
- **Sales view**: Sales sees **Final Rate** only (combined rate + conversion). On selection, the combined rate is applied; backend uses Price Config for rate and conversion (ignores payload).
- **Add Row**: Same popup behavior when adding a new Steel BOQ row from Price Config.

**Price Config API:**

- **Admin**: Receives `rate_per_mt` and `conversion_rate` separately.
- **Non-Admin** (Sales, Designer): Receives `final_rate` (rate + conversion); `conversion_rate` is hidden.

### 4.6 Publish with Discount

When **Sales** or **SuperAdmin** publishes a PENDING_SALES estimation, a discount popup appears before finalizing.

| Role           | Popup Content                                         | Discount Applied To    |
| -------------- | ----------------------------------------------------- | ---------------------- |
| **Sales**      | Total (Steel BOQ) = Rate total only (Quantity × Rate) | Single total           |
| **SuperAdmin** | Rate total, Conversion total (no combined total)      | Both, shown separately |

**Sales popup:** Total (Steel BOQ), discount %, Price after discount.

**SuperAdmin popup:** Rate total, Conversion total, discount %, Rate after discount, Conversion after discount (no combined total, no discount amount in red).

Quantities use breakdown totals by code when available (from takeoff); otherwise fall back to `quantity_mt` on each row. The discount percentage is persisted in the estimation revision details and shown in the estimation basic info after publish.

### 4.7 Estimation Revisions List (per QRF)

| Action                | SuperAdmin        | Sales                            | Designer                   | Purchase |
| --------------------- | ----------------- | -------------------------------- | -------------------------- | -------- |
| **Create Estimation** | ✓                 | ✗                                | ✓ (from published QRF rev) | ✗        |
| **Submit to Admin**   | ✓ (DRAFT)         | ✗                                | ✓ (DRAFT)                  | ✗        |
| **Submit to Sales**   | ✓ (PENDING_ADMIN) | ✗                                | ✗                          | ✗        |
| **Update Steel BOQ**  | ✓ (PENDING_SALES) | ✓ (PENDING_SALES, assigned only) | ✗                          | ✗        |
| **Publish**           | ✓ (PENDING_SALES) | ✓ (PENDING_SALES, assigned only) | ✗                          | ✗        |
| **Generate Proposal** | ✓ (PUBLISHED)     | ✓ (PUBLISHED)                    | ✗                          | ✗        |
| **Delete**            | ✓ (not PUBLISHED) | ✗                                | ✗                          | ✗        |
| **View**              | ✓                 | ✓ (own/PENDING_SALES assigned)   | ✓                          | ✗        |

---

## 5. PROPOSAL MODULE

| Role           | Generate Proposal PDF                                   |
| -------------- | ------------------------------------------------------- |
| **SuperAdmin** | ✓ (any PUBLISHED estimation)                            |
| **Sales**      | ✓ (PUBLISHED estimations whose QRF was created by them) |
| **Designer**   | ✗                                                       |
| **Purchase**   | ✗                                                       |

**Notes:** Proposal is generated from PUBLISHED estimations only. Sales see proposals for their own QRFs and for estimations they published from PENDING_SALES.

---

## 6. SETTINGS MODULE

| Role           | Users       | Activity Logs | Invite |
| -------------- | ----------- | ------------- | ------ |
| **SuperAdmin** | Full access | All           | ✓      |
| **Sales**      | ✗           | Own           | ✗      |
| **Designer**   | ✗           | Own           | ✗      |
| **Purchase**   | ✗           | Own           | ✗      |

**Notes:** SuperAdmin sees all; others see only their own activity logs.

---

## 7. END-TO-END WORKFLOW

### 7.1 Sales Flow

1. Create Inquiry
2. Create QRF from Inquiry
3. Edit QRF (or wait for Designer)
4. **Publish QRF** → QRF is **auto-assigned** to default Designer
5. (Optional) Manually assign/reassign Designer if needed
6. (Designer creates estimation, submits to Admin)
7. (Admin adds conversion, submits to Sales) → Estimation moves to **PENDING_SALES** assigned to this Sales user
8. **Update Steel BOQ Category** (optional): Pencil icon opens popup; Sales sees Final Rate; selection updates row from Price Config
9. **Publish** PENDING_SALES estimation → discount popup (optional %) → PUBLISHED
10. Generate proposal for PUBLISHED estimation

### 7.2 Designer Flow

1. View assigned QRFs (auto-assigned on publish or manually assigned)
2. Edit QRF
3. (Optional) **Reassign** QRF to another Designer internally
4. Create Estimation from published QRF revision (rates come from Price Config; Steel BOQ has categories only, no subcategories)
5. Submit DRAFT estimation to Admin
6. (Admin adds conversion, submits to Sales; Sales publishes) → Designer can view

### 7.3 Purchase Flow

1. Manage **Price Config** (Steel BOQ codes, rates, subcategories)
   - Add/edit **Rate (₹/MT)** only
   - Add items and subcategories (e.g. code 1, subcategory 1.1)
   - Cannot edit conversion rate (Admin only)
2. Does NOT see QRF or Estimation

### 7.4 Admin (SuperAdmin) Flow

1. Manage **Price Config** (Steel BOQ conversion rates)
   - Add/edit **Conversion (₹/MT)** only
   - Add items and subcategories
   - Cannot edit rate (Purchase only)
2. Add Conversion Rates on estimation (PENDING_ADMIN)
3. **Submit to Sales** (PENDING_ADMIN) → assigns to QRF's Inquiry sales person; moves to PENDING_SALES
4. **Update Steel BOQ Category** (PENDING_SALES): Pencil icon opens popup; Admin sees Rate + Conversion separately; selection updates both `rate_per_mt` and `conversion_rate` from Price Config
5. Publish estimation (from PENDING_SALES; discount popup with Rate + Conversion totals; or directly from PENDING_ADMIN if needed)
6. Manage users, settings, delete inquiries/estimations

---

## 8. NAVIGATION VISIBILITY

| Module       | SuperAdmin                 | Sales | Designer | Purchase                |
| ------------ | -------------------------- | ----- | -------- | ----------------------- |
| Dashboard    | ✓                          | ✓     | ✓        | ✓                       |
| Inquiry      | ✓                          | ✓     | ✗        | ✗                       |
| QRF          | ✓                          | ✓     | ✓        | ✗                       |
| Estimation   | ✓                          | ✓     | ✓        | ✗                       |
| Price Config | ✓ (Admin: conversion only) | ✗     | ✗        | ✓ (Purchase: rate only) |
| Settings     | ✓                          | ✓\*   | ✓\*      | ✓\*                     |

\*Sales/Designer/Purchase see Settings (Activity Logs or Users based on permission).

---

## 9. DATA ACCESS SUMMARY

| User Type      | Inquiries | QRFs             | Estimations                                            |
| -------------- | --------- | ---------------- | ------------------------------------------------------ |
| **SuperAdmin** | All       | All              | All                                                    |
| **Sales**      | Own       | Own + assigned   | PUBLISHED (own QRF) + PENDING_SALES (assigned to them) |
| **Designer**   | —         | Assigned to them | Own + assigned QRF                                     |
| **Purchase**   | —         | ✗                | ✗ (manages Price Config only)                          |

---

## 10. AUTHENTICATION & SESSION

| Setting           | Value                                   |
| ----------------- | --------------------------------------- |
| **Access Token**  | 1 hour (configurable in settings)       |
| **Refresh Token** | 7 days                                  |
| **Logout**        | After 7 days inactive, or manual logout |

**Single session per user:** A user can be logged in from only one browser/device at a time. When the user logs in from a new browser, all previous sessions are invalidated. The old session will receive 401 on its next API call and be redirected to login.

**401 handling:** On any 401 response, the frontend automatically attempts to refresh the access token using the refresh token (cookie). If refresh succeeds, the original request is retried. If refresh fails (e.g. token blacklisted, expired, or network error), the user is logged out and redirected to the login page. This applies to all API calls including PDF downloads and multipart uploads.

---

## 11. QUICK REFERENCE – WHO CAN DO WHAT

| Action                        |     SuperAdmin      | Sales | Designer |   Purchase    |
| ----------------------------- | :-----------------: | :---: | :------: | :-----------: |
| Create Inquiry                |          ✓          |   ✓   |    ✗     |       ✗       |
| Create QRF                    |          ✓          |   ✓   |    ✗     |       ✗       |
| Edit QRF                      |          ✓          |   ✓   |   ✓\*    |       ✗       |
| Publish QRF                   |          ✓          |   ✓   |    ✗     |       ✗       |
| Assign/Reassign Designer      |          ✓          |   ✓   |  ✓\*\*   |       ✗       |
| Create QRF Revision           |          ✓          |   ✓   |    ✗     |       ✗       |
| Create Estimation             |          ✓          |   ✗   |    ✓     |       ✗       |
| Submit to Admin               |          ✓          |   ✗   |    ✓     |       ✗       |
| Submit to Sales               |          ✓          |   ✗   |    ✗     |       ✗       |
| Manage Price Config           | ✓ (conversion only) |   ✗   |    ✗     | ✓ (rate only) |
| Add Conversion Rates          |          ✓          |   ✗   |    ✗     |       ✗       |
| Update Steel BOQ Category    |          ✓          |  ✓†   |    ✗     |       ✗       |
| Publish Estimation            |          ✓          |  ✓†   |    ✗     |       ✗       |
| Generate Proposal             |          ✓          |   ✓   |    ✗     |       ✗       |
| Delete Inquiry/QRF/Estimation |          ✓          |   ✗   |    ✗     |       ✗       |
| Manage Users                  |          ✓          |   ✗   |    ✗     |       ✗       |

\*Designer: only QRFs assigned to them  
\*\*Designer: only when they are the current assignee (reassign to another Designer)  
†Sales: only when PENDING_SALES and assigned to them
