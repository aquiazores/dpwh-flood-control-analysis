# DPWH Flood Control Projects — Cost Variance Analysis (2021–2024)

## Overview
I built this project to analyze cost variance patterns in DPWH flood control infrastructure projects across the Philippines. Specifically, I wanted to find out whether projects are consistently awarded near their Approved Budget for Contract (ABC), and which regions, project types, and contractors show the most variance between planned and actual cost.

As a licensed Civil Engineer and Quantity Surveyor, I have direct experience with ABC-setting and contractor claims — so this felt like a natural first project that combines my engineering background with my data analytics skills.

## Business Question
Are DPWH flood control contracts consistently awarded near their Approved Budget for Contract? And if not — where are the biggest gaps, and why?

## Tools Used
- Python (pandas) — data cleaning and exploratory data analysis
- Google BigQuery (SQL) — querying and aggregating the data
- Tableau Public — building the dashboard
- Google Colab — working environment

## Dataset
- Source: Kaggle — DPWH Flood Control Projects
- Raw rows: 9,855
- Cleaned rows: 9,698
- Coverage: Completed flood control projects from 2021–2024

Note: 2018–2020 and 2025 data were excluded from trend analysis because those years had fewer than 35 projects each — too small to draw reliable conclusions from.

## Data Cleaning
The raw dataset needed a few fixes before analysis:

- ApprovedBudgetForContract and ContractCost were stored as text instead of numbers — converted both to numeric format
- StartDate and ActualCompletionDate were stored as text — converted to proper datetime format
- 666 rows had missing Municipality values — filled with 'Unknown'
- 157 rows had non-numeric ABC values. After investigating, these turned out to be clustered projects (where multiple projects share one contract budget) and MYCA or Multi-Year Contracting Authority projects. I excluded them from the cost variance analysis but kept them in the dataset for project count and regional breakdowns.
- Created three new calculated columns: CostVariance, CostVariancePct, and ProjectDurationDays

## Understanding Cost Variance

Cost variance in this project is calculated as:

CostVariance = ContractCost - ApprovedBudgetForContract
CostVariancePct = (CostVariance / ApprovedBudgetForContract) x 100

A negative value means the contract was awarded below the ABC.
A positive value means it exceeded the ABC.

However, not all negative variance is good news:

- 0% to -3%: Normal and healthy. ABC is intentionally set conservatively and contractors bid competitively below it. This is expected under RA 9184 (Philippine Procurement Law).

- -3% to -10%: Worth monitoring. Could indicate the ABC was set too high, or the contractor may have underestimated scope — potentially leading to variations and change orders later.

- Below -10%: Red flag. Extreme underbidding at this level raises questions about scope reduction, data accuracy, or procurement compliance. This is the basis for the outlier analysis in this project.

This distinction is important when reading the regional and contractor findings — the goal isn't to find who spent the least, but to identify where variance patterns fall outside of what's considered normal in Philippine public procurement.

## What I Found

### The Big Picture
- 9,698 projects were analyzed after removing the 157 clustered/MYCA rows
- The national average cost variance is -1.47% — meaning projects are generally awarded slightly below their ABC
- 73.9% of projects came in under budget, 18.2% matched ABC exactly, and only 7.9% exceeded it
- This tells us that overspending is rare — the bigger concern is actually extreme underbidding in certain areas

### By Region
- Region III has the most projects at 1,617 — which makes sense given Central Luzon's major river systems like Pampanga and Agno
- Region V (Bicol) stands out the most — highest underspend at -4.45% and -₱3.43M average per project, despite being one of the most typhoon-prone regions in the country. This is partly driven by an extreme -97.52% outlier in Albay.
- Region I is the most conservative — 86.4% of projects come in under budget and only 0.8% exceed it
- Region VII has an unusually high rate of exact-budget awards at 37.9% — which could mean contractors are consistently pricing right at the ABC ceiling

### Procurement Trend Over Time
- Variance improved steadily from -4.49% in 2019 to -0.74% in 2024 — contracts are being awarded closer to ABC each year
- This is a positive sign for procurement efficiency, though it's worth noting that 2018–2020 had very few projects so those early numbers should be taken with a grain of salt

### Contractors
- Among the top 10 by project count, QM Builders is the most precise bidder at -0.05% average variance across 88 projects
- Centerways Construction stands out at -6.01% average variance — the highest among top contractors. They also appear multiple times in the extreme outlier list, which is worth flagging.
  
- Worth noting: Contractor performance here is based purely on cost variance. Project quality and completion rates are outside the scope of this dataset.

### By Project Type
- Construction of Flood Mitigation Structures dominates at 64.5% of all projects
- Rehabilitation and repair projects consistently show higher variance than new construction — probably because scope is harder to define upfront for rehab work
- Drainage projects show the highest variance by work type

### Outliers
- 235 projects (2.4% of total) were flagged as extreme outliers — variance below -10% or above +5%
- Almost all of them are extreme underspend cases. Only 1 project exceeded ABC by more than 5%.
- Region V accounts for a disproportionate share of these outliers, with several projects awarded at less than 15% of their ABC
- Centerways Construction appears in multiple outlier projects — consistent with their overall -6.01% average

## SQL Queries (Google BigQuery)

### Query 1 — Average Cost Variance by Region
```sql
-- Total projects and average variance per region
SELECT
  Region,
  COUNT(ProjectId) AS total_projects,
  ROUND(AVG(CostVariancePct), 2) AS avg_variance_pct,
  ROUND(AVG(CostVariance), 2) AS avg_variance_peso
FROM `sample-project-489102.dpwh_analysis.flood_control_projects`
GROUP BY Region
ORDER BY avg_variance_pct ASC
```

### Query 2 — Year Over Year Variance Trend
```sql
-- Year over year trend of cost variance across all regions
SELECT
  FundingYear,
  COUNT(ProjectId) AS total_projects,
  ROUND(AVG(CostVariancePct), 2) AS avg_variance_pct,
  ROUND(AVG(CostVariance), 2) AS avg_variance_peso
FROM `sample-project-489102.dpwh_analysis.flood_control_projects`
GROUP BY FundingYear
ORDER BY FundingYear ASC
```

### Query 3 — Top 10 Contractors by Project Count
```sql
-- Top 10 contractors by number of projects handled
SELECT
  Contractor,
  COUNT(ProjectId) AS total_projects,
  ROUND(AVG(CostVariancePct), 2) AS avg_variance_pct
FROM `sample-project-489102.dpwh_analysis.flood_control_projects`
GROUP BY Contractor
ORDER BY total_projects DESC
LIMIT 10
```

### Query 4 — Project Type Breakdown
```sql
-- Types of flood control work and their variance
SELECT
  TypeOfWork,
  COUNT(ProjectId) AS total_projects,
  ROUND(AVG(CostVariancePct), 2) AS avg_variance_pct,
  ROUND(AVG(CostVariance), 2) AS avg_variance_peso
FROM `sample-project-489102.dpwh_analysis.flood_control_projects`
GROUP BY TypeOfWork
ORDER BY total_projects DESC
```

### Query 5 — Over vs Under Budget Per Region
```sql
-- Budget status breakdown per region
SELECT
  Region,
  COUNT(ProjectId) AS total_projects,
  COUNT(CASE WHEN CostVariance > 0 THEN 1 END) AS over_budget,
  COUNT(CASE WHEN CostVariance < 0 THEN 1 END) AS under_budget,
  COUNT(CASE WHEN CostVariance = 0 THEN 1 END) AS exact_budget
FROM `sample-project-489102.dpwh_analysis.flood_control_projects`
GROUP BY Region
ORDER BY total_projects DESC
```

### Query 6 — Extreme Outliers
```sql
-- Projects with abnormally high or low cost variance
SELECT
  ProjectName,
  Region,
  Contractor,
  ApprovedBudgetForContract,
  ContractCost,
  ROUND(CostVariancePct, 2) AS variance_pct
FROM `sample-project-489102.dpwh_analysis.flood_control_projects`
WHERE CostVariancePct < -10
  OR CostVariancePct > 5
ORDER BY variance_pct ASC
```

## Recommendations
1. Region V and Region IX show consistently high underspend — DPWH may want to review how ABC is being set in these regions to make sure cost estimates reflect actual market rates
2. Centerways Construction appears repeatedly in the extreme outlier list and might be worth a closer look from a procurement compliance standpoint
3. Region VII's high exact-budget award rate is worth investigating — it could just be coincidence, or it could indicate contractors pricing strategically at the ceiling
4. Rehabilitation and drainage projects show the most variance — better scope definition upfront could help tighten these numbers

## Limitations
- This only covers flood control projects — I wouldn't generalize these findings to all DPWH infrastructure
- Cost variance doesn't tell you anything about project quality or whether it was actually completed properly
- 157 clustered and MYCA projects were excluded because their ABC values weren't available at the individual project level
- Early years (2018–2020) and 2025 don't have enough projects for reliable trend analysis

## Dashboard
Full interactive Tableau dashboard:
[DPWH Flood Control Cost Variance Analysis] 
https://public.tableau.com/views/DPWHFloodControlCostVarianceAnalysis/DPWHFloodControlAnalysis?:language=enUS&publish=yes&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link

## Files
- `dpwh_flood_control_projects.csv` — original raw data
- `dpwh_flood_control_cleaned.csv` — cleaned data used for analysis
- `DPWH_FloodControl_Cleaning.ipynb` — full Python notebook
