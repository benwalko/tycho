# Output Generation

## XLSX with Formatted Headers

```python
from openpyxl import Workbook
from openpyxl.styles import Font, Alignment
from openpyxl.utils import get_column_letter

def create_xlsx(data: list[dict], output_path: str, sheet_name: str = "Data"):
    """Create a formatted XLSX from a list of dicts."""
    wb = Workbook()
    ws = wb.active
    ws.title = sheet_name
    if not data:
        wb.save(output_path)
        return
    headers = list(data[0].keys())
    for col_idx, header in enumerate(headers, 1):
        cell = ws.cell(row=1, column=col_idx, value=header)
        cell.font = Font(bold=True)
        cell.alignment = Alignment(horizontal="center")
    for row_idx, row in enumerate(data, 2):
        for col_idx, header in enumerate(headers, 1):
            ws.cell(row=row_idx, column=col_idx, value=row.get(header, ""))
    for col_idx, header in enumerate(headers, 1):
        max_len = len(str(header))
        for row in ws.iter_rows(min_row=2, min_col=col_idx, max_col=col_idx):
            for cell in row:
                if cell.value:
                    max_len = max(max_len, len(str(cell.value)))
        ws.column_dimensions[get_column_letter(col_idx)].width = min(max_len + 2, 50)
    ws.freeze_panes = "A2"
    wb.save(output_path)
    print(f"Saved {len(data)} rows to {output_path}")
```

## Pandas DataFrame to XLSX

```python
import pandas as pd

def df_to_xlsx(df: pd.DataFrame, output_path: str, sheet_name: str = "Data"):
    """Write DataFrame to formatted XLSX."""
    with pd.ExcelWriter(output_path, engine="openpyxl") as writer:
        df.to_excel(writer, index=False, sheet_name=sheet_name)
        ws = writer.sheets[sheet_name]
        from openpyxl.styles import Font, Alignment
        from openpyxl.utils import get_column_letter
        for col_idx, col_name in enumerate(df.columns, 1):
            cell = ws.cell(row=1, column=col_idx)
            cell.font = Font(bold=True)
            cell.alignment = Alignment(horizontal="center")
            max_len = max(len(str(col_name)), df[col_name].astype(str).str.len().max() or 0)
            ws.column_dimensions[get_column_letter(col_idx)].width = min(max_len + 2, 50)
        ws.freeze_panes = "A2"
```

## CSV to XLSX Conversion

```python
import csv

def csv_to_xlsx(csv_path: str, xlsx_path: str, sheet_name: str = "Data"):
    """Convert CSV to formatted XLSX."""
    with open(csv_path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        data = list(reader)
    create_xlsx(data, xlsx_path, sheet_name)
```

## Job Directory Setup

```python
import os
import json
from datetime import datetime, timezone

def init_job_dir(base_path: str, slug: str, job_meta: dict) -> str:
    """Create job directory structure and write job.json."""
    job_dir = os.path.join(base_path, slug)
    os.makedirs(os.path.join(job_dir, "output"), exist_ok=True)
    job_meta.setdefault("created_at", datetime.now(timezone.utc).isoformat())
    job_meta.setdefault("status", "active")
    with open(os.path.join(job_dir, "job.json"), "w") as f:
        json.dump(job_meta, f, indent=2)
    return job_dir
```

## Logging Setup

```python
import logging
import os

def setup_logging(output_dir: str, name: str = "scraper"):
    """File + console logger with timestamps."""
    os.makedirs(output_dir, exist_ok=True)
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    fmt = logging.Formatter("%(asctime)s [%(levelname)s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
    fh = logging.FileHandler(os.path.join(output_dir, f"{name}.log"))
    fh.setFormatter(fmt)
    logger.addHandler(fh)
    ch = logging.StreamHandler()
    ch.setFormatter(fmt)
    logger.addHandler(ch)
    return logger
```
