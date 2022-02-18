# Exeter Diabetes codelists
CPRD Aurum codelists from the Exeter Diabetes team

&nbsp;

## Diabetes algorithms
Only codes with 'valid' dates used unless otherwise stated ('any date'). A valid date is an obsdate (for medcodes) or issuedate (for prodcodes) which is no earlier than the patient's date of birth (no earlier than the month of birth if date of birth is not available; no earlier than full date of birth if this is available), no later than the patient's date of death (earliest of cprd_ddeath (Patient table) and dod/dor where dod not available (ONS death data)) where this is present, no later than deregistration where this is present, and no later than the last collection date from the Practice. In addition, biomarker test before 1990 removed as most not developed by then.

&nbsp;

### Defining a cohort of mixed Type 1 and Type 2 diabetes
Include participants with:
* 'acceptable' patient flag (Patient table) = 1
* At least one diabetes QOF medcode (Diabetes/exeter_medcodelist_qof_diabetes)
* No diabetes exclusion medcodes (codes for non-Type 1/non-Type 2 diabetes mellitus; Diabetes/exeter_medcodelist_exclusion_diabetes) with any date

&nbsp;

### Defining the diagnosis date of diabetes
Diagnosis date is earliest of:
* Any diabetes medcode (Diabetes/exeter_medcodelist_all_diabetes) except those with obstypeid=4 (family history)
* A prescription for glucose lowering medication including insulin (Diabetes medications/exeter_prodcodelist_ohas and Diabetes medications/exeter_prodcodelist_insulin)
* Any HbA1c (Biomarkers/exeter_medcodelist_hba1c) >=47.5 mmol/mol (if multiple values on one day, the mean is used; values <=20 assumed to be in % units and converted to mmol/mol)
If this date is within 90 days of the start of registration, then it may have occurred prior to registration.
Patient date of birth (needed to calculate age at diagnosis) = 15th of month of birth (where month of birth and year of birth available), and 1st July where only year of birth available.

&nbsp;

### Defining Type 1 vs Type 2
* No insulin prescriptions: Type 2
* With at least one insulin prescription:
  * With multiple Type 1- and/or Type 2-specific medcodes with any date (exeter_medcodelist_all_diabetes, category="type 1" or "type 2"):
    * If number of Type 1 medcodes >=2 x number of Type 2 medcodes, Type 1, otherwise Type 2
  * With one Type 1- or Type 2-specific medcode with any date:
    * Patient is categorise according to this medcode
  * With no one Type 1- or Type 2-specific medcodes:
    * If time between diagnosis date and start of insulin treatment is available (i.e. diagnosis date >= start of registration):
      * If diagnosed <35 years of age and on insulin within 1 year of diagnosis, Type 1, otherwise Type 2
    * If time between diagnosis date and start of insulin treatment is not available (i.e. diagnosis date < start of registration):
      * If diagnosed <35 years and not currently taking a non-insulin glucose lowering medication (no prescription for a non-insulin glucose lowering medication within 6 months of end of records (earliest of death/deregistration/last collection date from Practice)), Type 1, otherwise Type 2
