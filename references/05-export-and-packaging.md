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
