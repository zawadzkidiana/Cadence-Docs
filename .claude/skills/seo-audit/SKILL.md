---
name: seo-audit
description: Read-only audit of head and meta tags across docusaurus.config.ts and the built HTML. Use when asked to review SEO, head tags, meta tags, social cards, or canonical URLs. Makes no edits.
---

# seo-audit

Read-only. This skill never edits a file. It reports what is there and what is wrong with it.

## Steps

1. Read the declared configuration in `docusaurus.config.ts`. Record, with line numbers:
   `title`, `tagline`, `url`, `baseUrl`, `trailingSlash`, `titleDelimiter` if set,
   `themeConfig.metadata`, `themeConfig.image`, and every entry in the top-level `headTags` and
   `themeConfig.headTags`.

2. Build, so the audit covers what ships rather than what is declared.

   ```
   npm run build
   ```

3. Extract the rendered head from at least three pages: `build/index.html`,
   `build/docs/get-started.html`, and one blog page under `build/blog/`.

   ```
   grep -oE '<title[^>]*>[^<]*</title>|<link[^>]*rel="canonical"[^>]*>|<meta[^>]*(name|property)="[^"]*"[^>]*>' <file>
   ```

4. Output one table with the columns `tag`, `value`, `issue`. One row per tag per page audited.
   Leave `issue` empty when the tag is correct. Do not omit correct rows, the absence of an issue
   is itself a finding.

5. Check at minimum:
   - `<title>` length. Flag above 60 characters, since it truncates in results.
   - Meta description length. Flag above 160 characters and flag any page missing one.
   - `og:title`, `og:description`, `og:image`, `og:url`, `og:site_name` present and absolute.
   - `twitter:card` present.
   - `canonical` present, absolute, and matching the page URL.
   - Duplicate titles or duplicate descriptions across the pages audited.
   - Any tag declared in config that does not appear in the built HTML, which means it is not
     shipping.

6. Report findings ordered by severity. Do not propose a diff unless asked. This skill audits, it
   does not fix.
