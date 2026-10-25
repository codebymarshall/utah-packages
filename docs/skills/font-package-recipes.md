---
name: font-package-recipes
description: >-
  Conventions and naming subtleties for font package recipes, including
  dist-git source names versus subpackage provides, spec structure under
  fonts-rpm-macros, and source verification. Load when importing or updating
  font recipes.
metadata:
  type: procedure
---

# Font Package Recipes and Dist-Git Naming

Font packages in Fedora have specific packaging conventions driven by
`fonts-rpm-macros`. This guide captures subtleties encountered when resolving
dependencies and importing font recipes into `utah-packages`.

## Reconcile consumer decisions before importing

[Utah's font decision](https://github.com/projectbluefin/utah/issues/393#issuecomment-5920428811)
rejects bulk font parity with Bluefin classic. A font's presence in the
original audit is not approval to import it: record the rejected parity
request in `[wontfix]` with that decision link, preserving the audit name,
and regenerate `reports/factory-build-backlog.json`. A deliberate individual
font addition needs its own proposal. This does not remove existing recipes
or claim their binaries are published.

Do not extend that font decision to character mappings or input-method data.
In particular, `adobe-mappings-*`, `anthy-unicode`, and
`cldr-emoji-annotation*` remain separate functionality work; the maintainer
explicitly distinguishes input methods from fonts. Likewise, an existing
variable-font recipe is not proof that an older, differently named binary
is provided. Check its actual provides and published metadata rather than
closing a name based on a similar recipe directory.

## Source package names vs. provided binary subpackages

When satisfying a runtime or metapackage font requirement (e.g., from
`default-fonts-*` in `langpacks`), the required package name is frequently a
subpackage or virtual provide rather than the dist-git repository name.

- **Example**: `default-fonts-core-emoji` requires `google-noto-emoji-color-fonts`.
  Querying Fedora dist-git for `rpms/google-noto-emoji-color-fonts` returns 404
  because the upstream source package is named `google-noto-color-emoji-fonts`.
  Its spec explicitly declares:
  ```spec
  %global fontpkgheader0 %{expand:
  Provides: google-noto-emoji-color-fonts = %{epoch}:%{version}-%{release}
  }
  ```
- **Rule**: Always verify which source package provides a given font capability
  in Fedora before importing via `tools/import_rawhide.py`. Import the source
  RPM name, never the subpackage name.

## Specs driven by `fonts-rpm-macros` often omit `Name:`

Specs using `fonts-rpm-macros` generate binary package definitions using
`%fontpkg -a` or `%fontpkg -z ...` and derive the package name from
`%global fontname` or `%global fontfamily` (for example, `stix-fonts.spec` and
`google-noto-sans-cjk-vf-fonts.spec`).

- Tools parsing specs must not rely solely on top-level `Name:` fields.
- `tools/package_inventory.py` relies on the recipe directory name under
  `packages/<name>/` and matches it against `config/upstream-sources.json` and
  `.packit.yaml`.

## Source verification and forge macros

- Font upstreams commonly publish release tarballs on GitHub (e.g., `stix-fonts`
  via `stipub/stixfonts` with `%forgemeta`).
- Direct source URLs can be locked in `config/upstream-sources.json` against the
  canonical release tag archives and matched against the dist-git `sources`
  SHA-512 manifest.
- Always include Fedora lookaside URLs in `fallback_urls` for mirror redundancy.

## Updating package count test assertions

Importing new recipes expands the monorepo package inventory. Whenever adding or
removing recipes, update the inventory count assertions across the test suite:
- `tests/test_package_inventory.py`
- `tests/test_packit_srpm.py`
- `tests/test_render_packit_config.py`
- `tests/test_source_inventory.py`
