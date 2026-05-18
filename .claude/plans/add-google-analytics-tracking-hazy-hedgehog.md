# Add Google Analytics tracking via GTM to Mintlify docs

## Context

The Mintlify docs site at `dev-docs` currently has no analytics wired up. A placeholder `analytics.js` sits at the repo root but is not referenced anywhere and is not a file Mintlify auto-loads.

The user wants traffic on the docs site tracked. They provided:

- `GTM_CONTAINER_ID=GTM-TV46FJWQ`
- `GA4_MEASUREMENT_ID=G-0YKEW43GMY`

Decision (confirmed with user): wire **GTM only** in `docs.json`. The GA4 property is expected to be loaded via a GA4 Configuration tag inside the GTM container. Wiring both natively would double-count pageviews when the container fires its own GA4 tag.

## Change

Edit `/Users/xer/code/dev-docs/docs.json`: add a top-level `integrations` block with the GTM tag id.

Insert after the existing `"feedback": {...}` block (before `"anchors"`), preserving the file's existing key ordering style:

```json
"integrations": {
  "gtm": {
    "tagId": "GTM-TV46FJWQ"
  }
}
```

That is the only edit. No other files change.

## Schema source

Confirmed against the Mintlify integrations reference (`https://www.mintlify.com/docs/integrations/analytics/google-tag-manager.md`):

- Path: `integrations.gtm.tagId`
- Mintlify injects the GTM script on every docs page automatically when this field is set.
- Public IDs only — the tag id is not a secret, so committing it is fine.

## Files

- **Modify:** `docs.json` (add `integrations.gtm.tagId`)
- **Leave alone:** `analytics.js` (unrelated placeholder; not wired up, not loaded by Mintlify). If the user wants it removed, that's a separate cleanup.

## Verification

1. Inside the GTM workspace for `GTM-TV46FJWQ`, confirm a **GA4 Configuration tag** exists with Measurement ID `G-0YKEW43GMY` firing on All Pages. Without this, GA4 will receive nothing.
2. Run the docs locally: `mintlify dev` in `/Users/xer/code/dev-docs`.
3. Load any page and view source — confirm the GTM snippet (`https://www.googletagmanager.com/gtm.js?id=GTM-TV46FJWQ`) is present in `<head>` and the `<noscript>` iframe in `<body>`.
4. Open Chrome DevTools → Network, filter `collect` — confirm a request to `google-analytics.com/g/collect` fires on page load with `tid=G-0YKEW43GMY`.
5. Use the **GTM Preview** mode (Tag Assistant) against the local dev URL to confirm the container loads and the GA4 Configuration tag fires.
6. After deploy, check GA4 → Reports → Realtime to confirm live traffic appears.

## Out of scope

- Removing or repurposing `analytics.js`.
- Configuring custom events, consent mode, or cookie banners.
- Adding any of the other 17 analytics platforms Mintlify supports.
