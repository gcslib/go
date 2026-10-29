# gcslib/go

Go programs built from the modules your `go.sum` names, offline, with a
pinned Go: one package at a time, each a step of its own.

A GCSL library for pekit, Peios's build tool. Import it as
`github.com/gcslib/go`.

## Usage

In `workspace.gcsl`, register its toolchain, and the environment it runs
on:

```text
import Toolchain::* from "github.com/gcslib/buildenv"
import Toolchain::* from "github.com/gcslib/gcc"
import Toolchain::* from "github.com/gcslib/go"
import Toolchain from toolchain

pekitconf = {
    toolchains = [Toolchain::C gcc_x86_64, Toolchain::Go go_1_26, Toolchain::BuildEnv busybox_x86_64]
}
```

Pin the module files `go.sum` names, with the network, in a script:

```text
import "github.com/gcslib/go"

def script pin_modules() {
    let count = go.pin_modules(file("go.sum"), "{dir(".").path}/go.pins")
}
```

`pekit run pin_modules` writes `go.pins`. Run it again after a change to
`go.sum`, and commit both.

In a build file, plan the module and build what you need of it:

```text
import "github.com/gcslib/go"

fn module() -> go.GoWorkspace {
    = go.workspace(dir("."), file("go.sum"), file("go.pins"))
}

def build app() {
    = go.bin(module(), "./cmd/app")
}

def test store(tests: List<go.GoTest> = go.tests(module(), "./internal/store")) {
    let ran = for each in tests { = go.run_test(each) }
}
```

Each package is a step, compiled as the go command compiles it, so a
change rebuilds the packages it reaches and no others, and programs that
use the same packages share them.

## What it provides

- `go_1_26`: Go 1.26.8 from go.dev, pinned by its SHA-256, running on the
  buildenv library's pinned busybox environment.
- `pin_modules(sum, out)`: for a script, run with the network: writes
  `go.pins`, the SHA-256 of each module file `sum` names, as the module
  proxy serves it. It uses the host's `curl` and `sha256sum`.
- `proxy(sum, pinned_by)`: the module files `sum` names, each fetched by
  its pin, laid out as a module proxy serves them. Go checks each against
  `sum` as it reads it, so the pins name the bytes and `go.sum` what they
  must be.
- `workspace(src, sum, pinned_by)`: a Go module, planned for `goos` and
  `goarch`, by default Linux on x86-64, in `buildmode`, by default `exe`,
  with the build tags `tags`.
- `bin(ws, package)`: a main package's program, linked with `ldflags`, with
  the build information `go build` embeds.
- `tests(ws, package)`: a package's test program, as `go test -c` builds
  it; `run_test(test)` runs one in its package's directory, with `given`,
  `env` and `args` if given.
- `vet(ws)`: `go vet` of every package, as one step.
- `race(ws)`: `go test -race` of every package, as one step, with the
  registered C toolchain, which the race detector's runtime links with.
- `module_zip(pinned, module, version)`, `module_source(file)`,
  `sum_files(sum)`, `pins(file)` and `escaped(text)`: the parts the rest
  are made of.

The standard library is built once for each configuration, by the go
command, as one step. Each other package is compiled by `compile`, its
assembly by `asm`, and its archive packed by `pack`, which the library
builds from the toolchain's own source. A package's embedded files are
given to its compile with the go command's own configuration of what each
pattern matches.

Planning reads the sources' imports, as the go command does, so an edit
plans again. A package whose files and dependencies did not change is found
in the store.

A package with cgo files is an error: the library does not build cgo yet.
Neither does it build a module replaced by a local directory.

It needs the [buildenv](https://github.com/gcslib/buildenv) library, and
the [gcc](https://github.com/gcslib/gcc) library for `race`.

The comments at the top of `go.gcsl` and `packages.gcsl` document each in
full.

## Status

gcslib is new, and its libraries' APIs can still change. Pin the commit you
build with: `pekit lock` does so in `pekit.lock`.

## Licence

MIT. See [LICENSE](LICENSE).
