# ZGATPDB — Future EDATU order: backup `ZAPO_PRIME_BCKU` vs popup NT double-count

**System:** DEVSCMAD1 (`ZAPO_GATP_ALLOCATION_F005` — live source reviewed)  
**Program:** `ZAPO_GATP_ALLOCATION_REPORT` / **T-code:** `ZGATPDB`  
**Param:** `NT_MT_NEW` / `REPORT_NEW`  
**Table:** `ZAPO_PRIME_BCKU`  
**Popup:** gATP Manual & NO Touch Report  
**Version:** 1.0 — 09/07/2026  

---

## 1. Symptom (screenshot)

**Materials on dashboard:** `B120MA` / `3605`, `H110MA` / `3617` (Div `23`)

| Popup row | PMR / PP shown | Expected today |
|-----------|----------------|----------------|
| **NT (Plant)** | **15** | **0** |
| MT (Plant) | 0 | 0 |

**Business facts**

| When | What happened |
|------|----------------|
| **Yesterday** | `B120MA` order released — **15 MT NT**, **Requested Delivery Date (EDATU) = future date** (within forward window) |
| **Yesterday backup** | User took **Data Backup** — **15 MT NT** should be saved in **`ZAPO_PRIME_BCKU`** (`NO_TOUCH = 15`, `ZDATE = yesterday`) |
| **Today** | Fresh execution — **no new order** released |
| **Expected** | Popup NT = **0** (already backed up yesterday) |
| **Actual** | Popup NT = **15** again |

**Rule**

> Future-dated NT quantity is **consumed on the backup run date** (`ZDATE`).  
> On the next calendar day, if no new SO quantity exists, **popup NT must be zero**.

This applies to **regular** materials like **`B120MA`** (`REQ_NT = Z1/KSV/KL`), not stock-strategy-only materials (`H029SG` / KP — see separate MD for backup exclusion).

---

## 2. AD1 code — how NT is built today

### 2.1 Daily SO NT window (`lt_so_daily_nt_sum`) — `get_data` CHANGE C (~811)

```abap
lv_so_nt_edatu_high = sy-datum + 2.

IF lw_so_all-edatu >= sy-datum
AND lw_so_all-edatu <= lv_so_nt_edatu_high
AND ( lw_so_all-req_nt = 'Z1' OR 'KSV' OR 'KL' ).
  ls_so_daily_nt_sum-daily_nt = bmeng OR wmeng.
  COLLECT ls_so_daily_nt_sum INTO lt_so_daily_nt_sum.
ENDIF.
```

| Item | Value |
|------|-------|
| Forward window | **`sy-datum` .. `sy-datum + 2`** (today + 2 days) |
| Purpose | Pick up **future EDATU** orders for today's NT bucket |
| `B120MA` 15 MT | Included every day the order's **EDATU** falls in the sliding window |

So on **today** and **yesterday**, the same 15 MT order can still match `lt_so_daily_nt_sum` → **15** each day unless prior backup is netted off.

---

### 2.2 Prior backup is read but **not subtracted** — CHANGE F (~2570–2665)

```abap
READ TABLE mt_prime_bcku_sum INTO lw_prime_sum
  WITH KEY material = <fs_output>-material ...
IF sy-subrc = 0.
  lw_saved_mtd = lw_prime_sum-inc_ord_quan.
  lv_saved_nt  = lw_prime_sum-no_touch.    " ← SUM prior ZAPO_PRIME_BCKU this month
ENDIF.

READ TABLE lt_so_daily_nt_sum INTO ls_so_daily_nt_sum ...
IF sy-subrc = 0.
*--- Daily NT for zdate row (matches popup/backup; do not subtract MTD lv_saved_nt)
  <fs_output>-no_touch = ls_so_daily_nt_sum-daily_nt.   " ← BUG: ignores lv_saved_nt
ENDIF.
```

**`mt_prime_bcku_sum`** is loaded from **`ZAPO_PRIME_BCKU`** for `zdate BETWEEN month_start AND sy-datum - 1` (~570–618).

| Variable | Loaded | Used for `no_touch`? |
|----------|--------|----------------------|
| `lv_saved_nt` | Prior **`SUM(no_touch)`** from backup | **No** — explicitly skipped |
| `ls_so_daily_nt_sum-daily_nt` | SO NT in forward window | **Yes** — full 15 every day |

**Result:** Grid / backup source always shows **gross 15**, even when **15 was already saved yesterday**.

---

### 2.3 Popup uses gross SO totals — `get_gatprep_data` (~7025–7067)

```abap
*--- NT popup: drop selected-row NT; get_zapo_so_list sums all daily SO NT
IF lv_new_gatp_param = abap_true.
  CLEAR: gw_ntch_pp_plant, ...    " grid NT cleared
ENDIF.

go_alloc->get_zapo_so_list(
  EXPORTING im_output = lt_output iv_popup_mode = abap_true ).

IF lv_new_gatp_param = abap_true.
  gw_ntch_pp_plant = gw_apo_so_nt_pp_plant.   " full SO NT — no backup netting
ENDIF.
```

**`get_zapo_so_list`** popup mode uses **`EDATU` = `sy-datum` .. `sy-datum + 2`** (~3540–3546) — same forward window as `lt_so_daily_nt_sum`.

**No step subtracts prior `ZAPO_PRIME_BCKU.no_touch`** → popup shows **15** again on today's fresh run.

---

### 2.4 Backup save — gross `no_touch` — `collect_final_output` (~4038)

```abap
*--- Save grid MT/NT as-is (CHANGE F already daily for new param; legacy unchanged)
<lfs_prime_buk>-no_touch     = gw_output-no_touch.
<lfs_prime_buk>-inc_ord_quan = gw_output-inc_ord_quan.
```

Comment assumes CHANGE F already stores **daily net** — but CHANGE F stores **gross** `ls_so_daily_nt_sum-daily_nt` without subtracting `lv_saved_nt`.

If backup runs **twice** on the same order without netting, duplicate risk exists.

---

## 3. Root cause summary

| # | Gap | Effect |
|---|-----|--------|
| **1** | `no_touch = daily_nt` without **`- lv_saved_nt`** | Yesterday's backed-up **15** still appears as **15** today |
| **2** | Popup **`gw_apo_so_nt_*`** has no backup netting | NT popup **15** on fresh run |
| **3** | `collect_final_output` saves gross `no_touch` | Backup may re-save same future EDATU qty |
| **4** | Forward window **`sy-datum..+2`** is correct for **capturing** future EDATU on backup day | Window is **not** wrong — **missing netting** is wrong |

**Why Old param appeared correct:** Old popup used **`EDATU = sy-datum` only** (single day) + additive logic; future EDATU orders were handled differently and prior backup interaction was implicit via daily SO filter.

---

## 4. Required behaviour

```text
daily_so_nt(cvc)  = SUM( ZAPO_SO_LIST.BMENG )
                      WHERE REQ_NT in (Z1,KSV,KL)
                      AND EDATU BETWEEN sy-datum AND sy-datum+2

prior_backup_nt   = SUM( ZAPO_PRIME_BCKU.NO_TOUCH )
                      WHERE same CVC keys
                      AND zdate >= month_start
                      AND zdate < sy-datum

net_nt_today      = MAX( 0, daily_so_nt - prior_backup_nt )
```

| Layer | Formula |
|-------|---------|
| **Grid `no_touch`** (CHANGE F) | `net_nt_today` |
| **Backup `NO_TOUCH`** (`collect_final_output`) | `net_nt_today` (increment not yet saved) |
| **Popup NT** (`get_gatprep_data`) | `SUM(net_nt_today)` per division/plant **or** `gw_apo_so_nt_* - prior_backup_nt` |

**Yesterday backup example (`B120MA`, future EDATU 15 MT):**

| Day | `daily_so_nt` | `prior_backup_nt` | Save / show |
|-----|---------------|-------------------|-------------|
| Yesterday | 15 | 0 | Backup **`NO_TOUCH = 15`** |
| Today (no new order) | 15 | **15** | Popup **`NT = 0`** |

---

## 5. Code corrections (`ZAPO_GATP_ALLOCATION_F005`)

### 5.1 CHANGE N — `get_data` CHANGE F: net NT against prior backup

**Location:** All three CHANGE F blocks (~2657, ~2889, ~3163) — replace:

```abap
*--- Daily NT for zdate row (matches popup/backup; do not subtract MTD lv_saved_nt)
<fs_output>-no_touch = ls_so_daily_nt_sum-daily_nt.
```

**With:**

```abap
*--- Net NT today: forward-window SO minus already-backed-up NT (future EDATU rule)
DATA: lv_net_nt TYPE zbmeng.
lv_net_nt = ls_so_daily_nt_sum-daily_nt - lv_saved_nt.
IF lv_net_nt < 0.
  CLEAR lv_net_nt.
ENDIF.
<fs_output>-no_touch = lv_net_nt.
```

Keep **`inc_ord_quan`** daily logic (subtract `lw_saved_mtd` for plant) as already coded (~2607–2621).

> **`lv_saved_nt`** is already populated from **`mt_prime_bcku_sum`** — no new SELECT required.

---

### 5.2 CHANGE N — `get_gatprep_data`: net popup NT from prior backup

**Location:** After `get_zapo_so_list` (~7039), before assigning `gw_ntch_*` (~7057).

```abap
DATA: lv_prior_nt_plant TYPE zbmeng,
      lv_prior_nt_pp    TYPE zbmeng,
      lv_prior_nt_pe    TYPE zbmeng,
      lv_prior_nt_pvc   TYPE zbmeng,
      lv_prior_nt_els   TYPE zbmeng,
      lv_mstart         TYPE datum.

IF lv_new_gatp_param = abap_true.
  CONCATENATE sy-datum+0(6) '01' INTO lv_mstart.

  SELECT SUM( no_touch )
    FROM zapo_prime_bcku CLIENT SPECIFIED
    INTO lv_prior_nt_plant
    WHERE mandt     = sy-mandt
      AND prime_sto = 'PRIME'
      AND zdate     >= lv_mstart
      AND zdate     <  sy-datum
      AND div       IN so_divi
      AND material  IN so_mat
      AND location  IN so_loc
      AND grp_cust  IN so_gc.
  " Optional: split by div/location_type if mail layout needs PE/PP columns —
  " or net proportionally from gw_apo_so_nt_* after full SELECT

  gw_apo_so_nt_tot_plant = gw_apo_so_nt_tot_plant - lv_prior_nt_plant.
  IF gw_apo_so_nt_tot_plant < 0. CLEAR gw_apo_so_nt_tot_plant. ENDIF.

  " Per-division net (PP for B120MA div 23) — preferred:
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
  IF gw_apo_so_nt_pp_plant < 0. CLEAR gw_apo_so_nt_pp_plant. ENDIF.

  " Repeat for PE(22), PVC(24), ELS(37) and depot columns as needed
ENDIF.

IF lv_new_gatp_param = abap_true.
  gw_ntch_pp_plant = gw_apo_so_nt_pp_plant.
  ...
ENDIF.
```

**Alternative (cleaner):** Build **`lt_so_daily_nt_sum_net`** in `get_data` (daily_nt − lv_saved_nt per CVC) and pass to popup via class attribute — popup sums that table instead of raw `get_zapo_so_list` NT totals.

---

### 5.3 CHANGE N — `collect_final_output`: save net NT increment

**Location:** Before assign to `<lfs_prime_buk>` / `gw_prime_buk_t` (~4038).

```abap
DATA: lw_prior_nt TYPE zbmeng,
      lw_net_nt   TYPE zbmeng,
      lw_mstart   TYPE datum,
      lw_prime_sum TYPE lty_prime_mtd_sum.  " use existing type

CONCATENATE sy-datum+0(6) '01' INTO lw_mstart.

SELECT SUM( no_touch )
  FROM zapo_prime_bcku CLIENT SPECIFIED
  INTO lw_prior_nt
  WHERE mandt      = sy-mandt
    AND prime_sto  = lw_sto
    AND material   = gw_output-material
    AND location   = gw_output-location
    AND dist_chan  = gw_output-dist_chan
    AND div        = gw_output-div
    AND grp_cust   = gw_output-grp_cust
    AND region     = gw_output-region
    AND ship_to_party = gw_output-ship_to_party
    AND zdate      >= lw_mstart
    AND zdate      <  sy-datum.

lw_net_nt = gw_output-no_touch.   " already net if §5.1 applied
IF lw_net_nt < 0. CLEAR lw_net_nt. ENDIF.

<lfs_prime_buk>-no_touch     = lw_net_nt.
<lfs_prime_buk>-inc_ord_quan = gw_output-inc_ord_quan.
<lfs_prime_buk>-manual_touch = gw_output-manual_touch.
```

If §5.1 is applied, `gw_output-no_touch` is already net — backup can save as-is. §5.3 is a safety net if backup runs without re-running full `get_data`.

---

### 5.4 Keep forward EDATU window — do not narrow to `EDATU = sy-datum` only

| Use | `EDATU` range | Reason |
|-----|---------------|--------|
| **Cap** (`iv_popup_mode = false`) | Month start → month end | BR5 / BR8 |
| **Daily NT / popup** (`iv_popup_mode = true`) | **`sy-datum` .. `sy-datum + 2`** | Capture **future delivery** orders on backup day |
| **Netting** | Subtract **`ZAPO_PRIME_BCKU` prior `no_touch`** | Prevent re-show next day |

Do **not** revert popup to single-day `EDATU = sy-datum` only — that would stop **yesterday's backup** from capturing future-dated orders.

---

### 5.5 Optional — track backup at SO line level (future enhancement)

If multiple future orders on same CVC net incorrectly, extend backup key with **`VBELN`/`POSNR`** or a hash of backed SO lines. Not required for the **single 15 MT / B120MA** case.

---

## 6. Old vs New comparison

| Aspect | Old `NT_MT` | New `NT_MT_NEW` (bug) | After CHANGE N |
|--------|-------------|------------------------|----------------|
| Popup `EDATU` filter | `= sy-datum` | `sy-datum .. +2` | `sy-datum .. +2` |
| Future EDATU on backup day | Limited / implicit | Captured in `lt_so_daily_nt_sum` | Captured |
| Subtract prior `ZAPO_PRIME_BCKU` NT | Implicit via daily filter | **Not done** | **`daily_nt - lv_saved_nt`** |
| Today fresh run (backed yesterday) | ~0 | **15** ✗ | **0** ✓ |
| Yesterday backup `NO_TOUCH` | 15 (if backup run) | 15 (if backup run) | **15** ✓ |

---

## 7. Test plan — `B120MA` / `3605` / GC `5000000617`

| # | Step | `ZAPO_SO_LIST` | `ZAPO_PRIME_BCKU` | Popup NT (Plant) |
|---|------|----------------|-------------------|------------------|
| T1 | Create order 15 MT NT, **EDATU = sy-datum + 1** | 1 line, `REQ_NT` set | — | — |
| T2 | Run report **yesterday**, take **backup** | Unchanged | **`ZDATE=yesterday`, `NO_TOUCH=15`** | 15 |
| T3 | Run report **today**, no new order | Same line still in forward window | Row from T2 exists | **0** (not 15) |
| T4 | Release **new** 10 MT NT today | New line added | Prior row unchanged | **10** |
| T5 | Backup today after T4 | — | **`NO_TOUCH=10`** for today | — |
| T6 | `H110MA` / `3617` — no orders | — | No new row | **0** |

**Verification SQL (AD1):**

```sql
-- SO: future EDATU order
SELECT vbeln, matnr, werks, gccode, edatu, bmeng, req_nt
  FROM zapo_so_list
 WHERE matnr  = 'B120MA'
   AND werks  = '3605'
   AND gccode = '5000000617'
   AND delind = space
   AND abgru  = space;

-- Backup: should show 15 on yesterday, nothing new today
SELECT zdate, material, location, no_touch, inc_ord_quan, manual_touch
  FROM zapo_prime_bcku
 WHERE material = 'B120MA'
   AND location = '3605'
   AND grp_cust = '5000000617'
 ORDER BY zdate;
```

---

## 8. Transport checklist

| # | Object | Change |
|---|--------|--------|
| 1 | `ZAPO_GATP_ALLOCATION_F005` | §5.1 — net `no_touch` in CHANGE F (3 blocks) |
| 2 | Same | §5.2 — net popup `gw_apo_so_nt_*` vs prior backup |
| 3 | Same | §5.3 — confirm backup saves net increment |
| 4 | Test | `B120MA` future EDATU scenario T1–T3 |

---

## 9. Related MDs (do not confuse)

| MD | Topic |
|----|-------|
| `ZGATPDB_NewParam_Backup_Exclude_StockStrategy_KL_Code_Correction.md` | **`H029SG` / KP** — exclude from backup, keep in popup |
| `ZGATPDB_Popup_NT_MTD_1605_Daily_Code_Correction.md` | Month-wide SO sum **1605** vs daily **15** (different netting issue) |
| `ZGATPDB_PRIME_BCKU_Daily_ABAP_Code_Change.md` | Daily increment for **`inc_ord_quan`** save |

**This MD:** Regular material **`B120MA`** — future **EDATU** qty must **backup once**, then popup **nets to zero** next day.

---

## 10. Summary

| Question | Answer |
|----------|--------|
| Why popup shows **15** today? | `lt_so_daily_nt_sum` / `get_zapo_so_list` still find the future EDATU order in **`sy-datum..+2`** |
| Why not **0**? | **`lv_saved_nt`** from yesterday's backup is read but **not subtracted** (AD1 line ~2658) |
| What should backup have done yesterday? | Save **`NO_TOUCH = 15`** on **`ZDATE = yesterday`** |
| Primary fix | **`no_touch = daily_nt - lv_saved_nt`** in CHANGE F + net popup NT |
| Keep forward window? | **Yes** — required to capture future EDATU on backup day |

---

*End of document*
