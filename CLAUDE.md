# Cadence-Docs project instructions

Docusaurus 3.10.0 documentation site for cadenceworkflow.io. Package manager is npm with
package-lock.json. Default branch is master. Node 22, pinned by both .nvmrc and .node-version.

## Commands

| Purpose | Command |
| --- | --- |
| Install | `npm ci` |
| Dev server | `npm start` |
| Production build | `npm run build` |
| Serve the built site | `npm run serve` |
| Swizzle a theme component | `npm run swizzle` |
| Typecheck | `npm run typecheck` |
| Clear caches | `npm run clear` |

Do not run `npm run deploy`. Deployment is not a task for a Claude session.

### Head-tag verification pattern

Head tags only exist after a production build, so build first, then grep the built HTML:

```
npm run build
grep -oE '<title[^>]*>[^<]*</title>|<meta[^>]*(og:title|og:site_name|name="description")[^>]*>' build/index.html
```

Two details that break the obvious version of this grep:

1. Docusaurus 3.10 emits `<title data-rh="true">`, not `<title>`. A pattern anchored on the
   literal string `<title>` matches nothing and looks like a missing tag. Always allow
   attributes: `<title[^>]*>`.
2. `trailingSlash` is false in docusaurus.config.ts, so a built doc page is
   `build/docs/get-started.html`, not `build/docs/get-started/index.html`.

Always verify a doc page alongside the homepage, so a homepage change is proven not to have
moved every other page's title:

```
grep -oE '<title[^>]*>[^<]*</title>|<meta[^>]*(og:title|og:site_name)[^>]*>' build/docs/get-started.html
```

## Design principles

Do not overengineer. One way to do a thing. No fallbacks, no backups, no compatibility shims.
Clarity over compatibility. Fail fast: throw errors, do not swallow them. Separation of concerns.
Surgical minimal diffs. Fix root causes, not symptoms.

## Detective methodology

Form a theory, collect evidence, only fix after evidence proves the theory. Show evidence, do not
assert success. Paste real command output rather than describing it. When uncertain about a
Docusaurus, GA4, or GSC API, check the official docs on the web or read the installed source under
node_modules, do not assume.

## Writing style

No em dashes in any authored content. Use commas, colons, or separate sentences. Be precise: exact
paths, exact commands, exact strings, exact numbers. Never invent a number.

## Docusaurus facts for this repo

- `siteConfig.title` is `'Cadence'` and `siteConfig.titleDelimiter` is unset, so the default `|`
  applies.
- A page title falls through to `siteConfig.title` when the page passes no title. It also falls
  through when the page passes a string equal to `siteConfig.title`, because
  `TitleFormatterFnDefault` in
  `node_modules/@docusaurus/theme-common/lib/utils/titleFormatterUtils.js` returns the bare site
  title when `trimmedTitle === siteTitle`. Passing `'Cadence'` and passing nothing produce
  identical HTML.
- Any page title that differs from `siteConfig.title` is suffixed as
  `<page title> | Cadence`. This is why changing `siteConfig.title` is never the way to change one
  page: it rewrites the suffix on every page in the site.
- `og:title` derives from the formatted page title, suffix included, not from the raw string the
  page passed.
- The title formatter is a swizzleable theme component,
  `@theme/ThemeProvider/TitleFormatter`. It is not a `docusaurus.config.ts` option.
- Analytics plugins emit only in production builds. `npm start` will not show gtag or GTM tags, so
  never verify analytics against the dev server.

## Freeze rule

No homepage title or meta change for 28 days after the CDNC-20311 deploy timestamp recorded in
docs/experiments/cdnc-20311.md. The experiment reads its result at that timestamp plus 28 days, and
any title or meta edit inside the window invalidates the read.

## Branching and commits

Branch naming: `cdnc-<number>-<slug>`. Conventional commits referencing the ticket, for example
`feat(homepage): disambiguate title for CDNC-20311`.
