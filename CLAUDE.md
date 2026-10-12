# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`gnnpls/wp-self-test` is a Composer **library** (not a plugin) that other WordPress plugins bundle in their vendor directory. A consumer plugin registers a `manifest.json` of case classes; this package runs those cases on a real site and exposes the same runner on five surfaces: wp-admin dashboard, REST (`mwp-self-test/v1`), WP-CLI (`wp mwp self-test`), the Abilities API (`mwp-self-test/*`), and a pre-push hook (`bin/pre-push`, `bin/run.php`). The README is the consumer-facing contract (manifest schema, `Case_Base` helpers, hook table) and must stay in sync with code changes.

## Commands

There is no test suite, build step or `composer install` needed in this repo itself; the only automated check is lint (mirrors `.gitlab-ci.yml`):

```bash
git ls-files -z -- "*.php" | xargs -0 -n1 php -l >/dev/null
```

Cases are exercised from a consumer plugin, e.g. `php bin/run.php --bootstrap=tests/bootstrap.php --autoload=tests/vendor/autoload.php [--plugin=<slug>] [case,ids]` run from the consumer's root, or `wp mwp self-test run` on a site.

`assets/self-test.css` is compiled from `assets/sass/self-test.scss`; no build tooling is committed, so keep both in sync by hand.

## Architecture

**Multi-copy loading.** Several active plugins may each ship a copy. Composer's `autoload.files` includes `bootstrap.php` from every copy → each calls `Loader::register($version, $dir)` → on `plugins_loaded` (-100) `Loader::boot()` picks the highest `version.php` and installs its own PSR-4 autoloader for `Gnnpls\SelfTest\` from that copy's `src/`. Consequences:
- `src/Loader.php` is declared by whichever plugin loads first, possibly an **older** copy. It must stay backwards compatible, and `bootstrap.php` guards with `method_exists`. Any new `Loader` method that `boot()` calls must actually ship in the tag — 0.4.0 and 0.6.0 both shipped broken because of this (see CHANGELOG 0.4.1 / 0.6.1).
- Composer's `autoload-dev` PSR-4 is for development only; at runtime `Loader::autoload()` does the class loading.
- Every PHP file bails on `!defined('ABSPATH')`; `version.php` returns `'0.0.0'` outside WordPress.

**Gates (two layers).** `Loader::enabled()` (`mwp_self_test_enabled`, default true everywhere) decides whether anything boots. `Config::ui_enabled()` (`mwp_self_test_ui_enabled`, false when `Loader::environment()` is `production`; environment = `WP_ENV` if defined else `wp_get_environment_type()`) gates only the dashboard, its REST routes and the WP_CLI shim. CLI commands and abilities register regardless of environment. Capability: `Config::capability()`.

**Boot flow.** `Self_Test::boot()` loads `Wp_Cli_Shim.php` when needed, fires `mwp_self_test_register` at `init` 0 (consumers call `Registry::register()`), attaches `Surfaces\Cli` and `Surfaces\Abilities` always, and `Surfaces\Rest` + `Admin` (renders `templates/dashboard.php` with `assets/`) only when the UI is enabled.

**Execution.** `Registry` holds `Manifest`s (each validates entries; case ids are unique across plugins) and instantiates case classes. `Runner` drives the four `Case_Base` phases — `preview()`, `run()` → JSON-serialisable context, `validate($context)` → `check()`/`skip()` rows, `cleanup($context)` — and persists the last report plus contexts awaiting cleanup in the `mwp_self_test_state` option so cleanup can happen in a later request. All surfaces are thin wrappers over `Runner`.

**WP_CLI shim.** Cases call CLI handlers in-process. Outside a real `wp` process (dashboard, PHPUnit, `bin/run.php`) `src/Wp_Cli_Shim.php` provides a minimal `WP_CLI` that records output and throws instead of exiting; consumers' `cli_loaders` re-declare command classes that returned early when `WP_CLI` was missing.

## Conventions

- PHP ≥ 7.4, namespace `Gnnpls\SelfTest`, class files named `Snake_Case.php`, WordPress-style docblocks with `@since` on every method and above every `do_action`/`apply_filters`.
- Public names are stable API and were deliberately **not** renamed in 0.3.0: hooks `mwp_self_test_*`, command `wp mwp self-test`, routes `mwp-self-test/v1`, abilities `mwp-self-test/*`, option `mwp_self_test_state`. New hooks go in `HOOKS.md` (full reference) and the README "Filters and actions" table.
- Releasing: bump `version.php`, add a CHANGELOG.md entry (Keep a Changelog, SemVer), and tag `vX.Y.Z` — all three together. The newest-copy-wins loader depends on `version.php` being correct.
- The package intentionally ships no CI template (removed in 0.5.0); the consumer's pre-push hook is the gate.
