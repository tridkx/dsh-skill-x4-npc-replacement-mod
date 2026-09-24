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
