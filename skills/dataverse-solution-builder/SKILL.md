---
name: dataverse-solution-builder
description: "Configure a Microsoft Dynamics 365 / Dataverse environment from a design document — build tables, columns, relationships, plug-ins, a model-driven app, views, forms, a dashboard and demo data via scripted Dataverse Web API calls."
---

# Dataverse solution builder

Turn a solution **design document** (tables, columns, relationships, forms, views, app, dashboard) into a working Microsoft Dynamics 365 / Dataverse configuration. Everything is done by generating **self-contained Python scripts** that the user runs against their environment's Web API. Claude does not need direct access to the org; the user authenticates and runs each script, then pastes the output back.

## When to use

Use when someone wants to stand up or extend a Dataverse/Dynamics 365 solution — custom tables, a model-driven app, forms, views, dashboards, sample data — especially from a written design. Also use for one-off tasks (add a plug-in, recolor choices, fix an app) using the relevant phase below.

## Why scripts (not the maker portal, not a connector)

- The Power Apps **maker portal is flaky to automate** (panels revert, command bars don't render). Scripting the Web API is far more reliable and repeatable.
- The **Dataverse MCP connector applies the default publisher prefix**, so it can't honor a custom prefix; the Web API with a solution header can.
- Scripts are **idempotent and re-runnable**, and the user keeps control of auth.

Deliver **one script per phase**, in order. Each is standard-library-only Python (runs on the python3 that ships with macOS — no pip). Tell the user to run `python3 <script>.py`, sign in with the device code, and paste the output. Fix forward on any error.

## Inputs to confirm before building

1. **Org URL** (e.g. `https://<your-org>.crm6.dynamics.com`) — from maker portal gear → Session details → Instance URL.
2. **Tenant ID**.
3. **Publisher**: display name, unique name, **customization prefix** (e.g. `contoso`), and **option value prefix** (an integer like `12563` → option values start at `125630000`).
4. **Solution unique name**.
5. The **design doc** — parse it for tables, columns (with types), choices (option order matters), relationships (with required level and cascade), form layout, view layout, app sitemap, dashboard.

If building unattended, take the most reasonable reading and state assumptions; only stop for irreversible choices.

## Auth + HTTP skeleton (reuse in every script)

Device-code flow with the first-party client `51f81489-12ee-4a9e-aaae-a2591f45987d`.

```python
import json, sys, time, re, urllib.request, urllib.parse, urllib.error
ORG_URL="https://<your-org>.crm6.dynamics.com"; TENANT_ID="<tenant>"
CLIENT_ID="51f81489-12ee-4a9e-aaae-a2591f45987d"; API=ORG_URL+"/api/data/v9.2"
SOLUTION="YourSolutionUniqueName"

def _post_form(url, form):
    data=urllib.parse.urlencode(form).encode()
    req=urllib.request.Request(url, data=data, method="POST")
    try:
        with urllib.request.urlopen(req) as r: return json.load(r), None
    except urllib.error.HTTPError as e:
        try: return None, json.load(e)
        except Exception: return None, {"error":"http_%s"%e.code}

def get_token():
    dc,err=_post_form("https://login.microsoftonline.com/%s/oauth2/v2.0/devicecode"%TENANT_ID,
        {"client_id":CLIENT_ID,"scope":ORG_URL+"/.default offline_access openid profile"})
    if err: sys.exit(err)
    print("SIGN IN: %s  CODE: %s"%(dc["verification_uri"],dc["user_code"]))
    interval=max(int(dc.get("interval",5)),5); deadline=time.time()+int(dc["expires_in"])
    while time.time()<deadline:
        time.sleep(interval)
        tok,err=_post_form("https://login.microsoftonline.com/%s/oauth2/v2.0/token"%TENANT_ID,
            {"grant_type":"urn:ietf:params:oauth:grant-type:device_code","client_id":CLIENT_ID,"device_code":dc["device_code"]})
        if tok: return tok["access_token"]
        e=(err or {}).get("error")
        if e in ("authorization_pending","slow_down"): continue
        sys.exit(e)
    sys.exit("timed out")

TOKEN=get_token()
BASE={"Authorization":"Bearer "+TOKEN,"OData-MaxVersion":"4.0","OData-Version":"4.0",
      "Accept":"application/json","Content-Type":"application/json; charset=utf-8"}

def _url(p): return API+"/"+p.replace(" ","%20")   # Python 3.14 rejects raw spaces in URLs
def api_get(p):
    with urllib.request.urlopen(urllib.request.Request(_url(p),headers=BASE)) as r: return json.load(r)
def api_send(method, path, body, in_solution=True):
    h=dict(BASE)
    if in_solution: h["MSCRM.SolutionUniqueName"]=SOLUTION   # land component in the solution
    data=json.dumps(body).encode() if body is not None else None
    req=urllib.request.Request(_url(path),data=data,headers=h,method=method)
    with urllib.request.urlopen(req) as r:
        m=re.search(r"\(([0-9a-fA-F-]{36})\)", r.headers.get("OData-EntityId") or "")
        return m.group(1) if m else None   # created record id from the header
```

Every script: call `WhoAmI`, then do its work idempotently (query first, create or patch), then `PublishAllXml` (`POST` `{}`) or `PublishXml` with a `ParameterXml` scoped to what changed.

## Build order (one script per phase)

1. **Publisher + solution** — `POST /publishers` (`uniquename`, `friendlyname`, `customizationprefix`, `customizationoptionvalueprefix`), `POST /solutions` (`publisherid@odata.bind`).
2. **Schema** — tables, columns, choices, relationships (see next section).
3. **Rollup / calculated columns** — rollup columns via `Behavior=Rollup`. Power Fx formula columns **cannot map local choices to numbers**; for choice×choice math use a **plug-in**.
4. **Plug-in** (if needed) — C# `IPlugin`, target **net462**, **strong-named** (`.snk` required — the sandbox rejects unsigned assemblies with `0x8004416c`), sandbox isolation. Register: upload DLL (base64) to `pluginassemblies` (`isolationmode=2`, `sourcetype=0`), create `plugintypes`, then `sdkmessageprocessingsteps` on Create/Update (`stage=40` Pre-Operation, `mode=0` sync). Build on macOS via `Microsoft.NETFramework.ReferenceAssemblies`. Generate the `.snk` in Python if no `sn.exe` (Microsoft PRIVATEKEYBLOB, 596 bytes for 1024-bit RSA).
5. **Model-driven app** — `POST /appmodules` (`name`, `uniquename` with prefix, `webresourceid` default icon). Create a **sitemap** (`sitemaps`, `isappaware=true`, SiteMapXml with `<Area><Group><SubArea Entity="..."/>`). Then `AddAppComponents` the sitemap + entities. **CRITICAL: set `clienttype=4` (Unified Interface)** — an app created via bare `POST` defaults to `2` (legacy web) and triggers the "designed for the legacy web client" warning.
6. **Table icons** — SVG web resources (`webresourceset`, `webresourcetype=11`, `fill="currentColor"`), then set each table's `IconVectorName` (full entity **PUT**, see gotchas).
7. **Views** — `savedqueries` (`querytype=0`, `returnedtypecode`=logical name, `fetchxml`+`layoutxml`). `AddAppComponents` to include in the app.
8. **Forms** — `systemforms` (`type=2`), edit `formxml` (see forms section). Back up first.
9. **Dashboard** — `savedqueryvisualization` charts + a `systemforms` (`type=0`) dashboard, `AddAppComponents`.
10. **Demo data** — records respecting required lookups (see data section).
11. **Choice colors** — `UpdateOptionValue` action per option with a hex `Color`.

## Schema specifics

- **Tables**: `POST /EntityDefinitions` with `@odata.type` `Microsoft.Dynamics.CRM.EntityMetadata`, a primary name `StringAttributeMetadata` (`IsPrimaryName:true`), `OwnershipType:UserOwned`. Add `AutoNumberFormat` for auto-numbered names.
- **Columns**: `POST /EntityDefinitions(LogicalName='x')/Attributes`. Type → `@odata.type`: String, Memo, Integer, Decimal, Money, DateTime (`DateOnly` behavior for dates), Picklist (local option set), Boolean.
- **Choices**: options are created in order; option **value = optionprefix*10000 + index** (prefix `12563` → `125630000`, `125630001`, …). Keep the option order from the design; downstream code maps label→value by index.
- **Relationships**: `POST /RelationshipDefinitions` (`OneToManyRelationshipMetadata`) with a `Lookup` `LookupAttributeMetadata`. Set `RequiredLevel` and `CascadeConfiguration` (`Delete: Cascade` for parental, `RemoveLink` for referential). Denormalize by adding a direct lookup where the design wants one (e.g. a child linked to both its parent and grandparent).
- Idempotency: `table_exists`, `column_exists`, `relationship_exists` (filter `RelationshipDefinitions` by `SchemaName`).

## Views (FetchXML + LayoutXML)

- Resolve display column names to logical names from live metadata (`EntityDefinitions/Attributes` → map `DisplayName.UserLocalizedLabel.Label` → `LogicalName`). This survives rollups whose logical names you don't know — include if present, skip if not.
- LayoutXml grid: `<grid object="{ObjectTypeCode}" jump="{PrimaryNameAttribute}"><row id="{PrimaryIdAttribute}"><cell name="..." width="..."/></row></grid>`.
- FetchXML filters: choice `in`/`not-in`/`eq`/`ne` with option values; **current user** = `<condition attribute="..." operator="eq-userid"/>`; **overdue / before today (dynamic)** = `<condition attribute="contoso_duedate" operator="olderthan-x-days" value="0"/>` (do not hardcode a date — it rots).

## Forms (formxml)

**Always back up `formxml` to a local file before patching.** Parse with `xml.etree.ElementTree`.

- Main forms are `systemforms` `type=2`; pick the default (`isdefault`) first.
- **Header**: a `<header>` element (sibling of `<tabs>`, before it) with cells for `ownerid` and `statuscode`. Remove those from body sections if present.
- **Body**: two 50% `<column>`s, each with labelled `<section>`s of field cells.
- **Field control class IDs** by attribute type: String `{4273EDBD-AC1D-40d3-9FB2-095C621B552D}`, Memo `{E0DECE4B-6FC8-4a8f-A065-082708572369}`, Integer/BigInt `{C6D124CA-7EDA-4a60-AAD6-42056B2C61CF}`, Decimal/Double `{0D2C745A-E5A8-4c8f-BA63-C6D3BB604660}`, Money `{533B9E00-756B-4312-95A0-DC888637AC78}`, Boolean `{B0C6723A-8503-4fd7-BB28-C8A06AC933C2}`, DateTime `{5B773807-9FB2-42db-97C3-7A91EFF8ADFF}`, Picklist/Status `{3EF39988-22BB-4f0b-BBBE-64B5A3748AEE}`, Lookup/Owner/Customer `{270BD3DB-D9AF-4782-9025-509E298DEC0A}`. A cell: `<control id="{logical}" classid="{...}" datafieldname="{logical}" uniqueid="{guid}"/>`.
- **Only place fields with `IsValidForForm=true`** — a metadata attribute that isn't form-valid (e.g. a lookup's `...name` helper) causes a runtime `Could not find a property named '...'` error that blocks the record from opening.
- **Subgrids**: control classid `{E7A81278-8635-4d9e-8D4D-59480B391C5B}`, `indicationOfSubgrid="true"`, params `TargetEntityType`, `ViewId` (a public view of the child), `RelationshipName` (the 1:N schema), `IsUserView=false`.

## Dashboard

- **Charts**: `POST /savedqueryvisualizations` (`name`, `primaryentitytypecode`, `datadescription`, `presentationdescription`). Data description groups by a choice/number and counts the primary id. Presentation is a serialized Microsoft Chart Controls `<Chart>` (Column/Bar/Pie/Line/Funnel).
- **Dashboard form**: `systemforms` `type=0`, `formxml` is a `<form><tabs><tab>` with a `<section columns="111">` of cells using the same `{E7A81278...}` control — `ChartGridMode=Chart` + `VisualizationId` for charts, `ChartGridMode=Grid` + `ViewId` for grids. Omit `objecttypecode` (dashboards are multi-entity).
- `AddAppComponents` the dashboard form and the charts.

## Demo data

- Create parents first, capture ids, then children. Honor **required lookups** (a user-owned project may require a Client account and an owner user — reuse `WhoAmI` UserId).
- Bind lookups with the relationship's `ReferencingEntityNavigationPropertyName` (fetch from `RelationshipDefinitions/...OneToManyRelationshipMetadata`), e.g. `"<nav>@odata.bind": "/contoso_projects(<id>)"`.
- Choice values = option base + index (see schema). Skip auto-numbered primary names.
- For a plug-in-calculated field, leave it unset so the plug-in fills it. Seed a full matrix (e.g. every Probability×Impact combo) to exercise the calc and show variety.

## Gotchas / hard-won lessons

- **Python 3.14 rejects raw spaces in URLs** — always `%20`-encode query strings.
- **`return=representation` isn't reliable on `appmodules`/`sitemaps`** — read the new id from the `OData-EntityId` response header (or re-query), not a read-back.
- **Table & attribute (data-model) metadata cannot be PATCHed** — use **PUT** of the full definition with header `MSCRM.MergeLabels: true` (retrieve, modify, PUT). This is how you set a table's `IconVectorName`.
- **`IsValidForForm` may come back as a plain bool OR a `{"Value":bool}` object** — handle both.
- **App created via API defaults to `clienttype=2` (legacy web)** — set to `4` and republish, else the legacy-client warning appears. Diagnose by listing `appmodules?$select=name,clienttype`.
- **Plug-in assemblies must be strong-named**; environments reject unsigned sandbox assemblies (`0x8004416c`).
- **OptionSet colors in list views** need the **Power Apps grid control → "Enable OptionSet colors = Yes"** per table (a portal setting, not cleanly scriptable). Setting option `Color` alone shows on forms, charts and dashboards only.
- **Publishing**: `PublishAllXml` (`POST` with body `{}`) after metadata/form/view changes; `PublishXml` with `ParameterXml` for a scoped publish.
- Everything should be **idempotent**: query-then-create/patch, safe to re-run; add retry on transient `ResponseEnded` errors for long metadata runs.

## Interaction workflow

1. Read the design doc; confirm the five inputs. Ask only what's genuinely ambiguous.
2. Deliver the **phase 1–2 schema script** first; user runs it, pastes output.
3. Proceed phase by phase (plug-in → app → icons → views → forms → dashboard → data → colors), delivering one script at a time, fixing forward on errors.
4. Keep a task list of phases. Back up forms before touching them. After each phase, tell the user what to verify in the app (open a record, check a view, test the plug-in).
5. When done, offer to record the final state in the design doc.
