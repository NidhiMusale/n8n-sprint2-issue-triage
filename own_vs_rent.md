# Own vs. rent: self-hosted n8n vs. Zapier

## What was compared
The Sprint 2 workflow: a GitHub `issues.opened` event is classified by title, checked against a ledger, written to an internal route log, and marked failed if the route-log write fails. The alternative is the same logic built as a Zap in Zapier.

## Side-by-side
| Area | Self-hosted n8n (what I built) | Zapier (rented) |
| --- | --- | --- |
| Trigger | GitHub CLI forwards events to a localhost webhook; I had to keep a terminal running | Native GitHub "New Issue" trigger, no tunnel or CLI needed |
| Classification | Switch node with regex rules, visible and testable | Paths or Filter plus Formatter steps; similar logic |
| Ledger and route log | Two n8n Data Tables | Zapier Tables, or a Google Sheet |
| Duplicate guard | If Row Exists / Does Not Exist branches I wired myself | Find-record step plus a Path or filter; built-in "find or create" can shorten it |
| Failure handling | Error outputs routed to a node that marks the row `failed` | Zap error handling and replay from Zap history; less control over writing a failure state |
| Hosting and uptime | My laptop and Docker; down whenever the laptop sleeps (this caused missed events and 401s during testing) | Zapier runs it continuously |
| Setup time | Long (Docker, WSL, CLI forwarding, debugging) | Short, mostly point and click |
| Control and data | Data stays on my machine; full access to logic and executions | Data passes through a vendor; limited to supported actions |

## Cost analysis (assumptions stated)
Prices below came from public 2026 pricing summaries and change often, so verify them on zapier.com/pricing before relying on them.
- **Zapier:** a Free plan exists with about 100 tasks per month, but it is limited to two-step Zaps. A multi-step Zap like this one needs the Professional plan, listed at roughly $20/month billed annually or about $30/month billed monthly, for 750 tasks. Zapier counts each successful action step as a task; triggers and filters are free.
- **Estimated tasks per issue (assumption):** about 4 for a new issue (look up record, create ledger row, create route-log row, update status) and about 2 for a duplicate delivery (look up, update). My test run had 5 new issues and 1 duplicate, which is roughly 22 tasks. A small repository with 100 new issues a month is roughly 400 tasks, inside the 750-task plan; 1,000 issues a month is roughly 4,000 tasks and would need a much higher tier.
- **Self-hosted n8n:** the software was $0 for this sprint. Real costs are the machine it runs on (here, an existing laptop, so no new spend), electricity, and my time. If run continuously, a small cloud server would be needed (a typical small VPS costs on the order of a few to ten-plus dollars a month; this is an assumption, not a quote).
- **Time:** the build took many hours of setup and debugging. At even a modest hourly value, that dwarfs one or two months of a Zapier subscription for a low-volume workflow.

## Conclusion
For a low volume of issues and a team that wants reliability without maintenance, renting Zapier is cheaper in total effort and easier to keep running. Self-hosting n8n is better when data must stay in-house, when volume is high enough that per-task pricing grows quickly, or when custom logic and visible failure handling matter. For this course prototype, building in n8n was the better learning path, but I would not rely on a laptop-hosted workflow for production use.
