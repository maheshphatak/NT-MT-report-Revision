# ZGATPDB — New param logic Plant-only (`1001`); Depot (`1002`) must stay on Old logic

**System:** DEVSCMAD1 (`ZAPO_GATP_ALLOCATION_F005` — last verified live source; AD1 MCP unavailable at write time)  
**Program:** `ZAPO_GATP_ALLOCATION_REPORT` / **T-code:** `ZGATPDB`  
**Param:** `NT_MT_NEW` / `REPORT_NEW` (`gt_zapoparam` / `param1 = 'GATP'`)  
**Version:** 1.0 — 30/07/2026  

---

## 1. Requirement

| Location type | New param (`NT_MT_NEW`) | Expected behaviour |
|---------------|-------------------------|--------------------|
| **Plant `1001`** | **Apply** all new NT/MT / credit-block / daily NT / ADB MT rules | New logic |
| **Depot `1002`** | **Do not apply** new param derivation | **Same as Old `NT_MT`** |

**Observation:** Credit-block changes under new param make **Depot** credit-block orders **visible** (incoming / NT / popup).  
**Old param:** Depot credit-block orders are **not** visible / not treated like plant CB.

---

## 2. Root cause — new-param blocks run for Depot

### 2.1 Credit-block add sits **outside** Plant `IF` (first CHANGE F ~2593–2653)

```abap
IF lw_loc_type_c = '1001'.
  " SO cap + MTD guard
ENDIF.                                    " ← closes too early (~2632)

*--- Add credit block orders (BR5)     " ← NO plant check
READ TABLE lt_cb_ord ...
<fs_output>-inc_ord_quan = <fs_output>-inc_ord_quan + lw_cb_qty.  " runs for 1002 ✗

IF lw_loc_type_c = '1001'.
  " daily incoming convert
ENDIF.

*--- BOTH PLANT + DEPOT: MT from ADB   " ← explicit depot include ✗
manual_touch = usr_adj.

*--- NT from lt_so_daily_nt_sum         " ← also depot ✗
no_touch = daily_nt - lv_saved_nt.

*--- inc_ord from monthly SO cap        " ← also depot ✗
inc_ord_quan = nt_qty + mt_qty.
```

| Step | Plant `1001` | Depot `1002` (bug) |
|------|--------------|---------------------|
| Cap / MTD guard | Yes (inside `IF 1001`) | Skipped |
| **Credit block → `inc_ord_quan`** | Yes | **Yes — wrong** |
| **MT from `ZAPO_ADB_ADJ`** | Yes | **Yes — wrong** |
| **NT from daily SO** | Yes | **Yes — wrong** |
| Legacy PAG NT/MT formula | Skipped (`ELSE`) | **Skipped** — depot never gets Old formula |

So under new param, **Depot never falls through to Old logic**, and **credit block is applied to Depot**.

### 2.2 Same pattern in other CHANGE F copies

| Block | Approx. area | Issue |
|-------|--------------|--------|
| Prime / PM loop #1 | ~2570–2760 | Credit + MT + NT **outside** `IF 1001` |
| Prime / PM loop #2 | ~2856–3050 | MT/NT comment “BOTH PLANT + DEPOT”; MT/NT after plant IF |
| NP loop | ~3185–3370 | Same “BOTH PLANT + DEPOT” MT/NT; ensure CB not applied to `1002` |

### 2.3 `lt_cb_ord` built for **all** plants/depots (~864)

```abap
IF lw_so_all-cmgst = 'B' ...
  COLLECT lw_cb_ord INTO lt_cb_ord.   " no loctype filter
ENDIF.
```

Depot werks are collected and later applied when CHANGE F has no `1001` guard.

### 2.4 Popup sums Depot NT/MT from grid (~2826–2828, ~3111+)

```abap
ELSEIF gw_loc_type-loc_type = '1002'.
  gw_mtch_tot_depot = ... + manual_touch.
  gw_ntch_tot_depot = ... + no_touch.
```

If Depot `no_touch` / `inc_ord_quan` were filled by new logic (incl. CB), **Depot credit appears in popup** — unlike Old param.

---

## 3. Target design

```text
IF new_param (gt_zapoparam GATP) AND loc_type = '1001'.
  " Full new CHANGE F: cap, CB, daily inc, ADB MT, daily NT, etc.
ELSE.
  " Old / legacy formula (existing ELSE branch)
  " Depot 1002 always here when new param is on
ENDIF.
```

| Feature | `1001` Plant | `1002` Depot |
|---------|--------------|--------------|
| Credit block → incoming / NT | New (visible if rules say so) | **Old — not visible** |
| ADB MT / daily NT / SO cap overwrite | New | **Legacy only** |
| Popup Depot row | From legacy grid fields | No new-param CB inflation |

---

## 4. Code corrections (`ZAPO_GATP_ALLOCATION_F005`)

### 4.1 CHANGE-P1 — Wrap **entire** new-param derivation in `IF loc_type = '1001'`

**Apply in every CHANGE F block** (PM ×2, NP ×1).

**Structural pattern:**

```abap
READ TABLE gt_zapoparam ... WITH KEY param1 = 'GATP' value1 = <fs_output>-div.
IF sy-subrc = 0.

  READ TABLE gt_loc_type INTO gw_loc_type
    WITH KEY loc = <fs_output>-location BINARY SEARCH.
  IF sy-subrc = 0.
    lw_loc_type_c = gw_loc_type-loc_type.
  ENDIF.

*--- NEW PARAM: Plant only
  IF lw_loc_type_c = '1001'.

    " 1) SO cap + prior MTD guard
    " 2) Credit block add (BR5)          ← MUST be inside this IF
    " 3) Daily incoming convert
    " 4) MT from ZAPO_ADB_ADJ
    " 5) NT from lt_so_daily_nt_sum (− prior backup)
    " 6) Optional: set inc_ord from daily/cap per plant rules
    " 7) Clear negatives

  ELSE.
*--- Depot 1002 (and any non-1001): OLD PARAM formula — unchanged ELSE body
    IF <fs_output>-alloc_quan = 0.
      <fs_output>-no_touch = <fs_output>-inc_ord_quan.
    ELSE.
      " ... existing legacy NT/MT derivation ...
    ENDIF.
    <fs_output>-manual_touch = <fs_output>-inc_ord_quan - <fs_output>-no_touch.
    IF <fs_output>-manual_touch < 0.
      CLEAR <fs_output>-manual_touch.
    ENDIF.
  ENDIF.

ELSE.
  " Old param entirely (existing legacy) — unchanged
  ...
ENDIF.
```

**Critical move:** Cut the block currently **after** `ENDIF` of plant cap (~2634–2761 in first copy) **into** the `IF lw_loc_type_c = '1001'` branch.  
**Delete** comments like `BOTH PLANT + DEPOT` for new-param MT/NT — replace with `PLANT ONLY (1001)`.

---

### 4.2 CHANGE-P2 — First CHANGE F: fix premature `ENDIF` (~2632)

**Current (wrong):**

```abap
IF lw_loc_type_c = '1001'.
  " cap + MTD guard
ENDIF.                    " ← too early

" credit block          " runs for depot
" MT / NT / SO cap      " runs for depot
ELSE.                     " legacy — depot never reaches this under new param
```

**Correct:** One `IF 1001` … `ELSE` (legacy) … `ENDIF` covering credit + MT + NT.

---

### 4.3 CHANGE-P3 — Build `lt_cb_ord` for Plant werks only (optional harden)

**Location:** CHANGE C (~864), when collecting credit-block qty.

```abap
IF lw_so_all-cmgst = 'B'
AND lw_so_all-abgru = space
AND ( req_nt/mt Z1/KSV/KL ).

  READ TABLE gt_loc_type INTO gw_loc_type
    WITH KEY loc = lw_so_all-werks BINARY SEARCH.
  IF sy-subrc = 0 AND gw_loc_type-loc_type = '1001'.
    " COLLECT into lt_cb_ord  — plant only
    ...
  ENDIF.
ENDIF.
```

Prevents depot CB qty sitting in `lt_cb_ord` even if a future bug re-applies the READ.

---

### 4.4 CHANGE-P4 — Popup: do not feed new-param Depot NT from CB

After CHANGE-P1, Depot `no_touch` / `manual_touch` come from **legacy** only → popup Depot rows match Old param.

**Do not** add a special new-param path that forces CB into `gw_ntch_*_depot`.

If `get_zapo_so_list` still adds SO NT to Depot under new param and that surfaces CB, gate:

```abap
" When accumulating gw_apo_so_nt_*_depot under new param:
" either skip depot add for new param, or exclude cmgst = 'B' for depot only
```

Preferred: **skip new-param SO NT/MT add-on for depot** (plant popup already uses grid `no_touch` for new param). Old param additive SO for depot can remain if that was Old behaviour.

---

### 4.5 CHANGE-P5 — PAG row keep for credit (dashboard)

PAG `AEMENGE = 0` keep-for-new-param (~1031) must **not** force Depot CB CVCs onto the ALV if Old hid them.

```abap
IF lw_qttab-aemenge IS INITIAL AND p_report = abap_true.
  READ TABLE gt_zapoparam ... param1 = 'GATP'.
  IF sy-subrc = 0.
    " New param: only keep zero-AEMENGE rows for PLANT credit-relevant CVCs
    READ TABLE gt_loc_type INTO gw_loc_type
      WITH KEY loc = lw_qttab-locno BINARY SEARCH.  " use actual loc field
    IF sy-subrc <> 0 OR gw_loc_type-loc_type <> '1001'.
      IF lw_qttab-kcqty IS INITIAL.
        CONTINUE.   " depot: same as old skip when empty
      ENDIF.
    ENDIF.
  ELSEIF lw_qttab-kcqty IS INITIAL.
    CONTINUE.
  ENDIF.
ENDIF.
```

Tune using your exact QTTAB location field name.

---

## 5. Inconsistency checklist (audit)

| # | New-param feature | Must be `1001` only? | Current issue |
|---|-------------------|----------------------|---------------|
| 1 | Credit block → `inc_ord_quan` | **Yes** | Outside plant IF → hits `1002` |
| 2 | ADB `manual_touch` | **Yes** (per this request) | Comment says BOTH |
| 3 | Daily `no_touch` from SO | **Yes** | Runs for `1002` |
| 4 | SO cap overwrite of incoming | **Yes** | Runs for `1002` |
| 5 | Cap / MTD guard | Already plant-only | OK |
| 6 | `lt_cb_ord` fill | Prefer plant-only | All werks |
| 7 | Popup Depot totals | Legacy / Old | Inflated if grid has new CB |
| 8 | KL RDD / future EDATU / UA rules | Plant-only if part of new param | Confirm not applied to depot rows |

---

## 6. Test plan

| # | Scenario | Expected |
|---|----------|----------|
| T1 | Plant `1001`, `CMGST=B`, new param | CB visible in plant NT/incoming per plant rules |
| T2 | Depot `1002`, `CMGST=B`, new param | **Not** visible like Old param (no CB top-up; legacy NT/MT) |
| T3 | Depot `1002`, normal (no CB), new param | Same as Old param depot |
| T4 | Plant UA / daily NT / KL RDD | Unchanged for `1001` |
| T5 | Popup MT/NT Depot row with CB SO | **0** / Old behaviour — not new CB qty |
| T6 | Old param (`NT_MT`) plant + depot | Regression: unchanged |

---

## 7. Transport checklist

| # | Object | Change |
|---|--------|--------|
| 1 | `ZAPO_GATP_ALLOCATION_F005` | §4.1–4.2 — entire CHANGE F new body inside `IF 1001` / `ELSE` legacy |
| 2 | Same | §4.3 — optional `lt_cb_ord` plant-only |
| 3 | Same | §4.4–4.5 — popup / PAG depot guards |
| 4 | Test | T1–T2 mandatory |

---

## 8. Summary

| Question | Answer |
|----------|--------|
| Should new param hit Depot? | **No** — **Plant `1001` only** |
| Why Depot CB appears? | Credit + MT + NT new logic runs **outside** `IF 1001` |
| Fix | One plant IF wrapping **all** new derivation; Depot → **Old ELSE** |
| Old param Depot CB | Remain **not visible** |

---

*End of document*
