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

**Pushing is deploying.** A push to either branch tells the matching forum to pull
it, through [`.github/workflows/deploy-theme.yml`](.github/workflows/deploy-theme.yml).
Nobody has to open Discourse admin and press `Check for Updates` any more.

| Push to | Forum | Theme id |
|---|---|---|
| `staging` | https://community-stg.robotshop.com/forum/ | **11** -- "RobotShop Header Theme Staging" |
| `master` | https://community.robotshop.com/forum/ | **7** -- "RobotShop Layout Theme" |

The theme id belongs to the forum, not to the theme: **the staging forum also has a
theme with id 7**, an old copy tracking `master` that nothing serves. Pointing a job at
the wrong id would report success having updated a theme nobody sees. The pairing lives
in exactly one place -- the `case` statement in the workflow -- and that is the only
place to change it.

The workflow does not trust HTTP 200. Discourse answers 200 for a no-op too, so after
the call the job reads back the commit the theme now sits on and fails unless it is the
commit that was pushed. A green run means the forum is serving that commit.

### When it fails

The run tells you which of these it hit; each has one cause.

| Symptom in the log | What happened | Fix |
|---|---|---|
| `HTTP 403` | Cloudflare blocked the runner. Production sits behind bot protection and rejects a bare request. | `CF_AUTOMATION_HEADER` is missing, wrong, or the WAF skip rule was changed. |
| `HTTP 404` | Discourse rejected the API key -- it hides admin routes from non-admins rather than answering 403. | The key was revoked or never set. Re-issue it (below). |
| `HTTP 400` | That theme id does not exist on that forum. | Someone changed the id mapping, or the theme was deleted. |
| `HTTP 000` | The request never completed -- forum down, or DNS. | Check the forum is up before touching anything here. |
| `is at <sha>, not <sha>` | The call succeeded but the forum did not end up on this commit. | Usually a git error on the Discourse side; check the theme in admin. |

Until it is fixed the forum keeps serving the previous commit -- a failed run changes
nothing, it just does not deploy. The manual path still works as a fallback:
`Admin -> Customize -> Themes -> [theme] -> Check for Updates -> Update`.

### Credentials

Three repository secrets, all required:

| Secret | What it is |
|---|---|
| `DISCOURSE_STAGING_API_KEY` | Discourse admin API key on the staging forum |
| `DISCOURSE_PRODUCTION_API_KEY` | Discourse admin API key on the production forum |
| `CF_AUTOMATION_HEADER` | Value of the `X-RS-Automation` header that skips Cloudflare bot protection on `robotshop.com`. Held in 1Password; shared with the other RobotShop automations. |

Both API keys are **global scope**, bound to the `system` user. That is not a shortcut:
Discourse has no `themes` scope (`ApiKeyScope.scope_mappings` covers topics, posts,
users and so on, but not themes), so an unscoped key is the only kind that can reach
`/admin/themes`. Treat them accordingly -- each one is full admin on its forum.

Discourse revokes API keys unused for 180 days (`revoke_api_keys_unused_days`). A branch
that goes six months without a push will need its key re-issued.

To re-issue one, on the forum host:

```
sudo docker exec -w /var/www/discourse app rails runner \
  'k = ApiKey.new(description: "GitHub Actions - theme auto-deploy (COMM-36)", user_id: -1, created_by_id: -1, scope_mode: "global"); k.save!; puts k.key'
```

The plaintext is printed once and never again -- store it straight into the repository
secret. To revoke, set `revoked_at` on the key with the matching description.

## Related Repositories

- [Community](https://github.com/RobotShop/Community) -- Symfony app + Discourse config + central documentation
- [discourse-plugin](https://github.com/RobotShop/discourse-plugin) -- Custom webhook + Zendesk plugin

## Documentation

Full platform documentation is centralized in the [Community repo docs](https://github.com/RobotShop/Community/tree/master/docs).
