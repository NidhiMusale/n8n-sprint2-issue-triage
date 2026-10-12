# n8n Sprint 2: GitHub Issue Triage Automation

## Purpose
A self-hosted, event-driven n8n workflow that receives **new GitHub issues** from this public repository, classifies each one from its title with deterministic rules (no LLM), saves a durable triage record, writes a category-specific internal route-log entry, and handles a duplicate webhook delivery without creating a second record or route-log row.

## Repository
- URL: https://github.com/NidhiMusale/n8n-sprint2-issue-triage
- Visibility: **Public** (contains only the harmless course test issues; no credentials or secrets)

## Services
| Layer | Service |
| --- | --- |
| Event source | This GitHub repository (`issues` events) |
| Event delivery | GitHub CLI with the `gh webhook` extension, forwarding to `http://localhost:5678/webhook/sprint2-issue-triage` |
| Automation host | Self-hosted n8n in Docker at `http://localhost:5678` |
| State and routing evidence | Two n8n Data Tables: `github_issue_ledger` and `github_issue_route_log` |

## Node sequence
1. **GitHub issue webhook** (POST, path `sprint2-issue-triage`)
2. **Opened action?** passes only `{{ $json.body.action }}` equal to `opened`
3. **Normalize issue** builds the fields listed below
4. **Classify title** (Switch, Rules mode) sends the item to one of four outputs
5. **Set bug route / Set feature route / Set help route / Set other route** add `category` and `route_name`
6. Duplicate guard, run in parallel from every Set route node:
   - **If ledger row exists** then **Update existing last seen** (updates `last_seen_at` only; nothing else runs)
   - **If ledger row does not exist** then **Insert pending ledger row** (`route_log_status = pending`)
7. **Intentional failure test?** checks whether the title contains `intentional route-log failure test`
   - false: **Record internal route** (insert into `github_issue_route_log`) then **Update route-log recorded** (`route_log_status = recorded`)
   - true: **Simulate route-log failure** (Code node that throws an error)
8. **Update route-log failed** sets `route_log_status = failed` and a safe `error_note`. It is reached from the error output of both **Record internal route** and **Simulate route-log failure**.

## Field mapping (Normalize issue)
| Field | Expression |
| --- | --- |
| repository_full_name | `{{ $json.body.repository.full_name }}` |
| issue_number | `{{ $json.body.issue.number }}` |
| issue_key | `{{ $json.body.repository.full_name + "#" + $json.body.issue.number }}` |
| issue_url | `{{ $json.body.issue.html_url }}` |
| title | `{{ $json.body.issue.title }}` |
| title_lower | `{{ $json.body.issue.title.toLowerCase() }}` |
| reporter | `{{ $json.body.issue.user.login }}` |
| opened_at | `{{ $json.body.issue.created_at }}` |
| last_seen_at | `{{ $now.toISO() }}` |

## Data tables
- `github_issue_ledger`: issue_key, issue_url, title, category, reporter, opened_at, last_seen_at, route_log_status, route_name, error_note
- `github_issue_route_log`: issue_key, category, route_name, recorded_at

## Exact classification rules
All rules test `{{ $json.title_lower }}` and are evaluated in this order.

| Order | Match | Category | Route name |
| --- | --- | --- | --- |
| 1 | matches regex `\b(bug\|error)\b` | bug | bug-triage |
| 2 | matches regex `\b(feature\|request)\b` | feature | feature-requests |
| 3 | matches regex `\b(help\|question)\b` | help | support |
| 4 | fallback: does **not** match regex `\b(bug\|error\|feature\|request\|help\|question)\b` | other | general-triage |

(In the regex, `\|` above is only Markdown table escaping; the real pattern uses a plain `|`.) Rule 4 is implemented as "does not match" the combined keyword pattern, which makes it equivalent to a fallback output.

## Setup notes
1. Start n8n: `docker compose up -d`, then open `http://localhost:5678`. The workflow data lives in a Docker volume; never run `docker compose down -v`.
2. Create the two Data Tables with the columns above, import or build the workflow, and click **Publish**.
3. Install and authenticate GitHub CLI (`gh auth login`), then `gh extension install cli/gh-webhook`.
4. Forward events: `gh webhook forward --repo=NidhiMusale/n8n-sprint2-issue-triage --events=issues --url=http://localhost:5678/webhook/sprint2-issue-triage` and leave the window open.
5. `sprint2_github_issue_triage.json` is a submission-safe export for reference. It is evidence, not the working workflow, and should not be re-imported.

## Evidence
See [test_evidence.md](test_evidence.md) for the four issue types, the duplicate-delivery test, the opened-action gate test, and the failed route-log test. Operating instructions are in [runbook.md](runbook.md); the Zapier comparison is in [own_vs_rent.md](own_vs_rent.md); the build journal is in [journals/week-06.md](journals/week-06.md).
