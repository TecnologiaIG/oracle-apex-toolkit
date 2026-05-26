---
name: apex-session-bootstrap
description: Logs into an Oracle APEX 20.x application and captures a reusable session bundle (cookie, p_instance, p_flow_id, p_flow_step_id, ck values). Reusable for subsequent wwv_flow.accept POSTs. Tested on APEX 20.2 and 21.1.
triggers:
  - "apex login"
  - "bootstrap apex session"
  - "get apex session"
---

# APEX Session Bootstrap

Logs into an Oracle APEX 20.x app and returns a structured session bundle. APEX sessions are short-lived (typically 8h) and tied to a browser cookie + state token (`p_instance`). Without the bundle, no `wwv_flow.accept` POST will succeed.

## Inputs

- `apex_base_url`: e.g. `https://your-tenant.example.com/ords/your-workspace/`
- `app_id`: numeric APEX application id (e.g. `2240`)
- `username`, `password`: APEX credentials (NOT database credentials)
- Optional `start_page` (defaults to `1`)

## Flow

### 1. Fetch the login page

```bash
curl -c cookies.txt -b cookies.txt \
  "${APEX_BASE_URL}f?p=${APP_ID}:LOGIN" -o login.html
```

This sets the initial `ORA_WWV_*` cookie. Parse `login.html` for:
- `p_md5_checksum` (hidden field)
- `p_instance` (URL or hidden field)
- `p_flow_id`, `p_flow_step_id`

### 2. POST credentials to wwv_flow.accept

```bash
curl -c cookies.txt -b cookies.txt \
  -X POST "${APEX_BASE_URL}wwv_flow.accept" \
  -d "p_flow_id=${APP_ID}" \
  -d "p_flow_step_id=101" \
  -d "p_instance=${P_INSTANCE}" \
  -d "p_page_submission_id=${PAGE_SUB_ID}" \
  -d "p_request=LOGIN" \
  -d "p_md5_checksum=${MD5}" \
  -d "p_t01=${USERNAME}" \
  -d "p_t02=${PASSWORD}" \
  -o login_response.html
```

### 3. Verify login

Check `login_response.html` for absence of `APEX_ERROR_MESSAGE` div. If present, credentials failed. If absent, APEX redirected to the home page — extract the new `p_instance` for subsequent calls.

### 4. Probe a known page to capture authoritative state

```bash
curl -c cookies.txt -b cookies.txt \
  "${APEX_BASE_URL}f?p=${APP_ID}:${START_PAGE}:${P_INSTANCE}" \
  -o home.html
```

Parse `home.html` for all `p_arg_names[]`, `p_arg_checksums[]` (ck values), and current `p_instance`.

## Output bundle

```json
{
  "base_url": "https://your-tenant.example.com/ords/your-workspace/",
  "app_id": "2240",
  "p_instance": "12345678901234",
  "session_cookie_file": "/tmp/apex_session_abc.cookies",
  "current_page": "1",
  "ck_values": {
    "P1_FIELD_A": "C7A2F...",
    "P1_FIELD_B": "9DE61...",
    "...": "..."
  },
  "expires_at": "2026-MM-DD HH:MM (assume 8h)"
}
```

Persist this bundle to a local file (`~/.apex-sessions/<app_id>.json`) so subsequent calls reuse it.

## Gotchas

- **Trailing slash matters** on `apex_base_url`. Without it, some APEX deploys 302-redirect and break the cookie chain.
- **p_md5_checksum** is required even when the login page doesn't visibly show it — APEX expects it from a hidden input.
- **2FA / SSO**: this skill assumes username+password local auth. For Entra SSO / SAML, use the SSO endpoint URL and intercept the post-redirect callback.
- **CSRF**: APEX 21+ adds a `g_p_widget_action_mod` token to some requests. The bundle should capture it via the probe step.

## Reset / re-bootstrap

If a subsequent `wwv_flow.accept` returns `APEX_ERROR_MESSAGE: "Session expired"`, delete the bundle file and re-run this skill.
