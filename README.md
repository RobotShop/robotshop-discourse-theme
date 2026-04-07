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

These files are maintained in the [Community repo](https://github.com/RobotShop/Community) under `public/web/`.

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
