# ansible-role-php

[![CI](https://github.com/dfo-osdt/ansible-role-php/actions/workflows/ci.yml/badge.svg)](https://github.com/dfo-osdt/ansible-role-php/actions/workflows/ci.yml)

Installs PHP and PHP-FPM from [packages.sury.org](https://packages.sury.org/php/) on Ubuntu, and refuses to leave a server on a PHP release with a known security fix missing.

What it does, in order:

1. Checks the host is a supported Ubuntu release and that `php_version` has a minimum version.
2. Removes ondrej/php PPA sources and adds packages.sury.org.
3. Installs `php<version>`, `-fpm`, `-cli`, `-common` and the extensions in `php_extensions`.
4. Writes one ini override file and the `www` pool.
5. Upgrades the installed `php<version>-*` packages (and nothing else).
6. **Fails the play if PHP is older than the minimum patch release** for its minor version.
7. Installs Composer, verifying the installer's signature.

## Requirements

| | |
|---|---|
| Ubuntu | 22.04 jammy, 24.04 noble, 26.04 resolute |
| PHP | 8.4, 8.5 |
| ansible-core | 2.16 or later, on the control node |
| Privileges | Run with `become: true` |

## Installation

Add the role to your `requirements.yml`, pinned to a release tag:

```yaml
roles:
  - name: php
    src: https://github.com/dfo-osdt/ansible-role-php
    version: v1.0.0
```

```sh
ansible-galaxy install -r requirements.yml --force
```

`--force` is needed to pick up a new tag once the role is already installed.

## Example playbooks

Laravel on PostgreSQL, with bigger uploads, opcache tuning and the pool run by the deploy user:

```yaml
- hosts: app
  become: true
  roles:
    - role: php
      tags: [php]
      vars:
        php_version: "8.5"
        php_extensions: [bcmath, curl, gd, intl, mbstring, pgsql, redis, xml, zip]
        php_ini:
          upload_max_filesize: 250M
          post_max_size: 260M
          opcache.enable: true
          opcache.memory_consumption: 192
        php_fpm_pool:
          user: deploy
          group: www-data
```

Laravel on MySQL, with a larger pool:

```yaml
- hosts: app
  become: true
  roles:
    - role: php
      tags: [php]
      vars:
        php_extensions: [bcmath, curl, gd, intl, mbstring, mysql, xml, zip]
        php_fpm_pool:
          pm.max_children: 75
          pm.start_servers: 10
          pm.min_spare_servers: 5
          pm.max_spare_servers: 25
          pm.max_requests: 500
```

In practice, put these values in `group_vars` rather than in the playbook.

### Restarting queue workers after a PHP upgrade

The role sets the fact `php_upgraded` to `true` when it upgraded any PHP package. Laravel queue workers keep running the old PHP until they restart, so restart them after the role:

```yaml
  post_tasks:
    - name: Restart Laravel queue workers
      ansible.builtin.command: php artisan queue:restart
      args:
        chdir: "{{ document_root }}"
      become_user: "{{ create_user }}"
      when: php_upgraded | default(false)
      tags: [php, php_upgrade]
```

`default(false)` matters: the fact is only set when the upgrade tasks run.

## Security: the minimum version gate

`php_min_versions` maps each PHP minor version to the lowest patch release the role accepts:

```yaml
php_min_versions:
  "8.4": "8.4.27"
  "8.5": "8.5.12"
```

On every run, after installing and upgrading, the role runs `php<version> -v` and `php-fpm<version> -v`. If either is older than the floor, **the play fails**. This catches:

- a server where the upgrade silently did nothing (for example, the old PPA was still pinned),
- packages.sury.org not having published the fix yet, and
- a `php-fpm` that is older than the CLI.

If it fails because sury.org has not published the fix yet, wait and re-run with `--tags php_upgrade`.

In check mode on a server where PHP is not installed yet, the check is skipped.

### Raising a floor when PHP ships a security release

The floors are role defaults, so one role release protects every repo that uses it.

1. Find the fixed versions on the [PHP changelog](https://www.php.net/ChangeLog-8.php).
2. Update `php_min_versions` in `defaults/main.yml` and `meta/argument_specs.yml`, with the advisory IDs in the comment.
3. Run `aar-doc . markdown` to update this README, add a `CHANGELOG.md` entry, and release a new **minor** version.
4. Bump the tag in each repo's `requirements.yml` and run its playbook with `--tags php_upgrade`.

To patch ahead of a role release, set `php_min_versions` in the consuming repo. Remove the override once the role catches up.

## Package source

The role uses packages.sury.org only. The ondrej/php Launchpad PPA is being merged into it, and it is the canonical source from Ubuntu 26.04 onwards.

The repository is written in deb822 format, as `/etc/apt/sources.list.d/sury-php.sources`, and pinned to the Sury signing key. The role also removes:

- ondrej/php PPA lines from any `*.list` file under `/etc/apt`,
- ondrej/php `.list` and `.sources` files, and
- the one-line `/etc/apt/sources.list.d/sury-php.list` written by older in-repo roles.

If the PPA maintainer publishes a different URL or keyring, set `php_sury_uri`, `php_sury_keyring_url` and `php_sury_keyring_path`.

## Configuration files

| File | Source |
|---|---|
| `/etc/php/<version>/<sapi>/conf.d/99-ansible.ini` | `php_ini`, for each SAPI in `php_ini_sapis` |
| `/etc/php/<version>/fpm/pool.d/www.conf` | `php_fpm_pool_defaults` merged with `php_fpm_pool` |

The role owns `www.conf` completely. With the defaults it has the same effective settings as Ubuntu's stock file. Any change to either file runs `php-fpm -t` before restarting, so a bad value fails the play without taking PHP-FPM down.

## Tags

The playbook usually tags the whole role, for example `tags: [php]`, so `--tags php` runs everything. These tags select part of it:

| Tag | Runs |
|---|---|
| `php_repo` | Package source setup and PPA cleanup |
| `php_config` | ini file and `www.conf`, then a restart if they changed |
| `php_upgrade` | Upgrade the installed PHP packages, then restart PHP-FPM |
| `php_composer` | Install or update Composer |

The platform check and the minimum version check run with each of these tags, so no PHP run can skip the security gate. Runs for other roles (`--tags nginx`) don't touch PHP.

Patch PHP everywhere after a security release:

```sh
ansible-playbook server-setup.yml --tags php_upgrade
```

<!-- BEGIN_ANSIBLE_DOCS -->
<!-- Generated from meta/argument_specs.yml by aar-doc. Do not edit by hand. -->
## Role variables

| Variable | Description | Type | Default |
|---|---|---|---|
| `php_version` | PHP minor version to install, for example `8.4` or `8.5`. Must have an entry in `php_min_versions`. | str | `8.5` |
| `php_min_versions` | Lowest patch release allowed for each PHP minor version. After installing and upgrading, the role fails if the `php` CLI or `php-fpm` binary is older than this. The defaults are raised in a new role release whenever PHP ships a security fix. Override only to raise a floor ahead of a release. | dict | `{"8.4": "8.4.27", "8.5": "8.5.12"}` |
| `php_extensions` | Extensions installed as `php<version>-<name>`, for example `pgsql`, `mysql` or `redis`. `fpm`, `cli` and `common` are always installed. Setting this replaces the default list, so include the defaults you still want. | list of `str` | `['bcmath', 'curl', 'gd', 'intl', 'mbstring', 'xml', 'zip']` |
| `php_upgrade` | Upgrade the installed `php<version>-*` packages to the latest available release on every run. Other packages are never upgraded. Sets the fact `php_upgraded` to `true` when anything was upgraded, so a playbook can restart queue workers. | bool | `True` |
| `php_sury_uri` | URI of the PHP APT repository. | str | `https://packages.sury.org/php/` |
| `php_sury_keyring_url` | URL of the `.deb` package that installs the repository signing key. | str | `https://packages.sury.org/debsuryorg-archive-keyring.deb` |
| `php_sury_keyring_path` | Path of the keyring installed by `php_sury_keyring_url`. The repository is pinned to this key. | str | `/usr/share/keyrings/debsuryorg-archive-keyring.gpg` |
| `php_ini` | ini settings written to `/etc/php/<version>/<sapi>/conf.d/99-ansible.ini` for each SAPI in `php_ini_sapis`. Booleans are written as `On` or `Off`. Quote sizes, for example `"250M"`. | dict | `{}` |
| `php_ini_sapis` | SAPIs that get the `php_ini` file. Add `cli` to apply the settings to command-line PHP as well. | list of `str` | `['fpm']` |
| `php_ini_legacy_files` | File names in `conf.d` to delete for each SAPI in `php_ini_sapis`, for ini files that `php_ini` replaces. Use it while migrating from another role, then remove it. | list of `str` | `[]` |
| `php_fpm_pool_defaults` | Base settings of the `www` pool, matching Ubuntu's stock `www.conf`. Usually left alone. Set `php_fpm_pool` instead. | dict | `{"user": "www-data", "group": "www-data", "listen": "/run/php/php<version>-fpm.sock", "listen.owner": "www-data", "listen.group": "www-data", "pm": "dynamic", "pm.max_children": 5, "pm.start_servers": 2, "pm.min_spare_servers": 1, "pm.max_spare_servers": 3}` |
| `php_fpm_pool` | Settings merged over `php_fpm_pool_defaults` to build `/etc/php/<version>/fpm/pool.d/www.conf`. The role writes the whole file, so any setting not in the merged dict is PHP's built-in default. | dict | `{}` |
| `php_composer_install` | Install Composer, verifying the installer against its published SHA-384 signature. | bool | `True` |
| `php_composer_path` | Where Composer is installed. | str | `/usr/local/bin/composer` |
| `php_composer_min_version` | If the installed Composer is older than this, the role runs `composer self-update`. | str | `2.9` |
<!-- END_ANSIBLE_DOCS -->

## Migrating from an in-repo `php` role

1. Before changing anything, snapshot each server:

   ```sh
   ansible-playbook -i inventory.yml ~/.ansible/roles/php/extras/snapshot_php.yml -e snapshot=before
   ```

2. Delete `roles/php`, add this role to `requirements.yml`, and move the repo's values to `group_vars`. List the ini files the old role wrote in `php_ini_legacy_files` so they are removed.
3. Run the playbook with `--check --diff --tags php`, review it, then run it for real.
4. Snapshot again with `-e snapshot=after` and compare:

   ```sh
   diff -ru snapshots/<host>/before snapshots/<host>/after
   ```

   The expected differences are the APT sources (if the server used the PPA), the ini file names, and comments in `www.conf`. Package versions, ini values, pool settings and the socket owner should be unchanged unless you meant to change them.

5. Once every server is migrated, remove `php_ini_legacy_files`.

## Development

```sh
uv venv && uv pip install -r requirements-dev.txt
source .venv/bin/activate
ansible-galaxy collection install -r molecule/requirements.yml

yamllint . && ansible-lint     # lint
aar-doc . markdown             # regenerate the variables section of this README
molecule test -s coip          # needs Docker; scenarios: default, coip, osp, forge
MOLECULE_DISTRO=ubuntu2204 molecule test
```

The `coip`, `osp` and `forge` scenarios use each consuming repo's real values and check the effective result: installed packages, ini values as PHP-FPM loads them, pool settings, socket owner, Composer and the service. Values that are vaulted in those repos are replaced by stand-ins, noted in each scenario.

Edit variables in `meta/argument_specs.yml` and `defaults/main.yml` together. CI fails if this README is out of date.

## License

MIT
