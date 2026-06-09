# ZGATPDB — KL stock strategy (`REQ_NT = 'KL'`) — popup "No Data Found"

**Program:** `ZAPO_GATP_ALLOCATION_REPORT`  
**Include:** `ZAPO_GATP_ALLOCATION_F005`  
**T-code / dashboard:** `ZGATPDB` / gATP Business On Demand Dashboard Polymer  
**Popup:** `gATP Manual & NO Touch Report` (`get_gatprep_data` → `disp_mtch_ntch_per`)  
**Version:** 1.0 — 09/06/2026  

---

## 1. Symptom (from screenshots)

| Selection | Value |
|-----------|-------|
| Function | Allocation Vs S.O. — Manual & NO Touch |
| Mode | Prime Grade's |
| Material | `H029SG` |
| Location | `3925` |
| Division | `23` |
| Group Customer | `5000000617` |

**Observed**

- Main ALV grid is **empty** or row cannot be used for MT/NT popup.
- Status bar shows **`No data found`** (message `text-005` on popup path, or `'No data found'(003)` on CVC sub-screen).
- Order exists in **`ZAPO_SO_LIST`** with stock strategy **`REQ_NT = 'KL'`** (KL = No Touch / stock-strategy order).
- **Expected:** NT-MT popup should show **NT quantity** from `ZAPO_SO_LIST` for the selected CVC.

**Test CVC:** `H029SG` / `3925` / Div `23` / GC `5000000617` / `REQ_NT = 'KL'`.

---

## 2. Root cause (two defects)

### 2.1 KL-only CVCs are not merged into `gt_output` when new param is ON

In **`collect_final_output`** (~3476), missing CVCs are read from `ZAPO_SO_LIST` and appended only when **`gt_zapoparam` has no `GATP` entry**:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc <> 0.                    " ← OLD param only
  SELECT * FROM zapo_so_list ...
    AND req_nt = 'KL'.
  ...
  APPEND LINES OF gt_fetch_outtab TO lt_output.
ENDIF.
```

**Effect**

- KL stock-strategy orders often have **no PAG quota row** (or `AEMENGE = 0`, `KCQTY = 0`).
- Under **new param** (`ZAPOPARAM` `GATP` active), the merge block is **skipped entirely**.
- `gt_output` stays **empty** → user cannot select a row → **`get_gatprep_data`** raises **`text-005`** (*No data found*).

`collect_final_output` runs on backup; the **same gap** must be fixed in **`get_data`** so the ALV is populated before the popup.

---

### 2.2 `get_zapo_so_list` drops KL lines when `MTVFP = 'KP'`

Stock-strategy materials use **`/SAPAPO/V_MATLOC-MTVFP = 'KP'`**. In **`get_zapo_so_list`** (~3221):

```abap
DELETE lt_matloc WHERE mtvfp EQ 'KP'.
...
READ TABLE lt_matloc ... 
IF sy-subrc EQ 0.
  APPEND gw_apo_so_list TO gt_apo_so_list.
ENDIF.
```

**Effect**

- All `ZAPO_SO_LIST` lines for **`REQ_NT = 'KL'`** on **KP** materials are **removed** from `gt_apo_so_list`.
- Popup NT totals (`gw_apo_so_nt_pp_plant`, etc.) stay **0** even when a grid row exists.

Per FS **BR6**, **KP/SD** must be excluded from **monthly cap** (`lt_so_cap` in CHANGE C), **not** from **NT popup** quantities where **`REQ_NT = 'KL'`**.

---

## 3. Required behaviour

| Area | Rule |
|------|------|
| Grid (`gt_output`) | If a CVC has **`REQ_NT = 'KL'`** orders in `ZAPO_SO_LIST` for the selection but **no PAG row**, still **show the CVC** on the ALV (old **and** new param). |
| NT popup | **`SUM( BMENG )`** from `ZAPO_SO_LIST` where **`REQ_NT = 'KL'`** (plus DLBL/SORG filters from other MDs). **KP `MTVFP` must not block this.** |
| Monthly cap (CHANGE C) | Keep excluding **KP/SD** from **`lt_so_cap`** only — do **not** apply that delete to NT popup path. |

---

## 4. Code corrections

Apply in **`ZAPO_GATP_ALLOCATION_F005`** (class `LCL_GATP_ALLOC`).

---

### 4.1 CHANGE G — New FORM: merge KL CVCs into output table

Add a **private method** (or FORM) callable from **`get_data`** and **`collect_final_output`**:

```abap
METHOD merge_kl_so_cvc_to_output.
  IMPORTING
    it_so_cap_sum TYPE ty_so_cap_sum_tab   " lt_so_cap_sum from get_data
  CHANGING
    ct_output     TYPE gty_output_tab.

  DATA: lt_fetch     TYPE TABLE OF zapo_so_list,
        ls_fetch     TYPE zapo_so_list,
        ls_out       TYPE gty_output,
        ls_cap_sum   TYPE lty_so_cap_sum,
        lv_nt_qty    TYPE zbmeng,
        lv_mt_qty    TYPE zbmeng,
        lrt_edatu    TYPE RANGE OF dats,
        lr_edatu     LIKE LINE OF lrt_edatu,
        lv_mstart    TYPE datum,
        lv_mend      TYPE datum,
        lv_nxt       TYPE datum.

  " Date range: month window for new param, else today..today+2 (legacy)
  READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
  IF sy-subrc = 0.
    CONCATENATE sy-datum+0(6) '01' INTO lv_mstart.
    lv_nxt = lv_mstart.
    IF lv_nxt+4(2) = '12'.
      lv_nxt+4(2) = '01'.
      lv_nxt+0(4) = lv_nxt+0(4) + 1.
    ELSE.
      lv_nxt+4(2) = lv_nxt+4(2) + 1.
    ENDIF.
    lv_nxt+6(2) = '01'.
    lv_mend = lv_nxt - 1.
    lr_edatu-sign = 'I'. lr_edatu-option = 'BT'.
    lr_edatu-low = lv_mstart. lr_edatu-high = lv_mend.
  ELSE.
    lr_edatu-sign = 'I'. lr_edatu-option = 'BT'.
    lr_edatu-low = sy-datum. lr_edatu-high = sy-datum + 2.
  ENDIF.
  APPEND lr_edatu TO lrt_edatu.

  IF so_mat IS INITIAL AND so_loc IS INITIAL
     AND so_divi IS INITIAL AND so_gc IS INITIAL.
    RETURN.
  ENDIF.

  SELECT matnr werks spart vtweg gccode edatu bmeng req_nt req_mt
    FROM zapo_so_list CLIENT SPECIFIED
    INTO TABLE lt_fetch
    WHERE mandt   = sy-mandt
      AND vtweg  IN so_vtweg
      AND spart  IN so_divi
      AND gccode IN so_gc
      AND matnr  IN so_mat
      AND werks  IN so_loc
      AND delind = space
      AND abgru  = space
      AND edatu  IN lrt_edatu
      AND req_nt = 'KL'.

  IF lt_fetch IS INITIAL.
    RETURN.
  ENDIF.

  SORT ct_output BY material location dist_chan div grp_cust.

  LOOP AT lt_fetch INTO ls_fetch.
    READ TABLE ct_output TRANSPORTING NO FIELDS
      WITH KEY material  = ls_fetch-matnr
               location  = ls_fetch-werks
               dist_chan = ls_fetch-vtweg
               div       = ls_fetch-spart
               grp_cust  = ls_fetch-gccode
      BINARY SEARCH.
    IF sy-subrc = 0.
      CONTINUE.    " PAG row already present
    ENDIF.

    CLEAR ls_out.
    ls_out-material  = ls_fetch-matnr.
    ls_out-location  = ls_fetch-werks.
    ls_out-div       = ls_fetch-spart.
    ls_out-dist_chan = ls_fetch-vtweg.
    ls_out-grp_cust  = ls_fetch-gccode.
    ls_out-date      = sy-datum.

    " NT qty for KL strategy (popup + grid)
    CLEAR: lv_nt_qty, lv_mt_qty.
    LOOP AT lt_fetch INTO ls_fetch
      WHERE matnr  = ls_out-material
        AND werks  = ls_out-location
        AND spart  = ls_out-div
        AND vtweg  = ls_out-dist_chan
        AND gccode = ls_out-grp_cust
        AND req_nt = 'KL'.
      lv_nt_qty = lv_nt_qty + ls_fetch-bmeng.
    ENDLOOP.

    ls_out-no_touch     = lv_nt_qty.
    ls_out-inc_ord_quan = lv_nt_qty.
    ls_out-manual_touch = 0.
    ls_out-alloc_quan   = 0.
    ls_out-bal_quan     = 0.

    INSERT ls_out INTO ct_output INDEX sy-tabix.  " keep sort; or APPEND + resort
  ENDLOOP.

  SORT ct_output BY material location dist_chan div grp_cust date.
ENDMETHOD.
```

> **Note:** Replace `ty_so_cap_sum_tab` / `lty_so_cap_sum` with your actual type names from CHANGE C.

---

### 4.2 CHANGE G — Call merge in `get_data` (after selection filters)

**Location:** After division/customer filters (~1034), **before** `IF gt_output IS NOT INITIAL`:

```abap
*--- KL stock strategy: ensure CVC appears when only ZAPO_SO_LIST has REQ_NT = KL
merge_kl_so_cvc_to_output(
  EXPORTING it_so_cap_sum = lt_so_cap_sum
  CHANGING  ct_output     = gt_output ).
```

This fixes **empty grid** for `H029SG` / `3925` / Div `23` / GC `5000000617` under **new param**.

---

### 4.3 CHANGE G — `collect_final_output`: always merge KL CVCs

**Replace** the guarded block (~3476–3527):

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc <> 0.
  ... SELECT req_nt = 'KL' ...
ENDIF.
```

**With:**

```abap
" KL merge required for OLD and NEW param
merge_kl_so_cvc_to_output(
  EXPORTING it_so_cap_sum = lt_so_cap_sum   " pass from get_data via class attr if needed
  CHANGING  ct_output     = lt_output ).
```

If `lt_so_cap_sum` is not in scope here, either:

- store `lt_so_cap_sum` in a **class-level attribute** in `get_data`, or  
- duplicate the lightweight `SELECT req_nt = 'KL'` inside the method (recommended — single source of truth).

---

### 4.4 CHANGE I — `get_zapo_so_list`: do not drop `REQ_NT = 'KL'` for `MTVFP = 'KP'`

**Location:** Loop that builds `gt_apo_so_list` (~3224–3231).

**Current:**

```abap
LOOP AT lt_apo_so_list INTO gw_apo_so_list.
  READ TABLE lt_matloc INTO lw_matloc
    WITH KEY matnr = gw_apo_so_list-matnr
             locno = gw_apo_so_list-werks
             planner_snp = gw_apo_so_list-spart BINARY SEARCH.
  IF sy-subrc EQ 0.
    APPEND gw_apo_so_list TO gt_apo_so_list.
  ENDIF.
ENDLOOP.
```

**Replace with:**

```abap
LOOP AT lt_apo_so_list INTO gw_apo_so_list.
  READ TABLE lt_matloc INTO lw_matloc
    WITH KEY matnr       = gw_apo_so_list-matnr
             locno       = gw_apo_so_list-werks
             planner_snp = gw_apo_so_list-spart BINARY SEARCH.
  IF sy-subrc EQ 0.
    APPEND gw_apo_so_list TO gt_apo_so_list.
  ELSEIF gw_apo_so_list-req_nt = 'KL'.
    " Stock strategy (MTVFP KP): still valid for NT popup — BR6 cap-only exclusion
    APPEND gw_apo_so_list TO gt_apo_so_list.
  ENDIF.
  CLEAR gw_apo_so_list.
ENDLOOP.
```

Apply the **same pattern** in **`get_so_list`** (~5347) if that method feeds any NT display path.

**Do not remove** `DELETE lt_matloc WHERE mtvfp EQ 'KP'` if other logic depends on it; the **`ELSEIF req_nt = 'KL'`** branch bypasses the KP filter for NT-relevant SO lines.

---

### 4.5 CHANGE I — `get_gatprep_data`: allow popup when only SO NT exists

After grid row loop (~6218), before `get_zapo_so_list`:

```abap
IF lt_output IS INITIAL.
  lt_output = gt_output.
ENDIF.

" If still empty but SO KL data exists, build driver from selection
IF lt_output IS INITIAL AND ( so_mat IS NOT INITIAL OR so_loc IS NOT INITIAL ).
  merge_kl_so_cvc_to_output(
    CHANGING ct_output = lt_output ).
ENDIF.
```

Keep existing `get_zapo_so_list` call (~6226). With CHANGE I, **`gw_apo_so_nt_pp_plant`** will receive **BMENG** for Div 23 / PP.

For **new param**, use **SO NT bucket for popup NT columns** (see `ZGATPDB_Popup_NT_Reduced_UA_Order_Code_Correction.md` §4.3 Option A) so KL quantity is not lost when grid `no_touch` is still 0:

```abap
go_alloc->get_zapo_so_list( im_output = lt_output ).

READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0.
  " NT (Plant/Depot + division cols) from SO_LIST — required for REQ_NT = KL / KP materials
  gw_ntch_tot_plant = gw_apo_so_nt_tot_plant.
  gw_ntch_pe_plant  = gw_apo_so_nt_pe_plant.
  gw_ntch_pp_plant  = gw_apo_so_nt_pp_plant.
  gw_ntch_pvc_plant = gw_apo_so_nt_pvc_plant.
  gw_ntch_elstmr_plant = gw_apo_so_nt_elstmr_plant.
  gw_ntch_tot_depot = gw_apo_so_nt_tot_depot.
  gw_ntch_pe_depot  = gw_apo_so_nt_pe_depot.
  gw_ntch_pp_depot  = gw_apo_so_nt_pp_depot.
  gw_ntch_pvc_depot = gw_apo_so_nt_pvc_depot.
  gw_ntch_elstmr_depot = gw_apo_so_nt_elstmr_depot.
  " MT: grid UA + SO MT bucket (unchanged)
  gw_mtch_tot_plant = gw_mtch_tot_plant + gw_apo_so_mt_tot_plant.
  gw_mtch_pp_plant  = gw_mtch_pp_plant  + gw_apo_so_mt_pp_plant.
  " ... other divisions
ELSE.
  " Legacy additive (unchanged)
  gw_ntch_tot_plant = gw_ntch_tot_plant + gw_apo_so_nt_tot_plant.
  gw_ntch_pp_plant  = gw_ntch_pp_plant  + gw_apo_so_nt_pp_plant.
  ...
ENDIF.
```

---

### 4.6 Optional — CHANGE C cap: confirm KP exclusion scope

In **`get_data`** CHANGE C (~537), after building `lt_so_cap`, exclude **KP/SD** from **cap table only** (FS BR6):

```abap
" After SORT lt_so_cap — cap only, not popup
DATA: lt_mtvfp TYPE TABLE OF /sapapo/v_matloc,
      ls_mtvfp TYPE /sapapo/v_matloc.

SELECT matnr locno mtvfp
  FROM /sapapo/v_matloc
  INTO TABLE lt_mtvfp
  FOR ALL ENTRIES IN lt_so_cap
  WHERE matnr = lt_so_cap-matnr
    AND locno = lt_so_cap-werks.
SORT lt_mtvfp BY matnr locno.

LOOP AT lt_so_cap INTO lw_so_all.
  READ TABLE lt_mtvfp INTO ls_mtvfp
    WITH KEY matnr = lw_so_all-matnr locno = lw_so_all-werks BINARY SEARCH.
  IF sy-subrc = 0 AND ( ls_mtvfp-mtvfp = 'KP' OR ls_mtvfp-mtvfp = 'SD' ).
    DELETE lt_so_cap INDEX sy-tabix.
  ENDIF.
ENDLOOP.
```

Re-collect `lt_so_cap_sum` after delete if cap sums are built earlier.

---

## 5. Error message mapping

| Message | Source | When |
|---------|--------|------|
| `text-005` | `get_gatprep_data` / `get_monthly_data` | No ALV row selected — often because **`gt_output` is empty** |
| `'No data found'(003)` | `ZAPO_GATP_CVC_CREATE` polymer include | CVC sub-screen `gt_shp_final` empty — downstream of empty allocation extract |

Fixing **§4.2 + §4.4** addresses both: grid populated and SO NT totals available.

---

## 6. Test plan

| # | Scenario | Steps | Expected |
|---|----------|-------|----------|
| T1 | KL only, new param | `H029SG`/`3925`/Div`23`/GC`5000000617`, `REQ_NT=KL`, no PAG row | ALV shows CVC; popup **NT (Plant) > 0** |
| T2 | KL + KP MTVFP | SE16: `V_MATLOC` `MTVFP=KP` for material | Popup NT = sum `BMENG` `REQ_NT=KL`; not blocked by KP delete |
| T3 | Old param regression | Division **without** `GATP` in `ZAPOPARAM` | Same as before + KL CVC still visible |
| T4 | UA + KL mix | 3×21 NT KL + 1 UA released | NT stays 63; MT shows UA (per UA fix MD) |
| T5 | Backup | Run with Data Backup ticked | `ZAPO_PRIME_BCKU` receives KL CVC row for today |

**SQL check (AD1):**

```sql
SELECT matnr, werks, spart, gccode, req_nt, req_mt, bmeng, edatu
  FROM zapo_so_list
 WHERE matnr  = 'H029SG'
   AND werks  = '3925'
   AND spart  = '23'
   AND gccode = '5000000617'
   AND req_nt = 'KL'
   AND delind = space
   AND abgru  = space.
```

---

## 7. Transport checklist

| # | Object | Change |
|---|--------|--------|
| 1 | `ZAPO_GATP_ALLOCATION_F005` | ADD method `merge_kl_so_cvc_to_output` |
| 2 | Same | `get_data` — call merge after filters |
| 3 | Same | `collect_final_output` — remove `gt_zapoparam` guard; call merge |
| 4 | Same | `get_zapo_so_list` — `ELSEIF req_nt = 'KL'` after KP matloc filter |
| 5 | Same | `get_gatprep_data` — fallback `lt_output` + new-param NT from SO |
| 6 | Optional | `get_data` CHANGE C — KP/SD delete on `lt_so_cap` only |

---

## 8. Related documents

- `ZGATPDB_Popup_NT_Reduced_UA_Order_Code_Correction.md` — independent NT/MT buckets in popup  
- `ZGATPDB_NewParam_Fixed_NT_Popup_Code_Correction.md` — new-param popup must use SO NT for KL rows  
- `ZGATPDB_SORG1020_DLBL_NT_MT_Code_Correction.md` — DLBL filter on `REQ_NT = 'KL'` lines  
- `FS_TS_NT_MT_Report_Revision.md` — BR6: KP/SD excluded from **cap**, not from NT display  

---

*End of document*
