---
name: Manage sample retention and recovery
description: Set or clear sample expiration (single and bulk), follow org-default retention, and restore samples from Trash within the recovery window.
api: openapi/strand-ai-openapi-original.json
operations: [setSampleExpiration, setSamplesExpirationBulk, restoreSample]
---

# Manage sample retention and recovery

Control how long uploaded slides (samples) are retained, and recover ones that were trashed.

## Auth
`Authorization: Bearer sk-strand-...` (organization-scoped). Caller must be the sample's creator,
an org owner/admin, or a Strand admin.

## Steps

1. **Set one sample's expiration** — `setSampleExpiration` with the sample `id` and an
   `ExpirationUpdate` body. Provide exactly one of: `expiresAt` (ISO date-time), `neverExpire:
   true` (sugar for `expiresAt: null`), or `useOrgDefault: true` (clear custom expiry, follow the
   org policy). Optional `reason` (<=500 chars) is audit-logged.
2. **Bulk set expiration** — `setSamplesExpirationBulk` with `sampleIds[]` (1–500 UUIDs) plus the
   same `ExpirationUpdate` fields. All-or-nothing: if any sample fails the permission gate, the
   whole call is rejected before any writes. Returns `{updated, batchId}`.
3. **Restore from Trash** — `restoreSample` with the sample `id`, available within the 7-day Trash
   window. Brings the sample back to the active list and extends its expiration so it isn't
   immediately re-trashed.

## Rules
- `403` = not allowed for this sample / not in the API key's org; `404` = sample not found in this
  org; `412` = sample is not archived (nothing to restore); `422` = invalid expiration input.
- Expiration is opt-in; expired samples sit in Trash for 7 days, then permanent deletion occurs.
- Retention actions are audit-logged (`retentionChangedAt`/`retentionChangedBy`).
