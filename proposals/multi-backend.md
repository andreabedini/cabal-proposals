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
and the tools it runs named by role. A **backend** executes that graph.
Today's builder becomes the native backend, with no change in behaviour.
A new `cabal plan --format json|ninja` writes the graph out, and building
the Ninja rendering with cabal-install absent is the test that the graph
is complete. Nix and Buck2 integrations could then consume the same graph
instead of re-deriving Cabal's logic; writing them is not part of this
proposal.

Some steps learn their dependencies only by running: the headers a C file
includes, a component's module graph, Hooks rules. §4 describes them in
the data model, in three bounded forms, so that a backend that cannot
handle one of them is given a coarser graph instead.

The design comes from a working prototype (see Implementation Notes).
It contains a restructured copy of cabal-install's `ProjectPlanning` and
`ProjectBuilding`, produces an action graph for the units of the plans
it has been tried on, and builds its fixtures and itself from the Ninja
rendering of that graph.

### What this proposal asks for

This proposal seeks agreement on the following:

1. **The direction.** cabal-install represents the build it is about to
   run as data (the action graph, §2 and §4), and today's in-process
   build becomes one consumer of that data.
2. **Steps 1–3 of Implementation Notes**, each landing on its own:
   separating the plan from the build environment (§3(a)); the per-unit
   build record and a unit-level `cabal plan --format json` (§3(b)); and
   component actions for in-library units, with a Ninja export built in
   CI (§3(c)). The JSON and Ninja outputs ship as experimental, with no
   compatibility promise.

The rest of this document presents the design as it stands, so that the
direction can be judged against something concrete. It is provisional,
and several parts still need design work; §8 lists them. A complete design of the store layout (§6), a stable JSON schema and its versioning, the trace-based native executor (§3(d)), or `--backend` (§7) are left to subsequent proposals, and accepting this one does not represent any commitment to them.

## Motivation

The two RFCs, #9089 (Nix) and #12456 (Buck2), ask for different things
on the surface. #9089 wants Nix to own the store: garbage collection,
sharing, system dependencies in the hash, a sandbox. #12456 wants Buck2
to own the executor: a general build cache at the granularity of
individual steps, fast rebuilds, custom build steps. Both keep cabal's
solver, `.cabal` semantics and command line. What they replace is
everything from "the plan is decided" to "the files exist", and today
that part is not a thing one can hand over.

Concretely, in `cabal-install` 3.18 and on `master`:

- `ProjectBuilding.UnpackedPackage` structures a unit's build as a
  `PackageBuildingPhase` GADT whose fields are `IO` actions
  (`runConfigure :: IO InLibraryLBI`, `runBuild :: IO
  [MonitorFilePath]`, ...). The phases can be sequenced and nothing else.
- Since #11703 (first released in 3.18.1.0) cabal-install builds most
  non-`Custom` packages in-library (`Distribution.Client.InLibrary`
  calling `Cabal` directly), which is the right place to start from. The
  exceptions are a package whose `cabal-version` is newer than the
  linked `Cabal`, and a `Hooks` package whose hooks executable cannot be
  used in-library; both fall back to an external Setup. Even in-library,
  what the build phase runs is still decided inside
  `Distribution.Simple.Build` at run time: which preprocessor to call,
  whether a C file is stale, which objects go into the archive. There is
  no point at which the list of commands exists.
- `ElaboratedConfiguredPackage` mixes the plan (what to build) with the
  machine (configured program paths, package db stacks). `plan.json`
  prints some machine facts (`dist-dir`, `bin-file`, the compiler) next
  to plan facts, so a consumer has to know which fields are which.
- The `Rebuild` monad does IO during planning. The plan is a function of
  the inputs its file monitors track, but not a pure function of values
  one can hand to it, which is what a backend with its own cache needs.

The external tools that exist work around this rather than through it.
`cabal-buck2` links the cabal-install library (`ProjectOrchestration`,
`SetupWrapper`, `InLibrary`) and runs Cabal's own configure to learn what
each component needs, then generates Buck2 targets; it is tied to one
version of that library. Linking cabal-install costs both sides: during
the 3.18.2.0 release, fixes such as #12289 were not backported because
they would break the library's API (#12326), and its reverse
dependencies pin narrow version bounds, some of them no longer building
(haskell/cabal-proposals#8). haskell.nix builds each component through
Cabal's Setup interface, `Custom` included, but keeps its own
translation of `.cabal` files and projects into Nix and its own maps
from system dependencies to nixpkgs. Neither has an interface to build
against, so each follows Cabal's changes by hand. Two comments on #9089
point the same way: "the core work of separating planning and building
within Cabal/cabal-install is good for both" (@Ericson2314, of an
in-tree Nix backend and haskell.nix), and "this is the part I wish we
could start with [...] because it would force us to refactor the
existing codebase" (@andreabedini, 2023).

Users who never leave the native backend gain too. Separating the plan
from the machine shrinks the plan cache and gives #6885 a plan without
an environment (§3(a)). Steps 1–3 also give a readable export of the
build (`cabal plan --format ninja`), which distribution packagers and
anyone debugging a build can use. Later steps, which are not asked for
here, would give a per-file C build with tracked headers, parallel
registration, early cutoff and a remote cache (#9137).

What is wanted, then, is not a Nix backend or a Buck2 backend but the
interface they would both plug into, defined precisely enough that a
backend can be written against it without knowing Cabal.

## Proposed Change

### 1. Two graphs

Following Buck2, the build has a *target graph* and an *action graph*.

The **target graph** is what cabal-install already computes: configured
units (a component of a package under a flag assignment, Backpack
instantiation, compiler, platform and machine/stage) with
`build-depends`, `build-tool-depends` and `pkgconfig-depends`
dependencies between them. It is static: no compiler runs to compute it,
and nothing found out during the build can add a unit or a dependency
between units. The `ElaboratedInstallPlan` after pruning is this graph,
and it stays as it is.

The **action graph** is new. *Analysis* of each configured unit
(computing its actions from its configuration and its sources) yields
its actions: fetch, unpack, configure, autogen, preprocess, compile,
link, register, test. An action is a value: a stable key, a command as
data, inputs and outputs as paths, and nothing else. The action graph is
the union of every unit's actions. A dependency between units needs no
special form: the dependent's compile reads a file that the dependency's
register action writes.

**Analysis and configure.** "Configured" already means two things in
Cabal. A *configured unit* in cabal-install's plan is a unit whose flags,
dependencies and components have been chosen
(`ElaboratedConfiguredPackage`): a node of the target graph above.
*Configuring a package* is a build phase (`Setup configure`, or the
in-library configure since #11703), which produces the `LocalBuildInfo`
that the later phases read. Analysis takes the place of the second. For
an in-library unit there is no configure step left, because what
configure does today falls into four parts, each of which has a home
elsewhere:

- **Decisions the planner has already made**: dependency selection,
  component and Backpack configuration, install directories, flag
  assignment, language and extension checks. Today configure re-derives
  them from the `ConfigFlags` that cabal-install flattens the plan into.
  Analysis reads them from the plan.
- **Facts about the machine**: programs on the search path and their
  versions, `pkg-config` answers. They go into the `BuildEnvironment`,
  computed once per run (§3(a)).
- **Work that has to run**: a `Configure` package's script, and a Hooks
  package's configure hooks. These become actions. What they produce
  (the `.buildinfo` file, a hook's changes to a component) is an input
  to analysis, so the unit's analysis has to wait for them (§4).
- **Checks**, such as whether foreign headers and libraries exist, which
  Cabal answers by compiling a test program. Whether these become
  planning queries or actions that run before the first compile is open
  (§8).

The result of analysing a unit is its actions and its *provider*: what
its dependents may know about it (unit id, exposed modules, import,
library and include directories, `extra-libraries`, linker and C
options). The provider is computed from the plan and the environment
before any action runs; it is the content of the unit's registration
record, and the register action writes the record from it. For a
pre-installed unit, such as a boot library, the provider is read from
the package db the planner reads before solving, so a dependent cannot
tell the two apart. A unit's analysis therefore needs only its own plan
node, the environment, an inventory of its sources and its dependencies'
providers. That is what Buck2 calls analysis, and it is what lets units
be analysed independently. It also turns "when to reconfigure" into
"when to re-run analysis", which is always safe because analysis has no
effects; caching its result is an optimisation, not a correctness
question.

Two qualifications. In the prototype, analysis still builds a full
`LocalBuildInfo`, projected from the plan and the environment rather
than configured, because the `Cabal` functions it calls to compute
compiler and linker options consume one; nothing is persisted between
runs, and there is no `setup-config` file. And a Setup-mode unit
(`Custom`, or Hooks through Setup) keeps its configure: it is the first
of the unit's opaque Setup actions, and `setup-config` is its output.

The split matters for what a backend has to understand. A backend that
only wants the target graph (the "turn a plan into a package db"
interface michaelpj sketched on #9089) consumes it at *unit
granularity*: one opaque action per Setup phase per unit. A backend that
wants finer caching consumes it at *action granularity*. Both are views
of the same value.

### 2. The action graph as data

Following Ninja, files are the nodes and actions are the edges: in the
types below an `Edge` is one action, and a dependency between two actions
is a file one writes and the other reads. The types are the shape, not
the final API; the prototype implements them under these names, and its
Ninja and JSON renderers consume nothing else.

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
  , edgeReadsSourceTree :: Bool         -- see §4, coarsening
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
- **One writer per output.** The graph is the producer-of-path relation.
  Two actions writing one path is an error at construction, which Ninja
  and Buck2 also refuse. This rules out cabal's shared store package db
  as an output (see §6).
- **Commands are values.** A path in an argument is a tagged piece, so a
  renderer decides its spelling (relative to the run directory, absolute
  for a configure script or a Setup) and a backend knows the inputs and
  outputs a command names without parsing argv.
- **Paths are rooted.** `p` is instantiated to `Rooted Root`, a root
  (project, build directory, a unit's directory, a package's sources, the
  compiler's installation) and a relative path that does not climb. The
  graph carries where each root is on this machine; a backend relocates
  by rebinding roots, much as Cabal's `SymbolicPath` does. This covers
  paths in commands, not yet paths inside generated contents (§8).
- **Tools by role.** A command names `ghc`, `ghc-pkg`, `hsc2hs`, `ar`,
  `cc` as roles. The graph says what each resolved to here, for the
  native backend; a Nix backend substitutes derivations, a Buck2 backend
  its toolchain. This answers the "which GHC derivation?" question on
  #9089: the graph does not say, and the backend's configuration does.
- **No ambient state.** Nothing is read from the environment, the
  current directory or a cache that the graph does not name. The test is
  §5's Ninja backend: if `ninja` builds the export with cabal-install
  absent, every dependency is a file. The exception is a configure
  script, which reads the environment by design (§4).

Per unit, the vocabulary of actions is fixed and small. The prototype's
list, from a pass over every spawn and file write on cabal-install's
build path:

| per unit, before components | per component | per unit, after |
|---|---|---|
| fetch, unpack | autogen (`Paths_*`, `cabal_macros.h`) | install includes, install data files |
| configure (`Configure` packages: the script, writing `.buildinfo`) | preprocess (hsc2hs, c2hs, alex, happy, cpphs) | register (write the record, `ghc-pkg recache`) |
| setup compile, setup phase (`Custom`/Hooks in Setup mode: one opaque edge per phase) | compile (`ghc --make` per way or dynamic-too pair) | test (a phony target) |
| | compile extra source (`ghc -c` per C/C++/asm/cmm/js file and way) | |
| | link (archive, `ld -r`, shared, executable, foreign library) | |

What is *not* an action, and was IO on the build path: toolchain probes
(`ghc --info`, `ghc-pkg dump`, version checks, `pkg-config` queries),
which run once into a `BuildEnvironment`; *planning queries* (which file
is the main module, which sources need a preprocessor, whether an `.hsc`
has a `#def`, whether an `extra-lib-dirs` entry exists), answered before
the graph is built and recorded as fields; and the rebuilder's own
bookkeeping (file monitors, `plan.json`), which wraps actions and is
never inside one.

### 3. Where the graph comes from: refactoring cabal-install

The refactoring is a sequence of steps, each useful on its own.

**(a) Separate the plan from the machine.** `ElaboratedConfiguredPackage`
loses the fields that are facts about this machine (program paths,
build-tool paths, package db locations) and keeps program *arguments*.
A `BuildEnvironment` (configured programs, package db stack, scratch
directory, `pkg-config` answers) is computed once per run next to the
plan and kept out of the plan cache. After pruning, a `concretise` step
joins plan, environment and `DistDirLayout` into a concrete plan whose
units know their directories. The dry run, the build and `plan.json`
read the concrete plan; the plan cache and any future lock file see only
the pure plan. This removes one source of the plan-cache growth noted on
#11703 (a configured `ProgramDb`, `PATH` included, per unit), and goes
part of the way to #6885: a plan without an environment exists as a
value. Supplying the environment from a file instead of probing for it,
which is the rest of #6885 and what a Nix evaluation needs, is open
(§8).

**(b) The unit record.** `PackageBuildingPhase`'s `IO` fields become a
`UnitBuild` value: the source (local, or a tarball with its hash), the
Setup mode (in-library, or a Setup executable with its Cabal version and
closure), and per phase the flags and arguments as data. `SetupWrapper`
is split so the part that *decides* the Setup command line is pure and
the part that *runs* it is the native backend's. At this point every
unit has unit-level actions: a Setup-mode unit is complete; an in-library
unit's build phase is still one opaque action.

**(c) Component actions in-library.** For `Simple` and `Configure`
packages, the build phase (`InLibrary.build` since #11703) becomes a list
of component actions computed from the plan and an *inventory* of the
unpacked sources (the planning queries above). Much of what the Cabal
library computes on this path is already pure (`GhcOptions` and
`renderGhcOptions`, the preprocessor invocation builders, the record
built by `absoluteInstalledPackageInfo`); the prototype's analysis is
about 700 lines over them. The functions on the build path that do IO
the graph cannot contain move to planning queries or to the rebuilder:
`ghcInvocation` searching `PATH`, `getRPaths` canonicalising, filtering
`extra-lib-dirs` by existence, `checkNeedsRecompilation` comparing
modification times, and `generateRegistrationInfo` running
`ghc --abi-hash`.

Registration writes the record from the plan (`${pkgroot}`-relative)
and runs `ghc-pkg recache` on the unit's own db, rather than
round-tripping through `Setup register`. The record's ABI hash needs
care. GHC does read it: it keys its unit index on unit id and ABI hash,
and its home-unit closure check compares ABI hashes. Cabal already
writes a constant (`"inplace"`) for inplace registrations and an empty
`abi-depends` (#5465), so the prototype's constant hash has precedent,
but whether a constant is acceptable for store registrations, or the
hash must come from a post-compile `ghc --abi-hash` action, needs
confirming with GHC developers.

After this step, cabal-install's in-library build runs the enumerated
actions, interpreted by today's machinery; `InLibrary.build` stays
behind a flag for one release so the two can be compared. That makes
the actions the only description of the in-library build inside
cabal-install. `Distribution.Simple.Build` remains the description for
Setup mode, Stack and other `Cabal` library users, so the repository
then holds two descriptions of the build, which is the drift this
proposal criticises in external tools. Two mitigations: a CI job that
builds a set of packages both ways and compares the commands run; and a
later step that moves the enumeration into `Cabal` so that
`Distribution.Simple.Build` interprets it, which would also let a Setup
executable emit actions (`Setup build --emit-actions`). That later step
is out of scope here.

**(d) The native backend becomes a graph executor.** Not asked for here;
described because it is where the design leads. Initially, the native
backend interprets the graph with today's machinery: per-unit
scheduling, file monitors, the store. That is the migration path and
changes no behaviour. The target is one scheduler over the whole action
graph that records, per action, the hashes of its inputs and outputs.
That record is what allows early cutoff (skipping dependents when a
rebuilt output is unchanged), and it is the key a remote cache would
use (#9137). In the terms of *Build Systems à la Carte*, it is a
suspending scheduler with constructive traces. It would replace
`rebuildTargets`, the `PackageFileMonitor` and the two-level
`JobControl`, and it needs the store change of §6 first. The prototype
has such an engine (about 800 lines, generic over the action type).

### 4. Dynamism as data

Everything a build learns by running falls into three forms. Each is
part of the graph, each is bounded, and a backend that lacks one is
given a coarser graph instead. This section is the least settled part of
the design: the forms are clear, but how each existing source of
dynamism maps onto them, and onto each backend, is still being worked
out (§8).

The invariant that makes this work: **dynamism is confined inside a
unit.** It can add inputs to a unit's actions, or actions within the
unit, never a unit or a dependency between units. A unit's actions may
depend on each other and on finished outputs of units the target graph
already names. An import that no `build-depends` provides is an error at
the unit's module-graph discovery, not a new dependency.

**Level 1, discovered inputs.** After an action runs, it reports files it
read that it could not name beforehand: the headers a C compile included
(the C compiler's `-MD` depfile), the files Template Haskell declared
through `addDependentFile` (GHC's `-ddump-hi` usages). The action's
`edgeDiscovered` names where the report is and in what format. The
report only decides whether the action must rerun next time. A first
run executes it anyway, so not knowing these files in advance is never
a correctness problem. Ninja's `deps` and Shake's `needed` work this
way. Buck2's dep files are close, but they narrow a set of inputs that
was declared up front, so the graph would need to carry that superset
(the include directories, the unit's sources).

**Level 2, dyndep files.** An action runs after its producers and writes
a file that the scheduler reads before scheduling the actions that name
it (`edgeDyndep`). The rule: the file may add inputs and implicit outputs
to *existing* actions; it may not create an action. Two uses. First, the
module graph of a component (`ghc -M -include-cpp-deps`, or a plugin),
which gives each compile its home-module interfaces and CPP includes and
is what makes per-module compilation and the `--make` action exact.
Second, the scan for the C sources `c2hs` wrote, whose compile actions
exist from the start as candidates and are demanded only if the scan
lists their object. This is Ninja's `dyndep`. Buck2's `dynamic_output`
covers the same ground differently: it cannot amend an existing action,
so the dependent actions are created inside the dynamic step.

**Level 3, dynamic subgraphs.** An action produces a *graph fragment*:
new actions with new outputs. The fragment is a `Graph` value in the same
IR with two constraints: its outputs lie under a declared *envelope* (an
output directory of the enclosing unit, declared statically), and its
inputs are either inside the envelope or already inputs of the enclosing
unit. So the enclosing graph knows the fragment's footprint before it
exists. Nix's dynamic derivations (RFC 92: a derivation whose output is
a `.drv`) are exactly this. Whether Buck2's `dynamic_output` can bind an
output directory as an envelope and declare fresh actions inside it is
not yet verified. Ninja has nothing.

Where the existing sources of dynamism fall, as far as it is understood:

- **Hooks.** The set of pre-build rules is computed by running the hooks
  executable (`preBuildRules`), which makes it level 3. A rule's
  `dynamicRule` dependency computation only adds dependencies to a rule
  that already exists, which is level 2. A rule's command is a
  serialised closure run through the hooks executable, so to a backend
  each rule is an opaque action that runs that executable.
  `preBuildRules` also takes a `LocalBuildInfo`, which carries the
  machine facts §3(a) removes from the plan; how the two fit is open.
- **`build-type: Configure`.** The component flags come from the
  `.buildinfo` file the script writes, so the unit's component actions
  depend on a build output. In the prototype the script is an action
  that runs `sh configure` with the toolchain's C compiler and flags
  passed explicitly (`CC`, `CFLAGS`, `--host` when cross-compiling) and
  the flag assignment as `CABAL_FLAG_*` variables; its `.buildinfo` is
  merged into the main library and the executables, as Cabal's merge
  does. That is how `network` builds in the self-export. Three things
  are not right yet: the action runs once per unit rather than once per
  package; the script still reads the machine (system headers,
  `pkg-config`) beyond what the action declares; and the prototype also
  runs the script while making the graph, which is the staging question
  below, answered for now by running it twice.

- **Programs that must run before a unit's graph exists** (Hooks'
  pre-configure component hook adding modules; `test-code-generators`,
  whose output lists modules). They are level 3 if the backend has it,
  and otherwise force the unit to be analysed after that program's unit
  is built, which is a staging of analysis, not a hole in the graph.
  `Configure` falls into the same class.

**Backend capabilities and coarsening.** A backend declares what it runs:

```haskell
data Capabilities = Capabilities
  { capDiscoveredInputs :: Bool   -- level 1
  , capDiscoveryEdges   :: Bool   -- level 2
  , capDynamicSubgraphs :: Bool   -- level 3
  , capSandboxed        :: Bool   -- inputs are the whole input set; nothing else is visible
  }
```

The planner then lowers a graph to a backend's capabilities by
**coarsening**: a region of the graph that uses a form the backend lacks
is folded into one opaque action. Its inputs are the region's static
inputs plus, as the last resort, the unit's whole source tree
(`edgeReadsSourceTree`); its outputs are the region's declared outputs or
envelope. Coarsening is a graph-to-graph function in cabal-install, so
each backend is written against the forms it supports and nothing else.
`capSandboxed` tells the planner that a coarsened action must name its
source tree as files, because nothing else will be visible to it.

| | native | Ninja | Buck2 (provisional) | Nix |
|---|---|---|---|---|
| level 1 | traces | `depfile`/`deps` | dep files, over a declared superset | not needed: sandboxed, the unit's sources are the input |
| level 2 | scheduler reads the file | `dyndep` | `dynamic_output`, creating the dependent actions | coarsen to the unit |
| level 3 | scheduler extends the graph | coarsen to the unit | to be verified; otherwise coarsen | dynamic derivations, or coarsen |

Coarsening to "one derivation per unit" is what haskell.nix and the
cabal store do today, so a Nix backend's first version loses nothing
against the status quo and gains finer granularity where it wants it.

Two things this does **not** promise. First, a sandboxed backend cannot
see a file a Template Haskell splice reads through `runIO` without
declaring it; neither can anything else, and the native backend's
behaviour (rebuild only when a declared input changes) is the best
available. Second, staging of analysis (above) is a cost: a unit whose
graph depends on a built program cannot be analysed until that program
exists.

On the granularity question dcoutts raised on #12456: the IR does not fix
it. The first cut is one `ghc --make` per component with level 1
discovered inputs, which is what cabal does today. Per-module
compilation is a level 2 refinement of the same unit's graph (one
compile action per module and way, one module-graph discovery per
component) and needs nothing new from backends that have `dyndep`. The
improvements to `ghc -M` that make it correct are wanted either way; the
graph is where their output would go.

### 5. Backends

The layering is as follows. A package is the unit of source
distribution: its `.cabal` file says what it contains and what it
needs. A project determines the build configuration (packages,
constraints, flag assignments, compiler, optimisation, ways), and a
package built without a `cabal.project` is only a shorthand for a
project of one. The planner turns the project into the action graph. A
backend executes that graph and nothing more: like `jobs:`, which
backend runs the build is a detail of execution, and a project yields
the same graph whichever backend runs it. Nothing in a `.cabal` file
selects a backend or speaks to one.

This agrees with the Hooks proposal (`setup-hooks.md` in this
repository): one build system builds every package, and packages
customise it through Hooks rather than replace it. Hooks rules reach a
backend as the actions and fragments of §4. `Custom` packages, which
exist today and will for some time, reach it as opaque Setup actions
(see Backwards Compatibility).

Three terms. A *renderer* writes the graph in a format (JSON, Ninja). A
*backend* executes the graph: the native backend in-process, any other
by consuming a rendering. The *planner* produces the graph and, for a
given backend, coarsens it (§4).

**Native.** In-process, today's behaviour. The trace-based executor of
§3(d) would come later, with the store change of §6.

**Ninja.** `cabal plan --format ninja -o DIR` writes `build.ninja`,
`vars.ninja` (tool roles as variables) and the generated inputs; running
`ninja` on it is the Ninja backend. It is an export and an oracle: CI
builds cabal-install from its own export with cabal-install absent,
which is the proof that the graph is complete. It is also useful on its
own for anyone who wants a build they can read. What regenerates the
export when a `.cabal` file, a module list or an `extra-source-files`
glob changes is not designed yet; Ninja's `generator` rules are the
obvious mechanism (§8).

**Nix.** Not part of this proposal; sketched to show the graph is enough.
A backend, in or out of tree, would turn the graph into derivations and
submit them with `nix derivation add` (no Nix language involved, which
is what #9089 asks for). Fetch actions with a hash are fixed-output
derivations. Tools and `pkgconfig-depends` resolve through a
user-supplied map from role to store path or derivation; a missing entry
is an error that names the role, not a search. The unit boundary is a
package db directory, so a local package built natively can depend on a
store path Nix produced: the graph is partitioned at unit boundaries,
dependencies go to Nix, and workspace packages stay in-process and
incremental. That holds under conditions not yet worked out in full: the
Nix part must be closed downwards (no Nix-built unit depends on a native
one), both sides must use the same toolchain binding, cabal must
register GC roots, and starting a local unit as soon as its own
dependencies are realised (fgaz's interleaving concern on #9089) is a
scheduling question this partition makes possible but does not answer.
A Nix backend also needs the graph without running cabal-install's
probes at evaluation time (§8).

**Buck2.** Not part of this proposal either. `cabal-buck2` links the
cabal-install library and runs Cabal's configure, and so is tied to one
cabal-install version. A versioned JSON rendering across a process
boundary would remove that tie. But `cabal-buck2` emits *targets* that
Buck2's own Haskell rules analyse, and its users override them and pick
build modes at build time; a fixed action graph with ways already chosen
would take that away. So a Buck2 integration most likely wants a
per-component view (modules, generated files, flags, dependencies) and
uses actions only where it lacks Cabal's logic: `Custom` and Hooks units,
and foreign libraries. Whether that view is a separate rendering or the
graph at a coarser granularity is open (§8).

**External backends** read the JSON rendering of the graph
(`cabal plan --format json`) and are ordinary `cabal-<name>` external
commands, or tools that read the rendering themselves. In-tree backends
(native, Ninja) link the types. No Backpack, no plugin loader: the
interface is data and a process boundary.

### 6. The store

Not asked for here: a store layout change needs its own proposal. It is
described because the one-writer rule of §2 forces it once the native
backend executes the graph (§3(d)).

Every registration into `store/<ghc>/package.db` rewrites one
`package.cache`, so the shared db cannot be the output of a build action.
The prototype's answer is Nix's: **each unit's registration is its own
package db** (`store/<ghc>/<unit>/package.db`, one entry), and a
dependent's compile names the dbs of its closure through the response
file. The shared store db would survive as a derived view for tools that
want one (`ghc-pkg` users, `cabal path`, GHC environment files,
`cabal install --lib`), rebuilt from the per-unit dbs and never written
by a build action. This removes the in-process registration lock and
the contention on `package.cache`, so registrations can run in
parallel; the per-unit-id lock that guards the store against concurrent
cabal processes stays.

Open design points for that proposal: the view cannot be a directory of
symlinks to `${pkgroot}`-relative records, because `ghc-pkg` resolves
`${pkgroot}` against the directory the db is in, so the view needs
copies or regenerated records; the inplace db
(`dist-newstyle/packagedb/<ghc>`) has the same several-writers problem;
and the cost of passing GHC one `-package-db` per unit in the closure
needs measuring.

### 7. Command surface

- `cabal plan [--format json|ninja] [-o PATH] [TARGET...]`: the
  action-level query. Reads the plan, fetches and unpacks what the
  inventory needs, writes the graph, builds nothing. (#6885's
  `project-plan`, at the action level.) Experimental, with a schema
  version that promises nothing, until a schema proposal. The name is
  provisional: a built-in `plan` command would shadow the `cabal-plan`
  tool, which its users run as `cabal plan` (Backwards Compatibility).
- `plan.json` is unchanged. The graph is written only by `cabal plan`, so
  readers of `plan.json` (HLS, hie-bios, CI scripts) do not pay for it.
- `cabal build --backend=NAME`: run the graph through a backend. Not
  asked for here. `native` would be the default. A project-file field
  `backend:` is natural but not proposed: the project file says what the
  project is, and which tool executes its graph is, like `jobs:`, a
  detail of execution (§5). Execution can also be delegated without
  `cabal build` at all: a tool reads `cabal plan --format json` and runs
  the graph itself.

Commands other than `build` are not designed yet: `repl` and multi-repl
(and through them HLS and hie-bios), `haddock`, running tests and
benchmarks with their data-directory environment, and `install` of
executables. The expectation is that each consumes a prefix of the graph
plus a step of its own, but that has to be worked out command by
command (§8).

### 8. Design work still to do

The direction and steps 1–3 do not depend on any of these, but the
design is not complete without them:

1. **Supplying the environment.** The graph cannot yet be produced
   without probing the machine (`ghc --info`, `ghc-pkg dump`,
   `pkg-config`) and unpacking sources. A Nix evaluation needs a
   serialisable `BuildEnvironment` it can supply, e.g.
   `cabal plan --environment FILE`, so that probes become checks.
2. **Cross-compilation.** `Graph` has one platform and one compiler, and
   tool roles have no stage, while the target graph has a machine per
   unit. Platform and compiler should move to the unit, and roles be
   qualified by machine.
3. **Relocating generated contents.** `Write p ByteString` contents are
   not rooted: `Paths_*` modules embed absolute install directories, and
   RPATHs are absolute. `Write` probably needs `[Piece p]` contents.
4. **Mapping dynamism.** The classification of Hooks and `Configure` in
   §4 is a first reading and needs checking with the Hooks authors; the
   Buck2 column of the §4 table is provisional; and level 1 for Buck2
   needs a declared superset of inputs in the graph.
5. **A per-component view** for backends that do their own analysis
   (Buck2), and whether it is a rendering of its own.
6. **Fetches and sources.** A fetch without a hash
   (`source-repository-package`, local packages) cannot be a
   fixed-output derivation, and `edgeReadsSourceTree` without a file
   list makes any change in a package directory a rebuild.
7. **Commands other than `build`** (§7) and **regenerating the Ninja
   export** (§5).
8. **Configure-time checks** (§1). Cabal's configure verifies that
   foreign headers and libraries exist by compiling a test program.
   Without a configure step, the check becomes either a planning query,
   which runs a compiler while planning, or an action that runs before
   the unit's first compile; neither is chosen yet.

## Alternatives Considered

**Replace "plan → package db" only** (michaelpj, #9089). A backend that
consumes the graph at unit granularity is exactly this, and it is the
right first target for Nix. On its own it keeps Cabal's build logic
opaque, so step-level caching and per-module builds stay out of reach.

**Extend `plan.json` and let tools re-derive the rest** (the status quo).
The plan says what to build, not what running it means. Every consumer
either links cabal-install, as `cabal-buck2` does, or keeps its own
translation, as haskell.nix does, and follows Cabal's changes by hand.

**Link cabal-install as a library** (what `cabal-buck2` does). The
library exposes about 140 modules with no stability boundary. Keeping
its few reverse dependencies building already stops fixes from being
backported (#12326), and haskell/cabal-proposals#8 proposes a stable API
tier to contain the damage. Even with such a tier, a backend would get
cabal-install's functions, tied to its release cycle and to Haskell, not
a description of the build it can execute. A versioned data format
across a process boundary is the smaller contract. The two are
complementary: the graph's types are natural candidates for a stable
tier, but this proposal does not depend on one.

**`build-type: Buck2`.** simonmar's item 3 on #12456 asks for more than a
`.cabal` file can express (code generators such as Glean's, in packages
published to Hackage) and floats a Buck2 build-type for it, while
doubting it is workable. The need is real; a build-type is the wrong
place to meet it, for two reasons. It goes against the Hooks proposal,
which inverts Cabal's original design: instead of each package bringing
its own build system, one build system builds every package, with Hooks
covering the use cases of `Custom`. A new build-type that hands a
package to an outside build system would extend the per-package model
rather than contain it. And a build-type says how one package is built,
whereas which tool executes the build is decided when the project is run
(§5, §7), not by a package, which is the unit of source distribution.
simonmar is doubtful that Hooks are what he wants for this, and this
proposal does not change that: what a package can express is a separate
problem from which tool executes the build. Within this design it would
land in Hooks and level 3 fragments (§4); concrete needs such as Glean's
deserve their own tickets.

**A Shake-based executor without the IR.** Shake has dynamic
dependencies and early cutoff, but its rules are closures, so a Shake
build cannot be exported. The engine part of Shake is what the native
backend would reimplement, over a graph it already has as data.

**Backpack-pluggable backends.** An in-process plugin interface would
have to be stable across cabal-install releases and would still need the
data types; the data types with a process boundary are the smaller
contract.

**Per-module builds first, backends later** (dcoutts, #12456). Compatible,
and the order matters: per-module compilation written into today's
`IO`-closure builder would be a third thing to defunctionalise. With the
graph in place it is a change to one unit's analysis.

**`GhcOptions` records in actions.** Keeping Cabal's option records in
the graph instead of rendered arguments was tried in the prototype and
dropped: paths inside them become text, and a backend can no longer see
or relocate them.

**An applicative "wiring" layer over the IR**, to compose actions before
they are values. Also tried and dropped: it is a second language for
edges next to the graph itself.

## Backwards Compatibility / Migration

- No user-visible change by default. The native backend reproduces
  today's behaviour, and the test suite runs unchanged against it. The
  trace-based executor of §3(d), if it comes, will change rebuild
  decisions, and will be proposed separately.
- `build-type: Custom` and Hooks-through-Setup are supported on every
  backend at Setup-phase granularity, which is their granularity today.
  Nothing here depends on `Custom` going away: a `Custom` package is a
  unit whose build is a sequence of opaque Setup actions, and every
  backend can run that. This departs from dcoutts's expectation on
  #12456 that backends would follow the retirement of `Custom`; the
  departure is deliberate, because `Custom` packages will be with us for
  some time. Sandboxed backends will have to decide whether a `Custom`
  Setup action, which can read ambient state, is acceptable to them.
- Registration (§3(c)) writes the record from the plan and runs
  `ghc-pkg recache`, instead of `Setup register`. The record's ABI hash
  is the one behavioural change; see §3(c).
- The `Cabal` library changes in §3(c) move IO out of functions that
  Hadrian, Stack and custom Setups call. The existing functions stay;
  pure variants are added beside them.
- `plan.json` is unchanged. The graph's JSON rendering is new and
  experimental.
- A built-in `cabal plan` would shadow the `cabal-plan` tool, since
  external commands are dispatched only when no built-in exists. Either
  the command gets another name or `cabal-plan`'s users are given a
  migration path; to be settled before step 2 lands.
- Hooks: when cabal-install speaks the Hooks protocol in-library, rules
  are the actions and fragments of §4; a backend without level 3 gets
  the unit coarsened. No change to `Cabal-hooks`.

## Interested parties

- fgaz, michaelpj, angerman, Ericson2314, TravisWhitaker, hamishmack
  (#9089, Nix, haskell.nix)
- simonmar (#12456, `cabal-buck2`); rules_haskell maintainers
- dcoutts, mpickering (per-module builds, `ghc -M`)
- adamgundry, sheaf, alt-romes (Hooks: the classification in §4)
- fendor (HLS, hie-bios: the plan/environment split, `repl`)
- phadej (#6885); `cabal-plan` users (the command name)
- GHC developers, for `ghc -M` improvements, `-include-cpp-deps` and the
  ABI hash question; Hadrian maintainers, for the `Cabal` API
- mpilgrem (Stack, downstream of the `Cabal` library)
- Distribution packagers (nixpkgs Haskell maintainers, Debian, Fedora),
  who would get a readable build out of `cabal plan`; most of them build
  through `Setup.hs` rather than cabal-install projects

None contacted yet beyond the issue threads. The proposal should be
cross-posted to #9089, #12456, #3882 and #6885, and to Haskell
Discourse.

## Implementation Notes

I am willing to implement this and have most of the design running in a
prototype. The prototype contains a restructured copy of cabal-install's
`ProjectPlanning` and `ProjectBuilding` (52 of their 90-module import
closure, restructured as in §3(a)–(c)). It computes the action graph for
every unit of a plan (`Simple`, `Configure`, `Custom` and Hooks via Setup
mode; all four ways; foreign libraries; Backpack mix-in linking; hsc2hs,
alex, happy and c2hs untested), renders it as JSON and as Ninja, and
builds its fixture projects and itself from the Ninja export. The
self-export is about 1070 actions over 99 units, 85 of them tarballs
fetched by `curl` with the hash checked, `network`'s configure script and
a plan-built `hsc2hs` among them. A second `ninja` has nothing to do,
`ninja test` passes, and CI checks the fixpoint (the prototype binary
that the committed export builds writes that export back byte for byte).
Checked on Linux with GHC 9.12 only; not on Windows or macOS, and not
with `pkgconfig-depends`. What the prototype does *not* have is the
capability/coarsening layer of §4 (its one external backend is Ninja,
which has levels 1 and 2) and per-module compilation (designed, not
built).

<!-- TODO(andrea): the prototype repository is private as of 2026-10-27. Either publish it with the proposal or replace the references above with the relevant extracts. -->

Suggested order. Steps 1–3 are what this proposal asks for; each lands on
its own and changes no behaviour unless stated.

1. §3(a), plan/environment split. Pure refactoring with measurable
   benefit (the plan-cache size; a pure plan for #6885). Useful even if
   nothing after it lands.
2. §3(b), the unit record and an experimental
   `cabal plan --format json` at unit granularity. A Nix backend could
   start experimenting here.
3. §3(c), component actions for in-library units, run by the native
   build; the Ninja rendering; a CI job that builds cabal-install from
   its export. Windows and macOS have to pass before this step is done.

Later, each as its own proposal: the JSON schema and its versioning,
with the capabilities and coarsening of §4; the store layout (§6); the
trace-based native executor (§3(d)), which depends on the store change;
`--backend`. Out-of-tree backends (`cabal-buck2`, a Nix backend) would
follow.

<!-- TODO(andrea): commitment. Whose time pays for steps 1–3, how many PRs each is expected to be, and which reviewers for ProjectPlanning, ProjectBuilding and SetupWrapper have agreed to review. -->

Steps 1–3 are months, not weeks, and depend on review bandwidth more
than on code: the prototype's versions of them exist and the work is
re-expressing them as reviewable commits against `master`. If the work
stops after step 1 or 2, what has landed stays useful on its own, and
the experimental `cabal plan` is withdrawn if step 3 has not landed
within two releases.

## Open Questions

1. **Per-unit package dbs in the store** (§6). Deferred to the store
   proposal. It is irreversible once stores exist in the new layout. Is
   a store version bump acceptable, and what does the shared-db view
   have to be?
2. **Nix granularity.** One derivation per unit, or per action with
   content-addressed derivations. Content-addressed derivations are
   still experimental, and early cutoff would need deterministic GHC
   output; input-addressed per-action derivations give nothing over one
   per unit. One per unit is the likely first step.
3. **Programs that run at analysis time** (§4). Stage analysis so that a
   unit whose graph needs a built program is analysed after it, or make
   such programs level 3 fragments with an envelope. The Hooks protocol
   fixes when each hook is called, so this is one-way for Hooks and
   two-way for `test-code-generators`.
4. **Execution settings** for external backends: where the choice of
   backend and its tool bindings (a map from role to path or derivation)
   live when not on the command line. They are execution details, not
   project configuration (§7), so the candidates are the user's cabal
   config, the backend's own configuration, or a file kept apart from
   `cabal.project`.
5. **Does `Cabal` eventually emit actions** (`Setup build
   --emit-actions`), giving Setup-mode packages action-granularity
   graphs? Not needed for anything above, but it would retire the
   "Custom is opaque" limit without retiring Custom, and remove the
   second description of the build (§3(c)).
6. **Windows.** The prototype's Ninja rendering for `cmd` is written and
   unchecked. Everything in §2 is platform-neutral by construction, but
   nothing has run there.
7. **Schema ownership.** Deferred to the schema proposal: who versions
   the JSON graph and how a breaking change is announced to external
   backends; this is the same question PR cabal-proposals#8 asks for the
   library API.

## References

- haskell/cabal#9089, RFC: Nix backend
- haskell/cabal#12456, RFC: Buck2 backend
- haskell/cabal#3882, Interface for system package manager to execute
  build plan
- haskell/cabal#6885, Add `project-plan` command
- haskell/cabal#9137, Remote cache in the style of Bazel (closed)
- haskell/cabal#11703, Directly call in-library functions to build
  packages
- haskell/cabal#12326, Usefulness of releasing `cabal-install` the
  library
- haskell/cabal-proposals#8, Designing the API for cabal-install
- *Replacing the Cabal Custom build-type* (the Hooks proposal),
  `proposals/setup-hooks.md` in this repository; haskell/cabal#9292
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
