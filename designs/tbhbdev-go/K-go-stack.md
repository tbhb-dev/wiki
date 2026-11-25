---
type: Design Appendix
title: tbhbdev Go port stack
description: The Go libraries the tbhbdev port uses for commands, flags, config, and schemas, how it wires them without a shared dependency object or package state, and the rules that keep each library in its place.
tags: [tbhbdev, go, cobra, koanf, cue, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words cobra pflag koanf posflag viper cuelang cuecontext jsonschema gomodguard depguard gochecknoinits gochecknoglobals termio procexec selfupdate buildinfo knadh userconfig unmarshals stdlib gostd syscall golangci -->

# tbhbdev Go port stack

Back to [the design](README.md).

Everything here is new unless it names an existing file. The operator chose the libraries on 2026-11-25 (D18 and D20), and [appendix D](D-evidence.md#the-go-stack) cites the existing behavior.

## The libraries

| Need | Library | Where it may appear |
| --- | --- | --- |
| Command tree | `github.com/spf13/cobra` | Each plugin's `io/app` and `internal/plugin/io/run` |
| Flags | `github.com/spf13/pflag`, through cobra | The same packages |
| Config | `github.com/knadh/koanf/v2`, with its file provider, TOML parser, and `posflag` provider | `internal/config/io/load` only |
| Schemas | `cuelang.org/go`: its `cue`, `cue/cuecontext`, `cue/errors`, and `encoding/jsonschema` packages | The pure schema packages `internal/schemas` and `internal/agent/protocol` |

None of them appears in the front, `internal/dispatch`, or `internal/selfupdate`. awrel already builds its command with cobra 1.10.2, under go-arch-lint vendor rules like the ones below (existing).

## Commands with cobra

### Why the front doesn't use cobra

The front stays on the standard library, and `dispatch.Resolve` stays a pure function over argv and the manifest. The front runs before any flag parsing, and it must hand unknown names and flags through untouched to a plugin or to Python. A flag parser consumes or rejects flags it doesn't define, which is why gh turned off flag parsing for extensions and later matched cobra's error text ([appendix B](B-gh-pitfalls.md#14-matching-the-unknown-command-error-string)).

The front also starts on every hook call, so each library it links adds start-up time to every tool call ([appendix B](B-gh-pitfalls.md#51-dependency-start-up-costs)). Cobra lives in the plugin binaries, where each flag belongs to the command that defines it.

### Explicit dependencies and no shared object

gh's pain with cobra came from its wiring: one shared Factory passed to 216 command constructors, and package state that `init()` read before flags were parsed ([appendix B](B-gh-pitfalls.md#3-the-shared-dependency-object)). tbhbdev wires cobra as awrel's `cmd/awrel/main.go` does, where a function builds the root from the streams and environment lookup it receives as arguments (existing). The command shape is gh's own `label delete`: a private options struct, narrow dependencies, and a run function ([appendix A](A-gh-architecture.md#how-commands-are-tested)).

- **One constructor per command, with its own dependencies.** Each constructor takes a private struct with only the values and one-method interfaces that command uses. No struct is shared between commands, and no type is named `Factory`.
- **No registration in `init()`.** A parent adds its children in its own constructor with `AddCommand`, and `gochecknoinits` runs on every package.
- **No package state.** Commands, flag variables, and config live in locals and closures. `gochecknoglobals` keeps its one stamp exception, and an ast-grep rule bans cobra's global hooks and settings.
- **One place for errors.** `run.Main` sets `SilenceUsage` and `SilenceErrors`, turns flag errors into a usage error, and maps every returned error to an exit code through `plugin.ExitCode`.

```go
// internal/review/io/app/post.go, a sketch
type postDeps struct {
	config func() (config.Config, error) // loads at most once, on first call
	poster reviewPoster                  // one method over the GitHub adapter
}

func newPostCmd(d postDeps) *cobra.Command {
	return &cobra.Command{
		Use:  "post <number>",
		Args: cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			cfg, err := d.config()
			if err != nil {
				return err
			}
			plan, err := review.PlanPost(args[0], cfg.Review) // pure
			if err != nil {
				return err
			}
			return d.poster.Post(cmd.Context(), plan)
		},
	}
}
```

A test builds the command with fakes in its struct and calls `SetArgs`, so no test swaps a global. Pure functions such as `review.PlanPost` keep their own table tests, with no cobra in them.

### The plugin contract under cobra

- `run.Main(name, newRoot)` parses the contract environment, answers a lone `--version` before cobra sees it, builds the root, and calls `ExecuteContext` with the argv tail.
- The root's `Use` comes from `TBHBDEV_PROG`, so usage lines read `tbhbdev review post`.
- Cobra's hidden `__complete` command answers the contract's reserved completion argument (unverified against cobra 1.10).
- A flag error or an unknown subcommand exits 2, as the contract requires ([appendix E](E-plugin-contract.md#exit-codes)).

## Config with koanf

The Python `userconfig` reads one file, `$XDG_CONFIG_HOME/tbhbdev/config.toml`, when a value is first needed, so `--help` works without it (existing). Go keeps that file, its format, and that timing.

- **Loaded once, in the input/output (I/O) shell.** `internal/config/io/load` makes one koanf instance in a local, loads the TOML file, overlays the flags a command maps explicitly through `posflag`, and unmarshals into the plain `config.Config` struct. The koanf instance then goes out of scope.
- **Passed as a value.** Pure code receives `config.Config`, or the one table it needs, and never imports koanf. A pure `config.Check` names each missing `[table] key`, as the Python messages do.
- **Lazy per command.** `io/app` hands each command a loader that runs at most once per process, and only when the command calls it. `tbhbdev review --help` therefore works with a broken config file ([appendix B](B-gh-pitfalls.md#36-lazy-in-name-only)).
- **No environment layer.** The Python loader reads no environment overrides, so the Go loader adds none.

Viper stays out of the module, because its usual entry points act on a package-level default instance (unverified, from its documentation). koanf has no default instance, so each load is a value the shell owns (unverified, the same).

## Schemas with CUE in Go

Go has a native package for Configure Unify Execute (CUE), `cuelang.org/go`, so the port never runs the `cue` binary (D20). Python runs it today: `cue_runtime.py` downloads a pinned `cue`, and `content_vet.py` runs `cue vet` and `cue export` on it as child processes (existing).

- **Embedded.** Each schema sits in its pure package and enters the binary through `go:embed`. No schema is read from disk at run time.
- **Validated in-process.** A pure function compiles the embedded bytes with `cuecontext`, unifies the document with the definition it names, and returns findings. It takes bytes and returns data, so it carries `unit_go_test` table tests.
- **JSON Schema through the same library.** The D16 protocol schema stays the JSON Schema file it names, and the review draft and policy schemas stay JSON Schema too. Go reads them through `cuelang.org/go/encoding/jsonschema` in the same pure package, and Python keeps `jsonschema`.

| Schema | Python today | Go |
| --- | --- | --- |
| Issue and pull request templates: `core.cue`, `issue.cue`, and `taxonomy.cue` | `cue vet` as a child process | `internal/schemas`, as CUE |
| Review documents: `review.cue` | The same | `internal/schemas`, as CUE |
| Review draft and review policy schemas | `jsonschema` | `internal/schemas`, through `encoding/jsonschema` |
| Run request and run metadata: `schemas/agent-protocol.json` | `jsonschema`, from plan step 2 | `internal/agent/protocol`, through `encoding/jsonschema` |

### Moving the schema source

The Python side's `src/tbhbdev/schemas/*.cue` files are the source that moves over. At tools `bf420ff` they're symbolic links to `core.cue`, `issue.cue`, and `review.cue` under the root `schemas/` directory, where `taxonomy.cue` also sits (existing).

Go's `embed` skips symbolic links (unverified, from its documentation), so the files themselves move into `internal/schemas/cue/` and the Python links point there. Both languages then read one source. The schema slice moves the files and the links in one pull request, which the Go tests and the Python suite both run on.

## The rules that hold it

| Rule | Tool | Forbids |
| --- | --- | --- |
| `stack-vendors` | go-arch-lint `vendors` and `canUse`, with `depOnAnyVendor: false` as awrel sets it | cobra and pflag outside `io/app` and `internal/plugin/io/run`, koanf outside `internal/config/io/load`, and `cuelang.org/go` outside the two schema packages |
| `front-stdlib-only` | depguard, strict list mode | Any import but `$gostd` and the front's four packages in `cmd/tbhbdev`, `internal/dispatch/**`, `internal/selfupdate`, `internal/plugin`, and `internal/buildinfo` |
| `pure-no-exec` | depguard | `os/exec` and `syscall` in pure files and their tests, which the pure allowlist in [appendix F](F-build-and-checks.md#components) already leaves out, named here so a planted fixture proves it |
| `exec-one-package` | depguard | `os/exec` in every file outside `internal/procexec/io/command`, so every child process starts in one package |
| `no-cue-load` | depguard | `cuelang.org/go/cue/load` and `cuelang.org/go/mod/...` everywhere, since they read files and can fetch modules |
| `no-viper` | gomodguard_v2 | The `github.com/spf13/viper` module, as awrel bans modules there |
| `no-cue-binary` | ast-grep, tools-local | `"cue"` as the program of `exec.Command`, `exec.CommandContext`, `exec.LookPath`, `syscall.Exec`, or the `procexec` constructor |
| `no-cobra-globals` | ast-grep, tools-local | `cobra.OnInitialize(...)`, `cobra.OnFinalize(...)`, and any assignment to `cobra.$X` |
| `gochecknoinits` | golangci-lint | Any `init()` function |

depguard reads imports, not arguments, so it can't see which program a call runs. The two depguard exec rules put every child process behind one constructor, and `no-cue-binary` catches `cue` by name at that door. Each rule ships with a planted fixture that must fail it, as [appendix F](F-build-and-checks.md#checks) requires.
