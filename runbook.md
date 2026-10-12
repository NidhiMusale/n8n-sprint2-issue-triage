# Runbook: GitHub Issue Triage (n8n Sprint 2)

## Purpose
Operate and troubleshoot the local n8n workflow **Sprint 2 - GitHub Issue Triage**, which accepts newly opened GitHub issues, classifies them, records them in `github_issue_ledger`, and writes one row per new issue to `github_issue_route_log`.
- Action: use this runbook whenever the workflow is started, tested, or shut down.
- Expected result: every opened issue has exactly one ledger row and, on success, exactly one route-log row.
- Evidence: [test_evidence.md](test_evidence.md).

## Normal operation
- Action: run `docker compose up -d`; confirm n8n loads at `http://localhost:5678` and the workflow shows **Published**. In a separate window run `gh webhook forward --repo=NidhiMusale/n8n-sprint2-issue-triage --events=issues --url=http://localhost:5678/webhook/sprint2-issue-triage` and wait for "Forwarding Webhook events from GitHub...". Open a new issue in the repository.
- Expected result: within about 10 seconds a new successful execution appears. The ledger gets one row with `route_log_status = recorded`, and the route log gets one row with the matching route name (`bug-triage`, `feature-requests`, `support`, or `general-triage`). Non-opened events such as edits or deletions stop at **Opened action?** and write nothing.
- Evidence: Tests 1-4 and the gate test in `test_evidence.md`.

## Duplicate delivery
- Action: if the same `opened` delivery arrives again, nothing needs to be done. The workflow looks up `issue_key` in the ledger first.
- Expected result: the existing row's `last_seen_at` is updated, **Insert pending ledger row** does not run, and no second route-log row is created. Check by confirming the ledger and route-log row counts are unchanged.
- Evidence: the replay test in `test_evidence.md` (execution #26). GitHub's own Redeliver button returned HTTP 401 for the temporary CLI forwarding hook, so the original Test 1 payload was replayed to the local webhook with `curl`.

## Route-log failure
- Action: if a route-log write fails, the **Record internal route** error output sends the item to **Update route-log failed**. To test it deliberately, open an issue whose title contains `intentional route-log failure test`.
- Expected result: the ledger row is kept with `route_log_status = failed` and the `error_note` "Route-log write failed; review the route-log node configuration and execution details." No route-log row is written for that `issue_key`.
- Evidence: Test 5 in `test_evidence.md` (execution #28).

## Recovery action
1. Find rows in `github_issue_ledger` where `route_log_status = failed`.
2. Open the failed execution in n8n and read the error on **Record internal route** (for example a missing column or a renamed table). Fix the cause and republish.
3. Because the ledger row already exists, a redelivery only takes the update path and will not retry the route-log write. To finish the record, either (a) add the missing row to `github_issue_route_log` by hand (same `issue_key`, `category`, `route_name`, and a `recorded_at` time) and set the ledger row to `recorded` with a blank `error_note`, or (b) delete the failed ledger row and trigger a fresh `opened` event.
- Expected result: the issue has one ledger row marked `recorded` and one route-log row.
- Evidence: described here; the failed state is captured in Test 5. Automatic retry is not implemented (see the last section).

## Safe shutdown
1. After all evidence is captured and exported, open the workflow in n8n and unpublish/deactivate it.
2. In the `gh webhook forward` window press **Ctrl+C**.
3. In GitHub, open the repository's **Settings > Webhooks** and confirm no active webhook points at the forwarder (remove or disable it if one is listed).
4. Optionally stop n8n with `docker compose stop` (do not use `down -v`).
- Expected result: new issues no longer create executions.
- Evidence / shutdown record: TODO: fill in date, time, and what you confirmed (workflow unpublished, forwarding stopped, webhook removed or disabled).

## What this prototype does not test
- Automatic retry of a failed route-log write (recovery is manual, see above).
- Real external destinations such as Slack or email; routing is recorded only in an n8n Data Table.
- Webhook signature verification; the local endpoint trusts requests sent to its URL.
- Concurrent deliveries of the same issue arriving at the same instant, or issue volumes above a few test issues.
- GitHub's own Redeliver button against the CLI forwarder (it returned 401); a local `curl` replay of the original payload was used instead.
- Issue titles in languages other than English or titles with several keywords (the first matching rule wins).
