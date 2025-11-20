# Run Order

1. Activate Python `.env`
2. Run ingestion via Python scripts or Jupyter notebooks (see below for commands to run based on format)
3. Configure connection to ClickHouse (scripts integrated with ClickHouse ingestion)
4. Build + test models via dbt (see Transformation section)
5. Open Metabase and refresh mart tables

---

# Ingestion

To extract from external data sources, we’ve used VSCode and Jupyter Notebook. When running Python scripts, we first set up a virtual environment then activated it:

```bash
uv venv
source .venv/bin/activate 
uv pip install -r requirements.txt
```

Once activated, run the following scripts depending on the format.

---

# CSVs / XLSX

**Script:** `ingest-ch.py`

```python
import os
import sys
import glob
import pandas as pd
import pdfplumber
import chardet
from clickhouse_connect import get_client

# === ClickHouse connection settings ===
CH_HOST = '54.87.106.52'
CH_PORT = 8123
CH_USER = 'ftw_grp4'
CH_PASS = 'Pika@025!FTW'
CH_DB = 'raw_grp4'

# === Data directories ===
BASE_DIR = os.environ.get('DATA_DIR', '/var/data/csv')
CSV_DIR = os.path.join(BASE_DIR, 'csv')
XLSX_DIR = os.path.join(BASE_DIR, 'xlsx')
PDF_DIR = os.path.join(BASE_DIR, 'pdf')

def normalize_name(name: str) -> str:
    """Normalize column or table names for ClickHouse."""
    return ''.join(c.lower() if c.isalnum() else '_' for c in name).strip('_')

def create_table_if_not_exists(client, db, table_name, df):
    """Create ClickHouse table if it doesn't exist."""
    cols = ', '.join(f"`{normalize_name(c)}` String" for c in df.columns)
    sql = f"CREATE TABLE IF NOT EXISTS {db}.{table_name} ({cols}) ENGINE = MergeTree() ORDER BY tuple()"
    client.command(sql)

def read_safely(file_path: str):
    """Read CSV or XLSX robustly with encoding and delimiter detection."""
    try:
        if file_path.endswith('.csv'):
            with open(file_path, 'rb') as fh:
                raw = fh.read(8192)
            encoding = (chardet.detect(raw)['encoding'] or 'utf-8').strip()
            try:
                df = pd.read_csv(
                    file_path,
                    dtype=str,
                    encoding=encoding,
                    on_bad_lines='skip',
                    skip_blank_lines=True
                )
            except Exception:
                df = pd.read_csv(
                    file_path,
                    dtype=str,
                    sep=None,
                    engine='python',
                    encoding=encoding,
                    on_bad_lines='skip'
                )
        elif file_path.endswith('.xlsx'):
            df = pd.read_excel(file_path, dtype=str).fillna('')
        else:
            return None
        df.columns = [str(c).strip() for c in df.columns]
        df = df.dropna(how='all')
        if df.shape[0] > 1 and set(df.columns) == set(df.iloc[0]):
            df = df[1:]
        df = df[df.apply(lambda r: any(str(x).strip() for x in r), axis=1)]
        df = df.reset_index(drop=True).fillna('')
        print(f"Successfully cleaned {os.path.basename(file_path)} with {len(df)} rows and {len(df.columns)} columns.")
        return df
    except Exception as e:
        print(f"Error reading {file_path}: {e}")
        return pd.DataFrame()
    
def extract_pdf_table(file_path):
    """Extract tabular data from PDF using pdfplumber."""
    try:
        with pdfplumber.open(file_path) as pdf:
            all_dfs = []
            for page in pdf.pages:
                table = page.extract_table()
                if table:
                    df = pd.DataFrame(table[1:], columns=table[0])
                    all_dfs.append(df)
            if all_dfs:
                return pd.concat(all_dfs, ignore_index=True)
    except Exception as e:
        print(f"Could not extract {file_path}: {e}")
    return None

def load_dataframe(client, db, df, file_path):
    """Insert dataframe into ClickHouse."""
    base = os.path.basename(file_path)
    table = normalize_name(os.path.splitext(base)[0])
    df = df.rename(columns=lambda c: normalize_name(str(c)))
    df = df.astype(str).fillna('')
    if df.empty:
        print(f"Skipping {file_path}: empty DataFrame.")
        return
    try:
        create_table_if_not_exists(client, db, table, df)
        columns = list(df.columns)
        data = [tuple(x) for x in df.to_numpy().tolist()]
        if not data:
            print(f"Skipping {file_path}: no valid records to insert.")
            return
        client.insert(f"{db}.{table}", data, column_names=columns)
        print(f"Loaded {len(data)} rows into {db}.{table}")
    except Exception as e:
        print(f"Error loading {table} into ClickHouse: {type(e).__name__} - {e}")
        
def main():
    # Allow optional single-file input via CLI
    target_file = sys.argv[1] if len(sys.argv) > 1 else None
    client = get_client(host=CH_HOST, port=CH_PORT, username=CH_USER, password=CH_PASS)
    try:
        client.command('SELECT 1')
        print(f"Connected to ClickHouse host={CH_HOST}, port={CH_PORT}, user={CH_USER}")
    except Exception as e:
        print(f"Connection failed: {e}")
        return
    # Ensure database exists and select it
    client.command(f"CREATE DATABASE IF NOT EXISTS {CH_DB}")
    client.command(f"USE {CH_DB}")
    print(f"Using database: {CH_DB}")
    print("Databases available:", client.command("SHOW DATABASES"))
    if target_file:
        files = [target_file]
    else:
        csv_files = glob.glob(os.path.join(CSV_DIR, '*.csv'))
        xlsx_files = glob.glob(os.path.join(XLSX_DIR, '*.xlsx'))
        pdf_files = glob.glob(os.path.join(PDF_DIR, '*.pdf'))
        files = csv_files + xlsx_files + pdf_files
    if not files:
        print("No files found in", BASE_DIR)
        return
    for f in files:
        print(f"\nProcessing {f}")
        if f.endswith('.pdf'):
            df = extract_pdf_table(f)
        else:
            df = read_safely(f)
        if df is None or df.empty:
            print(f"Skipping {f}: could not extract data.")
            continue
        load_dataframe(client, CH_DB, df, f)
if __name__ == '__main__':
    main()
```

**Command:**

```bash
uv run ingest-ch.py
```

---

# PDF

**Script:** `extract-4p-pdf.py`

```python
import requests
import pdfplumber
import pandas as pd
import re

# URLs of the quarterly reports
urls = {
    1: "https://pantawid.dswd.gov.ph/wp-content/uploads/2020/07/1st-Quarterly-Report-2020.pdf",
    2: "https://pantawid.dswd.gov.ph/wp-content/uploads/2020/09/2nd-Quarterly-Report-2020.pdf",
    3: "https://pantawid.dswd.gov.ph/wp-content/uploads/2022/01/4Ps-3rd-Quarterly-Report-2020.pdf",
    4: "https://pantawid.dswd.gov.ph/wp-content/uploads/2021/11/2020.12.31_Quarterly-Report-on-Program-Coverage.pdf"
}

# List of the 17 regions we want to extract 
REGIONS = [
    "NCR", "CAR", "I", "II", "III", "IV-A CALABARZON", "MIMAROPA", "V",
    "VI", "VII", "VIII", "IX", "X", "XI", "XII", "Caraga", "BARMM"
]

def download_pdf(url, quarter):
    pdf_path = f"temp_Q{quarter}.pdf"
    print(f"\nQ{quarter}: Downloading PDF...")
    r = requests.get(url)
    r.raise_for_status()
    with open(pdf_path, "wb") as f:
        f.write(r.content)
    return pdf_path

def clean_number(val):
    if pd.isna(val):
        return None
    val = str(val).replace(",", "").replace("%", "").strip()
    try:
        return float(val)
    except ValueError:
        return None

def parse_line(line):
    """
    Parse a single line of table into Region, Target, Actual, Difference, Percentage
    Handles negative differences when Actual > Target.
    """
    # Match line: Region Target Actual Difference Percentage
    # Example: NCR 229,824 215,206 14,618 93.6%
    match = re.match(r"^([A-Za-z0-9\-/ ]+?)\s+([\d,]+)\s+([\d,]+)\s+([\d,()-]+)\s+([\d\.]+)", line)
    if match:
        region = match.group(1).strip()
        target = match.group(2).replace(",", "")
        actual = match.group(3).replace(",", "")
        diff = match.group(4).replace(",", "")
        perc = match.group(5)
        # Convert difference with parentheses to negative
        if "(" in diff and ")" in diff:
            diff = "-" + diff.replace("(", "").replace(")", "")
        return [region, target, actual, diff, perc]
    return None

def extract_table_from_pdf(url, quarter):
    pdf_path = download_pdf(url, quarter)
    table_rows = []
    found_table = False

    with pdfplumber.open(pdf_path) as pdf:
        # Pages 7-12 usually contain Table 3
        for i, page in enumerate(pdf.pages[6:12], start=7):
            text = page.extract_text() or ""

            if "Table 3. Regional Breakdown" in text or "Island /" in text:
                print(f"Found Table 3 header on page {i}")
                found_table = True

            if found_table:
                lines = text.split("\n")
                for line in lines:
                    row = parse_line(line)
                    if row and row[0] in REGIONS:  # Only include the 17 regions
                        table_rows.append(row)

                # Stop if Grand Total appears
                if any("Grand Total" in l for l in lines):
                    break

    # Create DataFrame
    df = pd.DataFrame(table_rows, columns=["Region", "Target", "Actual", "Difference", "Percentage"])

    # Convert numeric columns
    for col in ["Target", "Actual", "Difference", "Percentage"]:
        df[col] = df[col].apply(clean_number)

    # Save CSV
    filename = f"raw_2020_Q{quarter}_4Ps_beneficiaries.csv"
    df.to_csv(filename, index=False)
    print(f"Saved {filename} ({len(df)} rows)")
    print(df)

# Run for all 4 quarters
for q in range(1, 5):
    extract_table_from_pdf(urls[q], q)
```

**Command:**

```bash
uv run extract-4p-pdf.py
```

---

# Notebook Workflow (macOS tested)

```bash
# !conda install tabula
# !conda install pdfplumber
```

## Load Packages

```python
from typing import List, Dict, Optional, Union
import pandas as pd
import numpy as np
import importlib
import re
from tqdm import tqdm
from urllib.parse import urljoin
import warnings
warnings.filterwarnings("ignore")

# Python Package to extract and automate webscraping
import requests
from bs4 import BeautifulSoup
import time
import os

# Python Package to extract pdf tables
import tabula
import pdfplumber
```

## Web Scraping

```python
# Install required packages if not already installed
# !pip install beautifulsoup4 requests pandas tqdm

# URL to scrape
url = "https://pantawid.dswd.gov.ph/programimplementationreport/"

# Create a directory to store downloaded files
download_dir = "pantawid_reports"
os.makedirs(download_dir, exist_ok=True)

# Send a request to the website with proper headers to mimic a browser
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
}

def download_file(url, filename):
    """Download a file from a URL to a specified filename"""
    try:
        response = requests.get(url, headers=headers, stream=True)
        response.raise_for_status()  # Raise an exception for HTTP errors
        
        # Get file size for progress bar
        file_size = int(response.headers.get('Content-Length', 0))
        
        # Show download progress
        progress_bar = tqdm(total=file_size, unit='B', unit_scale=True, desc=filename)
        
        with open(filename, 'wb') as file:
            for chunk in response.iter_content(chunk_size=8192):
                if chunk:
                    file.write(chunk)
                    progress_bar.update(len(chunk))
        
        progress_bar.close()
        return True
    except Exception as e:
        print(f"Error downloading {url}: {e}")
        return False

try:
    # Make the request to the main page
    response = requests.get(url, headers=headers)
    
    # Check if the request was successful
    if response.status_code == 200:
        # Parse the HTML content
        soup = BeautifulSoup(response.text, 'html.parser')
        
        # Find all anchor tags (links)
        links = soup.find_all('a')
        
        # Filter links that match the quarterly report pattern
        quarterly_reports = []
        
        for link in links:
            href = link.get('href', '')
            text = link.text.strip()
            
            # Check if the link points to a PDF file and contains quarterly report keywords
            if href.endswith('.pdf') and re.search(r'quarter|quarterly|q[1-4]|report', href.lower() + ' ' + text.lower(), re.IGNORECASE):
                # Make sure we have the full URL
                full_url = urljoin(url, href)
                
                # Extract year from the URL or text if possible
                year_match = re.search(r'20\d{2}', href + ' ' + text)
                year = year_match.group(0) if year_match else "Unknown"
                
                # Extract quarter information
                quarter = ""
                if '1st' in text.lower() or 'first' in text.lower() or 'q1' in text.lower():
                    quarter = "1st Quarter"
                elif '2nd' in text.lower() or 'second' in text.lower() or 'q2' in text.lower():
                    quarter = "2nd Quarter"
                elif '3rd' in text.lower() or 'third' in text.lower() or 'q3' in text.lower():
                    quarter = "3rd Quarter"
                elif '4th' in text.lower() or 'fourth' in text.lower() or 'q4' in text.lower():
                    quarter = "4th Quarter"
                else:
                    quarter = "Quarterly Report"
                
                quarterly_reports.append({
                    'Quarter': quarter,
                    'Year': year,
                    'Link Text': text,
                    'URL': full_url
                })
        
        # If no quarterly reports found, look for any PDF files
        if not quarterly_reports:
            print("No specific quarterly reports found. Looking for any PDF files...")
            for link in links:
                href = link.get('href', '')
                text = link.text.strip()
                
                if href.endswith('.pdf'):
                    full_url = urljoin(url, href)
                    quarterly_reports.append({
                        'Quarter': 'Unknown',
                        'Year': 'Unknown',
                        'Link Text': text,
                        'URL': full_url
                    })
        
        # Create a DataFrame from the collected reports
        if quarterly_reports:
            df = pd.DataFrame(quarterly_reports)
            
            print(f"Found {len(df)} report files to download.")
            
            # Download each file
            for index, row in df.iterrows():
                # Create a filename based on the report information
                if row['Year'] != 'Unknown' and row['Quarter'] != 'Unknown':
                    filename = f"{row['Year']}_{row['Quarter'].replace(' ', '_')}.pdf"
                else:
                    # Extract filename from URL if year/quarter unknown
                    url_filename = os.path.basename(row['URL'])
                    filename = url_filename if url_filename else f"report_{index}.pdf"
                
                # Full path for saving
                filepath = os.path.join(download_dir, filename)
                
                print(f"\nDownloading: {row['Link Text']} ({row['URL']})")
                success = download_file(row['URL'], filepath)
                
                if success:
                    print(f"Successfully downloaded to {filepath}")
                else:
                    print(f"Failed to download {row['URL']}")
            
            print(f"\nAll downloads completed. Files saved to {os.path.abspath(download_dir)}")
        else:
            print("No PDF reports found on the page.")
    else:
        print(f"Failed to retrieve the webpage. Status code: {response.status_code}")

except Exception as e:
    print(f"An error occurred: {e}")
```

## PDF Reports Directory Check and Listing

```python
import os

# Path to the directory
pantawid_reports_dir = "pantawid_reports"

# Check if the directory exists
if os.path.exists(pantawid_reports_dir) and os.path.isdir(pantawid_reports_dir):
    # Get list of all files in the directory
    files = os.listdir(pantawid_reports_dir)
    
    # Print the number of files found
    print(f"Found {len(files)} files in {pantawid_reports_dir} directory:")
    
    # Loop through and print each filename
    for i, filename in enumerate(files, 1):
        full_path = os.path.join(pantawid_reports_dir, filename)
        
        if os.path.isfile(full_path):
            # Get file size in KB
            file_size = os.path.getsize(full_path) / 1024
            
            print(f"{i}. {filename} ({file_size:.2f} KB)")
            
            # Extract year and quarter from filename if PDF
            if filename.lower().endswith('.pdf'):
                parts = filename.split('_')
                if len(parts) >= 2 and parts[0].isdigit():
                    year = parts[0]
                    quarter = parts[1].replace('_', ' ')
                    print(f"   Year: {year}, Quarter: {quarter}")
    
    # Simple list of filenames
    print("\nSimple list of filenames:")
    for a in list(os.listdir(pantawid_reports_dir)):
        print(a)
else:
    print(f"Directory '{pantawid_reports_dir}' does not exist. Please run the download script first.")
    
    # Prompt to create the directory
    create_dir = input("Would you like to create the directory now? (y/n): ")
    if create_dir.lower() == 'y':
        os.makedirs(pantawid_reports_dir, exist_ok=True)
        print(f"Directory '{pantawid_reports_dir}' created. It's empty.")

## Load Specific PDF and Page

```bash
PDF = "pantawid_reports/2025_4th_Quarter.pdf"
page_num = 22
```

```python
def extract_table_from_pdf_tabula(pdf_path, page_num, table_index=0):
    """
    Extracts a table from a given PDF page using Tabula.
    Returns the specified table as a DataFrame.
    table_index: selects which table if multiple exist on page.
    """
    dfs = tabula.read_pdf(
        pdf_path,
        pages=page_num,
        multiple_tables=True,
        stream=False,
        guess=True
    )
    return dfs[table_index] if dfs else None

# Example usage
tabula_df = extract_table_from_pdf_tabula(PDF, page_num, table_index=1)
tabula_df.head(10)
```

## Extract Table from PDF using pdfplumber

```python
def extract_table_from_pdf_pdfplumber(pdf_path, page_num, table_index=0):
    """
    Extracts tables from a given PDF page using pdfplumber.
    Returns the specified table as a DataFrame.
    """
    with pdfplumber.open(pdf_path) as pdf:
        page = pdf.pages[page_num - 1]  # 0-based index
        tables = page.extract_tables()

    dfs = [pd.DataFrame(t) for t in tables if t and any(any(cell for cell in row) for row in t)]
    
    if not dfs:
        print(f"No tables found on page {page_num}.")
        return None

    df_all = pd.concat(dfs, ignore_index=True)
    return dfs[table_index] if len(dfs) > table_index else df_all

# Example usage
pdfplumber_df = extract_table_from_pdf_pdfplumber(PDF, page_num, table_index=1)
pdfplumber_df.head(10)
```


## Data Preprocessing: Consolidate Columns

```python
def combine_every_3_columns(df):
    """
    Combines every 3 consecutive columns into one.
    Keeps the first non-empty value in each row of the 3-column block.
    """
    combined_cols = []
    num_cols = df.shape[1]

    for i in range(0, num_cols, 3):
        block = df.iloc[:, i:i+3]
        combined = block.apply(lambda row: next((x for x in row if pd.notna(x) and str(x).strip() != ""), None), axis=1)
        combined_cols.append(combined)

    combined_df = pd.concat(combined_cols, axis=1)
    combined_df.columns = [f"group_{i+1}" for i in range(combined_df.shape[1])]
    return combined_df

# Run consolidation
cleaning_step_1_df = combine_every_3_columns(pdfplumber_df)
cleaning_step_1_df.head(10)
```


## Count Non-Empty Cells After First Row

```python
def count_non_empty_after_first(before_ncr: pd.DataFrame) -> pd.Series:
    """
    Count non-empty cells per column after the first row.
    """
    return before_ncr.iloc[1:].apply(
        lambda col: (col.astype(str)
                       .str.strip()
                       .replace(["None","none","NaN","nan","NULL","null",""], pd.NA)
                       .notna()
                       .sum())
    )

counts = count_non_empty_after_first(before_ncr)
counts
```


## Clean and Join Column Values

```python
def clean_and_join(series: pd.Series) -> str | None:
    """
    Combine non-empty cells into a single string.
    Replaces newlines, trims spaces, ignores None/NaN/blanks.
    """
    cleaned = [str(x).replace("\n", " ").strip()
               for x in series
               if pd.notna(x) and str(x).strip().lower() != "none" and str(x).strip() != ""]
    return " ".join(cleaned) if cleaned else None
```

## Build Final Column Headers

```python
def build_headers(before_ncr: pd.DataFrame, counts: pd.Series) -> list[str]:
    """
    Builds cleaned headers using first row or combining non-empty rest.
    """
    new_headers = []
    for pos, (col, count) in enumerate(zip(before_ncr.columns, counts)):
        if count == 0:
            new_headers.append(before_ncr.iloc[0, pos])
        else:
            new_headers.append(clean_and_join(before_ncr[col].iloc[1:]))
    return new_headers

new_headers = build_headers(before_ncr, counts)
new_headers
```


## Apply Cleaned Headers

```python
def apply_headers(final_df: pd.DataFrame, ncr_idx: int, new_headers: list[str]) -> pd.DataFrame:
    """
    Slice data from NCR onward and assign new headers.
    """
    clean_df = final_df.iloc[ncr_idx:].copy()
    clean_df.columns = new_headers
    return clean_df

cleaning_step_2_df = apply_headers(cleaning_step_1_df, ncr_idx, new_headers)
cleaning_step_2_df.tail(15)
```

## Map Region to Island and Filter

```python
def get_region_to_island():
    return {
        "NCR": "Luzon", "CAR": "Luzon", "I": "Luzon", "II": "Luzon",
        "III": "Luzon", "IV-A": "Luzon", "MIMAROPA": "Luzon", "V": "Luzon",
        "VI": "Visayas", "VII": "Visayas", "VIII": "Visayas",
        "IX": "Mindanao", "X": "Mindanao", "XI": "Mindanao", "XII": "Mindanao",
        "Caraga": "Mindanao", "BARMM": "Mindanao",
    }

def add_island_and_filter(df: pd.DataFrame) -> pd.DataFrame:
    """
    Adds Island column and drops rows where Island is NaN.
    """
    region_to_island = get_region_to_island()
    out = df.copy()
    out["Island"] = out.iloc[:, 0].map(region_to_island)
    out = out[out["Island"].notna()].reset_index(drop=True)
    return out

cleaning_step_3_df = add_island_and_filter(cleaning_step_2_df)
cleaning_step_3_df
```

## Add Year and Quarter from Filename

```python
def add_year_quarter(df: pd.DataFrame, pdf_path: str) -> pd.DataFrame:
    """
    Extracts Year and Quarter from PDF filename and adds as columns.
    """
    fname = pdf_path.split("/")[-1]
    parts = fname.split("_")
    year = parts[0]
    quarter = parts[1][0]
    out = df.copy()
    out["Year"] = year
    out["Quarter"] = quarter
    return out

cleaning_step_4_df = add_year_quarter(cleaning_step_3_df, PDF)
cleaning_step_4_df
```

## Example Application: Single Header Extraction

```python
pdfplumber_df = extract_table_from_pdf_pdfplumber(PDF, page_num, table_index=1)
step1 = combine_every_3_columns(pdfplumber_df)
ncr_idx, before_ncr = get_rows_before_ncr(step1)
counts = count_non_empty_after_first(before_ncr)
headers = build_headers(before_ncr, counts)
step2 = apply_headers(step1, ncr_idx, headers)
step3 = add_island_and_filter(step2)
final_df = add_year_quarter(step3, PDF)
final_df
```

## Example Application: Multiple Header Extraction

```python
pdfplumber_df = extract_table_from_pdf_pdfplumber(PDF, page_num, table_index=1)
step1 = combine_every_3_columns(pdfplumber_df)
ncr_idx, before_ncr = get_rows_before_ncr(step1)
counts = count_non_empty_after_first(before_ncr)
headers = build_headers(before_ncr, counts)
step2 = apply_headers(step1, ncr_idx, headers)
step3 = add_island_and_filter(step2)
final_df = add_year_quarter(step3, PDF)
final_df
```

## Transformation: dbt Commands0
- Use dbt build command to run tests + models (target can be local or remote)
```bash
# Build models and run tests
docker compose --profile jobs run --rm dbt build --profiles-dir . --target remote
```
- For testing only, run the ff:
```bash
# Run tests only
docker compose --profile jobs run --rm dbt test --profiles-dir . --target remote
```

