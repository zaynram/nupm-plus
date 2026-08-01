---
title: "nupm+ — `nu`'s package manager"
purpose: This fork exists since compatibility with my existing `nupm` registry; the version of `nupm` at the time of forking had a few bugs related to installation, types, and breaking changes introduced in newer `nu` versions.
compatibility: { nu: { version: '>=0.114.0', tested-with: '0.114.2-nightly.19' } }
source: https://github.com/nushell/nupm.git
changelog: https://github.com/zaynram/nupm-plus/compare/9a28419bf91aadc58999d83e0f6ef8d9dbfa7d9f...main
---

## Table of Contents

- [*Installation*](#installation-)
- [*Configuration*](#configuration-)
- [*Usage*](#usage-)
  - [*Installing and Updating Packages*](#installing-and-updating-packages-)
  - [*Defining Packages*](#defining-packages-)
  - [*Testing Packages*](#testing-packages-)
- [*Design*](#design-)

## Notice

:warning: **This project is in an experimentation stage and not intended for serious use!** :warning:

## Installation [↑](#table-of-contents)

`nupm+` is a module, and must be manually installed (once):

Clone the repository and navigate to it:

```nu
git clone http://github.com/zaynram/nupm-plus.git
cd nupm-plus
```

Then, `use` the module in whichever way is preferable (all will produce the same command surface):

```nu
use nupm+                             # Load it as a standard module
overlay use nupm+                     # Load it as an overlay (can be hidden with `overlay hide`)
overlay use --prefix nupm+ as nupm    # Load it as an overlay as `nupm` (if you prefer the original name)
```

Since you won't want to have to be at the repository root everytime, install `nupm+` with `nupm+`:

```nu
cd ..                                     # Navigate up from the repository (if needed)
use nupm-plus/nupm+                       # Load the module (if you have not already)
nupm+ install nupm-plus --force --path    # Run the installation command
```

After installing `nupm+`, you can remove the cloned repository.

## Configuration [↑](#table-of-contents)

One can change the location of the Nupm directory with `$env.NUPM_HOME`, e.g.

```nu
# env.nu
$env.NUPM_HOME = $nu.data-dir | path basename --replace nupm
```

Because Nupm will install modules and scripts in `{{nupm-home}}/modules/` and `{{nupm-home}}/scripts/` respectively, it's a good idea to add the respective paths to `$env.NU_LIB_DIRS` and `$env.PATH`:

```nu
# env.nu
$env.NU_LIB_DIRS ++= [($env.NUPM_HOME | path join modules)]
$env.PATH = $env.PATH | split row (char esep) | prepend ($env.NUPM_HOME | path join "scripts") | uniq
```

## Usage [↑](#table-of-contents)

Nupm can install different types of packages, such as modules and scripts. It also provides a mechanism for a custom installation using a `build.nu` file.

As an illustrative example, the following demonstrates use of a fictional `foo` module-based package.
Some examples additionally incorporate an `example` registry repository.

### Installing and Updating Packages [↑](#table-of-contents)

For bare packages from a repository:

```nu
# install
git clone https://github.com/nushell/foo.git
nupm+ install foo --path

# update
git -C foo/ pull
nupm+ install foo --path
```

or

```nu
# install (run with `--force` to update)
nupm+ install --git "https://github.com/example/foo.git"
```

For packages from a configured registry:

```nu
# install (run with `--force` to update)
nupm+ install --registry=example foo
```

For packages from a non-configured registry:

```nu
with-env {
  NUPM_REGISTRIES: {
    example: 'https://raw.githubusercontent.com/example/repo/main/registry.nuon'
  }
} {
  # install (run with `--force` to update)
  nupm+ install --registry=example foo
}
```

> It may be convenient to alias `nupm+ update` to your preferred update method until a dedicated command gets implemented upstream; e.g.,
>
> ```nu
> alias "nupm+ update" = nupm+ install --force
> ```

### Defining Packages [↑](#table-of-contents)

For script packages:

```sh
├─ nupm.nuon    # metadata file
└─ run-me.nu    # script definition (`run-me` is an example name)
```

For module packages:

```sh
├─ nupm.nuon    # metadata file
└─ foo/         # package directory
  ├─ mod.nu     # module definition
  └─ ...        # other scripts and modules
```

---

The `nupm.nuon` should contain the following metadata fields:

```nu
{
    name: foo                                               # module dirname or script filename (exact)
    description: 'This is an example package description.'
    type: module                                            # module | script
    license: MIT
}
```

### Testing Packages [↑](#table-of-contents)

The original `nupm` test framework allows package maintainers to define tests for their package and run them with the `nupm test` command, and this remains true in the `nupm+` implementation:

- create a package with a `nupm.nuon` file
- create a `tests/` directory next to the `pkg/` directory
- `tests/` is a regular Nushell directory module, put a `mod.nu` there and any structure you want
- import definitions from the package with something like

#### Example

Consider a package with the following layout:

```sh
│─ nupm.nuon
├─ pkg/
│  ├─ mod.nu
│  └─ foo/
│     └─ bar.nu    # exports commands `baz` and `brr`
└─ tests/
  ├─ mod.nu
  └─ fixtures/
     └─ ...
```

In the `tests/` module, import the commands:

```nu
const NU_LIB_DIRS = [(path self ..)]
use pkg/foo/bar.nu [ baz brr ]
```

Then, define tests as exported commands from the `tests/` module:

```nu
# tests/mod.nu
# NOTE: The `test x` naming convention is not required.
# - ALL exported commands will be treated as test definitions.

# Let's say `baz` adds two numbers and returns the result, while
# `brr` concatenates two strings.
# Here's tests that we could write for checking the desired behavior:
export def "test baz" [] { assert equal (baz 1 1) 2 }
export def "test brr" [] { assert equal (brr a b) ab }
```

Finally, run the `nupm+ test` command from the repository root:

```nu
nupm+ test
```

The output should look something like this:

```text
Testing package /path/to/your/package/pkg
tests test baz ... SUCCESS
tests test brr ... SUCCESS
Ran 2 tests. 2 succeeded, 0 failed.
```

## Design [↑](#table-of-contents)

The original maintainers also request that individuals hoping to learn more take a look at [the design document](docs/design/README.md)
