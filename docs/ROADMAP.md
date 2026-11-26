# ROADMAP: visitatlantida.com
Last consolidated: 2026-11-26 | Sources: work log of the 2026-11-25 PHP 8 migration session, [README.md](../README.md) (patch table), live-site checks of 2026-11-26, `wp-content/debug.log`, [custom.css](../wp-content/themes/parallelus-go-explore/custom.css) (CSS stored in the database)

## HOW TO READ THIS DOCUMENT
Work is grouped into five workstreams (WS1–WS5). Statuses are evidence-based: code is checked in this repo or on the server, and settings stored in the WordPress database (Theme Options, Additional CSS) are checked by measuring the live site.
- **COMPLETED**: deployed and verified.
- **IN PROGRESS**: partly deployed, or deployed but not yet effective.
- **UPCOMING**: identified, not started. Usually waiting on a decision or a WP-admin action.

Priority: none were stated, so the column says *not defined*.

## Context

On 2026-11-25 the site moved to SiteGround (All-in-One WP Migration). The host's PHP 8 turned several outdated plugins and the theme into fatal errors. Production and staging (`dev.`) run **PHP 8.6.0**; SiteGround applies one PHP version to both. The stack is WordPress 7.1.5, the Go Explore! 1.3.12 theme (Parallelus, Runway framework), and WPML (ES default, EN at `/en/`).

---

## WS1: PHP 8 compatibility (theme and plugin code)
Scope: code changes that keep outdated, unmaintained theme and plugin files running on PHP 8.x.

### COMPLETED
| Feature | Description | Evidence |
|---|---|---|
| Theme Runway encode/decode | `$s{0}` string offsets → `$s[0]` (parse error took down every page) | `themes/parallelus-go-explore/framework/includes/load-functions.php` L1114–1117, 1155–1157, 1166 · commit `ce5c5c4` |
| Theme destination page | `count()` on a `WP_Query` object → `count(->posts)` | `single-destination.php` L205 · `ce5c5c4` |
| Theme destination articles | `->have_posts()` on an array (also fatal on 7.4) | `taxonomy-destinations-articles.php` L30 · `ce5c5c4` |
| Ajax Search Pro 4.0 | PHP 4 constructors → `__construct`; `create_function()` → closures | `ajax-search-pro/includes/widgets.php` L7–356 · `ce5c5c4` |
| Code Snippets Extended 1.4.7 | `get_magic_quotes_gpc()` removed | `proc/add_snippet.proc.php` L129–130 · `ce5c5c4` |
| Elfsight InstaLink 2.1.10 | Nested ternary parenthesized (only tie order differs from 7.4) | `api/index.php` L944 · `ce5c5c4` |
| GTmetrix for WordPress 0.4.8 | `create_function()` → closure | `gtmetrix-for-wordpress.php` L279 · `ce5c5c4` |
| Verification | `php -l` 8/8 clean on 7.4 and 8.6; 137-URL crawl → 200; no fatals from these files since deploy | README § Verification; `debug.log` |

### UPCOMING
| Feature | Description | Priority | Depends on |
|---|---|---|---|
| Deprecation cleanup | Runway framework raises `E_DEPRECATED` (dynamic properties, 8.2+); harmless but noisy | not defined | WS5 log decision |
| `ninja-forms-legacy/deprecated` | Old PHP 8-incompatible code; only loads if option `ninja_forms_load_deprecated` is on | not defined | Check that option |

---

## WS2: Multilingual (WPML)
Scope: language switching and per-language behaviour.

### COMPLETED
| Feature | Description | Evidence |
|---|---|---|
| Switcher fix (code) | Travel Destinations 1.1.22: JS matched menu items to URLs by position, so EN got the ES URL; destination override produced 404s | `destinations.php` L772–774 + `destinations(.min).js` · `ce5c5c4` · fixture test 2/6 → 6/6 |
| Switcher on staging | ES↔EN verified by clicking, on destination, archive, category, directory and home pages | Playwright run on `dev.` 2026-11-25 |

### IN PROGRESS
| Feature | Description | Current state | Next step |
|---|---|---|---|
| Switcher on production | Files deployed and read back | 2026-11-26: home serves the fixed JS; `/destino/` and destination pages still serve the old combined JS (EN link points to the ES page) | Purge SiteGround cache (dynamic + combined assets), then re-check |

---

## WS3: Front-end styling (Additional CSS / Theme Options)
Scope: visual fixes stored in the database, not in files. *Additional CSS* and *Theme Options* are both saved separately per language (WPML). All changes are documented in [custom.css](../wp-content/themes/parallelus-go-explore/custom.css), which the theme does not load.

### COMPLETED
| Feature | Description | Evidence |
|---|---|---|
| Hero overlay fills the image | `.hero` height limited to the home hero (`.hero-overlap`); overlay `height:100%`; in Theme Options, ES + EN | Live 2026-11-26: home 990/990, inner 640/640 at 1920×1080 · `custom.css` § 2 |

### IN PROGRESS
| Feature | Description | Current state | Next step |
|---|---|---|---|
| Site fonts | Body/menu **Inter Tight**, headings **Antonio**, ES + EN | Rules live in Theme Options, but its `@import` comes after ~50 theme rules and is ignored. The fonts never download; the page falls back to generic sans (2026-11-26: only the Archivo stylesheet is requested) | Move the `@import` to the top of Additional CSS (ES + EN), replacing the unused Archivo one (`custom.css` § 1) |

### UPCOMING
| Feature | Description | Priority | Depends on |
|---|---|---|---|
| Interior hero at 66% of screen | `.hero:not(.hero-overlap) .container { height: 66vh !important; }`; simulated, not applied (`custom.css` § 3) | not defined | Choice: `66vh` vs `max(66vh, 640px)` (U3) |
| Unused font downloads | Archivo `@import` (rules commented out) and Theme Options fonts (Libre Franklin / Open Sans) still load | not defined | Set Theme Options fonts in both languages |
| EN heading sizes | EN Theme Options set h1–h5 sizes, ES only h3; some headings may differ between languages | not defined | U4 |

---

## WS4: Plugins and SEO hygiene
Scope: plugin conflicts, outdated plugins, update policy.

### UPCOMING
| Feature | Description | Priority | Depends on |
|---|---|---|---|
| Yoast SEO Premium 14.7 conflict | Loads before Yoast 28.9; its old `SEO_Links` class breaks WPML SEO's upgrade task, so admins get a critical error about once a minute (7 more overnight) | not defined | U1, U2 |
| Patched-plugin update policy | Updating a patched plugin or the theme overwrites the patches; check the README table before every update | not defined | none |

---

## WS5: Hosting, operations and tooling
Scope: server, environments, logging, deployment, repository.

### COMPLETED
| Feature | Description | Evidence |
|---|---|---|
| Debug logging | Production `WP_DEBUG` + `WP_DEBUG_LOG` on, display off | `wp-config.php` (not in repo) |
| Staging mirror | `dev.visitatlantida.com`, same code and PHP as production | Used for every deploy since 2026-11-25 |
| Safe deploy procedure | Compare the server file with the known original, deploy to staging, verify, deploy to production, read back | README § Verification |
| Patch repository | Public `ahvega/visitatlantida`; allowlist `.gitignore` (patched files only, secrets excluded) | PR #1 merged `4258903` |

### IN PROGRESS
| Feature | Description | Current state | Next step |
|---|---|---|---|
| Intermittent HTTP 502 | About 5% of uncached PHP requests get a fast 502; static files never; PHP logs nothing | Sample of 2026-11-25 (5/100); cause unknown, likely a PHP 8.6.0 worker crash | Site Tools error log at the sample times → try PHP 8.4 (U5) → SiteGround ticket |

### UPCOMING
| Feature | Description | Priority | Depends on |
|---|---|---|---|
| `debug.log` growth | 52 MB on 2026-11-26 (+30 MB in 10 h), mostly deprecations from Yoast Premium + Runway | not defined | U6; WS4 Yoast fix cuts most of it |

---

## UNRESOLVED ITEMS
1. **U1:** Is there a current Yoast SEO Premium license? (Update Premium, or deactivate it.)
2. **U2:** Does *SEO → Redirects* list any redirects? (They stop working if Premium is deactivated; the server export file is empty.)
3. **U3:** Interior hero: plain `66vh` (shorter than today's 640px on screens under ~970px tall), or `max(66vh, 640px)`?
4. **U4:** Should EN and ES headings have identical sizes?
5. **U5:** Is switching PHP 8.6.0 → 8.4 acceptable for the 502 test? (It changes production and staging together.)
6. **U6:** Keep `WP_DEBUG_LOG` on in production, or turn it off once the Yoast conflict is fixed?
7. **U7:** The last rule in Additional CSS (`.cb_lbl span, a, a:hover, a:active {…}`) styles **every** link that no more specific rule covers (fuchsia, `margin-right: 15px`). Is that intended, or should it be `.cb_lbl span, .cb_lbl a, …`?
