---

## 2026/02/18 Updates ##

---

### 18+ Patients ###
Creatinine and Lactate components were not being calculated for 18+ patients, consistent with Lurie's implementation. We now calculate them the same way as we would for the oldest age bin for both respective variables.

### D-Dimer ###
D-dimer is now being monitored using a new Cerner variable, D-Dimer Quantitative (as opposed to D-Dimer Titer) to accomodate 2025 changes to how this is now going into Cerner. This variable is being recorded in ng/mL FEU. So Cerner variable is being divided by 1000 before threshholding.

### Pupillary Response Variables ###
We now account the "Non-Reactive" value to determine whether pupils are fixed. Pupil Size Before Light must also be greater than 4mm because Roby does not want to mistaken small pupils with fixed ones.

### Infrastructure
* **Output Directory:** There is now only one output directory, which contains not only the phoenix sepsis score + constituents but also raw, scrubbed, and wide format files.
* **Code Structure:** Simplified writing out files by removing Onsen dependencies.

---

## 2025/10/08 Updates

---

### Cardiovascular Constituent
* ** Lactate:** PLM changed their lab systems. Lactate is now recorded in mmol/L instead of mg/dL, so we no longer need a conversion factor.

---

## 2025/09/15 Updates

---

### General Updates
This document reflects changes made following discussions with Nelson, Robi, and Justin on September 4 with Robi, Nelson, Paul, and Justin

### Respiratory Constituent
* **IMV/ORS:** Having an FiO2 of 21 will not exclude a child from being considered as being on respiratory support. This is because we believe that excluding CTICU kids will already protect us from very many cardio-based false positives.
* **Most Recent Value vs. Worst Value**: We now take the most recently available scoring components, rather than the worst.

### Cardiovascular Constituent
* **Vasopressin:** Vasopressin needs to occur concurrently with another vasoactive.
* **Extending lookback window:**  Nurses are going to retro-chart, rather than doing it in realtime. It's better to be overly sensitive about detecting whether a kid is still on a vasoactive drug, so we extended the lookback window from 2 to 6 hours.

### Infrastructure
* **File Format:** Output file now contains the Total score and Phoenix Sepsis subscore components. The latter is written out as a dictionary encoded as a json string which contains data elements used to calculate the subscore, as well as the time that they came from.

---

## 2025/08/20 Updates

---

### General Updates
This document reflects changes made following discussions with Nelson, Robi, and Justin on July 24, 2025 and with Justin on August 15, 2025.

---

### Respiratory Constituent
* **Type of Oxygen Administration:** The method for determining a patient's respiratory support has been changed. Previously, we used an internal numeric scale called **Oxygen Model Level** to encode the severity of respiratory support, but this was removed due to its complexity and conflation of different support modes. We now use direct string matching on the raw "Type of Oxygen Administration" event to identify respiratory support.
* **Any Respiratory Support (ARS):** The definition of **Other Respiratory Support (ORS)** has been corrected and renamed to Any Respiratory Support (ARS). The previous ORS definition, which incorrectly excluded mechanical ventilation, has been redefined to include it. ORS also considered the presence of variables like PEEP, PIP, and Tidal Volumes as respiratory support; this is no longer being done. **ARS is now defined as any "Type of Oxygen Administration" value other than "Discontinued" and an FiO2 > 21.**
* **IMV Definition:** Patients were previously considered to be on **Invasive Mechanical Ventilation (IMV)** if either "Type of Oxygen Administration" takes on a"Mechanical Ventilation" value or if an HFOV setting (amplitude or frequency) was recorded. The definition for **IMV** now ignores the presence of HFOV settings and has been redefined as **"Type of Oxygen Administration" taking on a "Mechanically Ventilated" value and an FiO2 > 21.**
* **FiO2 Forward-Filling:** FiO2 is now only forward-filled within the same "Type of Oxygen Administration" group. This prevents the assumption that FiO2 levels remain constant when a patient's respiratory support mode changes, a suggestion from Justin.
* **Scoring Bug Fix:** A bug was fixed where a score of 3 was incorrectly defined as an SpO2/FiO2 ratio of less than 148, or a PaO2/FiO2 ratio of less than to 100. This has been corrected to use a "less or equal to" comparison instead of "less than" to ensure continuity.

---

### Cardiovascular Constituent
* **Unit Conversion:** The unit for lactate has been corrected from mg/dL to mmol/L by multiplying the value by 0.111.
* **Systemic Vasoactives:** The method for identifying systemic vasoactive medications has been updated. We now use a proxy created by Justin for his screening tool: checking for the presence of the substring "kg/" in the Cerner `event tag` column for vasoactive drugs. This is used instead of using Cerner's order details table to look up the drug's route of administration.
* **Discontinued Vasoactive Drugs:** Unlike Lurie, we do not have have indicators that tell us when a child has been discontinued from a continuous infusion (eg. continuous infusion rate of 0). However, we can assume that our nurses chart continuous infusions at least every hour. If there has been no charting for 2 hours, we believe that it is safe to assume that the drug has been discontinued. Therefore, we now only count the number of systemic vasoactive medications that have been charted within the past 2 hours, rather than the 12 hours listed in Lurie's implementation document.
---

### Data Aggregation
* **Most Recent Value vs. Worst Value:** For most Phoenix Sepsis constituents, the calculation now uses the **most recently recorded value** within the observation window rather than the "worst" possible value. For example, we previously used the worst GCS and worst Pupillary Response State within 6 hours of the pull to calculate the neurologic component. We now only use the most recently recorded values within that 6 hour window. This change applies to the cardiovascular, coagulation, neurologic, endocrine, immunologic, renal, and hepatic components.
* **Respiratory Constituent Exception:** The respiratory component is the only exception to this change. It still uses the **lowest (worst)** combination of the O2 Saturation/FiO2 and Respiratory Support mode to align with [this section in Lurie's implementation document](https://cu-dbmi-peds.github.io/phoenix/articles/ehr_implementation_notes.html#scoring)

---

### Lookback Window Fixes
* **Glasgow Coma Score:** The lookback window for GCS has been corrected from 6 hours to 12 hours. 
* **Creatinine:** The lookback window for creatinine has been corrected from 6 hours to 24 hours. 

---

### Infrastructure
* **File Format:** All files are now saved in the **parquet format** to ensure backward compatibility.
* **Time Conversion:** The UTC to PST time conversion has been fixed to properly account for daylight saving time.