# Differentiable Rendering 光源位置优化示例

本项目使用 [Taichi](https://www.taichi-lang.org/) 实现一个简单的可微渲染示例：通过渲染一个 Lambertian 漫反射球体，并使用自动微分与 Adam 优化器，反向优化三维光源位置，使当前渲染图像逐步逼近目标参考图像。

程序运行后会打开一个 GUI 窗口：

- 左侧：目标图像，即由预设目标光源位置生成的 Ground Truth。
- 右侧：当前优化中的渲染结果。

随着迭代进行，右侧图像会逐渐接近左侧目标图像，同时终端会输出 Loss 和当前光源坐标。
![图像渲染](https://github.com/dogbear-china/Graphic_works/blob/main/work%206/ezgif-573e4e5724f5bafb.gif)
## 功能特点

- 使用 Taichi CPU 后端，便于跨平台运行。
- 基于球体表面法线与光照方向计算 Lambertian 漫反射强度。
- 使用 Taichi 自动微分 `ti.ad.Tape` 计算光源位置梯度。
- 使用 Adam 优化器更新三维光源坐标。
- 采用 Leaky Lambertian 模型，为阴影区域保留微弱梯度，改善优化稳定性。
- 使用 GUI 实时展示目标图像和当前渲染结果。

## 运行环境

建议使用 Python 3.8 或更高版本。

需要安装依赖：

```bash
pip install taichi
```

如果你的环境中还没有图形界面支持，请确保运行环境能够打开 Taichi GUI 窗口。

## 运行方法

将代码保存为 Python 文件，例如：

```text
differentiable_rendering.py
```

然后在终端运行：

```bash
python differentiable_rendering.py
```

运行后，程序会进行 300 次优化迭代，并每隔 10 次输出一次当前结果。

示例输出：

```text
Target Light Position: [0.8, 0.8, 0.2]
Initial Light Position: [0.200, 0.200, 0.800]
----------------------------------------
Iter 010 | Loss: 0.012345 | Light Pos: [0.398, 0.397, 0.605]
Iter 020 | Loss: 0.006789 | Light Pos: [0.552, 0.551, 0.452]
...
```

## 核心参数

| 参数 | 说明 |
| --- | --- |
| `res = 256` | 单张渲染图像分辨率，显示窗口宽度为 `res * 2` |
| `sphere_center = [0.5, 0.5, 0.5]` | 球体中心位置 |
| `sphere_radius = 0.3` | 球体半径 |
| `TARGET_LIGHT = [0.8, 0.8, 0.2]` | 目标光源位置 |
| `light_pos = [0.2, 0.2, 0.8]` | 初始待优化光源位置 |
| `lr = 0.02` | Adam 优化器学习率 |
| `iter = 300` | 优化迭代次数 |

## 实现原理

程序首先通过 `generate_target()` 生成目标图像。对于图像中的每个像素，程序判断该像素是否落在球体投影范围内：

```python
dist_sq = dx**2 + dy**2
```

如果像素位于球体区域内，则根据球面方程计算对应的三维表面点 `p`，再计算该点的法线方向 `n`。随后根据目标光源位置计算光照方向，并使用 Lambertian 漫反射模型得到目标亮度：

```python
dot_val = n.dot(l_dir)
target_pixels[i, j] = ti.max(0.0, ti.min(1.0, dot_val))
```

在优化阶段，`render_and_compute_loss()` 使用当前光源位置重新渲染图像，并与目标图像计算均方误差：

```python
diff = intensity - target_pixels[i, j]
loss[None] += (1.0 / (res * res)) * (diff ** 2)
```

由于 `light_pos` 开启了梯度追踪：

```python
light_pos = ti.Vector.field(3, dtype=ti.f32, shape=(), needs_grad=True)
```

程序可以通过 Taichi 自动微分得到 Loss 对光源位置的梯度：
