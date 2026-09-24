# §7 材质与贴图

## §7.1 材质槽命名决定材质解析

XAC 里的**网格节点名**直接决定材质怎么解析：

```
网格节点名           → collection   → material
rose.skin              rose           skin
rose.whiteclothes      rose           whiteclothes
rose.head / rose.hair1 rose           head / hair1
```

即 **`<collection>.<material>`**，与 `libraries/material_library.xml` 的
`name` 属性对应。

**硬约束**（导出器强制）：

| 对象 | 允许字符 |
|---|---|
| Blender 对象名 | `[A-Za-z0-9_]+` |
| 材质名 | `[a-z0-9_]+\.[a-z0-9_]+` |

**一个 XAC 可以带 100+ 个材质槽**（原版躯干就带了一堆没用到的），
游戏只解析实际被网格引用的那几个。

## §7.2 贴图角色与格式

| role | X4 格式 | 来源（本例 RE Engine） |
|---|---|---|
| `Diffuse`（必需） | BC1 / BC3 | `_albd`（需要 alpha 用 BC3） |
| `Normal` | BC5 | `_nrmr` 的 RGB |
| `Smoothness` | BC4 | `_nrmr` 的 A，**取反**（粗糙度→光滑度） |
| `Metal` | BC4 | 可选 |

导出器**按 Blender 节点名找贴图**（节点名必须是 `Diffuse` / `Normal` /
`Smoothness` / `Metal` / `Glow`），**不是**按材质属性名。

## §7.3 没有 albedo 的材质：必须继承主材质的**颜色与粗糙度**

这是"衣服颜色不对、腰部有横线"这类问题的根源。

### §7.3.1 症状

夹克上出现一条突兀的深色横带、或一块浅褐色补丁，与周围颜色对不上。

### §7.3.2 成因

源资产里有一类材质**只有 shader 参数、没有 albedo**（典型命名
`*_Stitch_Mat` 缝线、`*_Base_Mat` 底座）。

**关键认知：它们不是细线，是成片的几何。** 本例：

```
Jacket_Stitch_Mat   3799 顶点（占夹克 11%），UV 还是平铺的 v∈[−10, +11]
Metal_Mat           7776 顶点（占夹克 23%）
```

**两个坑叠在一起**：

1. 手工填的深色占位（如 `(58,54,50)`）→ 军绿夹克上一条深色带
2. 缺 smoothness 贴图时导出器写死 `Smoothness = 0.5`，而夹克本体是 0.08（哑光）
   → 那些饰边在哑光大衣上**反光发亮**

### §7.3.3 修法：从主材质推导

```python
# 1) 颜色：读同部件主材质的 UV 区域采样均色（不要用整图平均！）
#    源资产常把上身所有材质打进一张图集，整图平均会被牛仔裤/皮肤拉偏
def material_mean_colours(mat_defs):
    """用各子网格自己的 UV 去采样它引用的 albedo，取平均色"""
    uv = np.asarray(submesh.uvList, np.float32) % 1.0
    px = (uv[:, 0] * (w - 1)).astype(int); py = (uv[:, 1] * (h - 1)).astype(int)
    cols = image_array[py, px]
    return cols.mean(axis=0)

# 2) 粗糙度：读主材质 normalRoughness 贴图的 alpha 通道，取反求均值
def base_material_smoothness(mat):
    rough = np.asarray(Image.open(nrmr).convert('RGBA'))[..., 3] / 255.0
    return 1.0 - rough.mean()

# 3) 生成为纯色 BC1(diffuse) + 纯色 BC4(smoothness)，并写进 manifest
#    —— 导出器只有在 Blender 里存在对应名字的节点时才会写出 smooth_map
```

**映射规则**：`X_Stitch_Mat` → `X_Mat`（正则 `_(stitch|base|detail|decal)_mat$`）。
名字里看不出来的（如 `Metal_Mat`）用显式表。

**实测效果**：

```
Jacket_Stitch_Mat  (58,54,50) → (80,75,66)  军绿灰 ✓ + smooth 0.08
Pants_Stitch_Mat   (62,56,48) → (86,100,115) 牛仔蓝 ✓ + smooth 0.23
Metal_Mat          (117,108,92) → (80,75,66) 压到夹克色 ✓（保留自己的法线/粗糙度）
```

### §7.3.4 眼球贴图的两个额外坑

- **不要把 `_eyeao_albd`（虹膜/瞳孔层）乘进眼球 albedo**：那张图是**着色器参数图**
  （均值 `(24,4,0)`、最大 24、没有亮区），相乘会把眼球渲染成全黑。
- 眼球 albedo 若是**纯 RGB 无 alpha**，**不要**标成 `ALPHA1` —— 否则眼球按透明渲染。

## §7.4 自研 BC 编码器（当外部工具不可用时）

### §7.4.1 没有 albedo 的材质：按 **UV 是否平铺** 二选一

这类材质（缝线 / 底座 / 装饰条）需要一张占位图。**两个极端都会出错**，实测：

| 做法 | 结果 |
|---|---|
| 16×16 纯色占位 | 游戏**拒绝**这张贴图，回退成"贴图缺失"的**洋红** ✗ |
| 复用主材质的贴图 | 若该材质 **UV 是平铺的**，超出 [0,1] 的采样被 **clamp 到图集边缘**（那里是深色 padding）→ **整条饰边变黑** ✗ |
| **与主贴图同尺寸/同格式的纯色（如 1024²）** | ✓ 安全 |

**所以先量 UV 范围再决定**：

```
缝线一族  *_Stitch_Mat        UV  v ∈ [-10, +11]   ← 平铺，必须用纯色
金属件    Metal_Mat           UV  u[0.40,1.00] v[0.06,0.48]  ← 在 0..1 内，可复用主贴图
拉链底座  Jacket_Zipper_*     UV 在 0..1 内        ← 同上
```

**颜色怎么定**：用**该部件主材质、按自身 UV 采样的均色**，别用整图平均
（源资产常把上身所有材质打进一张图集，整图平均会被牛仔裤/皮肤拉偏）。
**粗糙度**也一并继承 —— 缺 smoothness 贴图时导出器会写死 `Smoothness = 0.5`，
而夹克本体是 0.08，那些饰边就会在哑光大衣上反光发亮。

### §7.4.2 **先记住这条**：块压缩格式是格式，不是偏好

```python
# DXT5 / BC3 的块布局是【8 字节 alpha】【8 字节 color】—— 顺序是格式的一部分
out = np.concatenate([a, colour], axis=2)      # 对
out = np.concatenate([colour, a], axis=2)      # 错：解码成 color:=alpha(≈255，白)
```

**写反了会怎样**：每张 BC3 贴图解码后 `颜色 = alpha`（通常 ≈255，所以**整张发白**）、
`alpha = 颜色`。

**为什么很久没被发现**：只有**带 alpha 的材质**才走 BC3。而这里面
**只有深色的那些会露馅** —— 本例的斜挎带源贴图均值 `(40,36,34)` 深色，
转换后成了 `(235,218,234)` 几乎全白，实机就是"帽子下面一个粉色的不明物体"；
同样走 BC3 的头发本来就是浅金色，白一点看不出来。

**教训**：编码器的自检必须**覆盖它输出的每一种格式**。
本例的自检一直只覆盖 BC1/BC4，BC3 是"顺带实现"的，从没被单独验证过。
正确做法是每种格式都跑一次往返：

```python
src = Image.new('RGBA', (64,64), (40,36,34,128))   # 深色 + 半透明 alpha
encode_bc3(src, p)
back = np.asarray(Image.open(p).convert('RGBA'))
assert abs(back[...,:3].mean() - 37) < 5      # 颜色要回来
assert abs(back[...,3].mean() - 128) < 5      # alpha 也要回来
```

**排查提示**：`洋红 (255,0,255)` 在 X4 里通常是**"贴图缺失"**的占位色，
不是贴图内容 —— 先怀疑"游戏没找到/没接受这张贴图"，再怀疑贴图本身画错了。

### §7.4.1 占位贴图：**不要生成 16×16 的纯色**


插件的 `texconv.dll` 在受限环境下可能返回 `E_NOINTERFACE`。按"外部工具失效先绕开"
的原则，用 numpy 自己实现 BC1/BC3/BC4/BC5 是可行的。**四个坑**：

1. **DX10 扩展头少写 4 字节**（`miscFlags2`）→ 数据偏移应为 152 而非 148，
   否则整幅图错位成彩色雪花。
2. **块内 texel 顺序**必须是 `i = x + 4y`（x 变化最快），写成行优先会得到逐块噪点。
3. **transpose 后不能直接 reshape** —— 数组非连续时 numpy 按内存顺序重排，
   会静默打乱 texel。必须 `reshape(...).transpose(...)` 一次写成。
4. **端点不能按亮度取极值**。BC1 每块只有 4 色，按亮度选端点会让
   含"白+黑"的块把红/蓝/品红全压成灰。改为对块内 3×3 协方差做幂迭代求
   **主色轴**，沿主轴取两端。

> 质量自检：真实 2048² 角色贴图上 BC1 39.7 dB / BC4 54.8 dB（>35 dB 视觉无损）。
> **纯随机噪声不是 BC1 的有效测试数据**（每块仅 4 色，随机纹必然 ~12 dB），
> self-test 要用空间平滑的图。

## §7.5 贴图尺寸

2048² 全套会让 mod 到 118 MB。降到 1024² 约 42 MB，且**要在拆分 NRMR 之前缩放**，
保证三张图尺寸一致。
