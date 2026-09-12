# owoow.Core AGENTS.md

This file extends the repository-level `AGENTS.md` for work under `owoow.Core/`.

## Scope

`owoow.Core` owns reusable non-UI behavior, including RNG algorithms, encounter generation, connection logic, structures, resources, enums, and interfaces.

Do not introduce WinForms dependencies here.

## RNG correctness rules

RNG changes require evidence. Treat RNG call order, bounds, state transitions, jump counts, and seed derivation as observable game behavior rather than implementation details that may be freely rearranged.

When changing RNG code, document or preserve where practical:

- input RNG state or seed,
- expected output,
- exact or expected number of RNG calls/advances,
- order of draws,
- game/version/build,
- encounter or mechanic type,
- source/reference or reproduction procedure.

Do not change RNG call order simply to make a target result pass.

Do not collapse multiple RNG stages into one abstraction when doing so would hide game-observable state transitions that matter to prediction, reidentification, hook placement, or reverse engineering.

## Main / Outer / fixedSeed / Inner research

When work concerns Sword/Shield generation stages such as Main RNG, an outer RNG stream, `fixedSeed`, encounter generation, or later Pokémon attribute generation, keep each stage conceptually separate until its relationship has been verified.

Do not assume that similarly named RNG states are interchangeable.

If a relationship is not fully established, mark conclusions as `INFERRED` or `HYPOTHESIS` rather than presenting them as fact.

## Reverse-engineering data

For addresses, offsets, signatures, hooks, structures, and constants:

- identify the game and version/build when known,
- distinguish static file offsets from runtime virtual addresses,
- state whether the value came from disassembly, debugger observation, memory inspection, external research, or inference,
- avoid replacing a well-named constant with an unexplained magic number,
- avoid spreading version-specific offsets throughout unrelated code.

Prefer centralized, named, version-aware constants where production code must contain such values.

## Encounter logic

Preserve the separation between:

- encounter selection,
- encounter-table/resource lookup,
- RNG state progression,
- Pokémon attribute generation,
- filtering/search logic,
- console/memory connection concerns.

Do not embed UI assumptions into encounter or RNG classes.

## Connection code

Treat console-memory reads/writes and SysBot-related behavior as external I/O.

- Validate addresses and expected byte widths.
- Preserve cancellation and error handling behavior.
- Avoid adding writes where a read-only operation is sufficient.
- Do not alter bundled `SysBot.Base` or other binary dependencies unless explicitly requested.

## Resources and data files

Encounter tables and personal/resource data are part of algorithm correctness.

When modifying resource data, record the source and the affected game/version/area where practical. Avoid mass reformatting resource files in an unrelated change.

## Validation

For pure RNG or encounter changes, prefer deterministic tests or a small reproducible harness before relying on UI/manual testing.

At minimum, ensure the solution builds in Release configuration after code changes.
