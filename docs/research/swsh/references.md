# Pokémon Sword / Shield RNG 与逆向研究参考资料

> 适用范围：Pokémon Sword / Shield，重点面向 1.3.2、Overworld RNG、运行时内存、fixedSeed、Hook / Breakpoint、合法性验证与 IDA 逆向。
>
> 本文不是单纯链接列表，而是给 `owoow` 后续研究和 AI/Codex 使用的“参考资料索引 + 核心内容摘要”。
>
> 结论状态遵循仓库规则：
>
> - `VERIFIED`：已有源码、运行时实现或多来源交叉验证。
> - `INFERRED`：证据较强，但仍建议在 Sword/Shield 1.3.2 `main` 中重新验证。
> - `HYPOTHESIS`：仅作为后续逆向线索。

---

## 1. 推荐阅读顺序

如果目标是继续研究剑盾 Overworld RNG / fixedSeed / One-Shot Hook，建议按以下顺序阅读：

1. **Lusamine - SWSH Overworld Spawn RNG**：先理解游戏内部遭遇生成调用顺序和 Shield 1.3.2 函数地址。
2. **当前 `owoow`**：理解社区已经实现并实际使用的 RNG 模拟流程。
3. **Lincoln-LM `rng-ovl`**：理解运行时 Breakpoint、寄存器、Overworld 初始化结构和对象生命周期。
4. **PKHeX `Overworld8RNG`**：理解 32-bit seed 与 EC/PID/IV/Height/Weight 的合法性关联和反推方法。
5. **CaptureSight / SysBot.NET / PyNXReader**：补齐 RAM offset、pointer chain、KCoordinates、Den 等内存结构。
6. **Admiral-Fish RaidFinder / RNGWriteups**：补 Raid RNG、xoroshiro 与 seed recovery 背景。
7. **旧 `swsh-overworld-rng-gui`**：仅用于历史比较，不作为现行算法权威。

### 本地工作站资料路径

以下路径是当前 Windows + WSL 工作站中已确认存在的本地参考资源。机器环境的完整说明见 [`docs/reverse-engineering-environment.md`](../../reverse-engineering-environment.md)。

| 类别 | Windows 路径 | WSL 路径 | 主要用途 |
| --- | --- | --- | --- |
| Sword 1.3.2 解包目录 | `E:\Test\sword` | `/mnt/e/Test/sword` | Sword `main`、RomFS/ExeFS 等解包结果参考 |
| Shield 1.3.2 解包目录 | `E:\Test\shield` | `/mnt/e/Test/shield` | Shield `main`、RomFS/ExeFS 等解包结果参考 |
| hactool | `E:\Test\hactool` | `/mnt/e/Test/hactool` | NCA/NSP 等 Nintendo Switch 文件解包；可执行文件为 `hactool.exe` |
| nstool v1.9.2 x64 | `E:\Test\nstool-v1.9.2-win_x64` | `/mnt/e/Test/nstool-v1.9.2-win_x64` | NSO/NSP/NCZ 等文件检查；可执行文件为 `nstool.exe` |
| PokeBot | `D:\WorkSpace\Pokemon\SaveEditor\PKHeX\PKHeX-Tools\PokeBot` | `/mnt/d/WorkSpace/Pokemon/SaveEditor/PKHeX/PKHeX-Tools/PokeBot` | 参考剑盾指针、偏移、内存读取和自动化实现 |
| SysBot.Base 项目依赖 | `D:\WorkSpace\Pokemon\RNG\Outbreaker\SysBot.Base` | `/mnt/d/WorkSpace/Pokemon/RNG/Outbreaker/SysBot.Base` | owoow 当前引用的 `SysBot.Base.csproj`，构建输出为 `bin\$(Configuration)\net10.0\SysBot.Base.dll` |
| PKHeX 项目 | `D:\WorkSpace\Pokemon\SaveEditor\PKHeX` | `/mnt/d/WorkSpace/Pokemon/SaveEditor/PKHeX` | 参考 PKHeX.Core 的 RNG、合法性、结构和版本实现 |
| CaptureSight 项目 | `D:\WorkSpace\Pokemon\RNG\CaptureSight` | `/mnt/d/WorkSpace/Pokemon/RNG/CaptureSight` | 参考 SWSH RAM offset、pointer chain 和运行时结构 |
| `swsh-mods-exl` | [GitHub 仓库](https://github.com/Lincoln-LM/swsh-mods-exl) | N/A | 参考 Sword/Shield 1.3.2 的 exlaunch Hook/mod 工程；当前未配置本地副本 |

路径状态为 `VERIFIED`；游戏版本标注来自资源说明，实际进行 IDA 或运行时分析前仍需按环境文档完成文件、SHA-256、架构和 Build 校验。

---

# P0：核心参考资料

## 2. Lusamine - SWSH Overworld Spawn RNG

### 链接

- https://gist.github.com/Lusamine/09618e2ccf1f409da1338c13e0e32dc4
- 作者研究目录：https://gist.github.com/Lusamine

### 资料定位

这是目前最重要的 **Sword/Shield 1.3.2 Overworld Spawn 游戏内部逻辑参考**。

文章明确说明：

```text
Encounter generation RNG:
HEAP + 0x4C2AAC18
```

使用 `xoroshiro128+`。

所有文中地址均针对：

```text
Pokémon Shield 1.3.2
```

Sword 地址略有差异，但逻辑一致。

### 核心内容

#### Main / Encounter RNG

```text
HEAP + 0x4C2AAC18
size = 0x10

+0x00 : uint64 s0
+0x08 : uint64 s1
```

这是当前 `owoow`、CaptureSight 等多个项目共同使用的 Main RNG state。

**状态：VERIFIED（多来源一致）**

#### 关键 Shield 1.3.2 函数

| 功能 | Shield 1.3.2 |
|---|---:|
| Lead Ability / encounter modifier | `sub_7100D30AB0` |
| Hidden encounter rate | `sub_7100D30C10` |
| Further overworld Pokémon generation | `sub_7100D311F0` |
| Type Pulling | `sub_7100D31BE0` |
| KO / respawn related logic | `sub_7100D31EA0` |
| Pokédex Recommendation | `sub_7100D32530` |
| Normal Encounter Slot | `sub_7100D328D0` |
| Initial Pokémon values | `sub_7100D32A00` |

在 IDA 中，如果 `main` base 对应标准 Switch 模块映射，可优先从相对位置：

```text
MAIN + D30AB0
MAIN + D30C10
MAIN + D311F0
MAIN + D31BE0
MAIN + D31EA0
MAIN + D32530
MAIN + D328D0
MAIN + D32A00
```

开始恢复符号和调用关系。

**注意：这些函数名是研究阶段命名；实际 IDA 数据库需重新确认版本、base 和函数边界。**

### Hidden Encounter 核心逻辑

文章给出的高层流程包括：

```text
Lead ability roll
    ↓
计算 encounter rate
    ↓
rand.NextInt(100)
    ↓
决定是否生成 hidden spawn
```

步行时失败次数会逐步提高遭遇生成率，文中给出了 22% → 44% → 66% → 88% → 100% 的增长规律，并讨论了 Bike、Cleanse Tag、Pure Incense 以及 encounter rate modifying ability 的影响。

### Encounter Slot / Pokémon Generation

资料明确将过程拆成：

```text
Encounter Slot selection
    ↓
sub_7100D32A00
    初始 Pokémon values
    ↓
sub_7100D311F0
    后续 Overworld Pokémon generation
```

对于我们后续 IDA 工作，这比只知道 RNG 数学算法更重要，因为它直接给出了可追的游戏代码入口。

### 对本项目的价值

优先用于：

- 恢复 Shield 1.3.2 `main` 的函数名称。
- 画 Overworld generation call graph。
- 从 Main RNG 向 fixedSeed 生成位置追踪。
- 对照 `owoow` 中 `Symbol.cs` 的模拟调用顺序。
- 为 One-Shot Hook 找到比“直接改 Main RNG”更局部的 Hook 点。

---

## 3. Lusamine - Overworld RNG in Gen 8 (SWSH)

### 链接

https://gist.github.com/Lusamine/e17168d92019672c1f9dcc1a246d9818

### 资料定位

这是面向机制和合法性关系的说明，比 `SWSH Overworld Spawn RNG` 更偏“概念层”。

### 核心内容

资料指出，使用 Overworld RNG 的 Pokémon 在 overworld block 中保存一组独立属性和一个 **32-bit seed**。

常见独立属性包括：

```text
Species
Form
Level
Shiny policy / result
Nature
Gender
Ability
Held item
Guaranteed IV count
Mark
Brilliant Aura
```

另外还有一个 32-bit seed 用于后续生成：

```text
EC
PID
IVs
Height
Weight
```

核心结论：

```text
32-bit seed
    ↓
xoroshiro128+
    ↓
EC
    ↓
PID
    ↓
IVs
    ↓
Height
    ↓
Weight
```

### 哪些遭遇使用 Overworld RNG

资料列出常见 Overworld RNG 类型，包括：

- 普通 encounter slots。
- Hidden `!` encounters。
- Overworld static / strong spawns。
- Galarian birds、Swords of Justice 等 overworld 保存型遭遇。
- Fishing spots。

而部分 scripted static / gift / 特殊脚本遭遇并不走同一 Overworld RNG 体系。

### 对本项目的价值

用于明确：

- fixedSeed 不是完整 Pokémon 所有属性的唯一来源。
- One-Shot fixedSeed override 理论上更适合只改变 `EC/PID/IV/Height/Weight`，而不改变前面的 Species / Nature / Gender / Ability / Mark 等。
- 研究时必须区分“Overworld 初始化属性”和“fixed-seed-derived 属性”。

---

## 4. `LegoFigure11/owoow` / 当前 `969981/owoow`

### 链接

- 上游：https://github.com/LegoFigure11/owoow
- 当前 fork：https://github.com/969981/owoow

### 资料定位

这是当前实际可运行的 Sword/Shield Overworld RNG 工具，也是我们继续开发的主要代码库。

### 核心源码

```text
owoow.Core/RNG/Generators/Overworld/
├── Common.cs
├── Fishing.cs
├── Fixed.cs
├── Hidden.cs
├── Static.cs
└── Symbol.cs
```

### Symbol generation 的关键流程

`Symbol.cs` 当前的高层结构是：

```text
Main state / outer state
    ↓
Placement
    ↓
Lead Ability
    ↓
Dex Recommendation / Encounter Slot
    ↓
Level
    ↓
Mark roll
    ↓
Brilliant Aura
    ↓
Shiny decision
    ↓
Gender
    ↓
Nature
    ↓
Ability
    ↓
Held Item
    ↓
Aura IV / Egg Move
    ↓
GenerateFixedSeed()
    ↓
新 xoroshiro128+ 实例
    ↓
EC
PID
IVs
Height
Mark / 后续
```

当前代码中：

```csharp
var outer = new Xoroshiro128Plus(s0, s1);
var os = outer.GetState();
var rng = new Xoroshiro128Plus(os.s0, os.s1);
```

说明工具以当前 Main state 为每个 frame 的起点模拟完整的 Overworld consumption。

### fixedSeed

`Fixed.GenerateFixedSeed()` 当前本质是：

```csharp
return (uint)rng.Next();
```

随后：

```csharp
new Xoroshiro128Plus(fixedSeed, XOROSHIRO_CONST)
```

用于生成固定相关属性。

### 需要谨慎的术语

在 `owoow` 语境中，“outer”更多是工具层对 Main state frame progression 的建模。

目前不要直接认定游戏内部存在一个独立、长期存储的 `Outer RNG state`。

更安全的表述：

```text
Main RNG state
    ↓
一次 Overworld generation 中的 encounter RNG consumption
    ↓
32-bit fixedSeed
    ↓
fixed-seed-derived xoroshiro stream
```

**状态：INFERRED，需 IDA / runtime trace 进一步确认游戏内部对象和 state copy 方式。**

### 对本项目的价值

- 作为算法实现基准。
- 用于和 IDA 反编译逐步比对 RNG call order。
- 为 deterministic tests 提供现成实现。
- 适合最终承载 fixedSeed override / one-shot generation 功能。

---

## 5. Lincoln-LM - `rng-ovl`

### 链接

https://github.com/lincoln-lm/rng-ovl

### 资料定位

这是非常重要的 **Runtime Breakpoint / Overworld spawn 对象研究**。

比单纯 RAM reader 更接近我们计划做的 Hook。

### 核心 Breakpoint 地址

`include/swsh/swsh_manager.hpp` 中记录：

| 功能 | Sword | Shield |
|---|---:|---:|
| Overworld generation end | `MAIN+0xD3178C` | `MAIN+0xD317BC` |
| Object creation | `MAIN+0xEA2FB4` | `MAIN+0xEA2FE4` |
| Pokémon object update | `MAIN+0xD5EFB0` | `MAIN+0xD5EFE0` |
| Rain RNG | `MAIN+0xEB6BC8` | `MAIN+0xEB6BF8` |
| Fly GetPublicRand | `MAIN+0x1471C90` | `MAIN+0x1471CC0` |
| Fly related rand | `MAIN+0xD5AD10` | `MAIN+0xD5AD40` |
| NPC rand | `MAIN+0xCECBD0` | `MAIN+0xCEEC00` |

项目还记录了多个 NPC init / sub-object function 地址，用来统计 Menu Close / Fly / NPC advances。

### 最重要的运行时信息

命中：

```text
Shield MAIN + 0xD317BC
```

后，代码读取：

```text
X20 = overworld Pokémon initialization structure pointer
```

即：

```cpp
u64 pokemon_addr = thread_context->cpu_gprs[20].x;
```

这给我们提供了非常明确的动态验证点。

### Overworld 初始化结构

`include/swsh/overworld_pokemon.hpp` 恢复了一个 packed `init_spec`，核心字段包括：

| 相对位置 | 语义 |
|---:|---|
| `+0x00` | Species |
| `+0x08` | Shininess / policy |
| `+0x0C` | Nature |
| `+0x10` | Gender |
| `+0x14` | Ability |
| `+0x1C` | Held Item |
| `+0x20` | Egg Move |
| `+0x35` | Guaranteed IV count |
| `+0x48` | Mark |
| `+0x4A` | Brilliant |
| `+0x50` | `uint32 seed` |

这里的：

```text
init_spec + 0x50
```

就是目前最重要的 **fixedSeed runtime candidate**。

### generatedFixed()

项目直接用：

```text
seed
 ↓
xoroshiro128+
 ↓
EC
 ↓
PID
 ↓
Guaranteed IVs
 ↓
Remaining IVs
 ↓
Height
 ↓
Weight
```

重新生成 fixed information。

这与 PKHeX / owoow 的核心算法高度一致。

### 对本项目的价值

这是当前 One-Shot Hook 研究的 P0 参考：

```text
MAIN+D317BC
    ↓
X20
    ↓
init_spec
    ↓
+0x50 fixedSeed
```

优先验证：

1. Shield 1.3.2 中 `D317BC` 的函数边界和 caller。
2. X20 在命中时是否稳定指向相同结构。
3. `+0x50` 写入时机是否已经晚于 Species/Nature/Gender/Ability/Mark 的决定。
4. 修改 `+0x50` 后是否只影响 EC/PID/IV/Height/Weight。
5. Hook Restore 后 Main RNG 是否继续正常推进。

---

## 6. PKHeX - `Overworld8RNG`

### 链接

- 仓库：https://github.com/kwsch/PKHeX
- 核心文件：`PKHeX.Core/Legality/RNG/Methods/Gen8/Overworld8RNG.cs`

### 资料定位

这是当前最重要的 **Overworld fixedSeed 合法性验证参考实现**。

### 核心结论

PKHeX 明确认为 Overworld entity 被赋予一个：

```text
32-bit seed
```

并使用：

```text
new Xoroshiro128Plus(seed)
```

生成：

```text
EC
PID
IVs
HeightScalar
WeightScalar
```

### Seed recovery

PKHeX 实现：

```csharp
GetOriginalSeed(PKM pk)
```

利用 SWSH xoroshiro 初始化特征和第一个 EC roll，从 PK8 的 EC 等数据恢复原始 32-bit seed。

这意味着我们可以建立非常强的 Hook 验证闭环：

```text
游戏 runtime 读取 / 修改 fixedSeed
          ↓
捕获 Pokémon
          ↓
导出 PK8
          ↓
PKHeX GetOriginalSeed()
          ↓
比较 recovered seed 与 runtime seed
```

### 对本项目的价值

- fixedSeed Hook 的黄金验证器。
- 合法性检查回归测试来源。
- 用于确认 PID shiny correction、Guaranteed IV、Height/Weight 的调用顺序。

---

## 7. CaptureSight

### 链接

https://github.com/zaksabeast/CaptureSight

### 资料定位

成熟的 Switch Pokémon RAM viewer / RNG reader。

其 SWSH offset 文件是当前重要的 RAM 地址参考。

### 主要 SWSH offsets

`libs/csight-core/src/swsh/offsets.rs` 中记录：

| 内容 | Offset |
|---|---:|
| Wild PK8 | `0x8FEA3648` |
| Wild Legendary | `0x886BC058` |
| Trade | `0xAF286078` |
| Raid | `0x886C1EC8` |
| Party | `0x450C68B0` |
| Main RNG | `0x4C2AAC18` |
| MyStatus | `0x45068F18` |
| Vanilla Den | `0x450C8A70` |
| Isle of Armor Den | `0x450C94D8` |
| Crown Tundra Den | `0x450C9F40` |

### 对本项目的价值

- Main RNG offset 交叉验证。
- Den / Raid 内存研究。
- PK8 runtime buffer 对比。
- 与 SysBot.NET / owoow 地址差异分析。

### 注意事项

不同项目中 “Raid Pokémon / Legendary Pokémon” 地址存在差异，不能直接判断某个项目错误。

可能原因包括：

- 读取的是不同生命周期对象。
- 一个是正在生成的对象，一个是 battle / result buffer。
- 结构或读取时机不同。

以后 `memory-map.md` 应同时记录：

```text
Base type
Offset / pointer chain
Structure size
Lifecycle / read timing
Purpose
Source
Version
```

---

# P1：内存、自动化与运行时结构参考

## 8. SysBot.NET

### 链接

https://github.com/kwsch/SysBot.NET

### 资料定位

成熟的 Switch sys-botbase 自动化项目，SWSH 部分长期经过大量实机使用。

### 核心版本信息

`PokeDataOffsetsSWSH` 明确：

```text
SWSHGameVersion = 1.3.2
Sword  = 0100ABF008968000
Shield = 01008DB008C2C000
```

### 重要 RAM offsets

包括：

```text
BoxStart        0x45075880
CurrentBox      0x450C680E
TrainerData     0x45068F18
TextSpeed       0x450690A0
Wild Pokémon    0x8FEA3648
Raid Pokémon    0x886A95B8
Legendary       0x886BC348
```

以及多种 Trade / Raid / Battle / Daycare 状态地址。

### Stable Overworld Pointer

当前代码记录：

```text
MAIN + 0x2636678
    -> +0xC0
    -> +0x80
```

最终值：

```text
1 = On Overworld
0 = Not On Overworld
```

这是我们以后 One-Shot Hook state gating 很有价值的 pointer chain。

### 对本项目的价值

- 连接与版本验证。
- 游戏状态检测。
- Pointer validation / absolute memory read 模式参考。
- 可作为 Hook 安全条件之一：只在 Overworld 状态激活。

---

## 9. Lincoln-LM - PyNXReader

### 链接

https://github.com/Lincoln-LM/PyNXReader

### 资料定位

较早但很有价值的 Sword/Shield RAM / save block / structure 研究项目。

### 核心结构资料

包括：

```text
Daycare8
Den
EncounterNest8
KCoordinates
MyStatus8
NestHoleCrystalEncounter8
PK8Overworld
```

### KCoordinates

`structure/KCoordinates.py` 能从 KCoordinates block 中扫描 Overworld Pokémon 数据，并解析 Overworld record。

当前 `owoow` 也有基于 PyNXReader / SysBot 研究演化而来的 KCoordinates 读取逻辑。

### 对本项目的价值

- Overworld saved/spawned object block 解析。
- Den / raid 数据结构。
- 将 generation-time init spec 与已经进入 KCoordinates 的对象做双向比对。

---

## 10. Lincoln-LM - `swsh-mods-exl`

### 链接

https://github.com/lincoln-lm/swsh-mods-exl

### 资料定位

基于 exlaunch / Switch runtime modification 的 Sword/Shield mod 参考工程。

### 对本项目的价值

它的重要性主要不在 RNG 算法，而在：

- Switch `main` runtime hook / patch 工程结构。
- exlaunch 模块化 Hook 写法。
- 地址组织方式。
- 在实机 Sword/Shield 中部署原生 hook 的参考。

当我们确认 fixedSeed hook 点后，可以参考其工程方式实现 Atmosphère / exlaunch 侧验证。

---

## 11. Admiral-Fish - RaidFinder

### 链接

https://github.com/Admiral-Fish/RaidFinder

> 该仓库已于 2025-11-08 归档，只读。

### 资料定位

Sword/Shield Max Raid RNG 的经典参考工具。

### 核心内容

项目包含：

- Raid seed 读取与搜索。
- 从 save / CaptureSight 获取 seed。
- 根据捕获 Pokémon 的 EC/PID/IV 等信息恢复 raid seed 的思路。
- `raid_solver.py` 使用 Z3 做 seed recovery。

### 对本项目的价值

虽然 Raid RNG 不等于 Overworld RNG，但它对以下内容很重要：

- Sword/Shield xoroshiro 使用背景。
- 64-bit raid seed 与 Pokémon generation 的对照。
- seed recovery 思路。
- 区分 Raid RNG 与 Overworld 32-bit fixedSeed RNG。

不要把 RaidFinder 的“only raids are RNGable”旧 README 结论理解成今天的完整 SWSH RNG 研究状态；Overworld RNG manipulation 后续已经有大量新研究。

---

## 12. Admiral-Fish - RNGWriteups

### 链接

https://github.com/Admiral-Fish/RNGWriteups

### 资料定位

跨世代 Pokémon RNG 机制研究资料库。

### 对本项目的价值

主要用于：

- RNG 方法背景。
- xoroshiro / seed / generation sequence 的历史参考。
- Raid / Egg 等与 Overworld 不同 generation path 的对照。

研究 Overworld 时应优先以 Lusamine + owoow + PKHeX + rng-ovl 为主，RNGWriteups 作为补充。

---

# P2：历史 / 辅助资料

## 13. `LegoFigure11/swsh-overworld-rng-gui`

### 链接

https://github.com/LegoFigure11/swsh-overworld-rng-gui

### 状态

旧项目 README 已明确说明其算法 / 实现已经非常过时，并推荐改用 `owoow`。

### 使用原则

只用于：

- 查看早期社区如何理解 Sword/Shield Overworld RNG。
- 对照历史算法变化。
- 查旧 UI / workflow。

不要用它覆盖 `owoow` 当前实现，也不要把旧 advance / offset 当作 1.3.2 的现行事实。

---

# 14. 当前多来源一致的关键事实

以下内容目前已经得到多个项目互相支持：

## Main RNG

```text
HEAP + 0x4C2AAC18
16 bytes
xoroshiro128+
```

来源：

- Lusamine SWSH Overworld Spawn RNG。
- owoow `Offsets.cs`。
- CaptureSight `offsets.rs`。

**状态：VERIFIED**

## Wild Pokémon buffer

```text
HEAP + 0x8FEA3648
size 0x158
```

来源：

- owoow。
- CaptureSight。
- SysBot.NET。

**状态：VERIFIED**

## Trainer / MyStatus

```text
HEAP + 0x45068F18
```

来源：

- owoow。
- CaptureSight。
- SysBot.NET。
- rng-ovl 用 `+0xA0` 读取 trainer ID32 / TSV。

**状态：VERIFIED**

## Overworld fixed 32-bit seed

多来源一致认为：

```text
32-bit seed
    ↓
xoroshiro128+
    ↓
EC → PID → IVs → Height → Weight
```

来源：

- Lusamine Overworld RNG explanation。
- owoow `Fixed.cs`。
- PKHeX `Overworld8RNG.cs`。
- rng-ovl `overworld_pokemon.hpp`。

**状态：VERIFIED**

## Runtime init spec seed candidate

```text
Shield MAIN + 0xD317BC
    ↓
X20 -> Overworld init spec
    ↓
init_spec + 0x50 = uint32 seed
```

来源：Lincoln-LM `rng-ovl`。

**状态：VERIFIED in rng-ovl implementation / 需要我们自己的 Shield 1.3.2 runtime 复现后升级为项目 VERIFIED。**

---

# 15. 当前存在差异或仍需验证的内容

## Raid / Legendary RAM offsets 不一致

例如：

```text
CaptureSight Raid     = 0x886C1EC8
SysBot.NET Raid       = 0x886A95B8

CaptureSight Legend   = 0x886BC058
SysBot.NET Legendary  = 0x886BC348
```

不要直接合并成一个“正确地址”。

后续应在 `memory-map.md` 中标记对象用途和生命周期。

## “Outer RNG” 的定义

当前 `owoow` 使用 `outer` 对 Main state frames 进行迭代，但尚不能仅凭工具变量名证明游戏内部存在独立存储的 “Outer RNG”。

推荐在文档中暂时使用：

```text
Main RNG state
    ↓
Overworld generation RNG consumption
    ↓
fixedSeed
```

而不是：

```text
Main RNG -> 独立 Outer RNG -> fixedSeed
```

直到 IDA / runtime trace 证明具体 state copy / object model。

**状态：INFERRED**

---

# 16. 建议后续直接建立的研究文档

本 `references.md` 只负责来源和核心内容索引。建议逐步拆出：

```text
docs/research/swsh/
├── references.md
├── memory-map.md
├── rng-pipeline.md
├── function-map.md
├── runtime-breakpoints.md
├── structures.md
└── validation-vectors.md
```

其中第一批优先内容：

### `memory-map.md`

统一记录：

```text
HEAP / MAIN / absolute / pointer-chain
address
size
Sword / Shield
version
structure
lifecycle
source
status
```

### `function-map.md`

从 Shield 1.3.2：

```text
D30AB0
D30C10
D311F0
D317BC
D31BE0
D31EA0
D32530
D328D0
D32A00
```

开始恢复函数语义和 call graph。

### `runtime-breakpoints.md`

优先验证：

```text
MAIN + D317BC
X20
init_spec + 0x50
```

以及：

```text
MAIN + EA2FE4
MAIN + D5EFE0
MAIN + EB6BF8
MAIN + 1471CC0
```

### `validation-vectors.md`

建立闭环：

```text
Main RNG s0/s1
    ↓
Runtime init_spec seed
    ↓
Generated PK8
    ↓
PKHeX GetOriginalSeed()
    ↓
seed 对比
```

每条记录至少包含：

```text
Game
Version
main SHA-256
Main RNG state
fixedSeed
EC
PID
IVs
Height
Weight
runtime breakpoint
PKHeX recovered seed
Result
```

---

# 17. 研究资料使用原则

1. **版本优先**：任何 MAIN-relative 地址都必须带 Sword/Shield + 版本。
2. **Base 类型必须写明**：`HEAP+`、`MAIN+`、runtime VA、pointer chain 不得混写。
3. **算法源码不等于游戏内部结构**：`owoow` 的模拟变量名不能直接作为 IDA 函数/对象事实。
4. **运行时证据优先级高于旧 README 描述**。
5. **PKHeX 用于 fixedSeed/PK8 合法性闭环，不用于证明前半段 Encounter Slot 的 Main RNG 消费顺序。**
6. **旧项目只做历史参考**，尤其 `swsh-overworld-rng-gui`。
7. **地址冲突先研究 lifecycle，再判断对错。**
8. 后续 IDA 结论应使用 `VERIFIED / INFERRED / HYPOTHESIS` 标注。

---

# 18. Reference URLs

```text
Lusamine - SWSH Overworld Spawn RNG
https://gist.github.com/Lusamine/09618e2ccf1f409da1338c13e0e32dc4

Lusamine - Overworld RNG in Gen 8 (SWSH)
https://gist.github.com/Lusamine/e17168d92019672c1f9dcc1a246d9818

Lusamine Research Catalog / Gists
https://gist.github.com/Lusamine

owoow upstream
https://github.com/LegoFigure11/owoow

owoow private development fork
https://github.com/969981/owoow

Lincoln-LM rng-ovl
https://github.com/lincoln-lm/rng-ovl

Lincoln-LM swsh-mods-exl
https://github.com/lincoln-lm/swsh-mods-exl

PKHeX
https://github.com/kwsch/PKHeX

CaptureSight
https://github.com/zaksabeast/CaptureSight

SysBot.NET
https://github.com/kwsch/SysBot.NET

PyNXReader
https://github.com/Lincoln-LM/PyNXReader

Admiral-Fish RaidFinder
https://github.com/Admiral-Fish/RaidFinder

Admiral-Fish RNGWriteups
https://github.com/Admiral-Fish/RNGWriteups

Legacy swsh-overworld-rng-gui
https://github.com/LegoFigure11/swsh-overworld-rng-gui
```
