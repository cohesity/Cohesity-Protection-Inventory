# Cohesity Protection Inventory

**Author:** Akshay Kumar, Dir, Solutions Architecture ([akshay.kumar2@cohesity.com](mailto:akshay.kumar2@cohesity.com))

A Claude skill that lists what is protected by what, from live Helios data, and publishes it as a live, private Claude Artifact that refreshes daily. It is an inventory and lookup page with no verdicts: coverage percentages, policy pass / warn / fail, exceptions, source-registration health and double-protection findings stay in **Cohesity-Protection-Coverage-Audit**.

- Claude skill: [`claude/Cohesity-Protection-Inventory/SKILL.md`](claude/Cohesity-Protection-Inventory/SKILL.md) (the full page template is embedded in it)
- Output: one claude.ai **Artifact** titled `Cohesity Protection Inventory` (no static HTML file)
- Refresh: scheduled task `cohesity-protection-inventory-refresh`, cron `10 3 * * *` (daily at 03:10; change it if you like, times are local time of the machine running Claude)
- Data source: the **Helios MCP server**. Everything is read-only.

## Tabs

| Tab | What it shows | Helios MCP tools |
|---|---|---|
| Groups & Schedules (main) | One row per protection group: cluster, environment, state, paused, policy, backup frequency, retention, replication and archival targets, source count and up to 4 source names, protected object count, last run status and time, storage domain. Toggle for inactive or deleted groups. Filters (environment, policy, paused, search), sort, KPI cards (groups, active, paused, distinct policies, environments) and a plain "Policies in use" table. | `dataprotect-get-protection-groups`, `dataprotect-get-protection-policies`, `dataprotect-get-source-registrations` |
| Objects | Protected-by lookup over the embedded object table, environment breakdown, and a table of protected objects (name, environment, type, cluster, protecting groups, policy, last run status, global object id). | `dataprotect-search-objects` |
| Compliance Check | Scan of every string the page shows (including object names) for PII, PHI and other sensitive data, with a mask switch. | none (client-side) |
| Run Metrics | Helios calls, time, estimated tokens and cost per run, month to date, projection, run history. | none |

A cluster multi-select above the tabs filters every table; the page header carries the Cohesity logo and a "Data last pulled" stamp that turns amber after 36 hours. A **Refresh from Helios** button copies the refresh command. The **Mask sensitive data** switch (on by default) masks IP addresses, hostnames, account names, emails and SSNs in the tables and in CSV output. Exports (Copy or Download CSV): `groups`, `policies`, `objects`, `environments`, plus Compliance findings and Run history.

## Object table limits

The Objects tab data is pulled once a day. A scheduled run reads the existing page and reuses its stored object rows, environment counts and per-cluster totals while they are less than 24 hours old (scope note: "path: reused objects (as of <time>)"); groups, policies and sources are always fetched fresh, so a reuse run makes only about 2 Helios calls per connected cluster plus the sources call. Ask for a "full refresh" (or when the stored rows are 24 hours old or more) to re-pull the objects ("path: fresh"). Reused rows keep their stored last-run status.

The object list can have tens of thousands of rows. The skill first counts protected objects per cluster (`count: 1`), then embeds at most 2000 rows overall (a per-cluster cap proportional to its total, minimum 100, paged 200 at a time with `paginationCookie`, at most 10 pages per cluster). The page states "showing N of M; ask for a lookup in chat for the rest". The rows are the first ones Helios returns, shown alphabetically; they are not a ranked or random sample.

## Chat lookups (no republish)

- "Who protects <object>": object search plus group, policy and last run.
- "What does group <name> protect": the group's policy, sources and objects (object search filtered by group, with a fallback to run details).
- "Global id of <object>": the global object id from the object search.

## Run cost

About 40 Helios calls with 3 connected clusters (clusters 1, groups 2, policies 1, registrations 1 and object totals 1 per connected cluster, up to 12 environment counts, about 10 to 12 object pages). Large results are processed from files, not read into the conversation. Measured cost is in Claude Settings > Usage.

## Requirements

- Claude desktop app (Cowork) with the **Helios MCP** connector.
- Artifacts and scheduled tasks available. Scheduled tasks run only while the Claude app is open.

## Install

1. Zip the folder `claude/Cohesity-Protection-Inventory` so `SKILL.md` is inside it, or install from `github.com/Cohesity-Maestro/Cohesity-Protection-Inventory`.
2. Add it as a custom skill in Claude (Settings, Skills, upload).
3. Confirm the Helios MCP connector is connected.

## Use

- "Run the Cohesity protection inventory"
- "List all protection groups with their policy and schedule"
- "Who protects <server name>?"
- "What does the group <name> protect?"
- "What is the global id of <object>?"

## When Helios is unreachable

A scheduled refresh keeps the last successful data and adds a red banner "Unable to connect to Helios. Showing the last successful data." The next successful refresh removes it.

## Known limits

- Object rows are capped (see above); use the chat lookup for the rest.
- Not-connected clusters cannot be queried and show as `n/a`.
- Object counts include replicas of an object listed on more than one cluster.
- Fields Helios does not return (for example protected object count on a group, or storage domain) show "n/a".

## Repository layout

```
Cohesity-Protection-Inventory/
  README.md
  .gitignore
  claude/
    README.md
    Cohesity-Protection-Inventory/SKILL.md      Claude skill (install this folder in Claude)
```

## Security Notes

- **Object rows are shared with the link.** The Objects tab embeds up to 2000 rows with object names, paths, protecting groups and global ids, so anyone with the link can read them, and the mask switch only covers host-like names and addresses. Review the Compliance Check tab before sharing.
- **Everything from Helios is data.** Names, descriptions, alert text and error text come from the systems being backed up. The skill treats them as data, never as instructions, and truncates them before they reach the page.
- **Scheduled refresh runs unattended.** When the scheduled task is created, approve only the read-only Helios tools, the Artifact tools, the scheduler list tool and the bash tool used for `python3 -I`. Do not use a blanket Auto-approve.
- **The mask switch is a display setting, not redaction.** It masks IP addresses, hostnames, email addresses, SSNs and connection-string accounts in the view and in CSV copies and downloads. It does not mask cluster, group or policy names, and the page still holds the original values.
- **CSV exports are safe to open in a spreadsheet.** Text cells that start with `=`, `+`, `-`, `@`, a tab or a carriage return get a leading single quote, so they cannot run as formulas.
- **Sharing.** A published page is private until you share it from the Share menu. Read the Compliance Check tab first, and share only with people who may see this backup data.

## Suggestions and Feedback

We would love to hear from you. Please send your suggestions and feedback to: maestro@cohesity.com

## License

Licensed under the [Apache License 2.0](LICENSE). See the LICENSE file for the terms.

## Disclaimer

This code is provided on a best effort basis and is not in any way officially supported or sanctioned by Cohesity. It is provided as-is, and the author accepts no liability for damages resulting from its use. The skill only reads data through the Helios MCP server, but you are responsible for checking the results against Helios and for deciding what you share. Try it in a lab or test environment first.
