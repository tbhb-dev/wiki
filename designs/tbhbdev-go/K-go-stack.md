---
type: Design Appendix
title: tbhbdev Go port stack
description: The Go libraries the tbhbdev port uses for commands, flags, config, schemas, and GitHub, how it wires them without a shared dependency object or package state, and the rules that keep each library in its place.
tags: [tbhbdev, go, cobra, koanf, cue, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words cobra pflag koanf posflag viper cuelang cuecontext jsonschema gomodguard depguard gochecknoinits gochecknoglobals termio procexec selfupdate buildinfo knadh userconfig unmarshals stdlib gostd syscall golangci apptoken bradleyfalzon ghapi ghinstallation githubv shurcoo -->

# tbhbdev Go port stack

Back to [the design](README.md).

Everything here is new unless it names an existing file. The operator chose the libraries on 2026-11-25 (D18, D20, and D24), and [appendix D](D-evidence.md#the-go-stack) cites the existing behavior. Every Go path here sits under `<cli-root>` in tbhb-dev/tbhbdev ([the design](README.md#the-open-cli-folder)).

## The libraries

| Need | Library | Where it may appear |
| --- | --- | --- |
| Command tree | `github.com/spf13/cobra` | Each plugin's `io/app` and `internal/plugin/io/run` |
| Flags | `github.com/spf13/pflag`, through cobra | The same packages |
| Config | `github.com/knadh/koanf/v2`, with its file provider, TOML parser, and `posflag` provider | `internal/config/io/load` only |
| Schemas | `cuelang.org/go`: its `cue`, `cue/cuecontext`, `cue/errors`, and `encoding/jsonschema` packages | The pure schema packages `internal/schemas` and `internal/agent/protocol` |
| GitHub REST | `github.com/google/go-github/v93`, one pinned major | `internal/ghapi/**` only |
| GitHub GraphQL and app tokens | No library: a helper on the go-github client, and a standard-library JWT | `internal/ghapi/io/graphql` and `internal/ghapi/io/apptoken` |

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
// <cli-root>/internal/review/io/app/post.go, a sketch
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

### The schema source across two repositories

The Python side's `src/tbhbdev/schemas/*.cue` files are symbolic links to `core.cue`, `issue.cue`, and `review.cue` under the tools root `schemas/` directory, where `taxonomy.cue` also sits (existing, at tools `bf420ff`).

The Go code lives in another repository (D22), so no link can join the two. tools keeps the files as the source while its Python reads them. The schema slice copies them into `internal/schemas/cue/` as plain files, since Go's `embed` skips symbolic links (unverified, from its documentation). A monorepo test reads the same files from the pinned tools commit and fails on any byte difference, so a schema change lands in tools first and reaches Go through the pin bump. When the Python content libraries go, the monorepo copies become the source.

### The CUE boundary

`cuecontext.New` reads `CUE_DEBUG` and `CUE_EXPERIMENT` from the process environment (per this design's review, checked again by the schema slice against the pinned version). That is an environment read inside a pure package's call, which no import ban can see. The port pins the boundary in three parts:

- **Cleared in the shell.** `run.Main`, the I/O entry of every plugin, removes every variable whose name starts with `CUE_` from the process environment before it builds the cobra root, so no CUE call ever sees one. Children that a plugin starts get the cleaned environment, which is safe because no Go code runs `cue`.
- **One constructor.** The pure schema packages create their CUE context only through one unexported function, and an ast-grep rule, `cue-one-context`, bans `cuecontext.New` anywhere else.
- **Pinned and checked.** `cuelang.org/go` is pinned in `go.mod`. A test, `cue_env_reads_test`, lists every call to `os.Getenv`, `os.LookupEnv`, `os.Environ`, and `syscall.Getenv` in the CUE packages the module links, through `go list -deps` and a syntax scan, and fails if the set differs from the reviewed list, now the reads of `CUE_DEBUG` and `CUE_EXPERIMENT`. It also fails if any of those reads runs from an `init()` function, since clearing in `run.Main` comes after package start. A CUE upgrade that adds a read fails until a reviewer updates the list.

An `io_go_test` runs a plugin binary with `CUE_DEBUG` and `CUE_EXPERIMENT` set to values that would change or break evaluation, and checks that every schema fixture validates exactly as it does with the variables unset.

## GitHub with go-github

The operator chose the client stack on 2026-11-25, on a research report into Go GitHub clients (D24).

- **One package imports it.** Only packages under `internal/ghapi` import go-github. The rest of the code sees narrow one-method interfaces and plain result types from `internal/ghapi`, so a go-github major bump edits one directory.
- **One client.** `internal/ghapi/io/client` builds the client with an injected transport, token, User-Agent, and timeout, using only the constructor options that exist in both v93 and go-github's main branch: `WithTransport`, `WithAuthToken`, `WithUserAgent`, and `WithTimeout`. Tests inject a transport that matches fakes by request content.
- **GraphQL on the same client.** `internal/ghapi/io/graphql` posts a query through the same client for what REST lacks: marking a pull request ready for review, the merge queue state query and enqueue, and Projects v2 by node ID. It isn't a general GraphQL client, and the module doesn't use `githubv4`.
- **App installation tokens.** `internal/ghapi/io/apptoken` signs the app JWT with the standard library and calls go-github's installation endpoints, not `ghinstallation`, which pins go-github v88. It keeps today's behavior from `gh_app`: it looks up the repository's installation and then the organization's, caches tokens until near expiry, waits out a cooldown after a failure, and never puts a secret in an error.
- **Majors on tbhb-dev's schedule.** Six of the eight majors from v86 to v93 broke calls this code uses, all renames or by-value changes the compiler finds. A bump is its own pull request, never part of a feature slice.

## The rules that hold it

| Rule | Tool | Forbids |
| --- | --- | --- |
| `stack-vendors` | go-arch-lint `vendors` and `canUse`, with `depOnAnyVendor: false` as awrel sets it | cobra and pflag outside `<cli-root>/**/io/app` and `<cli-root>/internal/plugin/io/run`, koanf outside `<cli-root>/internal/config/io/load`, `cuelang.org/go` outside the two schema packages, and go-github outside `<cli-root>/internal/ghapi/**` |
| `ghapi-only-go-github` | depguard | `github.com/google/go-github/**` in any file outside `<cli-root>/internal/ghapi/**`, a second line beside `stack-vendors` with its own planted fixture |
| `front-stdlib-only` | depguard, strict list mode | Any import but `$gostd` and the front's four packages in `<cli-root>/cmd/tbhbdev`, `<cli-root>/internal/dispatch/**`, `<cli-root>/internal/selfupdate`, `<cli-root>/internal/plugin`, and `<cli-root>/internal/buildinfo` |
| `pure-no-exec` | depguard | `os/exec` and `syscall` in pure files and their tests, which the pure allowlist in [appendix F](F-build-and-checks.md#components) already leaves out, named here so a planted fixture proves it |
| `exec-one-package` | depguard | `os/exec` in every file outside `<cli-root>/internal/procexec/io/command`, so every child process starts in one package |
| `no-cue-load` | depguard | `cuelang.org/go/cue/load` and `cuelang.org/go/mod/...` everywhere, since they read files and can fetch modules |
| `no-viper` | gomodguard_v2 | The `github.com/spf13/viper` module, as awrel bans modules there |
| `no-other-github-clients` | gomodguard_v2 | The `github.com/shurcooL/githubv4` and `github.com/bradleyfalzon/ghinstallation` modules, and any go-github major but the pinned one (D24) |
| `no-cue-binary` | ast-grep, monorepo-local | A string literal equal to `cue` or ending in `/cue`, as the program of `exec.Command`, `exec.CommandContext`, `exec.LookPath`, `syscall.Exec`, or the `procexec` constructor |
| `cue-not-run` | a run-time check in `procexec`, with a unit test | Any program whose `filepath.Base` is `cue`, after the path resolves, so a computed or absolute path such as `/usr/local/bin/cue` is refused too |
| `cue-one-context` | ast-grep, monorepo-local | `cuecontext.New(...)` outside the one constructor in the schema packages |
| `no-cobra-globals` | ast-grep, monorepo-local | `cobra.OnInitialize(...)`, `cobra.OnFinalize(...)`, and any assignment to `cobra.$X` |
| `gochecknoinits` | golangci-lint | Any `init()` function |

depguard reads imports, not arguments, so it can't see which program a call runs. The two depguard exec rules put every child process behind one constructor. `no-cue-binary` catches a literal `cue` program by name or path at that door, and `cue-not-run` refuses one that the code computes, by its base name at run time. Each rule ships with a planted fixture that must fail it, as [appendix F](F-build-and-checks.md#checks) requires.
