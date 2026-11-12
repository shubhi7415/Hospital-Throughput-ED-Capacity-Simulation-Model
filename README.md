# Hospital-Throughput-ED-Capacity-Simulation-Model
Analysed 8,000+ patient encounters across a 300-bed facility using Power BI/Excel queue modeling to diagnose capacity bottlenecks and inform operational recommendations.

# Hospital Throughput & ED Capacity Simulation

A queueing-theory analysis of **emergency department (ED) flow and inpatient bed capacity** in a simulated 300-bed hospital. The project identifies where patients get stuck, quantifies why, and tests operational fixes using Little's Law and Erlang-B / Erlang-C models.

It pairs Excel workbooks(Hospital_Throughput_Queue_Model.xlsx) with a Power BI-oriented management report, so the same findings are available as a live model, a quick-look dashboard, and a written report.

> **Simulated data.** Every dataset in this repository is synthetically generated. It is **not** real hospital or patient data, and the thresholds and recommendations must be validated against actual policy, staffing, acuity and patient-safety requirements before any operational use.

---

## Key findings

| Metric | Value |
|---|---|
| ED encounters (90-day window) | **8,125** |
| Admit rate | **24.6%** (2,002 admits) |
| Left without being seen (LWBS) | **2.15%** (175 patients) |
| Avg door-to-bed | **3.1 min** |
| Avg boarding time (admitted patients) | **95.6 min** |
| Avg total ED length of stay | **170.7 min** |

**ICU is the binding constraint.** ICU and Med-Surg run at nearly the same average occupancy, but ICU's small bed pool makes the same utilisation far more damaging:

| Unit | Beds | Avg utilisation | % of hours >90% occupied | Status |
|---|---|---|---|---|
| ED | 20 | 52.6% | 10.9% | Healthy |
| **ICU** | **32** | **84.2%** | **55.6%** | **Critical bottleneck** |
| **Med-Surg** | **164** | **83.8%** | **32.2%** | **Critical bottleneck** |
| Telemetry | 54 | 66.7% | 3.2% | Watch |
| Observation | 50 | 52.0% | 0.6% | Healthy |

- **Pooling effect:** at ~84% utilisation, the 32-bed ICU has an Erlang-C predicted wait of about **293 min** and simulated boarding of about **445 min**. The 164-bed Med-Surg unit has a predicted wait of about **2.7 min** and simulated boarding of about **21 min**.
- **Simulated boarding exceeds the Erlang-C prediction** for ICU because real length of stay is right-skewed (lognormal), whereas classic Erlang-C assumes exponential service times. High-variance stays inflate queues beyond the textbook figure.
- **Scenario, move 4 beds from Observation to ICU:** utilisation falls from 84.1% to 74.7%, and the Erlang-C predicted wait falls from about **293 min to about 41 min**.
- **ED surge window:** demand and LWBS risk concentrate between roughly **12:00 and 18:00**.

---

## Repository contents

```
.
├── README.md
├── models/
│   ├── Hospital_Throughput_Queue_Model.xlsx
│   └── ED_Capacity_Simulation_Dashboard.xlsx
└── reports/
    └── Hospital_Throughput_PowerBI_Report.docx
```

### 1. `Hospital_Throughput_Queue_Model.xlsx`: the full model

The main workbook. KPI cells are live formulas over the raw sheets, so everything recalculates if the source data changes.

| Sheet | Purpose |
|---|---|
| `Overview` | KPI cards, unit utilisation table with status flags, key finding |
| `Raw_Data` | 8,125 ED encounters: ESI level, chief complaint, arrival mode, door-to-bed, service time, boarding, ED LOS, disposition, admit unit, inpatient LOS |
| `Occupancy_Hourly` | Hourly census, capacity and utilisation for each unit (10,800 rows) |
| `Hourly_DOW_Patterns` | Arrival volume, door-to-bed, LWBS and ED LOS by hour of day and day of week |
| `Erlang_Helper` | Stable recursive Erlang-B engine, `B(n) = a·B(n-1) / (n + a·B(n-1))`, which avoids factorial overflow for large bed counts |
| `Unit_Bottleneck_Queue_Model` | Erlang-C theoretical wait vs simulated boarding for each receiving unit |
| `Daily_Trend` | 90-day trend of ED volume, wait times and unit occupancy |
| `Recommendations` | Five prioritised actions plus the Observation-to-ICU scenario |

### 2. `ED_Capacity_Simulation_Dashboard.xlsx`: ED front-door view

A lighter workbook focused on **ED bay utilisation by hour** using **Little's Law** (`L = λ × W`).

| Sheet | Purpose |
|---|---|
| `Raw Data` | 900-encounter sample (EncounterID, date, hour, ESI level, department, bed, length of stay) |
| `Hourly Summary` | Per-hour arrival rate λ, average length of stay W, patients present L, and bay utilisation, scaled from the sample to the full 8,000-encounter volume |
| `Dashboard` | Headline utilisation KPIs, charts and an **evening-surge what-if** |

Headline results: peak-hour utilisation of **96%** at **20:00**, a 24-hour average of **47.5%**, and a 6-11 pm window average of **77.4%**. Adding 3 flex bays and moving 4 nurses from overnight to evening takes peak utilisation from 96% to about 64%, an estimated 33% reduction in boarding time. This is a utilisation-proportional approximation for quick scenario planning, not a substitute for the full queue model.

> The two workbooks are independent simulations. The dashboard workbook models a 6-bay acute-care area over a 182-day window; the queue model covers 20 ED bays and 300 inpatient beds over 90 days. Their numbers should not be compared one-to-one.

### 3. `Hospital_Throughput_PowerBI_Report.docx`: management report

A written report covering the executive summary, KPI snapshot, capacity and queueing findings, ED demand pattern, a **four-page Power BI dashboard design**, core DAX measures and the recommendations.

**Planned Power BI pages**

1. Executive Overview: KPI cards, daily volume trend, unit alert table (85% warning / 90% high-pressure)
2. ED Flow & Demand: hourly arrivals, door-to-bed, LWBS, ED LOS by ESI; slicers for date, ESI, arrival mode and day
3. Capacity & Bottlenecks: occupancy trend, theoretical vs simulated boarding, P90 boarding, utilisation-vs-boarding scatter
4. Recommendations: action cards, Observation-to-ICU scenario, daily huddle trigger table

**Core DAX measures**

```DAX
Total Encounters      = COUNTROWS(Encounters)
Admission Rate        = DIVIDE(CALCULATE(COUNTROWS(Encounters), Encounters[disposition]="Admit"), [Total Encounters])
LWBS Rate             = DIVIDE(CALCULATE(COUNTROWS(Encounters), Encounters[disposition]="LWBS"), [Total Encounters])
Avg Door-to-Bed       = AVERAGE(Encounters[door_to_bed_min])
Avg Boarding - Admits = CALCULATE(AVERAGE(Encounters[boarding_min]), Encounters[disposition]="Admit")
Avg ED LOS            = AVERAGE(Encounters[ed_los_min])
Avg Occupancy         = AVERAGE(Occupancy_Hourly[utilization])
```

The report refers to a Power BI-ready file, `PowerBI_Hospital_Throughput_Model.xlsx`. That file is not part of this repository. To build the dashboard, import `Raw_Data` (as `Encounters`) and `Occupancy_Hourly` from the queue model, and add a Calendar table related on `Calendar[Date]`.

---

## Method

- **Little's Law:** `L = λ × W` gives the average number of patients present from arrival rate and length of stay.
- **Offered load:** `a = λ × mean LOS` (erlangs), using ED-sourced admits plus assumed direct and scheduled admissions per unit.
- **Erlang-B** is computed with the stable recursion above, then converted to **Erlang-C** to get the probability of waiting and the predicted wait for each unit.
- **Simulation vs theory:** simulated boarding is compared with the Erlang-C prediction to show the effect of non-exponential, right-skewed length of stay.

## Recommendations

1. **Add ICU surge or step-down capacity** (highest priority): 3-4 flexed beds, potentially drawn from spare Observation capacity.
2. **Reduce ICU length-of-stay variance** through standardised discharge-readiness rounds and early step-down criteria.
3. **Smooth Med-Surg elective and direct admissions** away from the daytime peak.
4. **Add ED surge capacity from 12:00 to 18:00** using swing bays and targeted staffing.
5. **Alert early on occupancy:** trigger discharge and transfer workflows at 85%, before units pass 90%, where boarding grows non-linearly.

## How to use

- Open the `.xlsx` files in Excel or Google Sheets. Recalculation is automatic; the `Erlang_Helper` and `Unit_Bottleneck_Queue_Model` sheets are the place to change bed counts and see predicted waits move.
- To explore in Python:

```python
import pandas as pd

xl = "models/Hospital_Throughput_Queue_Model.xlsx"
enc = pd.read_excel(xl, sheet_name="Raw_Data")
occ = pd.read_excel(xl, sheet_name="Occupancy_Hourly")

print(enc["disposition"].value_counts(normalize=True))
print(occ.groupby("unit")["utilization"].mean())
```

## Tech stack

Microsoft Excel (live formulas, charts), Little's Law and Erlang-B/C queueing theory, Power BI and DAX (design and measures), Python/pandas (optional exploration).

## Author

**Shubhi**: BDS, MBA (Healthcare Administration), IIHMR University. Portfolio project combining clinical healthcare knowledge with healthcare operations analytics.

## Disclaimer

Academic / portfolio project built on simulated data. Not affiliated with or endorsed by any hospital or organisation.

