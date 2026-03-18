# ADX Schema Builder V2 — User Guide

## Overview

ADX Schema Builder V2 is a browser-based tool that generates KQL management commands for Azure Data Explorer. It creates staging and typed tables, configures Row-Level Security (RLS) by agency, and sets retention policies—all in one place.

**How to open it:** Just open `index.html` in Chrome or Edge. No installation, no server, no internet needed.

---

## Getting Started

### Step 1: Fill in the Header Bar

When you first open the tool, you'll see a header bar at the top with three fields:

- **Cluster URI** — The ADX cluster URL (e.g., `https://myadxcluster.eastus.kusto.windows.net`)
- **Database** — Your target database name (e.g., `SecurityLogs`)
- **Tenant ID** — Your Azure tenant ID (optional, used for per-tenant RLS)

These values are saved in your session and are used in the KQL commands you generate.

### Step 2: Choose a Schema Pack

The **Schema Pack** dropdown (top right) selects which data source you're working with. Currently, only **Netskope N2Av2** is available. Future releases will add more packs for other data sources (CrowdStrike, Defender, etc.).

### Step 3: Save or Load Your Configuration

Before you start, you can:
- **Load Config** — Load a previously saved JSON file to restore all your settings
- **Save Config** — At any time, export your current state as a JSON file. This checkpoints your work.

All tabs and settings are included in the saved configuration.

---

## Tab 1: Data & Mapping

This tab is where you upload or paste data that maps agencies to their AAD groups and domains.

### Upload or Paste Data

You have two options:

1. **Upload a CSV/TSV/JSON file** — Click the upload box and select a file with columns like `Group`, `Domain`, `TenantId`, etc.
2. **Paste data directly** — Paste the raw CSV or JSON text into the text area.

The tool will parse your data and show you:
- The number of rows detected
- A list of column headers

### Assign Columns to Roles

After upload, you'll see a **Column Mapper** with dropdowns. Assign each column to a role:

- **Group Name** — The AAD group identifier (required). Used in RLS to check group membership.
- **Domain Filter** — The agency domain (recommended). Used to extract agency from email or as a grouping label.
- **Tenant ID** — Azure tenant ID for multi-tenant setups. Optional.
- **Custom Field** — Any extra column you want to reference later. Optional.

Once assigned, you'll see a **Mapping Summary** that shows your data grouped by agency domain, with row counts for each group.

---

## Tab 2: Table Builder

This tab generates all the KQL commands needed to create Netskope tables and fields in ADX.

### Understanding Phases

Table Builder works in **two phases**:

- **Phase 1 (Staging)** — Creates the raw staging table and sets up JSON ingestion mapping. Run this first.
- **Phase 2 (Typed Tables)** — For each selected table, creates the typed table schema, adds a transform function (to flatten fields and extract AgencyDomain), and applies an update policy to route data from staging to typed. Run this after Phase 1.

### Step 1: Select Tables

The **Table Selection** card shows all 21 Netskope tables (8 events + 13 alerts) in a collapsible tree grouped by category.

- Check or uncheck tables to include them.
- If you check all tables, the tree auto-collapses to save space.
- The status badge shows "X/21 selected".

### Step 2: Configure Fields

The **Field Configuration** card lets you choose which fields from RawData to flatten into real columns.

#### Presets (Quick Selection)

Use these buttons to quickly enable/disable fields by tier:

- **Recommended** — Core SOC/hunting fields only (22 fields). Start here for most setups.
- **Rec + Extended** — Recommended + deeper analysis fields (54 total). Good for investigation-heavy teams.
- **All Fields** — Every available field (57 total). Use only if you need everything.
- **Sentinel/LA** — Fields required for Sentinel entity mapping, ASIM schemas, and Log Analytics workbooks (34 fields). Use if you're feeding Sentinel/SOAR.
- **None** — Disable all fields (minimal table, just raw data).
- **Custom** — Clear the preset and allow individual field toggles.

#### Manual Field Selection

Each field has a checkbox. You can:
- Toggle individual fields on or off.
- Use **All / None** links in each section header (Common, Event-Only, Alert-Only) to toggle an entire section at once.
- Selecting individual fields automatically switches the preset to "Custom".

Fields marked with an **LA** badge are needed for Sentinel workloads.

#### Table Options

At the top of the field configuration:

- **Add Agency Filter** — Extracts the agency domain from your data and adds an `AgencyDomain` column to every table. Recommended for multi-agency setups.
- **Flatten Fields** — Enables field flattening. If unchecked, fields are not extracted and you get a minimal table with just RawData.

### Step 3: Copy and Run Commands

Once you've selected tables and fields, scroll down to the **Generated KQL** section. You'll see KQL blocks ready to copy.

- **Phase 1** — One block with staging table creation and ingestion mapping.
- **Phase 2** — Per-table blocks (one for each selected table), each with 3 steps: Create table, Create/Alter function, Alter update policy.

**Copy All** button (appears when 2+ blocks exist) — Copies all KQL with separators, ready to paste.

#### Running in ADX Web UI

1. Open [Azure Data Explorer Web UI](https://dataexplorer.azure.com).
2. Select your cluster and database.
3. Copy one KQL block from the tool.
4. Paste it into the query editor and **Run** (or press Shift+Enter).
5. Wait for success, then copy the next block and repeat.

**Important:** ADX only executes one management command at a time. You cannot paste all blocks and run them together. Run each block individually.

---

## Tab 3: RLS Builder

This tab generates Row-Level Security (RLS) that restricts data by agency domain. Users only see rows where their AAD group matches the agency.

### Understanding RLS

RLS in ADX is enforced through:
1. A function that checks if the current user is a member of an authorized AAD group
2. A table policy that applies this function to filter rows

**Example:** If user `alice@company.com` is a member of group `agency-a-soc`, they'll only see rows tagged with the agency-a domain.

### Step 1: Load Mapping Data

The tool automatically loads the data from your **Data & Mapping** tab. If you haven't mapped data yet, go back to Tab 1 and upload/paste.

### Step 2: Review Agency → Group Mappings

The **Agency → Group Mappings** card shows:
- Groups grouped by agency domain
- A checkbox next to each group to include/exclude from the RLS function
- An **All / None** link per agency to toggle all groups in that agency
- **Expand All / Collapse All** buttons to manage large lists

Only **checked groups** will be included in the generated RLS function.

### Step 3: Configure RLS Output

Two toggles control RLS generation:

- **Generate RLS Function** — Check to include Block 1 (the RLS function itself).
- **Apply RLS Policy** — Check to include Block 2 (applies the function to tables).

**Important:** Always run Block 1 (the function) before Block 2 (the policy). If you have existing policies, you may only need Block 2.

### Step 4: Select Table Scope

Check which tables should have the RLS policy applied. Typically, all typed tables get the same policy, but you can customize per-table if needed.

### Step 5: Copy and Run Commands

The **Generated KQL** section shows:

- **Block 1** — The RLS function definition. Include check `current_principal_is_member_of()` calls with group names baked directly into the function.
- **Block 2** — Applies the policy to your selected tables.

**Copy All** button joins blocks with separator banners.

Run Block 1 first, then Block 2 in ADX Web UI (one at a time).

#### Technical Note: Why Group Names Are Baked In

`current_principal_is_member_of()` only accepts string literals (e.g., `"agency-a-soc"`). It does not accept variables, `let` bindings, or datatable lookups. This is why the tool generates the group names directly in the function rather than referencing a lookup table.

---

## Tab 4: Retention

This tab generates retention policies for each table category (Staging, Events, Alerts).

### Step 1: Set Category Presets

The **Category Presets** section has three dropdowns (one each for Staging, Events, Alerts). Choose from:

- **90 days (3 mo)**
- **180 days (6 mo)**
- **365 days (1 yr)**
- **Custom** — Type your own timespan

#### Custom Timespan Format

If you choose **Custom**, use ADX timespan notation:

- `d` — days (e.g., `180d` for 6 months)
- `h` — hours (e.g., `24h` for 1 day)
- `m` — minutes (e.g., `1440m` for 1 day)

Do **not** use month shorthand (ADX doesn't have it). For 6 months, use `180d`.

### Step 2: Per-Table Overrides (Optional)

If you need a different retention for specific tables:

1. Open the **Per-Table Overrides** card.
2. Select a table from the dropdown.
3. Choose a preset or enter a custom timespan.
4. Click **Set Override** — it will appear in the list below.

Overrides take precedence over category presets for that table.

### Step 3: Recoverability Toggle

Check **Allow Recoverability** if you want to enable recovery of deleted data. This adds storage overhead but allows 30 days of recovery after soft delete. Leave unchecked for cost savings.

### Step 4: Copy and Run Commands

The **Generated KQL** section shows retention commands grouped by category and table.

**Copy All** button copies all commands with separators. Run each command individually in ADX Web UI.

---

## Common Workflows

### Workflow 1: Set Up Netskope Tables from Scratch

1. **Open the tool** and fill in Cluster URI, Database, Tenant ID in the header.
2. **Go to Tab 2 (Table Builder)**.
3. **Select tables** — check all 21 Netskope tables (or a subset).
4. **Choose a field preset** — "Recommended" for most teams, or "Sentinel/LA" if you're feeding Sentinel.
5. **Enable Add Agency Filter** if you have multiple agencies.
6. **Copy Phase 1 KQL** and run it in ADX Web UI to create the staging table.
7. **Copy Phase 2 KQL** and run each table block one at a time.

### Workflow 2: Add RLS for a New Agency

1. **Go to Tab 1 (Data & Mapping)** and upload an updated CSV that includes the new agency group.
2. **Go to Tab 3 (RLS Builder)** — the new group will automatically appear in the mappings.
3. **Check the new group** to include it in RLS.
4. **Copy Block 1 (RLS Function)** and run it in ADX Web UI. This updates the function with the new group.
5. **Block 2 (Apply Policy)** — if the policy is already applied to tables, you don't need to re-run it. If tables are new, run Block 2.

### Workflow 3: Change Retention for One Table

1. **Go to Tab 4 (Retention)**.
2. **Set the category presets** for Staging, Events, Alerts as a baseline.
3. **Open Per-Table Overrides** and select the table that needs different retention.
4. **Choose a preset or custom timespan** and click **Set Override**.
5. **Copy the generated KQL** for just that table override and run in ADX Web UI.

### Workflow 4: Checkpoint and Restore Your Configuration

1. **At any time**, click **Save Config** to export your current state as a JSON file. This includes all tabs, table selections, field toggles, RLS mappings, and retention settings.
2. **Later**, click **Load Config** and select the JSON file to restore everything.

---

## Tips & Best Practices

### General

- **Always run one KQL command at a time** in ADX Web UI. ADX management commands are not batched.
- **Save Config before making big changes.** If you need to experiment with different table sets or field presets, save first.
- **The tool is completely offline.** Your data and settings never leave your browser.
- **When you switch Schema Packs**, all tabs automatically regenerate to reflect the new pack's tables and fields.

### Table Builder

- Start with the **Recommended** preset. Add extended fields only if your team needs deeper analysis.
- Use **Sentinel/LA preset** if you plan to feed data into Sentinel, Log Analytics workbooks, or SOAR playbooks.
- **Enable Add Agency Filter** if you have multiple agencies. This makes downstream filtering and RLS much easier.

### RLS Builder

- Test RLS with a small group first. Run the RLS function, then apply it to one table, then verify access with a test account.
- If you need to add a new agency group, **reload your data in Tab 1 first** so that the new group appears in the mappings.
- Remember: `current_principal_is_member_of()` does a live AAD lookup. Group membership changes take effect immediately.

### Retention

- Set category defaults in **Category Presets** to enforce a baseline.
- Use **Per-Table Overrides** sparingly. Per-table policies add operational complexity.
- For cost optimization, set Staging retention to 90 days or less. Long-term history belongs in typed tables.

---

## Troubleshooting

### "No columns detected" when uploading a file

- Ensure your CSV/TSV/JSON has a header row.
- If pasting, make sure the format is valid (comma-separated, tab-separated, or JSON).

### RLS mappings not appearing

- Go back to Tab 1 and ensure your data is uploaded and columns are assigned.
- Check that **Group Name** and **Domain Filter** are both assigned in the column mapper.

### Can't copy KQL from a tab

- Ensure you've filled in required fields (table selection, field presets, etc.).
- If the tab is blank, click **Generate** or **Render** if visible, or adjust your selections to trigger output.

### Changes to my data don't show in RLS

- The tool reads from **Data & Mapping** automatically. If you upload new data in Tab 1, go to Tab 3 and refresh (close/reopen or click a checkbox to retrigger the mapping load).

### One command runs but the next one fails

- Check that you're running commands in the correct order (Phase 1 before Phase 2, RLS function before RLS policy).
- Ensure your Cluster URI and Database name are correct in the header.
- Check ADX permissions — you may need **Admin** or **Ingestor** role.

---

## Keyboard Shortcuts

- **Ctrl+C** — Copy text from the KQL output area (standard browser copy).
- **Tab navigation** — Click the tab names in the left sidebar to switch between tabs.

---

## Questions or Feedback?

This tool is actively maintained. For bug reports, feature requests, or questions, reach out to your ADX administrator.

---

**Last updated:** 2026-03-17
