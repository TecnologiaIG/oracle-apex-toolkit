---
name: apex-export-pdf
description: Triggers an Oracle APEX Application Process (typically GENERAR_PDF_*) and downloads the resulting PDF report. Handles redirect chain and content-disposition parsing.
triggers:
  - "export apex pdf"
  - "download apex report"
  - "generate apex pdf"
---

# APEX Export PDF

Calls an APEX Application Process that generates a PDF (a common pattern in enterprise APEX apps — invoice, contract, report exports) and downloads the resulting file.

## Inputs

- `session_bundle`: from `/apex-session-bootstrap`
- `app_process_name`: e.g. `GENERAR_PDF_CONTRATO`, `EXPORT_INVOICE_PDF`
- `page_id`: page where the Application Process is registered
- `params`: dict of `Pn_FIELD` values needed by the process (e.g. `P21_CONTRATO_ID=1358`)
- `output_path`: local path to save the PDF

## Flow

### 1. POST to trigger the process

```bash
curl -c cookies.txt -b cookies.txt -L \
  -X POST "${APEX_BASE_URL}wwv_flow.accept" \
  -d "p_flow_id=${APP_ID}" \
  -d "p_flow_step_id=${PAGE_ID}" \
  -d "p_instance=${P_INSTANCE}" \
  -d "p_request=APPLICATION_PROCESS=${APP_PROCESS_NAME}" \
  -d "p_arg_names=${FIELD_1}" \
  -d "p_arg_values=${VALUE_1}" \
  -d "p_arg_checksums=${CK_1}" \
  -o response_or_pdf.bin
```

The `-L` flag is critical — APEX often returns a 302 redirect to the PDF endpoint, which curl follows automatically.

### 2. Detect PDF vs HTML response

APEX may return:
- A PDF directly (Content-Type: application/pdf)
- A redirect to a CDN/blob URL
- An HTML page with embedded `<iframe src="..pdf">` (rare)
- An error page with `APEX_ERROR_MESSAGE`

Check the first bytes:

```bash
head -c 4 response_or_pdf.bin
# Should start with: %PDF
```

If `%PDF`:

```bash
mv response_or_pdf.bin "${OUTPUT_PATH}"
```

If HTML:
- Parse for `APEX_ERROR_MESSAGE` — if present, report and abort.
- Parse for embedded PDF URL — if present, fetch separately with curl.

### 3. Validate the PDF

```bash
file "${OUTPUT_PATH}"
# Should report: PDF document, version X.Y
```

If `file` says "ASCII text" or similar, the download failed silently — re-run with verbose curl (`-v`) and inspect headers.

## Common APEX Application Process names

These are conventions, not guaranteed:

| App Process | Typical use |
|---|---|
| `GENERAR_PDF_*` | Spanish-language apps (Latin America) |
| `EXPORT_*` | English-language apps |
| `DOWNLOAD_*` | Variation |
| `IR_DOWNLOAD` | Interactive Report download (build-in APEX) |

The name is defined by the app developer — check the APEX Builder under Shared Components → Application Processes.

## Headers to inspect

```bash
curl -v ... 2>&1 | grep -i "content-disposition\|content-type"
# content-disposition: attachment; filename="contrato_1358.pdf"
# content-type: application/pdf
```

The `filename` from Content-Disposition is your hint for the right `output_path` if you didn't pre-decide.

## Gotchas

- **Long-running PDFs**: large reports can take 30-60s. Set curl `--max-time 120` to avoid premature timeout.
- **Memory**: APEX PDF generation is server-side BI Publisher; if the server is under load, it returns 503. Retry with backoff.
- **Authentication tokens in URL**: some APEX deploys put a token in the PDF URL — preserve it when following redirects (curl `-L` handles this).
- **Anti-bot challenge**: if APEX is behind Cloudflare Turnstile or similar, this skill won't work — you need a headful browser (Playwright).

## Example: download a contract PDF

```
/apex-export-pdf
```

Inputs:
- App Process: `GENERAR_PDF_CONTRATO`
- Page: `21`
- Params: `P21_CONTRATO_ID=1358`
- Output: `/tmp/contrato_1358.pdf`

Result: a valid PDF saved locally, ready to attach to a ticket/email.
