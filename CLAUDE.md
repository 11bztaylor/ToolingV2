# ToolingV2 — Session Primer
# READ THIS FIRST in any new session working on this tool.
# Last updated: 2026-03-17

---

## What This Is

**ADX Schema Builder V2** — a standalone, browser-based, single-HTML-file tool that
generates ADX KQL management commands from uploaded data + a schema pack system.

Lives at: `C:\Users\11bzt\Documents\WorldofClaude\NetskopetoADX\N2Av2\ToolingV2\`

### Key differences from V1 (Tooling/)
- **Generic + schema packs**: Netskope is pack #1, future data sources register as new packs
- **Three output generators** (Table Builder, RLS Builder, Retention)
- **Column mapper** with dropdown assignment (simpler than V1's drag-and-drop)
- **Table Builder** with 2-phase output (staging → typed), flatten fields system, preset tiers, Sentinel/LA preset
- **Retention Builder** with category presets + per-table overrides

---

## File Structure

```
ToolingV2/
├── CLAUDE.md                          ← This file (session primer)
├── index.html                         ← The assembled application (container architecture)
└── TestingBeforeProd/
    ├── DataMapping/
    │   ├── DataMapping_PreprodVer.html
    │   └── sample_agency_groups.csv
    ├── RLSBuilder/
    │   └── RLSBuilder_PreprodVer.html
    ├── Retentiontab/
    │   └── Retention_PreprodVer.html
    └── TableBuilder/
        └── TableBuilder_PreprodVer.html
```

### Dev Workflow: Preprod → Merge
Each tab has a standalone `*_PreprodVer.html` file in its subfolder:
1. **Edit** the preprod file (no app shell overhead, instant reload)
2. **Test** in browser — each file is self-contained with shared design tokens
3. **Merge** the stable panel HTML + JS into the matching section of `index.html`
4. Each tab is clearly delimited in index.html with `// ═══ TAB: Name ═══` banner comments

Preprod files carry their own minimal state + schema pack data for standalone testing.
The real shared state lives in `index.html`.

## Hard Constraints

- NO CDN. Everything inline in index.html.
- NO server. Must work as file:// in Chrome/Edge.
- NO localStorage. Session-only state. Save/Load via JSON file.
- NO iframes. Single HTML file with JS tab switcher.
- NO npm, build step, or bundler.

---

## Architecture (Container Pattern)

### index.html internal structure
```
<style>  — All CSS (app shell + all tab styles)
<body>   — App shell + 4 tab HTML containers
<script> — Organized JS:
  SHARED CORE:  DEFAULT_FLATTEN_FIELDS, TIER_META, SCHEMA_PACKS, state, utilities
  SHARED SHELL: showTab, bindConfig, session save/load
  TAB: Data & Mapping   ← all its functions
  TAB: Table Builder     ← all its functions
  TAB: RLS Builder       ← all its functions
  TAB: Retention         ← all its functions
  init()
```

### Layout
- Header bar: Cluster URI, Database, Tenant ID, Schema Pack selector
- Sidebar: 4 nav tabs + session save/load buttons
- Main: tab content area

### Tabs
1. **Data & Mapping** — file upload/paste, column mapper with dropdown assignment
2. **Table Builder** — 2-phase (staging → typed), field flattening, tier presets (rec/ext/notrec), Sentinel/LA preset, per-field enable/disable, Copy All
3. **RLS Builder** — baked-literal RLS function + per-table policy application, reads from Data & Mapping
4. **Retention** — category presets + per-table overrides + recoverability toggle

### State Schema
```js
var state = {
  globalConfig: { clusterUri, database, tenantId },
  activePack: "netskope",
  // Data & Mapping owns:
  files: [{ id, name, columns, rows, rowCount }],
  mapping: {
    groupName:    { fileId, column },  // AAD group for RLS
    domainFilter: { fileId, column },  // Agency domain for RLS
    tenantId:     { fileId, column },  // Tenant ID per row
    customField:  { fileId, column }   // Optional extra
  },
  // Table Builder owns:
  tableBuilder: {
    phase,                // "staging" | "typed"
    selectedTables,       // table names checked in tree
    emailFieldOverride,   // override pack's userEmailField
    addAgencyFilter,      // bool — toggle AgencyDomain extraction
    flattenFields,        // bool — toggle field flattening
    fields: [],           // [{cat, raw, col, type, tier, sentinel, enabled}]
    activePreset          // "rec" | "ext" | "notrec" | "sentinel" | "all" | "none" | "custom"
  },
  // RLS Builder owns:
  rls: { rlsFunction, applyPolicies, selectedTables, mappingPairs: [{group, domain, tenantId, displayName, enabled}] },
  // Retention owns:
  retention: { staging, events, alerts, recoverability, overrides: {} }
};
```

### Schema Packs
Defined in `SCHEMA_PACKS` object. Each pack provides:
- `stagingTable`, `stagingSchema`, `mappingName`, `mappingDef`
- `tables[]` with `{ name, stream, cat }` for each typed table
- `baseTypedSchema` (base columns — AgencyDomain + flatten fields added dynamically)
- `folder`, `userEmailField`

Netskope pack: 1 staging + 21 typed tables (8 events + 13 alerts).

### DEFAULT_FLATTEN_FIELDS
57 field definitions with:
- `cat`: "common" | "event" | "alert"
- `tier`: "rec" | "ext" | "notrec" — drives preset selection
- `sentinel`: bool — flags fields needed for Sentinel entity mapping, ASIM, analytics rules
- `enabled`: bool — toggled by presets or individually

### Design Tokens
CSS `:root` block. Dark theme, GitHub-adjacent palette.

---

## Build Status

| Component | Status |
|---|---|
| CSS / design tokens | ✅ Done |
| Container architecture (index.html) | ✅ Done |
| App shell (header, nav, tabs, session save/load) | ✅ Done |
| Data & Mapping (upload, parse, column mapping) | ✅ Done |
| Table Builder (2-phase, flatten, presets, Sentinel, Copy All) | ✅ Done |
| RLS Builder (mapping pairs, agency grouping, display names, Copy All) | ✅ Done |
| Retention Builder (presets, overrides, recoverability, Copy All) | ✅ Done |
| Session save/load (with full UI state restoration) | ✅ Done |
| Preprod folder structure (per-tab subfolders) | ✅ Done |

---

## Implementation Notes & Design Decisions

### Function Wiring — Key Patterns

**Namespaced functions per tab** — Table Builder functions are prefixed (`initTBChecklists`, `onTBTableCheck`, `renderTBKqlBlocks`) to avoid collision with RLS's own checklist functions (`initRLSChecklists`, `onRLSTableCheck`). Tree toggle functions (`toggleTreeGroup`, `toggleTreeSub`) are shared since they're generic DOM operations.

**`toggleSwitch(el, key)`** — shared across RLS and Retention. Handles three keys: `rlsFunction`, `applyPolicies` (both `state.rls`), and `recoverability` (`state.retention`). Also manages the RLS dependency warning visibility.

**`toggleOption(key, el)`** — Table Builder only. Handles `addAgencyFilter` and `flattenFields` toggles. Uses teal-colored toggle switches (`.toggle-switch.on` = `var(--color-teal)`) vs RLS/Retention which use the default info color.

### Table Builder — Preset System

**Tier presets are cumulative:**
- "Recommended" → enables only `tier:"rec"` fields
- "Rec + Extended" → enables `tier:"rec"` AND `tier:"ext"`
- "All Fields" → enables everything including `tier:"notrec"`

**Sentinel/LA preset is cross-cutting:** enables all fields with `sentinel:true` regardless of tier. This is a workload-based selection (what Sentinel entity mapping / ASIM / analytics rules need) not a quality-based one.

**Auto-detection:** `getPresetState()` reverse-detects which preset matches the current enabled pattern, so the active pill highlights correctly even after individual toggles.

**Per-section All/None:** Each field section header (Common, Event-Only, Alert-Only) has All/None links via `toggleSectionFields(cat, enabled)`. Using these clears the preset to "custom".

### Table Builder — Field Tier Assignments (why)

**Recommended (22 fields):** Core SOC/hunting fields. user, UPN, srcip, dstip, app, category, access_method, traffic_type, policy, severity, severity_level, url (event), domain, alert_name, alert_type, action, activity, md5, sha256, from_user, to_user, url (alert).

**Extended (32 fields):** Valuable for deeper analysis but not always needed. Includes endpoint fields (hostname, os, browser, device), network fields (protocol, dstport, numbytes, bytes), DLP fields (dlp_profile, exposure, file_name/type/size), malware fields (malware_name/type, malsite_category), and misc (appcategory, ccl, org_unit, useragent, ur_normalized, page, site, conn_duration, ssl_decrypt_policy, object/type, risk_level, shared_with).

**Not Recommended (3 fields):** Netskope internal correlation IDs only (instance_id, request_id, transaction_id). High cardinality, no SOC value. Available in RawData if needed.

**Sentinel-flagged (34 fields):** Fields that map to Sentinel entities (Account, IP, Host, URL, DNS, FileHash, File, CloudApp), ASIM schemas (NetworkSession, activity mapping), incident severity mapping, DLP workbooks, or malware analytics rules.

### Table Builder — KQL Output

**2-phase system** (not the old 3-phase):
- Phase 1 (Staging): `.create table` + `.create ingestion json mapping` in one block
- Phase 2 (Typed): Per-table blocks, each containing 3 sequential commands: Step 1 `.create table` (with dynamic schema from `getTypedSchema`), Step 2 `.create-or-alter function` (with `| extend` for AgencyDomain + flatten fields, using `castExpr` for type-safe extraction), Step 3 `.alter table policy update` (attaches update policy)

**`getTypedSchema(cat)`** — builds schema dynamically from `baseTypedSchema` + optional AgencyDomain + enabled flatten fields for that category. This is why `SCHEMA_PACKS` uses `baseTypedSchema` instead of a static `typedSchema`.

**Copy All** — `_lastRenderedTBBlocks` stores the blocks from the most recent render. `copyAllKql()` joins them with double newlines. Button only appears when 2+ blocks exist.

### RLS Builder — Architecture

**Mapping pairs model:** RLS no longer reads directly from `getMappedTriples()`. Instead, `loadRLSMappingPairs(data)` loads mapping data into `state.rls.mappingPairs[]` — each pair is `{group, domain, tenantId, displayName, enabled}`.

**Agency → Group Mappings card:** Pairs are displayed grouped by agency domain with per-pair checkboxes, per-agency All/None, and Expand All / Collapse All. Only enabled pairs are included in the generated KQL.

**`getRlsGroupId(pair)`** — returns the AAD group identifier for KQL. Uses `displayName` as the identifier (not the UUID), appends `;tenantId` when available. This is because `current_principal_is_member_of()` only accepts **string literals** — variables, `let` bindings, and datatable lookups silently fail.

**KQL output format:** Agency-grouped OR clauses with comment headers per agency section, inline display name comments, and a human-readable mapping reference block at the top of the function.

**Copy All** — `_lastRlsBlocks` tracks rendered blocks. `copyAllRlsKql()` joins with `////////` separator banners between each command.

**Test data:** `RLS_TEST_MAPPING_DATA` (8 sample mappings across 5 agencies) is included in index.html for testing. In production, this will be populated from uploaded CSV via the Data & Mapping tab.

### Data & Mapping → Tab Data Flow

Currently RLS Builder loads test data directly via `loadRLSMappingPairs()`. Table Builder and Retention are self-contained — they read from `SCHEMA_PACKS` directly. This is intentional: Table Builder generates table creation commands from the pack definition, not from uploaded data.

**Available data access functions** (defined in Data & Mapping section):
- `getMappedValues(role)` — returns array of values for a mapping role
- `getMappedPairs(roleA, roleB)` — returns paired arrays
- `getMappedTriples(roleA, roleB, roleC)` — returns tripled arrays

**Known issue:** `getMappedTriples` returns `{a,b,c}` objects but some old code expected arrays. New code bypasses this by using `loadRLSMappingPairs()` directly. Will be fixed when hub architecture lands.

### Retention Builder — Architecture

**Category presets:** Staging (7d/10d/14d/30d/custom), Events (90d/180d/365d/730d/custom), Alerts (180d/365d/730d/1095d/custom). Dropdown labels include human-readable equivalents like "365 days (1 yr)".

**Per-table overrides:** Each table row shows a category badge (STG/EVT/ALT), table name, override input, and **resolved value** — the actual retention that will be applied (override if set, otherwise category preset). Resolved values update in real-time when presets change.

**`resolveRet(tableName, cat)`** — single resolution function: checks overrides first, then category preset, then falls back to "365d".

**ADX timespan format:** `d` = days, `h` = hours, `m` = minutes. No month shorthand — users must convert (6 months = `180d`). Full format `90.00:00:00` also supported. Info alert at top of tab explains this.

**Staging table always included** — `Netskope_Raw` gets its own retention command, separate from events/alerts.

**Copy All** — `_lastRetBlocks` tracks rendered blocks. `copyAllRetKql()` joins with separator banners.

### Session Save/Load

`exportSession()` serializes the entire `state` object as JSON v2. `importSession()` restores it and fully rebuilds UI:
- Config fields (cluster, database, tenantId) restored
- Table Builder: fields re-initialized from `DEFAULT_FLATTEN_FIELDS` if missing, checkbox selections restored from saved `selectedTables`, phase/toggles/email override synced
- RLS: toggle visual states restored, mapping pairs re-rendered, table selections restored
- Retention: preset dropdowns restored (including custom values), recoverability toggle synced
- All outputs regenerated after restore

### UX Decisions

- **Card order in Table Builder:** Table Selection → Field Configuration → Generated KQL. Natural top-down decision flow.
- **Table Options merged into Field Configuration:** AgencyDomain toggle and Flatten toggle live at the top of the Field Config card, not in a separate card. They directly control field output so they should be co-located.
- **Table Selection auto-collapses** when all tables are checked (the default). Count badge shows "21/21 selected" so the state is visible without expanding.
- **Field section headers show enabled/total counts** and have per-section All/None links (same pattern as the table tree).

---

## What's Next

### Phase 1: Hub Architecture (next iteration)
Design and implement a registration pattern:

**Tab Registry** — each tab declares:
- name, icon, nav position
- column dependencies (what it needs from Data & Mapping)
- its HTML template + init/render/generate functions

**Data & Mapping as Hub** — column mapping grid auto-generated from what tabs have registered.
New tab = declare column needs, they auto-appear in mapping UI.

**Pack Registry** — schema packs as pluggable containers:
- Each pack registers tables, flatten fields, staging config
- Adding CrowdStrike/Defender/etc = push a new pack object
- All tabs adapt to whatever pack is selected

### Backlog
- Custom schema pack UI (let user define tables manually when pack = "custom")
- Testing with real Netskope mapping CSV (replace test data in RLS)
- Additional schema packs (CrowdStrike, Defender, etc.)
- Wire Data & Mapping → RLS (replace test data with live `getMappedTriples` + display name column)
- Fix `getMappedTriples` to return arrays instead of `{a,b,c}` objects
- Data & Mapping hub architecture (auto-generate mapping slots from tab registrations)
