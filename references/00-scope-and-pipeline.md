# §0 适用范围与前提 / §1 架构、工具、流程

## §0 适用范围与前提

### §0.1 这个 skill 解决什么

把**任意来源**的角色模型做成《X4：基石》里的 NPC 外观。实测来源是 RE8
（RE Engine，203 骨 UE/Mixamo 风格骨架），但方法对 UE、Mixamo、Blender
原生骨架同样适用 —— 只要你能拿到**两样东西**：

1. **带蒙皮权重的网格**（每顶点 → 骨骼名 + 权重）
2. **源骨架的骨骼世界变换**（位置 + 朝向，用来算逐骨转移）

拿不到权重就别做：X4 的引擎只认权重，重新刷权重的工作量远大于移植本身。

### §0.2 可行性前提：X4 是「换网格、留骨架」

X4 的 NPC 外观分三层，**动画与骨架在最下面一层，与网格无关**：

```
libraries/charactergroups.xml   按职业/性别注册「可选外观池」
        ↓ 引用
libraries/character_macros.xml  macro：定义 head / torso / props 三个 mesh 槽位
        ↓ 引用
.xac 网格文件                   真正的模型
        +
libraries/material_library.xml  collection.material → 贴图绑定
```

原生条目（`libraries/character_components.xml`）里，所有 Argon 女性 NPC
**共享**一个 component：

```xml
<component name="character_argon_female_01" class="npc">
  <properties>
    <boneroot name="Bip01" />
    <skeleton>
      <root name="bip01" /><head name="bip01 head" />
      <righteye name="right_eye_dummy" /><lefteye name="left_eye_dummy" />
      <righthand name="attachment_helper" />
    </skeleton>
    <animations> ... 1100+ 条共享动画 ... </animations>
  </properties>
</component>
```

**动画、骨骼根、注视控制器全挂在共享 component 上，macro 只负责挑模型。**
这意味着两件事：

- ✅ 替换外观**不需要**碰动画、不需要写脚本；
- ❌ 替换物**必须**带上逐字节相同的 91 骨骼 Biped 骨架，否则共享动画驱动不了它。

### §0.3 91 根骨骼的构成（知道有哪些骨可绑）

```
Bip01 …                        标准 3ds Max Biped
  Pelvis / Spine / Spine1 / Spine2 / Neck / Head / HeadNub
  {L,R} Clavicle / UpperArm / Forearm / Hand
  Finger0..4 × {0,1,2,Nub}     五指完整骨链（Finger0=拇指）
  {L,R} Thigh / Calf / Foot / Toe0 / Toe0Nub
left_eye_dummy / right_eye_dummy   注视控制器（look-at 驱动）
Attachment_Helper                  手持物挂点
{L,R}_Elbow_{front,back}_Helper / _Delt_Helper / _Pec_* / _Lat_Helper
{L,R}_Knee_* / _Glute_Helper / _Hip_Helper / _Quad_Helper
LookAt_Target
```

**注意**：`Ter_f_cau.phon_*` / `mt_*` 在名称表里是**材质 / morph 槽名而非骨骼**，
容易误判。

**没有的东西**（决定了哪些部位只能做刚体，见 `02-retargeting.md` §4.5–§4.7）：
发丝链、布料链、脚趾以外的足部小骨。

### §0.4 开工前必须量清的 10 件事

1. X4 版本（`version.dat`）与解包资产（`XRCatTool`）
2. **目标种族**的 macro 名与 component 名 —— 注意不同种族可能共用同一个
   component（Terran 女性用的就是 `character_argon_female_01`），
   但**继承方式可能完全不同**，见 §0.5
3. 目标种族的 macro **是否各自重写 `<models>`**（决定改池还是逐个替换）
4. 宿主资产的**mesh 槽位数量**（导出器只能重建已存在的槽位）
5. 源骨架的**骨骼世界变换**（位置 + 朝向）与**父子关系**
6. 源网格的**蒙皮权重**是否可读
7. 源骨架与 X4 骨架的**姿态差**（手/脚/脊柱各差多少度）
8. 源骨架与 X4 骨架的**比例差**（各段骨长比）—— 同时决定关节撕裂的严重程度，
   见 `02-retargeting.md` §4.10
9. 源材质的**贴图通道**（albedo / normal / roughness 怎么打包），
   以及**alpha 到底在材质里还是在贴图里**（MMD 几乎总在贴图 alpha 里）
10. 源贴图的 **UV 朝向约定**（v 向上还是向下）—— 见 `01-coordinate-frames.md` §3.6

第 2、3 条决定"替换策略"；第 7、8 条决定"逐骨转移能不能用、撕裂有多重"；
第 9、10 条决定材质与贴图环节。

### §0.5 目标种族决定替换策略（**动手前第一件事**）

**改哪一层取决于目标种族的 macro 是怎么写的**（三层结构见 §0.2）。
**两种截然不同的继承方式，实测都存在**：

| | Argon 女性 | Terran 女性 |
|---|---|---|
| 派生 macro 的 `<models>` | **继承** base（自己没有） | **每个都重写自己的** |
| 改 base macro 的 models | 有效 | **无效**（被派生 macro 盖掉） |
| 改外观池 | 够用 | 能覆盖普通 NPC，**漏掉所有剧情 / 任务 NPC** |
| 正确做法 | 改池，或新增一条 macro | **逐个 macro 替换 `<models>`** |

**怎么判**：把候选 macro 的 `<models>` 块都打出来，数有几个非空。
全都有 → 逐个替换；都不带 → 改池。

**逐个替换为什么更好**（即使池方案也能用）：

- 池仍然挑选不同的 macro，**NPC 的身份（名字、背景、职称、语音）保持多样**；
  整池替换会把该种族所有女性塌缩成同一个 macro；
- **剧情 NPC 一并覆盖** —— 它们由脚本直接 `ref`，从不经过外观池。

**"哪些 macro 算目标种族"必须按数据判，不能按名字判**：
判据 = 沿 `ref` 链解析出的**有效 `race` + `female` + `faction`**，
外加一条兜底 —— "是否被目标势力的池选中"（**不是**"是否穿该种族的制服"）。

> **别按名字猜。** 实测的坑：名字写着某个势力、实际上 `race` 是另一个种族；
> 同一个势力里有的成员是 A 种族、有的被标成 B 种族；`faction="player"` 的
> 是玩家自己；名字里没有性别的 macro 可能靠 `ref` 才是女性；名字像女性的
> 可能是男性（torso 资产名里带 `_m_`）。

**具体的 XML 写法、完整反例表、兜底判据与自动发现脚本**见
`05-export-and-packaging.md` §9.7。

### §0.6 哪些结论不能跨来源搬

| 可以搬 | 必须重新量 |
|---|---|
| "换网格留骨架"的架构 | 骨架骨数与名字（Biped vs 别的） |
| 坐标手性判据（用眼球 / 脚趾定方向） | 源空间到底是 `(x,up,forward)` 还是 `(x,forward,up)` |
| 逐骨绑定姿态转移的算法 | 骨轴推导里"主链第一个子骨"是哪一根 |
| 两阶段都要设平滑着色 | 减面比例（取决于源网格密度） |
| 折叠骨要锚定 | 哪些骨算"折叠骨"（取决于源骨架命名） |
| 顶点预算的量级（~6× 保守线） | 具体比例（取决于源资产密度） |
| **同一来源、同一套骨架规范的模型，骨骼映射表可以整体复用**（实测两个原神 MMD 模型 scale 差 1.5%，CORE 表一根骨都不用改） | **材质名表绝对不可复用** —— 见 §0.7 |

### §0.7 材质名表必须从**实际材质列表**生成，不能照抄

同来源、同骨架规范、同一批工具链做出来的两个模型，**材质名大部分对不上**。
实测（荧 / 甘雨）：

```
荧    颜 颜2 白目 眉 二重 睫 口 齿 星目 髪 目 头饰 花蕊 衣 飘带 裙 南瓜裤 肌 元素+ 髪+      （20 个）
甘雨  面1 面2 睫 二重 眉 目 目星 口舌 齿 白目 发 角 服 新規 腰饰 神之眼 肌 裙摆 脸红        （19 个）

字面同名：睫 / 二重 / 眉 / 目 / 齿 / 白目 / 肌  —— 只有 7 个
```

**不要用"材质名完全不同"当结论**：真正会翻车的是那些**长得几乎一样、码点却不同**的字：

```
髪 U+9AEA （荧）      髮 U+9AEE （艾梅莉埃）      发 U+53D1 （甘雨）
```

抄错一个，那个材质就静默走到"没有 ascii 词干"的分支 —— 要么被跳过（少一块身体），
要么让贴图步骤直接 `SystemExit`（这个还算响亮）。
**所以材质表必须从实际材质列表生成，不能照抄另一个模型。**

**另外两个要现查的点**：

* 材质的**语义不能按名字猜**：`角` 是麒麟角（头饰），`新規`（"新建"）是后腰的一块布，
  `目` 是瞳孔 —— 归 head 还是 torso 要看**几何与权重**，不看名字；
* 一个材质可能引用**看起来不相干**的图集：甘雨的 `目`（瞳孔）用的是 `发.png` ——
  作者的 atlas 把头发、角、瞳孔打包在一起。填"图集共享表"时漏了这层，
  同一张图会被编两遍。反过来，**"哪些材质共享一张图"要按 UV 区间实测**，
  不要按名字配对。

> **判定材质归属的便宜办法**：按材质求顶点 bbox 与"主导骨 top3"，一眼就能分出
> 头部件（z 高、主导骨是 `頭`）与身体件（主导骨是 `下半身`/`上半身2`/腿骨）。

---

## §1 工具链与端到端流程

### 必要工具

| 工具 | 用途 | 备注 |
|---|---|---|
| **XRCatTool.exe**（X Tools） | 解包 / 打包 `.cat/.dat` | `-out` **必须以 `.cat` 结尾** |
| **X4CharacterConverter**（Blender 插件） | `.xac` 导入 / 导出 | 硬约束很多，见 `05-export-and-packaging.md` §9.2 |
| **RE-Mesh-Editor** | 读 RE Engine `.mesh`（位置 + 权重 + UV） | 换来源游戏时换成对应解析器 |
| Blender 4.2+ | 网格处理、减面、渲染验证 | 本例 5.2 LTS |
| Python 3.13 + numpy + Pillow | 数值处理、贴图 | Blender 自带 Python 没有 Pillow |

### 端到端流程

```
1. 索引游戏 catalog，取原版基准资产（head/torso .xac + macro + component）
2. 源资产提取：网格（含权重）+ 骨架世界变换 + 材质定义
3. ★ 逐骨绑定姿态转移：把网格搬到 X4 的绑定姿态（见 §4）
4. 减面到顶点预算（见 §5）
5. 建 .blend（stage1），逐子网格建网格 + 顶点组 + UV + 平滑着色
6. 材质与贴图：拆通道 → BC 编码 → DDS（见 §7）
7. stage2：导入宿主 .xac → 填充 mesh 槽位 → 导出新 .xac（见 §9）
8. 组装 mod 树 + 三个 XML → XRCatTool 打包
9. ★ 发版前跑校验清单（见 §10）
```

**第 3 步是整个工程的核心**，也是失败率最高的地方。
**第 9 步不能省**：X4 的失败模式（透明、面片、频闪）在离线渲染里各有对应判据。

### 建议的工程布局

#### 单项目（一个 mod）

```
tools/          管线脚本 + 诊断脚本
docs/           工程日志（强烈建议逐轮记录症状→根因→修法）
work/
  vanilla/      原版基准资产（.xac + libraries/*.xml）
  npz/          dump 出来的网格/骨骼（供纯 Python 侧分析）
  preview/      渲染判据图
  x4_rose_mod/  最终 mod 树
```

`docs/` 那份日志的价值极高：本例的多个结论都是"回看日志发现当时的判断是错的"。
每轮实机反馈都记一条 **症状 → 测量 → 根因 → 修法 → 验证数据**。

#### 多项目工作区（**打算做第二个 mod 时先这么摆**）

做第二个 mod 时，最容易犯的错是把"跨项目共享的东西"留在第一个项目的工作目录里 ——
然后第二个项目的脚本要么复制一份（两份渐渐不同步），要么跨目录引用（路径互相咬死）。

**建议一开始就分三层**：

```
<workspace>/                       ← 例如 D:\dsh-x4
├── shared/                        ← 跨项目复用，不进任何项目的 git
│   ├── x4root/                    ← 解包的游戏根（X4CharacterConverter 的 data_root）
│   ├── <converter addon>/         ← Blender 插件本体
│   └── reference-mods/            ← 别人的 mod（对照用）+ 解包内容
├── <project-a>/                   ← 一个 mod 一个 git 仓库
│   ├── tools/  docs/  README.md
│   └── work/                      ← 本项目专属产物（.blend / npz / preview / mod 树）
└── <project-b>/
```

**判断标准**：这个目录里的东西，换个角色 / 换个来源游戏还用得上吗？

| 放 `shared/` | 放 `<project>/work/` |
|---|---|
| 解包的游戏根（含 `libraries/*.xml` 原版定义） | 本项目的 stage1 `.blend` |
| Blender 插件、`XRCatTool.exe` | dump 出来的 `.npz` |
| 别人的参考 mod 及其解包内容 | 渲染判据图、贴图中间产物 |
| 从游戏取的原版基准资产（head/torso 骨架） | 最终 mod 树与打包产物 |

**两个操作要点**：

1. **脚本里的路径要能一眼看出哪些是共享的** —— 例如
   `WORK = r"<ws>\<project>\work"`、`SHARED = r"<ws>\shared"`，
   再 `X4_ROOT = os.path.join(SHARED, "x4root")`。
   本例早期把 `x4root` 放在 `work/` 下，等到要抽出来时发现 19 个脚本各自硬编码了
   它和插件的绝对路径，只能批量替换 + 重跑整条管线验证（一次替换还因为规则重叠
   生成了 `shared\shared` 这种路径，直接 `ModuleNotFoundError`）。
2. **移动共享资产后必须重跑一遍完整管线**（stage1 → stage2 → 组装 → 自检），
   不能只跑自检 —— 自检脚本不碰 Blender，路径错了它照样"全部通过"。
