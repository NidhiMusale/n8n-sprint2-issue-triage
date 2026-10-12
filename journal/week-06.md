# Week 6 Build Journal

> Paste this into the course journal template if it has a required layout; the content below covers the build, problems, and the extensibility observation.

## What I built
A self-hosted n8n workflow that receives new GitHub issues through GitHub CLI forwarding, classifies them by title, records them in a ledger Data Table, writes a route-log row per category, and marks a ledger row `failed` if the route-log write fails.

## What worked
- The duplicate guard (ledger lookup before any route-log write) behaved correctly on a replayed delivery.
- The opened-action gate ignored edit and delete events.
- The failure path produced a readable `failed` ledger row with no route-log row.

## Problems and how I fixed them
- **Docker and WSL hung** ("WSL is unresponsive"). Fixed by shutting down WSL and restarting Docker Desktop; my data survived because it lives in a volume.
- **Ledger rows full of Null values.** The Set route nodes had **Include Other Input Fields** turned off, so they dropped every field except `category` and `route_name`. One node also had its field name and value swapped. Turning the option on and correcting the fields fixed it.
- **Everything routed to "other"; Help issues never routed.** The Classify title regex had doubled backslashes (`\\b`), and Rule 3 used "is equal to" instead of "matches regex". Retyping the patterns and changing the operation fixed it.
- **Failure test condition.** After the insert, `title_lower` was no longer available, so the Intentional failure test? node reads it from **Normalize issue** instead.
- **GitHub Redeliver returned 401** for the temporary CLI webhook. I replayed the original payload to the local webhook with `curl` and documented the substitution.

## What I'd do differently
Test each node's output as I built it instead of only at the end, and keep the tutorial's exact node names and settings from the start.

## Extensibility observation
Adding a new category (for example `security`) would take one new Switch rule, one new Set route node with its category and route name, and two connections from that node into the existing duplicate-guard nodes; nothing downstream changes because the ledger and route-log use the same fields for every category. Sending a route to a real destination such as Slack or email would mean replacing or following the **Record internal route** node, and the existing error output already routes failures to **Update route-log failed**, so a failed notification would still leave a visible `failed` ledger row. The workflow's weak point for extension is that retries are manual: a failed row is not retried automatically on redelivery because the ledger row already exists.

## Reflection
TODO: This sprint taught me that most of the work in an automation is checking what each step actually outputs, not just wiring nodes together. My workflow looked finished long before it worked, because settings like "Include Other Input Fields" and a wrong regex operation quietly broke the data. I'm now much more comfortable reading execution data to find where a workflow goes wrong, and I'd test each node as I build it next time. I'm less sure about running this reliably over time, since it depends on my laptop, Docker, and a terminal staying open.
