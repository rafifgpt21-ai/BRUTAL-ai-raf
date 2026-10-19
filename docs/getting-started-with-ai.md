# Start a game project with BRUTAL and AI agents

Use this guide to turn your game concept into a project that you and AI agents can develop through production. It applies to solo projects and larger teams, 2D and 3D games, and games with demanding simulation or networking requirements. The decisions, repository structure, and development milestones should follow your game's needs.

The starting deliverables are a project brief, a chosen technical stack, a buildable game repository, instructions agents can follow, and a roadmap whose first milestone proves the most important assumptions. You remain responsible for the game's direction, architecture, and acceptance of the work.

## 1. Describe the game you intend to ship

Create a short `docs/project-brief.md` in the new game repository. Give the agent enough information to reason about the actual project:

```text
Working title and genre:
Player experience: what the player does, decides, and cares about
Core loop: the sequence of actions that sustains play
Design pillars: the qualities that should guide tradeoffs
Target platforms, input devices, and distribution format:
Presentation: 2D/3D, camera, visual style, UI needs
World and simulation: scale, entity counts, physics, AI, persistence
Multiplayer: player count, session model, authority, latency needs, or none
Content: levels, assets, procedural generation, modding, authoring workflow
Performance targets: hardware, frame/update budgets, memory and load times
Team: roles, experience, review capacity, tools, available time
Release scope: what belongs in the first release and what can wait
Open questions: decisions still requiring investigation
```

Use ranges or mark unknowns where you lack evidence. A space simulation's coordinate scale, a strategy game's simulation load, and a competitive game's networking model can affect the architecture from the beginning. Have the agent identify those dependencies before generating a project structure.

Separate the eventual release scope from the next milestone. A large game can begin with a representative slice of its systems while keeping the larger requirements visible.

**Human action:** review the brief until it describes the game you want, including its constraints. Ask the agent to challenge contradictions and identify missing decisions.

You can start that conversation with:

```text
I am starting [describe your game] using BRUTAL.
The framework checkout is at [path]; the game repository is at [path].
Read the framework overview and agent guides before proposing a stack.
Help me turn my concept into docs/project-brief.md. Ask about unknowns that
affect the game's scope, architecture, content workflow, or target platforms.
Keep unresolved answers visible, and distinguish my requirements from your suggestions.
Then propose the technical decisions and experiments needed to start this project.
```

## 2. Establish what BRUTAL supplies and what your project owns

BRUTAL provides native APIs and supporting libraries. Your project owns the application lifecycle, game architecture, gameplay, and the tools needed to produce content. The [framework README](../README.md), [module index](../AGENTS.md), and [binding architecture guide](agents/architecture.md) describe the available foundation.

Choose a starting stack against the brief. Keep a short decision table in `docs/architecture.md` with the choice, reason, source evidence, and unresolved questions.

| Decision | BRUTAL options to investigate | What your game must decide or implement |
| --- | --- | --- |
| Windows, input, platform integration | `Brutal.Glfw` or `Brutal.Sdl` | Window lifecycle, input mapping, focus/controller handling, platform behavior |
| Rendering | `Brutal.Vulkan` with `.Abstractions`/`.Vma`, `Brutal.OpenGl`, or SDL's rendering API | Rendering features, presentation, camera, material/shader conventions, resource lifetime and tooling |
| Asset loading | `Brutal.Texture*`, `Brutal.Gltf`, mesh and shader libraries | Source-to-runtime asset workflow, identifiers, dependencies, caching, streaming and content validation |
| Physics | `Brutal.Box2D` for 2D or `Brutal.Jolt` for 3D | Collision rules, simulation timing, gameplay integration and callback ownership |
| Audio | `Brutal.Fmod`, `Brutal.OpenAl`, or SDL Mixer | Mixing, playback rules, asset formats and audio lifecycle |
| Diagnostics and tools | `Brutal.Logging`, `Brutal.ImGui*`, `Brutal.ImPlot`, RenderDoc/monitor modules | Debug views, editor or authoring needs, metrics, crash reproduction and reporting |
| Parallel work and networking | `Brutal.Concurrency`, `Brutal.RakNet` | Scheduling dependencies, synchronization, network authority, replication and recovery |
| Data and math | Core, Memory, Collections, Numerics, Strings, Localization | Data ownership, coordinate conventions, serialization, text/localization policies |

Read [graphics and assets](agents/graphics-assets.md), [physics/audio/jobs](agents/physics-audio-jobs.md), and [foundations](agents/foundation.md) for concrete APIs and lifetime rules. Folder names are not always C# namespaces. Verify signatures against source and the package version your project actually uses.

BRUTAL's graphics helpers support renderer construction; they do not supply a complete game renderer or scene architecture. Asset loaders expose data, while your project supplies the production pipeline around it. RakNet bindings do not determine your multiplayer design. Put these responsibilities in the roadmap explicitly.

Choose an ECS, scene hierarchy, custom simulation, or separate engine layer because it serves the game. Start with the boundaries needed for the known workload and workflow; extract more projects or subsystems when there is a concrete reason.

**Human action:** agree on the starting stack and identify which choices need an experiment before commitment. Record consequential decisions and their reasons in `docs/adr/` as they are made.

## 3. Give the game its own repository and reproducible build

A practical workspace keeps the framework reference and the game separate:

```text
workspace/
  BRUTAL/                  framework checkout and these guides
  MyGame/                  your game repository
    AGENTS.md
    README.md
    docs/
      project-brief.md
      architecture.md
      roadmap.md
    src/
      MyGame.Client/
    assets/
    tests/
    tools/
```

This is a starting layout, not a requirement to create every folder immediately. A solo project may keep more code in one executable. Add a shared runtime, dedicated server, editor, importer, or separate test projects when your architecture calls for them. Your game owns its tracker, release process, and instructions; the framework fork's tracker settings do not configure the game repository.

The [README prerequisites](../README.md#prerequisites) specify .NET 10 and Windows/Linux. Check target-platform support and native dependencies for the modules you select. A platform mentioned in an individual binding is not evidence that the entire stack is ready to ship there.

For a solution-based starting layout, run these commands from the workspace folder:

```powershell
mkdir MyGame
cd MyGame
git init
dotnet new gitignore
dotnet new sln --name MyGame
dotnet new console --name MyGame.Client --framework net10.0 --output src/MyGame.Client
dotnet sln add src/MyGame.Client/MyGame.Client.csproj
dotnet add src/MyGame.Client/MyGame.Client.csproj package Ahwoo.Brutal.Core
dotnet build
```

Add the `Ahwoo.Brutal.*` packages selected in step 2 to the projects that use them. Inspect and commit the resolved version numbers in the project or central package file. Pin the chosen SDK in `global.json`; use NuGet lock files and locked restore where the build workflow requires them. Record the framework source revision used for API reference and the target runtime/architecture. Establish a deliberate update process for packages, native binaries, and shaders.

The framework snapshot currently has no project/solution for building its source at the root. Use published packages for the game unless you have deliberately set up a supported source build. Keep framework modifications isolated so package upgrades remain understandable.

Configure `AllowUnsafeBlocks` on projects that need pointer APIs. Define where native code may be called and who owns each resource. Keep borrowed handles, copied handle structs, and owning wrappers distinct. The [foundation guide](agents/foundation.md) covers the relevant memory and type rules.

Write the real commands in the game README. For the layout above, the initial commands are:

```powershell
dotnet build
dotnet run --project src/MyGame.Client/MyGame.Client.csproj
```

The console template proves the project setup only. Your first framework integration task still needs to initialize the selected libraries and run their actual window/input/render or headless lifecycle. Document test, asset-build, server, editor, and packaging commands as those targets are introduced.

**Completion evidence:** another checkout can restore and build using documented versions. The bootstrap milestone must additionally prove native library loading, initialization, useful error reporting, and orderly shutdown on the target platform.

## 4. Give agents persistent project context

Open the game repository as the working directory. Give the agent access to the framework checkout for reference, and state its path explicitly. Keep the same information available to every agent or session:

| Context | Authoritative location in the game repository |
| --- | --- |
| What the game is and its constraints | `docs/project-brief.md` |
| System boundaries, stack choices, data flow | `docs/architecture.md` and relevant ADRs |
| How to build, run, test, build assets, and package | Game README and checked-in scripts/configuration |
| How agents should work | Root `AGENTS.md`, with focused instructions for specialized areas as needed |
| What to do next and what blocks it | Your chosen issue tracker or a documented local task system |
| Current milestone, known failures, reproduction steps | A brief progress note or milestone issue |

Create a game-specific `AGENTS.md`. This template is intended to be edited after the stack and workflows are chosen:

```markdown
# MyGame

Read docs/project-brief.md before making gameplay or scope decisions.
Read docs/architecture.md and relevant ADRs before changing system boundaries.

Framework reference: ../BRUTAL (replace with the actual checkout path).
Before using a BRUTAL module, read its AGENTS.md, the relevant docs/agents
page, and the source for the API. Match it to the installed package version.

Read README.md for current build/run/test/content-build commands.
Use the project's tracker and its recorded dependencies to select work.
Treat the framework checkout as library reference; its tracker settings do not
override this game's workflow.

Implement the assigned acceptance criteria within the agreed scope.
Keep native resource ownership, callback lifetimes, and threading explicit.
Preserve established gameplay behavior and data compatibility.
Record unresolved design decisions instead of silently choosing game rules.
Validate the change with relevant automated checks and a manual test plan.
Report changed files, checks actually run, results, and remaining limitations.
```

A short root file should route agents to the relevant material. Keep renderer-specific constraints near rendering code and content rules near the asset tools as the project grows. Update instructions when the workflow changes. If an assistant does not load instruction files automatically, explicitly ask it to read them.

The human supplies design intent and resolves tradeoffs affecting scope, player experience, and long-lived technical commitments. Agents can investigate options, write code and tests, build tools, and maintain the supporting documentation within assigned work.

## 5. Investigate the risks that shape this game

Before scheduling a large implementation, ask the agent to map the brief to the codebase and list the assumptions that could change the architecture. Turn the important uncertainties into short experiments with a question and an exit criterion.

| If the game depends on | A useful early experiment |
| --- | --- |
| A large simulation | Exercise representative entity counts and update patterns; measure update time, allocations and memory on target hardware. |
| Large worlds or high precision | Prove coordinate conventions, precision, origin handling, and interaction with rendering and physics. |
| Online play | Run the intended client/server authority path with latency and disconnects; investigate replication, prediction, persistence and version compatibility as needed. |
| A particular visual result | Render representative assets with the required materials, lighting and camera on the intended hardware. |
| Heavy content production | Import a real asset through the authoring-to-runtime path and prove iteration time, validation and dependency handling. |
| Saving, procedural worlds or mods | Exercise the intended data/schema lifecycle, reload behavior and compatibility boundaries. |

These experiments inform scope and architecture. Keep successful findings, measurements, and source references; decide explicitly which experimental code belongs in the production foundation.

A discovery prompt you can adapt:

```text
Read the project brief, current architecture notes, game code, and BRUTAL reference.

Map each major game requirement to:
- an existing BRUTAL API, with a source reference and package-version check;
- a system our game must implement;
- or an unresolved dependency/assumption.

Identify architecture-changing risks and propose the smallest experiment that
can answer each important question. Include success criteria and measurements.
Propose a starting stack and project boundaries with reasons and alternatives.
Record unknowns. Produce findings and a proposed foundation milestone first;
implementation tasks will be selected from that evidence.
```

## 6. Plan from a representative slice to production

Build the roadmap around working outcomes. A representative slice is a narrow path through the game's real systems: the intended renderer, actual asset path, one meaningful player interaction, and any simulation/network/persistence constraint central to that interaction.

For a tactical game, that might be selecting a unit, issuing an order, simulating movement and combat, and saving the resulting state. For a networked action game, it might be joining a server, moving, interacting, and observing authoritative results on a second client. Choose the slice for your game.

| Stage | Outcome to demonstrate |
| --- | --- |
| Foundation | Reproducible build, chosen library initialization, lifecycle, logging, and a target-platform launch/package check |
| Technical proof | The important performance, platform, rendering, content, or networking assumptions have evidence |
| First representative slice | One end-to-end gameplay path exercises the agreed core systems and can be played and inspected |
| Production foundation | Content iteration, diagnostics, save/configuration rules, relevant automated checks and repeatable builds support ongoing work |
| Feature/content milestones | Measurable increments move the game toward its release scope, with regular playtests |
| Release preparation | Packaged builds, platform coverage, performance, compatibility, recovery and applicable notices are verified |

The exact ordering can overlap. A multiplayer architecture may require server work before the first playable slice; a content-heavy game may need authoring tools very early. Resolve those dependencies in the task graph.

Each implementation ticket should contain:

```text
Goal: the player, developer, or content-creator outcome
Context: brief/architecture/source pointers and current behavior
Scope: systems/files involved and explicit boundaries
Dependencies: prerequisites and contracts needed from other tasks
Acceptance criteria: observable behavior, relevant budgets and failure handling
Validation: automated checks, manual scenario, and target environment
Deliverables: implementation, tests, documentation or content changes required
```

A large project needs many connected, reviewable tasks. Keep the roadmap's overall ambition and make each assignment fit the team's ability to review and validate it.

## 7. Run a repeatable human-and-agent development cycle

For each ready task, have the agent read the context and current code, identify the implementation approach, make the change, and run the relevant checks. Then review the result against the acceptance criteria and play or inspect the affected behavior.

An implementation prompt:

```text
Implement [ticket ID/path] in this game repository.
Read AGENTS.md, the ticket's linked documents, and the current implementation.
Verify BRUTAL APIs against the specified source revision and installed packages.

Respect the agreed system boundaries and native resource/threading contracts.
Identify unresolved decisions that materially affect this task; state assumptions
and continue independent, reversible work where possible.
Implement the acceptance criteria and appropriate tests or diagnostic tooling.
Run the relevant documented checks and provide a manual validation scenario.

Report the behavior changed, files changed, actual check results,
remaining gaps, and any architecture/documentation changes needed.
```

Review game behavior and engineering separately. A technically clean feature can still have the wrong controls, pacing, or rules. A feature that looks right can still leak resources, corrupt saves, or fail on another platform. Automated checks should cover stable rules and contracts; playtests should answer the questions about player experience.

When a failure occurs, provide expected behavior, observed behavior, reproduction steps, environment/package versions, and relevant logs or captures. Ask the agent to explain the cause and validate a focused correction. Keep working commits and task references so regressions can be traced.

For parallel agents, give each a bounded task, its own branch/worktree, and explicit ownership of files or subsystem contracts. Agree on shared interfaces before dependent work starts. Integrate through a common branch and rerun cross-system checks; parallel code generation does not replace integration work. A solo developer can use the same process sequentially.

Before ending a session, record what works, what remains, the current failure if any, and the next ready task in the project's tracker/progress note. The next agent should be able to resume from files and evidence.

## 8. Scale the project when the work requires it

| As needs grow | Add the structure that addresses the need |
| --- | --- |
| A solo project accumulates repeatable workflows | Check in build/content scripts and keep a concise task list with acceptance criteria. |
| Multiple people or agents change the same systems | Define ownership and interface contracts, review integration changes, and use CI for the shared validation steps. |
| Client and server share rules/data | Establish shared simulation/schema boundaries and compatibility checks; keep presentation-specific dependencies out of headless paths. |
| Content creation becomes a bottleneck | Build or integrate import, validation, preview and authoring tools for the actual asset workflow. |
| Performance becomes a constraint | Use representative captures and benchmarks to guide data layout, allocations, threading and rendering changes. |
| Builds must reach players reliably | Automate asset builds and packaging, test target platforms, and version saves/content/network contracts as required. |

Keep the current commands and failure diagnostics available in scripts and the README. Establish CI as soon as work needs a shared, repeatable integration check, and retain manual runs on actual target hardware for graphics, input, audio, and performance behavior.

Before distributing the game, review [LICENSE.txt](../LICENSE.txt), [NOTICE.txt](../NOTICE.txt), and the notices for the native components you ship. Test packaged output as a user would launch it, including asset discovery, configuration, save locations and failure reporting. Follow [CONTRIBUTING.md](../CONTRIBUTING.md) if contributing framework changes upstream.

## Your project kickoff checklist

- [ ] Write and review the actual game's brief and release scope.
- [ ] Choose the initial BRUTAL modules and record the systems the game must own.
- [ ] Set up a separate game repository with documented SDK/package/native versions.
- [ ] Give agents a game-specific instruction file and access to the correct framework reference.
- [ ] Investigate the assumptions most likely to change scope or architecture.
- [ ] Define the first representative slice and its success criteria.
- [ ] Create dependency-linked tasks and select the first ready foundation task.
- [ ] Establish how results will be built, checked, played, reviewed and integrated.
