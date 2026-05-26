---
name: apex-form-submit
description: Submits an Oracle APEX page via POST to wwv_flow.accept. Handles ck checksum values, hidden fields, LOV (List of Values) popups, and multi-step wizards. Requires a session bundle from /apex-session-bootstrap.
triggers:
  - "submit apex form"
  - "post apex page"
  - "execute apex action"
---

# APEX Form Submit

Posts a page submission to `wwv_flow.accept`. This is the primary endpoint for any APEX action — saving a record, advancing a wizard, calling an Application Process.

## Inputs

- `session_bundle`: from `/apex-session-bootstrap`
- `page_id`: target page to submit (e.g. `21` for a wizard, `2` for a tab)
- `request`: APEX request name — common values:
  - `SUBMIT` (save form)
  - `CREATE` (insert)
  - `SAVE` (update)
  - `DELETE` (delete row)
  - `APPLICATION_PROCESS=<name>` (call a server-side process — most flexible)
- `field_values`: dict mapping `Pn_FIELD_NAME` → value
- Optional `lov_fields`: list of fields that use popup LOV (require name=field hidden input, not id=field)

## Build the POST body

The body is `application/x-www-form-urlencoded` with these required fields:

```
p_flow_id=<app_id>
p_flow_step_id=<page_id>
p_instance=<p_instance from bundle>
p_page_submission_id=<fresh — get from a page probe>
p_request=<request>
p_md5_checksum=<may be empty in 20.x; required in 21+>
p_arg_names=<field1>
p_arg_values=<value1>
p_arg_checksums=<ck1 from bundle>
p_arg_names=<field2>
...
```

Note: `p_arg_names`, `p_arg_values`, `p_arg_checksums` are **repeated** per field. Order matters and they must align by index.

### LOV popup fields (gotcha)

For LOV (popup) fields, APEX expects `name=Pn_FIELD` as a hidden input with the **real value** (not the display label). If you set it via `id=Pn_FIELD`, APEX uses the display label which breaks downstream lookups.

```python
# Wrong (uses display label):
fields["P21_VENDOR"] = "Acme Corp"

# Right (uses code/id):
fields["P21_VENDOR_HIDDEN"] = "12345"  # hidden=ck-protected
```

### Numeric fields with locale

APEX 20.x has a bug where `por_imp_vent=0` (0 IVA) silently fails for vendors with a fiscal profile. Workaround: never POST raw 0; if you need 0, POST empty string, then UPDATE post-create.

## curl example

```bash
curl -c cookies.txt -b cookies.txt \
  -X POST "${APEX_BASE_URL}wwv_flow.accept" \
  -d "p_flow_id=${APP_ID}" \
  -d "p_flow_step_id=21" \
  -d "p_instance=${P_INSTANCE}" \
  -d "p_request=APPLICATION_PROCESS=AGREGAR_MASIVO_PLSQL" \
  -d "p_arg_names=P21_PROYECTO" \
  -d "p_arg_values=MAIN" \
  -d "p_arg_checksums=${CK_PROYECTO}" \
  -d "p_arg_names=P21_LOTE" \
  -d "p_arg_values=L01" \
  -d "p_arg_checksums=${CK_LOTE}" \
  -o response.html
```

## Detecting success

Parse `response.html` for:
- `APEX_ERROR_MESSAGE` div → action failed; extract message
- `APEX_SUCCESS` div or 302 redirect to next page → action succeeded
- `<script>apex.navigation.redirect(...)</script>` → success with redirect

If neither marker is present, the action may have triggered a server-side dialog — re-probe the page to see new state.

## DELETE pattern

```bash
curl -X POST "${APEX_BASE_URL}wwv_flow.accept" \
  -d "p_request=DELETE_${PK_FIELD}" \
  -d "p_arg_names=${PK_FIELD}" \
  -d "p_arg_values=${ROW_ID}" \
  ...
```

## UPDATE pattern

Same as SUBMIT but with `p_request=SAVE` and include ALL ck values (not just changed fields) — APEX revalidates them all.

## Application Process pattern (most flexible)

For server-side logic that doesn't map to a CRUD operation:

```bash
curl ... \
  -d "p_request=APPLICATION_PROCESS=GENERAR_PDF_CONTRATO" \
  -d "p_arg_names=P21_CONTRATO_ID" \
  -d "p_arg_values=1358" \
  ...
```

The response will be either a redirect (download), JSON payload, or a refreshed page.

## See also

- `/apex-export-pdf` — wraps the Application Process pattern for PDF exports.
- [docs/APEX-PATTERNS.md](../../docs/APEX-PATTERNS.md) — deeper notes on session reverse-engineering.
