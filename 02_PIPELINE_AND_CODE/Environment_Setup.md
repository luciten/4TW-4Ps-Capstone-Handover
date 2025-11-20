# ENVIRONMENT SETUP SUMMARY

1. PYTHON VERSION
- Python 3.12+ (Project tested on Python 3.12.3)

2. TOOLS USED
- dlt (for extraction + loading to ClickHouse)
- ClickHouse (as the data warehouse)
- dbt-core (for transformations)
- Docker + Docker Compose (environment + ClickHouse instance)
- Tableau (for dashboard / visualization)

3. REQUIRED INSTALLATIONS
Install all dependencies:
    pip install -r requirements.txt

Or install manually:
- dlt
- dlt[clickhouse]
- clickhouse-connect
- pandas
- psycopg2-binary
- openpyxl
- pdfplumber
- tabula-py
- python-dotenv


4. PROJECT STRUCTURE (Important folders)
- extract-loads/  
- extract-loads/csv_pipeline.py  
- extract-loads/pdf_pipeline.py  
- transform/ (dbt project)  
- transform/models/staging  
- transform/models/marts  
- configs/ (contains .env template)

5. CONFIGURATION (No secrets, use placeholders)
All credentials stored in:
    .env  (NOT included in submission)

Template (.env.example):
    REMOTE_CH_HOST=<your_clickhouse_host>
    REMOTE_CH_USER=<your_user>
    REMOTE_CH_PASS=<your_password>
    REMOTE_CH_DB=raw_grp4

6. CLICKHOUSE ENVIRONMENT
You may run ClickHouse using Docker:
    docker compose up -d clickhouse

7. NOTES
- No secrets are committed.
- All pipelines assume files are inside /var/dlt/data/staging/.
