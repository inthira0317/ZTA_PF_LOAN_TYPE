# ZTA_ESSMSS_PF_LOAN -- Complete ABAP Code and Explanation

> This document explains the supplied implementation of the PF Loan ESS
> function module. The code is reproduced in logical sections so that
> each operation can be understood. It is based on the source supplied
> for this documentation; it is not a claim that the program has been
> syntax-checked or executed in an SAP system.


## 1. Business purpose

`ZTA_ESSMSS_PF_LOAN` supports the employee self-service PF loan process.
Depending on the importing flags, it can:

-   Display configured PF loan categories.
-   Retrieve previous PF membership history.
-   Check whether an employee is eligible for a PF loan.
-   Receive and save a new PF loan application.
-   Update an existing application.
-   Retrieve pending and approved employee requests.
-   Retrieve requests for approver processing.
-   Calculate the employee's PF balance using vendor open-item data.

### Main database tables

  -----------------------------------------------------------------------
  Table                               Purpose
  ----------------------------------- -----------------------------------
  `ZTA_ESS_PF_LTYPE`                  Loan category configuration and
                                      eligibility rules

  `ZTA_PF_MEM_HIS_K`                  Previous PF membership history

  `ZTA_ESS_PF_MAIN`                   Main PF loan application data

  `ZTA_ESS_PF_TXN`                    PF loan status/change transaction
                                      data

  `ZTA_T_HRMAST`                      Employee/vendor mapping

  `BSIK`                              Vendor open items used in balance
                                      calculation

  `PA0001`                            Employee organizational assignment
                                      and name

  `T500P`                             Personnel area descriptions
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 2. Function module interface

``` abap
FUNCTION zta_essmss_pf_loan.
*"----------------------------------------------------------------------
*"*"Local Interface:
*"  IMPORTING
*"     VALUE(IM_INT_APP_FLAG) TYPE BOOLE_D OPTIONAL
*"     VALUE(IM_W_R) TYPE BOOLE_D OPTIONAL
*"     VALUE(IM_APP_ID) TYPE ZHR_ESS_WF_MASTR-WFID OPTIONAL
*"     VALUE(IM_EMP_NO) TYPE PERNR_D OPTIONAL
*"     VALUE(IM_WITHDRAWREASON) TYPE ZHR_ESSMSS_PFLON-ZLOAN_TYPE
*"       OPTIONAL
*"     VALUE(IM_WFST) TYPE ZTA_ESS_PF_MAIN-ZSTATUS_ID OPTIONAL
*"  EXPORTING
*"     VALUE(EX_FLAG) TYPE CHAR1
*"     VALUE(EX_ERRORMESSAGE) TYPE CHAR100
*"     VALUE(ET_PF_SUMMARY) TYPE ZTA_T_PF_SUMMARY
*"  TABLES
*"      ET_PFLOAN_TYPE STRUCTURE ZTA_ESS_PF_LTYPE OPTIONAL
*"      ET_PF_LOAN TYPE ZTA_ESS_PF_T OPTIONAL
*"      ET_PFMEM_HIS TYPE ZTA_T_PFMEM_HIS OPTIONAL
*"----------------------------------------------------------------------
```

### Importing parameters

  -----------------------------------------------------------------------
  Parameter                           Description
  ----------------------------------- -----------------------------------
  `IM_INT_APP_FLAG`                   Identifies the application context:
                                      employee input (`I`), output/save
                                      (`O`), or approver (`A`).

  `IM_W_R`                            Identifies the requested operation
                                      within that context.

  `IM_APP_ID`                         Workflow ID. It is declared but is
                                      not used in the supplied
                                      implementation.

  `IM_EMP_NO`                         Employee number or approver
                                      employee number.

  `IM_WITHDRAWREASON`                 Selected PF loan category.

  `IM_WFST`                           Workflow status used when selecting
                                      approver requests.
  -----------------------------------------------------------------------

### Exporting and table parameters

  -----------------------------------------------------------------------
  Parameter                           Description
  ----------------------------------- -----------------------------------
  `EX_FLAG`                           Result indicator: `S` for success
                                      and `E` for error.

  `EX_ERRORMESSAGE`                   Result or validation message.

  `ET_PF_SUMMARY`                     PF summary output; the supplied
                                      code does not populate it.

  `ET_PFLOAN_TYPE`                    Available PF loan categories.

  `ET_PF_LOAN`                        PF loan application records.

  `ET_PFMEM_HIS`                      Previous PF membership records.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. Operation routing -- main function module

The function module uses `IM_INT_APP_FLAG` and `IM_W_R` together to
decide which block should run.

  `IM_INT_APP_FLAG`   `IM_W_R`     Operation
  ------------------- ------------ ---------------------------------------------------
  `I`                 `D`          Display PF loan types
  `I`                 `P`          Retrieve previous PF membership
  `I`                 `E`          Run eligibility check
  `O`                 `W`          Validate and save/update application
  `I`                 `R`          Retrieve pending employee requests
  `I`                 `H`          Retrieve approved employee requests
  `A`                 `A` or `J`   Retrieve approver requests using transaction data
  `A`                 `R`          Retrieve requests by main-table status

### 3.1 Display PF loan types -- `I + D`

**Code**

``` abap
IF im_int_app_flag = 'I'
   AND im_w_r = 'D'.

  SELECT *
    FROM zta_ess_pf_ltype
    INTO TABLE gt_pfloan_type.

  IF sy-subrc = 0.
    ex_flag = 'S'.
    ex_errormessage = 'Loan type is found'.

    et_pfloan_type[] = gt_pfloan_type[].

  ELSE.
    ex_flag = 'E'.
    ex_errormessage = 'Loan type is not available'.
  ENDIF.
```

**Breakdown**

1.  The system checks whether the caller is in employee input mode (`I`)
    and the requested operation is display (`D`).
2.  `SELECT *` reads the configured PF loan categories from
    `ZTA_ESS_PF_LTYPE`.
3.  The records are stored in internal table `GT_PFLOAN_TYPE`.
4.  `SY-SUBRC = 0` means the selection found records.
5.  On success, `EX_FLAG` is set to `S`, a success message is returned,
    and the internal table is copied to `ET_PFLOAN_TYPE`.
6.  If no records are found, the function returns `E`.

**Business purpose:** Display the loan categories that an employee can
select in ESS.

### 3.2 Retrieve previous PF membership -- `I + P`

**Code**

``` abap
ELSEIF im_int_app_flag = 'I'
   AND im_w_r = 'P'.

  SELECT *
    FROM zta_pfmem_his_k
    INTO TABLE gt_pfmem_his_k
    WHERE pernr = im_emp_no.

  IF sy-subrc = 0.

    ex_flag = 'S'.
    ex_errormessage = 'Previous membership is found'.

    LOOP AT gt_pfmem_his_k INTO gs_pfmem_his_k.

      MOVE-CORRESPONDING gs_pfmem_his_k
        TO gs_pfmem_his.

      APPEND gs_pfmem_his TO et_pfmem_his.

    ENDLOOP.

  ELSE.

    ex_flag = 'E'.
    ex_errormessage = 'Previous membership not found'.

  ENDIF.
```

**Breakdown**

1.  The system checks for employee input mode and previous-history
    operation.
2.  It reads `ZTA_PF_MEM_HIS_K` for the employee number supplied in
    `IM_EMP_NO`.
3.  If records exist, it sets the success flag and message.
4.  `LOOP AT` processes each history record.
5.  `MOVE-CORRESPONDING` transfers fields with matching names from the
    database work area to the output work area.
6.  `APPEND` adds each converted record to `ET_PFMEM_HIS`.
7.  If no records exist, the function returns an error.

**Business purpose:** Provide the employee's previous PF service history
so that previous-company membership can be included in the eligibility
calculation.

### 3.3 Run eligibility check -- `I + E`

**Code**

``` abap
ELSEIF im_int_app_flag = 'I'
   AND im_w_r = 'E'.

  gv_flag = 'E'.

  PERFORM loan_check
    USING im_emp_no im_withdrawreason
    CHANGING ex_flag
             ex_errormessage.

  IF ex_flag NE 'E'.
    ex_flag = 'S'.
  ENDIF.
```

**Breakdown**

1.  `GV_FLAG` is set to `E`.
2.  `PERFORM LOAN_CHECK` calls the eligibility form.
3.  `IM_EMP_NO` is passed as the employee number.
4.  `IM_WITHDRAWREASON` is passed as the loan category.
5.  The form returns its result through `EX_FLAG` and `EX_ERRORMESSAGE`.
6.  If the returned flag is not `E`, the function sets it to `S`.

**Important:** In the supplied implementation, `GV_FLAG = 'E'` prevents
the amount-validation blocks guarded by `GV_FLAG NE 'E'` from executing.
This branch therefore does not perform all the amount checks.

### 3.4 Submit or update a PF loan application -- `O + W`

This is the application-processing and database-save branch.

**Step 1 -- Receive the application data**

``` abap
ELSEIF im_int_app_flag = 'O'
   AND im_w_r = 'W'.

  CLEAR: gs_pf_ess_main,
         gs_pf_ess_txn,
         gv_unique_id.

  LOOP AT et_pf_loan INTO gs_pf_ess_s.
    MOVE-CORRESPONDING gs_pf_ess_s
      TO gs_pf_ess_main.
  ENDLOOP.

  lv_req_amt = gs_pf_ess_main-zloan_amt.
```

Explanation:

-   The main and transaction work areas and application number variable
    are cleared.
-   The incoming application table is processed.
-   Corresponding fields are moved to `GS_PF_ESS_MAIN`.
-   The requested amount is copied to `LV_REQ_AMT`.

The loop leaves the work area containing the last row. This
implementation assumes that the caller supplies one application row.

**Step 2 -- Recheck eligibility**

``` abap
PERFORM loan_check
  USING im_emp_no gs_pf_ess_main-zloan_type
  CHANGING ex_flag
           ex_errormessage.

IF ex_flag NE 'E'.
```

The function calls `LOAN_CHECK` before saving. The application continues
only if the form does not return an error.

**Step 3 -- Read loan category description**

``` abap
SELECT SINGLE *
  FROM zta_ess_pf_ltype
  INTO gs_pfloan_type
  WHERE zloan_type = gs_pf_ess_s-zloan_type.

IF sy-subrc IS INITIAL.
  gs_pf_ess_main-zloan_type_desc =
    gs_pfloan_type-zloan_type_desc.
ENDIF.
```

The selected loan category is used to read its description. If a
matching category exists, the description is copied to the main
application record.

**Step 4 -- Generate a number for a new application**

``` abap
IF gs_pf_ess_main-zappln_no IS INITIAL
   OR gs_pf_ess_main-zappln_no EQ '0000000000'.

  CALL FUNCTION 'NUMBER_GET_NEXT'
    EXPORTING
      nr_range_nr = '01'
      object      = 'ZHR_ESS_PF'
    IMPORTING
      number      = gv_unique_id.

  gs_pf_ess_main-zcreated_by =
    gs_pf_ess_main-zchanged_by.

  gs_pf_ess_main-zappln_no = gv_unique_id.
  gs_pf_ess_main-zcreated_date = sy-datum.
  gs_pf_ess_main-zcreated_time = sy-uzeit.

  gs_pf_ess_txn-zchanged_by =
    gs_pf_ess_main-zcreated_by.
```

Explanation:

1.  The function checks whether the application number is initial or all
    zeroes.
2.  `NUMBER_GET_NEXT` gets the next number from number range object
    `ZHR_ESS_PF`, interval `01`.
3.  The generated number is assigned to `ZAPPLN_NO`.
4.  The creator is assigned from the changer field supplied in the
    application work area.
5.  The creation date and time are filled using `SY-DATUM` and
    `SY-UZEIT`.
6.  The transaction changer is initialized with the creator.

**Business purpose:** Generate a unique application number for a new PF
loan request.

**Step 5 -- Prepare an existing application for update**

``` abap
ELSE.

  gv_unique_id = gs_pf_ess_main-zappln_no.

  gs_pf_ess_txn-zstatus_id =
    gs_pf_ess_main-zstatus_id.

  gs_pf_ess_txn-zchanged_by =
    gs_pf_ess_main-zchanged_by.

  gs_pf_ess_txn-zremarks =
    gs_pf_ess_main-zremarks.

  gs_main1-zstatus = gs_pf_ess_main-zstatus.

  CLEAR gs_pf_ess_main.

  SELECT SINGLE *
    FROM zta_ess_pf_main
    INTO gs_pf_ess_main
    WHERE zappln_no = gv_unique_id.

  gs_pf_ess_main-zstatus_id =
    gs_pf_ess_txn-zstatus_id.

  gs_pf_ess_main-zchanged_by =
    gs_pf_ess_txn-zchanged_by.

  gs_pf_ess_main-zremarks =
    gs_pf_ess_txn-zremarks.

  gs_pf_ess_main-zstatus = gs_main1-zstatus.

ENDIF.
```

Explanation:

1.  The existing application number is stored in `GV_UNIQUE_ID`.
2.  The incoming status, changer and remarks are copied into the
    transaction work area.
3.  The incoming status text is temporarily stored in `GS_MAIN1`.
4.  The main work area is cleared.
5.  The existing main record is retrieved from `ZTA_ESS_PF_MAIN` using
    the application number.
6.  The changed status, changer, remarks and status text are copied back
    into the main work area.

This preserves existing main-table fields while applying the incoming
update fields.

**Step 6 -- Set payment date for approved applications**

``` abap
IF gs_pf_ess_main-zstatus_id = '99'.
  gs_pf_ess_main-zpayment_date = sy-datum.
ENDIF.
```

When status `99` is reached, the payment date is set to the current SAP
system date.

**Step 7 -- Save the main application record**

``` abap
APPEND gs_pf_ess_main TO gt_main.

MODIFY zta_ess_pf_main FROM TABLE gt_main.

COMMIT WORK.
```

Explanation:

-   The prepared main record is appended to `GT_MAIN`.
-   `MODIFY ... FROM TABLE` inserts or updates the database records
    based on the table's key.
-   `COMMIT WORK` commits the database changes.

**Step 8 -- Prepare and save transaction data**

``` abap
gs_pf_ess_txn-zappln_no =
  gs_pf_ess_main-zappln_no.

gs_pf_ess_txn-zstatus_id =
  gs_pf_ess_main-zstatus_id.

gs_pf_ess_txn-zchanged_date = sy-datum.
gs_pf_ess_txn-zchanged_time = sy-uzeit.

gs_pf_ess_txn-zremarks =
  gs_pf_ess_main-zremarks.

MODIFY zta_ess_pf_txn FROM gs_pf_ess_txn.

COMMIT WORK.
```

Explanation:

-   The transaction record is assigned the application number and
    status.
-   The current date and time are recorded.
-   Remarks are copied from the main record.
-   The transaction record is saved to `ZTA_ESS_PF_TXN`.
-   A second commit is executed.

**Step 9 -- Return the application status message**

``` abap
IF sy-subrc IS INITIAL.

  ex_flag = 'S'.

  CASE gs_pf_ess_main-zstatus_id.

    WHEN '01'.
      ex_errormessage =
        'PF Loan Submitted Successfully '.

    WHEN '02'.
      ex_errormessage =
        'Level 1 Approver Approved Pending with Final Approver. '.

    WHEN '99'.
      ex_errormessage =
        'PF Loan approved.'.

  ENDCASE.

ENDIF.
```

  Status   Meaning in the supplied code
  -------- ---------------------------------------------------
  `01`     Submitted
  `02`     Level 1 approver approved; pending final approver
  `99`     Approved

The function returns a success message according to the final status.

### 3.5 Retrieve pending requests -- `I + R`

**Code**

``` abap
ELSEIF im_int_app_flag = 'I'
   AND im_w_r = 'R'.

  CLEAR et_pf_loan[].

  SELECT *
    FROM zta_ess_pf_main
    INTO TABLE gt_pf_ess_main
    WHERE zcreated_by = im_emp_no
      AND zloan_type = im_withdrawreason
      AND ( zstatus_id = '01'
         OR zstatus_id = '02' ).

  IF sy-subrc = 0.

    LOOP AT gt_pf_ess_main INTO gs_pf_ess_main.

      MOVE-CORRESPONDING gs_pf_ess_main
        TO gs_pf_ess_s.

      APPEND gs_pf_ess_s TO et_pf_loan.

    ENDLOOP.

    ex_flag = 'S'.
    ex_errormessage = 'Pending request found'.

  ELSE.

    ex_flag = 'E'.
    ex_errormessage = 'No pending PF Loan request'.

  ENDIF.
```

**Breakdown**

1.  Clear the output table so that previous rows are not retained.
2.  Select main-table applications created by the employee.
3.  Restrict the selection to the requested loan category.
4.  Restrict the status to `01` or `02`.
5.  Loop through the matching records.
6.  Use `MOVE-CORRESPONDING` to map the main-table fields to the output
    structure.
7.  Append each record to `ET_PF_LOAN`.
8.  Return success if requests exist; otherwise return the
    no-pending-request message.

**Business purpose:** Show the employee their pending PF loan
applications for the selected category.

### 3.6 Retrieve approved requests -- `I + H`

**Code**

``` abap
ELSEIF im_int_app_flag = 'I'
   AND im_w_r = 'H'.

  CLEAR et_pf_loan[].

  SELECT *
    FROM zta_ess_pf_main
    INTO TABLE gt_pf_ess_main
    WHERE pernr = im_emp_no
      AND zstatus_id = '99'.

  IF sy-subrc = 0.

    LOOP AT gt_pf_ess_main INTO gs_pf_ess_main.

      MOVE-CORRESPONDING gs_pf_ess_main
        TO gs_pf_ess_s.

      APPEND gs_pf_ess_s TO et_pf_loan.

    ENDLOOP.

    ex_flag = 'S'.
    ex_errormessage = 'Approved PF Loan Request'.

  ELSE.

    ex_flag = 'E'.
    ex_errormessage = 'No Approved PF Loan request'.

  ENDIF.
```

**Breakdown**

1.  Clear the output table.
2.  Read the employee's main-table applications.
3.  Select only records with status `99`.
4.  Map the selected records to the output structure.
5.  Append them to `ET_PF_LOAN`.
6.  Return a success or no-approved-request message.

**Business purpose:** Display approved PF loan applications belonging to
the employee.

### 3.7 Approver-side request retrieval -- `A`

The approver branch starts by clearing the output table.

``` abap
ELSEIF im_int_app_flag = 'A'.

  CLEAR et_pf_loan[].

  CASE im_w_r.
```

#### Case A -- `IM_W_R = 'A'` or `'J'`

``` abap
WHEN 'A' OR 'J'.

  SELECT DISTINCT
     a~pernr,
     a~zappln_no,
     a~zloan_type,
     a~zloan_type_desc,
     a~zcreated_date,
     a~zcreated_time,
     a~zcreated_by,
     a~zpayment_date,
     a~zloan_amt,
     a~zstatus_id,
     a~zstatus,
     b~zremarks,
     b~zchanged_by,
     a~zloan_amt

    FROM zta_ess_pf_main AS a

    INNER JOIN zta_ess_pf_txn AS b
      ON b~zappln_no = a~zappln_no

    INTO CORRESPONDING FIELDS OF TABLE @gt_pf_ess_s

    WHERE b~zstatus_id = @im_wfst
      AND b~zchanged_by = @im_emp_no.
```

**Breakdown**

1.  `A` is the alias for the main application table.
2.  `B` is the alias for the transaction table.
3.  The inner join connects the main and transaction records through
    application number.
4.  `SELECT DISTINCT` removes duplicate result rows.
5.  The query selects application information from the main table and
    remarks/changer information from the transaction table.
6.  The transaction status must match `IM_WFST`.
7.  The transaction changer must match the approver employee number
    supplied in `IM_EMP_NO`.
8.  The results are stored in `GT_PF_ESS_S`.

**Business purpose:** Retrieve applications associated with the
specified workflow status and approver employee number.

#### Case R -- `IM_W_R = 'R'`

``` abap
WHEN 'R'.

  SELECT *
    FROM zta_ess_pf_main
    INTO CORRESPONDING FIELDS OF TABLE gt_pf_ess_s
    WHERE zstatus_id = im_wfst.

ENDCASE.
```

This branch reads the main-table applications whose status matches
`IM_WFST`. Unlike the preceding branch, it does not join the transaction
table or filter by `ZCHANGED_BY`.

#### Enrich the request with employee and personnel-area details

``` abap
SELECT persa name1
  FROM t500p
  INTO TABLE gt_persa.

SELECT pernr sname persg persk werks
  FROM pa0001
  INTO TABLE gt_name
  WHERE endda EQ '99991231'.

LOOP AT gt_pf_ess_s INTO gs_pf_ess_s.

  READ TABLE gt_name INTO gs_name
    WITH KEY pernr = gs_pf_ess_s-pernr.

  gs_pf_ess_s-emp_name = gs_name-name.

  READ TABLE gt_persa INTO gs_persa
    WITH KEY persa = gs_name-werks.

  gs_pf_ess_s-persa_text = gs_persa-name1.

  APPEND gs_pf_ess_s TO et_pf_loan.

  CLEAR gs_pf_ess_s.

ENDLOOP.
```

**Breakdown**

1.  Read personnel area descriptions from `T500P`.
2.  Read employee assignment data from `PA0001`.
3.  Loop through the selected PF loan requests.
4.  Read the employee assignment using the request's personnel number.
5.  Copy the employee name to `EMP_NAME`.
6.  Read the personnel area description using the employee's personnel
    area.
7.  Copy the description to `PERSA_TEXT`.
8.  Append the enriched request to `ET_PF_LOAN`.

**Business purpose:** Provide the approver with application details
along with employee name and personnel-area description.

------------------------------------------------------------------------

## 4. Complete eligibility form -- `FORM LOAN_CHECK`

### Purpose

`LOAN_CHECK` determines whether the employee satisfies the configured PF
withdrawal rules.

The supplied form contains four major checks:

1.  Minimum PF membership duration.
2.  Maximum number of withdrawals.
3.  Minimum withdrawal amount and 75% balance calculation.
4.  Requested amount compared with the final eligible balance.

### Form interface

``` abap
FORM loan_check USING p_emp_no TYPE pernr_d
                      p_wd_reason TYPE zhr_essmss_pflon-zloan_type
                CHANGING p_flag TYPE char1
                         p_message TYPE char100.
```

-   `P_EMP_NO` receives the employee number.
-   `P_WD_REASON` receives the selected withdrawal category.
-   `P_FLAG` returns success/error status.
-   `P_MESSAGE` returns the eligibility result or error.

### 4.1 Read loan category configuration

``` abap
SELECT SINGLE *
  FROM zta_ess_pf_ltype
  INTO gs_pf_loan_type
  WHERE zloan_type = p_wd_reason.
```

The loan-type configuration is read to obtain:

-   `ZWD_ELIG_MONTHS` -- minimum membership months.
-   `ZMAX_WD_LIMIT` -- maximum permitted withdrawals.
-   `ZMIN_WD_AMT` -- minimum withdrawal amount.
-   `ZLOAN_PERCENTAGE` -- category-specific percentage.

### 4.2 Membership-duration check

**Business rule:** The employee must complete the configured membership
period using current-company and previous-company PF membership.

``` abap
CALL FUNCTION 'ZHR_HR_READ_INFOTYPE'
  EXPORTING
    ex_pernr = p_emp_no
    ex_infty = '0041'
    ex_begda = sy-datum
    ex_endda = sy-datum
  IMPORTING
    im_p0041 = lt_p0041.

READ TABLE lt_p0041 INTO ls_p0041
  WITH KEY zdatety = 'Z2'.

CALL FUNCTION 'FIMA_DAYS_AND_MONTHS_AND_YEARS'
  EXPORTING
    i_date_from = ls_p0041-zdate
    i_date_to = sy-datum
  IMPORTING
    e_months = lv_months.
```

**Line-by-line explanation**

1.  `ZHR_HR_READ_INFOTYPE` reads Infotype `0041` for the employee.
2.  The function is called for the current date.
3.  The returned date records are stored in `LT_P0041`.
4.  `READ TABLE` searches for date type `Z2`.
5.  `Z2` is treated by this implementation as the employee's
    current-company membership date.
6.  `FIMA_DAYS_AND_MONTHS_AND_YEARS` calculates the number of completed
    months between that date and the current date.
7.  The result is stored in `LV_MONTHS`.

The form then checks whether the current-company membership is below the
configured requirement.

``` abap
IF lv_months LT gs_pf_loan_type-zwd_elig_months.

  SELECT *
    FROM zta_pfmem_his_k
    INTO TABLE gt_pfmem_his_k
    WHERE pernr = p_emp_no.

  LOOP AT gt_pfmem_his_k INTO gs_pfmem_his_k.
    lv_months =
      lv_months + gs_pfmem_his_k-zno_of_months.
  ENDLOOP.

  IF lv_months LT gs_pf_loan_type-zwd_elig_months.

    p_flag = 'E'.

    CONCATENATE
      'Loan minimum eligible months -'
      gs_pf_loan_type-zwd_elig_months
      'Not serviced in PSU'
      INTO p_message
      SEPARATED BY space.

  ENDIF.

ENDIF.
```

**Explanation**

1.  If current-company months are sufficient, previous history is not
    needed for this check.
2.  If current-company months are insufficient, the system reads
    previous PF membership records.
3.  It loops through each previous membership record.
4.  `ZNO_OF_MONTHS` is added to the current membership month count.
5.  The combined duration is compared with the configured minimum.
6.  If the combined duration is still insufficient, the function returns
    an error.

**Example**

Suppose the category requires 12 months:

  Membership           Months
  ------------------ --------
  Current company           8
  Previous company          5
  Total                    13

The employee meets the 12-month requirement.

If the previous-company service were only 2 months, the total would be
10 months and the check would fail.

### 4.3 Maximum withdrawal count

**Business rule:** An employee cannot exceed the configured number of
withdrawals for a category. Special Circumstances (`SP`) has a
financial-year-based restriction.

``` abap
CLEAR: lv_wd_count,
       lv_fy_start.

IF p_flag NE 'E'.

  IF gs_pf_loan_type-zloan_type = 'SP'.
```

The maximum-withdrawal check is performed only if the membership check
has not already failed.

#### Special Circumstances -- financial-year start

``` abap
lv_d = sy-datum.

IF lv_d+4(2) LT 3.
  gv_check_mon = lv_d+0(4) - 1.
ELSE.
  gv_check_mon = lv_d+0(4).
ENDIF.

CONCATENATE gv_check_mon '0401'
  INTO gv_check_mon.

CONDENSE gv_check_mon NO-GAPS.

lv_fy_start = gv_check_mon.
```

**Explanation**

1.  `LV_D` receives the current SAP date.
2.  `LV_D+4(2)` extracts the month.
3.  If the month is January or February, the financial year started in
    the previous calendar year.
4.  Otherwise, the financial year started in the current calendar year.
5.  `CONCATENATE` adds `0401` to the selected year to create 1 April.
6.  The resulting value is assigned to `LV_FY_START`.

For example:

  Current date   Financial-year start
  -------------- ----------------------
  15.02.2027     01.04.2026
  15.07.2027     01.04.2027

#### Count Special Circumstances withdrawals

``` abap
SELECT COUNT( * )
  FROM zta_ess_pf_main
  INTO lv_wd_count
  WHERE pernr = p_emp_no
    AND zloan_type = 'SP'
    AND ( zstatus_id = '99'
       OR zstatus_id = '01'
       OR zstatus_id = '02' )
    AND zpayment_date >= lv_fy_start
    AND zpayment_date <= sy-datum.

IF lv_wd_count >= gs_pf_loan_type-zmax_wd_limit.

  p_flag = 'E'.

  CONCATENATE
    'Maximum'
    gs_pf_loan_type-zmax_wd_limit
    'withdrwals permitted for special circumstances'
    INTO p_message
    SEPARATED BY space.

ENDIF.
```

**Explanation**

The database count is restricted to:

-   The current employee.
-   Loan category `SP`.
-   Statuses `99`, `01` and `02`.
-   Payment dates within the current financial year.

If the count is greater than or equal to the configured maximum, the
request is rejected.

#### Count other withdrawal categories

``` abap
ELSE.

  IF gs_pf_loan_type-zmax_wd_limit IS NOT INITIAL.

    SELECT COUNT( * )
      FROM zta_ess_pf_main
      INTO lv_wd_count
      WHERE pernr = p_emp_no
        AND zloan_type = p_wd_reason
        AND ( zstatus_id = '99'
           OR zstatus_id = '01'
           OR zstatus_id = '02' ).

    IF lv_wd_count >= gs_pf_loan_type-zmax_wd_limit.

      p_flag = 'E'.

      p_message =
        'Maximum withdrwals permitted for these categories are exhausted'.

    ENDIF.

  ENDIF.

ENDIF.
```

**Explanation**

1.  For non-`SP` categories, the configured maximum is checked.
2.  If the maximum is not initial, the system counts applications for
    that employee and category.
3.  Statuses `99`, `01` and `02` are included.
4.  If the count reaches the configured maximum, an error is returned.

### 4.4 Minimum withdrawal amount and 75% balance check

**Business rule:** The requested amount must meet the configured
minimum. The amount must also satisfy the balance-based limit.

``` abap
IF p_flag NE 'E'.

  IF gv_flag NE 'E'.

    IF gs_pf_ess_main-zloan_amt <
       gs_pf_loan_type-zmin_wd_amt.

      p_flag = 'E'.
      p_message = gs_pf_loan_type-zmin_wd_amt.

      CONCATENATE
        'Minimum withdrawal amount shall be INR'
        p_message
        INTO p_message
        SEPARATED BY space.

    ELSE.

      PERFORM balance_amount USING p_emp_no.

      gs_final-bal_tot = 0 - gs_final-bal_tot.

      lv_elig_amt = 75 * gs_final-bal_tot / 100.

      IF gs_pf_main-zloan_amt GT lv_elig_amt.

        p_flag = 'E'.
        p_message = lv_elig_amt.

        CONCATENATE
          'Maximum withdrawal amount shall be INR'
          p_message
          INTO p_message
          SEPARATED BY space.

      ENDIF.

    ENDIF.

  ENDIF.

ENDIF.
```

**Explanation**

1.  The block runs only if the earlier validations have not returned an
    error.
2.  `GV_FLAG` controls whether the amount validation is executed.
3.  The requested amount is compared with `ZMIN_WD_AMT`.
4.  If it is below the configured minimum, an error is returned.
5.  Otherwise, the form calls `BALANCE_AMOUNT`.
6.  The balance sign is reversed using `0 - BAL_TOT`.
7.  The system calculates 75% of that balance.
8.  The requested amount is compared with this 75% amount.
9.  If the requested amount exceeds it, the form returns an error.

**Example**

If the balance used in this calculation is ₹100,000:

`75% of ₹100,000 = ₹75,000`

The requested amount must not exceed ₹75,000 at this stage.

### 4.5 Final eligible balance and requested amount validation

``` abap
IF p_flag NE 'E'.

  IF gv_flag NE 'E'.

    lv_elig_amt =
      lv_elig_amt *
      gs_pf_loan_type-zloan_percentage / 100.

    IF lv_req_amt IS INITIAL OR lv_req_amt <= 0.

      p_flag = 'E'.
      p_message = 'please enter the valid amount'.

    ELSEIF lv_elig_amt <= 0.

      p_flag = 'E'.
      p_message =
        'eligible member balnce is insufficient'.

    ELSEIF lv_req_amt < lv_min_amt.

      p_flag = 'E'.
      p_message =
        'Requested amount is below the minimum withdrawal amount'.

    ELSEIF lv_req_amt > lv_elig_amt.

      p_flag = 'E'.
      p_message =
        'Requested amount exceeds the eligible member balance '.

    ENDIF.

    p_flag = 'S'.
    p_message =
      'employee is eligilbe for pf withdrawals'.

  ENDIF.

ENDIF.

ENDFORM.
```

**Explanation**

1.  The 75% amount is further multiplied by the category-specific
    `ZLOAN_PERCENTAGE`.
2.  The requested amount must be positive and not initial.
3.  The eligible balance must be positive.
4.  The requested amount must not be below `LV_MIN_AMT`.
5.  The requested amount must not exceed `LV_ELIG_AMT`.
6.  If all conditions pass, the employee is considered eligible.

**Critical correction to consider:** In the supplied code, the final
assignments:

``` abap
p_flag = 'S'.
p_message = 'employee is eligilbe for pf withdrawals'.
```

run even after one of the preceding conditions sets `P_FLAG = 'E'`. This
overwrites the error. These success assignments must be placed in an
`ELSE` branch after the validation conditions.

Also review the amount sources: this form uses
`GS_PF_ESS_MAIN-ZLOAN_AMT`, `GS_PF_MAIN-ZLOAN_AMT`, `LV_REQ_AMT` and
`LV_MIN_AMT`. Confirm which work area/variable is authoritative and use
it consistently.

------------------------------------------------------------------------

## 5. Complete PF balance calculation -- `FORM BALANCE_AMOUNT`

### Purpose

This form calculates an employee's PF balance using the employee's
vendor number and open-item records from `BSIK`.

### Form interface

``` abap
FORM balance_amount USING p_emp_no TYPE pernr_d.
```

### 5.1 Read employee/vendor mapping

``` abap
SELECT *
  FROM zta_t_hrmast
  INTO TABLE gt_zta_t_hrmast
  WHERE pernr = p_emp_no
    AND lifnr = p_emp_no.
```

**Explanation**

1.  The form reads `ZTA_T_HRMAST`.
2.  It searches for the employee number.
3.  The supplied implementation also compares `LIFNR` with the employee
    number.
4.  The selected mapping records are placed in `GT_ZTA_T_HRMAST`.

### 5.2 Convert vendor number

``` abap
LOOP AT gt_zta_t_hrmast INTO gs_t_hrmast.

  CLEAR: vendor,
         gt_bsik[],
         gs_final.

  CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
    EXPORTING
      input = gs_t_hrmast-lifnr
    IMPORTING
      output = vendor.

ENDLOOP.
```

**Explanation**

1.  The mapping records are processed.
2.  Temporary balance data is cleared.
3.  `CONVERSION_EXIT_ALPHA_INPUT` converts the vendor number into SAP's
    internal ALPHA format.
4.  The converted vendor number is stored in `VENDOR`.

For example, a vendor number supplied without leading zeroes can be
converted to its internal fixed-length representation.

### 5.3 Read vendor open items

``` abap
SELECT *
  FROM bsik
  INTO TABLE gt_bsik
  WHERE lifnr = vendor
    AND bukrs = 'PF01'.
```

**Explanation**

-   `BSIK` contains vendor open-item information.
-   The selection uses the converted vendor number.
-   `BUKRS = 'PF01'` restricts the calculation to the specified company
    code.
-   The selected rows are stored in `GT_BSIK`.

### 5.4 Process G/L accounts and calculate balances

``` abap
LOOP AT gt_bsik INTO gs_bsik.

  CASE gs_bsik-hkont.

    WHEN '0000117000'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117003'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117004'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117005'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117006'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117007'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117008'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117009'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117010'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117001'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt1 =
          gs_final-bal_amt1 + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt1 =
          gs_final-bal_amt1 + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117021'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ENDIF.

    WHEN '0000117011'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr.
      ELSE.
        gs_final-bal_amt =
          gs_final-bal_amt + gs_bsik-dmbtr * -1.
      ENDIF.

    WHEN '0000117022'.
      IF gs_bsik-shkzg = 'H'.
        gs_final-bal_amt1 =
          gs_final-bal_amt1 + gs_bsik-dmbtr * -1.
      ELSE.
        gs_final-bal_amt1 =
          gs_final-bal_amt1 + gs_bsik-dmbtr.
      ENDIF.

  ENDCASE.

ENDLOOP.
```

**Explanation**

1.  The form loops through each BSIK record.
2.  `HKONT` identifies the G/L account.
3.  `CASE` determines which balance bucket receives the amount.
4.  `SHKZG` determines whether the amount is added or subtracted.
5.  Most accounts add credit (`H`) amounts as negative values and other
    amounts as positive values.
6.  Account `0000117011` follows the opposite sign rule in the supplied
    code.
7.  Accounts `0000117001` and `0000117022` accumulate into `BAL_AMT1`.
8.  The other listed accounts accumulate into `BAL_AMT`.

### 5.5 Calculate total PF balance

``` abap
MOVE-CORRESPONDING gs_t_hrmast TO gs_final.

gs_final-bal_tot =
  gs_final-bal_amt + gs_final-bal_amt1.

IF gs_final-bal_amt < 0
   OR gs_final-bal_amt1 < 0.

  APPEND gs_final TO gt_final.

ENDIF.
```

**Explanation**

-   Employee/vendor mapping fields are copied to the final work area.
-   `BAL_TOT` is calculated by adding the two balance buckets.
-   The final record is appended to `GT_FINAL` only if at least one
    balance bucket is negative.

The eligibility form then reverses the sign of `BAL_TOT` and uses the
result to calculate the 75% amount.

**Important implementation point:** `MOVE-CORRESPONDING` occurs after
the BSIK accumulation and may overwrite matching fields in `GS_FINAL`.
Confirm that `GS_T_HRMAST` does not contain fields that should retain
the calculated balance values. Also confirm the negative-balance append
condition against the intended accounting sign convention.

------------------------------------------------------------------------

## 6. Complete end-to-end process

``` text
Employee opens PF Loan ESS
        |
        v
I + D: Read available loan categories
        |
        v
I + P: Retrieve previous PF membership
        |
        v
Employee selects loan category
        |
        v
I + E: Run membership and withdrawal eligibility checks
        |
        v
Employee enters requested amount and submits
        |
        v
O + W: Receive application data
        |
        v
Run LOAN_CHECK again
        |
        v
Read loan category description
        |
        v
Is application number new?
       / \
     Yes  No
      |    |
Generate   Read existing main record
number     and prepare changes
      \    /
       v  v
Set payment date if status is 99
        |
        v
Save ZTA_ESS_PF_MAIN
        |
        v
Save ZTA_ESS_PF_TXN
        |
        v
Return status and message
        |
        v
Employee views pending (I + R)
or approved requests (I + H)
        |
        v
Approver retrieves requests (A + A/J/R)
```

------------------------------------------------------------------------

## 7. Important code-review points

These are observations from the supplied code, not changes silently
applied to it.

  --------------------------------------------------------------------------------
                           No. Area                  Observation
  ---------------------------- --------------------- -----------------------------
                             1 `LOAN_CHECK`          The final unconditional
                                                     `P_FLAG = 'S'` can overwrite
                                                     an earlier validation error.

                             2 Amount variables      The code uses
                                                     `GS_PF_ESS_MAIN-ZLOAN_AMT`,
                                                     `GS_PF_MAIN-ZLOAN_AMT` and
                                                     `LV_REQ_AMT`. Confirm the
                                                     intended source.

                             3 `GV_FLAG`             The flag is set to `E` in the
                                                     `I + E` branch, while amount
                                                     checks require
                                                     `GV_FLAG NE 'E'`.

                             4 Membership date       Check whether Infotype 0041
                                                     contains a valid `Z2` date
                                                     before calculating months.

                             5 Special Circumstances Pending statuses are
                                                     included, but the query
                                                     filters on payment date.
                                                     Pending applications may not
                                                     have a payment date.

                             6 Main/transaction save Two separate commits mean the
                                                     main save may succeed even if
                                                     the transaction save fails.

                             7 Approver join         Multiple transaction rows per
                                                     application can produce
                                                     multiple joined rows. Confirm
                                                     the required transaction
                                                     selection.

                             8 PA0001                `ENDDA = '99991231'` is a
                                                     specific selection
                                                     assumption; check validity
                                                     requirements.

                             9 `IM_APP_ID`           Declared but not used in the
                                                     supplied code.

                            10 `ET_PF_SUMMARY`       Declared but not populated in
                                                     the supplied code.

                            11 Input table           The `O + W` branch loops
                                                     through `ET_PF_LOAN` and
                                                     processes the final row.
                                                     Confirm that one row is
                                                     supplied.

                            12 Balance mapping       Confirm that `PERNR = LIFNR`
                                                     is the intended mapping
                                                     condition.

                            13 Balance work area     `MOVE-CORRESPONDING` after
                                                     balance calculation may
                                                     overwrite matching fields.

                            14 Status handling       Save messages are defined for
                                                     `01`, `02` and `99`; confirm
                                                     handling for other possible
                                                     statuses.
  --------------------------------------------------------------------------------

### Suggested correction for the final eligibility result

The final amount validation should follow this structure so that an
error is not overwritten:

``` abap
IF lv_req_amt IS INITIAL OR lv_req_amt <= 0.

  p_flag = 'E'.
  p_message = 'Please enter a valid amount'.

ELSEIF lv_elig_amt <= 0.

  p_flag = 'E'.
  p_message = 'Eligible member balance is insufficient'.

ELSEIF lv_req_amt < lv_min_amt.

  p_flag = 'E'.
  p_message = 'Requested amount is below the minimum withdrawal amount'.

ELSEIF lv_req_amt > lv_elig_amt.

  p_flag = 'E'.
  p_message = 'Requested amount exceeds the eligible member balance'.

ELSE.

  p_flag = 'S'.
  p_message = 'Employee is eligible for PF withdrawals'.

ENDIF.
```

This is a focused correction to the final result handling. It does not
resolve the separate global-flag and amount-source issues; those need to
be checked against the intended business rules.

------------------------------------------------------------------------

## 8. Quick revision notes

-   **Main table:** Stores the current PF loan application.
-   **Transaction table:** Stores status/change information for the
    application.
-   **Loan type table:** Stores category-specific eligibility rules.
-   **Membership history table:** Supplies previous-company PF service
    months.
-   **Infotype 0041:** Supplies the current-company date used by this
    implementation.
-   **Number range:** Generates a unique PF application number.
-   **BSIK:** Supplies vendor open items for PF balance calculation.
-   **`LOAN_CHECK`:** Validates membership, withdrawal count and
    requested amount.
-   **`BALANCE_AMOUNT`:** Calculates the balance from configured G/L
    accounts.
-   **`ET_PF_LOAN`:** Returns application/request data to the ESS or
    approver caller.

## 9. Final summary

`ZTA_ESSMSS_PF_LOAN` is the central function module for the supplied PF
loan ESS workflow. It supports category display, membership-history
retrieval, eligibility checks, application creation/update, employee
request history and approver request retrieval. Its eligibility logic
combines Infotype 0041 membership dates, previous membership history,
category configuration and a balance calculation based on BSIK vendor
open items. Its persistence logic saves application details in
`ZTA_ESS_PF_MAIN` and status/change information in `ZTA_ESS_PF_TXN`.
