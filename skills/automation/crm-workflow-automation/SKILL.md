---
name: crm-workflow-automation
description: Inspect and edit CRM workflows with persisted checks.
version: 1.0.0
platforms:
- macos
- linux
- windows
metadata:
  hermes:
    tags:
    - gohighlevel
    - ghl
    - workflows
    - automation
    - mcp
    - forms
    - email
    related_skills:
    - crm-data-integration
author: Minh Duc (minhduc6560-gif), Hermes Agent
license: MIT
---

# GoHighLevel Workflow Automation

Build and modify GoHighLevel workflows through the connected MCP while preserving exact user intent and verifying the persisted result.

## Browser and tool routing

- Prefer the connected API/MCP for workflow operations and verify writes by reading them back.
- If a task genuinely requires the authenticated CRM UI, use the available browser and vault workflow at the user-supplied base URL.
- Do not assume a shared browser profile, fixed Chrome executable, existing cookies, or a fixed debug port.
- Use Preview only for local/public pages that do not require the CRM application login. If Chrome control is unavailable, report the blocker instead of silently falling back to Preview.

## Core workflow

1. Confirm the active location/sub-account before reading or writing assets; never assume a saved token targets the same location as the live connection.
2. Inspect existing workflows to avoid name collisions and accidental duplicates.
3. Run the journey-planning tool when the server requires journey-first planning.
4. List available trigger and action types, then fetch the exact schemas for every type used.
5. Resolve concrete asset IDs for filtered triggers (form, calendar, pipeline, etc.). Never guess an ID.
6. If required information cannot be read, ask only for the missing identifier or let the user explicitly choose a draft/incomplete shell.
7. Prefer a dry run when the tool offers one and the payload is structurally complex.
8. Write fields explicitly: workflow name, status/publish choice, trigger filters, action attributes, sender fields, execution settings, and re-entry/stop behavior.
9. Read back the exact workflow by returned ID. Verify its name, status, triggers, action count, order, and user-visible message content.
10. Validate/audit the persisted workflow before calling it operational.

## Asset inventory reads

1. Read the active location and the requested asset list from the same connection.
2. Report the location name/ID, exact asset names and IDs, statuses when available, and the returned total.
3. Verify the declared total equals the number of enumerated rows; do not silently present a partial page as the full inventory.
4. Treat a connectivity/self-check response as endpoint health only, not as inventory data. If the dedicated list call is denied, report the inventory as unavailable rather than empty.
5. Use a direct API-token fallback only after confirming that token is authorized for the active location and required read scope; a token for another location cannot validate the current inventory.

## Drafts with intentionally missing triggers

A draft without a trigger is intentionally non-operational. State that clearly. If the API writes a partial workflow while reporting a post-write audit failure:

- Capture the returned workflow ID.
- Do **not** retry creation; that can produce duplicates.
- Read back that ID to determine what actually persisted.
- Repair the existing workflow with an update call when needed.
- Report it as a draft/incomplete workflow, not a valid published automation.

## Content preservation

Treat email/SMS copy as exact user-visible content. Builder defaults may decorate or expand it. Always compare persisted `subject`, `html`/`body`, sender fields, and merge fields with the requested copy. If defaults injected unrelated text, update the existing workflow with the exact intended content and read it back again.

## Publishing rules

- Publish only when the workflow has valid triggers, all asset IDs are resolved, and validation passes.
- Keep the workflow in draft when the user wants to select an asset later.
- Never claim a triggerless draft will run.
- For transactional confirmations, choose reply/exit behavior deliberately rather than accepting a nurture-sequence default blindly.

## Verification report

Keep the final report short and include:

- Workflow name and ID
- Draft or published status
- Trigger and filter, or that it is intentionally blank
- Actions created
- Validation state and anything the user must still configure

## Provider-specific notes

See `references/workflow-builder-quirks.md` before using an adapter-specific workflow builder.