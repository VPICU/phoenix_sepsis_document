# Realtime Phoenix Sepsis 8 
This is a realtime pipeline to calculate the Phoenix Sepsis-8 Score at every hour.
Phoenix Sepsis-8 is a prospectively validated and calibrated mortality model used to define sepsis and septic shock for kids who are suspected of having an infection in the ICU. The score is calculated by adding up components from 8 different organ systems: respiratory, cardiovascular, coagulation, neurologic, endocrine, immunologic, renal, and hepatic. The scores from all components are added up to compute a final score; a higher score means a higher acuity. Sepsis is defined as having a score greater than 2. Having a score of at least 1 in the cardiovascular component means that the patient is going through septic shock.

This implementation was heavily based on these [Github](https://cu-dbmi-peds.github.io/phoenix/articles/ehr_implementation_notes.html) notes from Lurie and [Supplementary Material 9 of the JAMA paper](https://docs.google.com/spreadsheets/d/1dzkRYRq-ehlqXpVw0DeMecpYPWbqJULDxmiNrc12rBs/edit?usp=sharing).

Here is how each organ system component is calculated.

## Respiratory
(3 possible points)

### Overview
Calculated based on the lowest available PFR and SFR with consideration for Invasive Mechanical Ventilation (IMV) and Any Respiratory Support (ARS) within the past 6 hours.

* 0 points if P:F ratio >= 400 and S:F ratio >= 292
* 1 point if P:F ratio < 400 on any respiratory support or S:F ratio < 292 on any respiratory support
* 2 points if P:F ratio between 101 and 200 + IMV or S:F ratio between 149 and 220 + IMV
* 3 points if P:F ratio $$\leq$$ 100 + IMV or S:F ratio $$\leq$$ 148 + IMV

### Defining IMV/ORS
* **IMV**: A Cerner event "Type of Oxygen Administration" was charted with a value of "With Ventilator"
* **ARS (Any Respiratory Support)**: "Type of Oxygem Administration" contained any value that was not "Discontinued." This includes the following values: Continuous Mist, Discontinued, High Flow Nasal Cannula, Trach Collar, Hood, Non-Rebreather, Bi-Pap, Blow-by, Partial Rebreather, Simple Mask, T-Tube/Piece, Trach Collar, Venturi Mask, With Bi-Level, With CPAP, With Ventilator

### Calculating Oxygenation Level
* **S/F Ratio**: SpO2/FiO2 (_cannot be calculated if SpO2 > 97_)
* **P/F Ratio**: ABG PO2/FiO2

FiO2 was forward-filled within the same "Type of Oxygen Administration" group. SpO2, PaO2 were forward-filled for up to 6 hours.

## Cardiovascular
(6 possible points)

### Overview 
Composed of 3 components: Presence of Systemic Vasoactives, Lactate, and Age-Based Mean Arterial Pressure

### Presence of Vasoactives
(2 points)

Based on the # of Vasoactivate meds received through an IV within the past 6h
* 0 - No vasoactive meds
* 1 - 1 vasoactive med
* 2 - 2 vasoactive meds

The following vasoactive medications were considered: Dobutamine, Dopamine, Epinephrine, Milrinone, Norepinephrine, Vasopressin.

IV route administration was determined by checking the presence of the substring "kg/" in the Cerner `event tag` column for vasoactive drugs.

### Lactate
(2 points)
We used the most recent lactate from the past 6 hours

* 0 - Lactate <5 mmol/L
* 1 - Lactate 5-11 mmol/L
* 2 - Lactate >=11 mmol/L


### MAP
(2 points)

We used the most recent MAP from the past 6 hours

|          | 0 pts | 1 pt  | 2pts |
|----------|-------|-------|------|
| 0 - <1m  | >30   | 17-30 | <17  |
| 1 - <1y  | >38   | 25-38 | <25  |
| 1 - 2y   | >43   | 31-43 | <31  |
| 2 - 5y   | >44   | 32-44 | <32  |
| 5 - 12y  | >48   | 36-48 | <36  |
| 12 - 17y | >51   | 38-51 | <38  |

**Which MAP Value to Use**: MAP can be taken from 4 different sources. When it is available at the same time from multiple different sources, we apply the following heiararchy to determine which one to use:
1. MAP from Arterial Line
2. MAP derived from Arterial Line SBP and DBP*
3. MAP from cuff
4. MAP derived from cuff SBP and DBP *

\* MAP = 2/3(DBP) + 1/3(SBP). Only calculate if DBP <= SBP. I did not use forward-filled blood pressures.

## Coagulation
(2 points)

We used the most recent value of each lab from the past 24h
1pt for each (up to 2 pts)
* Platelets <100 K/uL 
* INR > 1.3
* D-Dimer > 2 mg/L FEU
* Fibrinogen < 100 mg/dL

## Neurologic
(2 points)
We used the most recent GCS assessment within the past 12 hours and Pupillary Reaction assessment within the past 6 hours.

* 0 - GCS > 10
* 1 - GCS <= 10
* 2 - bilaterally fixed pupils*

\* Bilaterally fixed pupils are defined as the Cerner events, "Right Pupillary Response Level" and "Left Pupillary Response Level" both containing the value, "Fixed." 


## Endocrine
(1 point)
We used the most recent Glucose value from the past 12 hours

* 1 - Glucose < 50 or >50

## Immunologic
(1 point)
We used the most recent ANC/ALC values from the past 24 hours

* 1 - ANC < 0.5 or ALC < 1

## Renal
(1 point)

We used the most recent Creatinine value from the past 24 hours

|   Age    |  1 pt  |
|----------|--------|
| 0 - <1m  | >= 0.8 |
| 1 - <1y  | >= 0.3 |
| 1 - 2y   | >= 0.4 |
| 2 - 5y   | >= 0.6 |
| 5 - 12y  | >= 0.7 |
| 12 - 17y | >= 1.0 |

## Hepatic
(1 point)

* 1 - Total Bilirubin >= 4mg/dL or ALT > 102 IU/L

