# ZGATPDB — Future-date orders reappear next day after `ZAPO_PRIME_BCKU` backup (NT ≠ 0)

**System:** DEVSCMAD1 (`ZAPO_GATP_ALLOCATION_F005` — last verified live source; AD1 MCP unavailable at write time)  
**Program:** `ZAPO_GATP_ALLOCATION_REPORT` / **T-code:** `ZGATPDB`  
**Param:** `NT_MT_NEW` / `REPORT_NEW`  
**Table:** `ZAPO_PRIME_BCKU`  
**Popup:** gATP Manual & NO Touch Report (NT) — and related SO drill-down  
**Version:** 1.1 — 25/07/2026  

---

## 1. Symptom (screenshot 25.07.2026)

| Observation | Detail |
|-------------|--------|
| Orders | Future **EDATU** / delivery date (still inside `sy-datum .. sy-datum+2`) |
| Prior run | Already counted in NT–MT report and saved to **`ZAPO_PRIME_BCKU`** |
| Next day | Same orders still drive NT |
| **Expected** | **NT popup = 0** (no new demand) |
| **Actual** | NT still shows quantity (user: **70**; SO drill-down in screenshot shows open lines e.g. `0307026228` / `0307026233` @ 21 MT each) |

**Grid row in screenshot (example):** Date **25.07.2026**, Manual Tch **21**, No Tch **0** — grid may already net NT for that CVC, while **popup / SO list** still exposes the future-dated volume.

**Business rule**

> Once a future-dated order quantity is taken into the NT–MT report and **backed up** (`ZAPO_PRIME_BCKU`), the **next calendar day’s** run must **not** show that quantity again in **NT**.  
> NT next day = **only new** open demand not yet backed up.

---

## 2. Why future dates keep matching every day

**CHANGE C** builds daily NT with a **sliding** window (~795 / ~839):

```abap
lv_so_nt_edatu_high = sy-datum + 2.

IF lw_so_all-edatu >= sy-datum
AND lw_so_all-edatu <= lv_so_nt_edatu_high
AND ( req_nt = 'Z1' OR 'KSV' OR 'KL' ).
  COLLECT ... INTO lt_so_daily_nt_sum.
ENDIF.
```

| Day | Same order EDATU = 26.07 | In window? |
|-----|--------------------------|------------|
| 24.07 (backup day) | 24–26 Jul | **Yes** → NT saved to BCKU |
| 25.07 (next day) | 25–27 Jul | **Yes again** → still in `daily_nt` |

Sliding window is **correct for capture**; reappearance must be stopped by **netting prior backup**, not by removing the window.

---

## 3. Root cause — netting uses **yesterday only**, not all prior backup

### 3.1 Full month prior backup is already loaded

**CHANGE D** (~573–647):

```abap
lv_sydautm = sy-datum - 1.

SELECT ... no_touch ... FROM zapo_prime_bcku
  WHERE zdate BETWEEN lw_month_start AND lv_sydautm
  ...
  " → mt_prime_bcku_sum-no_touch = SUM of ALL prior days this month

  IF lw_prime_saved-zdate = lv_sydautm.
    " → lt_prime_yday_nt = YESTERDAY only
  ENDIF.
```

| Table | Content |
|-------|---------|
| **`mt_prime_bcku_sum`** | `SUM(no_touch)` for **month start .. yesterday** |
| **`lt_prime_yday_nt`** | `no_touch` for **yesterday only** |

### 3.2 CHANGE F subtracts only yesterday (~2689–2726)

```abap
READ TABLE lt_prime_yday_nt INTO lw_prime_sum ...   " ← yesterday only
lv_saved_nt = lw_prime_sum-no_touch.

lv_net_nt = ls_so_daily_nt_sum-daily_nt - lv_saved_nt.
<fs_output>-no_touch = lv_net_nt.
```

| Backup taken | Yesterday `NO_TOUCH` | Earlier days `NO_TOUCH` | Net today |
|--------------|----------------------|-------------------------|-----------|
| On **yesterday** with full future NT | = daily_nt | — | **0** ✓ |
| On **D−2 / earlier**, not re-backed yesterday | **0** | **70** (or 21/…) | **daily_nt − 0 = still full** ✗ |
| Yesterday backup ran but saved **0** NT (bug / empty row) | 0 | earlier qty | **full again** ✗ |

So future-dated orders that were backed up **before yesterday**, or not present on yesterday’s row, **reappear** on the next execution.

### 3.3 Popup / SO paths may ignore backup entirely

| Path | Nets vs `ZAPO_PRIME_BCKU`? |
|------|---------------------------|
| Grid `no_touch` (CHANGE F) | Only vs **yesterday** (`lt_prime_yday_nt`) |
| `get_zapo_so_list` (popup SO NT totals) | **No** — full `EDATU IN sy-datum..+2` |
| `get_so_list` / Sales Order drill-down | **No** — lists open SO by `edatu` |

If NT popup uses SO totals (old param add-on, or code drift), quantity returns even when grid `no_touch` is 0.

---

## 4. Required behaviour

```text
daily_so_nt(cvc)   = SUM(SO NT) where EDATU between sy-datum and sy-datum+2

prior_backup_nt    = SUM( ZAPO_PRIME_BCKU.NO_TOUCH )
                     where same CVC
                     and zdate >= month_start
                     and zdate <  sy-datum          " all prior days, not only yesterday

net_nt_today       = MAX( 0, daily_so_nt - prior_backup_nt )
```

| Layer | Must use |
|-------|----------|
| Grid `no_touch` | `net_nt_today` |
| gATP MT/NT popup NT | Same net (grid and/or SO after netting) |
| Next day, no new orders | **0** |

---

## 5. Code corrections (`ZAPO_GATP_ALLOCATION_F005`)

### 5.1 CHANGE-FUT1 — Net NT against **full prior-month** backup (primary)

**Location:** All three CHANGE F blocks (~2689, ~2974, ~3301).

**Replace** read from `lt_prime_yday_nt` with **`mt_prime_bcku_sum`** (already filled in CHANGE D):

```abap
*--- Prior backup NT: ALL prior days this month (not yesterday only)
CLEAR lv_saved_nt.
CLEAR lw_prime_sum.
READ TABLE mt_prime_bcku_sum INTO lw_prime_sum
  WITH TABLE KEY
    material  = <fs_output>-material
    location  = <fs_output>-location
    div       = <fs_output>-div
    grp_cust  = <fs_output>-grp_cust
    dist_chan = <fs_output>-dist_chan.
IF sy-subrc = 0.
  lv_saved_nt = lw_prime_sum-no_touch.   " month_start .. sy-datum-1
ENDIF.

CLEAR ls_so_daily_nt_sum.
READ TABLE lt_so_daily_nt_sum INTO ls_so_daily_nt_sum
  WITH TABLE KEY
    gccode = <fs_output>-grp_cust
    matnr  = <fs_output>-material
    werks  = <fs_output>-location
    vtweg  = <fs_output>-dist_chan
    spart  = lv_nt_spart.
IF sy-subrc <> 0 AND lv_nt_spart <> <fs_output>-div.
  READ TABLE lt_so_daily_nt_sum INTO ls_so_daily_nt_sum
    WITH TABLE KEY
      gccode = <fs_output>-grp_cust
      matnr  = <fs_output>-material
      werks  = <fs_output>-location
      vtweg  = <fs_output>-dist_chan
      spart  = <fs_output>-div.
ENDIF.

IF sy-subrc = 0.
  lv_net_nt = ls_so_daily_nt_sum-daily_nt - lv_saved_nt.
  IF lv_net_nt < 0.
    CLEAR lv_net_nt.
  ENDIF.
  <fs_output>-no_touch = lv_net_nt.
ELSE.
  CLEAR <fs_output>-no_touch.
ENDIF.
```

**Remove / stop using** `lt_prime_yday_nt` for NT netting (table can remain unused or be deleted later).

**Effect**

| Backup on 23.07 `NO_TOUCH=70`, run on 25.07, same future SO still in window | `daily_nt=70`, `prior=70` → **`no_touch=0`** |

---

### 5.2 CHANGE-FUT2 — Popup: net `get_zapo_so_list` NT vs prior backup

**Location:** `get_gatprep_data`, after `get_zapo_so_list` (~7202), for new param.

Even when popup uses grid `no_touch` only, keep SO path clean for old param / future changes:

```abap
DATA: lv_mstart      TYPE datum,
      lv_prior_nt_pp TYPE zbmeng.

IF lv_new_gatp_param = abap_true.
  CONCATENATE sy-datum+0(6) '01' INTO lv_mstart.

  SELECT SUM( no_touch )
    FROM zapo_prime_bcku CLIENT SPECIFIED
    INTO lv_prior_nt_pp
    WHERE mandt     = sy-mandt
      AND prime_sto = 'PRIME'
      AND zdate     >= lv_mstart
      AND zdate     <  sy-datum
      AND div       = '23'
      AND material  IN so_mat
      AND location  IN so_loc
      AND grp_cust  IN so_gc.

  gw_apo_so_nt_pp_plant = gw_apo_so_nt_pp_plant - lv_prior_nt_pp.
  IF gw_apo_so_nt_pp_plant < 0.
    CLEAR gw_apo_so_nt_pp_plant.
  ENDIF.
  " Repeat for PE(22), PVC(24), ELS(37), depot, and tot_plant as needed
ENDIF.
```

**Ensure new-param popup NT stays aligned with grid:**

```abap
* Prefer: do NOT add gw_apo_so_nt_* for new param (already in live code ~7219)
* Grid no_touch after CHANGE-FUT1 is the single source → popup NT = 0 when expected
```

---

### 5.3 CHANGE-FUT3 — Do not save gross NT again on next-day backup

**Location:** `collect_final_output` — save `no_touch` from grid **after** CHANGE-FUT1.

```abap
<lfs_prime_buk>-no_touch = gw_output-no_touch.  " already net (0 if only old future SO)
```

If next-day backup saves **0**, month sum of prior `NO_TOUCH` stays correct for the following day.

---

### 5.4 Optional — Sales Order drill-down (`get_so_list` / hotspot)

Screenshot “Sales Order” popup (`disp_so_data` / `gt_so_output`) lists open APO/SD quantities and **does not** read `ZAPO_PRIME_BCKU`.  

If business requires that drill-down to hide already-backed-up future NT:

- Filter display lines whose CVC+qty already covered by prior `NO_TOUCH`, **or**
- Show a note that drill-down is raw open SO, while **NT report** is net of backup.

**Primary fix for “NT = 0 next day” remains CHANGE-FUT1** on `no_touch` / MT–NT popup.

---

## 6. Keep forward EDATU window

| Use | Range |
|-----|--------|
| Capture future EDATU on backup day | `sy-datum .. sy-datum+2` — **keep** |
| Stop next-day re-show | Net **`SUM(NO_TOUCH)` prior days in month** |

Do **not** force `EDATU = sy-datum` only — that would miss future-dated orders on the backup day itself.

---

## 7. Test plan

| # | Step | Expected |
|---|------|----------|
| T1 | Future EDATU order in window; run report; **backup** | `ZAPO_PRIME_BCKU` `ZDATE=today`, `NO_TOUCH` = order qty (e.g. 70 or 21) |
| T2 | Next calendar day; **no new order**; run report | Grid **No Tch = 0**; MT/NT popup **NT = 0** |
| T3 | Skip a day without backup; run two days later | Still **0** (full prior-month sum, not yesterday-only) |
| T4 | New additional NT 15 tomorrow | Popup NT = **15** only |
| T5 | SE16 `ZAPO_PRIME_BCKU` | Prior `NO_TOUCH` sum ≥ future SO still in window |

**SQL**

```sql
SELECT zdate, material, location, no_touch, manual_touch, inc_ord_quan
  FROM zapo_prime_bcku
 WHERE zdate BETWEEN '20260701' AND sy-datum - 1
   AND prime_sto = 'PRIME'
 ORDER BY zdate;

SELECT vbeln, matnr, werks, edatu, wmeng, bmeng, req_nt
  FROM zapo_so_list
 WHERE edatu BETWEEN sy-datum AND sy-datum + 2
   AND delind = ' '
   AND abgru  = ' '
   AND req_nt IN ('Z1','KSV','KL');
```

---

## 8. Transport checklist

| # | Object | Change |
|---|--------|--------|
| 1 | `ZAPO_GATP_ALLOCATION_F005` | §5.1 — `lv_saved_nt` from **`mt_prime_bcku_sum`** (all prior days) |
| 2 | Same | §5.2 — optional popup SO NT netting |
| 3 | Same | §5.3 — backup saves net `no_touch` |
| 4 | Test | T1–T3: next day NT = **0** |

---

## 9. Related MDs

| MD | Topic |
|----|-------|
| `ZGATPDB_Future_EDATU_Backup_Popup_Net_NT_Code_Correction.md` | Original “do not subtract” bug (v1.0) |
| `ZGATPDB_PRIME_BCKU_Daily_Quantity_Code_Correction.md` | Daily vs MTD save |
| `ZGATPDB_CreditBlock_NT_Duplicate_After_Confirm_Code_Correction.md` | CB confirm double-count |

**This MD (v1.1):** Partial fix that nets **yesterday only** is insufficient — must net **all prior `NO_TOUCH` in the month**.

---

## 10. Summary

| Question | Answer |
|----------|--------|
| Why orders come back next day? | Future **EDATU** still in `sy-datum..+2` |
| Why netting fails? | Code subtracts **`lt_prime_yday_nt`** (yesterday only) |
| Where is full prior NT? | Already in **`mt_prime_bcku_sum-no_touch`** |
| Primary fix | `no_touch = daily_nt − mt_prime_bcku_sum-no_touch` |
| Expected next day | **NT popup = 0** when no new orders |

---

*End of document*
