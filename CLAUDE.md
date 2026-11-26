# visitatlantida.com: project instructions

WordPress tourism site (ES default, EN at `/en/` via WPML). Theme **Go Explore! 1.3.12** (Parallelus, Runway framework) with the bundled **Travel Destinations** plugin. Hosted on SiteGround, running **PHP 8.6.0**.

- **Status and open items:** [docs/ROADMAP.md](docs/ROADMAP.md). Read it first.
- **Every patched file**, with plugin/theme version and line numbers: [README.md](README.md).

## What this folder is

- A **partial local mirror** of production `public_html`: the PHP files of the theme and some plugins, plus `wp-config.php`. It is **not** a runnable site. There is no database and no uploads.
- A git repo that **publishes to a PUBLIC GitHub repo** (`ahvega/visitatlantida`). `.gitignore` is an **allowlist**: only the patched files and `README.md` are tracked.
  - To publish a newly patched file, add a `!/path` line to `.gitignore` **and** a row to the README table.
  - Never widen the allowlist with globs.
  - `CLAUDE.md`, `docs/ROADMAP.md`, `docs/roadmap.html` and the theme's `custom.css` are tracked too, so **they are public**. Keep them free of credentials, the FTP host and personal data.
- **Never commit or print:** `wp-config.php`, `.vscode/sftp.json` (FTP password), `debug.log`, DB dumps.
- **Never publish** the plugin/theme **JS/CSS**. ThemeForest/CodeCanyon split license: only PHP is GPL.
- Git:
  - Branch names must match `<tipo>/<id>--<fuente>` (a hook enforces it), e.g. `fix/php8-compat-patches--user`.
  - `core.autocrlf=false`: keep files byte-identical to the server (LF).
  - Commit/push only when asked.

## Environments

| | Production | Staging |
|---|---|---|
| URL | https://visitatlantida.com | https://dev.visitatlantida.com |
| FTP root | `/visitatlantida.com/public_html` | `/dev.visitatlantida.com/public_html` |
| Errors | `wp-content/debug.log` (WP_DEBUG_LOG on, display off) | `php_errorlog` in the site root (WP_DEBUG off) |

- **PHP is coupled.** Changing the PHP version for one site changes it for both, so staging can't test a different PHP version.
- **Credentials** live only in `.vscode/sftp.json` (`remotePath` = production root). Read them programmatically; never echo them.

## Deploying a file (the safe procedure)

1. Download the **current server copy**. Confirm it equals the known original before overwriting. If it differs, stop and ask.
2. Test locally first:
   - `php -l` on PHP 7.4 and 8.6 (Docker `php:7.4-cli`, `php:8.6-cli`).
   - Behaviour changes: a differential or red/green test (the original must fail it).
3. Deploy to **staging**, verify in a real browser, then deploy to **production**. **Read back** each upload and compare hashes.
4. Check `debug.log` for new fatals after the deploy.
5. Ask the user to **purge the SiteGround cache**. SiteGround Optimizer serves CSS/JS as **combined files** (per page group), so a changed JS file is not live until the purge.

## Gotchas

- **Git Bash path mangling:**
  - Remote paths like `/visitatlantida.com/...` get rewritten. Prefix commands with `MSYS_NO_PATHCONV=1`.
  - With that set, use `curl -o NUL`, not `/dev/null`.
- **SiteGround anti-bot.** After roughly 100 rapid requests from this IP, every HTTP response to curl becomes a `202` challenge, on both sites. FTP is unaffected.
  - Don't run probe loops.
  - Playwright (a real browser) passes the challenge. Wait for page content before measuring.
- **Direct `.php` blocked.** Requests to `.php` files inside `wp-content` return `403`, so a bare-PHP probe needs a file in the site root (staging only, delete after).
- **CSS lives in the database, saved separately per language (WPML):**
  - *Customizer → Additional CSS* prints as `#wp-custom-css`, after the theme CSS. Its ES and EN copies differed only in whitespace on 2026-11-26, so they are separate saves.
  - *Theme Options → Custom CSS* is appended to the end of `#theme-style-inline-css`, together with the theme font settings.
  - **Edit both languages.**
  - The changes are documented in `wp-content/themes/parallelus-go-explore/custom.css`. The theme never loads that file; keep it in sync when CSS changes.
- **`@import` only works at the very top of a stylesheet.**
  - In Theme Options it lands after ~50 theme rules and is silently ignored. The fonts then fall back while `getComputedStyle` still reports the declared family.
  - Put `@import` at the top of Additional CSS.
  - To prove a font loads, check the network requests or `document.fonts` loaded faces. `document.fonts.check()` returns `true` when no matching face exists.
- **Hero markup:**
  - The structure is `section.hero` → `.bg-overlay` → `.container`.
  - The container gets an **inline** `height:640px` (option `header-height-default`), so overrides need `!important`.
  - The home page is the only `.hero-overlap` hero, and its body class is `blog`, not `home`.
- **Theme font picker:** its bundled Google Fonts list is from about 2015 (no Inter Tight). It only fetches the live list once a *Google API Key* is saved in the Runway theme settings. Its loader also drops font weights. Load fonts via `@import` in Additional CSS instead.
- **Plugin updates overwrite patches.** Before updating a patched plugin or the theme, check the README table.
- **Yoast SEO Premium 14.7** (installed alongside Yoast 28.9) causes the recurring `SEO_Links::TYPE_EXTERNAL` fatal in wp-admin. It's a known issue (ROADMAP WS4), not something new.

## Useful checks

- **Language switcher:** load an ES destination page. Read `.navbar-nav li.menu-item-language > a` **after JS runs**; the raw HTML differs. It must point to the `/en/location/...` page.
- **Hero:** `section.hero` height must equal `.bg-overlay` height on the home page and on inner pages.
