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

### The route it takes, and why it is not the obvious one

**A GitHub runner cannot reach either forum.** Cloudflare answers it
`403 / error code 1006` -- "banned IP" -- from the zone's **IP Access Rules**, which are
evaluated *before* the WAF custom rules. The `X-RS-Automation` header that lets our other
automations through is a WAF skip rule, so it never gets the chance to apply. This is not
a credentials problem: the identical request from the corporate network returns 200.

So the runner does not talk to the forum at all. It sends an `aws ssm send-command` to the
forum's own EC2 instance, and the box drives Discourse **through its own Rails console** --
no HTTP, no Cloudflare, nothing opened to the public internet.

```
push -> GitHub Actions -> aws ssm send-command -> the forum box
                                                  -> docker exec app rails runner
                                                     -> RemoteTheme#update_from_remote
```

| Push to | Instances tagged | Theme id |
|---|---|---|
| `staging` | `Role=community-staging` | **11** |
| `master` | `Role=discourse-production` | **7** |

Going through Rails rather than the admin HTTP API removes the Discourse admin keys from
this repository entirely. There is no key to leak, rotate, or print by accident -- and
none sitting in SSM command history, where anyone with console access could read it back
for thirty days.

`update_from_remote` is called with `raise_on_import_error: true`. Without it Discourse
swallows a failed pull and the theme silently stays where it was.

### The workflow does not trust a zero exit code

Three separate things are checked, because each can be true while the deployment failed:

1. **The SSM invocation reached Success** -- `send-command` returning a command id only
   means the request was accepted, not that anything ran.
2. **The remote exit code is 0** -- read directly rather than inferred from the status, so
   a future change to how SSM reports failure cannot turn a red run green unnoticed.
3. **The theme's commit equals the commit that was pushed** -- Discourse reports success
   for a no-op too. This is the only honest evidence, and a green run means the forum is
   serving that exact commit.

### When it fails

The run tells you which of these it hit; each has one cause.

| Symptom in the log | What happened | Fix |
|---|---|---|
| `No running instance is tagged Role=...` | The command reached nobody. | The forum instance lost its `Role` tag, or is stopped. |
| `N instances are tagged Role=...` | More than one instance matched; the job refuses to guess which forum to update. | Fix the tags. |
| `status TimedOut` / `status Failed` | The remote script did not finish, or exited non-zero. Its output is printed above the error. | Read that output -- usually Docker or the Discourse container. |
| `theme is not git-backed` | That theme id is not a remote theme, so nothing can pull into it. | Someone changed the id mapping, or the theme was replaced. |
| `is at <sha>, not <sha>` | The pull reported success but the forum did not end up on this commit. | A git error on the Discourse side; check the theme in admin. |
| `AccessDenied` on `ssm:SendCommand` | The AWS identity lost its permission, or the instance tag changed. | See **Credentials** below. |

Until it is fixed the forum keeps serving the previous commit -- a failed run changes
nothing, it just does not deploy. The manual path still works as a fallback:
`Admin -> Customize -> Themes -> [theme] -> Check for Updates -> Update`.

### Credentials

Two repository secrets, and nothing else:

| Secret | What it is |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key for the `gha-theme-deploy` IAM user, AWS account `014446837051` |
| `AWS_SECRET_ACCESS_KEY` | Its secret |

That user is restricted to `ssm:SendCommand` on instances tagged
`Role=community-staging` or `Role=discourse-production`, plus reading back the result and
`ec2:DescribeInstances`. It can run a shell command on those two boxes and nothing else.

**Three older secrets are no longer read by anything here** and are left only so that
removing them is a deliberate act rather than a side effect:

| Obsolete secret | Why it is dead |
|---|---|
| `DISCOURSE_STAGING_API_KEY` | The Rails route needs no API key |
| `DISCOURSE_PRODUCTION_API_KEY` | Same |
| `CF_AUTOMATION_HEADER` | Nothing here crosses Cloudflare any more |

Both Discourse keys are **global scope** and bound to the `system` user -- each is full
admin on its forum. Now that nothing uses them, they should be revoked rather than left
lying around. To revoke, set `revoked_at` on the key described
`GitHub Actions - theme auto-deploy (COMM-36)`.

## Related Repositories

- [Community](https://github.com/RobotShop/Community) -- Symfony app + Discourse config + central documentation
- [discourse-plugin](https://github.com/RobotShop/discourse-plugin) -- Custom webhook + Zendesk plugin

## Documentation

Full platform documentation is centralized in the [Community repo docs](https://github.com/RobotShop/Community/tree/master/docs).
