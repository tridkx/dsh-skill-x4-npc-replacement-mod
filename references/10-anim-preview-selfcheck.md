# §14 改完资产先离线自查：`x4-anim-preview` 预览器

> **这一节与 §10 的区别**：§10 讲的是**你自己工程里的**判据工具（`verify_mod.py`、
> `check_normals.py` 这一批）与发版清单；本节讲的是**一件现成的外部预览器**，
> 它能直接读游戏里的 `.xsm` 动画来驱动 mod 的 `.xac` 网格，
> 于是"动作问题"可以在进游戏之前就被看见。
>
> 该工具**不在本 skill 仓库里**，它是一个独立的工程：
> **<https://github.com/tridkx/x4-anim-preview>**（本地目录丢了就 clone 回来，见 §14.0）。
> 路径示例按工作区布局写成 `x4-anim-preview/tools/xxx.py`；换工程时只需换路径，
> **判据口径不能改**。

## §14.0 工具在哪、丢了怎么重建

**仓库**：<https://github.com/tridkx/x4-anim-preview>

工作区里的 `x4-anim-preview/` 就是这个仓库的克隆（自带 `.xsm` 解析与软渲染预览器，
不依赖 3ds Max 或官方插件）。**它丢了不影响本 skill 的其它结论**，按下面重建即可：

```bash
git clone https://github.com/tridkx/x4-anim-preview.git x4-anim-preview
cd x4-anim-preview
pip install -r requirements.txt          # numpy / pillow / pyglet
cp config.example.json config.json       # 改 game_dir；或改用环境变量 X4_GAME_DIR
python tools/make_shortcut.py --desktop  # 生成快捷方式（图形界面用）
python tools/viewer.py --vanilla         # 先跑 vanilla，确认工具链正常
```

- `config.json` 里的 `game_dir` **可以写绝对路径，也可以写相对本文件的路径**；
  环境变量 `X4_GAME_DIR` 可覆盖它（换机器 / CI 里更方便）。
- **先 `--vanilla` 自检**：vanilla 都跑不起来时别去怀疑 mod —— 那是工具链问题
  （同 §12「尺子先被证伪」）。
- 重建后先跑一次 `tools/ai_check.py`，能与本节下面的判据口径对上，再拿它判 mod。

## §14.1 它解决什么

`.xac` 静态看起来正常，不代表动画里正常。§10.4.2 说过"预览正常 ≠ 实机正常"，
反过来同样成立：**实机要跑好几轮才能发现的问题，预览器一次就能圈出来**。

它能在游戏外看出的动作问题：

- 关节撕裂 / 断裂（对应 §4.10 的已知取舍）
- 四肢弯错方向（对应 §4.1 的全局姿态问题）
- 猫步（对应 §1.7、`09-legs-and-lateral-damping.md` §13）
- 悬空 / 陷地（对应 §4.6 的脚部地面双向对齐）
- 骨架与 vanilla 不一致（对应 §1.1 的硬规则）
- 绕序翻转（对应 §1.2 / §3.3）

**用它当"改完资产的第一道闸"**：比值没收敛就别进游戏，进了游戏也还是要回来量。

## §14.2 基本用法

**每次改完 mod 资产（head / torso 的 `.xac`），跑一次：**

```bash
python x4-anim-preview/tools/ai_check.py --mod <mod目录> --out report/
```

它按槽位自动识别 head / torso；认不出槽位时退化成"取前两个资产"（会提示）。

**读两样东西：**

1. **`report/report.json`** —— 结构化判据（AI 读这个，不必自己发明阈值）
2. **`report/shots/*.png`** —— 每个动画两张：

   - `<动画>.png` —— **时间序列**，从左到右 `mod(t0) vanilla(t0) mod(t1) vanilla(t1) ...`
   - `<动画>_views.png` —— **环绕视角**，同一时刻绕一圈（默认 4 个角度），
     每个角度先 mod 后 vanilla

   两张都是同一条动画、同一相机；mod 侧**带 albedo 贴图**（能看出颜色错位、纯黑、
   透明这类外观问题），两侧都叠**绿色骨骼线**（穿透显示，用于核对骨架与几何）；
   vanilla 侧是纯色，只作姿态基准。

`report/shots/index.json` 记了每张图的路径与每格含义。

### §14.2.1 可调参数

| 参数 | 默认 | 用途 |
|---|---|---|
| `--mod` | 必填 | mod 目录（内含 head / torso 的 `.xac`） |
| `--out` | `report` | 输出目录 |
| `--anims` | 3 | 检查几条代表性动画 |
| `--samples` | 6 | 每条动画采样多少帧 |
| `--shots` | 3 | 每条动画出几格对比图 |
| `--component` | `character_argon_female_01` | 取动画用的共享 component；**换目标种族要改这里** |
| `--views` | 4 | 环绕图绕几个等分角度（`0` = 关闭） |
| `--azimuth` | 200 | 时间序列那张图的方位角（度） |
| `--elevation` | 6 | 仰角（度），**负值从下往上看**（查腋下 / 裙摆内侧） |
| `--no-textures` | — | 不给 mod 侧上贴图（排"是不是贴图问题"时用） |
| `--no-bones` | — | 不叠加绿色骨骼线 |
| `--width` / `--height` | 330 / 540 | 出图尺寸 |

**视角一定要按症状调**——单角度会漏掉背面、侧面、腋下、裙摆内侧的问题：

```bash
python x4-anim-preview/tools/ai_check.py --mod <目录> --out report/ \
    --views 8              # 环绕 8 个角度（0 = 关闭）
    --azimuth 200          # 时间序列那张的方位角
    --elevation -20        # 仰角，负值从下往上看（查腋下/裙摆）
```

`report.json` 的 `shots[]` 里记了每张图的角度与时刻，不用猜。

**退出码**：`0` = ok，`1` = warn，`2` = fail（可直接用在 CI / 循环里）。

## §14.3 判据怎么读

| 字段 | 含义 |
|---|---|
| `status` | `ok` / `warn` / `fail` |
| `hard_failures` | **必须修**：骨架与 vanilla 不一致 / 绕序翻转 / 顶点数超 15× |
| `flags` | 疑似问题，每条都带 mod 与 vanilla 的比值，**越接近 1 越好** |

单项指标：

- **`joint_tear_cm` > 1.35×** —— 顶点到主导骨的距离偏大，蒙皮可能有问题。
  ⚠️ 长裙、大袖、披风这类**远离骨骼的服装几何会天然偏高**，必须对着 `shots/` 的图确认，
  不要只看数字就下结论。
- **`feet_gap_cm` < 0.85×** —— 两脚间距被收窄，就是"猫步"。
  注意这是**几何**间距，不是骨间距。
- **`ground_gap_min_cm` 与 vanilla 差 > 3 cm** —— 脚悬空或陷进地板。
- **`matched` < 0.5** —— 动画节点对不上骨架，动画根本驱动不了这个资产。
  （这条与 §1.1「骨架逐字节」是同一件事的两种观测方式。）
- 顶点数比（flag 里的 `N×`）与 §5 的预算是同一口径：**6× 起留意频闪**。

## §14.4 三条纪律

1. **判据是比值，不是绝对值。** 实测"只推到 vanilla 的 77% 仍然看得出猫步"——
   要跟 vanilla 比，不是跟上一版比（同 §1.7、§10.4.1）。
2. **数字报警了一定要去看图。** 指标是筛子不是判决，尤其服装类几何。
3. **修完再跑一次**，比值应该向 1 收敛；如果没变化，先确认你改的是不是游戏真正加载的
   那一份资产（历史上踩过"对着旧产物调试"的坑，见 §9.3、§10.3.2）。

## §14.5 其他入口

```bash
python x4-anim-preview/tools/skeleton_check.py --mod <目录>   # 只查骨架一致性
python x4-anim-preview/tools/studio.py                        # 图形界面，人工细看
python x4-anim-preview/tools/viewer.py --mod <目录>            # 命令行播放
```

- `skeleton_check.py` 的判据：共同骨名覆盖率、共同骨的 bind 位置偏差 < 1e-4 cm、
  bind 旋转偏差 < 0.01°（按骨名配对，不按出现顺序——原版躯干里同一套骨重复多次，
  见 §1.1 的 364 = 4×91）；mod 多出来的骨只提示，不算错。
- 工作区里各 mod 工程（`lumine/`、`boru/`、`ganyu/`、`emilie/`、`work/`）的产物目录都能
  直接作为 `--mod` 传入；工具会自动识别 head / torso 槽位。

## §14.6 别搞反因果：预览器是筛子，不是判决

预览器走的是**它自己那条软渲染链路**，与实机链路不同（§10.4.2 的完整案例）。
所以：

- 它说 `fail` → **一定有东西错了**，去图上找，去 `report.json` 里找比值；
- 它说 `ok` → **只代表它查的那几项过了**，不代替 §10.3 的发版清单，更不代替实机；
- 它报出来的每一个数字，都应当能用 `shots/` 里的图**指认到具体部位**；
  指认不出来，就先怀疑判据本身（§12 "尺子先被证伪"）。
