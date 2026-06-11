# ZGATPDB — Old `NT_MT` vs New `NT_MT_NEW`: KL stock strategy missing in popup

**Program:** `ZAPO_GATP_ALLOCATION_REPORT`  
**Include:** `ZAPO_GATP_ALLOCATION_F005`  
**Table:** `ZAPOPARAM`  
**Popup:** gATP Manual & NO Touch Report (`get_gatprep_data` → `disp_mtch_ntch_per`)  
**Stock strategy:** `ZAPO_SO_LIST-REQ_NT = 'KL'`  
**Version:** 1.0 — 10/06/2026  

---

## 1. `ZAPOPARAM` settings (from screenshot)

| Setting | PARAM1 | PARAM2 | PARAM3 | PARAM4 | PARAM5 | VALUE1 (Division) | ACTIVE_FLAG |
|---------|--------|--------|--------|--------|--------|-------------------|-------------|
| **Old NT-MT** | `GATP` | `NT_MT` | `DIVISION` | `REPORT` | 1 / 2 / 3 / 4 | 22 / 23 / 24 / 37 | **X** (22–24); blank (37) |
| **New NT-MT** | `GATP` | `NT_MT_NEW` | `DIVISION` | `REPORT_NEW` | 1 / 2 / 3 | 22 / 23 / 24 | **Must be X** when enabling new logic |

**Switching behaviour**

- **Old active:** `NT_MT` + `REPORT` rows flagged **X**; `NT_MT_NEW` rows **not** flagged.
- **New active:** `NT_MT_NEW` + `REPORT_NEW` rows flagged **X**; `NT_MT` rows **not** flagged.

---

## 2. How the program decides Old vs New

### 2.1 `get_param_data` (only loads **new** param)

```abap
METHOD get_param_data.
  IF so_divi[] IS NOT INITIAL.
    SELECT param1 param2 param3 param4 param5 value1
      FROM zapoparam
      INTO TABLE gt_zapoparam
      WHERE mandt       = sy-mandt
        AND param1      = 'GATP'
        AND param2      = 'NT_MT_NEW'      " ← only NEW rows
        AND param3      = 'DIVISION'
        AND param4      = 'REPORT_NEW'
        AND value1      IN so_divi[]
        AND active_flag = abap_true.
    IF sy-subrc <> 0.
      REFRESH gt_zapoparam.              " ← empty table
    ENDIF.
  ENDIF.
ENDMETHOD.
```

| Active config | `gt_zapoparam` after `get_param_data` | Effective mode |
|---------------|----------------------------------------|----------------|
| Old `NT_MT` only | **Empty** (no `NT_MT_NEW` rows) | **Legacy / Old** |
| New `NT_MT_NEW` only | **Filled** (divisions 22/23/24 in selection) | **New CHANGE F** |

> **Important:** Old `NT_MT` rows are **never** read into `gt_zapoparam`. Legacy mode is inferred because the table is **empty**, not because `NT_MT` is active.

### 2.2 Global switch used everywhere

Almost all Old/New branching uses:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0.
  " NEW param path (NT_MT_NEW active for selected division)
ELSE.
  " OLD param path (legacy)
ENDIF.
```

---

## 3. Side-by-side: Old vs New code paths (KL / popup impact)

| Step | Method | **Old `NT_MT`** (`gt_zapoparam` empty) | **New `NT_MT_NEW`** (`gt_zapoparam` filled) | KL stock strategy impact |
|------|--------|----------------------------------------|---------------------------------------------|--------------------------|
| A | `get_data` — zero PAG row (`AEMENGE=0`) | Skip row if `KCQTY` also 0 | **Keep** row when `gt_zapoparam` exists | New may show PAG shell; KL qty still missing in `no_touch` |
| B | `get_data` — CHANGE F MT/NT | **Legacy** formula (`alloc_quan`, `p_usr_adj`, …) | **BR1–BR5:** `manual_touch` from `ZAPO_ADB_ADJ`; `no_touch = inc_ord_quan − manual_touch` | New **does not** read `REQ_NT='KL'` SO bucket for `no_touch` |
| C | `get_data` — CHANGE C `lt_so_cap` | Month SO cap incl. `REQ_NT='KL'` | Same SELECT (both modes) | Cap data exists in new mode but not used for popup NT |
| D | `collect_final_output` | **Runs** `SELECT … req_nt = 'KL'` merge into `lt_output` | **Skipped** — entire block inside `IF sy-subrc <> 0` | **Root gap #1:** KL-only CVCs never added under new param |
| E | `get_zapo_so_list` — date | Today (`sy-datum`) | Month start → month end (CHANGE H) | New uses wider EDATU window (OK) |
| F | `get_zapo_so_list` — matloc | `DELETE lt_matloc WHERE mtvfp = 'KP'` then only matloc hits kept | **Same** | **Root gap #2:** KL materials (`MTVFP=KP`) dropped from `gt_apo_so_list` |
| G | `get_zapo_so_list` — NT split | `DELETE … WHERE req_nt NE 'KL'` | **Same** | Correct filter; useless if step F already removed lines |
| H | `get_gatprep_data` popup | `grid no_touch` **+** `gw_apo_so_nt_*` (additive) | Same additive call, but grid `no_touch` often **0** and SO add **0** (step F) | Popup NT = **0** or row missing |
| I | `pull_data_prime_buck` (mail MTD) | Adds `ZAPO_SO_LIST` `REQ_NT/REQ_MT='KL'` to month totals | **Skipped** (`IF sy-subrc <> 0` only) | MTD mail also loses KL under new param |

---

## 4. Why Old popup **includes** KL stock strategy orders

Under **Old `NT_MT`**:

1. **`gt_zapoparam` is empty** → program stays on legacy branches.
2. **`collect_final_output`** (~3476) executes KL merge:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc <> 0.                              " TRUE for Old param
  SELECT * FROM zapo_so_list
    ...
    AND req_nt = 'KL'.
  " Append missing CVC keys to lt_output / backup
ENDIF.
```

3. **`get_gatprep_data`** always calls **`get_zapo_so_list`** and **adds** SO NT totals:

```abap
go_alloc->get_zapo_so_list( im_output = lt_output ).
gw_ntch_pp_plant = gw_ntch_pp_plant + gw_apo_so_nt_pp_plant.
gw_ntch_tot_plant = gw_ntch_tot_plant + gw_apo_so_nt_tot_plant.
```

4. **Legacy `no_touch`** in `get_data` can still reflect incoming PAG quantity for rows that exist; SO add-on covers remaining KL lines.

**Result:** Popup shows NT-MT quantities including **`REQ_NT = 'KL'`** stock strategy orders.

---

## 5. Why New popup **excludes** KL stock strategy orders

Under **New `NT_MT_NEW`** (`active_flag = X` on `REPORT_NEW` rows):

### Gap 1 — KL CVC merge disabled

Same block as §4 step 2, but `READ TABLE gt_zapoparam …` **succeeds** → KL `SELECT` **never runs**.  
KL-only orders (no PAG / `AEMENGE = 0`) never create an ALV row → user gets **No data found** or cannot open popup.

### Gap 2 — CHANGE F does not use SO KL bucket

New MT/NT (~2456 / 2634):

```abap
<fs_output>-manual_touch = lw_mt_adj_sum-usr_adj.        " UA only
<fs_output>-no_touch     = <fs_output>-inc_ord_quan
                         - <fs_output>-manual_touch.       " NOT REQ_NT='KL'
```

For KL stock strategy: `inc_ord_quan` from PAG is often **0** → `no_touch = 0`.

### Gap 3 — `MTVFP = 'KP'` filter drops KL lines in `get_zapo_so_list`

```abap
DELETE lt_matloc WHERE mtvfp EQ 'KP'.
LOOP AT lt_apo_so_list ...
  READ TABLE lt_matloc ...
  IF sy-subrc EQ 0.
    APPEND gw_apo_so_list TO gt_apo_so_list.   " KL/KP lines never appended
  ENDIF.
ENDLOOP.
```

Even when `get_gatprep_data` adds `gw_apo_so_nt_pp_plant`, the value is **0** because `gt_apo_so_list` is empty for KP materials.

### Gap 4 — Mail / MTD path also skips SO KL add-on

`pull_data_prime_buck` (~3741): SO `REQ_NT='KL'` additive loop runs only when `gt_zapoparam` **empty** (old param). New param skips it entirely.

---

## 6. Required code correction (parity: New = Old KL behaviour)

Apply in **`ZAPO_GATP_ALLOCATION_F005`**. Goal: **New `NT_MT_NEW` must include `REQ_NT='KL'` in ALV + popup** the same way Old did, while keeping CHANGE F UA/MT logic.

---

### 6.1 ADD helper — detect new param per division

```abap
METHOD is_nt_mt_new_active.
  IMPORTING
    iv_div TYPE spart
  RETURNING
    VALUE(rv_active) TYPE abap_bool.

  READ TABLE gt_zapoparam TRANSPORTING NO FIELDS
    WITH KEY param1 = 'GATP'
             param2 = 'NT_MT_NEW'
             value1 = iv_div.
  IF sy-subrc = 0.
    rv_active = abap_true.
  ENDIF.
ENDMETHOD.
```

Call `get_param_data` at report start (already done). Ensure **`NT_MT_NEW` rows have `ACTIVE_FLAG = 'X'`** in `ZAPOPARAM` before testing.

---

### 6.2 CHANGE G — `merge_kl_so_cvc_to_output` (shared Old + New)

Add method (full listing in `ZGATPDB_KL_REQ_NT_No_Data_Popup_Code_Correction.md` §4.1).

**Purpose:** For any CVC with `ZAPO_SO_LIST` lines where **`REQ_NT = 'KL'`** but no PAG row, append a driver row to `gt_output` with:

```text
no_touch     = SUM( bmeng ) WHERE req_nt = 'KL'
inc_ord_quan = same (for display)
manual_touch = 0
```

---

### 6.3 `get_data` — call KL merge for **New param** (after filters ~1034)

```abap
*--- Stock strategy KL: required for NT_MT_NEW (was Old-only via empty gt_zapoparam)
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0.
  merge_kl_so_cvc_to_output( CHANGING ct_output = gt_output ).
ENDIF.
```

> Old param already gets KL rows via `collect_final_output` backup path; this makes **`get_data` / ALV** correct for New before popup.

---

### 6.4 `collect_final_output` — remove inverted guard (~3476)

**Current (wrong for New):**

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc <> 0.
  SELECT ... AND req_nt = 'KL'.
  ...
ENDIF.
```

**Replace with:**

```abap
*--- KL stock strategy CVCs: Old AND New param
merge_kl_so_cvc_to_output( CHANGING ct_output = lt_output ).
```

Delete the old `IF sy-subrc <> 0` wrapper and inline `SELECT` duplicate.

---

### 6.5 `get_zapo_so_list` — keep `REQ_NT='KL'` when `MTVFP='KP'` (~3224)

**Replace matloc append loop:**

```abap
LOOP AT lt_apo_so_list INTO gw_apo_so_list.
  READ TABLE lt_matloc INTO lw_matloc
    WITH KEY matnr       = gw_apo_so_list-matnr
             locno       = gw_apo_so_list-werks
             planner_snp = gw_apo_so_list-spart BINARY SEARCH.
  IF sy-subrc EQ 0.
    APPEND gw_apo_so_list TO gt_apo_so_list.
  ELSEIF gw_apo_so_list-req_nt = 'KL'.
    " Stock strategy (KP): include for NT popup — BR6 excludes KP from cap only
    APPEND gw_apo_so_list TO gt_apo_so_list.
  ENDIF.
  CLEAR gw_apo_so_list.
ENDLOOP.
```

Apply same pattern in **`get_so_list`** if used for NT display.

**Do not** remove `DELETE lt_matloc WHERE mtvfp EQ 'KP'` if other logic needs it; the `ELSEIF req_nt = 'KL'` bypass is sufficient.

---

### 6.6 `get_data` CHANGE F — NT from SO KL bucket when New param (~2532)

Inside `IF sy-subrc = 0` (new param) for each `<fs_output>` row, **after** MT from `ZAPO_ADB_ADJ`:

```abap
*--- NT: stock strategy bucket (REQ_NT = KL) — parity with Old SO add-on
CLEAR ls_so_cap_sum.
READ TABLE lt_so_cap_sum INTO ls_so_cap_sum
  WITH KEY gccode = <fs_output>-grp_cust
           matnr  = <fs_output>-material
           werks  = <fs_output>-location
           vtweg  = <fs_output>-dist_chan
           spart  = <fs_output>-div BINARY SEARCH.
IF sy-subrc = 0.
  " Use pre-aggregated NT from CHANGE C if you add nt_qty to lt_so_cap_sum
  " OR sum lt_so_cap for req_nt = KL only (see §6.7)
ENDIF.

" Minimum fix: set no_touch from SO KL sum, not inc_ord_quan - manual_touch
DATA: lv_so_nt_kl TYPE zbmeng.
CLEAR lv_so_nt_kl.
LOOP AT lt_so_cap INTO lw_so_all
  WHERE gccode = <fs_output>-grp_cust
    AND matnr  = <fs_output>-material
    AND werks  = <fs_output>-location
    AND vtweg  = <fs_output>-dist_chan
    AND spart  = <fs_output>-div
    AND req_nt = 'KL'.
  lv_so_nt_kl = lv_so_nt_kl + lw_so_all-bmeng.
ENDLOOP.

IF lv_so_nt_kl > 0.
  <fs_output>-no_touch = lv_so_nt_kl.
  IF <fs_output>-inc_ord_quan < lv_so_nt_kl.
    <fs_output>-inc_ord_quan = lv_so_nt_kl + <fs_output>-manual_touch.
  ENDIF.
ELSE.
  <fs_output>-no_touch = <fs_output>-inc_ord_quan - <fs_output>-manual_touch.
  IF <fs_output>-no_touch < 0. CLEAR <fs_output>-no_touch. ENDIF.
ENDIF.
```

Repeat in the second CHANGE F block (~2637) and NP block (~2914).

---

### 6.7 `get_gatprep_data` — popup NT for New param (~6226)

After `get_zapo_so_list`:

```abap
go_alloc->get_zapo_so_list( im_output = lt_output ).

READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0.
  " New NT_MT_NEW: NT columns from SO_LIST KL bucket (same as Old additive source)
  gw_ntch_tot_plant  = gw_apo_so_nt_tot_plant.
  gw_ntch_pe_plant   = gw_apo_so_nt_pe_plant.
  gw_ntch_pp_plant   = gw_apo_so_nt_pp_plant.
  gw_ntch_pvc_plant  = gw_apo_so_nt_pvc_plant.
  gw_ntch_elstmr_plant = gw_apo_so_nt_elstmr_plant.
  gw_ntch_tot_depot  = gw_apo_so_nt_tot_depot.
  gw_ntch_pe_depot   = gw_apo_so_nt_pe_depot.
  gw_ntch_pp_depot   = gw_apo_so_nt_pp_depot.
  gw_ntch_pvc_depot  = gw_apo_so_nt_pvc_depot.
  gw_ntch_elstmr_depot = gw_apo_so_nt_elstmr_depot.

  " MT: grid UA + SO MT bucket
  gw_mtch_tot_plant = gw_mtch_tot_plant + gw_apo_so_mt_tot_plant.
  gw_mtch_pp_plant  = gw_mtch_pp_plant  + gw_apo_so_mt_pp_plant.
  " ... PE / PVC / Elastomer / depot
ELSE.
  " Old NT_MT: unchanged additive
  gw_ntch_tot_plant = gw_ntch_tot_plant + gw_apo_so_nt_tot_plant.
  gw_ntch_pp_plant  = gw_ntch_pp_plant  + gw_apo_so_nt_pp_plant.
  ...
ENDIF.
```

**Also** before `get_zapo_so_list`, if `lt_output` is initial:

```abap
IF lt_output IS INITIAL.
  merge_kl_so_cvc_to_output( CHANGING ct_output = lt_output ).
ENDIF.
```

---

### 6.8 `pull_data_prime_buck` — MTD totals for New param (~3741)

Add **ELSE** branch (see `ZGATPDB_MTD_Blank_NewParam_Code_Correction.md`) **and** restore SO KL add-on for new param:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc <> 0.
  " Existing Old: SO_LIST REQ_MT/REQ_NT = KL additive loop
  ...
ELSE.
  " New: aggregate total_mt/total_nt from gt_prime_buk (MTD fix)
  " PLUS optional SO KL add-on if PRIME_BCKU rows lack KL-only CVCs
  merge_kl_so_cvc_to_output( CHANGING ct_output = lt_output_dummy ).
ENDIF.
```

---

### 6.9 Optional — `get_param_data` read both param types (diagnostics only)

For logging / support, optionally load old rows without switching logic:

```abap
" Do NOT use for branching — branching stays on NT_MT_NEW in gt_zapoparam
SELECT ... WHERE param2 = 'NT_MT' AND param4 = 'REPORT' AND active_flag = abap_true.
```

Logic must remain: **only `NT_MT_NEW` + `active_flag = X` fills `gt_zapoparam`**.

---

## 7. `ZAPOPARAM` maintenance checklist

Before UAT on New setting:

| # | Action |
|---|--------|
| 1 | Set **`ACTIVE_FLAG = X`** on `GATP / NT_MT_NEW / DIVISION / REPORT_NEW` for divisions **22, 23, 24** |
| 2 | Clear **`ACTIVE_FLAG`** on `GATP / NT_MT / DIVISION / REPORT` for same divisions |
| 3 | Add **`NT_MT_NEW` row for division 37** (Elastomer) if required — Old had `REPORT` param5=4 / value1=37 |
| 4 | Transport `ZAPOPARAM` + code changes together |

---

## 8. Test plan

| # | Config | Material / CVC | Expected |
|---|--------|----------------|----------|
| T1 | Old `NT_MT` active | `H029SG` / `3925` / Div `23` / GC `5000000617`, `REQ_NT=KL` | Popup NT > 0 (baseline) |
| T2 | New `NT_MT_NEW` active — **before fix** | Same | Popup NT = 0 or No data found |
| T3 | New `NT_MT_NEW` active — **after fix** | Same | Popup NT = sum `BMENG` `REQ_NT=KL` (match T1) |
| T4 | New param | `V_MATLOC-MTVFP = KP` | KL lines in `gt_apo_so_list` |
| T5 | New param | UA released order | MT from `ZAPO_ADB_ADJ`; NT KL unchanged |
| T6 | Toggle | Switch Old ↔ New in `ZAPOPARAM` | No regression on Old path |

**Verification SQL:**

```sql
SELECT matnr, werks, spart, gccode, req_nt, bmeng, edatu
  FROM zapo_so_list
 WHERE req_nt = 'KL'
   AND delind = space
   AND abgru  = space.
```

---

## 9. Transport objects

| Object | Changes |
|--------|---------|
| `ZAPO_GATP_ALLOCATION_F005` | §6.1–6.8 |
| `ZAPOPARAM` (config) | Activate `NT_MT_NEW`; deactivate `NT_MT` |
| Related TOP/class defs | Types for `merge_kl_so_cvc_to_output` if needed |

---

## 10. Summary

| Question | Answer |
|----------|--------|
| Why does Old work? | `gt_zapoparam` empty → KL merge + SO popup add-on + legacy `no_touch` |
| Why does New fail? | `gt_zapoparam` filled → KL merge **skipped**, CHANGE F ignores `REQ_NT='KL'`, **KP** filter drops SO lines |
| Core fix | Run KL merge for **both** params; bypass **KP** filter for `req_nt='KL'`; drive popup NT from **SO KL bucket** under `NT_MT_NEW` |

**Related MDs**

- `ZGATPDB_KL_REQ_NT_No_Data_Popup_Code_Correction.md` — detailed `merge_kl_so_cvc_to_output` ABAP  
- `ZGATPDB_MTD_Blank_NewParam_Code_Correction.md` — MTD `total_*` for new param  
- `ZGATPDB_Popup_NT_Reduced_UA_Order_Code_Correction.md` — independent NT/MT buckets  
- `FS_TS_NT_MT_Report_Revision.md` — BR6: KP excluded from **cap**, not NT display  

---

*End of document*
