# AGENTS.md

## Project identity

`owoow` is a Windows .NET 10 RNG tool and sys-botbase client for Pokémon Sword and Shield.

This fork is maintained as a long-lived private development repository based on the upstream project `LegoFigure11/owoow`.

## Branch model

- `master` is the upstream mirror branch. Keep custom work out of it whenever possible.
- `develop` is the long-lived integration branch for this fork.
- Use `feature/*` for product features.
- Use `fix/*` for bug fixes.
- Use `research/*` for reverse-engineering, RNG investigation, memory research, hooks, prototypes, and experiments.
- Use `chore/*` for CI, dependency, repository, and tooling work.
- Prefer pull requests into `develop` for non-trivial changes.

Do not merge custom development back into `master` unless the task explicitly requires changing the mirror strategy.

## Architecture

The repository has two primary projects:

- `owoow.Core`: RNG, encounter, connection, structures, resources, and non-UI logic.
- `owoow.WinForms`: Windows Forms UI and application coordination.

Maintain the dependency direction from WinForms to Core. Do not introduce a dependency from `owoow.Core` to `owoow.WinForms`.

Place reusable algorithms and domain logic in `owoow.Core`; keep presentation and UI coordination in `owoow.WinForms`.

## Build environment

The current target is Windows x64 on .NET 10.

Baseline validation for code changes:

```powershell
dotnet restore
dotnet build owoow.slnx -c Release
```

When a change affects publishing or deployment, also run the relevant publish command or GitHub Actions workflow.

## Change discipline

- Make the smallest change that correctly solves the requested task.
- Avoid unrelated cleanup, renaming, formatting, or architecture changes.
- Preserve upstream behavior unless the task intentionally changes it.
- Do not silently upgrade package references, bundled DLLs, SDK versions, or GitHub Actions merely because newer versions exist.
- Do not rewrite generated WinForms code or resource files unless the UI task requires it.
- Follow `.editorconfig`; do not duplicate or override its formatting rules here.

## Reverse-engineering and RNG research

Research findings must distinguish evidence from inference.

Use these terms when documenting uncertain technical conclusions:

- `VERIFIED`: reproduced or directly observed.
- `INFERRED`: strongly supported by evidence but not directly confirmed.
- `HYPOTHESIS`: plausible and still requires validation.

For memory addresses, offsets, hooks, constants, signatures, or game-specific behavior, document the applicable game, version/build, source, and validation method whenever known.

Do not convert a hypothesis into production logic without noting the supporting evidence.

### Local reverse-engineering environment

For tasks involving binary analysis, IDA, IDAPython/IDALib, hooks, symbol or structure recovery, binary diffing, Android/Frida work, CPU emulation, or other local reverse-engineering tooling, read `docs/reverse-engineering-environment.md` before execution.

That document is the authoritative machine-specific record for the current Windows + WSL reverse-engineering environment.

Key requirements:

- Default to IDA Professional 9.4 for IDA work unless the task explicitly requires another version.
- Do not silently fall back to IDA 9.2.
- Use the documented absolute paths; do not invent `ida64.exe` or other executable names.
- Prefer the existing IDA 9.4 probe/launcher scripts documented there instead of creating duplicate launcher logic.
- Select the minimum necessary installed reverse-engineering skill and read its `SKILL.md` before using it.
- Skill presence does not prove tool availability.
- Verify required executables/runtimes and versions before execution.
- Do not assume a Windows-installed tool is available in WSL `PATH`.
- Before actual binary analysis, verify the input absolute path, SHA-256, file format, target architecture, target version/build when known, and static/dynamic analysis mode.
- If a skill is installed but its required execution tool is missing, explicitly report: `skill 已有，执行依赖缺失`.
- Never claim a local tool or skill was executed unless it was actually invoked and its output was observed.

## Testing expectations

Prefer deterministic tests for RNG and encounter logic. When modifying an algorithm, preserve or add known input/output vectors where practical.

A useful RNG test records:

- input state or seed,
- expected output,
- RNG call/advance count when relevant,
- game/version/mechanic context,
- source or reference for the expected result.

Do not change RNG call order only to make a desired output appear correct.

## Documentation

Keep durable project-management rules in `docs/project-management.md`.

Keep the local reverse-engineering tool/skill inventory and execution preflight rules in `docs/reverse-engineering-environment.md`.

Keep development, fix, configuration, Git, environment, and repository-operation history under `docs/development-log/YYYY/MM/YYYY-MM-DD.md`, following `docs/development-log/README.md`.

Research notes should live under `docs/research/` as that area is created and expanded.

Architecture notes should live under `docs/architecture/` as that area is created and expanded.

## Commit and PR scope

Keep commits focused. Separate research/prototype work from production implementation where practical.

A PR should explain:

- what changed,
- why it changed,
- how it was validated,
- any game-version assumptions,
- unresolved risks or follow-up research.
