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

The names above are the ones each **forum** shows, read from `Theme#name` on the two boxes.
Do not expect them to match [`about.json`](about.json), which says `RobotShop Header Theme`
for both: the name in the file is only the initial value, and each forum's copy has been
renamed since. The id is what identifies a theme; the name is a label.

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
3. **The theme's commit is the commit that was pushed, or a descendant of it** --
   Discourse reports success for a no-op too, so the recorded commit is the only honest
   evidence. A descendant is accepted because `update_from_remote` pulls the branch *tip*,
   and the tip can legitimately be ahead of the run: a push made with another
   repository's `GITHUB_TOKEN` does not trigger a workflow, so the cache-buster
   automation can land here without a run of its own.

What a green run proves is precisely this: **`RemoteTheme#local_version` in the forum's
database is that commit.** The workflow never fetches a stylesheet, so it cannot tell you
what a browser receives -- Cloudflare and the `?v=` cache-buster on the
[external assets](#external-assets) both sit between the two. To confirm that, hard-refresh
the forum (Ctrl+F5) and look.

### When it fails

The run names which one it hit. The likely cause is given below, not the only possible one.

| Symptom in the log | What happened | Where to look |
|---|---|---|
| `No forum is mapped to branch ...` | The workflow ran for a branch the `case` statement does not know. | The push trigger and the `case` statement disagree. Fix whichever is wrong. |
| `No running instance is tagged Role=...` | The command reached nobody. | The forum instance lost its `Role` tag, or is stopped. |
| `N instances are tagged Role=...` | More than one instance matched, and **each already ran the command as root**. The job refuses to say which forum it updated. | Fix the tags before re-running. |
| `status TimedOut` / `status Failed` | The remote script did not finish, or exited non-zero. Its stdout **and stderr** are printed above the error -- the remote script reports every refusal on stderr. | That output, usually Docker or the Discourse container. |
| `status InProgress` | The command outlived the poll budget (600 s). It is probably still running and may yet succeed. | Check the command in SSM before assuming failure, then re-run via `workflow_dispatch`. |
| `theme is not git-backed` | That theme id is not a remote theme, so nothing can pull into it. | Someone changed the id mapping, or the theme was replaced. |
| `discourse recorded an import error` | Discourse moved the git pointer but the import was partial -- the sha advanced, the content did not. | `last_error_text` on the theme, in admin. |
| `printed no theme version` | The remote script exited 0 without its markers. | The stdout block above; something changed in the remote script or the container. |
| `is at <sha>, ... neither ... nor a descendant` | The forum is on a commit unrelated to this run. | The theme's remote URL and branch in admin -- it may be tracking something else. |
| `AccessDenied` on `ssm:SendCommand` | The AWS identity lost its permission, or the instance tag changed. | See **Credentials** below. |

A run that ends `::notice:: ... a descendant of this run's <sha>` is **green and correct**:
a newer commit reached the branch mid-flight and the forum serves that instead.

Until it is fixed the forum keeps serving the previous commit -- a failed run changes
nothing, it just does not deploy. Re-run from the Actions tab (`workflow_dispatch`) once the
cause is addressed; an empty commit is not needed. The manual path still works as a
fallback: `Admin -> Customize -> Themes -> [theme] -> Check for Updates -> Update`.

### Credentials

Two repository secrets, and nothing else:

| Secret | What it is |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key for the `gha-theme-deploy` IAM user, AWS account `014446837051` |
| `AWS_SECRET_ACCESS_KEY` | Its secret |

That user is restricted to `ssm:SendCommand` on instances tagged
`Role=community-staging` or `Role=discourse-production`, plus reading back the result and
`ec2:DescribeInstances`. It can run a shell command on those two boxes and nothing else.

An earlier draft of this workflow called the Discourse admin HTTP API and needed three more
secrets. The Rails route needs none of them, so on 11 August 2026 they were removed from
this repository and the two Discourse API keys behind them were **revoked on both forums** --
each was global scope, bound to the `system` user, and therefore full admin on its forum.
Nothing here reads `DISCOURSE_STAGING_API_KEY`, `DISCOURSE_PRODUCTION_API_KEY` or
`CF_AUTOMATION_HEADER`; if you find them again, something has regressed.

If a future change ever does need a Discourse API key, note that **Discourse has no `themes`
scope** -- `ApiKeyScope.scope_mappings` covers topics, posts, users and so on, but not
themes -- so any key that can reach `/admin/themes` is necessarily unscoped. That asymmetry
is the main argument for staying on the Rails route.

## Related Repositories

- [Community](https://github.com/RobotShop/Community) -- Symfony app + Discourse config + central documentation
- [discourse-plugin](https://github.com/RobotShop/discourse-plugin) -- Custom webhook + Zendesk plugin

## Documentation

Full platform documentation is centralized in the [Community repo docs](https://github.com/RobotShop/Community/tree/master/docs).
