# x4-npc-replacement-mod

> 仓库：<https://github.com/tridkx/dsh-skill-x4-npc-replacement-mod>

一个给 DeepSeek Harness（DSH）用的 **skill**：
教 agent 如何给《X4：基石》(X4: Foundations) 做 **NPC 外观替换 mod**。

> ⚠️ **本仓库内容由 AI 生成，未经人工逐条复核。** 它是**五次**真实工程的**过程沉淀**：
> 把《生化危机8：萝丝之影》的成年萝丝、《原神》的艾梅莉埃 / 荧 / 甘雨、
> 《女鬼桥 开魂路》的孟柏汝，分别移植成《X4：基石》的 Argon / Terran 女性 NPC。
> 所有数字与判据都来自这些实测，**换角色 / 换来源游戏 / 换 X4 版本时必须重新量**。

## 这个 skill 教什么

X4 的 NPC 替换是「**换网格、留骨架**」——动画 / 注视 / 挂点全挂在共享 component 上，
macro 只挑 head / torso / props 三个网格槽位，所以替换物必须带上**逐字节相同的
91 骨骼 Biped 骨架**。

在此前提下：

- `SKILL.md` 给出**八条硬规则**（骨架逐字节、坐标手性与绕序、两段都要平滑着色、
  目标种族决定替换策略、UV 的 v 轴、背面用 TWOSIDED、横向阻尼不能逐级递减、
  `blendmode` 单值 → 删掉作者的隐藏面），外加数字速查、动手顺序与症状索引；
- `references/` 下 11 个文档承载细节：逐骨绑定姿态转移、骨轴推导、
  眼球 / 脚 / 手指 / 头颈的逐项处理、顶点预算与法线、材质与贴图、导出打包与 XML、
  发版校验清单、实机症状决策树、诊断纪律、腿与步态，
  以及**改完资产后用离线预览器（`x4-anim-preview/tools/ai_check.py`）在进游戏前自查**。

**动手前先读 `SKILL.md` §0**，它说明每一步该读哪个文档。

## 结构

```
SKILL.md                        索引 + 八条硬规则 + 数字速查 + 动手顺序 + 症状索引
references/
  00-scope-and-pipeline.md      适用范围、X4 三层架构、目标种族与替换策略、工具链、端到端流程
  01-coordinate-frames.md       三个坐标系、手性 / 绕序、判据、UV 的 v 轴
  02-retargeting.md             逐骨转移、骨轴、折叠骨与目标唯一性、眼 / 脚 / 手 / 头颈、权重平滑、关节撕裂
  03-decimation-and-normals.md  顶点预算、减面、法线、切线
  04-materials-and-textures.md  材质槽、无 albedo 材质、TWOSIDED、重合层外推、
                                 blendmode 单值 → 删掉作者的隐藏面、BC 编码与 mip 链
  05-export-and-packaging.md    导出器约束与陷阱、打包、三个 XML、逐个 macro 覆盖、
                                 组装脚本的静默陷阱
  06-verification.md            判据工具、指标口径、发版清单
  07-symptom-triage.md          实机症状 → 判据 → 修法（22 条）
  08-diagnostic-discipline.md   诊断纪律：尺子先被证伪、判据方向写反的案例
  09-legs-and-lateral-damping.md 腿与步态：猫步的可测判据、横向阻尼不能逐级递减
  10-anim-preview-selfcheck.md  改完资产先离线自查：x4-anim-preview 预览器的用法、
                                  report.json 判据、看图纪律（工具在独立工程里）
```

## 安装

放进 DSH 的 skills 目录即可（DSH 启动时扫描）：

```bash
git clone https://github.com/tridkx/dsh-skill-x4-npc-replacement-mod.git \
  ~/.dsh/skills/x4-npc-replacement-mod
```

Windows 上是 `C:\Users\<你>\.dsh\skills\x4-npc-replacement-mod\`。
重启 DSH 后在 skill 目录里即可看到 `x4-npc-replacement-mod`。

## 依赖（做这件事本身需要）

| 工具 | 用途 |
|---|---|
| X4: Foundations（本例 9.00） | 目标游戏 |
| [X Tools](https://www.egosoft.com/download/x4/bonus_en.php) | `XRCatTool.exe` 解包 / 打包 |
| [X4 Character Converter](https://www.nexusmods.com/x4foundations/mods/2152) | Blender 插件，`.xac` 导入 / 导出（本例 v0.8.7） |
| [RE-Mesh-Editor](https://github.com/Percyqaz/RE-Mesh-Editor) | 读 RE Engine `.mesh`（换来源则换成对应解析器） |
| Blender 4.2+（本例 5.2 LTS）、Python 3.13、numpy、Pillow | 网格处理与贴图 |

## 参考工程

这套结论来自下面几个完整工程（每个都含管线脚本与逐轮工程日志）。
**艾梅莉埃那次没有单独建仓**，所以下表是五次实测里的四个仓库：

| 工程 | 来源 → 目标 | 它贡献了什么 |
|---|---|---|
| [`x4-character-retarget`](https://github.com/tridkx/x4-character-retarget) | RE8 萝丝 → Argon 女性 | 整套方法论的起点：手性 / 绕序、逐骨转移、导出器陷阱 |
| [`x4-lumine-mod`](https://github.com/tridkx/x4-lumine-mod) | 原神 荧 → Terran / Argon 女性 | 目标种族决定替换策略、UV 的 v 轴、TWOSIDED、猫步（横向阻尼） |
| [`x4-boru-mod`](https://github.com/tridkx/x4-boru-mod) | 女鬼桥 孟柏汝（UE4）→ Argon 女性 | 非 MMD / RE Engine 来源的完整重测、一键构建与自检脚本 |
| [`x4-ganyu-mod`](https://github.com/tridkx/x4-ganyu-mod) | 原神 甘雨 → Argon 女性 | **`blendmode` 是单值 → 删掉作者的隐藏面**、材质名不可跨模型复用 |

那边有可运行的代码；这边是可以复用的**方法与坑位**。

另外还有一个**配套工具仓库**：[`x4-anim-preview`](https://github.com/tridkx/x4-anim-preview)
—— `SKILL.md` §0.4 与 `references/10-anim-preview-selfcheck.md` §14 用的离线预览器就在那里
（直接读游戏 `.xsm` 动画来驱动 mod `.xac`，不进游戏就能看出动作问题）。
它**不在本仓库里**；本地那份丢了 / 换了机器，clone 下来按 §14.0 重建即可。

## Licence

MIT（见 `LICENSE`）。
《Resident Evil》与《X4：Foundations》是各自权利人的商标，本仓库不含任何游戏资产。
