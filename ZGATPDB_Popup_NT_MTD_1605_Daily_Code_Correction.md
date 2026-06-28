# ZGATPDB — Popup NT shows MTD 1605 instead of daily 15 (28-Jun)

**System:** DEVSCMAD1 (`ZAPO_GATP_ALLOCATION_REPORT`)  
**Include:** `ZAPO_GATP_ALLOCATION_F005`  
**Param:** `ZAPOPARAM` — `NT_MT_NEW` / `REPORT_NEW` active  
**Popup:** gATP Manual & NO Touch Report (`get_gatprep_data`)  
**Backup table:** `ZAPO_PRIME_BCKU`  
**Version:** 1.0 — 28/06/2026  

---

## 1. Symptom (screenshot)

**Selection:** `B120MA` / `3605` / Div `23` / GC `5000000617`  
**Run date:** **28-Jun-2026**

| Popup row | PMR / PP shown | Expected (today only) |
|-----------|----------------|------------------------|
| **MT (Plant)** | **15** | **15** ✓ |
| **NT (Plant)** | **1605** | **15** ✗ |

**Business facts**

| Date | Release | Already in `ZAPO_PRIME_BCKU` |
|------|---------|------------------------------|
| **27-Jun** | **1590 MT** NT (stock strategy / KL orders) | **Yes** — backed up on 27th |
| **28-Jun** | **15 MT** NT + **15 MT** MT (2 orders released today) | Today’s run |

**Math check:** `1605 = 1590 + 15` → popup NT is **month-to-date (MTD)**, not **today’s increment**.

MT is correct because it is sourced from **`ZAPO_ADB_ADJ`** with **`Approved_Date = sy-datum`** (daily). NT incorrectly includes **all June SO lines** (or full MTD SO bucket) without excluding quantities already saved on **27-Jun**.

---

## 2. Root cause — Old vs New param (AD1 code)

### 2.1 `get_zapo_so_list` — date range differs by param

**Old `NT_MT`** (`gt_zapoparam` empty) — popup uses **today only**:

```abap
ELSE.
  so_date-sign   = 'I'.
  so_date-option = 'EQ'.
  so_date-low    = sy-datum.          " ← 28-Jun only
  APPEND so_date TO so_date[].
ENDIF.
```

**New `NT_MT_NEW`** (`gt_zapoparam` filled) — **CHANGE H** uses **full month**:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0.
  CONCATENATE sy-datum+0(6) '01' INTO lw_gs_month_start.
  ...
  lw_gs_month_end = lw_gs_next_mth - 1.
  so_date-sign   = 'I'.
  so_date-option = 'BT'.
  so_date-low    = lw_gs_month_start.   " 01-Jun
  so_date-high   = lw_gs_month_end.     " 30-Jun
  APPEND so_date TO so_date[].
ENDIF.

SELECT ... bmeng req_nt req_mt ...
  FROM zapo_so_list
  WHERE edatu IN so_date              " ← entire month
    AND delind = space
    AND abgru  = space.
```

Then NT popup totals aggregate **all** `REQ_NT = 'KL'` lines in June:

```abap
DELETE lt_apo_so_list_nt WHERE req_nt NE 'KL'.
LOOP AT lt_apo_so_list_nt ...
  gw_apo_so_nt_pp_plant = gw_apo_so_nt_pp_plant + bmeng.  " MTD sum
ENDLOOP.
```

**Result:** 27-Jun **1590** + 28-Jun **15** = **1605** in popup NT.

Month-wide `EDATU` was added for **monthly cap / future-dated SO (BR5/BR8)** in `get_data` CHANGE C — it must **not** drive **daily popup** quantities.

---

### 2.2 MT is daily; NT is MTD — asymmetric sources

| Bucket | New param source | Date scope | 28-Jun value |
|--------|------------------|------------|--------------|
| **MT** | `ZAPO_ADB_ADJ` → `manual_touch` in `get_data` CHANGE F (BR1) | **`Approved_Date = sy-datum`** | **15** ✓ |
| **NT** | `get_zapo_so_list` → `gw_apo_so_nt_*` added in `get_gatprep_data` | **`EDATU` month start → month end** | **1605** ✗ |

```abap
" get_gatprep_data (~6226)
go_alloc->get_zapo_so_list( im_output = lt_output ).
gw_ntch_pp_plant = gw_ntch_pp_plant + gw_apo_so_nt_pp_plant.  " MTD SO NT
```

Grid `no_touch` is often **0** for stock-strategy materials (excluded from dashboard); popup NT comes **entirely** from `get_zapo_so_list` → full month sum.

---

### 2.3 `ZAPO_PRIME_BCKU` not used to net popup NT

**CHANGE D** in `get_data` reads prior backup for **MTD guard** only:

```abap
lw_saved_mtd = lw_prime_sum-inc_ord_quan.   " SUM prior days
IF lw_so_cap_qty > 0 AND lw_saved_mtd >= lw_so_cap_qty.
  CLEAR inc_ord_quan / manual_touch / no_touch.
ENDIF.
```

Prior saved **`no_touch = 1590`** on **27-Jun** is **never subtracted** from popup NT totals.

If backup still stores **cumulative** MTD per `ZDATE` (see `ZGATPDB_PRIME_BCKU_Daily_Quantity_Code_Correction.md`), popup must still show **daily 15**, not raw SO month sum.

---

## 3. Required behaviour

For **`NT_MT_NEW`** popup on **`ZDATE = sy-datum`**:

```text
Popup MT (today) = SUM( ZAPO_ADB_ADJ.USR_ADJ )     Approved_Date = today     → 15
Popup NT (today) = SUM( ZAPO_SO_LIST.BMENG )       REQ_NT = 'KL', EDATU = today
                 OR MAX( 0, SO_MTD_NT − SUM( ZAPO_PRIME_BCKU.NO_TOUCH prior days ) )
                 → 15 (not 1605)
```

**Rule:** Quantities already saved in **`ZAPO_PRIME_BCKU`** on prior dates in the month must **not** appear again in today’s popup.

Align with **BR3** (history not reduced) and **BR4** (daily increment vs cap headroom).

---

## 4. Code corrections (`ZAPO_GATP_ALLOCATION_F005`)

### 4.1 CHANGE L — `get_zapo_so_list`: daily date range for popup

**Add import parameter** to method signature (class definition + implementation):

```abap
METHOD get_zapo_so_list.
  IMPORTING
    im_output     TYPE ...
    im_popup_mode TYPE abap_bool DEFAULT abap_false.  " NEW
```

**Replace CHANGE H date block (~3077–3096)** with:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0 AND im_popup_mode = abap_false.
  " Monthly range — cap / mail / non-popup only (BR5, BR8)
  CONCATENATE sy-datum+0(6) '01' INTO lw_gs_month_start.
  ...
  so_date-low  = lw_gs_month_start.
  so_date-high = lw_gs_month_end.
  APPEND so_date TO so_date[].

ELSEIF sy-subrc = 0 AND im_popup_mode = abap_true.
  " Popup under NT_MT_NEW — today only (match Old NT_MT behaviour)
  CLEAR so_date[].
  so_date-sign   = 'I'.
  so_date-option = 'EQ'.
  so_date-low    = sy-datum.
  APPEND so_date TO so_date[].

ELSE.
  " Old NT_MT — unchanged
  IF p_mail = abap_true.
    ...
  ELSE.
    so_date-option = 'EQ'.
    so_date-low    = sy-datum.
    APPEND so_date TO so_date[].
  ENDIF.
ENDIF.
```

**Call from `get_gatprep_data` (~6226):**

```abap
go_alloc->get_zapo_so_list(
  im_output     = lt_output
  im_popup_mode = abap_true ).    " ← daily SO for popup
```

Keep existing calls from `get_data` / mail / backup **without** `im_popup_mode` (defaults `abap_false` → month range for cap).

---

### 4.2 CHANGE L — `get_gatprep_data`: net NT against prior backup (safety)

After `get_zapo_so_list` and before building `gt_gatp`, subtract prior-month saved NT for new param:

```abap
DATA: lv_prior_nt TYPE zbmeng,
      lv_prior_mt TYPE zbmeng,
      lw_mstart   TYPE datum.

READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0.
  CONCATENATE sy-datum+0(6) '01' INTO lw_mstart.

  SELECT SUM( no_touch ) SUM( manual_touch )
    FROM zapo_prime_bcku CLIENT SPECIFIED
    INTO (lv_prior_nt, lv_prior_mt)
    WHERE mandt      = sy-mandt
      AND prime_sto  = 'PRIME'          " or lw_sto from context
      AND zdate      >= lw_mstart
      AND zdate      <  sy-datum
      AND div        IN so_divi
      AND material   IN so_mat          " when selection filled
      AND location   IN so_loc
      AND grp_cust   IN so_gc.

  IF sy-subrc = 0.
    gw_apo_so_nt_pp_plant  = gw_apo_so_nt_pp_plant  - lv_prior_nt.
    gw_apo_so_nt_pe_plant  = gw_apo_so_nt_pe_plant  - lv_prior_nt.  " if div split needed
    gw_apo_so_nt_tot_plant = gw_apo_so_nt_tot_plant - lv_prior_nt.
    IF gw_apo_so_nt_pp_plant  < 0. CLEAR gw_apo_so_nt_pp_plant.  ENDIF.
    IF gw_apo_so_nt_tot_plant < 0. CLEAR gw_apo_so_nt_tot_plant. ENDIF.
    " Repeat for depot + PE/PVC/ELS columns if populated
  ENDIF.
ENDIF.
```

> **Prefer §4.1** (today-only `EDATU`) as primary fix. Use §4.2 only if SO `EDATU` does not align with physical release date.

---

### 4.3 CHANGE L — `get_gatprep_data`: do not double-count grid + SO NT

When new param uses SO bucket for popup NT (overwrite pattern):

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc = 0.
  " NT from daily SO only — do not add grid no_touch (often 0 or MTD-like)
  gw_ntch_pp_plant  = gw_apo_so_nt_pp_plant.
  gw_ntch_tot_plant = gw_apo_so_nt_tot_plant.
  " MT: grid UA (daily) + optional SO MT today
  gw_mtch_pp_plant  = gw_mtch_pp_plant + gw_apo_so_mt_pp_plant.
ELSE.
  gw_ntch_pp_plant  = gw_ntch_pp_plant  + gw_apo_so_nt_pp_plant.
  gw_mtch_pp_plant  = gw_mtch_pp_plant  + gw_apo_so_mt_pp_plant.
ENDIF.
```

Clear division NT accumulators from grid loop **before** assign when `sy-subrc = 0` if overwrite is used.

---

### 4.4 CHANGE L — `get_data` CHANGE F: daily NT for grid (optional, consistent display)

Inside CHANGE F when `gt_zapoparam` active, after reading `lw_saved_mtd` / `lt_prime_sum`:

```abap
DATA: lw_prior_nt TYPE zbmeng,
      lw_prior_mt TYPE zbmeng.

READ TABLE lt_prime_sum INTO lw_prime_sum
  WITH KEY material  = <fs_output>-material
           location  = <fs_output>-location
           div       = <fs_output>-div
           grp_cust  = <fs_output>-grp_cust
           dist_chan = <fs_output>-dist_chan.
IF sy-subrc = 0.
  lw_prior_nt = lw_prime_sum-no_touch.
  lw_prior_mt = lw_prime_sum-manual_touch.
ENDIF.

" SO KL NT for CVC (month or today — match popup)
READ TABLE lt_so_cvc_ntmt INTO ls_so_cvc
  WITH KEY gccode = <fs_output>-grp_cust
           matnr  = <fs_output>-material
           werks  = <fs_output>-location
           vtweg  = <fs_output>-dist_chan
           spart  = <fs_output>-div.
IF sy-subrc = 0.
  <fs_output>-no_touch = ls_so_cvc-nt_qty - lw_prior_nt.
  IF <fs_output>-no_touch < 0. CLEAR <fs_output>-no_touch. ENDIF.
ENDIF.

" MT stays daily from ZAPO_ADB_ADJ (already correct)
<fs_output>-manual_touch = lw_mt_adj_sum-usr_adj.
```

Build `lt_so_cvc_ntmt` with **`EDATU = sy-datum`** only when used for popup/grid daily NT.

---

### 4.5 Ensure `ZAPO_PRIME_BCKU` stores daily qty (prerequisite)

Apply **`zapo_prime_bcku_daily_qty`** from:

`ZGATPDB_PRIME_BCKU_Daily_ABAP_Code_Change.md`

So **27-Jun** row stores **`no_touch = 1590`** (daily that day), **28-Jun** stores **`no_touch = 15`**.  
Without daily save, §4.2 subtraction still works if prior rows hold **daily** values; if prior rows hold **cumulative MTD**, use **today-only SO** (§4.1) instead of subtracting cumulative backup.

---

## 5. Old vs New — summary

| Aspect | Old `NT_MT` | New `NT_MT_NEW` (before fix) | New (after fix) |
|--------|-------------|------------------------------|-----------------|
| `get_zapo_so_list` date (popup) | `EDATU = sy-datum` | `EDATU` month **01–30** | `EDATU = sy-datum` when `im_popup_mode = X` |
| Popup MT | SO + grid / legacy | **15** from ADB today ✓ | **15** ✓ |
| Popup NT | Daily SO add-on | **1605** MTD ✗ | **15** daily ✓ |
| Prior `ZAPO_PRIME_BCKU` | Not re-added (daily SO) | **1590** re-included via month SO | Excluded |

---

## 6. Test plan — reproduce 28-Jun case

| # | Step | Expected popup NT (Plant) | Expected popup MT (Plant) |
|---|------|---------------------------|---------------------------|
| T1 | **27-Jun** run, backup 1590 NT | **1590** | per UA |
| T2 | **28-Jun** run after T1 backup | **15** (not 1605) | **15** |
| T3 | SE16 `ZAPO_SO_LIST`: sum `BMENG` `REQ_NT=KL` `EDATU=28.06.2026` | **15** | — |
| T4 | SE16 `ZAPO_SO_LIST`: sum for full June | **1605** (MTD reference only) | — |
| T5 | SE16 `ZAPO_PRIME_BCKU` `ZDATE=27.06`, `NO_TOUCH` | **1590** | — |
| T6 | Old `NT_MT` regression same CVC | Unchanged daily behaviour | — |
| T7 | No release on 29-Jun | **0** NT / MT | — |

**SQL (AD1):**

```sql
-- Today's NT SO lines
SELECT edatu, SUM( bmeng ) AS nt_qty
  FROM zapo_so_list
 WHERE matnr  = 'B120MA'
   AND werks  = '3605'
   AND spart  = '23'
   AND gccode = '5000000617'
   AND req_nt = 'KL'
   AND delind = space
   AND abgru  = space
 GROUP BY edatu
 ORDER BY edatu;

-- Prior backup
SELECT zdate, no_touch, manual_touch, inc_ord_quan
  FROM zapo_prime_bcku
 WHERE material = 'B120MA'
   AND location = '3605'
   AND div      = '23'
   AND grp_cust = '5000000617'
   AND zdate    >= '20260601'
 ORDER BY zdate;
```

---

## 7. Transport checklist

| # | Object | Change |
|---|--------|--------|
| 1 | Class definition `LCL_GATP_ALLOC` | Add `im_popup_mode` to `get_zapo_so_list` |
| 2 | `ZAPO_GATP_ALLOCATION_F005` | §4.1 date split in `get_zapo_so_list` |
| 3 | Same | §4.1 `get_gatprep_data` call with `im_popup_mode = abap_true` |
| 4 | Same | §4.2 optional prior-backup net (new param) |
| 5 | Same | §4.3 popup NT overwrite (no double count) |
| 6 | Same | §4.4 optional daily NT in CHANGE F |
| 7 | Same | `zapo_prime_bcku_daily_qty` in `collect_final_output` (prerequisite) |

---

## 8. Summary

| Question | Answer |
|----------|--------|
| Why NT = **1605**? | New param **`get_zapo_so_list`** sums **`REQ_NT='KL'`** for **whole month** (1590 + 15) |
| Why MT = **15**? | MT from **`ZAPO_ADB_ADJ`** filtered to **today** only |
| Why 27-Jun not excluded? | **`ZAPO_PRIME_BCKU`** prior save not netted; month `EDATU` range used for popup |
| Primary fix | Pass **`im_popup_mode = abap_true`** → **`EDATU = sy-datum`** for popup SO NT/MT |
| Expected after fix | **28-Jun: NT = 15, MT = 15** |

**Related MDs**

- `ZGATPDB_PRIME_BCKU_Daily_ABAP_Code_Change.md` — daily backup save  
- `ZGATPDB_Popup_NT_Reduced_UA_Order_Code_Correction.md` — independent NT/MT buckets  
- `FS_TS_NT_MT_Report_Revision.md` — BR4 daily cap, BR5 month EDATU scope  

---

*End of document*
