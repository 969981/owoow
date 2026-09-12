# Project Management

This document defines the maintenance and development model for the `969981/owoow` fork.

## Repository purpose

This repository is maintained as a long-lived private development fork of `LegoFigure11/owoow`.

The main goals are:

1. remain able to track upstream owoow changes,
2. develop custom Sword/Shield RNG features without contaminating the upstream mirror branch,
3. support reverse-engineering and hook research in an auditable way,
4. make AI-assisted development predictable through scoped `AGENTS.md` files,
5. keep experimental research separate from production-quality implementation.

## Branch roles

### `master`

`master` is the upstream mirror branch.

Expected behavior:

- synchronize from `LegoFigure11/owoow:master`,
- avoid fork-specific feature commits,
- keep history as close to upstream as practical,
- use fast-forward synchronization when possible.

Typical synchronization flow:

```bash
git fetch upstream

git switch master
git merge --ff-only upstream/master
git push private master
```

### `develop`

`develop` is the long-lived integration branch for this fork.

Fork-specific features, fixes, documentation, research-derived implementation, CI changes, and tooling work are integrated here.

After updating `master`, bring upstream changes into `develop` explicitly and resolve any fork-specific conflicts there:

```bash
git switch develop
git merge master
git push private develop
```

### Short-lived branches

Use these prefixes:

- `feature/*` — user-visible or production functionality,
- `fix/*` — defects and regressions,
- `research/*` — RNG research, reverse engineering, memory investigation, hook experiments, prototypes,
- `chore/*` — CI, dependency, build, tooling, and repository maintenance,
- `docs/*` — documentation-only work when a separate branch is useful.

Examples:

```text
research/swsh-fixed-seed-hook
feature/fixed-seed-override
fix/seed-reident-off-by-one
chore/develop-ci
```

## Development flow

Preferred flow for non-trivial work:

```text
upstream/master
      |
      v
    master
      |
      v
    develop
      |
      +--> feature/*
      +--> fix/*
      +--> research/*
      +--> chore/*
             |
             v
         PR -> develop
```

Research work should not be promoted directly into production code merely because a prototype appears to work.

A useful progression is:

```text
research/*
   |
   | evidence + reproduction + validation
   v
feature/* or fix/*
   |
   v
PR -> develop
```

## Research status vocabulary

Use consistent evidence labels in research documentation and technical PR descriptions:

- `VERIFIED` — directly observed or independently reproduced.
- `INFERRED` — strongly supported but not directly confirmed.
- `HYPOTHESIS` — plausible and pending validation.

For reverse-engineered facts, record when practical:

- game and version/build,
- module/file/runtime address context,
- static offset vs runtime virtual address,
- calling context,
- RNG state before/after,
- reproduction procedure,
- debugger/disassembler/tool used,
- relevant external reference.

## Project architecture

The main projects are:

```text
owoow.Core
    ^
    |
owoow.WinForms
```

`owoow.Core` owns reusable domain behavior such as RNG, encounters, structures, resources, and connection logic.

`owoow.WinForms` owns forms, display state, user interaction, and application coordination.

Do not reverse this dependency direction.

## AI / Codex instruction model

AI development rules use hierarchical `AGENTS.md` files:

```text
AGENTS.md
owoow.Core/AGENTS.md
owoow.WinForms/AGENTS.md
```

The repository-level file defines stable global rules. Deeper files add rules specific to their directory subtree.

Do not duplicate `.editorconfig` formatting rules in `AGENTS.md`.

Add another scoped `AGENTS.md` only when a directory develops genuinely different constraints that cannot be expressed cleanly in the existing files.

## Documentation layout

Current and target documentation layout:

```text
docs/
├── project-management.md
├── development-log/
│   ├── README.md
│   └── YYYY/
│       └── MM/
│           └── YYYY-MM-DD.md
├── architecture/
│   ├── overview.md
│   └── rng-pipeline.md
└── research/
    ├── main-rng.md
    ├── outer-rng.md
    ├── fixed-seed.md
    ├── overworld-generation.md
    └── memory-layout.md
```

`docs/development-log/` records development work, fixes, configuration changes, Git/environment operations, and daily results. Follow `docs/development-log/README.md` and keep same-day items in the same dated file.

Directories and files under `architecture/` and `research/` should be created as material becomes sufficiently durable; empty placeholder documents are not required.

The development log does not replace issues, PRs, or technical research documents.

## Issue and PR conventions

Recommended labels:

- `feature`
- `bug`
- `research`
- `upstream-sync`
- `refactor`
- `build`
- `docs`

A non-trivial PR should state:

1. what changed,
2. why it changed,
3. how it was validated,
4. affected game/version assumptions,
5. RNG/advance implications when relevant,
6. unresolved risks or follow-up research.

Prefer focused PRs. Avoid combining unrelated UI redesign, dependency upgrades, RNG changes, and reverse-engineering experiments in one review unit.

## Build and CI baseline

Current baseline:

- .NET 10,
- Windows target,
- x64,
- Release build for validation.

Local baseline:

```powershell
dotnet restore
dotnet build owoow.slnx -c Release
```

GitHub Actions should validate pushes and pull requests for both `master` and `develop`.

`master` CI protects upstream synchronization. `develop` CI protects the fork integration line.

## Testing roadmap

The repository currently benefits most from tests around deterministic Core behavior.

When a test project is introduced, prioritize:

1. RNG forward/jump/reverse behavior,
2. seed reidentification,
3. known seed-to-result vectors,
4. encounter generation,
5. fixed-seed-related generation stages,
6. shiny/PID/IV/nature/ability calculations,
7. known game-mechanic regression cases.

Tests should avoid UI dependencies whenever possible.

## Dependency policy

Do not upgrade NuGet packages, checked-in DLLs, SDK versions, or GitHub Actions as incidental cleanup.

Dependency changes should be intentional and should record:

- reason,
- compatibility impact,
- build/runtime validation,
- any upstream divergence introduced.

## Upstream synchronization policy

Before a significant custom feature, check whether upstream changed the same subsystem.

When upstream changes overlap custom code:

1. synchronize `master`,
2. merge `master` into `develop`,
3. resolve conflicts on `develop`,
4. re-run build/tests,
5. update research assumptions if upstream behavior changed.

Never resolve an RNG-related upstream conflict by choosing whichever version merely produces the preferred target result; verify call order and mechanic behavior.

## Release strategy

Until a separate fork release policy is needed, treat releases as explicit, reviewed snapshots derived from validated fork code rather than automatic consequences of every `develop` commit.

Release automation can be added later without changing the `master`/`develop` responsibilities defined here.
