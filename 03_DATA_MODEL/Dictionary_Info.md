# 4Ps Intervention Program - Analytical Data Model

This section documents all final analytical tables prepared for Tableau. Each sheet corresponds to one **fact** or **dimension** table. The purpose of this dictionary is to clarify column meanings, data types, and business rules so downstream users can interpret fields uniformly across dashboards. Data types remain generic (`string`, `integer`, `float`, `date`). Notes clarify coded values, survey cycles, and percentage handling.

---

## Star Schema / ERD

- **Fact Tables**: Contain performance metrics and indicators for each program area.  
- **Dimension Tables**: Standardized descriptive attributes for aggregation and joins.  
- **Join Keys**: `region` or `RegionCode` used across fact tables.

```mermaid
erDiagram
    %% ========= DIMENSIONS =========
    dim_region_codes {
        string RegionCode PK
        string region
        int region_order
    }

    dim_island_group {
        string region PK
        string island PK
    }

    %% ========= FACT TABLES =========
    fact_poverty_incidence {
        string Region FK
        int Year FK
        float poverty_incidence_percent
        int region_order
    }

    fact_ave_income_per_region {
        string Region
        string Island_Group
        int Year
        float average_annual_family_income_thousands
        int d_region_order
    }

    fact_4ps_beneficiaries {
        string region FK
        string quarter FK
        int year FK
        string island FK
        int target
        int actual
    }

    fact_fds_attendance {
        string region FK
        int year FK
        string quarter FK
        string activity
        int target_attendance
        int total_actual_attendees
        float compliance_rate
    }

    fact_net_enrollment_rate {
        string Region FK
        string Island_Group FK
        int Year FK
        float Kindergarten_pct
        float Grades1to6_pct
        float Junior_High_School_pct
        float Senior_High_School_pct
    }

    fact_prenatal_postnatal_2022 {
        string region
        string island_group
        int survey_year
        int total_respondents
        int total_pregnant
        int total_pregnancy_history
        int respondents_with_anc_visits
        int total_postpartum_checked
        int total_baby_postnatal_checked
    }

    fact_immunization_2022 {
        string region FK
        string island FK
        int year FK
        int children_under5_count
        int health_card_count
        int hep_b_count
        int pentavalent_1_count
        int pentavalent_2_count
        int pentavalent_3_count
        int pneumoc_1_count
        int pneumoc_2_count
        int pneumoc_3_count
        int measles_2_count
    }

    fact_child_labor {
        string region_name FK
        int year FK
        string island_group FK
        int total_children
        int working_children
        float prop_working
    }

    %% ========= RELATIONSHIPS =========
    dim_region_codes ||--o{ fact_poverty_incidence : "region"
    dim_region_codes ||--o{ fact_ave_income_per_region : "region"
    dim_region_codes ||--o{ fact_4ps_beneficiaries : "region"
    dim_region_codes ||--o{ fact_fds_attendance : "region"
    dim_region_codes ||--o{ fact_net_enrollment_rate : "region"
    dim_region_codes ||--o{ fact_prenatal_postnatal_2022 : "region"
    dim_region_codes ||--o{ fact_immunization_2022 : "region"
    dim_region_codes ||--o{ fact_child_labor : "region"

    dim_island_group ||--o{ fact_ave_income_per_region : "island_group"
    dim_island_group ||--o{ fact_4ps_beneficiaries : "island"
    dim_island_group ||--o{ fact_net_enrollment_rate : "Island_Group"
    dim_island_group ||--o{ fact_prenatal_postnatal_2022 : "island_group"
    dim_island_group ||--o{ fact_immunization_2022 : "island"
    dim_island_group ||--o{ fact_child_labor : "island_group"

classDef dim fill:#2f2f2f,stroke:#ffffff,color:#ffffff
classDef poverty fill:#0b3d91,stroke:#062355,color:#ffffff       %% deep blue
classDef income fill:#008080,stroke:#004c4c,color:#ffffff        %% teal
classDef education fill:#28a745,stroke:#1c6b2c,color:#ffffff     %% bright green
classDef health fill:#e94e1b,stroke:#a33612,color:#ffffff        %% vivid orange-red
classDef labor fill:#6f42c1,stroke:#462d7e,color:#ffffff         %% strong purple
classDef comm_pat fill:#d4af37,stroke:#8c7a16,color:#000000      %% warm golden yellow (black text)

class dim_region_codes dim
class dim_island_group dim
class fact_fds_attendance comm_pat
class fact_net_enrollment_rate education
class fact_prenatal_postnatal_2022 health
class fact_immunization_2022 health
class fact_child_labor labor
```

---

## Fact Tables

### **A. Health & Nutrition**  

**Sheet: fact_prenatal_postnatal_2022**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region | string | Administrative region label | Matches PSA naming (I, II, CAR, etc.). Join key to dim_region_codes. |
| island_group | string | Island grouping of region | Value derived from dim_island_group; used for geographic aggregation. |
| survey_year | integer | Survey reference year | Year as reported in PDF/CSV source; no transformation. |
| total_respondents | integer | Total respondents in maternal survey | Raw count; includes both pregnant and non-pregnant respondents. |
| total_pregnant | integer | Number of currently pregnant respondents | Unchanged from raw source; may exceed total respondents due to multi-year history reporting. |
| total_pregnancy_history | integer | Count of respondents with past pregnancy history | Raw count; includes multiple pregnancy cycles. |
| total_with_anc_visits | integer | Number of respondents with at least one antenatal care visit | Represents ANC service utilization. |
| total_postpartum_checked | integer | Number of postpartum mothers checked | Service coverage indicator; raw count. |
| total_baby_postnatal_checked | integer | Number of babies receiving postnatal check | Reflects child postnatal service coverage. |

**Sheet: fact_immunization_2022**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region | string | Administrative region | Matches PSA region code naming. |
| island | string | Island grouping | Synonymous with island_group; from dimension table. |
| year | integer | Survey year | Year from PSA immunization dataset. |
| children_under5_count | integer | Total children under 5 surveyed | Raw PSA value. |
| healthcard_count | integer | Children with health card | PSA standard immunization indicator. |
| hepB_count | integer | Hepatitis B first dose count | Value corresponds to PSA vaccine_code “HepB”. |
| pentavalent1_count | integer | Pentavalent vaccine dose 1 | Raw count. |
| pentavalent2_count | integer | Pentavalent vaccine dose 2 | Raw count. |
| pentavalent3_count | integer | Pentavalent vaccine dose 3 | Raw count. |
| pneu1_count | integer | Pneumococcal vaccine dose 1 | Raw count. |
| pneu2_count | integer | Pneumococcal vaccine dose 2 | Raw count. |
| pneu3_count | integer | Pneumococcal vaccine dose 3 | Raw count. |
| measles2_count | integer | Measles vaccine second dose | Raw count extracted from PSA CSV. |

---

### **B. Education**

**Sheet: fact_net_enrollment_rate**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region | string | Administrative region | Used for joins; aligns with PSA/DepEd coding. |
| island_group | string | Island grouping | Pulled from dim_island_group. |
| year | string | School year identifier | Formatted as “2019-2020”. No conversion to integer. |
| kindergarten_pct | float | Net enrollment rate for Kindergarten | Already expressed in percent; retain original precision. |
| grades1_6_pct | float | Net enrollment rate for Grades 1–6 | Value sourced from DepEd PDF extraction. |
| jhs_pct | float | Net enrollment rate for Junior High School | Raw percentage extracted via Tabula. |
| shs_pct | float | Net enrollment rate for Senior High School | Raw percentage extracted via Tabula. |

---

### **C. Child Labor**

**Sheet: fact_child_labor**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region_name | string | Administrative region | Join key for dim_region_codes. |
| year | integer | Reference year | Raw PSA survey year. |
| island_group | string | Island grouping | Derived from dim_island_group. |
| total_children | integer | Total number of children in region | Raw PSA value. |
| working_children | integer | Number of working children | Raw PSA value; includes hazardous and non-hazardous work. |
| prop_working | float | Proportion of working children (%) | Already expressed as percentage; do not convert. |

---

### **D. Community Participation**

**Sheet: fact_fds_attendance**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region | string | Administrative region | Join key to dim_region_codes; follows DSWD regional classification. |
| activity | string | Program or activity name | In this dataset, always “FDS” (Family Development Session). |
| year | integer | Reporting year | Year from DSWD Quarterly Implementation Report. |
| quarter | string | Reporting quarter | Q1, Q2, Q3, Q4 format. |
| target_attendance | integer | Target number of attendees | Raw target set by DSWD for the FDS session. |
| total_actual_attendees | integer | Actual attendees | Raw performance output; number of beneficiaries who attended. |
| compliance_rate | float | Compliance rate (%) | Calculated as `(total_actual_attendees / target_attendance) * 100`; already expressed as a percentage. |

---

### **Other Fact Tables**

**Sheet: fact_4ps_beneficiaries**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region | string | Administrative region | Follows DSWD regional classification. |
| quarter | string | Quarter of reporting | Q1, Q2, Q3, Q4 format. |
| year | integer | Reporting year | Year from DSWD Implementation Report. |
| target | integer | Target number of 4Ps beneficiaries | Raw target set by DSWD. |
| actual | integer | Actual number of beneficiaries served | Raw performance output. |
| island | string | Island grouping | Derived from dim_island_group. |

**Sheet: fact_poverty_incidence**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region | string | Administrative region | Join to dim_region_codes. |
| poverty_incidence | float | Poverty incidence among families (%) | Already expressed as percent in original PSA dataset; do not re-scale. |
| year | integer | Reference year | Year directly taken from PSA publication. |

**Sheet: fact_ave_income_per_region**

| Column | Data Type | Description | Business Rule / Notes |
|--------|-----------|-------------|---------------------|
| region | string | Administrative region | Joins to dim_region_codes. |
| island_group | string | Island grouping | Derived from dim_island_group. |
| average_annual_income | float | Average annual family income (₱ thousands) | Values already in thousands; no scaling applied. |
| year | integer | Reference year | Taken from PSA Family Income and Expenditure Survey. |

---

## Dimension Tables

**Sheet: dim_region_codes**

| Column | Data Type | Description | Notes |
|--------|-----------|-------------|------|
| RegionCode | integer | Numeric region code | Surrogate join key. |
| region | string | Region label | Used across all fact tables. |
| region_order | integer | Ordering value for consistent visualization | Follows PSA official sequencing. |

**Sheet: dim_island_group**

| Column | Data Type | Description | Notes |
|--------|-----------|-------------|------|
| region | string | Administrative region | Join to region in fact tables. |
| island | string | Island grouping name | Luzon, Visayas, Mindanao. |







