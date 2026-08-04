# Implementation Guide — Stock Strategy in `ZAPO_PRIME_BCKU` (mid-month KL ↔ Allocation)

**System:** DEVSCMAD1  
**Program / Include:** `ZAPO_GATP_ALLOCATION_REPORT` / `ZAPO_GATP_ALLOCATION_F005`  
**T-code:** `ZGATPDB`  
**Param scope:** **New only** (`gt_zapoparam` `param1 = 'GATP'`)  
**Analysis doc:** `ZGATPDB_StockStrategy_Include_Backup_MidMonth_Switch_Code_Correction.md`  
**Version:** 1.0 — 04/08/2026  

---

## 0. Purpose

Implement today’s three mandatory changes so Stock Strategy (`REQ_NT = 'KL'`) is stored in `ZAPO_PRIME_BCKU` under New Param, NT nets like Allocation Strategy, and mid-month KL ↔ Allocation keeps history intact.

| ID | Method | What |
|----|--------|------|
| **SS1** | `collect_final_output` | Old-only KL exclude; New append KL CVCs to backup |
| **SS2** | `merge_kl_so_cvc_to_output` | New Plant KL qty = netted NT (SO − MTD BCKU) |
| **SS3** | `pull_data_prime_buck` | New Plant: stop KL SO additive (avoid double count) |

**Do not touch:** Old param paths, Depot legacy KL add, CHANGE F Plant formula, KL RDD Plant filters, FIX-KL-POP structure.

---

## 1. Pre-checks (before edit)

| # | Check | How |
|---|--------|-----|
| 1 | Open include on AD1 | SE80 / ADT → `ZAPO_GATP_ALLOCATION_F005` |
| 2 | Confirm FIX-KL-BAK Obs2 exists | Search `FIX-KL-BAK` or `lt_kl_chk` |
| 3 | Confirm New param flag usage | Search `param1 = 'GATP'` |
| 4 | Confirm MTD KL add comment | Search `KL not in ZAPO_PRIME_BCKU` |
| 5 | Create transport | Workbench TR for include only |
| 6 | Note a test CVC | One Plant `1001` material with `REQ_NT='KL'` today |

**Backup of source:** Download current F005 version into TR before change (or copy to local).

---

## 2. Implementation order (mandatory)

```text
1) SS2  — merge_kl netting          (qty correct before append/save)
2) SS1  — collect_final_output      (allow + append KL to backup)
3) SS3  — pull_data_prime_buck      (stop Plant New double count)
4) Syntax check → activate
5) Unit test T1–T6
```

Do **not** activate SS1 without SS3 — MTD would double-count Plant KL.

---

## 3. STEP SS2 — `merge_kl_so_cvc_to_output` (netted qty)

### 3.1 Find

Search for:

```abap
IF lv_new_param = abap_true.
  CLEAR lv_so_nt_kl.
  LOOP AT lt_fetch_all INTO lw_fetch_sum
```

(inside `merge_kl_so_cvc_to_output`, after building `lw_fetch_outtab` keys).

### 3.2 Add local DATA (top of method, with other DATA)

```abap
DATA: lv_saved_prime TYPE zbmeng,
      lv_sum_mt      TYPE zapo_prime_bcku-manual_touch,
      lv_sum_nt      TYPE zapo_prime_bcku-no_touch,
      lv_month_start TYPE dats,
      lv_yesterday   TYPE dats.
```

(Reuse existing `lv_month_start` if already declared in this method — do not duplicate.)

### 3.3 Replace New-param qty block

**Replace** the block that only does:

```abap
lv_so_nt_kl = lv_so_nt_kl + lw_fetch_sum-bmeng.
...
lw_fetch_outtab-no_touch     = lv_so_nt_kl.
lw_fetch_outtab-inc_ord_quan = lv_so_nt_kl.
```

**With:**

```abap
        IF lv_new_param = abap_true.
          CLEAR lv_so_nt_kl.
          LOOP AT lt_fetch_all INTO lw_fetch_sum
            WHERE matnr  = lw_fetch_outtab-material
              AND werks  = lw_fetch_outtab-location
              AND vtweg  = lw_fetch_outtab-dist_chan
              AND spart  = lw_fetch_outtab-div
              AND gccode = lw_fetch_outtab-grp_cust.
            lv_so_nt_kl = lv_so_nt_kl + lw_fetch_sum-bmeng.
          ENDLOOP.

*--- SS2: New — net prior MTD BCKU (MT+NT), same spirit as CHANGE F
          CLEAR: lv_saved_prime, lv_sum_mt, lv_sum_nt.
          CONCATENATE sy-datum+0(6) '01' INTO lv_month_start.
          lv_yesterday = sy-datum - 1.
          IF lv_yesterday >= lv_month_start.
            SELECT SUM( manual_touch )
                   SUM( no_touch )
              INTO (lv_sum_mt, lv_sum_nt)
              FROM zapo_prime_bcku
              WHERE zdate     BETWEEN lv_month_start AND lv_yesterday
                AND material  = lw_fetch_outtab-material
                AND location  = lw_fetch_outtab-location
                AND div       = lw_fetch_outtab-div
                AND grp_cust  = lw_fetch_outtab-grp_cust
                AND dist_chan = lw_fetch_outtab-dist_chan.
            IF sy-subrc = 0.
              lv_saved_prime = lv_sum_mt + lv_sum_nt.
            ENDIF.
          ENDIF.

          IF lv_so_nt_kl > lv_saved_prime.
            lv_so_nt_kl = lv_so_nt_kl - lv_saved_prime.
          ELSE.
            CLEAR lv_so_nt_kl.
          ENDIF.

          lw_fetch_outtab-no_touch     = lv_so_nt_kl.
          lw_fetch_outtab-inc_ord_quan = lv_so_nt_kl.
          CLEAR lw_fetch_outtab-manual_touch.
        ENDIF.
```

### 3.4 SS2 guardrails

| Keep | Do not change |
|------|----------------|
| Plant RDD filter (`CHANGE-KL-RDD2`) above this block | Old-param `ELSE` date window |
| Depot rows in `lt_fetch_keep` | `get_data` / CHANGE F |

---

## 4. STEP SS1 — `collect_final_output` (include KL in backup)

### 4.1 Find block A — Obs2 exclude

Search:

```abap
*--- Pure stock-strategy KL: no PAG / no UA -> never write ZAPO_PRIME_BCKU
```

### 4.2 Replace exclude with Old-param-only gate

**Replace:**

```abap
      IF lv_kl_hit = abap_true
         AND gw_output-alloc_quan   = 0
         AND gw_output-manual_touch = 0
         AND gw_output-p_usr_adj    = 0.
        CONTINUE.
      ENDIF.
```

**With:**

```abap
*--- SS1A: Old param only — pure KL never in ZAPO_PRIME_BCKU (legacy Obs2)
*--- New param — allow Stock Strategy backup (mid-month KL↔Allocation)
      READ TABLE gt_zapoparam TRANSPORTING NO FIELDS
        WITH KEY param1 = 'GATP'.
      IF sy-subrc <> 0.
        IF lv_kl_hit = abap_true
           AND gw_output-alloc_quan   = 0
           AND gw_output-manual_touch = 0
           AND gw_output-p_usr_adj    = 0.
          CONTINUE.
        ENDIF.
      ENDIF.
```

### 4.3 Find block B — after `merge_kl` / `SORT lt_kl_chk`

Immediately **after**:

```abap
    SORT lt_kl_chk BY material location dist_chan div grp_cust.
```

**Insert:**

```abap
*--- SS1B: New param — append missing KL CVCs into lt_output for backup
    READ TABLE gt_zapoparam TRANSPORTING NO FIELDS
      WITH KEY param1 = 'GATP'.
    IF sy-subrc = 0 AND lt_kl_chk IS NOT INITIAL.
      SORT lt_output BY material location dist_chan div grp_cust.
      LOOP AT lt_kl_chk INTO lw_kl_chk.
        READ TABLE lt_output TRANSPORTING NO FIELDS
          WITH KEY material  = lw_kl_chk-material
                   location  = lw_kl_chk-location
                   dist_chan = lw_kl_chk-dist_chan
                   div       = lw_kl_chk-div
                   grp_cust  = lw_kl_chk-grp_cust
          BINARY SEARCH.
        IF sy-subrc <> 0.
          lw_kl_chk-date = sy-datum.
          APPEND lw_kl_chk TO lt_output.
        ENDIF.
      ENDLOOP.
    ENDIF.
```

### 4.4 Update comment at merge call

Change:

```abap
*--- FIX-KL-BAK Obs2: KL list via EXISTING merge - do NOT append to lt_output
```

To:

```abap
*--- SS1: KL list via merge_kl — New param appends missing CVCs (SS1B); Old keeps Obs2 exclude
```

### 4.5 Optional hardening — refresh today’s BCKU keys after append

If pure KL was missing from `lt_ouput_t` before SELECT of today’s BCKU, either:

- Move SS1B **before** `lt_ouput_t = lt_output` / FAE SELECT, **or**
- After SS1B, rebuild `lt_ouput_t` and re-SELECT today’s `gt_prime_buk` for new keys.

**Recommended order inside method:**

```text
1. merge_kl → lt_kl_chk
2. SS1B append KL into lt_output          ← insert here
3. lt_ouput_t = lt_output + FAE SELECT today BCKU
4. loc type load
5. LOOP lt_output → SS1A Old exclude → MODIFY buffer
```

If current code builds `lt_ouput_t` **before** SS1B, **move** SS1B up so FAE includes new KL materials.

---

## 5. STEP SS3 — `pull_data_prime_buck` (no Plant New KL SO add)

### 5.1 Find

Search:

```abap
*--- MTD: BCKU + KL additive (legacy + new param)
*--- New param: KL not in ZAPO_PRIME_BCKU (Obs2) -> add from SO like old
```

### 5.2 Update header comment

```abap
*--- MTD: BCKU + KL SO additive
*--- SS3: New Plant(1001) — KL now in ZAPO_PRIME_BCKU → do NOT add from SO
*---      Old + Depot(1002) — keep KL SO additive
```

### 5.3 Inside both KL loops (`lt_apo_so_list_mt` and `lt_apo_so_list_nt`)

After `loc_type` is known (`gw_loc_type_sto-loc_type`), **before** `lv_kl_take` logic, add:

```abap
*--- SS3: New Plant — KL already in BCKU
            IF lv_new_gatp_param = abap_true
            AND gw_loc_type_sto-loc_type = '1001'.
              CONTINUE.
            ENDIF.
```

Apply in **both** loops:

1. `LOOP AT lt_apo_so_list_mt ...`
2. `LOOP AT lt_apo_so_list_nt ...`

### 5.4 What stays

| Case | Behaviour |
|------|-----------|
| New + Plant `1001` | **No** SO KL add (BCKU only) |
| New + Depot `1002` | Existing additive (unchanged) |
| Old (any loc) | Existing additive (unchanged) |

---

## 6. Syntax / activate

| Step | Action |
|------|--------|
| 1 | Check → Activate `ZAPO_GATP_ALLOCATION_F005` |
| 2 | Resolve any unused-variable warnings for SS2 DATA |
| 3 | Ensure `gt_zapoparam` / `lv_new_gatp_param` still filled as today |
| 4 | Release only after T1–T6 |

---

## 7. Test plan (execute in order)

### 7.1 Setup

| Item | Value |
|------|--------|
| Param | `NT_MT_NEW` / `REPORT_NEW` active |
| Location | Plant `1001` |
| Material | Stock strategy with `ZAPO_SO_LIST-REQ_NT = 'KL'`, confirmed `BMENG > 0`, `EDATU = sy-datum` |
| Run | `ZGATPDB` with **Data Backup** |

### 7.2 Cases

| ID | Steps | Pass criteria |
|----|--------|----------------|
| **T1** | Backup with pure KL CVC | SE16 `ZAPO_PRIME_BCKU`: row for today, `NO_TOUCH` > 0 |
| **T2** | Next calendar day, same open demand (per RDD rule) | NT does **not** fully reappear; nets via MTD BCKU |
| **T3** | Change material to Allocation mid-month; re-run | Prior KL BCKU rows remain; new NT uses CHANGE F as today |
| **T4** | Allocation → KL mid-month | Continues writing BCKU; history intact |
| **T5** | MTD popup New Plant | Totals = SUM(BCKU); **no** double vs SO KL |
| **T6** | Switch to Old `NT_MT`; backup pure KL | **No** new KL-only BCKU policy (legacy exclude); MTD still SO-adds KL |

### 7.3 Verification SQL

```sql
-- Backup after T1
SELECT zdate, material, location, no_touch, manual_touch,
       inc_ord_quan, alloc_quan
  FROM zapo_prime_bcku
 WHERE material = '<MAT>'
   AND location = '<WERKS>'
 ORDER BY zdate;

-- SO strategy
SELECT edatu, req_nt, req_mt, bmeng, wmeng, vbeln
  FROM zapo_so_list
 WHERE matnr  = '<MAT>'
   AND werks  = '<WERKS>'
   AND delind = ' '
   AND abgru  = ' '
 ORDER BY edatu, req_nt;
```

---

## 8. Rollback

| If | Action |
|----|--------|
| Double count on MTD | Revert SS3 first (restore Plant SO add) **or** fix SS1/SS2 qty |
| Pure KL still missing from BCKU | Check SS1B order vs FAE SELECT; confirm `lt_kl_chk` not empty |
| Old param changed | Restore SS1A `sy-subrc <> 0` gate |
| Full rollback | Revert F005 to pre-TR version |

---

## 9. Checklist (sign-off)

- [ ] SS2 activated — netted KL qty in `merge_kl`
- [ ] SS1B append before today’s BCKU FAE SELECT
- [ ] SS1A Old-only exclude
- [ ] SS3 New Plant skip in **both** MT and NT KL loops
- [ ] T1–T6 passed
- [ ] Old param smoke test OK
- [ ] Depot New smoke test OK (legacy additive)
- [ ] Transport documented: “Include Stock Strategy KL in ZAPO_PRIME_BCKU (New param)”

---

## 10. Quick reference — files / comments

| Search string after impl | Meaning |
|--------------------------|---------|
| `SS1A` / `SS1B` | Backup include |
| `SS2` | merge_kl netting |
| `SS3` | MTD Plant no SO KL add |
| `FIX-KL-BAK Obs2` | Replaced by SS1 comments |

**Related analysis:** `ZGATPDB_StockStrategy_Include_Backup_MidMonth_Switch_Code_Correction.md`

---

*End of Implementation Guide*
