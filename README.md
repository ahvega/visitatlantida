<div align="center">

# visitatlantida.com: PHP 8 compatibility patches

**The subset of theme and plugin files patched so [visitatlantida.com](https://visitatlantida.com) runs on PHP 8.x, plus a fix for the WPML language switcher.**

[![PHP](https://img.shields.io/badge/PHP-8.6%20compatible-777BB4?logo=php&logoColor=white)](#php-8-fixes)
[![PHP 7.4](https://img.shields.io/badge/PHP%207.4-still%20compatible-8892BF?logo=php&logoColor=white)](#verification)
[![WordPress](https://img.shields.io/badge/WordPress-7.1.5-21759B?logo=wordpress&logoColor=white)](#environment)
[![WPML](https://img.shields.io/badge/WPML-language%20switcher%20fixed-2E8B57)](#wpml-language-switcher-fix)
[![Patched files](https://img.shields.io/badge/patched%20files-8-success)](#modified-files)
[![Status](https://img.shields.io/badge/status-deployed-brightgreen)](#verification)
[![License](https://img.shields.io/badge/license-upstream%20(GPL%20for%20PHP)-blue)](#licensing)

</div>

---

## Contents

- [What this repository is](#what-this-repository-is)
- [Environment](#environment)
- [Modified files](#modified-files)
- [PHP 8 fixes](#php-8-fixes)
- [WPML language switcher fix](#wpml-language-switcher-fix)
- [Verification](#verification)
- [Applying and maintaining the patches](#applying-and-maintaining-the-patches)
- [Known open issues](#known-open-issues)
- [Licensing](#licensing)

## What this repository is

The site runs a commercial theme (**Go Explore!** by Parallelus, built on the Runway framework) and several plugins that are no longer maintained. Moving it to a host running **PHP 8** turned several of them into fatal errors (HTTP 500), because PHP 8 removed functions and syntax they still used.

This repository holds **only the files that were changed**, at their real paths inside the WordPress install, so each patch can be reviewed, reapplied after an update, or reverted. It is **not** a copy of the site:

- `.gitignore` is an allowlist. Every file is ignored except the ones listed in it.
- `wp-config.php`, editor/FTP settings, logs, database dumps and uploads are explicitly excluded, even if the allowlist is widened later.

## Environment

| Component | Version |
|---|---|
| WordPress | 7.1.5 |
| PHP | 8.6.0 (patches also remain valid on 7.4) |
| Theme | Go Explore! 1.3.12 (Parallelus, Runway framework) |
| Multilingual | WPML (Spanish default, English at `/en/`) |

## Modified files

Line numbers refer to the **patched** file, for the plugin or theme version listed. A different version will have different line numbers, so check the version before reapplying.

| Component | Version | File | Line(s) | Issue fixed |
|---|---|---|---|---|
| Ajax Search Pro | 4.0 | [`plugins/ajax-search-pro/includes/widgets.php`](wp-content/plugins/ajax-search-pro/includes/widgets.php) | 7, 10, 66, 69, 210, 213 | PHP 4-style constructors never run on PHP 8; `$this->WP_Widget()` removed |
| | | | 354–356 | `create_function()` removed in PHP 8 (**fatal**) |
| Code Snippets Extended | 1.4.7 | [`plugins/code-snippets-extended/proc/add_snippet.proc.php`](wp-content/plugins/code-snippets-extended/proc/add_snippet.proc.php) | 129–130 | `get_magic_quotes_gpc()` removed in PHP 8 (**fatal**) |
| Elfsight InstaLink | 2.1.10 | [`plugins/elfsight-instalink/api/index.php`](wp-content/plugins/elfsight-instalink/api/index.php) | 944 | Unparenthesized nested ternary (**fatal** since PHP 8) |
| GTmetrix for WordPress | 0.4.8 | [`plugins/gtmetrix-for-wordpress/gtmetrix-for-wordpress.php`](wp-content/plugins/gtmetrix-for-wordpress/gtmetrix-for-wordpress.php) | 279 | `create_function()` removed in PHP 8 (**fatal**) |
| Travel Destinations | 1.1.22 | [`plugins/destinations/destinations.php`](wp-content/plugins/destinations/destinations.php) | 772–774 | Language switcher sent destination pages to 404s ([details](#wpml-language-switcher-fix)) |
| | | `plugins/destinations/assets/js/destinations.min.js` and `destinations.js` *(not published, see [Licensing](#licensing))* | `destinations.js` 24–38 | Language switcher matched menu items to URLs by position |
| Go Explore! (theme) | 1.3.12 | [`themes/parallelus-go-explore/framework/includes/load-functions.php`](wp-content/themes/parallelus-go-explore/framework/includes/load-functions.php) | 1114–1117, 1155–1157, 1166 | Curly-brace string offsets `$s{0}` removed in PHP 8 (**fatal**, breaks every page) |
| | | [`themes/parallelus-go-explore/single-destination.php`](wp-content/themes/parallelus-go-explore/single-destination.php) | 205 | `count()` on a `WP_Query` object throws `TypeError` in PHP 8 (**fatal**) |
| | | [`themes/parallelus-go-explore/taxonomy-destinations-articles.php`](wp-content/themes/parallelus-go-explore/taxonomy-destinations-articles.php) | 30 | `->have_posts()` called on an array when a destination has no articles (**fatal**; also broken on PHP 7.4) |

## PHP 8 fixes

<details>
<summary><b>Ajax Search Pro 4.0</b>: <code>includes/widgets.php</code></summary>

Lines 7, 66, 210: PHP 4-style constructors (a method named like the class) no longer run on PHP 8, so the widgets never initialised.

```diff
-    function AjaxSearchProWidget() {
+    function __construct() {
         ...
-        $this->WP_Widget('AjaxSearchProWidget', 'Ajax Search Pro', $widget_ops);
+        parent::__construct('AjaxSearchProWidget', 'Ajax Search Pro', $widget_ops);
```

The same change applies to `AjaxSearchProLastSearchesWidget` (lines 66, 69) and `AjaxSearchProTopSearchesWidget` (lines 210, 213).

Lines 354–356: `create_function()` was removed in PHP 8.

```diff
-add_action( 'widgets_init', create_function('', 'return register_widget("AjaxSearchProWidget");') );
+add_action( 'widgets_init', function () { register_widget( 'AjaxSearchProWidget' ); } );
```

(Same for the Last Searches and Top Searches widgets.)

Known upstream bug left as is: the Top Searches constructor (~line 214) increments `LastSearchesWidget::$instancenum`.
</details>

<details>
<summary><b>Code Snippets Extended 1.4.7</b>: <code>proc/add_snippet.proc.php</code></summary>

`get_magic_quotes_gpc()` always returned `false` from PHP 5.4 and was removed in PHP 8. The original branch therefore always ran, so the fix keeps its body unconditionally:

```diff
-if (get_magic_quotes_gpc() == FALSE){
-    $this->code = stripslashes($this->code);
-}
+// get_magic_quotes_gpc() always returned false since PHP 5.4 and was removed in PHP 8.
+$this->code = stripslashes($this->code);
```
</details>

<details>
<summary><b>Elfsight InstaLink 2.1.10</b>: <code>api/index.php</code></summary>

PHP 8 rejects nested ternaries without parentheses. The parentheses follow the evident intent of the sort callback: newest first, `0` for equal timestamps.

```diff
-return $item1['created_time'] == $item2['created_time'] ? 0 : $item2['created_time'] < $item1['created_time'] ? -1 : 1;
+return $item1['created_time'] == $item2['created_time'] ? 0 : ( $item2['created_time'] < $item1['created_time'] ? -1 : 1 );
```

PHP 7 grouped the original *left*-associatively, as `(a ? 0 : b) ? -1 : 1`, so it returned `1` instead of `0` for equal timestamps. A side-by-side run (original on PHP 7.4, patched on 8.6) gives the same newest-first order. The only difference is the relative order of posts with identical timestamps.
</details>

<details>
<summary><b>GTmetrix for WordPress 0.4.8</b>: <code>gtmetrix-for-wordpress.php</code></summary>

```diff
-add_filter( 'wp_mail_content_type', create_function( '', 'return "text/html";' ) );
+add_filter( 'wp_mail_content_type', function () { return 'text/html'; } );
```
</details>

<details>
<summary><b>Go Explore! 1.3.12</b>: <code>framework/includes/load-functions.php</code></summary>

`runway_base_decode()` / `runway_base_encode()` used curly-brace string offsets, a parse error in PHP 8. Because this file loads on every request, it took the whole site down.

```diff
-$h1 = strpos($b64, $data{$i++});
+$h1 = strpos($b64, $data[$i++]);
 ...
-$enc .= $b64{$h1} . $b64{$h2} . $b64{$h3} . $b64{$h4};
+$enc .= $b64[$h1] . $b64[$h2] . $b64[$h3] . $b64[$h4];
```
</details>

<details>
<summary><b>Go Explore! 1.3.12</b>: <code>single-destination.php</code></summary>

`$sub_nav_items['articles']` is a `WP_Query` object. In PHP 8, `count()` on a non-countable value throws `TypeError` instead of returning 1.

```diff
-... need_show_articles() && count($sub_nav_items['articles']) && count($sub_nav_items['articles']->posts) ): ?>
+... need_show_articles() && isset($sub_nav_items['articles']->posts) && count($sub_nav_items['articles']->posts) ): ?>
```
</details>

<details>
<summary><b>Go Explore! 1.3.12</b>: <code>taxonomy-destinations-articles.php</code></summary>

`blog_posts_query()` returns `array()` when there is no destination, and calling `->have_posts()` on an array is fatal. This bug also existed on PHP 7.4.

```diff
-if ( $the_query->have_posts() ) {
+if ( $the_query instanceof WP_Query && $the_query->have_posts() ) {
```
</details>

## WPML language switcher fix

**Symptom:** on destination pages, clicking **English** in the main menu reloaded the Spanish page, so the English site seemed not to exist. The home page worked.

**Cause** (Travel Destinations 1.1.22):

1. `destinations.php` prints one hidden `<input>` per language, **current language first**, with the URL each language should link to.
2. `destinations.min.js` copied those URLs into the menu's language items **by position**. That only works when the menu lists every language including the current one. This menu shows only the *other* language, so the English item received the Spanish URL.
3. On single destinations the plugin's English URL was also wrong (`/en/destino/atlantida/la-ceiba/`, a 404): `wpml_permalink()` only swapped the language prefix. WPML's own link (`/en/location/atlantida-region/la-ceiba-city/`) was already correct.

What each source produced, before the fix:

| Page type | WPML's link | Plugin's value |
|---|---|---|
| Destination (e.g. La Ceiba) | ✅ correct | ❌ 404 |
| Destination archive `/destino/` | ❌ 404 | ✅ `/en/location/` |
| Directory item | ✅ | ✅ |
| Category with no English version | ❌ 404 | ✅ "remove item" |

**Fix:**

- `destinations.php` lines 772–774: no override for single destinations, so WPML's correct link stays.
- JS: each menu item is matched to its hidden input by its `wpml-ls-item-<lang>` class, falling back to the old position only for items without that class. Current-language items are never touched.

```js
// destinations.js (destinations.min.js carries the same logic, minified)
$('li.menu-item-language').each(function(index) {
    var $item = $(this);
    if ($item.hasClass('menu-item-language-current') || $item.hasClass('wpml-ls-current-language')) {
        return;
    }
    var match = this.className.match(/(?:^|\s)wpml-ls-item-([a-z]{2,3}(?:-[a-z0-9]+)?)(?=\s|$)/i);
    var $alt = $('#lang-alt-switcher-' + (match ? match[1] : langs[index]));
    if ($alt.hasClass('lang-alt-switcher-del')) {
        $item.remove();
    } else if ($alt.val()) {
        $item.find('a').attr('href', $alt.val());
    }
});
```

> [!NOTE]
> SiteGround Optimizer serves this script inside a combined JS file. After changing it, purge the SiteGround cache so the combined file is rebuilt.

## Verification

- **Syntax:** all 8 patched PHP files pass `php -l` on both PHP 7.4 and 8.6 (Docker `php:7.4-cli`, `php:8.6-cli`).
- **Differential test (Elfsight sort):** the original callback on PHP 7.4 and the patched one on 8.6 sort items newest first in the same order. The only difference is equal timestamps, described above.
- **Equivalent syntax:** `$s{$i}` and `$s[$i]` are the same string-offset read, and closures replace `create_function()` one for one.
- **Site crawl:** a 137-URL sample from the sitemap, plus every URL that had failed, returned HTTP 200 on PHP 8.6 after deployment, with no new fatal errors in `debug.log`.
- **Language switcher:**
  - A fixture test runs the real `destinations.min.js` against menu markup copied from the live site. The original file fails 4 of 6 cases, reproducing the bug; the patched file passes 6 of 6.
  - On staging, a Playwright test clicked the language item on destination, archive, category, directory and home pages, in both directions (ES→EN and EN→ES). Every click landed on the matching page in the other language.
- **Deployment:** every file was compared with the server copy before overwriting. It was deployed to staging first, then to production, and read back after upload to confirm.

## Applying and maintaining the patches

1. Check the installed version of the plugin or theme against the [table](#modified-files).
2. Copy the file over the same path under `wp-content/`. Keep a backup of the original.
3. Purge the SiteGround cache (dynamic cache and combined assets).

> [!WARNING]
> **Updating a patched plugin or the theme overwrites these changes.** Before updating, check whether the new version already fixes the issue. If it doesn't, reapply the change by its description, not by line number.

## Known open issues

These are documented here and **not** fixed in this repository:

- **Yoast SEO Premium 14.7 next to Yoast SEO 28.9.** WordPress loads Premium first, so its old copy of a shared Yoast class (`SEO_Links`) wins. WPML SEO's one-time upgrade task then fails with `Undefined constant SEO_Links::TYPE_EXTERNAL`. Admins see a "critical error" page in `wp-admin` about once a minute. **Fix:** deactivate Premium (check *SEO → Redirects* first), or update it to a current licensed version.
- **Intermittent HTTP 502.** A sample showed about 5% of uncached PHP requests answered with 502 within about 0.5 s. Static files were unaffected and nothing was logged by PHP. This is under investigation on the server side.
- **Deprecation noise.** The Runway framework and Yoast Premium raise many `E_DEPRECATED` notices on PHP 8.2+ (mostly dynamic properties). They're harmless but make `debug.log` grow quickly.

## Licensing

The files here are modified copies of third-party code and keep their **original upstream licenses**. The PHP is GPL. Commercial items (ThemeForest/CodeCanyon) use a *split license*: PHP is GPL, while JavaScript, CSS and images are not. For that reason the patched **JavaScript files of the Travel Destinations plugin are not published**; the change is documented above instead.
