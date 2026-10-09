---
id: languagesgolang
title: Golang
description: This document describes the packaging guidelines for Golang modules in openRuyi.
slug: /guide/packaging-guidelines/languages/Golang
---

# Golang Packaging Guidelines

This document describes the Golang packaging guidelines for openRuyi. For the relevant build system, please refer to [Golang](/docs/guide/packaging-guidelines/BuildSystems/golang).

## Versioning

Many Go libraries, as well as some Go programs, exist only in version control systems and are never released as formal upstream versions. For versioning rules, please refer to the [Versioning](/docs/guide/packaging-guidelines/Versioning) section.

## Binary-Only Packages

A binary-only package contains a program written in Go, but does not ship source code. For example, `ollama` is written in Go, but it does not provide an API intended for reuse by other Go packages, so it should not ship a source package for others to import.

### Naming

The package name for this kind of software must match the upstream project name, for example, `ollama`. Naming it `go-ollama` is incorrect. Likewise, the resulting binary package name must not include the `go-` prefix.

## Library-Only Packages (or Packages Containing Both Binaries and Libraries)

Within openRuyi, the sole purpose of packaging Go libraries is to build other Go programs in the openRuyi system. We do not intend them for end users in day-to-day development workflows. For normal development use cases, users should use `go get`.

### Naming

Every Go library has its own import path. You should derive package names from that import path by replacing all slashes with hyphens and substituting the hostname with a standardized identifier.

Ideally, you will base the final package name directly on the import path. The table below shows some examples:

| Import Path                 | openRuyi Package Name      |
| --------------------------- | -------------------------- |
| golang.org/x/oauth2         | go-golang-x-oauth2         |
| google.golang.org/protobuf  | go-google-protobuf         |
| github.com/jpillora/backoff | go-github-jpillora-backoff |
| git.sr.ht/\~sbinet/cmpimg   | go-sourcehut-sbinet-cmpimg |

### File Placement

You must install all files under `/usr/share/gocode/src`, which corresponds to `$GOPATH/src`.

For example, for `github.com/boombuler/barcode` (that is, `go-github-boombuler-barcode`), you should install one of its files as: `/usr/share/gocode/src/github.com/boombuler/barcode/code93/encoder.go`

### Handling Multiple Versions

Go libraries frequently introduce incompatible changes across major versions. In most cases, the import path reflects this. If you need to package different versions of the same Go library, you should append the major version suffix to the package name exactly as it appears in the import path.

For example, the import paths for `github.com/cespare/xxhash` differ between the v1.x and v2.x series:

* You may package the former as `go-github-cespare-xxhash` (that is, `go(github.com/cespare/xxhash)`).

* You may package the latter as `go-github-cespare-xxhash-v2` (that is, `go(github.com/cespare/xxhash/v2)`).

There are, however, exceptional cases. Such an exception typically happens when upstream introduces a new incompatible major version without changing the import path.

For example, `github.com/olekukonko/tablewriter` has both a legacy `v0.0.5` version and the latest version, but they share the same import path. In such cases, the latest package should use the unsuffixed import-based name, while the legacy package should carry a major-version suffix.

* You may package the legacy version as `go-github-olekukonko-tablewriter-v0`.

* You may package the latest version as `go-github-olekukonko-tablewriter`.

At the same time, because both packages use the same import path, they must be declared as mutually exclusive in the RPM Spec and must not be installed simultaneously. For example, the latest version should include:

```specfile
Conflicts:      go(github.com/olekukonko/tablewriter) = 0.0.5
```

And the legacy version should include:

```specfile
Conflicts:      go(github.com/olekukonko/tablewriter) >= 1.0.0
```

## Dependencies

Automatic `Provides` generation does not generate `BuildRequires` or `Requires`. Library packages MUST continue to list all required Go library dependencies in both the `BuildRequires` and `Requires` sections of the RPM Spec. These dependencies must be available at build time to run tests and at install time so that other packages can build against them.

When declaring Go library dependencies, you MUST use the required virtual dependency format: `go(<import path>)`.

For each eligible Go library package discovered in the final installed source tree, `%go_generate_provides` creates a `.goipath` marker. It excludes `main` packages, packages without normal Go, Cgo, or assembly source files, packages below `example` or `examples` directories, and packages that `go list` reports as errors. The RPM Go file attribute generator reads the resulting markers and automatically emits one versioned `go(<import path>)` capability for the RPM binary package that owns each marker. Library packages MUST NOT manually declare `Provides` entries for canonical import paths covered by automatic generation.

Handwritten `Provides` entries remain necessary only for intentional capabilities that the generator cannot derive from packaged `.goipath` markers. Examples include a legacy import-path alias preserved only by a compatibility symlink and an intentional virtual module-root capability with no importable package directory. Packagers MUST verify that a handwritten capability is not already generated automatically.

Other library packages can reference the automatically generated capabilities in the same way:

```specfile
BuildRequires:  go(github.com/clipperhouse/stringish)
# ...
Requires:       go(github.com/clipperhouse/stringish)
```

### Required BuildRequires

The following two build dependencies are mandatory:

* `go` — provides the Go compiler and toolchain itself.

* `go-rpm-macros` — provides the RPM macros and Go file attribute generator required for packaging Go libraries. For example, you can reference the path `/usr/share/gocode/src` via `%{go_sys_gopath}`.

## When Upstream Migrates to a New Repository Location

In some cases, an upstream package may move from one hosting provider to another. The package MUST be renamed for the new canonical import path. If existing consumers still require the legacy import path, the `%install` section MUST create a compatibility symlink from the legacy path to the canonical path. The automatic generator emits capabilities for importable Go packages under the canonical path, but it does not infer aliases from compatibility symlinks. The RPM Spec MUST therefore declare a versioned `Provides` entry manually for each legacy import-path alias that existing consumers still require:

```specfile
# Preserve the legacy import path for existing consumers.
Provides:       go(example.org/legacy/project) = %{version}
```

## Automated Bootstrapping

openRuyi currently provides a small utility called `go2spec` for bootstrapping the packaging of Go libraries or programs. openRuyi adapted this tool from the Debian Go Packaging Team’s `dh-make-golang` to better suit openRuyi’s needs. We gratefully acknowledge their work.

For example, to package the Go library `github.com/fsnotify/fsnotify`:

```bash
./go2spec pack github.com/fsnotify/fsnotify
```

`go2spec` will automatically handle the relevant metadata (currently GitHub only) and then generate the corresponding Spec file.

## RPM Macros for Go Modules

This section lists several commonly used RPM macros for Go modules.

### %go_common

This macro performs two tasks:

* It tells Go to ignore the `go.mod` file and fall back to the legacy GOPATH mode.

* It defines the Go workspace (`GOPATH`) so that the source tree currently being built and the default system-wide location of Go dependency packages are both treated as part of `GOPATH`.

### %__go

This macro expands to the name of the `go` executable installed on the system.

### %go_generate_provides

This macro scans one Go import root in the final installed source tree and writes a `.goipath` marker for each eligible Go library package it discovers. The macro does not emit `Provides` lines itself. During RPM dependency generation, the RPM Go file attribute generator converts each packaged marker into a versioned `go(<import path>)` capability.

`%buildsystem_golangmodules_install` invokes this macro automatically after copying the source tree. A spec that uses `%buildsystem_golangmodules_install`, either directly or through the default `golangmodules` install phase, MUST NOT invoke `%go_generate_provides` again for the same import root.

A full custom `%install` that does not invoke `%buildsystem_golangmodules_install` MUST finish copying, relocating, and pruning the Go source tree before invoking the generator:

```specfile
%go_generate_provides -i %{go_import_path} -v %{version}
```

Packagers MUST NOT add or remove importable Go packages after invoking the generator, because the markers would no longer match the packaged source tree. Non-Go installation work MAY follow. If a custom install handles multiple distinct import roots, it MUST invoke the macro once for each root with explicit `-i` and `-v` arguments.

#### go_submodules

By default, each generated capability uses the version passed with `-v`. `%buildsystem_golangmodules_install` passes `%{version}`. If the source tree contains nested Go modules released at different versions, the spec MUST define `go_submodules` before invoking the generator:

```specfile
# Record the independent versions of nested Go modules.
%global go_submodules %{expand:
# PATH VERSION
example.org/project/sub 2.3.4
example.org/project/sub/nested 3.4.5

# This module may be absent on some architectures.
example.org/project/archonly 4.5.6
}
```

For each generated capability, the nearest declared module ancestor determines its version. A mapping matches when the import path is exactly `PATH` or begins with `PATH/`. For example, `example.org/project/sub` matches `example.org/project/sub/child`, but it does not match `example.org/project/submarine`. Table order does not matter. Import paths without a matching row retain the version passed with `-v`.

The table has the following requirements:

* Each non-empty, non-comment line MUST contain exactly two fields, `PATH` and `VERSION`, separated by spaces or tabs. Leading and trailing whitespace, blank lines, and full-line comments whose first non-whitespace character is `#` are allowed. Inline comments MUST NOT be used because they form an extra field.
* A valid mapping MAY remain unused when the corresponding Go package is absent from a particular build. All rows MUST still be valid on every architecture. Missing or extra fields, duplicate expanded paths, unresolved RPM macro tokens, and invalid paths or versions cause generation to fail.
* `PATH` supports ASCII letters, digits, `.`, `_`, `~`, `+`, `-`, and `/`. It MUST NOT begin or end with `/`, contain `//`, or contain `.` or `..` path components. Wildcards are not supported.
* `VERSION` supports ASCII letters, digits, `.`, `_`, `+`, `~`, and `^`, and MUST contain at least one letter or digit. The spec MUST provide only the RPM `Version` value, without an `Epoch` or `Release` suffix. For example, use `2.3.4`, not `2.3.4-1`.

### Macro Definitions

When using the following macros, you should add a one-line comment above each definition to explain its intent.

#### go_test_exclude

If you need to exclude a specific failing package during testing, you can define it at the top of the RPM Spec as follows:

```specfile
%define go_test_exclude github.com/minio/c2goasm/cgocmp
```

If you need to exclude multiple failing packages:

```specfile
%define go_test_exclude %{shrink:
    github.com/json-iterator/go/type_tests
    github.com/json-iterator/go/benchmarks
}
```

#### go_test_exclude_glob

If you need to exclude an entire class of failing packages during testing using a glob pattern, you can define it at the top of the RPM Spec as follows:

```specfile
%define go_test_exclude_glob github.com/pierrec/lz4/v4/cmd*
```

If you need to exclude multiple failing packages:

```specfile
%define go_test_exclude_glob %{shrink:
    github.com/gogo/protobuf/protoc-gen-gogo
    github.com/gogo/protobuf/test/dashfilename
    github.com/gogo/protobuf/test/embedconflict
    github.com/gogo/protobuf/test/issue270
}
```

In addition, you can achieve the same effect in the test section (`%check`) by setting `export GO_TEST_EXCLUDE_GLOB`.

#### go_test_include

If you want to test only a specific package, you can define it at the top of the RPM Spec as follows:

```specfile
%define go_test_include github.com/cespare/xxhash/v2
```

We do not recommend using this macro to include multiple packages for testing. You should generally exclude the packages that fail.

#### go_test_include_glob

If you want to test only a specific class of packages, you can define it at the top of the RPM Spec as follows:

```specfile
%define go_test_include_glob github.com/cespare/xxhash/v2*
```

We do not recommend using this macro to include multiple packages for testing. You should generally exclude the packages that fail.

In addition, you can achieve the same effect in the test section (`%check`) by setting `export GO_TEST_INCLUDE_GLOB`.

#### go_test_ignore_failure

If, for some reason, you must continue packaging even when tests fail, you can define it at the top of the RPM Spec as follows:

```specfile
%define go_test_ignore_failure 1
```

In addition, you can achieve the same effect in the test section (`%check`) by setting `export GO_TEST_IGNORE_FAILURE`.
