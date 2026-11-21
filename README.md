# py-ncl

Python bindings to [Nickel](https://nickel-lang.org), the configuration language. Nickel runs
inside your Python process: no `nickel` executable, no subprocess per call.

```sh
pip install py-ncl
```

```python
import ncl

ncl.eval('{ port | Number = 80, name = "my" ++ "server" }')
# {'port': 80, 'name': 'myserver'}
```

The engine is [nickel-embed](https://github.com/bahiamovementcenter/rust-nickel), which every
`*-ncl` binding shares. This package only converts between Nickel and Python.

## Programs

`ncl.Program` is the `nickel` command line tool, in process:

```python
from pathlib import Path
import ncl

program = ncl.Program(
    Path("server.ncl"),                     # files are paths; several are merged
    ncl.Source.code('{ debug = true }'),    # code held in memory (Nickel, or format="json"...)
    import_paths=["lib"],                   # like -I; NICKEL_IMPORT_PATH is searched too
    contracts=["schema.ncl"],               # like --apply-contract
    overrides={
        "server.port": "8080",              # Nickel code, like --override
        "server.host": ('"0.0.0.0"', "force"),  # with a priority: "default", "force" or an int
    },
    field="server",                         # like --field
)
program.eval()             # Python values
program.export("yaml")     # json, yaml, yaml-documents, toml, text
program.typecheck(strict=True)
program.query()            # {'doc', 'type', 'contracts', 'optional', 'not_exported', 'subfields'}
```

Shortcuts: `ncl.eval(code, **options)`, `ncl.eval_file(path, **options)` and
`ncl.export(*sources, format="json", **options)`.

A plain string given as a source is a path, as on the command line. Code goes through
`ncl.eval` or `ncl.Source.code`, so a mistyped path never runs as code.

A program isn't thread-safe: use it from the thread that made it.

## Values

| Nickel | Python |
| --- | --- |
| `null`, `true`, `"text"` | `None`, `True`, `str` |
| whole numbers | `int`, exact at any size |
| other numbers | `float` |
| records, arrays | `dict`, `list` |
| `'tag` | `ncl.Tag("tag")`, a `str` |
| `'Some 3` | `ncl.Variant("Some", 3)` |

## Packages

`import name` finds packages through a `PackageMap`, built by hand from directories you
already have, or from a `Nickel-pkg.ncl` manifest:

```python
packages = ncl.PackageMap()
packages.add("greeting", "vendor/greeting")

# Or: resolve a manifest, fetching git dependencies into Nickel's cache.
packages = ncl.PackageMap.from_manifest(ncl.PackageMap.find_manifest("."))

ncl.eval('(import greeting).hello "world"', packages=packages)
```

This is how [xD](https://github.com/bahiamovementcenter/xD)'s `xd` package is used: depend on
it in your manifest (a `'Git` dependency with `path = "nickel"`), then
`(import xd).units.jd.pattern.render [1, 1, 11]` gives `"11.11"`. Nickel's package manager is
experimental, so manifest and lock formats may change.

## Errors

Everything Nickel rejects raises `ncl.NickelError`. `str(error)` is Nickel's report;
`error.kind` is one of `parse`, `import`, `typecheck`, `contract`, `eval`, `export`, `io`,
`package` or `other`; `error.diagnostics` is the structured report (severity, message, notes,
and labels with file, byte range, line and column).

## Development

```sh
python -m venv .venv && . .venv/bin/activate
pip install maturin pytest
maturin develop
pytest
```

`NCL_TEST_OFFLINE=1` skips the test that fetches xD from GitHub.

The Rust side (`src/lib.rs`) depends on nickel-embed at a pinned rust-nickel revision
(`Cargo.toml`). To move to a new rust-nickel release, change the pin, run
`cargo update -p nickel-embed`, and run the tests.

## Releases

Pushing a tag `vX.Y.Z` builds wheels for Linux (x86_64, aarch64), macOS (arm64) and Windows
(x86_64), plus a source distribution, and publishes them to PyPI through trusted publishing.
The wheels use Python's stable ABI, so one wheel per platform covers Python 3.9 and later.

## Licence

Copyright © 2026 Ash. Licensed under the
[GNU Affero General Public License v3.0 only](LICENSE) (`AGPL-3.0-only`).

Copies published before this change stay available under the MIT licence they had then.
