# Differentiable Rendering 光源位置优化示例

本项目使用 [Taichi](https://www.taichi-lang.org/) 实现一个简单的可微渲染示例：通过渲染一个 Lambertian 漫反射球体，并使用自动微分与 Adam 优化器，反向优化三维光源位置，使当前渲染图像逐步逼近目标参考图像。

程序运行后会打开一个 GUI 窗口：
！[](https://github.com/dogbear-china/Graphic_works/blob/main/work7/ezgif-89610a5a9f454dec.gif)
# Taichi 布料质点弹簧仿真

- 左侧：目标图像，即由预设目标光源位置生成的 Ground Truth。
- 右侧：当前优化中的渲染结果。
这是一个基于 [Taichi](https://www.taichi-lang.org/) 的三维布料模拟示例。程序使用质点弹簧系统描述布料，将布料离散为 `N x N` 个质点，并通过结构弹簧连接相邻质点。仿真支持 GPU 加速、实时 3D 渲染和交互式积分方法切换。

随着迭代进行，右侧图像会逐渐接近左侧目标图像，同时终端会输出 Loss 和当前光源坐标。
运行后会打开一个 GGUI 窗口，显示由蓝色质点和灰色弹簧线构成的布料网格。布料上边缘的两个角点被固定，其余质点在重力和弹簧力作用下运动。

## 功能特点

```python
dot_val = n.dot(l_dir)
target_pixels[i, j] = ti.max(0.0, ti.min(1.0, dot_val))
```
step_semi_implicit()
    执行一步半隐式欧拉积分。

在优化阶段，`render_and_compute_loss()` 使用当前光源位置重新渲染图像，并与目标图像计算均方误差：
step_implicit_iter()
    执行一步固定点迭代形式的隐式欧拉积分。

```python
diff = intensity - target_pixels[i, j]
loss[None] += (1.0 / (res * res)) * (diff ** 2)
main()
    初始化布料、创建窗口、处理 GUI、更新物理仿真并渲染场景。
```

由于 `light_pos` 开启了梯度追踪：
## 实现原理

布料被离散为规则网格，每个网格顶点都是一个质点。相邻质点之间通过弹簧连接，弹簧力使用胡克定律计算：

```python
light_pos = ti.Vector.field(3, dtype=ti.f32, shape=(), needs_grad=True)
f_spring = -k_s * (dist - rest_length) * direction
```

程序可以通过 Taichi 自动微分得到 Loss 对光源位置的梯度：
其中 `dist` 是当前两个质点之间的距离，`rest_length` 是弹簧初始长度，`direction` 是两点之间的单位方向向量。

每个仿真子步中，程序会先清空并重新计算所有质点的受力：

```python
with ti.ad.Tape(loss=loss):
    render_and_compute_loss()
force[i] = gravity * mass - k_d * vel[i]
```

最后，使用 Adam 优化器根据梯度更新光源坐标，使渲染结果逐步接近目标图像。
然后遍历所有弹簧，将弹簧力分别累加到弹簧两端的质点上。由于多个弹簧可能同时作用于同一个质点，代码使用 `ti.atomic_add` 保证并行累加时的正确性。

## 三种积分方法

## Leaky Lambertian 说明
### 显式欧拉

普通 Lambertian 模型通常会将负的点积结果截断为 0：
显式欧拉先使用旧速度更新位置，再使用旧受力更新速度：

```python
intensity = max(0, dot_val)
x[i] += v[i] * dt
v[i] += (f[i] / mass) * dt
```

但在可微优化中，如果光源位于背光区域，过早截断可能导致梯度为 0，使优化难以继续。因此代码中使用了带泄漏的形式：
该方法实现简单，但在高刚度弹簧或较大时间步长下很容易发散。

### 半隐式欧拉

半隐式欧拉先更新速度，再使用新速度更新位置：

```python
intensity = ti.max(0.1 * dot_val, dot_val)
v[i] += (f[i] / mass) * dt
x[i] += v[i] * dt
```

这样即使 `dot_val` 为负，仍会保留一部分梯度，有助于优化器从较差的初始光源位置恢复。
相比显式欧拉，它通常具有更好的能量表现和稳定性，因此代码默认使用该方法。

## 代码结构
### 隐式欧拉

```text
generate_target()
    生成目标参考图像。
隐式欧拉尝试使用未来时刻的受力更新未来状态。代码中通过固定点迭代近似求解：

render_and_compute_loss()
    使用当前光源位置渲染图像，并累加 MSE Loss。
```python
for _ in ti.static(range(3)):
    compute_forces_on(x_next, v_next, f_next)
    v_next[i] = v[i] + (f_next[i] / mass) * dt
    x_next[i] = x[i] + v_next[i] * dt
```

main()
    初始化场景、优化器和 GUI，执行优化循环。
```
这种方法会带来更强的数值阻尼，通常更稳定，但每一步计算成本更高。

## 注意事项

- 当前代码使用 `ti.init(arch=ti.cpu)`，如果希望使用 GPU，可根据本机环境改为 `ti.gpu`。
- GUI 窗口关闭后程序会停止显示，但主循环逻辑仍由 `gui.show()` 控制。
- 如果优化震荡明显，可以适当降低 `lr`。
- 如果收敛速度较慢，可以增加迭代次数或调整初始光源位置。
- `k_s` 较大时，显式欧拉非常容易出现爆炸式发散。
- 如果窗口无法打开，请确认当前运行环境支持 Taichi GGUI。
- 如果 GPU 初始化失败，可以将 `ti.gpu` 改为 `ti.cpu`。
- `dt` 越小仿真越稳定，但需要更多子步才能达到同样的视觉速度。
- 当前代码只包含结构弹簧，没有加入剪切弹簧和弯曲弹簧，因此布料效果较基础。

## 预期结果
## 预期效果

优化开始时，右侧当前渲染图像与左侧目标图像差异较大。随着迭代进行，Loss 会整体下降，当前光源位置会逐渐向目标光源位置 `[0.8, 0.8, 0.2]` 靠近，右侧图像也会越来越接近左侧参考图像。
运行程序后，布料网格会在重力作用下下垂，并围绕固定的两个角点运动。切换不同积分方法后，可以观察到稳定性和阻尼效果的差异：显式欧拉更容易发散，半隐式欧拉较稳定，隐式欧拉更平滑但阻尼更明显。
