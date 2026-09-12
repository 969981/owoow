# Reverse-Engineering Environment

This document records the currently known local reverse-engineering environment for development and research on this repository.

It is intentionally machine-specific. Paths and installed-tool status describe the current Windows + WSL workstation and must not be treated as portable defaults for other machines.

## Execution rules

Before performing reverse-engineering work that depends on local tooling:

1. Read the repository-level `AGENTS.md` and the nearest scoped `AGENTS.md`.
2. Read this document when the task involves static analysis, dynamic analysis, binary patching, hooks, structure recovery, symbol recovery, IDA automation, Android reverse engineering, emulation, or similar work.
3. Select the minimum necessary reverse-engineering skill for the task.
4. Verify that the required executable or runtime actually exists before using it.
5. Do not assume that a tool mentioned by a skill is installed locally.
6. Before analyzing an input binary, record or verify:
   - absolute input path,
   - SHA-256,
   - file format,
   - target architecture,
   - applicable game/software version or build when known,
   - whether the analysis is static or dynamic.
7. If a skill is present but its execution dependency is missing, report exactly that condition rather than pretending the tool was used: `skill 已有，执行依赖缺失`.

For Windows/WSL interop, do not assume a Windows-installed executable is present in the WSL `PATH`. Use the verified Windows path explicitly when appropriate.

---

## 1. IDA Professional 9.4

Use **IDA Professional 9.4** by default for IDA-based work unless the task explicitly requires another version.

Do not default to the older IDA 9.2 environment.

### Version

```text
9.4.260714.951e98a4
```

### Installation paths

Windows installation directory:

```text
D:\Tool\IDA Professional 9.4
```

WSL view:

```text
/mnt/d/Tool/IDA Professional 9.4
```

GUI executable:

```text
D:\Tool\IDA Professional 9.4\ida.exe
```

Batch executable:

```text
D:\Tool\IDA Professional 9.4\idat.exe
```

IDAPython switcher:

```text
D:\Tool\IDA Professional 9.4\idapyswitch.exe
```

IDA bundled Python 3.11:

```text
D:\Tool\IDA Professional 9.4\python311\cpython-3.11.15-windows-x86_64-none\python.exe
```

IDALib:

```text
D:\Tool\IDA Professional 9.4\idalib.dll
D:\Tool\IDA Professional 9.4\idalib
```

IDA Clang:

```text
D:\Tool\IDA Professional 9.4\tools\idaclang\idaclang.exe
```

This installation uses:

```text
ida.exe
idat.exe
```

Do **not** invent or assume an `ida64.exe` executable exists.

---

## 2. IDA user configuration and plugins

IDA user directory:

```text
C:\Users\10626\AppData\Roaming\Hex-Rays\IDA Pro\
```

WSL view:

```text
/mnt/c/Users/10626/AppData/Roaming/Hex-Rays/IDA Pro
```

Plugin directory:

```text
C:\Users\10626\AppData\Roaming\Hex-Rays\IDA Pro\plugins\
```

Confirmed plugins:

- `d810-ng`
- `findcrypt-yara`

Relevant configuration and output paths include:

```text
cfg\idapython.cfg
ida-config.json
keypatch.cfg
logs\d810_logs
```

Existing IDA 9.4 helper tools:

```text
tools\ida94-plugin-probe.py
tools\ida94-python-probe.py
tools\launch-ida-9.4-isolated.cmd
tools\launch-ida-9.4-isolated.ps1
```

Prefer these existing scripts for IDA 9.4 probing and isolated startup. Do not create duplicate temporary launcher logic unless the existing scripts are insufficient for the task.

---

## 3. Confirmed Windows tools

### Android / screen control

scrcpy 4.1:

```text
D:\SDK\Android\scrcpy-win64-v4.1\scrcpy.exe
```

scrcpy 2.4:

```text
D:\Tool\scrcpy-win64-v2.4\scrcpy.exe
```

QtScrcpy:

```text
D:\Tool\QtScrcpy-win-x64-v4.1.1\QtScrcpy.exe
```

ADB and fastboot have been confirmed on the Windows side, but their exact path should still be resolved before scripted use when necessary.

### Runtime

Node.js:

```text
D:\SDK\nodejs\node.exe
```

Python:

```text
D:\SDK\anaconda3\python.exe
```

JDK 21:

```text
D:\SDK\Java\OpenJDK21U-jdk_x64_windows_hotspot_21.0.12.1_1\jdk-21.0.12.1+1\bin\java.exe
```

### Other confirmed Windows applications

- Cheat Engine
- Docker Desktop
- IDA Professional 9.4

GUI tools may still require explicit path discovery before automated invocation.

---

## 4. Confirmed WSL tools

The current WSL environment has been confirmed to provide these commands directly:

- `python3`
- `objdump`
- `readelf`
- `nm`
- `strings`
- `file`
- `npm`
- `npx`
- Docker Windows CLI integration

Do not treat Windows-installed tools as automatically available in the WSL `PATH`.

Useful preflight examples:

```bash
command -v python3
command -v objdump
command -v readelf
command -v nm
command -v strings
command -v file
```

---

## 5. Tools whose local availability is not yet confirmed

Do not assume any of the following are installed or executable until their actual path and version have been checked:

- `clang`
- `clang-17`
- `clang-18`
- `llvm-objcopy`
- `llvm-mc`
- GDB
- LLDB
- radare2
- rizin
- Ghidra
- Frida
- `frida-ps`
- jadx
- apktool
- Java in WSL
- ADB in WSL
- Binary Ninja
- pwntools
- angr
- Unicorn
- Capstone
- Keystone
- LIEF
- r2pipe
- pyelftools
- x64dbg

If one of these is required, check first rather than substituting an assumption.

Examples:

```bash
command -v clang
command -v llvm-mc
command -v gdb
command -v frida-ps
python3 -c "import capstone; print(capstone.__version__)"
python3 -c "import unicorn; print(unicorn.__version__)"
```

On Windows, prefer explicit executable discovery such as PowerShell `Get-Command` or checking the known installation directory.

---

## 6. `zhaoxuya520/reverse-skill`

Windows path:

```text
C:\Users\10626\.codex\skills\reverse-skill\
```

WSL path:

```text
/mnt/c/Users/10626/.codex/skills/reverse-skill
```

Primary entry:

```text
reverse-skill/SKILL.md
```

Primary router name:

```text
reverse-skill-router
```

Read the corresponding `SKILL.md` before performing a task routed through this skill collection.

Main modules include:

- `reverse-engineering`
- `ida-reverse`
- `ghidra-reverse`
- `radare2`
- `binary-ninja-reverse`
- `binary-diff`
- `patch-diff-exploit`
- `apk-reverse`
- `mobile-reverse`
- `dotnet-reverse`
- `go-rust-reverse`
- `js-reverse`
- `protocol-reverse`
- `firmware-pentest`
- `hardware-security`
- `digital-forensics`
- `malware-analysis`
- `pwn-chain`
- `ctf-sandbox`
- `code-audit`
- `thick-client`
- `browser-extension-reverse`
- `reverse-engineering/dsl-vm-reverse`

Complete tool index:

```text
/mnt/c/Users/10626/.codex/skills/reverse-skill/tool-index.md
```

A tool appearing in `tool-index.md` means the skill can recognize or work with that tool. It does **not** mean the tool is installed on this workstation.

---

## 7. `P4nda0s/reverse-skills`

Base Windows skills directory:

```text
C:\Users\10626\.codex\skills\
```

WSL view:

```text
/mnt/c/Users/10626/.codex/skills
```

### `rev-symbol`

Entry:

```text
rev-symbol/SKILL.md
```

Purpose: recover function and symbol names using strings, constants, code patterns, call relationships, and XREFs.

### `rev-struct`

Entry:

```text
rev-struct/SKILL.md
```

Purpose: recover C/C++ structures from load/store offsets, pointer accesses, arrays, field widths, and calling conventions.

### `rev-frida`

Entry:

```text
rev-frida/SKILL.md
```

Purpose: generate Frida hooks, traces, argument/return observation, memory inspection, and dynamic instrumentation scripts.

Frida itself is currently **not confirmed**. Check before execution.

### `rev-unicorn-debug`

Entry:

```text
rev-unicorn-debug/SKILL.md
```

Purpose: emulate and debug local ARM, ARM64, x86, or x64 code with Unicorn.

Unicorn itself is currently **not confirmed**. Check before execution.

### `rev-dex-dumper`

Entry:

```text
rev-dex-dumper/SKILL.md
```

Purpose: dump DEX from a running Android application through ADB.

### `rev-u3d-dump`

Entry:

```text
rev-u3d-dump/SKILL.md
```

Purpose: process Unity IL2CPP artifacts such as `GameAssembly`, `libil2cpp`, and `global-metadata.dat`, recover symbols, and produce IDA/Ghidra import data.

### `rev-idapython`

Entry:

```text
rev-idapython/SKILL.md
```

Purpose: IDA 9.x automation using IDAPython and IDALib, including IDB access, functions, basic blocks, XREFs, instructions, strings, decompiler/Hex-Rays APIs, and debugger APIs.

Prefer this skill for IDA automation tasks.

### `rev-ios-dump`

Entry:

```text
rev-ios-dump/SKILL.md
```

Purpose: dump decrypted iOS application binaries from a jailbroken device using `frida-ios-dump`.

The upstream skill entry previously used `skill.md`; the installed copy only normalizes the entry name to `SKILL.md`. Its content is otherwise unchanged.

---

## 8. Skill routing rules

Select the smallest necessary skill before execution.

| Task | Preferred skill |
| --- | --- |
| General binary reverse engineering | `reverse-engineering` |
| IDA static analysis | `ida-reverse` |
| IDAPython / IDALib | `rev-idapython` |
| Function/symbol recovery | `rev-symbol` |
| Structure recovery | `rev-struct` |
| Frida hooks | `rev-frida` |
| ARM/ARM64 local emulation | `rev-unicorn-debug` |
| Android APK | `apk-reverse` |
| Android DEX dump | `rev-dex-dumper` |
| Unity / IL2CPP | `rev-u3d-dump` |
| Ghidra | `ghidra-reverse` |
| radare2 | `radare2` |
| Binary version comparison | `binary-diff` |
| .NET reverse engineering | `dotnet-reverse` |
| Protocol analysis | `protocol-reverse` |
| Firmware | `firmware-pentest` |
| Malware analysis | `malware-analysis` |

For tasks spanning multiple domains:

1. read `reverse-skill/SKILL.md`,
2. read only the minimum required specialized `SKILL.md` files,
3. verify the corresponding tool dependencies,
4. execute the analysis only after the environment is confirmed.

Skill presence never proves tool availability.

---

## 9. Tool priority

Default static binary analysis:

```text
IDA Professional 9.4
```

IDA automation:

```text
IDA Professional 9.4 + rev-idapython
```

Symbol recovery:

```text
rev-symbol
```

Structure recovery:

```text
rev-struct
```

Android dynamic instrumentation:

```text
rev-frida
```

Android DEX dumping:

```text
rev-dex-dumper
```

Unity IL2CPP:

```text
rev-u3d-dump
```

Local CPU emulation:

```text
rev-unicorn-debug
```

Only execute a workflow after the corresponding tool has been verified to exist.

---

## 10. Analysis preflight record

Before actual reverse-engineering execution, record enough information to reproduce the analysis.

Recommended preflight block:

```text
Tool: IDA Professional 9.4
Tool path: D:\Tool\IDA Professional 9.4\idat.exe
Tool version: 9.4.260714.951e98a4
Input: <absolute path>
SHA-256: <sha256>
Format: <PE/ELF/NSO/NRO/code.bin/main.dol/etc.>
Architecture: <x86/x64/ARM/ARM64/PPC/etc.>
Target version/build: <version if known>
Analysis mode: static | dynamic
Skill(s): <selected minimum skill set>
Evidence status: VERIFIED | INFERRED | HYPOTHESIS
```

For Windows input hashing:

```powershell
Get-FileHash -Algorithm SHA256 "<path>"
```

For WSL/Linux input hashing:

```bash
sha256sum "<path>"
```

For basic file identification in WSL:

```bash
file "<path>"
readelf -h "<path>"   # ELF only
```

Do not claim that a local tool, plugin, script, debugger, or skill was executed unless it was actually invoked and its output was observed.

---

## 11. Pokémon Sword/Shield 1.3.2 local research assets

The following paths were verified to exist on the current Windows + WSL workstation. Windows paths are the canonical paths for Windows tools; use the WSL mapping only when running commands from WSL.

The Sword/Shield `1.3.2` labels below are based on the project resource description. Before analyzing a binary, verify the exact file, SHA-256, format, architecture, and build using the preflight record above.

| Resource | Windows path | WSL path | Intended use |
| --- | --- | --- | --- |
| Sword unpacked data | `E:\Test\sword` | `/mnt/e/Test/sword` | Pokémon Sword 1.3.2 unpacked files |
| Shield unpacked data | `E:\Test\shield` | `/mnt/e/Test/shield` | Pokémon Shield 1.3.2 unpacked files |
| hactool | `E:\Test\hactool` | `/mnt/e/Test/hactool` | NCA/NSP and related unpacking; root executable is `hactool.exe` |
| nstool v1.9.2 x64 | `E:\Test\nstool-v1.9.2-win_x64` | `/mnt/e/Test/nstool-v1.9.2-win_x64` | NSO/NSP/NCZ and Switch file inspection; executable is `nstool.exe` |
| PokeBot / SysBot reference | `D:\WorkSpace\Pokemon\SaveEditor\PKHeX\PKHeX-Tools\PokeBot` | `/mnt/d/WorkSpace/Pokemon/SaveEditor/PKHeX/PKHeX-Tools/PokeBot` | Sword/Shield pointer, offset, memory-read, and automation implementation reference |
| SysBot.Base project dependency | `D:\WorkSpace\Pokemon\RNG\Outbreaker\SysBot.Base` | `/mnt/d/WorkSpace/Pokemon/RNG/Outbreaker/SysBot.Base` | External `SysBot.Base.csproj` project referenced by owoow; output is `bin\$(Configuration)\net10.0\SysBot.Base.dll` |
| PKHeX source | `D:\WorkSpace\Pokemon\SaveEditor\PKHeX` | `/mnt/d/WorkSpace/Pokemon/SaveEditor/PKHeX` | PKHeX.Core legality, RNG, structure, and game-version implementation reference |
| CaptureSight source | `D:\WorkSpace\Pokemon\RNG\CaptureSight` | `/mnt/d/WorkSpace/Pokemon/RNG/CaptureSight` | SWSH RAM offsets, pointer chains, runtime readers, and structure reference |
| `swsh-mods-exl` | External repository: `https://github.com/Lincoln-LM/swsh-mods-exl` | N/A | exlaunch-based Sword/Shield 1.3.2 hook and mod project reference; no local clone path configured |

### Usage notes

- Keep game dumps, keys, and generated unpacking output outside this repository unless a task explicitly requires a checked-in artifact. Do not commit `prod.keys` or `title.keys`.
- Treat local paths as machine-specific and non-portable. If a path changes, update this section and the SWSH reference index together.
- For a new analysis, record the exact Sword/Shield input path and hash rather than relying only on the parent unpacked-data directory.
- The user-provided `win\_x64` spelling corresponds to the verified directory name `win_x64` on this workstation.
