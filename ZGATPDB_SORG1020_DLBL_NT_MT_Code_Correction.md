# ZGATPDB — SORG 1020 (GP block) & NT delivery-block (DLBL) correction

**System:** DEVSCMAD1 (`SAP_MVP_AD1`) — source reviewed via `ZAPO_GATP_ALLOCATION_F005`, `ZAPO_GATP_ALLOCATION_TOP1`  
**Program / T-code:** `ZAPO_GATP_ALLOCATION_REPORT` / `ZGATPDB`  
**Include (main changes):** `ZAPO_GATP_ALLOCATION_F005`  
**Types:** `ZAPO_GATP_ALLOCATION_TOP1`  
**Version:** 1.0 — 20/05/2026  

---

## 1. Observations (your request)

| # | Observation | Expected behaviour |
|---|-------------|-------------------|
| **1** | Sales organisation **`SORG = 1020`** — MT/NT logic does not exclude **GP block** quantity | For **`VKORG = '1020'`**, quantity on lines with delivery block **`GP`** must **not** enter cap, credit add-on, PAG MT/NT derivation, or **`get_zapo_so_list`** NT/MT totals |
| **2** | **NT MR** calculation includes blocks **TF**, **FD**, etc. | For **NT** (`REQ_NT = 'KL'`), only delivery block **`DLBL`** in **`FL`**, **`BE`**, **`OI`**, **`DZ`** may contribute; all other non-empty blocks (e.g. **`TF`**, **`FD`**, **`GP`**) must be **excluded** |

---

## 2. Field mapping (UI name → ABAP on `ZAPO_SO_LIST`)

| Your label | ABAP field (table `ZAPO_SO_LIST`) | Notes |
|------------|-----------------------------------|--------|
| **SORG** | **`VKORG`** | Already selected in CHANGE C / `get_zapo_so_list`; not used in filters today |
| **DLBL** (delivery block) | **`LIFSK`** | Standard SD delivery block; already used in **`get_so_list`** display SELECT, **missing** in MT/NT / popup path |

Confirm in **SE11** / **SE16** that `ZAPO_SO_LIST-LIFSK` is populated for your test orders (same values as ECC `VBAK-LIFSK` / item schedule block).

---

## 3. Root cause (AD1 code)

### 3.1 No `LIFSK` on MT/NT extract types

`gty_apo_so_list` in **`ZAPO_GATP_ALLOCATION_TOP1`** has **`vkorg`** but **no `lifsk`**:

```abap
TYPES: BEGIN OF gty_apo_so_list,
        ...
        vkorg  TYPE vkorg,
        ...
        bmeng  TYPE zbmeng,
        req_nt TYPE zreq_nt,
        req_mt TYPE zreq_mt,
        ...
       END OF gty_apo_so_list.
```

`get_zapo_so_list` SELECT does not read **`lifsk`**, so **TF / FD / GP** lines with **`REQ_NT = 'KL'`** still add **`BMENG`** to **`gw_apo_so_nt_*`** (popup inflation, e.g. 45 vs 30 MT).

### 3.2 CHANGE C (`get_data`) — monthly cap / credit block

`lty_so_all` SELECT includes **`vkorg`** but not **`lifsk`**. Loops building **`lt_so_cap`** and **`lt_cb_ord`** only check **`req_nt` / `req_mt`** and **`cmgst`**, not delivery block. **GP @ 1020** therefore still affects **`lw_so_cap_qty`**, **`inc_ord_quan`**, and **NT = inc_ord − MT**.

### 3.3 Display vs report path

**`get_so_list`** already selects **`lifsk`** into **`gty_disp_so_list`**; the **gATP MT/NT popup** path uses **`get_zapo_so_list`** + **`get_data`** CHANGE F — **not** the display SELECT.

---

## 4. Business rules (implementation)

Use one shared check (FORM or private method) everywhere SO lines affect NT/MT.

### 4.1 Constants (add in `ZAPO_GATP_ALLOCATION_TOP1` after other constants, or locally in `get_data`)

```abap
CONSTANTS: gc_vkorg_1020     TYPE vkorg VALUE '1020',
           gc_dlbl_gp         TYPE lifsk VALUE 'GP'.

CONSTANTS: BEGIN OF gc_nt_dlbl_allow,
             fl TYPE lifsk VALUE 'FL',
             be TYPE lifsk VALUE 'BE',
             oi TYPE lifsk VALUE 'OI',
             dz TYPE lifsk VALUE 'DZ',
           END OF gc_nt_dlbl_allow.
```

### 4.2 Helper — relevance for NT quantity from `ZAPO_SO_LIST`

```abap
*&---------------------------------------------------------------------*
*& Form zapo_so_line_nt_relevant
*&  im_vkorg, im_lifsk from ZAPO_SO_LIST
*&  NT MR: only blank (no block) or FL/BE/OI/DZ
*&  VKORG 1020: never count GP block
*&---------------------------------------------------------------------*
FORM zapo_so_line_nt_relevant
  USING    iv_vkorg TYPE vkorg
           iv_lifsk TYPE lifsk
  CHANGING cv_ok    TYPE abap_bool.

  DATA: lv_lifsk TYPE lifsk.

  cv_ok = abap_true.
  lv_lifsk = iv_lifsk.
  CONDENSE lv_lifsk.

  " Obs 1: Sales org 1020 — GP block qty must not count
  IF iv_vkorg = gc_vkorg_1020 AND lv_lifsk = gc_dlbl_gp.
    cv_ok = abap_false.
    RETURN.
  ENDIF.

  " Obs 2: NT MR — only FL, BE, OI, DZ (or no delivery block)
  IF lv_lifsk IS NOT INITIAL.
    IF lv_lifsk <> gc_nt_dlbl_allow-fl AND
       lv_lifsk <> gc_nt_dlbl_allow-be AND
       lv_lifsk <> gc_nt_dlbl_allow-oi AND
       lv_lifsk <> gc_nt_dlbl_allow-dz.
      cv_ok = abap_false.
    ENDIF.
  ENDIF.

ENDFORM.
```

Optional stricter variant for **cap only** (if functional wants **only** blocked NT lines FL/BE/OI/DZ and **exclude** unblocked lines): set `cv_ok = abap_false` when `lv_lifsk IS INITIAL` — **not** used below; default keeps normal (blank `LIFSK`) orders in NT.

### 4.3 Helper — relevance for monthly cap / credit (CHANGE C)

Same GP rule for **1020**; apply **NT DLBL allow-list** to lines that contribute to **cap** (recommended — aligns with FS BE/FL/DZ cap intent):

```abap
FORM zapo_so_line_cap_relevant
  USING    iv_vkorg TYPE vkorg
           iv_lifsk TYPE lifsk
  CHANGING cv_ok    TYPE abap_bool.

  PERFORM zapo_so_line_nt_relevant USING iv_vkorg iv_lifsk CHANGING cv_ok.

ENDFORM.
```

---

## 5. Code changes (paste-ready)

### 5.1 `ZAPO_GATP_ALLOCATION_TOP1` — extend type + constants

**A. Add `lifsk` to `gty_apo_so_list`:**

```abap
TYPES: BEGIN OF gty_apo_so_list,
        auart  TYPE auart,
        vbeln  TYPE vbeln_va,
        posnr  TYPE posnr_va,
        etenr  TYPE etenr,
        vkorg  TYPE vkorg,
        vtweg  TYPE vtweg,
        spart  TYPE spart,
        vkbur  TYPE vkbur,
        lifsk  TYPE lifsk,          " <<< ADD (DLBL)
        shcode TYPE kunwe,
        matnr  TYPE matnr,
        werks  TYPE zwerks,
        edatu  TYPE zedatu,
        bmeng  TYPE zbmeng,
        req_nt TYPE zreq_nt,
        req_mt TYPE zreq_mt,
        cmgst  TYPE zcmgst,
        zplgclass TYPE zplgclass,
       END OF gty_apo_so_list.
```

**B. Add constants** (section 4.1).

---

### 5.2 `ZAPO_GATP_ALLOCATION_F005` — method `get_data` (CHANGE C)

**A. Extend `lty_so_all`:**

```abap
TYPES: BEGIN OF lty_so_all,
        ...
        vkorg  TYPE vkorg,
        ...
        cmgst  TYPE zcmgst,
        lifsk  TYPE lifsk,    " <<< ADD
       END OF lty_so_all.
```

**B. Extend SELECT (CHANGE C block, ~line 509):**

```abap
SELECT   auart
         vbeln
         ...
         cmgst
         lifsk          " <<< ADD
  FROM zapo_so_list CLIENT SPECIFIED
  INTO TABLE lt_so_all
  ...
```

**C. In LOOP AT `lt_so_all` — before filling `lt_so_cap`**, insert:

```abap
DATA: lv_cap_ok TYPE abap_bool.

LOOP AT lt_so_all INTO lw_so_all.
  CLEAR lv_cap_ok.
  PERFORM zapo_so_line_cap_relevant
    USING    lw_so_all-vkorg
             lw_so_all-lifsk
    CHANGING lv_cap_ok.
  IF lv_cap_ok = abap_false.
    CONTINUE.
  ENDIF.

*--- Fill lt_so_cap (existing IF unchanged below)
  IF ( lw_so_all-abgru = space OR lw_so_all-bmeng > 0 )
  AND ( lw_so_all-req_nt = 'Z1' OR ... ).
    ...
  ENDIF.

*--- Fill lt_cb_ord (existing IF unchanged below)
  ...
ENDLOOP.
```

This stops **GP @ 1020** and **TF/FD/…** from inflating **`lt_so_cap_sum`** / **`lt_cb_ord`**, so CHANGE F MT/NT for **`p_pm` / `p_np`** uses correct **`inc_ord_quan`** and NT.

---

### 5.3 `ZAPO_GATP_ALLOCATION_F005` — method `get_zapo_so_list`

**A. Extend SELECT (~line 3166):**

```abap
SELECT auart  vbeln  posnr etenr
       vkorg  vtweg  spart vkbur
       lifsk                    " <<< ADD
       shcode matnr  werks edatu
       bmeng  req_nt req_mt cmgst
       zplgclass
  FROM zapo_so_list
  INTO TABLE lt_apo_so_list
  ...
```

**B. After `SELECT` / unit conversion loop, before building `gt_apo_so_list`**, filter driver table:

```abap
DATA: lv_nt_ok TYPE abap_bool.

LOOP AT lt_apo_so_list ASSIGNING <lfs_apo_so_list>.
  PERFORM zapo_so_line_nt_relevant
    USING    <lfs_apo_so_list>-vkorg
             <lfs_apo_so_list>-lifsk
    CHANGING lv_nt_ok.
  IF lv_nt_ok = abap_false.
    DELETE lt_apo_so_list INDEX sy-tabix.
  ENDIF.
ENDLOOP.
```

**C. Keep existing split** (`DELETE ... req_nt NE 'KL'` / `req_mt NE 'KL'`). NT totals then only see allowed DLBL (+ unblocked lines).

**Optional — MT SO add-on (`lt_apo_so_list_mt`):** If MT must also ignore GP@1020 only:

```abap
FORM zapo_so_line_mt_relevant ... " exclude vkorg 1020 + lifsk GP only
```

Apply the same DELETE loop **before** split, or only on `lt_apo_so_list_mt` after split.

---

### 5.4 `ZAPO_GATP_ALLOCATION_F005` — `collect_final_output` (email / prime bucket)

When **`gt_apo_so_list`** is filled from **`get_zapo_so_list`**, NT loops (~3773) already use filtered data **if step 5.3B is done**.

If **`gt_apo_so_list`** is populated elsewhere without filter, add inside **both** loops over `lt_apo_so_list_nt` / `_mt`:

```abap
PERFORM zapo_so_line_nt_relevant
  USING    lw_apo_so_list_nt-vkorg
           lw_apo_so_list_nt-lifsk
  CHANGING lv_nt_ok.
CHECK lv_nt_ok = abap_true.
```

---

### 5.5 Structural recommendation — `p_np` vs duplicate CHANGE F

AD1 still has a separate **`ELSEIF p_np`** block (~2837) duplicating PM MT/NT logic but **without** `lt_output_temp` → **`inc_ord_quan`** often **0**.

**Recommended (same as prior analysis):**

1. Change **`IF p_pm = abap_true OR p_sto = abap_true`** to **`IF p_pm = abap_true OR p_sto = abap_true OR p_np = abap_true`** for the block that fills **`inc_ord_quan`** from **`lt_output_temp`** and runs CHANGE F.
2. **Remove** the duplicate **`ELSEIF p_np`** MT/NT block (~2854–2937) after LIFSK/cap fixes are in CHANGE F.

Keep **`p_np`**-only BAPI loop for **`p_saleqty` / `p_alc_adj` / `p_stock`** if still required.

---

## 6. Where each observation is fixed

| Observation | Mechanism | Location |
|-------------|-----------|----------|
| **GP @ SORG 1020** | `zapo_so_line_nt_relevant` → `lifsk = GP` + `vkorg = 1020` → exclude | CHANGE C cap/credit; `get_zapo_so_list`; optional `collect_final_output` |
| **NT only FL/BE/OI/DZ** | Same FORM: non-empty `lifsk` not in allow-list → exclude | `get_zapo_so_list` (NT split); CHANGE C cap (recommended) |
| Popup **45 → 30 MT** | Excluded **15 MT GP** line no longer in **`gw_apo_so_nt_pp_plant`** | `get_gatprep_data` ← `get_zapo_so_list` |

---

## 7. Test plan

| ID | Setup | Expected |
|----|--------|----------|
| T1 | `VKORG=1020`, `LIFSK=GP`, `REQ_NT=KL`, BMENG=15 | **0** contribution to NT plant popup and CHANGE F **`no_touch`** |
| T2 | `VKORG=1020`, `LIFSK=FL`, `REQ_NT=KL` | Counts toward NT (if plgclass/location rules pass) |
| T3 | Any org, `LIFSK=TF` or `FD`, `REQ_NT=KL` | **Excluded** from NT MR / `gw_apo_so_nt_*` |
| T4 | `B120MA` / `3605` / div `23` / GC test case | NT Plant **30** (not **45**) after fix + transport |
| T5 | `VKORG≠1020`, `LIFSK=GP` | Confirm with functional whether GP is excluded globally or **only 1020** (this spec: **only 1020**) |

---

## 8. Transport checklist

- [ ] `ZAPO_GATP_ALLOCATION_TOP1` — type + constants  
- [ ] `ZAPO_GATP_ALLOCATION_F005` — FORM(s) + `get_data` + `get_zapo_so_list` + optional `collect_final_output`  
- [ ] Syntax check / activation `ZAPO_GATP_ALLOCATION_REPORT`  
- [ ] `ZGATPDB` regression: Prime / NP param, division 23, sales org **1020**  
- [ ] SE16 spot-check: `LIFSK` populated on failing SO lines  

---

## 9. Reference — current AD1 gaps (no filter)

**`get_zapo_so_list` SELECT (excerpt):** no `lifsk`, no `vkorg` filter:

```abap
SELECT ... vkorg ... bmeng req_nt req_mt cmgst zplgclass
  FROM zapo_so_list
  INTO TABLE lt_apo_so_list
  WHERE ... abgru = space AND edatu IN so_date AND delind = space.
DELETE lt_apo_so_list_nt WHERE req_nt NE 'KL'.
" No DELETE on lifsk — TF, FD, GP still counted
```

**CHANGE C cap loop:** no `lifsk` in `lty_so_all` — GP@1020 still in **`lt_so_cap_sum-wmeng`**.

---

*Document prepared from live AD1 includes. RFC table read was unavailable (401); validate `LIFSK` values on real orders in SE16 before production transport.*
