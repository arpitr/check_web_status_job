# Domain Health Checker

A Python script that reads domain CSVs, checks the HTTP response of each URL (with cache-busting), prints a colour-coded CLI report, and writes a per-file `report_<sitename>.csv` summary.

---

## Dependencies

**Python 3.9+** is required.

Install the single third-party dependency with:

```bash
pip install requests
```

All other modules (`csv`, `os`, `sys`, `uuid`, `time`, `argparse`, `datetime`, `pathlib`, `urllib`) are part of the Python standard library.

---

## Input CSV Format

Each input CSV must contain a `Domain` column (case-insensitive). Additional columns such as `Region`, `Country`, and `Language` are optional but will be preserved in the output report if present.

**Example:**

```csv
Region,Country,Language,Domain
Americas,United States,English,https://www.xyz.com/
Europe / Middle East / Africa,Germany,Deutsch,https://www.xyz.com/de/
Asia Pacific,Australia,English,https://www.xyz.com/au/
```

The following CSV files are included alongside this script:


| File                             | Site                     |
| -------------------------------- | ------------------------ |
| `xyz.csv`                        | www.xyz.com              |
| `pqr.csv`                        | www.pqr.com              |


---

## Usage

### Check a single CSV

```bash
python check_domains.py xyz.csv
```

### Check multiple CSVs

```bash
python check_domains.py xyz.csv pqr.csv
```

### Check all CSVs at once (glob)

```bash
python check_domains.py *.csv
```

### Optional: slow down requests (be polite to servers)

```bash
python check_domains.py *.csv --delay 1.0
```


| Flag              | Default | Description                                  |
| ----------------- | ------- | -------------------------------------------- |
| `--delay SECONDS` | `0.3`   | Pause between each HTTP request (in seconds) |

---

## How Cache-Busting Works

Every URL is appended with a unique query parameter before the request is sent:

```
https://www.xyz.com/de/  →  https://www.xyz.com/de/?_cb=a3f91bc042e44d8f9c12e7b3d5f60182
```

The `_cb` value is a randomly generated UUID hex string, ensuring each request bypasses CDN and reverse-proxy caches. The original URL is stored separately in the report for readability.

---

## CLI Output

### Live progress (per URL)

As each URL is checked, a progress line is printed in real time:

```
  [  1/32] Checking Brazil / Portuguese …  ✓  HTTP 200
  [  2/32] Checking Canada / English …     ✓  HTTP 200
  [  3/32] Checking Germany / Deutsch …    ✗  HTTP 404 Not Found
  [  4/32] Checking Austria / Deutsch …    ✓  HTTP 200
```

- ✓ green  = successful response (HTTP < 400)
- ✗ red    = error response or connection failure

### Summary table (per file)

After all URLs in a file are checked, a colour-coded table is printed:

```
Domain                                                  Code  Error?
------------------------------------------------------------------------
https://www.xyz.com/br/                          200  no
https://www.xyz.com/en-ca/                       200  no
https://www.xyz.com/de/                          404  YES
https://www.xyz.com/at/                          200  no
------------------------------------------------------------------------
```

### Error detail block

Any URL that returned an error is listed in full detail below the table:

```
⚠  1 error(s) detected:

  URL   : https://www.xyz.com/de/
  Tested: https://www.xyz.com/de/?_cb=a3f91bc042e44d8f9c12e7b3d5f60182
  Error : HTTP 404 Not Found
```

### Executive summary line

```
✓ No errors found.
  File total: 32 URLs — 32 OK, 0 error(s)
```

or when errors exist:

```
⚠  1 error(s) detected.
  File total: 32 URLs — 31 OK, 1 error(s)
```

### Grand total (multi-file runs only)

When more than one CSV is passed, a combined summary is printed at the end:

```
══════════════════════════════════════════════════════════════════════════════
  OVERALL SUMMARY — All Files
══════════════════════════════════════════════════════════════════════════════

  [combined table for all files]

  Grand total: 155 URLs — 153 OK, 2 error(s)

  📄 Combined report → report_all_sites.csv
```

---

## CSV Output Artifacts

### Per-file report

For each input CSV, a report file is written in the **same directory** as the input:

```
xyz.csv   →   report_xyz.csv
pqr.csv   →   report_pqr.csv
```

### Combined report (multi-file runs)

When multiple CSVs are passed, an additional combined report is written:

```
report_all_sites.csv
```

### Report structure

Each report CSV is divided into two sections:

#### Section 1 — Executive Summary

The first 8 rows contain a plain key/value summary for quick review at the top of the file:

```
EXECUTIVE SUMMARY,
Site,             XYZ
Generated,        2026-04-22 14:30:00
Total URLs,       32
OK,               32
Errors,           0
Status,           No errors found.
Summary,          File total: 32 URLs — 32 OK, 0 error(s)
```

#### Section 2 — Detail Table

A blank row separates the summary from the detail table. Each row represents one URL:

```csv
Region,Country,Language,Domain,Response Code,Error?,Error Detail
Americas,Brazil,Portuguese,https://www.xyz.com/br/,200,No,
Americas,Canada,English,https://www.xyz.com/en-ca/,200,No,
Europe / Middle East / Africa,Germany,Deutsch,https://www.xyz.com/de/,404,YES,HTTP 404 Not Found
Asia Pacific,Australia,English,https://www.xyz.com/au/,200,No,
```


| Column                            | Description                                                                  |
| --------------------------------- | ---------------------------------------------------------------------------- |
| `Region` / `Country` / `Language` | Carried over from the source CSV (if present)                                |
| `Domain`                          | The original URL (without cache-bust param)                                  |
| `Response Code`                   | HTTP status code returned, or`N/A` if a connection error occurred            |
| `Error?`                          | `No` for successful responses, `YES` for HTTP 4xx/5xx or connection failures |
| `Error Detail`                    | The specific error message (blank if no error)                               |

---

## Notes

- **Colour output** is automatically enabled when running in a terminal that supports ANSI codes, and disabled when output is piped or redirected to a file. Set `FORCE_COLOR=1` to override.
- **Redirects** are followed automatically (`allow_redirects=True`). The final response code after all redirects is what gets recorded.
- **Timeouts** default to 15 seconds per request. Connection errors, SSL errors, and timeouts are all treated as errors and reported with a descriptive message.
- **CDN/bot protection** — some sites use Cloudflare or similar CDN protection that may return `403 Forbidden` for requests from cloud/server IP ranges. Run the script from a regular office or home network for accurate results.
