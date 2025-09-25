# MRBP Validate – Excel vs PDF (OCR Khmer), PDF Table Extract, Excel→XML, Templates, SSO

A FastAPI web tool to:

- **Validate & match** soft **Excel** vs **scanned PDF** (OCR Khmer + English).
- **Extract table(s)** from PDFs (native or scanned).
- **Convert Excel → XML** (no XML header) using **mapping templates**.
- **Template Manager** (CRUD) for reusable XML mappings.
- **Better matches** using IDs, dates, fuzzy names, tolerance.
- **Downloadable Excel report** (Summary, Line Comparison, Raw PDF lines).
- **AD/SSO** with Microsoft Entra ID (Azure AD).

---
## 1) Quick start

### Local (Python)
```bash
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:api --reload --port 8000
```
Open: http://localhost:8000/ui

> OCR needs **Tesseract** with `eng` and `khm` trained data installed (Debian/Ubuntu: `sudo apt-get install tesseract-ocr tesseract-ocr-eng tesseract-ocr-khm`).

### Docker
```bash
docker build -t mrbp-validate:latest .
docker run --rm -p 8000:8000   -e DB_URL=sqlite:///./data.db   -e SESSION_SECRET=change-me   -e TENANT_ID=<tenant> -e CLIENT_ID=<client_id> -e CLIENT_SECRET=<client_secret>   -e REDIRECT_URI=http://localhost:8000/auth/callback   mrbp-validate:latest
```

---
## 2) Environment

Create `.env` or pass as env vars:
```
DB_URL=sqlite:///./data.db
SESSION_SECRET=<random>
TENANT_ID=<your-tenant-id>          # optional, required for SSO
CLIENT_ID=<app-client-id>           # optional, required for SSO
CLIENT_SECRET=<app-client-secret>   # optional, required for SSO
REDIRECT_URI=http://localhost:8000/auth/callback
```

- If SSO not configured, endpoints fallback to anonymous access (you can protect by replacing dependencies with `get_current_user`).

---
## 3) Features & Endpoints

### Validate (Excel vs PDF)
- **POST** `/validate` → JSON result (totals comparison, line matches).
- **POST** `/validate/report` → downloads an Excel report.

**Form-Data:**
- `excel_file`: Excel file
- `pdf_file`: PDF (scanned or native)
- `currency`: `AUTO`|`KHR`|`USD` (default: `AUTO`)
- `tolerance_abs`: integer absolute tolerance (e.g., 0)

**Logic:**
- Excel per-row amounts are detected (largest KHR-like integer per row).
- PDF text extracted by PyMuPDF; fallback OCR (eng+khm) if sparse.
- Totals compared; line level matching via amount, optional ID (e.g., `0700-...`), fuzzy name, date window.

### Extract tables (simple)
- **POST** `/extract-table` → JSON with raw PDF lines (can be piped to CSV).

### Convert Excel → XML (no header)
- **POST** `/convert-xml` (returns XML text without header)
- `mapping_json`: JSON mapping e.g. `{ "Currency":"G", "Amount":"H", "Payer":"I" }`
- `fixed_json`: JSON of fixed fields e.g. `{ "Currency": "USD" }`

### Template Manager (CRUD)
- **POST** `/templates` → create
- **GET** `/templates` → list
- **GET** `/templates/{id}` → retrieve
- **PUT** `/templates/{id}` → update
- **DELETE** `/templates/{id}` → delete

Save a template once (per partner). Then reuse `mapping_json` and `fixed_json` for XML conversion.

---
## 4) UI usage (http://localhost:8000/ui)
- **Validate**: upload Excel + PDF, choose currency/tolerance, click **Validate** or **Validate + Download Report**.
- **Extract**: upload PDF, click **Extract** to preview parsed lines.
- **XML**: upload Excel, paste **mapping_json** (and optional **fixed_json**), click **Convert**.
- **Templates**: create/list templates directly from the page.
- **Login**: click **Login with Microsoft (SSO)** if SSO configured.

---
## 5) Matching rules (configurable)
- **Amounts**: exact match within `tolerance_abs`.
- **IDs**: pattern like `0700-01-737919-2-6` recognized and boosts confidence.
- **Names**: fuzzy match (RapidFuzz) token set ratio; default threshold 85.
- **Dates**: ISO parsing; accept ±5 days (configurable).

Tune values in `app/services/matcher.py`.

---
## 6) Report format
- **Summary**: totals, status, tolerances.
- **LineComparison**: per-line outcomes (MATCH/NOT_FOUND), scores.
- **Duplicates**: reserved sheet.
- **RawPDF**: original parsed lines for debugging/audit.

---
## 7) Development
- Code lives under `app/` with services split by concern.
- Add advanced table extraction (Camelot/Tabula) if needed.
- Replace anonymous dependencies with `get_current_user` to enforce login.

---
## 8) GitHub
```bash
git init
git add .
git commit -m "feat: template manager, better matching, report, SSO"
git branch -M main
git remote add origin https://github.com/<your-org>/<your-repo>.git
git push -u origin main
```

---
## 9) Disclaimer
This starter focuses on deterministic validations for KHR/USD with OCR fallbacks. Real-world scanned PDFs vary—adjust OCR DPI, add layout models, and enrich keys (branch, account, month) as needed.
