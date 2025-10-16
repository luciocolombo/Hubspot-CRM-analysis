# SQL Queries and Insights — HubSpot CRM Dataset

We focus on pure information querying here. Visualization and interpretation will happen in the dashboard.

## 1. Volume: How many deals are created per period and how does volume trend?

**Deals created per month**

```sql
SELECT
  FORMAT_DATE('%Y-%m', deal_create_date) AS month_created,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY month_created
ORDER BY month_created;
```

| month_created | n_deals |
| ------------- | ------: |
| 2024-01       |       1 |
| 2024-02       |       4 |
| 2024-03       |       2 |

**Cumulative deals created by month**

```sql
WITH monthly AS (
  SELECT
    FORMAT_DATE('%Y-%m', deal_create_date) AS month_created,
    COUNT(*) AS n_deals
  FROM `hubspot-crm-analysis.export.users`
  GROUP BY month_created
)
SELECT
  month_created,
  n_deals,
  SUM(n_deals) OVER (ORDER BY month_created) AS cumulated
FROM monthly
ORDER BY month_created;
```

| month_created | n_deals | cumulated |
| ------------- | ------: | --------: |
| 2024-01       |       1 |         1 |
| 2024-02       |       4 |         5 |
| 2024-03       |       2 |         7 |

---

## 2. Source Mix: What is the distribution of deals by lead source?

**Deals by contact_lead_source**

```sql
SELECT
  contact_lead_source,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY contact_lead_source
ORDER BY n_deals DESC;
```

| contact_lead_source | n_deals |
| ------------------- | ------: |
| Email Campaign      |       9 |
| Organic Search      |       9 |
| Paid Ads            |       6 |

**Share of deals by source over time**

```sql
SELECT
  FORMAT_DATE('%Y-%m', deal_create_date) AS month,
  contact_lead_source,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY month, contact_lead_source
ORDER BY month, contact_lead_source;
```

| month   | contact_lead_source | n_deals |
| ------- | ------------------- | ------: |
| 2024-10 | Email Campaign      |       1 |
| 2024-10 | Organic Search      |       1 |
| 2024-11 | Email Campaign      |       2 |

---

## 3. Pipeline Mix: How are deals distributed across pipelines?

**Deals by deal_pipeline**

```sql
SELECT
  deal_pipeline,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY deal_pipeline
ORDER BY n_deals DESC;
```

| deal_pipeline   | n_deals |
| --------------- | ------: |
| Sales Pipeline  |      23 |
| Agency Pipeline |      17 |

**Pipeline mix over time**

```sql
SELECT
  FORMAT_DATE('%Y-%m', deal_create_date) AS month,
  deal_pipeline,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY month, deal_pipeline
ORDER BY month, deal_pipeline;
```

| month   | deal_pipeline   | n_deals |
| ------- | --------------- | ------: |
| 2024-10 | Sales Pipeline  |       2 |
| 2024-10 | Agency Pipeline |       1 |

---

## 4. Stage Snapshot: What is the current distribution of deals by stage?

**Deals by deal_stage per pipeline**

```sql
SELECT
  deal_pipeline,
  deal_stage,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
WHERE deal_stage IS NOT NULL
GROUP BY deal_pipeline, deal_stage
ORDER BY deal_pipeline, n_deals DESC;
```

| deal_pipeline   | deal_stage             | n_deals |
| --------------- | ---------------------- | ------: |
| Sales Pipeline  | Presentation Scheduled |       7 |
| Sales Pipeline  | Contract Sent          |       5 |
| Agency Pipeline | Contract Sent          |       4 |

**Open vs closed categorization**

```sql
SELECT
  deal_pipeline,
  CASE WHEN REGEXP_CONTAINS(deal_stage, r'(?i)Closed') THEN 'Closed' ELSE 'Open' END AS stage,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
WHERE deal_stage IS NOT NULL
GROUP BY deal_pipeline, stage
ORDER BY deal_pipeline, stage;
```

| deal_pipeline   | stage  | n_deals |
| --------------- | ------ | ------: |
| Agency Pipeline | Closed |       4 |
| Agency Pipeline | Open   |      13 |
| Sales Pipeline  | Closed |       1 |
| Sales Pipeline  | Open   |      22 |

---

## 5. Deal Value: What is total and average deal amount over time?

**Sum and average deal_amount_usd by month**

```sql
SELECT
  FORMAT_DATE('%Y-%m', deal_create_date) AS month,
  SUM(deal_amount_usd) AS sum_deals,
  ROUND(AVG(deal_amount_usd), 1) AS avg_deal
FROM `hubspot-crm-analysis.export.users`
GROUP BY month
ORDER BY month;
```

| month   | sum_deals | avg_deal |
| ------- | --------: | -------: |
| 2024-10 |  36019.39 |  18009.7 |
| 2024-11 |  26863.05 |  13431.5 |
| 2025-03 |  47177.67 |  15725.9 |

**Median and percentile deal size by month**

```sql
SELECT
  FORMAT_DATE('%Y-%m', deal_create_date) AS month,
  SUM(deal_amount_usd) AS sum_deals,
  ROUND(AVG(deal_amount_usd), 1) AS avg_deal,
  APPROX_QUANTILES(deal_amount_usd, 100)[OFFSET(25)] AS quartile_1,
  APPROX_QUANTILES(deal_amount_usd, 100)[OFFSET(50)] AS median,
  APPROX_QUANTILES(deal_amount_usd, 100)[OFFSET(75)] AS quartile_3
FROM `hubspot-crm-analysis.export.users`
GROUP BY month
ORDER BY month;
```

| month   | sum_deals | avg_deal | quartile_1 |   median | quartile_3 |
| ------- | --------: | -------: | ---------: | -------: | ---------: |
| 2024-10 |  36019.39 |  18009.7 |   14439.91 | 14439.91 |   21579.48 |
| 2024-11 |  26863.05 |  13431.5 |     6217.3 |   6217.3 |   20645.75 |
| 2025-03 |  47177.67 |  15725.9 |   10849.63 | 15004.29 |   21323.75 |

---

## 6. Value by Segment: Which sources and pipelines drive the most amount?

**Total deal_amount_usd by source, pipeline, stage**

```sql
SELECT
  contact_lead_source,
  deal_pipeline,
  deal_stage,
  ROUND(SUM(deal_amount_usd), 1) AS total_amount
FROM `hubspot-crm-analysis.export.users`
GROUP BY contact_lead_source, deal_pipeline, deal_stage
ORDER BY total_amount DESC;
```

| contact_lead_source | deal_pipeline   | deal_stage             | total_amount |
| ------------------- | --------------- | ---------------------- | -----------: |
| Paid Ads            | Agency Pipeline | Closed Won             |      45469.9 |
| Organic Search      | Agency Pipeline | Contract Sent          |      36583.8 |
| Partner             | Sales Pipeline  | Presentation Scheduled |      32906.3 |

---

## 7. Owner Performance: Which owners drive more deals and amount?

**Deals and amount by contact_owner_name**

```sql
SELECT
  contact_owner_name,
  COUNT(*) AS n_deals,
  ROUND(SUM(deal_amount_usd), 1) AS total_amount
FROM `hubspot-crm-analysis.export.users`
GROUP BY contact_owner_name
ORDER BY total_amount DESC;
```

| contact_owner_name | n_deals | total_amount |
| ------------------ | ------: | -----------: |
| Lily Bristow       |       6 |     103817.6 |
| Sara Lim           |       7 |      98736.0 |
| Maya Chen          |       8 |      81281.4 |

**Owner mix over time**

```sql
SELECT
  FORMAT_DATE('%Y-%m', deal_create_date) AS month,
  contact_owner_name,
  COUNT(*) AS n_deals,
  ROUND(SUM(deal_amount_usd), 1) AS total_amount
FROM `hubspot-crm-analysis.export.users`
GROUP BY month, contact_owner_name
ORDER BY contact_owner_name, month;
```

| month   | contact_owner_name | n_deals | total_amount |
| ------- | ------------------ | ------: | -----------: |
| 2024-02 | Alex Wong          |       1 |       2324.5 |
| 2024-08 | Alex Wong          |       1 |       5553.0 |
| 2024-09 | Alex Wong          |       1 |       3856.7 |

---

## 8. Response Time Quality: How fast are we responding to leads?

**Median and P90 of contact_response_time_days**

```sql
SELECT
  APPROX_QUANTILES(contact_response_time_days, 100)[OFFSET(50)] AS median,
  APPROX_QUANTILES(contact_response_time_days, 100)[OFFSET(90)] AS p90
FROM `hubspot-crm-analysis.export.users`;
```

| median | p90 |
| -----: | --: |
|     13 |  25 |

**Response time distribution buckets**

```sql
SELECT
  CASE
    WHEN contact_response_time_days <= 1 THEN '0-1'
    WHEN contact_response_time_days <= 3 THEN '1-3'
    WHEN contact_response_time_days <= 7 THEN '3-7'
    ELSE '7+'
  END AS time_bucket,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY time_bucket
ORDER BY
  CASE time_bucket
    WHEN '0-1' THEN 1
    WHEN '1-3' THEN 2
    WHEN '3-7' THEN 3
    ELSE 4
  END;
```

| time_bucket | n_deals |
| ----------- | ------: |
| 0-1         |       3 |
| 1-3         |       7 |
| 3-7         |       1 |
| 7+          |      29 |

---

## 9. Response Time Impact: Does response speed relate to deal outcomes and value?

**deal_stage distribution by response time bucket**

```sql
SELECT
  CASE
    WHEN contact_response_time_days <= 1 THEN '0-1'
    WHEN contact_response_time_days <= 3 THEN '1-3'
    WHEN contact_response_time_days <= 7 THEN '3-7'
    ELSE '7+'
  END AS time_bucket,
  deal_stage,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY time_bucket, deal_stage
ORDER BY time_bucket, n_deals DESC;
```

| time_bucket | deal_stage             | n_deals |
| ----------- | ---------------------- | ------: |
| 7+          | Contract Sent          |       6 |
| 7+          | Presentation Scheduled |       5 |
| 7+          | Closed Lost            |       7 |
| 7+          | Closed Won             |       5 |

**Average deal_amount_usd by response time bucket and stage**

```sql
SELECT
  CASE
    WHEN contact_response_time_days <= 1 THEN '0-1'
    WHEN contact_response_time_days <= 3 THEN '1-3'
    WHEN contact_response_time_days <= 7 THEN '3-7'
    ELSE '7+'
  END AS time_bucket,
  deal_stage,
  ROUND(AVG(deal_amount_usd), 1) AS avg_amount
FROM `hubspot-crm-analysis.export.users`
GROUP BY time_bucket, deal_stage
ORDER BY time_bucket, deal_stage;
```

| time_bucket | deal_stage             | avg_amount |
| ----------- | ---------------------- | ---------: |
| 0-1         | Appointment Scheduled  |    24351.2 |
| 0-1         | Closed Lost            |     9509.6 |
| 0-1         | Contract Sent          |    14626.3 |
| 1-3         | Closed Lost            |     5418.0 |
| 1-3         | Contract Sent          |    15004.3 |
| 1-3         | Presentation Scheduled |    13138.3 |
| 3-7         | Contract Sent          |     5553.0 |
| 7+          | Closed Won             |    16578.9 |
| 7+          | Closed Lost            |    12101.5 |
| 7+          | Contract Sent          |    13645.8 |
| 7+          | Presentation Scheduled |    12360.9 |
| 7+          | Qualified to Buy       |    16457.3 |

---

## 10. New Logos: How many companies place their first deal each month?

**First deal month per company**

```sql
SELECT
  company_domain,
  company_name,
  FORMAT_DATE('%Y-%m', MIN(deal_create_date)) AS first_deal_month
FROM `hubspot-crm-analysis.export.users`
GROUP BY company_domain, company_name
ORDER BY first_deal_month, company_name;
```

| first_deal_month | company_domain   | company_name |
| ---------------- | ---------------- | ------------ |
| 2024-01          | vantageworks.com | VantageWorks |
| 2024-02          | bluepeak.io      | BluePeak     |
| 2024-02          | quantumsoft.ai   | QuantumSoft  |

**Count of companies with first ever deal by month**

```sql
WITH firsts AS (
  SELECT
    company_name,
    FORMAT_DATE('%Y-%m', MIN(deal_create_date)) AS first_deal_month
  FROM `hubspot-crm-analysis.export.users`
  GROUP BY company_name
)
SELECT
  first_deal_month AS month,
  COUNT(*) AS n_first_deals
FROM firsts
GROUP BY month
ORDER BY month;
```

| month   | n_first_deals |
| ------- | ------------: |
| 2024-01 |             1 |
| 2024-02 |             4 |
| 2024-03 |             1 |

---

## 11. Repeat Business: What share of companies have multiple deals?

**Average response time by lifetime deal bucket**

```sql
WITH per_company AS (
  SELECT
    company_name,
    AVG(contact_response_time_days) AS avg_time_company,
    CASE
      WHEN COUNT(*) <= 1 THEN '1 deal'
      ELSE '2+ deals'
    END AS n_deals_bucket
  FROM `hubspot-crm-analysis.export.users`
  GROUP BY company_name
)
SELECT
  n_deals_bucket,
  ROUND(AVG(avg_time_company), 1) AS avg_time_days
FROM per_company
GROUP BY n_deals_bucket
ORDER BY n_deals_bucket;
```

| n_deals_bucket | avg_time_days |
| -------------- | ------------: |
| 1 deal         |           8.0 |
| 2+ deals       |          14.0 |

**Repeat rate by first deal month cohort**

```sql
WITH per_company AS (
  SELECT
    company_name,
    EXTRACT(MONTH FROM MIN(deal_create_date)) AS month_cohort,
    CASE WHEN COUNT(*) > 1 THEN 1 ELSE 0 END AS repeats
  FROM `hubspot-crm-analysis.export.users`
  GROUP BY company_name
)
SELECT
  month_cohort,
  ROUND(100 * SUM(repeats) / COUNT(*), 1) AS repeat_rate
FROM per_company
GROUP BY month_cohort
ORDER BY month_cohort;
```

| month_cohort | repeat_rate |
| -----------: | ----------: |
|            1 |       100.0 |
|            2 |        75.0 |
|            3 |       100.0 |

---

## 12. Source to Pipeline Path: How do sources map into pipelines?

**Cross tab counts of contact_lead_source by deal_pipeline**

```sql
SELECT
  contact_lead_source,
  deal_pipeline,
  COUNT(*) AS n_deals
FROM `hubspot-crm-analysis.export.users`
GROUP BY contact_lead_source, deal_pipeline
ORDER BY contact_lead_source, deal_pipeline;
```

| contact_lead_source | deal_pipeline   | n_deals |
| ------------------- | --------------- | ------: |
| Email Campaign      | Agency Pipeline |       5 |
| Email Campaign      | Sales Pipeline  |       4 |
| Organic Search      | Agency Pipeline |       6 |
| Organic Search      | Sales Pipeline  |       3 |

**Average deal_amount_usd by source and pipeline**

```sql
SELECT
  contact_lead_source,
  deal_pipeline,
  ROUND(AVG(deal_amount_usd), 1) AS avg_deal_amount
FROM `hubspot-crm-analysis.export.users`
GROUP BY contact_lead_source, deal_pipeline
ORDER BY contact_lead_source, deal_pipeline;
```

| contact_lead_source | deal_pipeline   | avg_deal_amount |
| ------------------- | --------------- | --------------: |
| Email Campaign      | Agency Pipeline |         11439.8 |
| Email Campaign      | Sales Pipeline  |         13146.2 |
| Organic Search      | Agency Pipeline |         12358.3 |
| Organic Search      | Sales Pipeline  |         16365.2 |

---

## 13. Top Opportunities: What are the largest current deals?

**Top N non-closed deals**

```sql
SELECT
  company_name,
  deal_pipeline,
  deal_stage,
  deal_create_date,
  deal_amount_usd
FROM `hubspot-crm-analysis.export.users`
WHERE NOT REGEXP_CONTAINS(deal_stage, r'(?i)Closed')
ORDER BY deal_amount_usd DESC
LIMIT 10;
```

| company_name | deal_pipeline   | deal_stage             | deal_create_date | deal_amount_usd |
| ------------ | --------------- | ---------------------- | ---------------- | --------------: |
| ExampleCo    | Sales Pipeline  | Presentation Scheduled | 2025-03-02       |         24351.2 |
| ExampleCo2   | Agency Pipeline | Contract Sent          | 2024-11-15       |         20645.8 |

---

## 14. Company Coverage: Are deals concentrated in few companies?

**Top 10 companies by cumulative deal_amount_usd**

```sql
SELECT
  company_name,
  ROUND(SUM(deal_amount_usd), 1) AS cumulated_usd
FROM `hubspot-crm-analysis.export.users`
GROUP BY company_name
ORDER BY cumulated_usd DESC
LIMIT 10;
```

| company_name    | cumulated_usd |
| --------------- | ------------: |
| VantageWorks    |      100365.9 |
| DataFlow        |       80630.9 |
| Innova          |       47660.3 |
| IronOak         |       41454.2 |
| BrightLabs      |       38375.4 |
| Skyline Systems |       34985.5 |
| QuantumSoft     |       26675.7 |
| BluePeak        |       25518.6 |
| Nexify          |       25183.2 |
| ClearPath       |       23880.3 |

---

## 15. Data Quality Checks: What basic issues exist in the flat export?

**Missing values in key fields**

```sql
SELECT
  COUNTIF(deal_amount_usd IS NULL) AS nulls_deal_amount_usd,
  COUNTIF(deal_stage IS NULL) AS nulls_deal_stage,
  COUNTIF(deal_pipeline IS NULL) AS nulls_deal_pipeline,
  COUNTIF(contact_lead_source IS NULL) AS nulls_contact_lead_source
FROM `hubspot-crm-analysis.export.users`;
```

| nulls_deal_amount_usd | nulls_deal_stage | nulls_deal_pipeline | nulls_contact_lead_source |
| --------------------: | ---------------: | ------------------: | ------------------------: |
|                     0 |                0 |                   0 |                         0 |

**Negative response days or non numeric type**

```sql
SELECT
  COUNTIF(
    contact_response_time_days < 0
    OR TYPEOF(contact_response_time_days) NOT IN ('INT64', 'FLOAT64', 'NUMERIC')
  ) AS errored_contact_response_time_days
FROM `hubspot-crm-analysis.export.users`;
```

| errored_contact_response_time_days |
| ---------------------------------: |
|                                 40 |
