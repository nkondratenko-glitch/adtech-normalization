## Kondratenko | HW 1 | Data Engineering: AdTech Dataset Normalization

This repository contains a fully normalized relational design for the provided denormalized `users.csv` and `campaigns.csv` datasets, plus scripts to transform the raw files into load-ready tables for MySQL.

## Why this schema
The source files contain several normalization issues:

- `users.csv`
  - `Interests` is a multi-valued comma-separated attribute, which violates 1NF.
  - `Location` is repeated for hundreds of thousands of users.
- `campaigns.csv`
  - `AdvertiserName`, `AdSlotSize`, and `TargetingCriteria` are repeated across many campaigns.
  - `TargetingCriteria` mixes age range, location, and interest in one text field, which makes filtering and indexing inefficient.

The proposed design separates reusable entities into dimension tables and uses bridge tables for many-to-many relationships.

## Main design decisions
1. **Users and interests are split** into `users`, `interests`, and `user_interests`.
2. **Campaign targeting is decomposed** into:
   - `campaign_targeting` for age range
   - `campaign_target_locations`
   - `campaign_target_interests`
3. **Advertisers and ad slot sizes are dimensions** to remove repeated text values.
4. **Future-proof event model** includes optional `impressions` and `clicks` fact tables. Every click references exactly one impression through `clicks.impression_id`, which is the cleanest way to compute CTR and attribute spend.

## Redundancy and anomaly analysis
- **Update anomaly:** Renaming an advertiser or correcting an ad slot size in the denormalized file would require updating many campaign rows.
- **Insertion anomaly:** A new interest or location cannot exist independently without being embedded into a user/campaign text string.
- **Deletion anomaly:** Deleting the last campaign of an advertiser would also remove the only record of that advertiser name.
- **Query inefficiency:** Filtering campaigns by target location or interest requires string parsing on every row in the raw format; the normalized schema converts this into indexed joins.

## Expected row counts after normalization
- advertisers: 100
- locations: 5
- interests: 8
- ad_slot_sizes: 3
- users: 700,000
- user_interests: 1,748,972
- campaigns: 1,013
- campaign_targeting: 1,013
- campaign_target_locations: 1,013
- campaign_target_interests: 1,013

## Repository layout
- `sql/01_schema.sql` – DDL for all tables
- `sql/02_load.sql` – bulk load statements for normalized CSVs
- `sql/03_demo_queries.sql` – simple `SELECT * LIMIT 10` queries for verification
- `scripts/transform_and_load.py` – transforms raw CSVs into normalized CSVs and can optionally load them to MySQL
- `scripts/01_start_mysql.sh` – starts MySQL with Docker Compose
- `scripts/02_create_schema.sh` – creates the schema
- `scripts/03_load_data.sh` – runs the transformation and loads normalized data
- `docker-compose.yml` – local MySQL 8 environment
- `docs/demo_selects.png` – screenshot-style proof of table contents
- `docs/schema_notes.md` – 1–2 sentence explanation for each table

## Local setup

### 1) Place raw data in `raw_data/`
```bash
mkdir -p raw_data
cp /path/to/users.csv raw_data/users.csv
cp /path/to/campaigns.csv raw_data/campaigns.csv
```

### 2) Start MySQL
```bash
bash scripts/01_start_mysql.sh
```

### 3) Create tables
```bash
bash scripts/02_create_schema.sh
```

### 4) Transform source files and load data
```bash
bash scripts/03_load_data.sh
```

### 5) Validate
```bash
docker compose exec mysql mysql -uadtech -padtech adtech < sql/03_demo_queries.sql
```

## CTR and click/impression linkage
If impressions and clicks are stored separately, the best practice is:

- `impressions(impression_id PK, campaign_id FK, user_id FK, impression_time, cost, ...)`
- `clicks(click_id PK, impression_id FK UNIQUE, click_time, cpc_amount, ...)`

This enforces a direct relationship from click to the exact impression that generated it. It also makes CTR efficient:

```sql
SELECT i.campaign_id,
       COUNT(*) AS impressions,
       COUNT(c.click_id) AS clicks,
       ROUND(COUNT(c.click_id) / COUNT(*) * 100, 2) AS ctr_pct
FROM impressions i
LEFT JOIN clicks c ON c.impression_id = i.impression_id
GROUP BY i.campaign_id;
```

## Spend tracking
Budget and remaining budget stay at the campaign level, but actual realized spend should be derived from event tables:

- impression-based CPM cost in `impressions.cost`
- click-based CPC cost in `clicks.cpc_amount`

That structure keeps campaign master data small while making analytical queries straightforward and index-friendly.
