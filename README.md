# analyst-archetype

The **Analyst** archetype bundle for [protoAgent](https://github.com/protoLabsAI/protoAgent):
an analyst that answers questions from **your own data files**. Point it at a folder of CSV,
TSV, Parquet, JSON, Excel or SQLite files and ask a question. It reads the schema, writes
read-only SQL, and answers with **one live chart and a one-line takeaway**.

**It is read-only, and it reads only the folders you allow.** It never writes to your data
files, makes no network calls, and can't widen its own folder list.

## What an answer looks like

> **Saturday sells the most: $2,351 in average daily revenue, 38.6% above the overall daily average.**
> *(a live bar chart of average revenue by weekday, in the Artifact panel)*
> Assumed: Jul 1 – Sep 30 2026 (92 days). "Sells the most" means average revenue per day;
> each weekday appears 13 times (Wednesday 14), so averages are the fair comparison.
> Source: `analyst-data/coffee_daily_sales.csv`

(A real answer from a synthetic coffee-shop dataset, on `claude-sonnet-5-5`.)

The default path for every question: **question → schema → query → one chart + a one-line
takeaway.**

## The rules it keeps

These are written into the persona (protoAgent's `config/soul-presets/analyst.md`):

1. **It never invents a number.** Every figure comes from a query it ran, derived ones
   (percentages, ratios) included. If the data can't answer the question, it says so and
   says what's missing.
2. **It cites the source file** (and the table or sheet) for every answer.
3. **It says what it assumed**: the date range, how it defined the metric, which rows it
   dropped and why.
4. **It keeps answers short**: one chart, the takeaway, the assumptions line, the source.
5. **It only reads the folders you allowed**, and it never tries to change that setting.

## What's inside

| plugin | source | role |
|---|---|---|
| `data` | [data-plugin](https://github.com/protoLabsAI/data-plugin) (Data Analyst) | Seven `data_*` tools on an embedded, read-only DuckDB (connect, sources, schema, profile, query, chart, export) and the skills `exploring-a-dataset` and `building-a-chart` |
| `artifact` | builtin | Renders `data_chart`'s live Vega-Lite charts, themed to the console |
| `notes` | builtin (on by default) | Metric definitions and findings between sessions |

The persona is protoAgent's `analyst` soul preset. It ships with core, and this bundle names
it, so there is one copy to keep correct.

**Not included:** dashboards, scheduled reports, databases over the network, coding
delegates, a browser. It answers questions; it doesn't build things.

## Create an agent

**Core floor: protoAgent ≥ 0.192.0** on the hub. The data plugin declares
`min_protoagent_version: 0.192.0` (the `vega-lite` artifact kind its charts render through),
and the loader refuses it on older cores. Bundles can't enforce a floor themselves, so it's
stated here.

### From the picker (once the archetype is listed)

The `analyst` row is in protoAgent's archetype catalog but **held**, so the picker doesn't
serve it yet. Once it's listed: **Fleet ▸ New agent ▸ Analyst**, optionally pick a **Data folder**
on the set-up step (the agent has its own folder either way), then **Create**.

Until then, there's a console route on any ≥ 0.192.0 hub: **Settings ▸ Plugins ▸ Install
from URL** → `https://github.com/protoLabsAI/analyst-archetype`. An installed bundle with an
`archetype:` block registers itself as a picker card. If the hub's core predates the
`analyst` preset, paste the persona into the set-up step's **Advanced ▸ Persona** field.

### From the API

This is the exact body the picker sends. The persona goes in inline, so it doesn't matter
whether the hub's core ships the preset yet:

```bash
curl -fsSL https://raw.githubusercontent.com/protoLabsAI/protoAgent/main/config/soul-presets/analyst.md -o analyst.md

jq -n --rawfile soul analyst.md '{
  name: "analyst",
  bundle: "https://github.com/protoLabsAI/analyst-archetype",
  soul: $soul,
  requires_tools: ["data_query", "data_chart"],
  config_inputs: { "data.data_dirs": "/absolute/path/to/your/data" }
}' | curl -s -X POST http://127.0.0.1:7870/api/fleet \
      -H 'content-type: application/json' \
      ${PROTOAGENT_TOKEN:+-H "authorization: Bearer $PROTOAGENT_TOKEN"} \
      -d @-
```

`7870` is the default instance; use your hub's port. The member inherits the hub's model
connection (`inherit_config: true` is the default).

### Direct install onto an existing agent

```
python -m server plugin install https://github.com/protoLabsAI/analyst-archetype
```

Then enable the suggested list (`data, artifact, notes`) and set the data folder (below).

## First run

Every Analyst has **its own data folder**, `<agent workspace>/data` (data-plugin ≥ 0.1.4).
It's created when the plugin loads and is always readable. On the desktop app it's
`~/Library/Application Support/studio.protolabs.protoagent/workspaces/<id>/workspace/data`.
Drop CSV, Excel, Parquet, JSON or SQLite files into it and ask. You don't need to configure
anything.

So the set-up step's **Data folder** field is **optional**. Fill it in only to read data
where it already lives. If nothing is there yet, the agent's first answer tells you to drop
files into your data folder (it shows the path, from `data_sources`) or add folders in:

> **Settings ▸ Plugins ▸ Data Analyst ▸ Data folders**

The value is one or more absolute folders (comma- or newline-separated). It's
**operator-only**: the agent's `set_config` refuses it, and the persona never asks to change
it. So is **Use the default data folder**, which turns the built-in folder off. Credential
files, your home directory as a whole and the rest of the agent's own home are refused even
when listed. The default folder is the one exception inside the agent home.

Excel workbooks need `openpyxl` on the host; the agent says how to install it when it meets
one. Then ask a question: *"Which weekday sells the most?"*

## Pin lifecycle (ADR 0049)

External members are pinned to release tags, and each pin is a **floor**: installs take the
newest *compatible* release (caret semantics, so for 0.x the minor is the boundary).
`scripts/verify_bundle.py`, run by `.github/workflows/verify-bundle.yml` on every PR and
weekly, installs this manifest into a scratch agent on a fresh protoAgent checkout, loads
every member, probes each declared console view, and checks the capability contract
(`requires_tools`). `scripts/check_bundle_updates.py` opens a bump PR only for an
out-of-range release.

### Pin-bump PR lifecycle

The `bump` job (weekly, on dispatch, and on a member's `member-released` dispatch) reuses
**one** `bump-pins` branch and PR, rewritten wholesale each run, so don't hand-edit it. A PR
opened with the repository `GITHUB_TOKEN` never auto-starts its `pull_request` run: GitHub
holds it as `action_required` until a maintainer approves it. The job detects that, labels
and comments on the PR, and fails, so an unapproved candidate turns the schedule red instead
of going stale unnoticed.

While this bundle depends on core that hasn't been released, the repo variable
`PROTOAGENT_REF` (e.g. `refs/pull/<n>/head`) points `verify` at that ref. Delete it once the
change lands.

Run the verify locally from a protoAgent checkout:

```
uv run --no-sync python /path/to/analyst-archetype/scripts/verify_bundle.py /path/to/analyst-archetype
```

## License

MIT
