# Exeter Diabetes codelists
CPRD Aurum codelists from the Exeter Diabetes team. All codelists are based on an October 2020 extract of CPRD Aurum; later versions may include extra medcodes/prodcodes not included here.

Our scripts which use these codelists and the below algorithms to define cohorts and pull in relevant biomarker/comorbidity/sociodemographic information from CPRD Aurum can be found in our [CPRD-Cohort-scripts repository](https://github.com/Exeter-Diabetes/CPRD-Cohort-scripts).

&nbsp;

## General notes on implementation
* Only codes with 'valid' dates used unless otherwise stated ('any date'). A valid date is an obsdate (for medcodes), issuedate (for prodcodes), epistart (for ICD10 codes) or evdate (for OPCS4 codes) which is no earlier than the patient's date of birth (no earlier than the month of birth if date of birth is not available; no earlier than full date of birth if this is available), no later than the patient's date of death (earliest of cprd_ddeath (Patient table) and dod/dor where dod not available (ONS death data)) where this is present, no later than deregistration where this is present, and no later than the last collection date from the Practice.
* Patient date of birth = 15th of month of birth (where month of birth and year of birth available), and 1st July where only year of birth available, or earliest medcode in Observation table if this is earlier (excluding medcodes before mob/yob)

&nbsp;

## Biomarker algorithms
* Values outside of acceptable limits removed (Biomarkers/biomarker_acceptable_limits; note this table is suitable for ADULT height, weight and BMI measurements only)
* Only values with acceptable units (numunitid) included (Biomarkers/biomarker_acceptable_units; includes missing numunitid for all biomarkers)
* If multiple biomarker values on the same day, the mean is used (including for HbA1c, after conversion to the same unit code)

&nbsp;

### ACR (Albumin Creatinine Ratio)
* Preferentially use coded ACR value
* If no coded value, where urine albumin and urine creatinine measurements recorded on same obsdate, calculate ACR

&nbsp;

### BMI
* Preferentially use coded BMI value
* If no coded value, where height and weight recorded on same obsdate, use these to calculate BMI (for adults only)

&nbsp;

### eGFR (estimated glomerular filtration rate)
* Use cleaned serum creatinine readings, age at reading and sex to calculate eGFR as per CKD-EPI Creatinine Equation 2021 (https://www.kidney.org/professionals/kdoqi/gfr_calculator/formula; ethnicity not used in this equation)

&nbsp;


### Haematocrit
* Convert all to proportion by dividing those >1 by 100

&nbsp;

### Haemoglobin
* Convert all to g/L (some in g/dL) by multiplying values <30 by 10

&nbsp;

### HbA1c
* Remove if before 1990 as HbA1c not widely used before then
* Convert all to mmol/mol by converting values <=20 (assume these are in % units)

&nbsp;

## Comorbidity algorithms
* Earliest (valid) medcode/ICD10 code/OPCS4 code date used as date of diagnosis

&nbsp;

### CKD (Chronic Kidney Disease) stage
* Use eGFR measurements calculated as above
* Convert eGFR measurements to CKD stage (1/2/3a/3b/4/5 as per https://www.kidney.org/atoz/content/gfr)
* Only keep CKD stages confirmed by multiple readings separated by >=90 days (without a measurement corresponding to a different stage within this 90 day period). This is to exclude readings corresponding to acute kidney injury (AKI).
* Remove any instances where patients appear to regress to a less severe CKD stage e.g. if they move from stage 4 to 3b, remove the 3b measurements
* Diagnosis date for CKD stage is the earliest eGFR measurement corresponding to this stage once values have been cleaned as above.
* CKD stage 5 (end stage renal disease [ESRD]) is defined using eGFR as above OR by the presence of a CKD5 medcode/ICD10/OPCS4 code

&nbsp;

## Diabetes algorithms

A flow diagram of the overall process of defining a mixed Type 1/Type 2 diabetes cohort can be found in our [CPRD-Cohort-scripts repository](https://github.com/Exeter-Diabetes/CPRD-Cohort-scripts#introduction).

### Defining a cohort of mixed Type 1 and Type 2 diabetes
Include participants with:
* At least one diabetes QOF (Quality and Outcomes Framework) medcode (Diabetes/exeter_medcodelist_qof_diabetes). The QOF codelist was constructed from Read codes from version 38 and SNOMED codes from version 44 of the QOF, which include all codes from previous versions. QOF codes for non-Type 1/non-Type 2 diabetes mellitus were not excluded, but these codes are also in the 'exclusion_diabetes' codelist, so people with these codes are removed (see next bullet point).
* No diabetes exclusion medcodes (codes for non-Type 1/non-Type 2 diabetes mellitus; Diabetes/exeter_medcodelist_exclusion_diabetes) with any date

&nbsp;

### Defining the diagnosis date of diabetes
Diagnosis date is earliest of:
* Any diabetes medcode (Diabetes/exeter_medcodelist_all_diabetes) except those with obstypeid=4 (family history), or those in year of birth for those with Type 2 diabetes (see Type 1/Type 2 classification algorithm below)
* A prescription for glucose lowering medication including insulin (Diabetes medications/exeter_prodcodelist_ohas and Diabetes medications/exeter_prodcodelist_insulin)
* Any HbA1c (Biomarkers/exeter_medcodelist_hba1c) >=47.5 mmol/mol (values <=20 assumed to be in % units and converted to mmol/mol)

Type 2 patients (see Type 1/Type 2 classification algorithm below) with a prescription for a glucose-lowering medication or HbA1c >=47.5 mmol/mol in their year of birth, or with no diabetes medcodes/prescriptions for glucose-lowering medications/HbA1cs >=47.5 mmol/mol after the year of birth are excluded from further analysis as presumably there are coding errors.

&nbsp;

### Defining Type 1 vs Type 2 in a mixed Type 1/Type 2 cohort
* No insulin prescriptions: Type 2
* With at least one insulin prescription:
  * With multiple Type 1- and/or Type 2-specific medcodes with any date (exeter_medcodelist_all_diabetes, category="type 1" or "type 2"):
    * If number of Type 1 medcodes >=2 x number of Type 2 medcodes, Type 1, otherwise Type 2
  * With one Type 1- or Type 2-specific medcode with any date:
    * Patient is categorised according to this medcode
  * With no one Type 1- or Type 2-specific medcodes:
    * If time between diagnosis date and start of insulin treatment is available (i.e. diagnosis date >= start of registration):
      * If diagnosed <35 years of age and on insulin within 1 year of diagnosis, Type 1, otherwise Type 2
    * If time between diagnosis date and start of insulin treatment is not available (i.e. diagnosis date < start of registration):
      * If diagnosed <35 years and not currently taking a non-insulin glucose lowering medication (no prescription for a non-insulin glucose lowering medication within 6 months of end of records (earliest of death/deregistration/last collection date from Practice)), Type 1, otherwise Type 2
      
Since diabetes type is used to inform whether diabetes medcodes in year of birth should be excluded or not (see above 'Defining the diagnosis date of diabetes'), this can create a circular problem in those with no Type 1 or Type 2-specific codes, where someone is classified as Type 1 if diabetes medcodes in year of birth are included, and Type 2 if diabetes medcodes in year of birth are excluded (or vice versa, if time to insulin from diagnosis is affected). These people are classed as 'unclassified', and need further investigation to establish diabetes type and date/age of diagnosis (we have excluded from our general Type 1/Type 2 cohort).

See our paper https://linkinghub.elsevier.com/retrieve/pii/S0895-4356(22)00272-4 for how this compares to other methods for classifying Type 1 and Type 2 diabetes.
      
&nbsp;

## Sociodemographics algorithms

### Ethnicity
* Three different ethnicity variables are produced: 5-category, 16-category, and QRISK2-category. Each is determined separately by:
 * Taking the most commonly recorded ethnicity category, excluding all 'unknown' ethnicity codes
 * If more than one most commonly recorded ethnicity category, use the most recently recorded. If multiple different ethnicities recorded on the most recent date, categorise as unknown
* For each person, if the 16-category or QRISK2-category category conflicts with the 5-category ethnicity\*, the 16-/QRISK2-category ethnicity is set to unknown
* For any unknown ethnicities, secondary care [HES] ethnicity is used where available

\*Conflicts:
* If 5-category ethnicity is 'White' and
 * 16-category ethnicity is 'Indian', 'Pakistani', 'Bangladeshi', 'Other Asian', 'Black Caribbean', 'Black African', 'Other Black', or 'Chinese'
 * QRISK2-category ethnicity is 'Indian', 'Pakistani', 'Bangladeshi', 'Other Asian', 'Black Caribbean', 'Black African', or 'Chinese'

* If 5-category ethnicity is 'South Asian' and
 * 16-category ethnicity is 'White British', 'White Irish', 'Other White', 'White and Black Caribbean', 'White and Black African', 'Black Caribbean', 'Black African', 'Other Black', or 'Chinese'
 * QRISK2-category ethnicity is 'White', 'Black Caribbean', 'Black African', or 'Chinese'

* If 5-category ethnicity is 'Black' and
 * 16-category ethnicity is 'White British', 'White Irish', 'Other White', 'White and Asian', 'Indian', 'Pakistani', 'Bangladeshi', 'Other Asian', or 'Chinese'
 * QRISK2-category ethnicity is 'White', 'Indian', 'Pakistani', 'Bangladeshi', 'Other Asian', or 'Chinese'


&nbsp;

### Smoking
* Take most recently recorded smoking category (non-smoker, ex-smoker, active smoker); if both non-smoker and ex-smoker recorded then choose ex-smoker
* If most recent is 'non-smoker' but have previously been recorded as active smoker, then categorise as 'ex-smoker'
* Look at the next most recent date if conflicting categories (ex-smoker and active smoker or non-smoker and active smoker) are recorded on the most recent date
* (NB: different algorithm used for smoking variable used in QRISK2/QDiabetes-Heart Failure: use most recent medcode within last 5 years and QRISK2 category associated with that code (0=non-smoker, 1=ex-smoker, 2=light smoker [1-10 cig/day], 3=moderate smoker [11-19 cig/day], 4=heavy smoker [20+ cig/day]). If both non-smoker and ex-smoker are recorded then choose ex-smoker. However, if medcode has a value associated with it and a valid numunitid (see 'smoking' in biomarker_acceptable_units.txt file in Biomarker directory), assume value is number of cigarettes per day, and use to re-assign those in category 0, 2, 3, and 4 (assume values associated with ex-smoking codes are cig/day that person used to smoke). Ignore values associated with medcode '1780396011' (Cigarette pack/years). If conflicting categories (0 and 2/3/4 or 1 and 2/3/4) recorded on the same day, use minimum.)

&nbsp;

### Alcohol consumption
* Take most recently recorded alcohol consumption level (0 = none, 1 = within limits, 2 = excess, 3 = harmful)
* If ever coded with a level 3 code then categorise as level 3
* If multiple levels coded on most recent date, take the highest level

&nbsp;

### Townsend Deprivation Scores
* 2015 Index of Multiple Deprivation (IMD) deciles were converted to 2011 Townsend Deprivation Scores (TDS) by matching on 2011 Lower Super Output Area (LSOA) and using the median TDS score (to 3 d.p.) of LSOAs with the same IMD decile of the patient
* 2015 IMD decile-2011 LSOA lookup from here: https://www.gov.uk/government/statistics/english-indices-of-deprivation-2015
* 2011 TDS-2011 LSOA lookup from here: https://statistics.ukdataservice.ac.uk/dataset/2011-uk-townsend-deprivation-scores/resource/de580af3-6a9f-4795-b651-449ae16ac2be

&nbsp;

## ICD10 and OPCS4 codelists
* 3 digit ICD10 and OPCS4 codes should be used to find matching 3 digit and 4 digit codes (i.e. those which start with the matching 3 digits)
* ICD10 and OPCS4 codes do not contain lowercase letters, so case sensitive matching is not required
* Episode start date used as date of diagnosis
