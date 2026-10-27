# Multiple build backends for cabal-install

## Summary

cabal-install plans a build and then runs it, and the two halves are not
separable today: what the build does is spread over `IO` closures in
`ProjectBuilding`, the `Rebuild` monad and the `Cabal` library's build
path. Every request for a different backend (Nix, #9089; Buck2, #12456;
a system package manager, #3882; a plan without an environment, #6885)
runs into the same wall.

This proposal makes the build a value. cabal-install's planner produces an
**action graph**: every program invocation and file operation the build
would perform, with its inputs and outputs as paths, its command as data,
and the tools it runs named by role. A **backend** consumes that graph.
The in-process executor becomes one backend; a Ninja export is the second
and doubles as the test that the graph is complete; Nix and Buck2 are
backends that consume the same data and own the cache and the sandbox.

Dynamism (what a build learns only by running) is part of the data model,
not an escape from it. The graph has three bounded forms of dynamism;
each backend declares which it supports and the frontend lowers the rest
by coarsening, so a backend never sees a construct it cannot run.

The design is taken from a working prototype (see Implementation Notes),
which carries cabal-install's `ProjectPlanning` and
`ProjectBuilding`, yields an action graph for every unit of a plan, and
builds its fixtures and itself from the Ninja rendering of that graph.

## Motivation

The two RFCs ask for different things on the surface. #9089 wants Nix to
own the store: garbage collection, sharing, system dependencies in the
hash, a sandbox. #12456 wants Buck2 to own the executor: a general
build cache at the granularity of individual steps, fast rebuilds, custom
build steps. Both keep cabal's solver, `.cabal` semantics and command
line. What they replace is everything from "the plan is decided" to "the
files exist", and today that part is not a thing one can hand over.

Concretely, in `cabal-install` as of 3.17:

- `ProjectBuilding.UnpackedPackage` structures a unit's build as a
  `PackageBuildingPhase` GADT whose fields are `IO` actions
  (`runConfigure :: IO (Maybe LocalBuildInfo)`, `runBuild :: IO
  [MonitorFilePath]`, ...). The phases can be sequenced and nothing else.
- Since #11703 cabal-install builds every non-`Custom` package in-library
  (`Distribution.Client.InLibrary` calling `Cabal` directly), which is
  the right place to start from. But what the build phase runs is still
  decided inside `Distribution.Simple.Build` at run time: which
  preprocessor to call, whether a C file is stale, which objects go into
  the archive. There is no point at which the list of commands exists.
- `ElaboratedConfiguredPackage` mixes the plan (what to build) with the
  machine (where programs and package dbs are). `plan.json` prints both,
  so a consumer has to know which fields are which.
- The `Rebuild` monad does IO during planning. The plan is not a pure
  function of its inputs, which is what a backend with its own cache needs
  it to be.

The external tools that exist work around this rather than through it.
`cabal-buck2` reads `plan.json`, re-derives from the `.cabal` files what
Cabal would run, and generates Buck2 rules; haskell.nix does the same for
Nix, with its own copy of a large part of Cabal's logic. Both drift from
Cabal whenever Cabal changes, and neither can support what they have not
reimplemented (`build-type: Custom`, Hooks rules, foreign libraries).
The discussion on #9089 reached the same conclusion from the other side:
"the core work of separating planning and building within
Cabal/cabal-install is good for both" (Ericson2314), and "this is the
part I wish we could start with [...] because it would force us to
refactor the existing codebase" (me, 2023).

What is wanted, then, is not a Nix backend or a Buck2 backend but the
interface they would both plug into, defined precisely enough that a
backend can be written against it without knowing Cabal.

## Proposed Change

### 1. Two graphs

Following Buck2, the build has a *target graph* and an *action graph*.

The **target graph** is what cabal-install already computes: configured
units (a component of a package under a flag assignment, Backpack
instantiation, compiler, platform and machine/stage) with `build-depends`,
`build-tool-depends` and `pkgconfig-depends` edges. It is static: no
compiler runs to compute it, and nothing found out during the build can
add a unit or an edge between units. The `ElaboratedInstallPlan` after
pruning is this graph, and it stays as it is.

The **action graph** is new. Analysis of each configured unit yields its
actions: fetch, unpack, configure, autogen, preprocess, compile, link,
register, test. An action is a value: a stable key, a command as data,
inputs and outputs as paths, and nothing else. The action graph is the
union of every unit's actions; a dependency between units is an ordinary
edge (the dependent's compile reads the dependency's registration).

The split matters for what a backend has to understand. A backend that
only wants the target graph (the "turn a plan into a package db"
interface michaelpj sketched on #9089) consumes the unit level: one
opaque action per Setup phase per unit. A backend that wants finer
caching consumes the component level. Both are views of the same value.

### 2. The action graph as data

The types below are the shape, not the final API. The prototype
implements them under these names, and the Ninja and JSON renderers
consume nothing else.

```haskell
data Graph p = Graph
  { graphPlatform   :: Platform
  , graphCompilerId :: CompilerId
  , graphRoots      :: Roots            -- where each Root is on this machine
  , graphTools      :: [(ToolName, FilePath)] -- what each role resolved to here
  , graphUnits      :: [UnitGraph p]
  }

data UnitGraph p = UnitGraph
  { unitId      :: UnitId
  , unitEdges   :: [Edge p]
  , unitOutputs :: [p]                  -- what "built" means for this unit
  }

-- One build statement, in Ninja's vocabulary.
data Edge p = Edge
  { edgeKey             :: EdgeKey      -- stable across runs: unit, component, kind, way, source
  , edgeOutputs         :: [p]
  , edgeImplicitOutputs :: [p]
  , edgeInputs          :: [p]          -- named by the command
  , edgeImplicitInputs  :: [p]          -- read, not named
  , edgeOrderOnly       :: [p]          -- must exist first; not a data dependency
  , edgeCommands        :: [Command p]  -- in order; first failure fails the edge
  , edgeDyndep          :: Maybe p      -- see §4, level 2
  , edgeDiscovered      :: [Discovered p] -- see §4, level 1
  , edgeReadsSourceTree :: Bool         -- see §4, level 0 fallback
  }

data Command p
  = Exec (Run p)                        -- a program with arguments
  | Write p ByteString                  -- a generated input (Paths_pkg.hs, a db record)
  | Copy p p | CopyTree p p | Move p p | RemoveFile p | RemoveTree p
  | Touch p | Stamp p
  | Fetch { url :: String, sha256 :: Maybe String, to :: p }
  | Unpack { tarball :: p, into :: p, exclude :: Maybe String }
  | Dyndep { ... } | MakefileDyndep { ... }   -- §4

data Run p = Run
  { runDirectory    :: p
  , runProgram      :: Program p
  , runArguments    :: [Arg p]
  , runResponseFile :: Maybe [Arg p]
  , runEnvironment  :: [(String, EnvValue)]
  }

data Program p = Tool ToolName | Built p   -- a tool of the machine by role, or a file the graph builds

newtype Arg p = Arg [Piece p]           -- pieces concatenate: "-i" <> Path dir
data Piece p = Text String | Input p | Output p | Path p
```

The rules that make this a backend interface rather than a dump:

- **No closures.** Everything a backend needs is in the value. A graph
  can be hashed, diffed, serialised and rendered without running it.
- **Edges are paths, one writer per output.** The graph is the
  producer-of-path relation. Two edges writing one path is an error at
  construction, which Ninja and Buck2 also refuse. This rules out
  cabal's shared store package db as an output (see §6).
- **Commands are values** (Buck2's `cmd_args`). A path in an argument is
  a tagged piece, so a renderer decides its spelling (relative to the
  run directory, absolute for a configure script or a Setup) and a
  backend knows the inputs and outputs a command names without parsing
  argv.
- **Paths are rooted.** `p` is instantiated to `Rooted Root`, a root
  (project, build directory, a unit's directory, a package's sources, the
  compiler's installation) and a relative path that does not climb. The
  graph carries where each root is on this machine; a backend relocates
  by rebinding roots. This is Buck2's artifact and Cabal's `SymbolicPath`.
- **Tools by role.** A command names `ghc`, `ghc-pkg`, `hsc2hs`, `ar`,
  `cc` as roles. The graph says what each resolved to here, for the
  native backend; a Nix backend substitutes derivations, a Buck2 backend
  its toolchain. This answers the "which GHC derivation?" question on
  #9089: the graph does not say, and the backend's configuration does.
- **No ambient state.** Nothing is read from the environment, the
  current directory or a cache that the graph does not name. The test is
  §5's Ninja backend: if `ninja` builds the export with cabal-install
  absent, every dependency is a file edge.

Per unit, the vocabulary of edges is fixed and small. The prototype's
list, from a pass over every spawn and file write on cabal-install's
build path:

| per unit, before components | per component | per unit, after |
|---|---|---|
| fetch, unpack | autogen (`Paths_*`, `cabal_macros.h`) | install includes, install data files |
| configure (`Configure` packages: the script, writing `.buildinfo`) | preprocess (hsc2hs, c2hs, alex, happy, cpphs) | register (write the record, `ghc-pkg recache`) |
| setup compile, setup phase (`Custom`/Hooks in Setup mode: one opaque edge per phase) | compile (`ghc --make` per way or dynamic-too pair) | test (a phony target) |
| | compile extra source (`ghc -c` per C/C++/asm/cmm/js file and way) | |
| | link (archive, `ld -r`, shared, executable, foreign library) | |

What is *not* an edge, and was IO on the build path: toolchain probes
(`ghc --info`, `ghc-pkg dump`, version checks, `pkg-config` queries),
which run once into a `BuildEnvironment`; *planning queries* (which file
is the main module, which sources need a preprocessor, whether an `.hsc`
has a `#def`, whether an `extra-lib-dirs` entry exists), answered before
the graph is built and recorded as fields; and the rebuilder's own
bookkeeping (file monitors, `plan.json`), which wraps edges and is never
inside one.

### 3. Where the graph comes from: refactoring cabal-install

The refactoring is a sequence of steps each useful on its own.

**(a) Separate the plan from the machine.** `ElaboratedConfiguredPackage`
loses the fields that are facts about this machine (program paths,
build-tool paths, package db locations) and keeps program *arguments*.
A `BuildEnvironment` (configured programs, package db stack, scratch
directory, `pkg-config` answers) is computed once per run next to the
plan, never cached and never serialised. After pruning, a `concretise`
step joins plan, environment and `DistDirLayout` into a concrete plan
whose units know their directories. The dry run, the build and
`plan.json` read the concrete plan; the plan cache and any future lock
file see only the pure plan. This is also what #6885 asks for (a plan
without an environment) and removes one source of the plan-cache growth
noted on #11703: a configured `ProgramDb`, `PATH` included, per unit.

**(b) The unit record.** `PackageBuildingPhase`'s `IO` fields become a
`UnitBuild` value: the source (local, or a tarball with its hash), the
Setup mode (in-library, or a Setup executable with its Cabal version and
closure), and per phase the flags and arguments as data. `SetupWrapper`
is split so the part that *decides* the Setup command line is pure and
the part that *runs* it is the native backend's. At this point every
unit has unit-level edges: a Setup-mode unit is complete; an in-library
unit's build phase is still one opaque edge.

**(c) Component actions in-library.** For `Simple` and `Configure`
packages, the build phase (`InLibrary.build` since #11703) becomes a list
of component actions computed from the plan and an *inventory* of the
unpacked sources (the planning queries above). The Cabal library's helpers are already mostly pure
(`GhcOptions` and `ghcInvocation`, the preprocessor invocation builders,
`Register.generateRegistrationInfo`); the prototype's analysis is ~700
lines over them. The functions on the build path that do IO they need
not do (`getRPaths` canonicalising, filtering `extra-lib-dirs` by
existence, `checkNeedsRecompilation` deciding whether a C file is stale)
move to planning queries or to the rebuilder. Registration writes the
record from the plan (`${pkgroot}`-relative, no ABI hash: GHC reads ABI
hashes from interface files, nothing reads the record's) and runs
`ghc-pkg recache` on the unit's own db, rather than round-tripping
through `Setup register`.

This step is where `Cabal` and `cabal-install` overlap, and it should be
done once, in `cabal-install`, as "the in-library build enumerates what
it would run" rather than as a second implementation. A later step can
move the enumeration into `Cabal` so that a Setup executable can also
emit actions (`Setup build --emit-actions`), which would give Setup-mode
units finer grain; that is out of scope here.

**(d) The native backend becomes a graph executor.** Initially, the
native backend interprets the graph with today's machinery: per-unit
scheduling, file monitors, the store. That is the migration path and
changes no behaviour. The target is one suspending scheduler over the
whole action graph with a constructive trace per edge (*Build Systems à
la Carte*, "cloud Shake"), which replaces `rebuildTargets`, the
`PackageFileMonitor` and the two-level `JobControl`, and gives early
cutoff and a remote cache (#9137) for free. The prototype has this
engine (~800 lines, generic over the action type).

### 4. Dynamism as data

Everything a build learns by running falls into three forms. Each is
part of the graph, each is bounded, and a backend that lacks one is given
a coarser graph instead.

The invariant that makes this work: **dynamism is confined inside a
unit, and adds edges, never nodes between units.** A unit's actions may
depend on each other and on finished outputs of units the target graph
already names. Nothing discovered at build time can add a unit or an
edge between units. An import that no `build-depends` provides is an
error at the unit's discovery edge, not a new dependency.

**Level 1, discovered inputs.** After an edge runs, it reports files it
read that it could not name beforehand: the headers a C compile included
(the C compiler's `-MD` depfile), the files Template Haskell declared
through `addDependentFile` (GHC's `-ddump-hi` usages). The edge's
`edgeDiscovered` names where the report is and in what format. They are
inputs for the *next* run: a first run is never wrong because there is
nothing to hit. Ninja's `deps`, Buck2's dep files, Shake's `needed`.

**Level 2, discovery edges.** An edge runs after its producers and writes
a file of edges that the scheduler reads before scheduling the edges that
name it (`edgeDyndep`). The file may add inputs and implicit outputs to
*existing* edges; it may not add an edge. Two uses: the module graph of a
component (`ghc -M -include-cpp-deps`, or a plugin), which gives each
compile its home-module interfaces and CPP includes and is what makes
per-module compilation and the `--make` node exact; and the scan for the
C sources `c2hs` wrote, whose compile edges exist from the start as
candidates and are demanded only if the scan lists their object. Ninja's
`dyndep`; Buck2's `dynamic_output` with pre-declared outputs.

**Level 3, dynamic subgraphs.** An edge produces a *graph fragment*: new
edges with new outputs. This is what Hooks pre-build rules are
(`dynamicRule`: a dependency computation returning `[Dependency]`, then
the rule command), what `test-code-generators` are (a program whose
output lists modules), and what a Custom `Setup.hs` is in the limit. The
fragment is a `Graph` value in the same IR with two constraints: its
outputs lie under a declared *envelope* (an output directory of the
enclosing unit, declared statically), and its inputs are either inside
the envelope or already inputs of the enclosing unit. So the enclosing
graph knows the fragment's footprint before it exists. Nix's dynamic
derivations (RFC 92: a derivation whose output is a `.drv`) are exactly
this; Buck2's `dynamic_output` covers it when the envelope is a declared
output directory (to be verified against the Buck2 rule API); Ninja has
nothing.

**Backend capabilities and coarsening.** A backend declares what it runs:

```haskell
data Capabilities = Capabilities
  { capDiscoveredInputs :: Bool   -- level 1
  , capDiscoveryEdges   :: Bool   -- level 2
  , capDynamicSubgraphs :: Bool   -- level 3
  , capSandboxed        :: Bool   -- inputs are the whole input set; nothing else is visible
  }
```

The frontend lowers a graph to a backend's capabilities by **coarsening**:
a region of the graph that uses a form the backend lacks is folded into
one opaque edge whose inputs are the region's static inputs (plus the
unit's whole source tree, `edgeReadsSourceTree`) and whose outputs are
the region's declared outputs or envelope. Coarsening is a graph-to-graph
function in the frontend, so each backend is written against the forms
it supports and nothing else:

| | native | Ninja | Buck2 | Nix |
|---|---|---|---|---|
| level 1 | traces | `depfile`/`deps` | dep files | not needed: sandboxed, the unit's sources are the input |
| level 2 | scheduler reads the file | `dyndep` | `dynamic_output` | coarsen to the unit |
| level 3 | scheduler extends the graph | coarsen to the unit | `dynamic_output` over an output dir, or coarsen | dynamic derivations, or coarsen |

Coarsening to "one derivation per unit" is what haskell.nix and the
cabal store do today, so the Nix backend's first version loses nothing
against the status quo and gains the component level where it wants it.

Two things this does **not** promise. First, a sandboxed backend cannot
see a file a Template Haskell splice reads through `runIO` without
declaring it; neither can anything else, and the native backend's
behaviour (rebuild only when a declared input changes) is the best
available. Second, programs that must run *before* a unit's graph exists
(Hooks' pre-configure component hook adding modules; `test-code-generators`)
are level 3 if the backend has it, and otherwise force the unit to be
analysed after that program's unit is built, which is a staging of
analysis, not a hole in the graph. The prototype treats this as its open
item 5 and the proposal does too (Open Questions).

On the granularity question dcoutts raised on #12456: the IR does not fix
it. The first cut is one `ghc --make` per component with level 1
discovered inputs, which is what cabal does today and what `cabal-buck2`
does. Per-module compilation is a level 2 refinement of the same unit's
graph (one compile edge per module and way, one discovery edge per
component) and needs nothing new from backends that have `dyndep`. The
improvements to `ghc -M` that make it correct are wanted either way; the
graph is where their output would go.

### 5. Backends

**Native.** In-process, today's behaviour, then the trace-based executor
of §3(d). The store stays as it is for dependencies; see §6 for the one
conflict.

**Ninja.** `cabal plan --format ninja -o DIR` writes `build.ninja`,
`vars.ninja` (tool roles as variables) and the generated inputs. It is
an export and an oracle: CI builds cabal-install from its own export with
cabal-install absent, which is the proof that the graph is complete. It
is also useful on its own for anyone who wants a build they can read.

**Nix.** A backend, in or out of tree, that turns the graph into
derivations and submits them with `nix derivation add` (no Nix language
involved, which is what #9089 asks for). Fetch edges are fixed-output
derivations (the hash is in the plan). Tools and `pkgconfig-depends`
resolve through a user-supplied map from role to store path or
derivation; a missing entry is an error that names the role, not a
search. The unit boundary is a package db directory, so a local package
built natively can depend on a store path Nix produced: the graph is
partitioned at unit edges, dependencies go to Nix, workspace packages
stay in-process and incremental. This is fgaz's interleaving concern
answered by construction rather than by scheduling.

**Buck2.** What `cabal-buck2` does, with the IR in place of `plan.json`
plus re-derived Cabal logic. The intermediate build spec Simon describes
`cabal-buck2` generating is this graph; the generator becomes a renderer
over it, and gains what it lacks today (`Custom` via coarse Setup edges,
Hooks rules via level 3, foreign libraries) without reimplementing them.

**External backends** read the JSON rendering of the graph
(`cabal plan --format json`, a versioned schema) and are ordinary
`cabal-<name>` external commands. In-tree backends (native, Ninja) link
the types. No Backpack, no plugin loader: the interface is data and a
process boundary.

### 6. The store

One rule of §2, one writer per output, meets cabal's store head on: every
registration into `store/<ghc>/package.db` rewrites one `package.cache`.
The prototype's answer is nix's: **each unit's registration is its own
package db** (`store/<ghc>/<unit>/package.db`, one entry), and a
dependent's compile names the dbs of its closure through the response
file. The shared store db survives as a derived view for tools that
want one (`ghc-pkg` users, `cabal path`), rebuilt from the per-unit dbs,
never written by a build edge. This also removes the registration lock
and lets registrations run in parallel. It is a store layout change and a
one-way door; see Open Questions.

### 7. Command surface

- `cabal plan [--format json|ninja] [-o PATH] [TARGET...]`: the
  action-level query. Reads the plan, fetches and unpacks what the
  inventory needs, writes the graph, builds nothing. (#6885's
  `project-plan`, at the action level.)
- `cabal build --backend=NAME`: run the graph through a backend.
  `native` is the default. A project-file field `backend:` is natural
  but not proposed here.
- `plan.json` gains the per-unit `edges`; existing fields are unchanged.

## Alternatives Considered

**Replace "plan → package db" only** (michaelpj, #9089). A backend that
consumes the unit level of the graph is exactly this, and it is the
right first target for Nix. On its own it keeps Cabal's build logic
opaque, so Buck2's step-level caching and per-module builds are out of
reach, and `cabal-buck2` keeps its own copy of Cabal.

**Extend `plan.json` and let tools re-derive the rest** (the status quo
for `cabal-buck2` and haskell.nix). The plan says what to build, not what
running it means; every consumer reimplements `Distribution.Simple` and
drifts.

**`build-type: Buck2`** (Simon's option 3). Orthogonal: it is a frontend
question (how a package describes its own rules), not a backend one. The
IR does not preclude it; a package with its own rules would be a level 3
fragment.

**A Shake-based executor without the IR.** Shake has dynamic
dependencies and early cutoff, but its rules are closures, so a Shake
build cannot be exported. The engine part of Shake is what the native
backend reimplements, over a graph it already has as data.

**Backpack-pluggable backends.** An in-process plugin interface would
have to be stable across cabal-install releases and would still need the
data types; the data types with a process boundary are the smaller
contract, and PR haskell/cabal-proposals#8 (a tiered cabal-install API)
is where the in-tree types would be published.

**Per-module builds first, backends later** (dcoutts, #12456). Compatible,
and the order matters: per-module compilation written into today's
`IO`-closure builder would be a third thing to defunctionalise. With the
graph in place it is a change to one unit's analysis.

## Backwards Compatibility / Migration

- No user-visible change by default. The native backend reproduces
  today's behaviour through step (d)'s first stage, and the test suite
  runs unchanged against it.
- `build-type: Custom` and Hooks-through-Setup are supported on every
  backend at Setup-phase granularity, which is their granularity today.
- `plan.json` is extended, not changed. The JSON graph schema is
  versioned from its first release.
- The store layout change (§6) is opt-in behind a store version and can
  ship separately; the shared db view keeps `ghc-pkg`-based workflows
  working.
- Hooks: when cabal-install speaks the Hooks protocol in-library, rules
  are level 3 fragments; a backend without level 3 gets the unit
  coarsened. No change to `Cabal-hooks`.

## Interested parties

- fgaz, michaelpj, angerman, Ericson2314, TravisWhitaker (#9089, Nix,
  haskell.nix)
- simonmar (#12456, `cabal-buck2`)
- dcoutts, mpickering (per-module builds, `ghc -M`, Hooks)
- phadej (#6885)
- zlonast (cabal-proposals#8: the graph types are a tier-0 API consumer
  request)
- GHC developers, for `ghc -M` improvements and `-include-cpp-deps`
- Distribution packagers, who get a readable build out of `cabal plan`

None contacted yet beyond the issue threads. The proposal should be
cross-posted to both RFCs.

## Implementation Notes

I am willing to implement this and have most of the design running in a
prototype. The prototype carries cabal-install's `ProjectPlanning` and
`ProjectBuilding` (52 of their 90-module import closure, restructured as
in §3(a)–(c)), computes the action graph for every unit of a plan
(`Simple`, `Configure`, `Custom` and Hooks via Setup mode; hsc2hs, alex,
happy, and c2hs untested; all four ways; foreign libraries; Backpack
mix-in linking), renders it as JSON and as Ninja, and builds its fixture
projects and itself from the Ninja export. The self-export is about
1070 edges over 99 units, 85 of them tarballs fetched by `curl` with the
hash checked, `network`'s configure script and a plan-built `hsc2hs`
among them; a second `ninja` has nothing to do, `ninja test` passes, and
CI checks the fixpoint (the prototype binary that the committed export
builds writes that export back byte for byte). Checked on Linux with GHC
9.12 only; not on Windows or macOS, and not with `pkgconfig-depends`. The
JSON rendering is the `plan.json` extension of §7. What the prototype
does *not* have is the capability/coarsening layer of §4 (its one
external backend is Ninja, which has levels 1 and 2) and per-module
compilation (designed, not built). Its design document records the
alternatives rejected on the way, in particular keeping `GhcOptions`
records in actions (paths become text) and an applicative "wiring"
layer over the IR (a second edge language).

<!-- TODO(andrea): the prototype repository is private as of 2026-10-27. Either publish it with the proposal or replace the references above with the relevant extracts. -->

Suggested order, each step landing on its own:

1. §3(a), plan/environment split. Pure refactoring with measurable
   benefit (#6885; the plan-cache size).
2. §3(b), the unit record and `cabal plan --format json` at unit level.
   A Nix backend can start here.
3. §3(c), component actions for in-library units; the Ninja backend; a
   CI job that builds cabal-install from its export.
4. §4, capabilities and coarsening; the JSON schema frozen at v1.
5. §3(d), the trace-based native executor.
6. Out-of-tree backends migrate (`cabal-buck2`), a Nix backend is
   written, store layout (§6).

Steps 1–3 are months, not weeks, and depend on review bandwidth more
than on code: the prototype's versions of them exist and the work is
re-expressing them as reviewable commits against `master`.

## Open Questions

1. **Per-unit package dbs in the store** (§6). The one-writer rule makes
   it necessary for the native executor over a graph; it is a one-way
   door for the store layout. Is a store version bump acceptable, and
   does the shared-db view need to be more than `ghc-pkg recache` over a
   directory of symlinks?
2. **Nix granularity.** One derivation per unit (store-shaped, simple,
   coarse) or per action with content-addressed derivations (early
   cutoff, more derivations than Nix is used to). The IR supports both;
   the first backend should pick one and measure.
3. **Programs that run at analysis time** (§4). Stage analysis so that a
   unit whose graph needs a built program is analysed after it, or make
   such programs level 3 fragments with an envelope. The Hooks protocol
   fixes when each hook is called, so this is one-way for Hooks and
   two-way for `test-code-generators`.
4. **Tool binding format** for external backends: a JSON map from role
   to path/derivation on the command line, or a section of the project
   file.
5. **Does `Cabal` eventually emit actions** (`Setup build
   --emit-actions`), giving Setup-mode packages component-level graphs?
   Not needed for anything above, but it would retire the "Custom is
   opaque" limit without retiring Custom.
6. **Windows.** The prototype's Ninja rendering for `cmd` is written and
   unchecked. Everything in §2 is platform-neutral by construction, but
   nothing has run there.
7. **Schema ownership.** Who versions the JSON graph and how a breaking
   change is announced to external backends; this is the same question
   PR cabal-proposals#8 asks for the library API.

## References

- haskell/cabal#9089, RFC: Nix backend
- haskell/cabal#12456, RFC: Buck2 backend
- haskell/cabal#3882, Interface for system package manager to execute
  build plan
- haskell/cabal#6885, Add `project-plan` command
- haskell/cabal#9137, Remote cache in the style of Bazel
- haskell/cabal-proposals#8, Designing the API for cabal-install
- `cabal-buck2`, https://hackage.haskell.org/package/cabal-buck2
- haskell.nix, https://input-output-hk.github.io/haskell.nix
- Pier, https://github.com/judah/pier
- Buck2, *Dynamic dependencies*,
  https://buck2.build/docs/rule_authors/dynamic_dependencies/
- Ninja, *dyndep*, https://ninja-build.org/manual.html#ref_dyndep
- Nix RFC 92, *Plan dynamism*; RFC 134, *Nix store layer*
- Mokhov, Mitchell, Peyton Jones, *Build Systems à la Carte: Theory and
  Practice*, JFP 2020
- `Distribution.Simple.SetupHooks.Rule` (`staticRule`, `dynamicRule`),
  Cabal 3.14+
