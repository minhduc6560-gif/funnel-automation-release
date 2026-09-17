# configured workflow adapter workflow-builder quirks

## Expired browser workflow session

Workflow reads may continue through the saved Private Integration Token while workflow writes require a short-lived session minted by a logged-in GoHighLevel browser tab. If a write reports `AUTH_EXPIRED`, have the user open or refresh a GoHighLevel workflow tab and wait a few seconds for the connector's documented session refresh mechanism, then retry the same write once.

## Triggerless creation can persist despite failure

A create call with no trigger may return `POST_WRITE_AUDIT_FAILED` / `WORKFLOW_NO_TRIGGER` while also saying `written: true` and returning `partialWorkflow.workflowId`. This is a persisted draft, not a clean failure.

Recovery:

1. Record `partialWorkflow.workflowId`.
2. Do not call create again.
3. Fetch that workflow by ID.
4. Update that same workflow if actions or settings need correction.
5. Leave it draft until a real trigger is added; then validate and publish.

## Best-practice rendering can alter message copy

The create builder may render an email through an expert template and append generic English prose, even when custom HTML was supplied. Verification must inspect the persisted subject and HTML. If the copy differs from the user's intent, call the update-workflow tool with the exact action attributes, preferably guarded by `expectedVersion`, and fetch it again.

Known-good email action shape:

```json
{
  "type": "email",
  "name": "Gửi email xác nhận",
  "attributes": {
    "subject": "Xác nhận đăng ký",
    "html": "<p>Xin chào {{contact.first_name}},</p><p>...</p>",
    "from_name": "{{location.name}}",
    "from_email": "{{location.email}}",
    "attachments": [],
    "trackingOptions": {
      "hasTrackingLinks": false,
      "hasUtmTracking": false,
      "hasTags": false,
      "sourceId": ""
    },
    "conditions": [],
    "fieldDefaults": {},
    "htmlDefaults": {}
  }
}
```

## Form discovery may be plan-gated

A forms endpoint health probe can succeed while the dedicated form-list or submission tool is denied by MCP entitlement checks. Treat the health probe only as evidence that the upstream endpoint responds; it does not prove that the form inventory was retrieved, and it must never be reported as an empty list.

If form listing is unavailable, do not invent a form ID or use an unfiltered form-submission trigger. Before trying a direct API fallback, confirm the token is authorized for the active location and has the required form-read scope. Otherwise ask for the exact form name/ID, or—with explicit user approval—create a draft shell and clearly mark the trigger as unfinished.

## Unverified social-comment trigger filters

For integration triggers such as Facebook comments, `ghl_get_trigger_schema` may show example conditions even when `ghl_trigger_filter_schema` exposes no writable fields. Treat the writable filter schema and a `ghl_create_workflow` dry run as the gate: example conditions are documentation, not proof that the builder accepts them. If dry-run validation reports `TRIGGER_FILTERS_UNSUPPORTED`, do not remove a user-required keyword filter or create an unfiltered live trigger; ask for explicit approval before creating an incomplete draft shell, and clearly identify the filter that must be finished in CRM application.

## Facebook comment workflows depend on both integration state and trigger-filter support

Facebook-specific actions such as `fb_interactive_messenger` can pass dry-run normalization but fail on the real write with `Required facebook integration is not connected for this location`. Recheck after the page is connected; the same action can then persist successfully. A known persisted interactive Messenger button shape is `{ title, type: "url", url }` inside `attributes.buttons`.

The `facebook_comment_on_post` catalog entry may expose example filters (`fb.pageId`, `fb.postId`, `fb.body`) while `ghl_trigger_filter_schema` still returns no supported filters. In that state, create an unfiltered trigger only while the workflow remains draft, then patch the trigger using the exact numeric Facebook Page ID and the documented filter shapes. `ghl_update_trigger` may return `FIELD_NOT_PERSISTED` even though the Page and phrase conditions actually persisted; immediately verify with both `ghl_list_triggers` and `ghl_get_workflow` before treating the write as failed. Never publish unless readback shows the intended filters exactly.

## Minimal intake workflows can hit the strategy gate

A short transactional form intake (tag, one confirmation email, internal alerts) can be refused by `ghl_create_workflow` as `STRATEGY_SINGLE_TOUCH_INTAKE`, even after `ghl_workflow_plan` was run, because the server classifies it as an underbuilt nurture sequence. Do not add unrequested follow-ups merely to pass the gate. Clone a suitable existing draft, replace all actions with the exact requested sequence, add the concrete trigger, validate, show the publish blast radius, obtain confirmation, and publish.

## Custom-email internal notifications need persistence checks

The canonical exported custom-email notification stores the recipient under `attributes.email.to` with `userType: custom_email`. The builder normalizer may strip that nested `to` while retaining recipient-like keys at the parent attributes level. Always read back the persisted action and audit it; report configuration as persisted, but do not claim email delivery was tested unless an actual controlled submission proves it. Named App notifications use `notification.userType: user` and a single user ID string in `notification.selectedUser`.

For App notifications, also verify that the persisted `attributes.notification` contains both a valid `type` (observed working value: `send_notification`) and a non-empty `title`. The normalizer can strip submitted `type` and `title` while leaving `message`/`body`; runtime then fails with `notificationData.type must be a valid enum value`, `notificationData.type should not be empty`, and `notificationData.title should not be empty`. Validation/audit may still report the workflow as structurally valid, so readback plus a controlled new execution is required. Historical execution logs remain failed and are labeled as an older workflow version after correction.