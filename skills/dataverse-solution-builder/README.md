# dataverse-solution-builder

Turn a Dataverse / Dynamics 365 solution design document into a working configuration — tables, columns, relationships, plug-ins, a model-driven app, views, forms, a dashboard, and demo data — by generating self-contained Python scripts you run yourself against your environment's Web API.

## What it does

- Reads your design doc and confirms the handful of inputs it needs (org URL, tenant ID, publisher prefix, solution name)
- Hands you one standard-library-only Python script per phase: publisher/solution → schema → rollups/plug-ins → model-driven app → icons → views → forms → dashboard → demo data → choice colors
- Each script authenticates with a device-code sign-in, does its work idempotently (safe to re-run), and prints what it did for you to paste back
- Captures a long list of Dataverse Web API quirks (strong-named plug-in assemblies, `clienttype` defaults, form control class IDs, PUT-vs-PATCH on metadata, and more) so you don't have to rediscover them

## Why this exists

The Power Apps maker portal is painful to drive end-to-end for a multi-table solution — panels revert, command bars don't always render, and there's no good way to replay the same build. Scripting the Web API directly is slower to set up once, but completely repeatable, and it's the only path that lets you keep a custom customization prefix (the Dataverse MCP connector forces the default one).

## How to use it

Install the skill (see the repo's main [README](../../README.md) for installation), then hand your assistant a design document:

> "Here's my solution design doc — build this out in my Dataverse environment."

It will confirm the org URL, tenant ID, publisher details, and solution name, then deliver one script at a time. You run each with `python3 <script>.py`, sign in via the device code it prints, and paste the output back so it can move to the next phase.

## Notes

- You need admin or maker access to the target Dataverse environment, and Python 3 (no extra packages — everything is standard library).
- Scripts are idempotent: safe to re-run if one fails partway through.
- If you need a plug-in, it's generated as strong-named C# (net462) — unsigned assemblies are rejected by the sandbox.
- The customization prefix, solution name, and table/column names in examples throughout the skill are placeholders — swap in your own.
