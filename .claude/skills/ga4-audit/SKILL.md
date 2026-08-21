---
name: ga4-audit
description: Read-only audit of GA4 and Google Tag Manager wiring, from docusaurus.config.ts through to the tags actually emitted in the production build. Use when asked about analytics, GA4, gtag, GTM, measurement IDs, container IDs, or missing tracking. Makes no edits.
---

# ga4-audit

Read-only. This skill never edits a file. Analytics plugins emit only in production builds, so an
audit against `npm start` is worthless and must not be attempted.

## ID format rules

These are the rules that decide pass or fail. They are not guidelines.

- A Google Tag Manager container ID matches `GTM-XXXXXXX`: the literal prefix `GTM-` followed by 7
  characters from A to Z and 0 to 9.
- A GA4 measurement ID matches `G-XXXXXXXXXX`: the literal prefix `G-` followed by 10 characters
  from A to Z and 0 to 9.
- The two are not interchangeable. A `G-` value in a `containerId` slot is a defect, not a
  stylistic choice. `googletagmanager.com/gtm.js?id=G-...` requests a container that does not
  exist, so the container silently loads nothing.

## Steps

1. Find every analytics wiring point in `docusaurus.config.ts`. Report each with its line number:

   ```
   grep -n 'gtag\|googleTagManager\|containerId\|trackingID\|G-\|GTM-\|analytics' docusaurus.config.ts
   ```

   Record the value of `presets[0][1].googleTagManager.containerId` and
   `presets[0][1].gtag.trackingID` separately. Also check for
   `@docusaurus/plugin-google-gtag` and `@docusaurus/plugin-google-tag-manager` registered in the
   `plugins` array, which would double-load the tags.

2. Build.

   ```
   npm run build
   ```

3. Grep the built output for what actually ships.

   ```
   grep -o -i 'googletagmanager\.com[^"]*\|gtag(\|G-[A-Z0-9]\{10\}\|GTM-[A-Z0-9]\{7\}' build/index.html | sort | uniq -c | sort -rn
   ```

   Repeat on `build/docs/get-started.html` to confirm the tags are site-wide and not homepage only.

4. Apply the ID format rules to every value found, in the config and in the built HTML. For each
   value report: where it was found, the value, which format it matches, which slot it occupies,
   and whether that is a match or a mismatch.

5. Flag every mismatch with pasted evidence. A mismatch claim without the grep line that proves it
   is not a finding. State the consequence in one sentence: which tag fails to load, and therefore
   which data is missing.

6. Report whether gtag and GTM are both active. Running both can double-count pageviews if the GTM
   container also fires a GA4 tag with the same measurement ID. State whether the container
   contents can be confirmed from this repo. They cannot, so say so rather than guessing.
