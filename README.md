# x4-npc-replacement-mod

> ⚠️ **本仓库内容由 AI 生成。**
> 它是一次真实工程的**过程沉淀**：把《生化危机8：萝丝之影》的成年萝丝
> 移植成《X4：基石》的 Argon 女性 NPC。所有数字、判据、坑位都来自那次实测，
> 由 AI 在工程过程中记录、复盘、整理成文，**未经人工逐条复核**。
> 换角色 / 换来源游戏 / 换 X4 版本时必须重新量，不要照搬数字。

一个给 [DeepSeek Harness](https://github.com/tridkx)（DSH）用的 **skill**：
教 agent 如何给《X4：基石》(X4: Foundations) 做 **NPC 外观替换 mod**。

## 这个 skill 教什么

X4 的 NPC 替换是「**换网格、留骨架**」——动画/注视/挂点全挂在共享 component 上，
macro 只挑 head/torso/props 三个网格槽位，所以替换物必须带上**逐字节相同的
91 骨骼 Biped 骨架**。

在此前提下，skill 覆盖从源模型到可加载 mod 的完整链路：

- **坐标手性**：源空间 `(x, up, forward)` → X4 `(x, forward, up)` 是 **det = −1 的反射**。
  写错符号会前后镜像（而镜像修不回来），不反转三角形绕序会让衣服透明、脸消失。
- **逐骨绑定姿态转移**：为什么一次刚体变换必然失败（两套骨架的姿态与比例都不同），
  以及怎么逐骨搬。
- **骨轴推导**：只能用**主链第一个子骨**——用"子骨均值"会让骨盆旋转 161°，
  把所有下摆/裤子顶点转翻。
- **特殊部位**：眼球（真眼球藏在 face 里 + look-at 会甩飞）、脚底（位移差把脚撬斜）、
  手指（源并拢/X4 张开 → 虎口撕裂）、头颈（脖子长度差 → 缩在衣领里）。
- **法线**：两段式管线里两段都要开平滑着色，否则实机满脸面片；
  而平滑法线顺带消掉法线接缝，导出顶点约 −70%。
- **顶点预算**：X4 实例化渲染，超预算会让空间站频闪、地图卡死。
- **材质**：没有 albedo 的材质必须继承主材质的**颜色与粗糙度**，
  否则军绿夹克上会出现一条深色横带。
- **XAC 导出器的陷阱**：目录已存在时静默保留旧产物、删对象无法移除槽位、
  空槽位在 .xac 里不可表达。
- **发版前自检清单**与**症状决策树**（实机报问题时从哪查起）。

## 结构

```
SKILL.md                        索引 + 三条硬规则 + 数字速查 + 症状索引
references/
  00-scope-and-pipeline.md      适用范围、X4 三层架构、工具链、端到端流程
  01-coordinate-frames.md       三个坐标系、手性/绕序、判据
  02-retargeting.md             逐骨转移、骨轴、折叠骨、眼/脚/手/头颈、权重平滑
  03-decimation-and-normals.md  顶点预算、减面、法线、切线
  04-materials-and-textures.md  材质槽、无 albedo 材质、BC 编码
  05-export-and-packaging.md    导出器约束与陷阱、打包、三个 XML、全替换模式
  06-verification.md            判据工具、指标口径、发版清单
  07-symptom-triage.md          实机症状 → 根因 → 修法
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
| [X Tools](https://www.egosoft.com/download/x4/bonus_en.php) | `XRCatTool.exe` 解包/打包 |
| [X4 Character Converter](https://www.nexusmods.com/x4foundations/mods/2152) | Blender 插件，`.xac` 导入/导出（本例 v0.8.7） |
| [RE-Mesh-Editor](https://github.com/Percyqaz/RE-Mesh-Editor) | 读 RE Engine `.mesh`（换来源则换成对应解析器） |
| Blender 4.2+（本例 5.2 LTS）、Python 3.13、numpy、Pillow | 网格处理与贴图 |

## 参考工程

这套结论来自 [`x4-character-retarget`](https://github.com/tridkx/x4-character-retarget)
——RE8 萝丝 → X4 Argon 女性 NPC 的完整工程（含管线脚本与逐轮工程日志）。
那边有可运行的代码；这边是可以复用的**方法与坑位**。

## Licence

MIT（见 `LICENSE`）。
《Resident Evil》与《X4：Foundations》是各自权利人的商标，本仓库不含任何游戏资产。
