# Qaiyo UTM Tracker for Fluent Forms

> Every Fluent Forms entry shows which ad or campaign brought the lead: the UTM parameters of the landing page and the Google Ads click ID, kept across pages and days, with Google Ads auto-tagged clicks recorded as `google` / `cpc` even when the link carries no UTM at all.

[![WordPress 6.5+](https://img.shields.io/badge/WordPress-6.5%2B-21759b.svg)](https://wordpress.org/)
[![PHP 7.4+](https://img.shields.io/badge/PHP-7.4%2B-777bb4.svg)](https://www.php.net/)
[![Requires Fluent Forms](https://img.shields.io/badge/requires-Fluent%20Forms-1a7efb.svg)](https://wordpress.org/plugins/fluentform/)
[![License: GPL v2+](https://img.shields.io/badge/License-GPLv2%2B-blue.svg)](https://www.gnu.org/licenses/gpl-2.0)
[![Free](https://img.shields.io/badge/free-1.0.0-6c5ce7.svg)](#features)

A free add-on for the free [Fluent Forms](https://wordpress.org/plugins/fluentform/) plugin
(Fluent Forms Pro is not required). There is no Pro edition.

- **Website:** [qaiyo-plugins.com](https://qaiyo-plugins.com)
- **Support:** info@qaiyo-plugins.com

---

## Why this plugin

Paid campaigns bring the visitor to one page; the enquiry arrives through a form on another page,
sometimes days later. By then the campaign parameters are gone from the address bar, and the entry
in Fluent Forms says nothing about where the lead came from. Fluent Forms can fill a hidden field
from the current URL (`{get.utm_source}`), but only on the page the visitor is looking at when they
send the form.

Google Ads makes it worse: with auto-tagging, the ad link carries a click ID (`gclid`) and no UTM
parameters, so most trackers store those leads with an empty source.

This plugin closes both gaps:

| Gap | What the plugin does |
|---|---|
| The visitor browses before sending the form | UTM parameters are kept for the browser session, the click ID in a first-party cookie (90 days by default). |
| Google Ads auto-tagging sends no source | An entry with a click ID and no source is recorded as `google` / `cpc`, on the server, before Fluent Forms stores it. |
| Older entries were stored without a source | The Entries screen shows them as `google` / `cpc` at once; a batched tool writes it into the database for exports and reports. |
| A second ad click in the same session | Last touch wins: a new campaign visit replaces every stored value, so campaigns never mix. |
| Consent requirements | With the WP Consent API present, nothing is stored until marketing consent, and stored values are removed when it is withdrawn. |

---

## Features

### Capture (front end)

- Tracks `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term` and `gclid`.
- UTM parameters in `sessionStorage`; `gclid` in the first-party cookie `qfut_gclid`
  (`SameSite=Lax`, `Secure` on HTTPS, lifetime 0–365 days; 0 = session only).
- Fills the hidden fields on page load, on Fluent Forms' `fluentform_init` event (popups,
  AJAX-loaded forms) and in the capture phase of the `submit` event — before Fluent Forms reads
  the form. No `MutationObserver`, no polling, no jQuery dependency.
- Values are stripped of control characters and capped at 500 characters.
- Works with full-page caching: everything happens in the browser.

### Google Ads detection (server)

- Filter `fluentform/insert_response_data`: when the submission has a `gclid` and an empty
  `utm_source`, the source becomes `google` and an empty medium `cpc`. Only fields the form actually
  has are written; a source set by the ad link is never overwritten.
- The values reach both places Fluent Forms stores data: the entry's JSON and the per-field
  `fluentform_entry_details` table that its reports and filters read.
- Optionally limited to selected forms (Fluent Forms → UTM Tracker).

### Older entries

- **Display:** `fluentform/get_raw_responses` (Entries list) and
  `fluentform/submission_before_parse` (single entry) apply the same rule to the loaded copy only.
  No database write, no DOM manipulation of Fluent Forms' Vue table.
- **Backfill:** a batched tool (200 entries per request, keyset pagination by ID) writes the value
  into the database the way Fluent Forms stores an edited entry: the JSON is re-encoded with
  `JSON_UNESCAPED_UNICODE` exactly as Fluent Forms encodes it (its entry search is a `LIKE` on that
  column, so accented names stay searchable), and the changed fields' `entry_details` rows are
  replaced. Each `UPDATE` matches the JSON it read (compare-and-set): an entry edited in the
  meantime is left alone. Running it again changes nothing.
- **Fluent Forms Pro:** its Advanced Filter supports hidden fields and reads the per-field
  `fluentform_entry_details` rows, so filtering by `utm_source = google` finds new entries at once
  and older ones after the backfill. Pro's partial entries (multi-step forms) pass through the same
  display filter as a collection and are handled too.

### Settings screen

- Copy-ready field names with setup instructions in the Fluent Forms editor's own wording.
- Per-form scope for the Google Ads detection, click-ID lifetime, the backfill tool.
- Qaiyo admin design system; keyboard-operable switches with real labels; status messages in
  `role="status"` live regions; WCAG AA contrast; high-contrast (forced colours) fallback.

### Privacy

- No external requests, no telemetry.
- Registers with the WP Consent API (`wp_consent_api_registered_{plugin}`) and gates storage on the
  `marketing` category (`wp_has_consent()`, `wp_listen_for_consent_change`).
- Adds suggested privacy-policy text to the WordPress Policy Guide, stating the actual cookie
  lifetime.

---

## Installation

### From a ZIP file

1. Install and activate **Fluent Forms** (free).
2. *Plugins → Add New → Upload Plugin*, choose the ZIP, activate.
3. *Fluent Forms → UTM Tracker*: copy the field names.
4. In the Fluent Forms editor add one *Advanced Fields → Hidden Field* per name and type the name
   into its **Name Attribute**; leave the default value empty.

Requirements: WordPress 6.5+ (the plugin declares `Requires Plugins: fluentform`), PHP 7.4+,
Fluent Forms (tested with Fluent Forms 6.2.14 and Fluent Forms Pro 6.2.14).

### From source (developers)

```bash
# Symlink or copy the folder into wp-content/plugins/
ln -s "$PWD/qaiyo-utm-tracker-for-fluent-forms" /path/to/wp-content/plugins/
```

No build step: the plugin ships plain PHP, CSS and JavaScript.

---

## Developer API

### Filters

| Filter | Arguments | Purpose |
|---|---|---|
| `qfut_gclid_attribution` | `array $values` (`utm_source`, `utm_medium`) | Values recorded for a Google Ads visit without a source. Only these two keys are honoured; an empty or non-string value falls back to the default (`google` / `cpc`). |

```php
add_filter( 'qfut_gclid_attribution', function ( $values ) {
	$values['utm_source'] = 'google_ads';
	return $values;
} );
```

### Fluent Forms hooks the plugin uses

| Hook | Type | Used for |
|---|---|---|
| `fluentform/insert_response_data` | filter | Google Ads detection on new submissions |
| `fluentform/get_raw_responses` | filter | Display of older entries in the Entries list |
| `fluentform/submission_before_parse` | filter | Display of an older entry in the single-entry view |
| `fluentform_init` (JS, on `document.body`) | jQuery event | Filling forms rendered after page load |

### Stored data

| Name | Where | Content |
|---|---|---|
| `qfut_enabled_forms` | option | Form IDs the Google Ads detection is limited to (empty = all forms) |
| `qfut_cookie_days` | option | Click-ID cookie lifetime in days (0–365, default 90) |
| `qfut_utm_*` | visitor's `sessionStorage` | UTM parameters of the current campaign visit |
| `qfut_gclid` | visitor's cookie | Google Ads click ID |

Uninstalling removes both options on every site of a network. The values already stored in Fluent
Forms entries are form data and stay.

---

## Translations

English source strings, with bundled translations for Hungarian, German, French, Spanish, Japanese,
Portuguese (Portugal), Italian, Russian, Turkish and Polish (`languages/*.po|.mo`, plus the `.pot`).
Regional variants use the closest bundled language (for example `de_AT` → `de_DE`, `pt_BR` →
`pt_PT`); other locales stay English. Where Fluent Forms has its own translation of an interface
term (for example *Hidden Field*, *Name Attribute*), the instructions use that wording.

---

## Standards & security

- WordPress Coding Standards (WordPress, WordPress-Extra, WordPress-Docs) and PHPCompatibilityWP
  7.4+: zero findings. The escaping, nonce, input-sanitisation and SQL sniffs are also run with
  `--ignore-annotations` — zero findings.
- PHPStan **level 10**, no `ignoreErrors`: every untrusted value (options, decoded JSON, request
  fields, filter results, database rows) passes through a checked narrowing helper (`Qfut_Cast`).
- Both state-changing endpoints are AJAX actions for logged-in users only, each with its own nonce
  and the `manage_options` capability. Request fields are an explicit allowlist (`forms`,
  `cookie_days`, `after`); form IDs are accepted only if the form exists.
- SQL uses `$wpdb->prepare()` with `%i` identifiers and `$wpdb->update()/delete()/insert()`.
- Filter callbacks declare no parameter or return types, so an unexpected value from Fluent Forms or
  another plugin passes through instead of becoming a fatal error.
- Everything printed is escaped where it is printed; templates are resolved inside `templates/`
  only (name allowlist, `realpath` containment, no symlinks).
- Official Plugin Check: no errors, no warnings.

---

## Development

### Repository layout

```
qaiyo-utm-tracker-for-fluent-forms.php   Header, constants, class loading, boot
uninstall.php                            Multisite-aware removal of the plugin's options
includes/
  class-qfut-plugin.php                  Composition root: dependency check, module wiring
  class-qfut-attribution.php             The rules: tracked fields, gclid → google / cpc
  class-qfut-settings.php                Options: read, validate, save
  class-qfut-cast.php                    Checked narrowing of untrusted values
  class-qfut-submission.php              Server-side rule on new submissions
  class-qfut-stored-entry.php            Rule applied to a stored entry's JSON (Fluent Forms encoding)
  class-qfut-entry-display.php           In-memory rule for the Entries list and single view
  class-qfut-backfill.php                Batched, compare-and-set write into older entries
  class-qfut-forms.php                   Fluent Forms form list
  class-qfut-admin-page.php              Settings screen controller + view model
  class-qfut-ajax.php                    Save and backfill endpoints
  class-qfut-view.php                    Template renderer (path-traversal safe)
  class-qfut-frontend.php                Capture-script loader
  class-qfut-privacy.php                 Policy text + WP Consent API registration
  class-qfut-i18n.php                    Bundled translations + locale-variant fallback
templates/admin/settings-page.php        The settings screen markup
assets/js/utm-capture.js                 Front-end capture (vanilla JS, deferred)
assets/js/admin-settings.js              Settings screen behaviour
assets/css/admin.css                     Settings screen styles (Qaiyo design system v2)
languages/                               Ten translations + the .pot
readme.txt                               WordPress.org readme
```

Everything in this folder runs on a site and is copied verbatim to the WordPress.org SVN. Tests,
static-analysis configuration, build scripts and the translation generator live in the sibling
`qaiyo-utm-tracker-for-fluent-forms-dev-tools/` folder.

### Quality gates

Run from `../qaiyo-utm-tracker-for-fluent-forms-dev-tools/`:

```bash
composer test          # PHPUnit 9 + Brain Monkey, SQL against in-memory SQLite
composer test-js       # capture script, Node's built-in test runner
composer analyse       # PHPStan level 10
composer coverage      # line + branch coverage (Xdebug)
composer phpcs         # WPCS + PHPCompatibilityWP via phpcs.xml.dist
composer i18n          # .pot from the source, .po/.mo for ten locales, msgfmt --check-format
composer build         # both ZIPs (see below)
composer plugin-check  # official Plugin Check on the WordPress.org package
```

### Building a release

```bash
cd ../qaiyo-utm-tracker-for-fluent-forms-dev-tools
composer build   # → ../qaiyo-utm-tracker-for-fluent-forms.zip       (WordPress.org, no .po/.mo)
                 # → ../qaiyo-utm-tracker-for-fluent-forms-full.zip  (self-hosted, with translations)
```

The build checks that the version agrees in the header, the constant and the readme, and refuses to
package development files.

---

## Contributing

Bug reports and suggestions are welcome at info@qaiyo-plugins.com.

Please follow the existing style (tabs, WPCS, PHPDoc on every method), keep new logic in a pure
service class with tests in the dev-tools folder, and add a `readme.txt` changelog entry under the
next version. Filter callbacks must not declare types (see *Standards & security*).

---

## License

GPL-2.0-or-later. See <https://www.gnu.org/licenses/gpl-2.0.html>.

---

## Credits

Made by **[Qaiyo](https://qaiyo-plugins.com)** — part of the Qaiyo plugin family.

Fluent Forms is a trademark of its respective owner; this plugin is not affiliated with or endorsed
by it.

Contact: info@qaiyo-plugins.com
