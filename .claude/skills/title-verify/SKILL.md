---
name: title-verify
description: Build the site and prove what the homepage and doc page head tags actually contain. Use after any change to a page title, meta description, siteConfig.title, or the Layout title prop, and before opening any PR that claims a title changed.
---

# title-verify

Proves, with pasted build output, what `<title>`, `og:title`, `og:site_name`, and the meta
description contain on both the homepage and a doc page. A homepage title change must not move doc
page titles, so both are checked every time.

## Steps

1. Build. Head tags do not exist before a production build.

   ```
   npm run build
   ```

   The build must exit 0. If it does not, stop and report the failure. Do not proceed to grep a
   stale `build/` directory from an earlier run.

2. Grep the homepage. Attributes are allowed in the tag pattern because Docusaurus 3.10 emits
   `<title data-rh="true">`, so a literal `<title>` matches nothing.

   ```
   grep -oE '<title[^>]*>[^<]*</title>|<meta[^>]*(og:title|og:site_name|name="description")[^>]*>' build/index.html
   ```

3. Grep a doc page. `trailingSlash` is false, so the path is `build/docs/get-started.html`, not
   `build/docs/get-started/index.html`.

   ```
   grep -oE '<title[^>]*>[^<]*</title>|<meta[^>]*(og:title|og:site_name|name="description")[^>]*>' build/docs/get-started.html
   ```

4. Paste both raw outputs side by side under headings `HOMEPAGE` and `DOC PAGE`. Paste them, do not
   summarise them.

5. Flag inconsistencies. Check each of these and state the result explicitly:
   - Does `<title>` on the homepage match the string the change intended, character for character?
   - Does `og:title` match `<title>`? It derives from the formatted title, suffix included, so a
     page title that differs from `siteConfig.title` appears as `<page title> | Cadence` in both.
   - Is `og:site_name` still `Cadence Workflow`?
   - Is the doc page title unchanged from before the edit? If a doc page title moved, the change
     touched `siteConfig.title` and is out of scope. Report it as a defect.
   - Is the meta description the intended string?

6. If any check fails, report the mismatch with both the expected and the actual string. Do not
   assert success on any tag you did not paste.
