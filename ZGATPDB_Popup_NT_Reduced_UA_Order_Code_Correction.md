# gATP MT/NT popup — No Touch reduced when UA orders are released

**Program:** `ZAPO_GATP_ALLOCATION_REPORT` / **T-code:** `ZGATPDB`  
**Include:** `ZAPO_GATP_ALLOCATION_F005`  
**Popup:** *gATP Manual & NO Touch Report* (`get_gatprep_data`)  
**Version:** 1.0 — 20/05/2026  

---

## 1. Your scenario (3 screenshots)

| Step | Action | Expected popup (PP / PMR) | Actual popup |
|------|--------|---------------------------|--------------|
| **1** | 3 orders × 21 MT via NT-MT / `ZAPO_SO_LIST` (`REQ_NT = KL`) | **NT = 63**, MT = 0 | **NT = 63** ✓ |
| **2** | 4th order 21 MT released via **UA Dashboard** | **NT = 63**, **MT = 21** | NT = **42**, MT = 21 ✗ |
| **3** | 5th order via UA | MT = 42, NT = 21 (unstable split) | ✗ |
| **4** | 6th order via UA | **NT = 63**, **MT = 63** | MT = **63**, NT = **0** ✗ |

**Business rule you described:**

- First 3 orders (63 MT) stay in the **No Touch** bucket.
- 4th order (21 MT) is a **separate Manual Touch** release.
- **Incoming = 84 MT** (63 + 21) → **MT = 21**, **NT = 63** (not 42 / 21).

**Observed pattern:** Every UA release **increases MT** but **reduces NT** — NT is treated as a **leftover** after MT, not a **fixed SO bucket**.

---

## 2. Root cause (AD1 code)

### 2.1 Wrong formula — NT = Incoming − MT (BR2 implemented incorrectly)

In **`get_data`**, CHANGE F (when `gt_zapoparam` / GATP new logic is on):

```abap
<fs_output>-manual_touch = lw_mt_adj_sum-usr_adj.   " Sum of UA today (ZAPO_ADB_ADJ)
<fs_output>-no_touch     = <fs_output>-inc_ord_quan - <fs_output>-manual_touch.
```

| Field | Typical value | Issue |
|-------|---------------|--------|
| `inc_ord_quan` | PAG **`AEMENGE`** (~63), not full **84** | 4th order not in incoming |
| `manual_touch` | **21 → 42 → 63** (all UA today, summed per CVC) | Grows with each UA |
| `no_touch` | **63−21=42**, then **63−63=0** | NT **shrinks** as MT grows |

So after 6 UA releases: **MT = 63**, **NT = 0** — exactly screenshot 3.

This is wrong for your process: **NT quantity from NT-classified SO lines must not be reduced when UA adds MT.**

---

### 2.2 `ZAPO_SO_LIST` split — UA lines move from NT to MT aggregate

**`get_zapo_so_list`** (~3274–3277):

```abap
lt_apo_so_list_mt = lt_apo_so_list_nt = gt_apo_so_list.
DELETE lt_apo_so_list_mt WHERE req_mt NE 'KL'.
DELETE lt_apo_so_list_nt WHERE req_nt NE 'KL'.
```

When the 4th order is UA-released, it often gets **`REQ_MT = KL`** (and not `REQ_NT = KL`). Then:

- **`gw_apo_so_nt_pp_plant`** drops by **21** (63 → **42**)
- **`gw_apo_so_mt_pp_plant`** increases by **21**

Popup then mixes **grid** `no_touch` (42) with this SO add-on → **42** NT instead of **63**.

---

### 2.3 Popup build — grid + global SO totals

**`get_gatprep_data`** (~6117–6232):

1. Sums **`gw_output-no_touch`** / **`manual_touch`** from selected ALV rows.
2. Calls **`get_zapo_so_list`** and **adds** `gw_apo_so_nt_pp_plant`, `gw_apo_so_mt_pp_plant`, etc.

```abap
gw_ntch_pp_plant = gw_ntch_pp_plant + gw_output-no_touch.   " grid (wrong NT)
...
go_alloc->get_zapo_so_list( ... ).
gw_ntch_pp_plant = gw_ntch_pp_plant + gw_apo_so_nt_pp_plant. " SO NT (shrinks on UA)
gw_mtch_pp_plant = gw_mtch_pp_plant + gw_apo_so_mt_pp_plant.
```

There is **no rule** that keeps **initial NT SO quantity (63)** independent of UA.

---

## 3. Target behaviour (functional rule)

Use **two independent buckets** (do not derive NT from a single incoming minus MT):

| Bucket | Source | Popup / grid |
|--------|--------|----------------|
| **No Touch (NT)** | `SUM( BMENG )` from `ZAPO_SO_LIST` where **`REQ_NT = 'KL'`** (and DLBL / SORG rules from other MD) | Fixed **63** until NT-classified SO qty changes |
| **Manual Touch (MT)** | `SUM( USR_ADJ )` from `ZAPO_ADB_ADJ` where **`UA_APPROV_STATUS IN ('A','AA')`** and **`APPROVED_DATE = sy-datum`** | **21** per UA order → **21, 42, 63** |
| **Incoming (informational)** | **NT SO + MT SO** (or monthly cap), not PAG-only | **84** after 4th order |

```text
NT  := SUM( SO.BMENG WHERE REQ_NT = 'KL' )     " not (incoming − MT)
MT  := SUM( UA.USR_ADJ for today per CVC )
```

Optional check: `NT + MT ≤ Incoming` (84); they may overlap in reporting but **NT must not decrease** when only MT UA is added.

---

## 4. Code correction

### 4.1 STEP 1 — Build per-CVC SO NT/MT totals in `get_data` (CHANGE C)

**After** `lt_so_all` is filled and before CHANGE F loop, aggregate by CVC:

```abap
    TYPES: BEGIN OF lty_so_cvc_ntmt,
             gccode TYPE zkungc,
             matnr  TYPE matnr,
             werks  TYPE zwerks,
             vtweg  TYPE vtweg,
             spart  TYPE spart,
             nt_qty TYPE zbmeng,
             mt_qty TYPE zbmeng,
             tot_qty TYPE zbmeng,
           END OF lty_so_cvc_ntmt.

    DATA: lt_so_cvc_ntmt TYPE HASHED TABLE OF lty_so_cvc_ntmt
          WITH UNIQUE KEY gccode matnr werks vtweg spart,
          ls_so_cvc      TYPE lty_so_cvc_ntmt.

    CLEAR lt_so_cvc_ntmt.
    LOOP AT lt_so_all INTO lw_so_all.
      " Apply same cap filters as lt_so_cap (abgru, lifsk, vkorg 1020, etc.)
      PERFORM zapo_so_line_cap_relevant
        USING    lw_so_all-vkorg lw_so_all-lifsk
        CHANGING DATA(lv_ok).
      CHECK lv_ok = abap_true.

      CLEAR ls_so_cvc.
      ls_so_cvc-gccode = lw_so_all-gccode.
      ls_so_cvc-matnr  = lw_so_all-matnr.
      ls_so_cvc-werks  = lw_so_all-werks.
      ls_so_cvc-vtweg  = lw_so_all-vtweg.
      ls_so_cvc-spart  = lw_so_all-spart.
      ls_so_cvc-tot_qty = lw_so_all-bmeng.
      COLLECT ls_so_cvc INTO lt_so_cvc_ntmt.

      READ TABLE lt_so_cvc_ntmt INTO ls_so_cvc
        WITH KEY gccode = lw_so_all-gccode
                 matnr  = lw_so_all-matnr
                 werks  = lw_so_all-werks
                 vtweg  = lw_so_all-vtweg
                 spart  = lw_so_all-spart.
      IF lw_so_all-req_nt = 'KL' OR lw_so_all-req_nt = 'Z1'
      OR lw_so_all-req_nt = 'KSV'.
        ls_so_cvc-nt_qty = ls_so_cvc-nt_qty + lw_so_all-bmeng.
      ENDIF.
      IF lw_so_all-req_mt = 'KL' OR lw_so_all-req_mt = 'Z1'
      OR lw_so_all-req_mt = 'KSV'.
        ls_so_cvc-mt_qty = ls_so_cvc-mt_qty + lw_so_all-bmeng.
      ENDIF.
      MODIFY lt_so_cvc_ntmt FROM ls_so_cvc.
    ENDLOOP.
```

> Add **`lifsk`** to `lty_so_all` SELECT if using delivery-block filters (`ZGATPDB_SORG1020_DLBL_NT_MT_Code_Correction.md`).

---

### 4.2 STEP 2 — Replace NT = inc − MT in CHANGE F (`get_data`)

**Find** (all occurrences — ~2532, ~2710, ~2928):

```abap
<fs_output>-no_touch = <fs_output>-inc_ord_quan - <fs_output>-manual_touch.
```

**Replace** with (inside `READ TABLE gt_zapoparam ... GATP` block):

```abap
                      DATA: lw_so_nt TYPE zbmeng,
                            lw_so_mt TYPE zbmeng,
                            lw_so_in TYPE zbmeng.

                      CLEAR: lw_so_nt, lw_so_mt, lw_so_in.
                      READ TABLE lt_so_cvc_ntmt INTO ls_so_cvc
                        WITH KEY gccode = <fs_output>-grp_cust
                                 matnr  = <fs_output>-material
                                 werks  = <fs_output>-location
                                 vtweg  = <fs_output>-dist_chan
                                 spart  = <fs_output>-div.
                      IF sy-subrc = 0.
                        lw_so_nt = ls_so_cvc-nt_qty.
                        lw_so_mt = ls_so_cvc-mt_qty.
                        lw_so_in = ls_so_cvc-nt_qty + ls_so_cvc-mt_qty.
                      ENDIF.

*--- MT: UA adjustments today (BR1) — independent bucket
                      CLEAR lw_mt_adj_sum.
                      READ TABLE lt_mt_adj_sum INTO lw_mt_adj_sum
                           WITH KEY product      = <fs_output>-material
                                    location     = <fs_output>-location
                                    division     = <fs_output>-div
                                    group_cust   = <fs_output>-grp_cust
                                    dist_channel = <fs_output>-dist_chan.
                      IF sy-subrc = 0.
                        <fs_output>-manual_touch = lw_mt_adj_sum-usr_adj.
                      ELSE.
                        CLEAR <fs_output>-manual_touch.
                      ENDIF.

*--- NT: SO REQ_NT bucket — do NOT subtract MT / UA (fix popup reduction)
                      <fs_output>-no_touch = lw_so_nt.

*--- Incoming for cap/display: SO NT + SO MT (or keep PAG + credit logic above)
                      IF lw_so_in > 0.
                        <fs_output>-inc_ord_quan = lw_so_in.
                      ENDIF.
                      " Else keep existing inc_ord_quan from PAG + lw_cb_qty

                      IF <fs_output>-no_touch < 0.
                        CLEAR <fs_output>-no_touch.
                      ENDIF.
                      IF <fs_output>-manual_touch < 0.
                        CLEAR <fs_output>-manual_touch.
                      ENDIF.
```

**Remove** the old lines that set `no_touch` from `inc_ord_quan - manual_touch` in this branch.

---

### 4.3 STEP 3 — Popup `get_gatprep_data` (avoid double / wrong NT)

**Option A (recommended when GATP param ON):** Use **SO list totals only** for division columns; use grid only for **MT (UA)** if SO MT is zero.

After `get_zapo_so_list` (~6226), when `gt_zapoparam` is active:

```abap
  READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
  IF sy-subrc = 0.
*--- Popup: NT from SO_LIST; MT from grid UA (already in gw_mtch_* from loop)
    CLEAR: gw_ntch_tot_plant, gw_ntch_pe_plant, gw_ntch_pp_plant,
           gw_ntch_pvc_plant, gw_ntch_elstmr_plant,
           gw_ntch_tot_depot, gw_ntch_pe_depot, gw_ntch_pp_depot,
           gw_ntch_pvc_depot, gw_ntch_elstmr_depot.

    gw_ntch_tot_plant = gw_apo_so_nt_tot_plant.
    gw_ntch_pe_plant  = gw_apo_so_nt_pe_plant.
    gw_ntch_pp_plant  = gw_apo_so_nt_pp_plant.
    gw_ntch_pvc_plant = gw_apo_so_nt_pvc_plant.
    gw_ntch_elstmr_plant = gw_apo_so_nt_elstmr_plant.
    gw_ntch_tot_depot = gw_apo_so_nt_tot_depot.
    " ... depot analog

    " MT: keep gw_mtch_* from grid loop + add SO MT bucket
    gw_mtch_tot_plant = gw_mtch_tot_plant + gw_apo_so_mt_tot_plant.
    gw_mtch_pp_plant  = gw_mtch_pp_plant  + gw_apo_so_mt_pp_plant.
    " ... other divs
  ELSE.
    " Legacy: existing grid + get_zapo_so_list add (unchanged)
    go_alloc->get_zapo_so_list( im_output = lt_output ).
    gw_ntch_pp_plant = gw_ntch_pp_plant + gw_apo_so_nt_pp_plant.
    ...
  ENDIF.
```

**Option B (minimal):** Keep current add logic but fix STEP 2 so grid `no_touch` is already **63**; then grid + SO add may **double** NT — use **either** grid **or** SO add, not both:

```abap
    go_alloc->get_zapo_so_list( im_output = lt_output ).
    READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
    IF sy-subrc = 0.
      " NT columns: SO bucket only (overwrite, do not add grid NT)
      gw_ntch_pp_plant = gw_apo_so_nt_pp_plant.
      gw_ntch_pe_plant  = gw_apo_so_nt_pe_plant.
      ...
    ELSE.
      gw_ntch_pp_plant = gw_ntch_pp_plant + gw_apo_so_nt_pp_plant.
    ENDIF.
```

---

### 4.4 STEP 4 — Data / master check (4th order)

Confirm in **`ZAPO_SO_LIST`** for the 4th order:

| Field | Expected for UA-only add-on |
|-------|-----------------------------|
| `REQ_NT` | Still **KL** on first 3 lines; 4th line may be MT-only |
| `REQ_MT` | **KL** on 4th after UA |
| `BMENG` | 21 on 4th |

If the 4th line **removes** `REQ_NT` on earlier lines, fix the **SO extract** (upstream), not only the report.

---

## 5. Expected results after fix

| Step | MT (Plant) PP | NT (Plant) PP |
|------|---------------|---------------|
| 3 NT orders | 0 | **63** |
| + 4th UA 21 | **21** | **63** |
| + 5th UA 21 | **42** | **63** |
| + 6th UA 21 | **63** | **63** |

Percent columns: recalc from `lw_total_pp_plant = MT + NT` (e.g. 63/84 and 63/84 for step 4 if total displayed is 84).

---

## 6. Test plan

| ID | Check |
|----|--------|
| T1 | 3×21 NT SO only → popup NT=63, MT=0 |
| T2 | +1×21 UA → popup NT=63, MT=21 (not 42) |
| T3 | SE16 `ZAPO_SO_LIST`: sum `BMENG` `REQ_NT=KL` = 63 after UA |
| T4 | `ZAPO_ADB_ADJ`: sum `USR_ADJ` today = 21 / 42 / 63 per step |
| T5 | +6th UA → MT=63, NT=63 (not NT=0) |
| T6 | Regression: credit block, monthly cap, `ZAPO_PRIME_BCKU` daily save |

---

## 7. Files to transport

| Object | Changes |
|--------|---------|
| `ZAPO_GATP_ALLOCATION_F005` | `get_data` — `lt_so_cvc_ntmt`, CHANGE F NT/MT |
| `ZAPO_GATP_ALLOCATION_F005` | `get_gatprep_data` — popup NT/MT source |
| Optional | `lifsk` / SORG 1020 filters in CHANGE C |

---

## 8. Related documents

| Topic | File |
|-------|------|
| SORG 1020 / DLBL FL BE OI DZ | `ZGATPDB_SORG1020_DLBL_NT_MT_Code_Correction.md` |
| `ZAPO_PRIME_BCKU` daily vs cumulative | `ZGATPDB_PRIME_BCKU_Daily_ABAP_Code_Change.md` |
| FS BR1/BR2 design | `FS_TS_NT_MT_Report_Revision.md` |

---

*AD1 reference: `get_data` CHANGE B/F ~464–504, ~2527–2533; `get_zapo_so_list` ~3274–3277; `get_gatprep_data` ~6076–6236.*
