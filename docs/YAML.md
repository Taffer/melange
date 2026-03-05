# Melange YAML

Comprehensive documentation for `melange`'s YAML file format.

See also:

- [YAML 1.2.2 specification](https://yaml.org/spec/1.2.2/)
- <BUILD-CACHE.md>
- <BUILD-FILE.md>
- <BUILD-PROCESS.md>
- <DEVENV.md>
- <LINTER.md>
- <PIPELINES-CARGO.md>
- <PIPELINES-CUSTOM.md>
- <PIPELINES-GIT.md>
- <PIPELINES-GO.md>
- <PIPELINES-R.md>
- <PIPELINES.md>
- <TESTING.md>
- <UPDATE.md>
- <VAR-TRANSFORMS.md>

**TODO:** What about a [schema](https://json-schema-everywhere.github.io/yaml)
so we can validate it before we try to `git commit`?
[Schema linter](https://www.json-schema-linter.com) and `pajv` can be used to
validate a `.yaml` against a schema.

**The schema is here:** <https://raw.githubusercontent.com/chainguard-dev/melange/main/pkg/config/schema.json>

## Top-level Sections

Preferred order for top-level sections:

- `package` (required)
- `data`
- `vars`
- `var-transforms`
- `environment` (required)
- `pipeline` (required)
- `subpackages`
- `test`
- `update`

### `data` Section

Constant data available to the build environment. A list of:

- `name`, `items`

### `environment` Section

**Required** Package build environment:

- `contents` - Build environment contents
- `environment` - Environment variables to set for the build

### `package` Section

**Required** Package metadata:

- `name` - Package name
- `version` - Full version
- `epoch` - Chainguard release number
- `description` - Package description
- `copyright` - One or more `license`s that apply to this package
- `dependencies` - List of runtime dependencies

### `pipeline` Section

**Required** Build pipeline steps. A list of:

- `uses`
- `runs`

### `subpackages` Section

Subpackages built from this package. Subpackages:

- Reuse the package `data`, `environment`, `vars`, and `var-transforms`.
- Inherit the filesystem state at the end of the package `pipeline`.
- Are built *in order* after the package `pipeline` runs.
- Run their `test` blocks after the package `test` block (during `make test/…`
  for example).

A list of:

- `name` (required), `dependencies`, `pipeline` (required), `test`

### `test` Section

Tests for the package:

- `environment`
- `pipeline` (required)

### `update` Section

Update checking information. Bots use this to check for package updates using
different services.

- `enabled` (required)
- one of `git`, `github`, `release-monitor`
- `ignore-regex-patterns`

### `vars` Section

Variables available to the rest of the package YAML. One or more "name: value"
pairs.

For example:

```yaml
vars:
  llvm-vers: 20
```

defines `vars.llm-vers` to mean 20.

### `var-transforms` Section

A list of transformations to apply to variables. One or more entries with all
four of:

- `from`
- `match`
- `replace`
- `to`

For example:

```yaml
var-transforms:
  - from: ${{package.version}}
    match: ^(\d+).*
    replace: $1
    to: major-version
  - from: ${{package.version}}
    match: ^(\d+\.\d+).*
    replace: $1
    to: major-minor-version
```

takes the contents of `package.version` and creates new `major-version` and
`major-minor-version` variables based on the first number and first two numbers
of the full version. If `package.version` is 1.2.3, `major-version` will be 1,
and `major-minor-version` will be 1.2.

## A Full Example

An example `melange` YAML file for GNU `hello`:

```yaml
package:
  name: hello
  version: 2.12
  epoch: 0
  description: "the GNU hello world program"
  copyright:
    - attestation: |
        Copyright 1992, 1995, 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2005,
        2006, 2007, 2008, 2010, 2011, 2013, 2014, 2022 Free Software Foundation,
        Inc.
      license: GPL-3.0-or-later
  dependencies:
    runtime:

environment:
  contents:
    repositories:
      - https://dl-cdn.alpinelinux.org/alpine/edge/main
    packages:
      - alpine-baselayout-data
      - busybox
      - build-base
      - scanelf
      - ssl_client
      - ca-certificates-bundle

pipeline:
  - uses: fetch
    with:
      uri: https://ftp.gnu.org/gnu/hello/hello-${{package.version}}.tar.gz
      expected-sha256: cf04af86dc085268c5f4470fbae49b18afbc221b78096aab842d934a76bad0ab
  - uses: autoconf/configure
  - uses: autoconf/make
  - uses: autoconf/make-install
  - uses: strip

subpackages:
  - name: "hello-doc"
    description: "Documentation for hello"
    dependencies:
      runtime:
        - foo
    pipeline:
      - uses: split/manpages
    test:
      pipeline:
        - uses: test/docs

test:
  environment:
    contents:
      packages:
        - bar
  pipeline:
    - runs: |
        hello
        hello --version
```

<!-- markdownlint-disable MD013 -->
| **Substitution**            | **Description**                                                          |
|-----------------------------|--------------------------------------------------------------------------|
| `${{package.name}}`         | Package name                                                             |
| `${{package.version}}`      | Package version                                                          |
| `${{package.epoch}}`        | Package epoch                                                            |
| `${{package.full-version}}` | `${{package.version}}-r${{package.epoch}}`                               |
| `${{package.description}}`  | Package description                                                      |
| `${{package.srcdir}}`       | Package source directory (`--source-dir`)                                |
| `${{subpkg.name}}`          | Subpackage name                                                          |
| `${{context.name}}`         | main package or subpackage name                                          |
| `${{targets.outdir}}`       | Directory where targets will be stored                                   |
| `${{targets.contextdir}}`   | Directory where targets will be stored for main packages and subpackages |
| `${{targets.destdir}}`      | Directory where targets will be stored for main                          |
| `${{targets.subpkgdir}}`    | Directory where targets will be stored for subpackages                   |
| `${{build.arch}}`           | Architecture of current build (e.g. x86_64, aarch64)                     |
| `${{build.goarch}}`         | GOARCH of current build (e.g. amd64, arm64)                              |
<!-- markdownlint-enable MD013 -->
