# RobotShop Discourse Theme

Custom Discourse theme for the RobotShop Community Forum. Replaces the default header and footer with RobotShop branding, navigation, and integrations.

## URLs

- **Production:** https://community.robotshop.com/forum/
- **Staging:** https://community-stg.robotshop.com/forum/

## Structure

```
robotshop-discourse-theme/
├── about.json              # Theme metadata
└── common/
    ├── common.scss         # Main stylesheet (brand colors, layout, hidden categories)
    ├── embedded.scss       # Embedded/iframe styles
    ├── head_tag.html       # JavaScript (composer validation, Klaviyo, Google Fonts)
    ├── header.html         # Custom header (logo, navigation, search)
    └── footer.html         # Custom footer (newsletter, social links, product categories)
```

## Key Customizations

- RobotShop header with navigation: Dashboard, Forums, Tutorials, Robots, Blog, Leaderboards, Shop, Support
- Footer with Klaviyo newsletter, social media links (9 platforms), product categories, global shop links
- Brand colors: `#b82101` (primary), `#791500` (hover)
- Google Fonts: Roboto
- Composer validation requiring 2+ categories
- Hidden categories for non-staff users (IDs: 2, 4, 8, 9, 10, 12, 13, 14, 36)
- Foundation 6 responsive grid

## External Assets

This theme loads external CSS and JS from the Symfony Community app:

- `//community.robotshop.com/web/build/styles/forum.css`
- `//community.robotshop.com/web/discourse.js`

These files are maintained in the [Community repo](https://github.com/RobotShop/Community) under `public/web/` (and for `forum.css`, the SCSS sources under `assets/nwayo/components/forum/styles/`).

## Version Bumps for External Assets

The two external URLs in [`common/head_tag.html`](common/head_tag.html) carry a `?v=NNNNNN` cache-buster:

```html
<link rel="stylesheet" href="//community.robotshop.com/web/build/styles/forum.css?v=NNNNNN" />
<script src='//community.robotshop.com/web/discourse.js?v=NNNNNN'></script>
```

Cloudflare sits in front of `community.robotshop.com` and caches each URL+query combo. Without a bump, browsers keep requesting the old URL and keep getting the pre-deploy asset from the edge — your Community deploy reaches origin but never reaches users.

### When to bump

| You changed... | Bump? |
|---|---|
| Only files in this repo (`*.scss`, `*.html`) | No bump. Push and `Check for Updates`. |
| `public/web/discourse.js` in Community | Bump the `<script>` `?v=` |
| Any SCSS compiling into `public/web/build/styles/forum.css` in Community | Bump the `<link>` `?v=` |
| Both | Bump both |

### Two starting points

**Starting in the Community repo** (the common case — you went there to edit `discourse.js` or the forum SCSS): the change isn't shipped when the Community pipeline finishes. **Wait** for the pipeline, **then** come back here and bump. Bumping first poisons the new URL: Cloudflare misses, fetches whatever is on origin at that instant (often still the old asset), and caches it under the new URL. Full procedure: [Community deployment.md → Path 4](https://github.com/RobotShop/Community/blob/master/docs/deployment.md#4-shared-asset-change-the-tricky-one).

**Starting in this repo** (theme-only — SCSS or HTML in `common/`): no bump needed, no Community coordination. Procedure: [Community deployment.md → Path 3](https://github.com/RobotShop/Community/blob/master/docs/deployment.md#3-theme-only-change).

### How to bump

1. Push the one-line `?v=` change direct to the branch (`staging` first, then `master`) — **no PR**. A cache-buster bump is overkill for review.
2. `Admin → Customize → Themes → [theme] → Check for Updates → Update` on the matching environment.
3. Hard-refresh (Ctrl+F5) and verify.

### Numbering convention

`YYYYMM` followed by a monotonic single-digit suffix: `202604`, `202605`, ..., `202619`, `2026110` (the suffix exists so multiple bumps in the same month don't collide). Always increment.

## Branches

- `master` -- Production
- `staging` -- Staging

## Deployment

1. Push changes to the appropriate branch
2. In Discourse admin, go to Customize > Themes > Check for Updates
3. The theme pulls from this GitHub repo automatically

## Related Repositories

- [Community](https://github.com/RobotShop/Community) -- Symfony app + Discourse config + central documentation
- [discourse-plugin](https://github.com/RobotShop/discourse-plugin) -- Custom webhook + Zendesk plugin

## Documentation

Full platform documentation is centralized in the [Community repo docs](https://github.com/RobotShop/Community/tree/master/docs).
