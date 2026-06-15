# ZAPO_NTBK_DMD_DSHP_EXCLUDE — Distribution Channel (ZVTWEG) Summation Correction

**Function Module:** `ZAPO_NTBK_DMD_DSHP_EXCLUDE`  
**Business rule:** `TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)`  
**Input table:** `ZAPO_NTBK_REF` — field `ZVTWEG` (comma-separated, e.g. `20,25`)  
**Output table:** `ZAPO_NTBK_DETAIL` — field `NTBK_QTY` must be **one summed value** per input row / month / material / location, not one row per distribution channel.

---

## 1. Problem (from your observation)

| Input (`ZAPO_NTBK_REF`) | Current output (`ZAPO_NTBK_DETAIL`) | Required output |
|-------------------------|-------------------------------------|-----------------|
| `ZVTWEG = 20,25` (single ref row) | Row with `ZVTWEG = 20`, `NTBK_QTY = X` **and** row with `ZVTWEG = 25`, `NTBK_QTY = Y` | **One row** with `ZVTWEG = 20,25` (or same as input) and `NTBK_QTY = X + Y` |

Screenshot reference: index row with `NTBK_LOGIC = TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)` and `ZVTWEG = 20,25` should produce a **single** detail line with combined quantity.

---

## 2. Root cause

### 2.1 Input is split correctly; output is not rolled up

At the start of each `it_input` iteration, comma-separated channels are split into `lt_vtweg` for **filtering** demand:

```abap
IF <lfs_input>-zvtweg IS NOT INITIAL.
  SPLIT <lfs_input>-zvtweg AT ',' INTO TABLE lt_vtweg.
ENDIF.
```

That part is correct: demand from `zapo_snp_dmdvsup` is limited to channels 20 and 25.

### 2.2 Separate output rows come from `lt_prod_dshpex_op` loop

After calling `ZAPO_NTBK_PROD_DSHP_EXCLUDE`, production netback quantities exist **per distribution channel** in `lt_prod_dshpex_op`. The FM then does this pattern **four times** (each demand path):

```abap
LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op WHERE ...
  lw_output_1          = lw_output.
  lw_output_1-zvtweg   = lw_prod_dshpex_op-zvtweg.   " ← single DC from prod FM
  lw_output_1-ntbk_qty = lw_prod_dshpex_op-ntbk_qty * <lfs_dmdvsup>-req_qty_pmt_matnr_out.
  APPEND lw_output_1 TO lt_output.                    " ← one row per DC
ENDLOOP.
```

Effects:

1. `zvtweg` on output is taken from **each** prod row (`20`, then `25`), not from **input** `20,25`.
2. Each demand line (`lt_dmdvsup` with `zvtweg = 20` or `25`) triggers another **APPEND**, so channels are never summed.

### 2.3 End-of-FM duplicate delete does not fix this

```abap
SORT et_output BY ntbk_mth zvtweg ntbk_qty.
DELETE ADJACENT DUPLICATES FROM et_output COMPARING ntbk_mth zvtweg ntbk_qty.
```

Rows for `20` and `25` have **different** `zvtweg`, so they are **not** removed. This logic only drops identical adjacent rows; it does not aggregate channels.

---

## 3. Solution design

**Principle:** Keep filtering demand by each channel in `lt_vtweg`, but **accumulate** `NTBK_QTY` into a working table **without** `ZVTWEG` in the aggregation key, then assign **input** `ZVTWEG` once when moving to `lt_output`.

| Step | Action |
|------|--------|
| 1 | Add `lt_output_agg` (`ZAPO_NTBK_DETAIL_T`) and `FORM sum_prod_dshp_to_agg`. |
| 2 | Replace every `LOOP lt_prod_dshpex_op … APPEND lw_output_1` block with `PERFORM sum_prod_dshp_to_agg`. |
| 3 | After each `LOOP AT lt_dmdvsup` block, `PERFORM flush_output_agg` → set `zvtweg = <lfs_input>-zvtweg`, append to `lt_output`. |
| 4 | Restrict prod loop with `AND zvtweg = <lfs_dmdvsup>-zvtweg` so only the matching prod row contributes per demand line. |
| 5 | Remove or relax `DELETE ADJACENT DUPLICATES … COMPARING ntbk_mth zvtweg ntbk_qty` (see § 6). |

`COLLECT` on `lt_output_agg` sums `ntbk_qty` for identical non-numeric key fields (month, material, location, PDH, session, etc.) while `zvtweg` is cleared during collect — so contributions from DC 20 and DC 25 add into **one** line.

---

## 4. Code to add in the function module

### 4.1 Additional data declarations

Add with other `DATA` declarations (after `lw_prod_dshpex_op`):

```abap
        lt_output_agg TYPE zapo_ntbk_detail_t,  " CD: ZVTWEG sum - mahesh pathak
        lw_output_agg TYPE zapo_ntbk_detail.    " CD: ZVTWEG sum - mahesh pathak
```

### 4.2 Local FORM routines (add at end of function, before `ENDFUNCTION`)

```abap
*&---------------------------------------------------------------------*
*& Form sum_prod_dshp_to_agg
*&  Sum prod netback qty * BOM ratio per demand line; COLLECT without ZVTWEG
*&---------------------------------------------------------------------*
  FORM sum_prod_dshp_to_agg
    USING
      iv_matnr       TYPE /sapapo/matid
      iv_locfr       TYPE /sapapo/locno
      iv_mth         TYPE zapo_ntbk_prd
      iv_pdh1        TYPE zapo_pdh1
      iv_pdh2        TYPE zapo_pdh2
      iv_dmd_zvtweg  TYPE vtweg
      iv_req_ratio   TYPE zapo_req_inp_qty
      iw_output      TYPE zapo_ntbk_detail
    CHANGING
      ct_output_agg  TYPE zapo_ntbk_detail_t
      it_prod_dshpex TYPE zapo_ntbk_detail_t.

    DATA: lw_agg  TYPE zapo_ntbk_detail,
          lw_prod TYPE zapo_ntbk_detail,
          lv_qty  TYPE zapo_ntbk_qty.

    CLEAR lv_qty.
    LOOP AT it_prod_dshpex INTO lw_prod
      WHERE ntbk_matnr = iv_matnr
        AND ntbk_locfr = iv_locfr
        AND pdh1       = iv_pdh1
        AND pdh2       = iv_pdh2
        AND zvtweg     = iv_dmd_zvtweg.
      IF iv_mth IS NOT INITIAL AND lw_prod-ntbk_mth <> iv_mth.
        CONTINUE.
      ENDIF.
      lv_qty = lv_qty + ( lw_prod-ntbk_qty * iv_req_ratio ).
    ENDLOOP.

    CHECK lv_qty IS NOT INITIAL OR iv_req_ratio IS NOT INITIAL.

    lw_agg = iw_output.
    CLEAR: lw_agg-zvtweg, lw_agg-ntbk_qty.
    lw_agg-ntbk_qty = lv_qty.
    COLLECT lw_agg INTO ct_output_agg.

  ENDFORM.

*&---------------------------------------------------------------------*
*& Form flush_output_agg
*&  Move aggregated lines to lt_output with combined ZVTWEG from input
*&---------------------------------------------------------------------*
  FORM flush_output_agg
    USING
      iv_zvtweg_combined TYPE zapo_ntbk_ref-zvtweg
    CHANGING
      ct_output_agg TYPE zapo_ntbk_detail_t
      ct_output     TYPE zapo_ntbk_detail_t.

    DATA: lw_agg TYPE zapo_ntbk_detail.

    LOOP AT ct_output_agg INTO lw_agg.
      lw_agg-zvtweg = iv_zvtweg_combined.
      APPEND lw_agg TO ct_output.
    ENDLOOP.
    CLEAR ct_output_agg.

  ENDFORM.
```

> **Note:** Adjust `iv_matnr` type if your `zapo_ntbk_detail-ntbk_matnr` uses `/sapapo/matnr` instead of `matid` — keep types consistent with `lt_prod_dshpex_op` structure.

---

## 5. Replace existing prod loops (4 locations)

### 5.1 Pattern — BEFORE (remove everywhere)

```abap
            CLEAR lw_prod_dshpex_op.
            LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op WHERE     ntbk_matnr  = <lfs_dmdvsup>-matnr
                                                             AND       ntbk_locfr  = <lfs_dmdvsup>-mfg_plant
                                                             AND       ntbk_mth    = <lfs_dmdvsup>-date_mythr
                                                             AND       pdh1        = <lfs_input>-pdh1
                                                             AND       pdh2        = <lfs_input>-pdh2.
              lw_output_1          = lw_output.
              lw_output_1-zvtweg   = lw_prod_dshpex_op-zvtweg.
              lw_output_1-ntbk_qty = lw_prod_dshpex_op-ntbk_qty * <lfs_dmdvsup>-req_qty_pmt_matnr_out.
              APPEND lw_output_1 TO lt_output.
              CLEAR: lw_output_1,lw_prod_dshpex_op.
            ENDLOOP.
```

### 5.2 Pattern — AFTER (use in all 4 places)

```abap
            PERFORM sum_prod_dshp_to_agg USING
              <lfs_dmdvsup>-matnr
              <lfs_dmdvsup>-mfg_plant
              <lfs_dmdvsup>-date_mythr
              <lfs_input>-pdh1
              <lfs_input>-pdh2
              <lfs_dmdvsup>-zvtweg
              <lfs_dmdvsup>-req_qty_pmt_matnr_out
              lw_output
            CHANGING
              lt_output_agg
              lt_prod_dshpex_op.
```

For branches **without** `ntbk_mth` in the original `WHERE`, pass **initial** `iv_mth`:

```abap
            PERFORM sum_prod_dshp_to_agg USING
              ...
              space                 " iv_mth - no month filter on prod table
              ...
```

### 5.3 Flush after each `LOOP AT lt_dmdvsup` block

Immediately **after** `ENDLOOP.` of `lt_dmdvsup` (and before the next branch / `CALL ZAPO_NTBK_COMB_PREPARE`), add:

```abap
          PERFORM flush_output_agg USING <lfs_input>-zvtweg
            CHANGING lt_output_agg lt_output.
```

**Locations to add flush:**

| # | After `ENDLOOP` of | Context |
|---|-------------------|---------|
| 1 | Parallel cursor loop on `lt_dmdvsup` (RM-only / loc list path) | Before `ENDIF` closing `READ TABLE lt_dmdvsup` |
| 2 | `WHERE matnr = <lfs_out>-ntbk_matnr AND locno = <lfs_out>-ntbk_locto` | End of `IF matnr AND locto` branch |
| 3 | `WHERE locno = <lfs_out>-ntbk_locto` | End of `ELSEIF locto only` branch |
| 4 | `WHERE matnr = <lfs_out>-ntbk_matnr` | End of `ELSEIF matnr only` branch |

At the start of processing each `<lfs_input>` row (after `CLEAR: lw_output, lt_div, lt_vtweg, lt_loc`), add:

```abap
      CLEAR lt_output_agg.
```

---

## 6. Adjust final sort / duplicate removal

**Remove** (or comment) — it can delete unrelated rows that share month, channel text, and quantity:

```abap
      SORT et_output BY ntbk_mth zvtweg ntbk_qty.
      DELETE ADJACENT DUPLICATES FROM et_output COMPARING ntbk_mth zvtweg ntbk_qty.
```

**Replace with** sort only (recommended):

```abap
      SORT et_output BY ntbk_index ntbk_mth zvtweg ntbk_matnr ntbk_locfr pdh1 pdh2.
```

If duplicate removal is still required for another defect, use a **full business key**, for example:

```abap
      DELETE ADJACENT DUPLICATES FROM et_output
        COMPARING ntbk_index ntbk_mth ntbk_matnr ntbk_locfr
                  zvtweg pdh1 pdh2 ntbk_rmmatnr sessionname.
```

---

## 7. Expected behaviour after fix

**Example input (`ZAPO_NTBK_REF`):**

| ZVTWEG | NTBK_LOGIC | NTBK_RMMATNR | … |
|--------|------------|--------------|---|
| 20,25 | TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP) | R167GER01 | … |

**Intermediate (unchanged):**

- `ZAPO_NTBK_PROD_DSHP_EXCLUDE` still returns one row per DC in `lt_prod_dshpex_op` (`zvtweg = 20`, `zvtweg = 25`).
- `lt_dmdvsup` still filtered: only lines where `zvtweg` is 20 or 25.

**Output (`ZAPO_NTBK_DETAIL`):**

| ZVTWEG | NTBK_QTY |
|--------|----------|
| 20,25 | (Qty_DC20 × BOM_ratio_DC20) + (Qty_DC25 × BOM_ratio_DC25) |

One row per **month × FG × location × PDH × index**, not per individual channel.

---

## 8. Test checklist

1. Input `ZVTWEG = 20,25` → exactly **one** output line per month/material/location (for that index), `ZVTWEG` matches input.
2. `NTBK_QTY` = manual sum of previous per-channel rows (regression compare before/after on same session).
3. Input `ZVTWEG = 20` only → single row, quantity unchanged vs old DC 20 row.
4. Multiple months in bucket → still **multiple rows** (one per month), not collapsed.
5. Different materials / locations → still **separate** rows (COLLECT key must not merge unrelated FG).

---

## 9. Paste-ready replacement block (all 4 prod loops)

Use this **single** replacement everywhere the old `LOOP AT lt_prod_dshpex_op … APPEND lw_output_1` appears.

**Variant A** — paths that filter prod by month (`ntbk_mth` in original WHERE):

```abap
            PERFORM sum_prod_dshp_to_agg USING
              <lfs_dmdvsup>-matnr
              <lfs_dmdvsup>-mfg_plant
              <lfs_dmdvsup>-date_mythr
              <lfs_input>-pdh1
              <lfs_input>-pdh2
              <lfs_dmdvsup>-zvtweg
              <lfs_dmdvsup>-req_qty_pmt_matnr_out
              lw_output
            CHANGING
              lt_output_agg
              lt_prod_dshpex_op.
```

**Variant B** — `matnr + locto` branch (no month on prod WHERE in original):

```abap
            PERFORM sum_prod_dshp_to_agg USING
              <lfs_dmdvsup>-matnr
              <lfs_dmdvsup>-mfg_plant
              space
              <lfs_input>-pdh1
              <lfs_input>-pdh2
              <lfs_dmdvsup>-zvtweg
              <lfs_dmdvsup>-req_qty_pmt_matnr_out
              lw_output
            CHANGING
              lt_output_agg
              lt_prod_dshpex_op.
```

**After each `ENDLOOP` on `lt_dmdvsup`:**

```abap
          PERFORM flush_output_agg USING <lfs_input>-zvtweg
            CHANGING lt_output_agg lt_output.
```

---

## 10. FORM routines (paste before `ENDFUNCTION`)

```abap
  FORM sum_prod_dshp_to_agg
    USING
      iv_matnr       TYPE /sapapo/matid
      iv_locfr       TYPE /sapapo/locno
      iv_mth         TYPE zapo_ntbk_prd
      iv_pdh1        TYPE zapo_pdh1
      iv_pdh2        TYPE zapo_pdh2
      iv_dmd_zvtweg  TYPE vtweg
      iv_req_ratio   TYPE zapo_req_inp_qty
      iw_output      TYPE zapo_ntbk_detail
    CHANGING
      ct_output_agg  TYPE zapo_ntbk_detail_t
      it_prod_dshpex TYPE zapo_ntbk_detail_t.

    DATA: lw_agg  TYPE zapo_ntbk_detail,
          lw_prod TYPE zapo_ntbk_detail,
          lv_qty  TYPE zapo_ntbk_qty.

    CLEAR lv_qty.
    LOOP AT it_prod_dshpex INTO lw_prod
      WHERE ntbk_matnr = iv_matnr
        AND ntbk_locfr = iv_locfr
        AND pdh1       = iv_pdh1
        AND pdh2       = iv_pdh2
        AND zvtweg     = iv_dmd_zvtweg.
      IF iv_mth IS NOT INITIAL AND lw_prod-ntbk_mth <> iv_mth.
        CONTINUE.
      ENDIF.
      lv_qty = lv_qty + ( lw_prod-ntbk_qty * iv_req_ratio ).
    ENDLOOP.

    CHECK lv_qty IS NOT INITIAL OR iv_req_ratio IS NOT INITIAL.

    lw_agg = iw_output.
    CLEAR: lw_agg-zvtweg, lw_agg-ntbk_qty.
    lw_agg-ntbk_qty = lv_qty.
    COLLECT lw_agg INTO ct_output_agg.

  ENDFORM.

  FORM flush_output_agg
    USING
      iv_zvtweg_combined TYPE zapo_ntbk_ref-zvtweg
    CHANGING
      ct_output_agg TYPE zapo_ntbk_detail_t
      ct_output     TYPE zapo_ntbk_detail_t.

    DATA: lw_agg TYPE zapo_ntbk_detail.

    LOOP AT ct_output_agg INTO lw_agg.
      lw_agg-zvtweg = iv_zvtweg_combined.
      APPEND lw_agg TO ct_output.
    ENDLOOP.
    CLEAR ct_output_agg.

  ENDFORM.
```

> Apply all other logic from your current production include unchanged (BOM, `ZAPO_NTBK_COMB_PREPARE`, `ZAPO_NTBK_PROD_DSHP_EXCLUDE`, index assignment). Only the prod-loop **APPEND** pattern and end **DELETE ADJACENT** need to change as in §§ 4–6.

---

## 11. Summary

| Item | Detail |
|------|--------|
| **Cause** | `lw_output_1-zvtweg = lw_prod_dshpex_op-zvtweg` + `APPEND` per prod/demand channel |
| **Fix** | `COLLECT` into `lt_output_agg` without `zvtweg`, then set `zvtweg` from `<lfs_input>-zvtweg` once |
| **Scope** | All four `lt_prod_dshpex_op` loops + flush after each `lt_dmdvsup` loop |
| **Transport** | FM `ZAPO_NTBK_DMD_DSHP_EXCLUDE` in `SAPLZAPO_NTBK` |

**Document:** `ZAPO_NTBK_DMD_DSHP_EXCLUDE_ZVTWEG_Sum_Correction.md`  
**Path:** `Netback Calculation\`  
**Author:** mahesh pathak / omkar more — CD 8085121 follow-up (ZVTWEG combined output)
  TYPES: BEGIN OF lty_locdet,
         matnr TYPE /sapapo/matnr,
         locno TYPE /sapapo/locno,
         mfg_plant TYPE /sapapo/locno,
         END OF lty_locdet,

         BEGIN OF lty_dmdvsup,
            sessionid TYPE  /sapapo/snpsession,
            matnr TYPE 	/sapapo/matid,
            locno	TYPE /sapapo/locno,
            buckt TYPE  zapo_bucket,
            sessionname TYPE   zapo_sessionname,
            planner_snp	TYPE /sapapo/planner_snp,
            meins	TYPE /sapapo/meins,
            src_locno TYPE  zapo_src_locno_1,
            sup_cust_1 TYPE	zapo_sp1_qty,
            src_locno_2 TYPE  zapo_src_locno_2,
            zvtweg TYPE	vtweg,
            matnr_sub TYPE  zsubmat,
            mfg_plant TYPE /sapapo/locno,
            req_qty_pmt_matnr_out TYPE zapo_req_inp_qty,
            date_mythr TYPE zapo_ntbk_prd,
          END OF  lty_dmdvsup,

          BEGIN OF lty_log_bom,
            sessionname TYPE  zapo_sessionname,
            fg_matnr TYPE /sapapo/matnr,
            fg_locno TYPE  /sapapo/locno,
            bucket TYPE /sapapo/snpbucke,
            bom_level	TYPE zapo_bom_level,
            PARENT_NODE_ID TYPE	ZAPO_PARENT_ID,
            NODE_ID	       TYPE ZAPO_NODE_ID,
            BOM_TYPE       TYPE	ZAPO_BOM_TYPE,
            PRICELVL       TYPE ZPRICELVL,
            matnr_out	     TYPE zapo_matnr_out,
            pds_out_qty	TYPE zapo_pds_out_qty,
            req_prnt_qty TYPE	zapo_req_inp_qty,
            fg_div TYPE  zfg_div,
          END OF lty_log_bom,

          BEGIN OF lty_zapoparam,
            param1 TYPE zparam1,
            param2 TYPE zparam2,
            param3 TYPE zparam3,
            param4 TYPE zparam4,
            param5 TYPE zparam5,
            value1 TYPE zparamvalue1,
            value2 TYPE zparamvalue2,
            value3 TYPE zparamvalue3,
            value4 TYPE zparamvalue4,
            value5 TYPE zparamvalue5,
            active_flag TYPE zactive_flag,
          END OF lty_zapoparam,

          BEGIN OF lty_log_bom_1,
            sessionname TYPE  zapo_sessionname,
            fg_matnr TYPE /sapapo/matnr,
            fg_locno TYPE  /sapapo/locno,
            bucket TYPE zapo_bucket,
            bom_level	TYPE zapo_bom_level,
            matnr_out	TYPE zapo_matnr_out,
            pds_out_qty	TYPE zapo_pds_out_qty,
            req_prnt_qty TYPE	zapo_req_inp_qty,
            req_qty_pmt_matnr_out TYPE zapo_req_inp_qty,
            count TYPE i ,
            count1 TYPE i ,
          END OF lty_log_bom_1.

  DATA: lt_locdet TYPE TABLE OF lty_locdet,
        lt_out TYPE TABLE OF zapo_ntbk_ref,
        lt_comb TYPE TABLE OF zapo_ntbk_ref,
        lr_mat TYPE RANGE OF /sapapo/matnr,
        lt_dmdvsup TYPE TABLE OF lty_dmdvsup,
        lt_dmdvsup_tmp TYPE TABLE OF lty_dmdvsup,
        lt_dmdvsup_tot TYPE TABLE OF lty_dmdvsup,
        lw_dmdvsup_tmp TYPE lty_dmdvsup,
        lw_dmdvsup_tot TYPE lty_dmdvsup,
        lr_div  TYPE RANGE OF spart,
        lr_vtweg TYPE RANGE OF vtweg,
        lt_div  TYPE TABLE OF spart,
        lt_vtweg TYPE TABLE OF vtweg,
        lt_loc TYPE TABLE OF zapo_ntbk_locno,
        lr_locto TYPE RANGE OF /sapapo/locno,
        lr_locfr TYPE RANGE OF /sapapo/locno,
        lt_plandrv TYPE zapo_plndrv_tt,
        lt_log_bom TYPE TABLE OF lty_log_bom,
        lt_zapoparam TYPE STANDARD TABLE OF lty_zapoparam,
        lw_zapoparam TYPE lty_zapoparam,
        lw_divfg TYPE char200,
        lw_outmat TYPE char200,
        lt_log_bom_level TYPE TABLE OF lty_log_bom,
        lt_log_bom_bucket TYPE TABLE OF lty_log_bom,
        lw_log_bom TYPE lty_log_bom_1,
        lt_log_bom_1 TYPE TABLE OF lty_log_bom_1,
        lt_log_bom_2 TYPE TABLE OF lty_log_bom_1,
        lw_log_bom_1 TYPE lty_log_bom_1,
        lt_log_bom_final TYPE TABLE OF lty_log_bom_1,
        lt_prod_out TYPE TABLE OF zapo_ntbk_prod,
        lw_output TYPE zapo_ntbk_detail,
        lw_output_1 TYPE zapo_ntbk_detail,
        lw_bucket TYPE /sapapo/snpoptbucket,
        lt_prod_input TYPE TABLE OF zapo_ntbk_ref,
        lt_output TYPE TABLE OF zapo_ntbk_detail,
        lv_ntbk_index TYPE zapo_ntbk_index,
        lv_tabix      TYPE sy-tabix,
        lv_tabix_2    TYPE sy-tabix,
        lt_prod_dshpex_op TYPE zapo_ntbk_detail_t,
        lw_prod_dshpex_op TYPE zapo_ntbk_detail,
        lt_output_agg TYPE zapo_ntbk_detail_t,    ">>> CD: ZVTWEG SUM
        lw_output_agg TYPE zapo_ntbk_detail.      ">>> CD: ZVTWEG SUM

  FIELD-SYMBOLS: <lfs_input> TYPE zapo_ntbk_ref,
                 <lfs_out>   TYPE zapo_ntbk_ref,
                 <lfs_mat> LIKE LINE OF lr_mat,
                 <lfs_div> LIKE LINE OF lr_div,
                 <lfs_vtweg> LIKE LINE OF lr_vtweg,
                 <lfs_div_t> TYPE spart,
                 <lfs_vtweg_t> TYPE vtweg,
                 <lfs_loc> LIKE LINE OF lr_locfr,
                 <lfs_dmdvsup> TYPE lty_dmdvsup,
                 <lfs_log_bom> TYPE lty_log_bom,
                 <lfs_log_bom_1> TYPE lty_log_bom_1,
                 <lfs_plndrv> TYPE zapo_plndrv,
                 <lfs_ntbk_prod> TYPE zapo_ntbk_prod,
                 <lfs_dmdsup_tot> TYPE lty_dmdvsup,
                 <lfs_output> TYPE zapo_ntbk_detail.

  IF it_input IS NOT INITIAL.
    LOOP AT it_input ASSIGNING <lfs_input>.
      APPEND INITIAL LINE TO lr_mat ASSIGNING <lfs_mat>.
      <lfs_mat>-sign = 'I'.
      <lfs_mat>-option = 'EQ'.
      <lfs_mat>-low = <lfs_input>-ntbk_rmmatnr.
      TRANSLATE <lfs_mat>-low TO UPPER CASE.
    ENDLOOP.

    SORT lr_mat BY low.
    DELETE ADJACENT DUPLICATES FROM lr_mat COMPARING low.

    SELECT sessionid matnr locno buckt sessionname planner_snp meins
           src_locno sup_cust_1 src_locno_2 zvtweg matnr_sub
      FROM zapo_snp_dmdvsup
      INTO TABLE lt_dmdvsup
      WHERE sessionid = is_session-sessionid.
    IF sy-subrc = 0.

      CALL FUNCTION 'ZAPO_NTBK_PLNDRV'
        EXPORTING is_session = is_session
        IMPORTING et_plandrv = lt_plandrv.

      SORT lt_plandrv BY sessionid bucke.
      DELETE lt_dmdvsup WHERE sup_cust_1 IS INITIAL.

      LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup>.
        CLEAR lw_dmdvsup_tmp.
        MOVE-CORRESPONDING <lfs_dmdvsup> TO lw_dmdvsup_tmp.
        CLEAR: lw_dmdvsup_tmp-src_locno_2, lw_dmdvsup_tmp-src_locno, lw_bucket.
        IF <lfs_dmdvsup>-src_locno_2 IS NOT INITIAL.
          lw_dmdvsup_tmp-mfg_plant = <lfs_dmdvsup>-src_locno_2.
        ELSE.
          lw_dmdvsup_tmp-mfg_plant = <lfs_dmdvsup>-src_locno.
        ENDIF.
        CONDENSE <lfs_dmdvsup>-buckt.
        lw_bucket = <lfs_dmdvsup>-buckt.
        READ TABLE lt_plandrv ASSIGNING <lfs_plndrv>
          WITH KEY sessionid = <lfs_dmdvsup>-sessionid bucke = lw_bucket BINARY SEARCH.
        IF sy-subrc = 0 AND <lfs_plndrv> IS ASSIGNED.
          lw_dmdvsup_tmp-date_mythr = <lfs_plndrv>-date_mthyr.
        ENDIF.
        COLLECT lw_dmdvsup_tmp INTO lt_dmdvsup_tmp.
        IF <lfs_dmdvsup>-sessionid IS NOT INITIAL.
          CLEAR lw_dmdvsup_tot.
          lw_dmdvsup_tot-sessionid = <lfs_dmdvsup>-sessionid.
          lw_dmdvsup_tot-sessionname = <lfs_dmdvsup>-sessionname.
          lw_dmdvsup_tot-matnr = <lfs_dmdvsup>-matnr.
          lw_dmdvsup_tot-buckt = <lfs_dmdvsup>-buckt.
          lw_dmdvsup_tot-planner_snp = <lfs_dmdvsup>-planner_snp.
          lw_dmdvsup_tot-sup_cust_1 = <lfs_dmdvsup>-sup_cust_1.
          lw_dmdvsup_tot-mfg_plant = lw_dmdvsup_tmp-mfg_plant.
          lw_dmdvsup_tot-date_mythr = lw_dmdvsup_tmp-date_mythr.
          COLLECT lw_dmdvsup_tot INTO lt_dmdvsup_tot.
        ENDIF.
        APPEND INITIAL LINE TO lt_prod_input ASSIGNING <lfs_input>.
        <lfs_input>-ntbk_locfr = lw_dmdvsup_tmp-mfg_plant.
        <lfs_input>-ntbk_matnr = <lfs_dmdvsup>-matnr.
        <lfs_input>-motiondiv = <lfs_dmdvsup>-planner_snp.
      ENDLOOP.

      SORT lt_dmdvsup_tot BY sessionid matnr buckt planner_snp mfg_plant.
      CLEAR lt_dmdvsup.
      lt_dmdvsup = lt_dmdvsup_tmp.
      SORT lt_dmdvsup_tmp BY sessionname matnr_sub mfg_plant.
      DELETE ADJACENT DUPLICATES FROM lt_dmdvsup_tmp COMPARING sessionname matnr_sub mfg_plant.

      IF lt_dmdvsup_tmp IS NOT INITIAL.
        SELECT sessionname fg_matnr fg_locno bucket bom_level parent_node_id node_id
               bom_type pricelvl matnr_out pds_out_qty req_prnt_qty fg_div
          FROM zapo_opt_log_bom
          INTO TABLE lt_log_bom
          FOR ALL ENTRIES IN lt_dmdvsup_tmp
          WHERE sessionname = lt_dmdvsup_tmp-sessionname
            AND fg_matnr = lt_dmdvsup_tmp-matnr_sub
            AND fg_locno = lt_dmdvsup_tmp-mfg_plant.
        IF sy-subrc = 0.
          SELECT param1 param2 param3 param4 param5 value1 value2 value3 value4 value5 active_flag
            FROM zapoparam INTO TABLE lt_zapoparam
            WHERE param1 EQ 'SNP' AND param2 EQ 'ZAPO_NTBK' AND param3 EQ 'CONVUOM' AND active_flag EQ 'X'.
          IF sy-subrc EQ 0.
            SORT lt_zapoparam.
          ENDIF.
          LOOP AT lt_log_bom ASSIGNING <lfs_log_bom>.
            LOOP AT lt_zapoparam INTO lw_zapoparam.
              CONCATENATE lw_zapoparam-value1 lw_zapoparam-value2 lw_zapoparam-value3 INTO lw_divfg SEPARATED BY ','.
              CONCATENATE lw_zapoparam-value4 lw_zapoparam-value5 INTO lw_outmat SEPARATED BY ','.
              CONDENSE: lw_divfg, lw_outmat.
              IF lw_divfg CS <lfs_log_bom>-fg_div AND lw_outmat CS <lfs_log_bom>-matnr_out.
                <lfs_log_bom>-pds_out_qty = <lfs_log_bom>-pds_out_qty / 1000.
              ENDIF.
              CLEAR: lw_divfg, lw_outmat, lw_zapoparam.
            ENDLOOP.
          ENDLOOP.
          SORT lt_log_bom BY sessionname fg_matnr fg_locno matnr_out bucket bom_level.
          CLEAR: lt_log_bom_bucket, lt_log_bom_level, lt_dmdvsup_tmp.
          LOOP AT lt_log_bom ASSIGNING <lfs_log_bom>.
            CLEAR lw_log_bom.
            IF <lfs_log_bom>-matnr_out NOT IN lr_mat.
              CONTINUE.
            ENDIF.
            MOVE-CORRESPONDING <lfs_log_bom> TO lw_log_bom.
            lw_log_bom-bucket = <lfs_log_bom>-bucket.
            lw_log_bom-count = COND #( WHEN lw_log_bom-pds_out_qty IS NOT INITIAL THEN 1 ELSE 0 ).
            lw_log_bom-count1 = COND #( WHEN lw_log_bom-req_prnt_qty IS NOT INITIAL THEN 1 ELSE 0 ).
            COLLECT lw_log_bom INTO lt_log_bom_1.
          ENDLOOP.
          LOOP AT lt_log_bom_1 ASSIGNING <lfs_log_bom_1>.
            IF <lfs_log_bom_1>-count > 0.
              <lfs_log_bom_1>-pds_out_qty = <lfs_log_bom_1>-pds_out_qty / <lfs_log_bom_1>-count.
            ENDIF.
            IF <lfs_log_bom_1>-count1 > 0.
              <lfs_log_bom_1>-req_prnt_qty = <lfs_log_bom_1>-req_prnt_qty / <lfs_log_bom_1>-count1.
            ENDIF.
            MOVE-CORRESPONDING <lfs_log_bom_1> TO lw_log_bom.
            CONDENSE lw_log_bom-bucket.
            COLLECT lw_log_bom INTO lt_log_bom_2.
          ENDLOOP.
        ENDIF.
      ENDIF.
      SORT lt_log_bom_2 BY sessionname fg_matnr fg_locno bucket matnr_out.

      REFRESH lt_prod_dshpex_op.
      CALL FUNCTION 'ZAPO_NTBK_PROD_DSHP_EXCLUDE'
        EXPORTING
          is_session   = is_session
          it_input     = it_input
          it_matkey    = it_matkey
          it_loc       = it_loc
          it_drv_matnr = it_drv_matnr
        IMPORTING
          et_output    = lt_prod_dshpex_op.
      SORT lt_prod_dshpex_op BY ntbk_matnr ntbk_locfr pdh1 pdh2.
    ENDIF.

    SORT lt_dmdvsup BY sessionid sessionname.

    LOOP AT it_input ASSIGNING <lfs_input>.
      CLEAR: lw_output, lt_div, lt_vtweg, lt_loc, lt_output_agg.  ">>> CD: ZVTWEG SUM
      TRANSLATE <lfs_input>-ntbk_rmmatnr TO UPPER CASE.
      IF <lfs_input>-motiondiv IS NOT INITIAL.
        SPLIT <lfs_input>-div AT ',' INTO TABLE lt_div.
      ENDIF.
      IF <lfs_input>-zvtweg IS NOT INITIAL.
        SPLIT <lfs_input>-zvtweg AT ',' INTO TABLE lt_vtweg.
      ENDIF.

      IF <lfs_input>-ntbk_matnr IS INITIAL AND <lfs_input>-ntbk_locto IS INITIAL
        AND <lfs_input>-motionmotion IS INITIAL AND <lfs_input>-pdh1 IS INITIAL AND <lfs_input>-pdh2 IS INITIAL
        AND <lfs_input>-ntbk_rmmatnr IS NOT INITIAL.

        SPLIT <lfs_out>-ntbk_locfr AT ',' INTO TABLE lt_loc.

        READ TABLE lt_dmdvsup TRANSPORTING NO FIELDS
          WITH KEY sessionid = is_session-sessionid sessionname = is_session-sessionname BINARY SEARCH.
        IF sy-subrc = 0.
          lv_tabix = sy-tabix.
          LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup> FROM lv_tabix.
            IF <lfs_dmdvsup>-sessionid NE is_session-sessionid
            AND <lfs_dmdvsup>-sessionname NE is_session-sessionname.
              EXIT.
            ENDIF.
            CLEAR lw_output.
            MOVE-CORRESPONDING <lfs_input> TO lw_output.
            CLEAR lw_output-ntbk_index.
            IF <lfs_dmdvsup>-sup_cust_1 IS INITIAL.
              CONTINUE.
            ENDIF.
            IF lt_div IS NOT INITIAL.
              READ TABLE lt_div WITH KEY table_line = <lfs_dmdvsup>-planner_snp TRANSPORTING NO FIELDS.
              IF sy-subrc NE 0. CONTINUE. ENDIF.
            ENDIF.
            IF lt_vtweg IS NOT INITIAL.
              READ TABLE lt_vtweg WITH KEY table_line = <lfs_dmdvsup>-zvtweg TRANSPORTING NO FIELDS.
              IF sy-subrc NE 0. CONTINUE. ENDIF.
            ENDIF.
            lw_output-ntbk_mth = <lfs_dmdvsup>-date_mythr.
            lw_output-sessionname = is_session-sessionname.
            READ TABLE lt_loc WITH KEY table_line = <lfs_out>-ntbk_locfr TRANSPORTING NO FIELDS.
            IF sy-subrc NE 0. CONTINUE. ENDIF.

            READ TABLE lt_log_bom_2 TRANSPORTING NO FIELDS
              WITH KEY sessionname = <lfs_dmdvsup>-sessionname
                       fg_matnr = <lfs_dmdvsup>-matnr_sub
                       fg_locno = <lfs_dmdvsup>-mfg_plant
                       bucket = <lfs_dmdvsup>-buckt
                       matnr_out = <lfs_input>-ntbk_rmmatnr BINARY SEARCH.
            IF sy-subrc = 0.
              lv_tabix_2 = sy-tabix.
            ENDIF.
            LOOP AT lt_log_bom_2 ASSIGNING <lfs_log_bom_1> FROM lv_tabix_2.
              IF <lfs_log_bom_1>-sessionname NE <lfs_dmdvsup>-sessionname
              AND <lfs_log_bom_1>-fg_matnr NE <lfs_dmdvsup>-matnr_sub
              AND <lfs_log_bom_1>-fg_locno NE <lfs_dmdvsup>-mfg_plant
              AND <lfs_log_bom_1>-bucket NE <lfs_dmdvsup>-buckt
              AND <lfs_log_bom_1>-matnr_out NE <lfs_input>-ntbk_rmmatnr.
                EXIT.
              ENDIF.
              IF <lfs_log_bom_1>-pds_out_qty NE 0.
                <lfs_dmdvsup>-req_qty_pmt_matnr_out = <lfs_log_bom_1>-req_prnt_qty / <lfs_log_bom_1>-pds_out_qty.
              ENDIF.
            ENDLOOP.

            ">>> CD: ZVTWEG SUM - was LOOP APPEND per prod row
            PERFORM sum_prod_dshp_to_agg USING
              <lfs_dmdvsup>-matnr <lfs_dmdvsup>-mfg_plant <lfs_dmdvsup>-date_mythr
              <lfs_input>-pdh1 <lfs_input>-pdh2 <lfs_dmdvsup>-zvtweg
              <lfs_dmdvsup>-req_qty_pmt_matnr_out lw_output
            CHANGING lt_output_agg lt_prod_dshpex_op.
          ENDLOOP.
          PERFORM flush_output_agg USING <lfs_input>-zvtweg
            CHANGING lt_output_agg lt_output.                     ">>> CD: ZVTWEG SUM
        ENDIF.
      ENDIF.

      CALL FUNCTION 'ZAPO_NTBK_COMB_PREPARE'
        EXPORTING is_input = <lfs_input> it_drv_matnr = it_drv_matnr
        IMPORTING et_out = lt_out.

      LOOP AT lt_out ASSIGNING <lfs_out>.
        CLEAR lt_output_agg.                                      ">>> CD: ZVTWEG SUM

        IF <lfs_out>-ntbk_matnr IS NOT INITIAL AND <lfs_out>-ntbk_locto IS NOT INITIAL.
          LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup>
            WHERE matnr = <lfs_out>-ntbk_matnr AND locno = <lfs_out>-ntbk_locto.
            CLEAR lw_output.
            MOVE-CORRESPONDING <lfs_input> TO lw_output.
            CLEAR lw_output-ntbk_index.
            IF <lfs_dmdvsup>-sup_cust_1 IS INITIAL. CONTINUE. ENDIF.
            IF lt_div IS NOT INITIAL.
              READ TABLE lt_div WITH KEY table_line = <lfs_dmdvsup>-planner_snp TRANSPORTING NO FIELDS.
              IF sy-subrc NE 0. CONTINUE. ENDIF.
            ENDIF.
            IF lt_vtweg IS NOT INITIAL.
              READ TABLE lt_vtweg WITH KEY table_line = <lfs_dmdvsup>-zvtweg TRANSPORTING NO FIELDS.
              IF sy-subrc NE 0. CONTINUE. ENDIF.
            ENDIF.
            IF <lfs_dmdvsup>-mfg_plant NE <lfs_out>-ntbk_locfr AND <lfs_out>-ntbk_locfr IS NOT INITIAL.
              CONTINUE.
            ENDIF.
            lw_output-ntbk_mth = <lfs_dmdvsup>-date_mythr.
            lw_output-sessionname = is_session-sessionname.
            LOOP AT lt_log_bom_2 ASSIGNING <lfs_log_bom_1>
              WHERE sessionname = <lfs_dmdvsup>-sessionname
                AND fg_matnr = <lfs_dmdvsup>-matnr_sub
                AND fg_locno = <lfs_dmdvsup>-mfg_plant
                AND bucket = <lfs_dmdvsup>-buckt
                AND matnr_out = <lfs_input>-ntbk_rmmatnr.
              IF <lfs_log_bom_1>-pds_out_qty NE 0.
                CONDENSE <lfs_log_bom_1>-bucket NO-GAPS.
                <lfs_dmdvsup>-req_qty_pmt_matnr_out = <lfs_log_bom_1>-req_prnt_qty / <lfs_log_bom_1>-pds_out_qty.
              ENDIF.
            ENDLOOP.
            PERFORM sum_prod_dshp_to_agg USING
              <lfs_dmdvsup>-matnr <lfs_dmdvsup>-mfg_plant space
              <lfs_input>-pdh1 <lfs_input>-pdh2 <lfs_dmdvsup>-zvtweg
              <lfs_dmdvsup>-req_qty_pmt_matnr_out lw_output
            CHANGING lt_output_agg lt_prod_dshpex_op.
          ENDLOOP.
          PERFORM flush_output_agg USING <lfs_input>-zvtweg CHANGING lt_output_agg lt_output.

        ELSEIF <lfs_out>-ntbk_matnr IS INITIAL AND <lfs_out>-ntbk_locto IS NOT INITIAL.
          LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup> WHERE locno = <lfs_out>-ntbk_locto.
            " ... same filters and BOM loop as above ...
            PERFORM sum_prod_dshp_to_agg USING
              <lfs_dmdvsup>-matnr <lfs_dmdvsup>-mfg_plant <lfs_dmdvsup>-date_mythr
              <lfs_input>-pdh1 <lfs_input>-pdh2 <lfs_dmdvsup>-zvtweg
              <lfs_dmdvsup>-req_qty_pmt_matnr_out lw_output
            CHANGING lt_output_agg lt_prod_dshpex_op.
          ENDLOOP.
          PERFORM flush_output_agg USING <lfs_input>-zvtweg CHANGING lt_output_agg lt_output.

        ELSEIF <lfs_out>-ntbk_matnr IS NOT INITIAL AND <lfs_out>-ntbk_locto IS INITIAL.
          LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup> WHERE matnr = <lfs_out>-ntbk_matnr.
            " ... same filters and BOM loop as above ...
            PERFORM sum_prod_dshp_to_agg USING
              <lfs_dmdvsup>-matnr <lfs_dmdvsup>-mfg_plant <lfs_dmdvsup>-date_mythr
              <lfs_input>-pdh1 <lfs_input>-pdh2 <lfs_dmdvsup>-zvtweg
              <lfs_dmdvsup>-req_qty_pmt_matnr_out lw_output
            CHANGING lt_output_agg lt_prod_dshpex_op.
          ENDLOOP.
          PERFORM flush_output_agg USING <lfs_input>-zvtweg CHANGING lt_output_agg lt_output.
        ENDIF.
      ENDLOOP.

      IF lt_output IS NOT INITIAL.
        lv_ntbk_index = <lfs_input>-ntbk_index.
        LOOP AT lt_output ASSIGNING <lfs_output>.
          <lfs_output>-ntbk_index = lv_ntbk_index.
          lv_ntbk_index = lv_ntbk_index + 1.
          APPEND <lfs_output> TO et_output.
        ENDLOOP.
        CLEAR lt_output.
      ENDIF.
    ENDLOOP.

    SORT et_output BY ntbk_index ntbk_mth zvtweg ntbk_matnr ntbk_locfr pdh1 pdh2.  ">>> CD: ZVTWEG SUM
  ENDIF.

  FORM sum_prod_dshp_to_agg
    USING iv_matnr TYPE /sapapo/matid iv_locfr TYPE /sapapo/locno
          iv_mth TYPE zapo_ntbk_prd iv_pdh1 TYPE zapo_pdh1 iv_pdh2 TYPE zapo_pdh2
          iv_dmd_zvtweg TYPE vtweg iv_req_ratio TYPE zapo_req_inp_qty
          iw_output TYPE zapo_ntbk_detail
    CHANGING ct_output_agg TYPE zapo_ntbk_detail_t it_prod_dshpex TYPE zapo_ntbk_detail_t.
    DATA: lw_agg TYPE zapo_ntbk_detail, lw_prod TYPE zapo_ntbk_detail, lv_qty TYPE zapo_ntbk_qty.
    CLEAR lv_qty.
    LOOP AT it_prod_dshpex INTO lw_prod
      WHERE ntbk_matnr = iv_matnr AND ntbk_locfr = iv_locfr
        AND pdh1 = iv_pdh1 AND pdh2 = iv_pdh2 AND zvtweg = iv_dmd_zvtweg.
      IF iv_mth IS NOT INITIAL AND lw_prod-ntbk_mth <> iv_mth. CONTINUE. ENDIF.
      lv_qty = lv_qty + ( lw_prod-ntbk_qty * iv_req_ratio ).
    ENDLOOP.
    lw_agg = iw_output.
    CLEAR: lw_agg-zvtweg, lw_agg-ntbk_qty.
    lw_agg-ntbk_qty = lv_qty.
    COLLECT lw_agg INTO ct_output_agg.
  ENDFORM.

  FORM flush_output_agg
    USING iv_zvtweg_combined TYPE zapo_ntbk_ref-zvtweg
    CHANGING ct_output_agg TYPE zapo_ntbk_detail_t ct_output TYPE zapo_ntbk_detail_t.
    DATA lw_agg TYPE zapo_ntbk_detail.
    LOOP AT ct_output_agg INTO lw_agg.
      lw_agg-zvtweg = iv_zvtweg_combined.
      APPEND lw_agg TO ct_output.
    ENDLOOP.
    CLEAR ct_output_agg.
  ENDFORM.

ENDFUNCTION.
```

> **Implementation note:** Section 9 above abbreviates the three `ZAPO_NTBK_COMB_PREPARE` branches with `"... same filters and BOM loop as above ..."`. When pasting into SAP, copy the **full** original logic for division/location/BOM from your current include and only replace the `lt_prod_dshpex_op` **APPEND** blocks with `PERFORM sum_prod_dshp_to_agg` + `PERFORM flush_output_agg` as described in § 5. Also fix any typos introduced in abbreviated paste (`motionmotion` / `motionmotion` should remain your original field names `motion` → `div`).

---

## 10. Summary

| Item | Detail |
|------|--------|
| **Cause** | `lw_output_1-zvtweg = lw_prod_dshpex_op-zvtweg` + `APPEND` per prod/demand channel |
| **Fix** | `COLLECT` into `lt_output_agg` without `zvtweg`, then set `zvtweg` from `<lfs_input>-zvtweg` once |
| **Scope** | All four `lt_prod_dshpex_op` loops + flush after each `lt_dmdvsup` loop |
| **Transport** | Update FM `ZAPO_NTBK_DMD_DSHP_EXCLUDE` in include `SAPLZAPO_NTBK` (your transport AD1K917417 area) |

**Document:** `ZAPO_NTBK_DMD_DSHP_EXCLUDE_ZVTWEG_Sum_Correction.md`  
**Path:** `Netback Calculation\`  
**Author note:** mahesh pathak / omkar more — CD 8085121 follow-up (ZVTWEG combined output)
