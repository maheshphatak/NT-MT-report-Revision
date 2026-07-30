# ZGATPDB — Stock strategy (`REQ_NT='KL'`): future RDD must not appear until RDD day

**System:** DEVSCMAD1 (`ZAPO_GATP_ALLOCATION_F005` — last verified live source; AD1 MCP unavailable at write time)  
**Program:** `ZAPO_GATP_ALLOCATION_REPORT` / **T-code:** `ZGATPDB`  
**Param:** `NT_MT_NEW` / `REPORT_NEW`  
**Scope:** **Stock strategy only** — `ZAPO_SO_LIST-REQ_NT = 'KL'`  
**Do not change:** Z1 / KSV / UA / credit-block / GP / general `sy-datum..+2` logic for non-KL  
**Version:** 1.0 — 30/07/2026  

---

## 1. Functional rule (as requested)

| Condition (`REQ_NT = 'KL'`) | Current Day NT–MT Report / Dashboard | MTD |
|-----------------------------|--------------------------------------|-----|
| **Confirmed** and **VDATU/EDATU** from **month start → today** | Include (on the matching day logic below) | Include |
| **Confirmed today** but **VDATU/EDATU = future day** | **Do not** show today | **Do not** include until RDD ≤ today |
| Future RDD becomes **today** (`EDATU = sy-datum`) | **Show on that RDD day** | Then include in MTD |

**Confirmed** = `BMENG > 0` on `ZAPO_SO_LIST`.

**Daily NT for KL (dashboard / popup day bucket):**

```text
REQ_NT = 'KL'
AND BMENG > 0
AND EDATU = sy-datum          " show only on specific RDD day
```

**MTD for KL:**

```text
REQ_NT = 'KL'
AND BMENG > 0
AND EDATU BETWEEN month_start AND sy-datum   " 1st of month till today — no future
```

**Non-KL (`Z1` / `KSV`):** keep existing forward window `sy-datum .. sy-datum+2` unchanged.

---

## 2. Symptom

| Step | What happens today (bug) |
|------|---------------------------|
| 1 | Stock strategy order (`REQ_NT='KL'`) with **future EDATU** is confirmed |
| 2 | Appears in **today’s** NT–MT report (because window is `sy-datum..+2`) |
| 3 | Backed up / counted |
| 4 | **Next day** still in sliding window → appears again |

**Expected:** Future-RDD KL confirmed today → **hidden today**; appears **only when `sy-datum = EDATU`**.

---

## 3. Root cause (AD1)

### 3.1 Daily NT treats KL like Z1/KSV — forward window (~839)

```abap
lv_so_nt_edatu_high = sy-datum + 2.

IF lw_so_all-edatu >= sy-datum
AND lw_so_all-edatu <= lv_so_nt_edatu_high
AND ( req_nt = 'Z1' OR 'KSV' OR 'KL' ).   " ← KL included in +2 window
  COLLECT INTO lt_so_daily_nt_sum.
ENDIF.
```

### 3.2 `merge_kl_so_cvc_to_output` popup mode (~3501)

```abap
*--- Popup: sy-datum..sy-datum+2
lrw_date-low  = sy-datum.
lrw_date-high = sy-datum + 2.
...
AND req_nt = 'KL'.
```

### 3.3 `get_zapo_so_list` popup (~3734)

Same `sy-datum..+2` for all SO; KL lines with future EDATU are summed into popup NT.

---

## 4. Code corrections — **KL only**

### 4.1 CHANGE-KL-RDD1 — `get_data` CHANGE C: split daily NT date rule

**Location:** ~839–862. **Replace** the single IF with two branches.

```abap
lv_so_nt_edatu_high = sy-datum + 2.

*--- Non-KL (unchanged): forward window sy-datum..+2
IF lw_so_all-edatu >= sy-datum
AND lw_so_all-edatu <= lv_so_nt_edatu_high
AND ( lw_so_all-req_nt = 'Z1'
   OR lw_so_all-req_nt = 'KSV' ).
  CLEAR ls_so_daily_nt_sum.
  ls_so_daily_nt_sum-gccode = lw_so_all-gccode.
  ls_so_daily_nt_sum-matnr  = lw_so_all-matnr.
  ls_so_daily_nt_sum-werks  = lw_so_all-werks.
  ls_so_daily_nt_sum-vtweg  = lw_so_all-vtweg.
  READ TABLE gt_matclass INTO gw_matclass
    WITH KEY zgrade = lw_so_all-matnr BINARY SEARCH.
  IF sy-subrc = 0.
    ls_so_daily_nt_sum-spart = gw_matclass-zpdi.
  ELSE.
    ls_so_daily_nt_sum-spart = lw_so_all-spart.
  ENDIF.
  IF lw_so_all-bmeng > 0.
    ls_so_daily_nt_sum-daily_nt = lw_so_all-bmeng.
  ELSE.
    ls_so_daily_nt_sum-daily_nt = lw_so_all-wmeng.
  ENDIF.
  COLLECT ls_so_daily_nt_sum INTO lt_so_daily_nt_sum.
ENDIF.

*--- Stock strategy KL only: confirmed + EDATU = today (specific RDD day)
IF lw_so_all-req_nt = 'KL'
AND lw_so_all-bmeng > 0
AND lw_so_all-edatu = sy-datum.
  CLEAR ls_so_daily_nt_sum.
  ls_so_daily_nt_sum-gccode = lw_so_all-gccode.
  ls_so_daily_nt_sum-matnr  = lw_so_all-matnr.
  ls_so_daily_nt_sum-werks  = lw_so_all-werks.
  ls_so_daily_nt_sum-vtweg  = lw_so_all-vtweg.
  READ TABLE gt_matclass INTO gw_matclass
    WITH KEY zgrade = lw_so_all-matnr BINARY SEARCH.
  IF sy-subrc = 0.
    ls_so_daily_nt_sum-spart = gw_matclass-zpdi.
  ELSE.
    ls_so_daily_nt_sum-spart = lw_so_all-spart.
  ENDIF.
  ls_so_daily_nt_sum-daily_nt = lw_so_all-bmeng.  " confirmed only
  COLLECT ls_so_daily_nt_sum INTO lt_so_daily_nt_sum.
ENDIF.
```

**Do not** change the `lt_so_cap` / `lt_cb_ord` blocks above/below except where noted in §4.4 (optional MTD cap for KL).

---

### 4.2 CHANGE-KL-RDD2 — `merge_kl_so_cvc_to_output` date windows

**Location:** ~3484–3507.

```abap
IF lv_new_param = abap_true AND iv_popup_mode = abap_false.
*--- MTD / non-popup: month start → today (no future RDD)
  CONCATENATE sy-datum+0(6) '01' INTO lv_month_start.
  lrw_date-sign   = 'I'.
  lrw_date-option = 'BT'.
  lrw_date-low    = lv_month_start.
  lrw_date-high   = sy-datum.          " was: lw_month_end / full month
  APPEND lrw_date TO lrt_date.

ELSEIF lv_new_param = abap_true AND iv_popup_mode = abap_true.
*--- Daily popup: RDD = today only (not sy-datum..+2)
  lrw_date-sign   = 'I'.
  lrw_date-option = 'EQ'.
  lrw_date-low    = sy-datum.
  APPEND lrw_date TO lrt_date.

ELSE.
  " legacy unchanged
  ...
ENDIF.
```

**After SELECT** (or in the sum loop), keep only confirmed:

```abap
DELETE lt_fetch_kl  WHERE bmeng <= 0.
DELETE lt_fetch_all WHERE bmeng <= 0.
```

(Or add `AND bmeng > 0` to the SELECT if the field is available in the Open SQL.)

---

### 4.3 CHANGE-KL-RDD3 — `get_zapo_so_list` popup: filter KL after SELECT

**Location:** After `lt_apo_so_list` / `lt_apo_so_list_nt` is filled for popup (~3986).

**Do not** change `so_date` for the whole SELECT (Z1/KSV still need `..+2`).  
**After** building `lt_apo_so_list_nt`:

```abap
IF iv_popup_mode = abap_true.
  LOOP AT lt_apo_so_list_nt ASSIGNING FIELD-SYMBOL(<nt>).
    IF <nt>-req_nt = 'KL'.
      IF <nt>-edatu <> sy-datum OR <nt>-bmeng <= 0.
        DELETE lt_apo_so_list_nt.
        CONTINUE.
      ENDIF.
    ENDIF.
  ENDLOOP.
ENDIF.
```

Ensure `edatu` and `bmeng` are on `gty_apo_so_list` / `lt_apo_so_list_nt` (already selected in new-param popup path).

---

### 4.4 Optional — monthly KL cap (`lt_so_cap_sum`) for MTD headroom

Only if functional wants KL month cap to exclude future RDD as well. **Otherwise leave cap unchanged** (user: no other logic change).

If required:

```abap
" When adding KL into ls_so_cap_sum-nt_qty:
IF lw_so_all-req_nt = 'KL'.
  IF lw_so_all-bmeng > 0
  AND lw_so_all-edatu >= lw_month_start
  AND lw_so_all-edatu <= sy-datum.
    ls_so_cap_sum-nt_qty = lv_so_line_qty.
  ENDIF.
ELSE.
  " existing Z1/KSV nt_qty assign
ENDIF.
```

Default recommendation: **skip §4.4** unless MTD/cap still inflates with future KL.

---

## 5. What must **not** change

| Area | Action |
|------|--------|
| `REQ_NT` Z1 / KSV daily window `sy-datum..+2` | **Unchanged** |
| UA / `ZAPO_ADB_ADJ` MT | **Unchanged** |
| Credit block / GP / LIFSK DELETE | **Unchanged** |
| Dashboard exclude stock strategy from ALV (separate MD) | **Unchanged** |
| Future EDATU backup netting for **non-KL** | **Unchanged** |

---

## 6. Behaviour matrix

| KL order | EDATU | Confirmed | Today’s NT dashboard/popup | MTD |
|----------|-------|-----------|----------------------------|-----|
| Stock strategy | **Today** | Yes (`BMENG>0`) | **Include** | Include |
| Stock strategy | **Past** (this month) | Yes | Not in *daily* bucket (`EDATU≠today`); in MTD | Include |
| Stock strategy | **Future** | Yes (confirmed today) | **Exclude** | Exclude until RDD ≤ today |
| Stock strategy | Future | Yes | On **RDD day** only | Then include |
| Z1 / KSV | Future within +2 | — | Still included (old rule) | Unchanged |

---

## 7. Test plan

| # | Scenario | Expected |
|---|----------|----------|
| T1 | KL, `EDATU = sy-datum`, `BMENG > 0` | In today’s NT popup / `no_touch` |
| T2 | KL, `EDATU = sy-datum + 1`, confirmed today | **Not** in today’s NT |
| T3 | Same order as T2 on next day (`sy-datum = EDATU`) | **In** that day’s NT |
| T4 | KL, `EDATU` earlier this month, confirmed | In **MTD**; not re-added as “today daily” unless RDD=today |
| T5 | Z1/KSV with `EDATU = sy-datum + 1` | Still in daily NT (unchanged) |
| T6 | KL unconfirmed `BMENG = 0` | **Not** in daily NT / MTD KL path |

**SQL**

```sql
SELECT vbeln, matnr, werks, edatu, bmeng, wmeng, req_nt
  FROM zapo_so_list
 WHERE req_nt = 'KL'
   AND delind = ' '
   AND abgru  = ' '
   AND bmeng  > 0
   AND edatu  > sy-datum;   -- must NOT drive today's KL NT
```

---

## 8. Transport checklist

| # | Object | Change |
|---|--------|--------|
| 1 | `ZAPO_GATP_ALLOCATION_F005` | §4.1 — KL daily NT: `EDATU = sy-datum` + `BMENG > 0` |
| 2 | Same | §4.2 — `merge_kl`: popup EQ today; MTD month_start→today |
| 3 | Same | §4.3 — `get_zapo_so_list`: drop future/unconfirmed KL |
| 4 | Test | T1–T5 |

---

## 9. Related MDs

| MD | Topic |
|----|-------|
| `ZGATPDB_Future_EDATU_NextDay_NT_Reappear_After_Backup_Code_Correction.md` | Non-KL / general backup netting |
| `ZGATPDB_NewParam_StockStrategy_Dashboard_Exclude_Popup_Only_Code_Correction.md` | KL not on ALV dashboard |
| `ZGATPDB_NT_MT_OLD_vs_NEW_KL_Stock_Strategy_Code_Correction.md` | Old vs New KL popup |

**This MD:** KL-only RDD calendar rule — future confirmed KL waits for **RDD day**.

---

## 10. Summary

| Question | Answer |
|----------|--------|
| Why KL reappears next day? | KL uses `sy-datum..+2` like Z1/KSV |
| KL daily rule | Confirmed + **`EDATU = sy-datum`** |
| KL MTD rule | Confirmed + **`EDATU` month start → today** |
| Future KL confirmed today | **Hidden** until RDD day |
| Z1/KSV | **No change** |

---

*End of document*
