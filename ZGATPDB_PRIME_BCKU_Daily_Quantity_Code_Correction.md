# ZGATPDB — `ZAPO_PRIME_BCKU` must store daily quantities (not cumulative)

**System:** DEVSCMAD1 (`ZAPO_GATP_ALLOCATION_F005`)  
**Table:** `ZAPO_PRIME_BCKU`  
**Fields:** `ZDATE` (run date), `INC_ORD_QUAN`, `MANUAL_TOUCH`, `NO_TOUCH`  
**Version:** 1.0 — 20/05/2026  

---

## 1. Symptom (your screenshot)

| Run date (`ZDATE`) | Incoming ord qty | No touch | Expected (daily) |
|--------------------|------------------|----------|------------------|
| **18.05.2026** | 210 | 210 | 210 / 210 |
| **19.05.2026** | 210 | 210 | **0 / 0** (no new qty that day) |
| **20.05.2026** | 245 | 245 | **35 / 35** (210 + 35 = 245 MTD) |

The table stores **month-to-date (MTD) totals** on every run date instead of **that date’s increment only**.

---

## 2. Root cause (AD1 code)

### 2.1 Save path stores `gw_output` values as-is

In **`collect_final_output`**, quantities written to backup are taken directly from **`gt_output`**:

```abap
<lfs_prime_buk>-inc_ord_quan  = gw_output-inc_ord_quan.
<lfs_prime_buk>-manual_touch  = gw_output-manual_touch.
<lfs_prime_buk>-no_touch      = gw_output-no_touch.
```

There is **no conversion** from MTD → daily before `MODIFY zapo_prime_bcku` in **`update_database`**.

### 2.2 `gw_output-inc_ord_quan` behaves like MTD

- Built from PAG **`/SAPAPO/QTTAB-AEMENGE`** (`get_data`, ~956) and/or **`lt_output_temp`** (historical dates rolled to `sy-datum`, ~999–1016).
- After NT/MT logic (CHANGE F), **`inc_ord_quan`** still reflects **cumulative / position** quantity for the CVC, not “today only”.
- On **19.05**, PAG can still show **210** MTD → program saves **210** again instead of **0**.

### 2.3 MTD sum is only used for guard — not for save

**CHANGE D** in **`get_data`** already reads prior **`ZAPO_PRIME_BCKU`** rows and sums them into **`lt_prime_sum`** / **`lw_saved_mtd`** (~595–634) for the **MTD guard** (`lw_saved_mtd >= lw_so_cap_qty` → skip row).

That sum is **not** subtracted before persisting to **`ZAPO_PRIME_BCKU`**.

### 2.4 `lw_remaining` never applied

**CHANGE F** declares **`lw_remaining`** but AD1 source does **not** assign  
`lw_remaining = lw_so_cap_qty - lw_saved_mtd` nor cap `inc_ord_quan` before save (planned in FS / HOTFIX, not wired in `collect_final_output`).

---

## 3. Required behaviour

For each CVC and **`ZDATE = sy-datum`**:

| Field | Daily value to save |
|-------|---------------------|
| **Incoming ord qty** | `MAX( 0, lw_mtd_inc - lw_prior_inc )` |
| **Manual touch** | `MAX( 0, lw_mtd_mt - lw_prior_mt )` *or* keep today’s UA sum if already daily |
| **No touch** | `MAX( 0, lw_daily_inc - lw_daily_mt )` after daily split |

Where:

- **`lw_mtd_inc`** = `gw_output-inc_ord_quan` after `get_data` (incl. credit add-on).
- **`lw_prior_inc`** = **SUM** of `inc_ord_quan` from **`ZAPO_PRIME_BCKU`** for same CVC, same `prime_sto`, **`zdate` from month start through `sy-datum - 1`**.
- Same pattern for **`manual_touch`** / **`no_touch`** if those are also MTD-like in output.

**Your example:**

- 18.05: prior = 0, MTD = 210 → **save 210**
- 19.05: prior = 210, MTD = 210 → **save 0**
- 20.05: prior = 210, MTD = 245 → **save 35**

---

## 4. Code correction — `collect_final_output`

**Include:** `ZAPO_GATP_ALLOCATION_F005`  
**Method:** `collect_final_output`  
**Insert:** After `lw_sto` is set (~3534–3538), **before** `LOOP AT lt_output`.

### 4.1 Types and data (reuse CHANGE D pattern from `get_data`)

```abap
    DATA: lw_month_start TYPE datum,
          lv_sydautm     TYPE datum.

    TYPES: BEGIN OF lty_prime_mtd_sum,
             material     TYPE zapo_prime_bcku-material,
             location     TYPE zapo_prime_bcku-location,
             div          TYPE zapo_prime_bcku-div,
             grp_cust     TYPE zapo_prime_bcku-grp_cust,
             dist_chan    TYPE zapo_prime_bcku-dist_chan,
             region       TYPE zapo_prime_bcku-region,
             ship_to_party TYPE zapo_prime_bcku-ship_to_party,
             prior_inc    TYPE zapo_prime_bcku-inc_ord_quan,
             prior_mt     TYPE zapo_prime_bcku-manual_touch,
             prior_nt     TYPE zapo_prime_bcku-no_touch,
           END OF lty_prime_mtd_sum.

    DATA: lt_prime_mtd_sum TYPE HASHED TABLE OF lty_prime_mtd_sum
          WITH UNIQUE KEY material location div grp_cust dist_chan
                          region ship_to_party,
          ls_prime_mtd     TYPE lty_prime_mtd_sum,
          lw_prime_mtd     TYPE lty_prime_mtd_sum,
          lw_daily_inc     TYPE zapo_prime_bcku-inc_ord_quan,
          lw_daily_mt      TYPE zapo_prime_bcku-manual_touch,
          lw_daily_nt      TYPE zapo_prime_bcku-no_touch.

    FIELD-SYMBOLS: <fs_mtd> TYPE lty_prime_mtd_sum.

    " Month start (same as CHANGE A in get_data)
    CONCATENATE sy-datum+0(6) '01' INTO lw_month_start.
    lv_sydautm = sy-datum - 1.

    SELECT material location div grp_cust dist_chan region ship_to_party
           SUM( inc_ord_quan )  AS prior_inc
           SUM( manual_touch ) AS prior_mt
           SUM( no_touch )     AS prior_nt
      FROM zapo_prime_bcku
      INTO TABLE @lt_prime_mtd_sum
      WHERE prime_sto = @lw_sto
        AND zdate BETWEEN @lw_month_start AND @lv_sydautm
        AND div IN @so_divi.
    IF sy-subrc <> 0.
      CLEAR lt_prime_mtd_sum.
    ENDIF.
```

### 4.2 Convert to daily before assign (both UPDATE and INSERT branches)

Replace direct assigns (~3604–3607 and ~3658–3661) with:

```abap
        CLEAR: lw_daily_inc, lw_daily_mt, lw_daily_nt, lw_prime_mtd.

        lw_daily_inc = gw_output-inc_ord_quan.
        lw_daily_mt  = gw_output-manual_touch.
        lw_daily_nt  = gw_output-no_touch.

        READ TABLE lt_prime_mtd_sum INTO lw_prime_mtd
          WITH KEY material      = gw_output-material
                   location      = gw_output-location
                   div           = gw_output-div
                   grp_cust      = gw_output-grp_cust
                   dist_chan     = gw_output-dist_chan
                   region        = gw_output-region
                   ship_to_party = gw_output-ship_to_party.
        IF sy-subrc = 0.
          lw_daily_inc = lw_daily_inc - lw_prime_mtd-prior_inc.
          lw_daily_mt  = lw_daily_mt  - lw_prime_mtd-prior_mt.
          lw_daily_nt  = lw_daily_nt  - lw_prime_mtd-prior_nt.
        ENDIF.

        IF lw_daily_inc < 0. CLEAR lw_daily_inc. ENDIF.
        IF lw_daily_mt  < 0. CLEAR lw_daily_mt.  ENDIF.
        IF lw_daily_nt  < 0. CLEAR lw_daily_nt.  ENDIF.

        " Prefer NT from daily inc − daily MT (consistent with BR2)
        lw_daily_nt = lw_daily_inc - lw_daily_mt.
        IF lw_daily_nt < 0. CLEAR lw_daily_nt. ENDIF.

        <lfs_prime_buk>-inc_ord_quan  = lw_daily_inc.
        <lfs_prime_buk>-manual_touch  = lw_daily_mt.
        <lfs_prime_buk>-no_touch      = lw_daily_nt.
```

Use the **same block** for new rows (`gw_prime_buk_t-...`).

### 4.3 Optional — persist zero rows on quiet days

If no PAG row exists on **19.05**, **`lt_output`** may have **no** line → no `MODIFY` → old wrong row might remain from a previous bug, or no row at all.

If business needs **explicit 0** rows when the job runs daily:

- Add a second pass over **yesterday’s** `ZAPO_PRIME_BCKU` keys not in `lt_output`, with daily qty **0**, **or**
- Ensure batch always builds `gt_output` for all active CVCs (even with `aemenge = 0`).

---

## 5. Align `get_data` display with daily save (recommended)

**CHANGE F** uses **`lw_saved_mtd`** only to **CLEAR** the row when cap reached; it does **not** set **`inc_ord_quan`** to daily remainder.

Optional enhancement in **`get_data`** (plant, when `gt_zapoparam` active), **after** credit add and **before** MT/NT:

```abap
lw_remaining = lw_so_cap_qty - lw_saved_mtd.
IF lw_remaining < 0. CLEAR lw_remaining. ENDIF.

" Daily incoming for today (same formula as save)
<fs_output>-inc_ord_quan = <fs_output>-inc_ord_quan - lw_saved_mtd.
IF <fs_output>-inc_ord_quan < 0.
  CLEAR <fs_output>-inc_ord_quan.
ENDIF.

" Optional: also cap to SO headroom
IF <fs_output>-inc_ord_quan > lw_remaining AND lw_so_cap_qty > 0.
  <fs_output>-inc_ord_quan = lw_remaining.
ENDIF.

" Then MT from ZAPO_ADB_ADJ and NT = inc_ord - MT
```

This keeps **ALV / backup** consistent so `collect_final_output` can still apply §4.2 as a safety net.

---

## 6. What not to change

| Item | Reason |
|------|--------|
| **`update_database`** | Already does `MODIFY ... FROM lw_prime_buk` per row; fix inputs in `collect_final_output` |
| **CHANGE D read for `lt_prime_sum`** | Keep for MTD guard; daily save uses same prior-month logic with **full CVC key** (incl. `region`, `ship_to_party`) |
| **Historical rows already wrong** | One-time correction / re-run from 1st of month may be needed after transport |

---

## 7. Test plan

| Step | Action | Expected in `ZAPO_PRIME_BCKU` |
|------|--------|-------------------------------|
| T1 | Run backup on **18.05** with MTD inc = 210 | `ZDATE=18.05`, inc = **210**, NT = **210** |
| T2 | Run on **19.05**, same CVC, PAG MTD still 210 | `ZDATE=19.05`, inc = **0**, NT = **0** |
| T3 | Run on **20.05**, PAG MTD = 245 | `ZDATE=20.05`, inc = **35**, NT = **35** |
| T4 | Sum `inc_ord_quan` for month | **245** (= 210 + 0 + 35), not 245+210+… double count |

---

## 8. Related documents

- `FS_TS_NT_MT_Report_Revision.md` — BR4 daily cap / MTD headroom  
- `MT_NT_Credit_Block_HOTFIX_Validation_AD1.txt` — `lw_remaining` cap (apply in `get_data` + daily delta in save)  
- `ZGATPDB_SORG1020_DLBL_NT_MT_Code_Correction.md` — delivery block / SORG 1020 (separate fix)  

---

## 9. Transport checklist

- [ ] `ZAPO_GATP_ALLOCATION_F005` — `collect_final_output` (§4.1–4.2)  
- [ ] Optional — `get_data` CHANGE F daily `inc_ord_quan` (§5)  
- [ ] Regression: `ZGATPDB` Prime backup + SE16 `ZAPO_PRIME_BCKU` by `ZDATE`  
- [ ] Clarify with functional: correct bad rows for May-2026 or wait until next month-start go-live  

---

*Analysis from AD1 include `ZAPO_GATP_ALLOCATION_F005`; `collect_final_output` ~3546–3677, `update_database` ~3818–3843.*
