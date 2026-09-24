# §9 导出、打包与 XML

## §9.1 导出器（X4CharacterConverter）的工作模型

它**不是从零导出**，而是「**导入原版 XAC → 在 Blender 里改 → 原地重建**」：

- `export_xac()` 只做**原地编辑**（改位置 / 材质 ID / 权重），**顶点数不能变**
- `body_format.rebuild_actor()` 则**重建网格**，允许改拓扑
- 导出包会一并产出 `.xac` + `.dds.gz` + `material_library.xml`

**因此宿主必须是一个原版 XAC**，其骨架被完整继承。

## §9.2 硬约束清单

| 约束 | 说明 |
|---|---|
| 对象名 | `[A-Za-z0-9_]+` |
| 材质名 | `[a-z0-9_]+\.[a-z0-9_]+` |
| mesh 槽位 | 只能**重建模板里已存在的 `mesh_id`**，所以导出能力被宿主的槽位数钉死 |
| XAC 落盘路径 | `write_package()` **硬编码**把每个 `.xac` 写进 `bodies/`（它没有 head/torso 概念），需在组装阶段按名字归位 |
| DDS 像素格式 | 必须 BC1/BC3/BC5/BC4 |
| **DDS 头** | 它找 DXGI 格式的偏移是 byte 128 —— 而 DX10 头里那是 `dwSize`(=60)，所以**只接受 legacy FourCC 头**（`DXT1`/`DXT5`/`ATI1`/`ATI2`） |
| 贴图查找 | 按 **Blender 节点名**（`Diffuse`/`Normal`/`Smoothness`），不是材质属性 |
| 权重和 | 必须**精确 1.0**，否则 `vertex N weights sum is 0.999885, expected 1.0` |
| 顶点权重 | 不能有**无权重**顶点：`vertex 0 has no bone weights` |
| 三角形 | 不能有**退化三角形**：`contains a degenerate triangle after seam welding` |
| 偏好设置 | 必须指定 "Unpacked X4 Root"（解包的游戏根目录） |

## §9.3 三个"改完没效果"的陷阱（**调试前先排除**）

### 陷阱 1：`export_package()` 拒绝写入已存在的目录

而且它是在**建完网格之后**才抛错：

```
PackageExportError: Export directory already exists: ...\rose_head
```

**后果**：第二次跑 stage2 时，目录里**旧 `.xac` 原样留着**，所有下游检查都在
看旧数据 —— 人会对着没变的东西调半天。

**修法**：导出前先清目录。

```python
if os.path.isdir(pkg_target):
    shutil.rmtree(pkg_target)
```

### 陷阱 2：删掉宿主对象 ≠ 移除该槽位

导出器**按宿主模板重建每个 `mesh_id`**，删掉对象后那个槽位会带着
**原版几何**回来（本例 slot 1 混进了原版的角膜）。

### 陷阱 3：`.xac` 表达不了"空槽位"

三种尝试都失败：

| 做法 | 结果 |
|---|---|
| 删掉宿主对象 | 原版几何回来了 ✗ |
| 3 个重合顶点 | 导出器：`vertex 0 has no bone weights` ✗ |
| 上面再加权重 | 导入器：`degenerate triangle after seam welding` ✗ |
| **1 cm 的三角形埋在头部内部** | 两边都能过、也看不见 ✓ |

```python
verts = [(0.0, 0.0, 152.0), (1.0, 0.0, 152.0), (0.0, 1.0, 152.0)]
weights = [{'Bip01 Head': 1.0} for _ in range(3)]
```

## §9.4 打包

```bash
XRCatTool.exe -in x4_rose_mod -out x4_rose_mod/ext_01.cat
```

**`-out` 必须以 `.cat` 结尾**，否则报
`Output ... is neither a catalog nor a folder`。
产出 `ext_01.cat` + `ext_01.dat`，两个一起放进
`X4 Foundations/extensions/<mod_id>/`。

**扩展目录里只放**：`content.xml` + `ext_01.cat` + `ext_01.dat`
（不要把 `assets/` 源文件也放进去，否则会覆盖 cat 内容）。

### §9.4.1 `XRCatTool` 的三个坑（解包与打包都会踩）

| 坑 | 现象 | 正确写法 |
|---|---|---|
| `-include` 是**正则** | 写 `-filter '...'` 直接失败 —— **这个 switch 根本不存在** | `-include '^libraries/'` |
| 输出目录必须**预先存在** | 报 `... is neither a catalog nor a folder, aborting`；它**不会自己建目录** | 先 `mkdir -p <out>`（打包时 `-out` 用 `.cat` 结尾的**文件**路径，见上） |
| `$` 在**双引号**里被 shell 吃掉 | 正则里的行尾锚 `$` 被 shell 展开成空 → 匹配范围**静默**变宽/变窄，不报错 | 用**单引号**：`-include '^libraries/.*\.xml$'` |

解包取证示例（只取需要的库文件）：

```bash
mkdir -p unpacked
XRCatTool.exe -in ego_dlc_terran/ext_03.cat -out unpacked -include '^libraries/'
```

## §9.5 三个 XML

### `libraries/character_macros.xml` — 定义新 macro

```xml
<diff>
  <add sel="/macros">
    <macro name="character_argon_female_rose_macro" class="npc"
           ref="character_argon_female_cau_base_01_macro">
      <component ref="character_argon_female_01" />
      <properties><models>
        <model type="head"  ref="extensions/x4_rose_mod/assets/characters/argon/rose/heads/rose_head" />
        <model type="torso" ref="extensions/x4_rose_mod/assets/characters/argon/rose/bodies/rose_body" />
        <model type="props"  ref="none" />
        <model type="props2" ref="none" />
      </models></properties>
    </macro>
  </add>
</diff>
```

- 继承 base macro 是为了拿到共享 component（骨架 + 动画）
- `props`/`props2` 置 `none`：本例把头发/帽子烘进了 head 资产，
  让原版随机发饰再生成会在头顶飘东西

### `libraries/charactergroups.xml` — 挂进外观池

```xml
<!-- 追加：萝丝成为 1/N 个随机选项 -->
<add sel="/characters/character[@name='argon.pilot.female']">
  <select macro="character_argon_female_rose_macro" />
</add>

<!-- 全替换：该池只出萝丝（测试期用） -->
<replace sel="/characters/character[@name='argon.pilot.female']">
  <character name="argon.pilot.female">
    <select macro="character_argon_female_rose_macro" />
  </character>
</replace>
```

**池是嵌套的**：`argon.trader.female` → `argon.civilian.female`（后者才真正列
macro），所以**只替换叶子池**就能覆盖上层。

### `libraries/material_library.xml` — 自定义材质

```xml
<diff>
  <add sel="/materiallibrary" pos="prepend">
    <collection name="rose">
      <material name="jacket_mat" shader="p1_character" blendmode="NONE">
        <properties>
          <property type="BitMap" name="diffuse_map" value="extensions\x4_rose_mod\assets\...\rose_jacket_mat_diff" />
          ...
        </properties>
      </material>
    </collection>
  </add>
</diff>
```

### `content.xml`

扩展声明（id 必须与扩展目录名一致）+ 多语言文本。

## §9.6 测试期的"全部替换"模式

不然要在一堆 NPC 里找那个 1/4 概率的，测一次成本极高。

**做法：从原版 `charactergroups.xml` 自动发现所有"含女性 macro 的池"**，
整节点 `<replace>` 掉：

```python
def discover_pools():
    for m in re.finditer(r'<character\s+name="([^"]+)"\s*>(.*?)</character>', vanilla, re.S):
        macros = re.findall(r'<select\s+macro="([^"]+)"', body)
        females = [x for x in macros if FEMALE_MACRO_RE.match(x)]
        if females:
            pure = len(females) == len(macros)
            if pure or name.startswith('argon.'):     # 纯女性池，或 argon 的混合池
                yield name, len(macros)
```

**自动发现的价值**：比硬编码列表多抓到**派系专属池**
（`antigone.factiondiplomat.female`、`hatikvah.factiondiplomat.female`）。

**要排除的**：`benchmark` / `testcharacter` 这类**男女混合的开发/跑分池** ——
整池替换会影响性能基准。

**切换开关**放在组装脚本顶部：

```python
REPLACE_ALL_ARGON_FEMALE = True   # False = 追加（约 1/N）
```

## §9.7 目标种族的替换策略：逐个 macro 覆盖 `<models>`

### §9.7.1 前提：跨种族常常共用同一个 component

解包 `ego_dlc_terran/ext_03.cat` 的 `libraries/character_macros.xml`，
`character_terran_female_cau_base_01_macro` 的定义里只有：

```xml
<component ref="character_argon_female_01" />
```

即 **Terran 女性与 Argon 女性共用同一个 component** —— 骨架、动画、
host 模板（挂点 / 注视）全部复用。所以换目标种族时，`01-coordinate-frames.md` /
`02-retargeting.md` 的全部结论**继续成立，不需要重做骨架映射**；
变的只是"怎么把这些网格挂到 NPC 上"（见下）。

> 取证命令：`XRCatTool.exe -in ego_dlc_terran/ext_03.cat -out unpacked -include '^libraries/'`
> —— 注意 §9.4.1 的三个坑（目录要先建、`-include` 是正则、单引号）。

### §9.7.2 但替换点完全不同：base 的 `<models>` 会被每个派生 macro 盖掉

Argon 项目只改 `charactergroups.xml`（外观池）就够 —— 池里列出的 macro 直接生效。
**Terran 侧不是这样：每一个派生 macro 都在自己的定义里重写 `<models>` 块**，
把 base macro 的三个网格槽位整个盖掉。于是：

| 做法 | 结果 |
|---|---|
| 改 base macro 的 `/properties/models` | **无效** —— 派生 macro 自己的 `<models>` 优先，base 的槽位根本不被读到 |
| 只改 `charactergroups.xml` 的池 | **漏掉所有剧情/任务 NPC** —— 它们由任务脚本直接 `ref` macro，不经过池 |
| **逐个 macro `<replace>` 整个 `models` 块** | ✓ 正确做法 |

```xml
<diff>
  <!-- 对 find_terran_female_macros.py 输出的每一个 macro 各生成一条 -->
  <replace sel="/macros/macro[@name='character_terran_female_cau_base_01_macro']/properties/models">
    <models>
      <model type="head"  ref="extensions/x4_lumine_mod/assets/characters/terran/lumine/heads/lumine_head" />
      <model type="torso" ref="extensions/x4_lumine_mod/assets/characters/terran/lumine/bodies/lumine_body" />
      <model type="props"  ref="none" />
      <model type="props2" ref="none" />
    </models>
  </replace>
</diff>
```

**整块替换，不是逐个改 `<model>` 子节点**：块的 schema 在各 macro 之间一致，
整块换少写一半 XPath，也不会漏掉某个槽位。

**顺带的两个好处**（相对"整池替换"）：

1. **保留 NPC 身份多样性** —— 池仍在多个 macro 之间挑，NPC 的名字 / 职称 /
   语音 / 派系由 macro 决定，换掉外观但身份还在；整池 `<replace>` 会把一个池里
   的所有 NPC 压成同一个 macro，身份信息跟着塌缩。
2. **剧情 NPC 一并覆盖** —— 脚本直接 `ref` 的剧情/任务 NPC 与池里的随机 NPC
   走同一条路径，不需要额外处理。

### §9.7.3 "哪些 macro 算目标种族"必须按数据判，不能按名字判

判据 = 沿 `ref` 链解析出的**有效**属性：`race` + `female` + `faction`。
macro 自己写的 `race`、名字前缀、`cau/plot/scenario` 这类词都不可靠。反例
（都真实存在于 X4 数据里）：

| macro | 名字/字面值给的暗示 | 有效属性 | 结论 |
|---|---|---|---|
| `character_yaki_female_cau_base_01_macro` | 看名字像别的派系的基础女性 | `race="argon"` | **不是** terran，排除 |
| `character_yaki_female_plot_yaki_civilian_macro` | 同上，派系是 yaki | `race="terran"` | **是**，收入 |
| `character_player_custom_f_terran_cau_macro` | 看起来完全符合 | `race="terran"`，但 `faction="player"` | **玩家自己**，不能换 |
| `character_scenario_combat_ter_gunner_macro` | 名字像 Terran 剧情角色 | torso 是 `char_ter_m_pilot_suit_01`（`_m_` = 男性） | **男性**，排除 |
| `character_scenario_combat_ter_marine_macro` | 名字里**没有性别** | `ref="character_terran_female_cau_base_01_macro"` | **是**女性，收入 |

两条可操作的结论：

- **必须解析 `ref` 链**看 macro 最终挂到哪个 base；名字、前缀、`race` 字面值
  三者都可能与实际不符。
- **性别看网格资产名，不看 macro 名**：`char_ter_m_pilot_suit_01` 的 `_m_` 是男性、
  `_f_` 是女性。名字里没有性别时（第 5 条），这是唯一可靠的线索。

### §9.7.4 兜底判据用"谁选它"，不用"它穿什么"

有一类 macro 的有效 `race` 与直觉相反 —— 外交官：

```
character_ter_f_diplomat_01_macro   ref 的是 Argon helper，有效 race = argon
character_pio_f_diplomat_01_macro   ref 的是 Argon helper，有效 race = argon
但 terran.factiondiplomat.female / pioneers.factiondiplomat.female 两个池选的就是它们
```

按"有效 race = terran"筛会**漏掉**这两个；改用"torso 是 terran 资产"筛又会**越界**：

```
character_yki_f_diplomat_01_macro   Yaki 外交官，恰好发了同一套制服，
                                    出现在 yaki.factiondiplomat.female 池里 → 不该换
```

**正确判据：是否被 `terran.*` / `pioneers.*` 的 female 池选中**
（与"有效 race"取并集，只用于兜底补漏）。

池的检查**必须递归展开 `<select character=...>` 间接链接** —— 池可以嵌套
（§9.5：`argon.trader.female` → `argon.civilian.female`），只看叶子池的
`<select macro=...>` 会漏掉上层链。**这个检查实际抓出过 2 个漏网的外交官 macro**，
是必做项而不是可选优化。

**工具链**：`find_terran_female_macros.py` 输出 **45 个** macro 及每个的入选理由
（落盘成 JSON），`make_mod.py` 读这份 JSON 生成 §9.7.2 的 `<replace>` 列表。
入选理由要写进文件而不是只打印在终端，否则漏网 macro 无法复盘。

## §9.8 导出器硬编码 shader / blendmode（材质设置全部失效）

`X4CharacterConverter/package_export.py` 的 `build_material_library()`：

```python
"shader": "p1_character",
"blendmode": "NONE",
```

**这两行是写死的**：Blender 材质上挂的 `x4cc_shader` / `x4cc_blendmode` 自定义属性
**一律被忽略**，导出结果永远是 `p1_character` + `NONE`（不透明 + 单面）。

**核对前一个项目的成品**（艾梅莉埃）：18 个材质**全部**是 `p1_character` + `NONE`，
尽管构建代码里明确设了 `ALPHA1` —— 那条黑丝**从未真正透明过**，
而项目全程以为它是透明的。

**后果清单**（都表现成"设了却没效果"，且不报错）：

| 想要的 | 实际得到 | 实机表现 |
|---|---|---|
| `ALPHA1`（alpha test 透明） | `NONE` | 该镂空的地方一律不透明 |
| `TWOSIDED`（双面，见 `07-symptom-triage.md` §11.15） | `NONE` | 单面几何背面消失，或反向层同深度频闪 |
| `p1_hair`（头发专用 shader） | `p1_character` | 头发着色与 vanilla 不一致 |

**修法**：`make_mod.py` 本来就要重写 `material_library.xml`（替换
`PUT_YOUR_TEXTURE_PATH_HERE` 占位符），**在同一处**按 manifest 覆盖两列：

```python
def write_material_library(manifest, out_path):
    for mat_name, spec in manifest.items():
        node = find_material(mat_name, out_path)
        if node is None:
            raise SystemExit(f"material {mat_name} not found in export/manifest")
        node.set("shader", spec["shader"])
        node.set("blendmode", spec["blendmode"])
```

**查不到的材质直接报错**，不允许静默退回导出器默认值 ——
静默退回正是"设了却没生效"能潜伏一整个项目的原因。

**最终材质表**（本例，可直接作 manifest 起点）：

```
薄片（hair/acc*/cloth/ribbon/skirt/petti/eyelash/brow/eyelid/face2/eyespec/mouth）
        p1_character（hair 用 p1_hair）+ TWOSIDED
封闭实体（face/eyewhite/iris/teeth）    p1_character + NONE
实测 alpha 面（skin/elem）              p1_character + ALPHA1
```

> **判据**：取打包后 `.cat`（或组装目录）里的 `material_library.xml`，
> **逐材质**断言 `shader` / `blendmode` 与 manifest 一致。
> 只看构建脚本里写了什么是查不出这个问题的。

> 材质侧要准备什么（材质命名、贴图节点名、占位色）见
> `04-materials-and-textures.md` §7.10。
