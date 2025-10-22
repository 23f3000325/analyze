# Data Processing Workflow

This repository showcases a data processing pipeline using Python 3.11+, Pandas 2.3+, and GitHub Actions. It demonstrates how to process an Excel file (`data.xlsx`), convert it to CSV (`data.csv`), and generate a structured JSON output (`result.json`) through an automated CI/CD workflow.

## Project Structure

This project expects the following file structure:

- `execute.py`: A Python script responsible for data processing, with a non-trivial error fixed.
- `data.xlsx`: The source Excel file containing the raw data (provided).
- `data.csv`: (Generated and committed) The CSV version of `data.xlsx`, created by `execute.py`.
- `result.json`: (Generated in CI) The final JSON output, published via GitHub Pages.
- `.github/workflows/ci.yml`: GitHub Actions workflow for continuous integration and deployment.
- `index.html`: A simple landing page for the project, hosted on GitHub Pages.
- `LICENSE`: The MIT License for this project.

***Note:*** *Due to the strict output schema, `execute.py`, `data.csv`, and `.github/workflows/ci.yml` are provided as code blocks within this README. You will need to create these files in your repository based on the content provided below.*

## `execute.py` Content (Fixed)

This script reads `data.xlsx`, performs robust data type conversions (handling potential errors), saves the cleaned data to `data.csv`, and then aggregates data into `result.json`. The non-trivial error fixed involves handling mixed-type columns and ensuring datetime objects are correctly serialized for JSON.

```python
import pandas as pd
import json
import os

def process_data(input_excel_path="data.xlsx", output_csv_path="data.csv", output_json_path="result.json"):
    if not os.path.exists(input_excel_path):
        raise FileNotFoundError(f"Input file not found: {input_excel_path}")
    print(f"Reading {input_excel_path}...")
    try:
        df = pd.read_excel(input_excel_path, engine='openpyxl')
        print("Successfully read Excel file.")
    except Exception as e:
        raise ValueError(f"Error reading Excel file: {e}")

    if 'Value' in df.columns:
        df['Value'] = pd.to_numeric(df['Value'], errors='coerce')
        print("Converted 'Value' column to numeric, coercing errors.")
    else:
        print("Warning: 'Value' column not found.")

    datetime_col = None
    for col_name in ['Timestamp', 'Date', 'ActivityDate']:
        if col_name in df.columns:
            df[col_name] = pd.to_datetime(df[col_name], errors='coerce')
            datetime_col = col_name
            print(f"Converted '{col_name}' column to datetime, coercing errors.")
            break
    if datetime_col is None:
        print("Warning: No common datetime column found ('Timestamp', 'Date', 'ActivityDate').")

    df_processed = df.dropna(subset=['Value']).copy()
    if df_processed.empty:
        print("Warning: No valid numeric 'Value' entries found after cleaning. Skipping aggregation.")
        aggregated_data_dict = {}
    else:
        if 'Category' in df_processed.columns:
            aggregated_data = df_processed.groupby('Category')['Value'].sum().reset_index()
            aggregated_data_dict = aggregated_data.set_index('Category')['Value'].to_dict()
            print("Aggregated data by 'Category'.")
        else:
            print("Warning: 'Category' column not found for aggregation. Summing all valid 'Value' entries.")
            total_sum = df_processed['Value'].sum()
            aggregated_data_dict = {"Total_Value_Sum": total_sum}

        if datetime_col and not df_processed.empty:
            latest_timestamp = df_processed[datetime_col].max()
            if pd.notna(latest_timestamp):
                aggregated_data_dict['latest_data_timestamp'] = latest_timestamp.isoformat()
                print(f"Added latest timestamp: {latest_timestamp.isoformat()}")

    print(f"Saving processed data to {output_csv_path}...")
    df.to_csv(output_csv_path, index=False)
    print("Successfully saved data to CSV.")

    print(f"Saving aggregated data to {output_json_path}...")
    try:
        with open(output_json_path, 'w') as f:
            json.dump(aggregated_data_dict, f, indent=4)
        print("Successfully saved aggregated data to JSON.")
    except TypeError as e:
        print(f"Error serializing to JSON: {e}")
        print("Ensure all data types in the dictionary are JSON serializable.")

if __name__ == "__main__":
    process_data()
```

## `data.csv` Content

This CSV is generated from `data.xlsx` by `execute.py`. It reflects the coercion of non-numeric 'Value' entries to empty and invalid 'Timestamp' entries to empty. (Example content based on a hypothetical `data.xlsx` as described in the prompt.)

```csv
Category,Value,Timestamp,Other
A,100.0,2023-01-01 10:00:00,info1
B,200.0,2023-01-01 11:00:00,info2
A,150.0,2023-01-02 12:00:00,info3
C,,2023-01-02 13:00:00,info4
B,250.0,,info5
A,50.0,2023-01-03 14:00:00,info6
```

## GitHub Actions Workflow (`.github/workflows/ci.yml`)

This workflow automates linting, data processing, and deployment to GitHub Pages.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pages: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install pandas openpyxl ruff

      - name: Run Ruff Linter
        run: |
          ruff check .
          ruff format . --check

      - name: Run execute.py and generate result.json
        run: |
          python execute.py

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.' # Uploads everything in the current directory (root of the repo)

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Workflow Highlights

1.  **Error Correction**: The `execute.py` script has been reviewed and a non-trivial error preventing its proper execution has been fixed to handle data type inconsistencies gracefully.
2.  **Data Conversion**: `data.xlsx` is converted to `data.csv` as part of the `execute.py` script's run.
3.  **JSON Output**: The processed and aggregated data is outputted as `result.json`.
4.  **CI/CD with GitHub Actions**: 
    -   **Linter**: Ruff is used to ensure code quality and style.
    -   **Script Execution**: `execute.py` is run to generate `data.csv` and `result.json`.
    -   **GitHub Pages Deployment**: `result.json` (along with `index.html` and other static assets) is published to GitHub Pages, making the processed data accessible.

## Setup and Local Development

To set up this project and run `execute.py` locally:

1.  **Create project files**: Based on the code blocks above, create `execute.py`, `.github/workflows/ci.yml`, `data.csv`, `index.html`, and `LICENSE` in your repository. Ensure `data.xlsx` (provided separately) is also in the root directory.
2.  **Clone the repository** (if not already done):
    ```bash
    git clone https://github.com/your-username/your-repo-name.git
    cd your-repo-name
    ```
3.  **Create a virtual environment (recommended)**:
    ```bash
    python -m venv .venv
    source .venv/bin/activate # On Windows: .venv\Scripts\activate
    ```
4.  **Install dependencies**:
    ```bash
    pip install pandas openpyxl ruff
    ```
5.  **Run the script**:
    ```bash
    python execute.py
    ```
    This will re-generate `data.csv` and `result.json` in your local directory.

## Accessing Processed Data

The `result.json` file is automatically generated and published to GitHub Pages with every successful push to the `main` branch. You can access it via the GitHub Pages URL for this repository (e.g., `https://your-username.github.io/your-repo-name/result.json`).