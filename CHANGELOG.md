# Changelog

All notable changes to this role. Versions follow [Semantic Versioning](https://semver.org/):
raising a `php_min_versions` floor is a minor release.

## [Unreleased]

### Added

- Install PHP 8.4 or 8.5, PHP-FPM and extensions from packages.sury.org on Ubuntu jammy, noble and resolute, using a deb822 source.
- Remove ondrej/php PPA sources and the old one-line `sury-php.list`.
- Upgrade only the installed `php<version>-*` packages, and set `php_upgraded`.
- Fail the play when PHP is older than `php_min_versions`: 8.4.27 and 8.5.12 (GHSA-w8hw-w9fx-j79w, GHSA-6wmf-4rv5-p3rp).
- `php_ini` override file, `php_ini_legacy_files` cleanup, and a templated `www` pool tested with `php-fpm -t` before restarting.
- Composer install with installer signature verification and `php_composer_min_version`.
- `extras/snapshot_php.yml` to compare a server's PHP before and after a change.
