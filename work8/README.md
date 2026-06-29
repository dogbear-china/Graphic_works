# SMPL Linear Blend Skinning (LBS) Visualization
# 熊海昕 202411081109 计算机科学与技术
## 项目简介

本项目基于 **SMPL (Skinned Multi-Person Linear Model)** 人体模型，手动实现并可视化 **Linear Blend Skinning (LBS)** 的完整计算流程，同时与 `smplx` 官方实现进行数值验证。

项目主要完成以下内容：

- 手动实现 SMPL 的 LBS 前向计算流程
- 可视化 Shape Blend Shapes
- 可视化 Joint Regression
- 可视化 Pose Blend Shapes
- 可视化最终 LBS 结果
- 可视化指定关节的蒙皮权重
- 对比官方 `smplx.forward()` 的计算结果
- 自动生成实验图片及统计信息

整个流程完全遵循 SMPL 官方论文中的计算步骤，适合作为 **计算机图形学**、**人体建模**、**动画** 或 **数字人技术** 的实验项目。

---

# 项目结构

```
.
├── models/
│   └── smpl/
│       └── SMPL_NEUTRAL.pkl
│
├── outputs/
│   ├── stage_a_template_weights.png
│   ├── stage_b_shaped_joints.png
│   ├── stage_c_pose_offsets.png
│   ├── stage_d_lbs_result.png
│   ├── comparison_grid.png
│   ├── all_joint_weights.png
│   └── summary.txt
│
└── main.py
```

---

# 环境要求

Python >= 3.9

建议使用 Conda。

需要安装：

```bash
pip install torch
pip install numpy
pip install matplotlib
pip install smplx
```

如果使用官方 SMPL 模型：

```
models/
└── smpl/
    └── SMPL_NEUTRAL.pkl
```

> 本程序内置了 **Chumpy Pickle Shim**，
> 无需安装 `chumpy` 即可加载旧版本 `.pkl` 模型。

---

# 运行方式

默认运行：

```bash
python main.py
```

指定模型目录：

```bash
python main.py \
    --model-dir ./models \
    --out-dir ./outputs
```

指定观察的关节：

```bash
python main.py \
    --joint-id 18
```

指定 Shape 参数数量：

```bash
python main.py \
    --num-betas 10
```

---

# 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--model-dir` | `./models` | SMPL 模型目录 |
| `--out-dir` | `./outputs` | 输出目录 |
| `--joint-id` | `18` | 可视化的关节编号 |
| `--num-betas` | `10` | Shape 参数数量 |

---

# LBS计算流程

程序按照 SMPL 官方流程依次完成：

```
Template Mesh
      │
      ▼
Shape Blend Shapes
      │
      ▼
Joint Regression
      │
      ▼
Pose Blend Shapes
      │
      ▼
Rigid Transformation
      │
      ▼
Linear Blend Skinning
      │
      ▼
Final Mesh
```

---

# 各阶段说明

## (a) Template Mesh + LBS Weights

输出文件：

```
stage_a_template_weights.png
```

内容：

- 初始人体模板
- 关节位置
- 指定关节的蒙皮权重热力图

对应公式：

\[
T(\beta)=\bar{T}+B_S(\beta)
\]

---

## (b) Shape Blend Shapes

输出：

```
stage_b_shaped_joints.png
```

内容：

- Shape Blend Shapes 后的人体
- Shape 后重新回归得到的关节

主要步骤：

- blend_shapes()
- vertices2joints()

---

## (c) Pose Blend Shapes

输出：

```
stage_c_pose_offsets.png
```

内容：

- 姿态修正后的网格
- Pose Offset 大小可视化

主要步骤：

- Rodrigues Rotation
- Pose Feature
- Pose Blend Shapes

对应公式：

\[
T_P(\theta)=T(\beta)+B_P(\theta)
\]

---

## (d) Final LBS Result

输出：

```
stage_d_lbs_result.png
```

内容：

最终经过：

- 刚体层级变换
- Linear Blend Skinning

得到最终人体模型。

对应公式：

\[
M(\beta,\theta)=W(T_P,J,\theta,\mathcal W)
\]

---

# 对比图

输出：

```
comparison_grid.png
```

四个阶段统一展示：

```
(a) Template

(b) Shape Blend

(c) Pose Blend

(d) Final LBS
```

方便观察每一步对于人体模型的影响。

---

# 全关节权重可视化

输出：

```
all_joint_weights.png
```

功能：

- 每个三角面按照影响最大的关节着色
- 不同关节采用不同颜色
- 颜色深浅表示权重大小

便于观察：

- 不同关节控制区域
- Skinning Weight 分布

---

# 数值验证

程序最后调用

```python
model(...)
```

与手写 LBS 结果进行比较。

计算：

- Mean Absolute Error
- Max Absolute Error

结果保存至：

```
summary.txt
```

例如：

```
manual_vs_official_mean_abs_error:
0.00000001

manual_vs_official_max_abs_error:
0.00000005
```

误差接近机器精度，说明手动实现正确。

---

# 输出文件说明

| 文件 | 说明 |
|------|------|
| stage_a_template_weights.png | 模板网格及关节权重 |
| stage_b_shaped_joints.png | Shape Blend 后模型 |
| stage_c_pose_offsets.png | Pose Blend 后模型 |
| stage_d_lbs_result.png | 最终 LBS 结果 |
| comparison_grid.png | 四阶段对比图 |
| all_joint_weights.png | 全关节蒙皮权重可视化 |
| summary.txt | 模型信息及误差统计 |

---

# 实现特点

- ✅ 手动实现完整 SMPL LBS 流程
- ✅ 不依赖官方 Forward 完成核心计算
- ✅ 自动兼容旧版 Chumpy SMPL 模型
- ✅ 与官方实现进行数值验证
- ✅ 支持 Shape 参数自定义
- ✅ 支持关节权重可视化
- ✅ 自动生成实验图片
- ✅ 模块化实现，代码结构清晰

---

# 参考文献

1. Loper M, Mahmood N, Romero J, et al. **SMPL: A Skinned Multi-Person Linear Model**. ACM Transactions on Graphics (SIGGRAPH Asia), 2015.

2. SMPL Official Website

https://smpl.is.tue.mpg.de/

3. SMPL-X GitHub

https://github.com/vchoutas/smplx

---

# License

本项目仅用于学习、科研及课程实验，不包含 SMPL 模型文件。

SMPL 模型版权归 **Max Planck Institute for Intelligent Systems** 所有，请遵守其官方许可协议。
