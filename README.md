# oracle-apex-toolkit

Reusable patterns for automating Oracle APEX 20.x applications with [Claude Code](https://claude.com/claude-code).

If you've ever needed to programmatically interact with an Oracle APEX app — submit a form, click a wizard step, export a report PDF — you know APEX doesn't have a clean REST API. Everything happens through `wwv_flow.accept` POSTs with session IDs, page item maps, and checksum-validated state.

This plugin packages the patterns that work in practice, distilled from real Oracle APEX 20.2 production usage (where I reverse-engineered IVA update, contract creation, and report export endpoints).

## Skills

| Slash | What it does |
|---|---|
| `/apex-session-bootstrap` | Login to APEX app, capture session ID + protected items, return reusable session bundle |
| `/apex-form-submit` | Submit a page via `wwv_flow.accept` — handles ck validation, LOV popups, hidden fields |
| `/apex-export-pdf` | Trigger an APEX `GENERAR_PDF_*` Application Process and download the resulting PDF |

## Use cases

- Integrating a legacy Oracle ERP (Oracle E-Business Suite, custom APEX apps) with Claude Code without a REST layer.
- Building agents that drive APEX forms to create records, attach files, or export reports.
- Auditing APEX usage by replaying captured sessions.

## Installation

### As a Claude Code plugin

Add to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "oracle-apex-toolkit": {
      "source": {
        "source": "github",
        "repo": "TecnologiaIG/oracle-apex-toolkit"
      }
    }
  }
}
```

Then in Claude Code:

```
/plugin install oracle-apex-toolkit@oracle-apex-toolkit
```

### Manual

Clone the repo and copy the skills under your `~/.claude/skills/`.

## Prerequisites

- An Oracle APEX 20.x application you have credentials for.
- `curl` (system) — used inside the skills for HTTP calls.
- Optional: `python3` if you want to use the parser helpers in `docs/APEX-PATTERNS.md`.

## Quickstart

```
/apex-session-bootstrap
```

When prompted:
- APEX base URL: `https://your-tenant.example.com/ords/your-workspace/`
- App ID: `2240` (or whatever your app is)
- Username / password

The skill returns a session bundle (JSON) you can store and reuse for follow-up calls.

## What this is NOT

- **Not** a REST API wrapper. APEX doesn't have one. This is a thin layer over `wwv_flow.accept`.
- **Not** a SQL connector. For SQL, use [Oracle ORDS](https://www.oracle.com/database/technologies/appdev/rest.html) or `cx_Oracle`.
- **Not** Oracle Cloud (OCI). This targets on-prem or hybrid APEX deployments.

## Origin

Extracted from production patterns at a Guatemalan real-estate group running Oracle APEX 20.2 (O4BI ERP — 6 companies, 12 projects). The patterns are agnostic — no app-specific schemas leak through.

## License

MIT. See [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. The patterns work for APEX 20.x; adjustments for 21+ likely required (page item naming conventions changed slightly).
