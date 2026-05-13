# ZGATPDB — NT (Plant) popup shows 45 MT instead of 30 MT (GP block)

**System checked:** DEVSCMAD1 (ADT source via `ZAPO_GATP_ALLOCATION_F005`)  
**Note:** SAP GUI **ZGATPDB** cannot be executed from this environment; analysis is from live include source. A quick SQL check on `ZAPO_SO_LIST` returned no rows here (client/data/RFC), so **confirm the GP block field on your row** in SE16/`ZAPO_SO_LIST` before coding.

---

## 1. Symptom

- Selection: **Prime Grade’s**, Material **B120MA**, Location **3605**, Division **23**, Group customer **500000617** (verify leading zeros vs **5000000617** in master data).
- **gATP Manual & NO Touch Report** popup: **NT (Plant)** shows **45** in **PMR** / **PP** (and **100%**).
- **Expected:** **30** MT NT; **15** MT must **not** count because the sales order (or schedule line) is in **GP block** (your term — map to the actual field in `ZAPO_SO_LIST` or ECC).

---

## 2. Where the popup number comes from (code)

Popup is built in **`get_gatprep_data`** (class **`LCL_GATP_ALLOC`**, include **`ZAPO_GATP_ALLOCATION_F005`**).

1. **From the ALV grid (`gt_output`)**  
   For division **23**, **`gw_ntch_pp_plant`** is increased by **`gw_output-no_touch`** (plant **1001**).

2. **From sales orders (`get_zapo_so_list`)** — additive on top of (1)  
   After the loop over selected rows, the code does (conceptually):

   - `gw_ntch_tot_plant = gw_ntch_tot_plant + gw_apo_so_nt_tot_plant`
   - `gw_ntch_pp_plant  = gw_ntch_pp_plant  + gw_apo_so_nt_pp_plant`
   - (and analogous for PE, PVC, depot, elastomer)

So **45 = (no_touch from PAG/grid) + (NT portion from `ZAPO_SO_LIST` via `get_zapo_so_list`)**, or a single source is already 45 if SO logic dominates. In your case the **15** MT you attribute to **GP block** is almost certainly still **inside the `ZAPO_SO_LIST` extract** used in **`get_zapo_so_list`**, because that method’s **`SELECT`** does **not** exclude delivery / document blocks that your process calls “GP block”.

---

## 3. Gap in current `get_zapo_so_list` extract

In **`get_zapo_so_list`**, the main driver table is **`lt_apo_so_list`** filled from:

```abap
SELECT ... bmeng req_nt req_mt cmgst zplgclass
  FROM zapo_so_list
  INTO TABLE lt_apo_so_list
  WHERE ...
    AND abgru  = space
    AND edatu  IN so_date
    AND delind = space.
```

**There is no filter on delivery block / GP block** (commonly **`LIFSK`** on sales-related extracts, or a **custom** indicator such as **`SPCODE`** / status on `ZAPO_SO_LIST` — your functional team must confirm which column marks “GP block”).

By contrast, **`get_so_list`** (display SO list elsewhere in the same include) **does** select **`lifsk`** into `lt_disp_so_list`, but that path is **not** the one feeding **`gw_apo_so_nt_*`** for the MT/NT popup.

**Result:** Lines that should be excluded for reporting still contribute **`bmeng`** into **`lt_apo_so_list_nt`** when **`req_nt`** matches **`'KL'`**, inflating **`gw_apo_so_nt_pp_plant`** (and totals).

---

## 4. Recommended fix (choose after confirming the field)

### Step A — Identify the GP block column

On a row that should be excluded, check in **`ZAPO_SO_LIST`** (or source of the extract):

| Candidate | Meaning |
|-----------|--------|
| **`LIFSK`** | Delivery block (classic SD) — often non-blank = blocked |
| **`CMGST`** | Credit status — you already use **`'B'`** elsewhere for credit block |
| **`SPCODE` / custom** | Project-specific “GP” flag |

Until this is confirmed, do not guess in production.

### Step B — Exclude blocked lines in `get_zapo_so_list` (preferred)

**Object:** Include **`ZAPO_GATP_ALLOCATION_F005`**, method **`get_zapo_so_list`**.

1. **Extend** the **`SELECT`** list to include the GP block field(s), e.g. **`lifsk`** (and **`spcode`** if GP is coded there).

2. **Extend** type **`gty_apo_so_list`** (in the **TOP** / type pool where it is defined — often **`ZAPO_GATP_ALLOCATION_CLS`** or report top include) with the same fields.

3. **Filter** in one of these ways (pick one; avoid double filtering):

   - **Open SQL:** add to **`WHERE`**  
     `AND lifsk = @space`  
     (or `AND lifsk NOT IN @lt_gp_block` if some blocks are allowed),
   - **Or** after **`SELECT`:**  
     `DELETE lt_apo_so_list WHERE lifsk IS NOT INITIAL.`  
     (only if business rule is “any delivery block = exclude from NT add-on”).

4. **Optional (flexible):** maintain allowed/blocked **`LIFSK`** values in **`ZAPOPARAM`** (same pattern as division / plgclass ranges) and build **`lt_lifsk_excl`** / **`lt_lifsk_allow`** for the **`WHERE`** clause so Basis can tune without code change.

### Step C — Re-test the popup

- Same selection as production issue.  
- **NT (Plant)** **PP** / **PMR** should drop by the **15** MT that belonged to GP-blocked orders **if** those lines were the only extra contributor.  
- If the number is still high, check whether **`no_touch`** on **`gt_output`** already includes that 15 from PAG — then you may need a **cap** or **reconciliation** rule between PAG and SO list (separate functional design).

---

## 5. Optional: avoid double counting (design check)

If business rule is **“NT plant popup = PAG NT only, do not add SO list BMENG”** for Prime mode, then the fix is **not** `LIFSK` but **skipping** the block that adds **`gw_apo_so_nt_*`** into **`get_gatprep_data`** for that mode. That is a **product decision**; the GP-block issue you described is narrower (**exclude blocked SO lines**).

---

## 6. Files to touch (typical)

| Area | Likely object |
|------|----------------|
| `get_zapo_so_list` SELECT + optional DELETE | **`ZAPO_GATP_ALLOCATION_F005`** |
| Type `gty_apo_so_list` | **TOP / CLS include** for **`ZAPO_GATP_ALLOCATION_REPORT`** (where the type is defined) |
| Param-driven block list | **`ZAPOPARAM`** + small read in **`get_zapo_so_list`** |

---

## 7. Validation checklist

- [ ] One SO line with GP block: **`LIFSK`** (or chosen field) set vs blank.  
- [ ] Before fix: line appears in **`lt_apo_so_list`** / contributes to **`gw_apo_so_nt_pp_plant`**.  
- [ ] After fix: line **excluded**; **NT (Plant)** reduced by blocked **BMENG**.  
- [ ] Regression: credit block **`CMGST = B`** behaviour unchanged if you still need those lines for BR5 elsewhere.

---

*Document version: 1.0 — code references from `ZAPO_GATP_ALLOCATION_F005` on AD1.*
