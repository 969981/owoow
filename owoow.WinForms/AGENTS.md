# owoow.WinForms AGENTS.md

This file extends the repository-level `AGENTS.md` for work under `owoow.WinForms/`.

## Scope

`owoow.WinForms` owns presentation, user interaction, application coordination, forms, configuration UI, and display logic.

Reusable RNG and encounter algorithms belong in `owoow.Core`.

## MainWindow discipline

`MainWindow.cs`, `MainWindow.Designer.cs`, and `MainWindow.resx` are large and high-risk files.

When changing the main window:

- make the smallest targeted edit possible,
- avoid unrelated refactors,
- do not move RNG or encounter algorithms into event handlers,
- prefer calling Core APIs from the UI layer,
- do not rename controls unless the task requires it,
- do not mix broad UI redesign with RNG algorithm changes in the same task.

## Designer and resource files

Treat `*.Designer.cs` and `*.resx` as generated or tooling-managed artifacts.

- Do not rewrite large sections manually.
- Do not mass-sort or reformat generated content.
- Preserve control names and event wiring unless intentionally changed.
- For a small UI change, keep the generated-file diff correspondingly small when practical.

## UI architecture

Event handlers should coordinate user input, UI state, validation, and calls into Core. They should not become alternate implementations of RNG, encounter, connection, or search algorithms.

If logic can be tested without a Form instance, prefer placing it in `owoow.Core` or a small non-visual helper with a clear responsibility.

Keep long-running work off the UI thread where existing architecture supports asynchronous execution. Preserve cancellation/error reporting patterns.

## Console connection and memory operations

UI controls may initiate console reads/writes, but address calculation and reusable connection behavior should stay out of presentation code where practical.

Do not add a memory write merely to simplify UI behavior when a read-only solution exists.

Clearly surface failures rather than swallowing connection, parsing, or validation errors.

## Configuration

Preserve backward compatibility with existing user configuration where practical.

When adding a setting:

- choose a stable name,
- define a safe default,
- consider behavior for existing config files that do not contain the new property,
- avoid unrelated changes to configuration serialization.

## Validation

After WinForms code changes, at minimum verify:

```powershell
dotnet build owoow.slnx -c Release
```

For UI changes, also inspect the affected form/control behavior when practical. A successful build alone does not verify layout, event wiring, or interaction behavior.
