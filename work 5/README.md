# 基于 Taichi GPU 的 Whitted-Style 光线追踪实验
# 熊海昕 202411081109 计算机科学与技术
## 一、实验简介

本实验基于 **Taichi GPU** 实现了一个简易的 **Whitted-Style Ray Tracing（Whitted 风格光线追踪）** 渲染器。

![实验结果](https://github.com/dogbear-china/Graphic_works/blob/main/work%205/ezgif-26137e57b42faf54.gif)

实验完成了以下功能：

- 无限大平面（棋盘格纹理）
- 红色漫反射球
- 银色镜面反射球
- Phong 光照模型
- 硬阴影（Hard Shadow）
- 理想镜面反射（Perfect Reflection）
- GPU 迭代式光线弹射
- 可交互 UI（动态调节光源位置与反射次数）

整个程序完全采用 GPU Kernel 进行光线计算，没有导入任何外部模型，而是采用隐式几何（Implicit Geometry）构建场景。

---

# 二、实验目标

本实验主要学习以下几个内容：

1. 理解 Ray Casting 与 Ray Tracing 的区别
2. 理解 Whitted-Style 光线追踪流程
3. 掌握 Secondary Ray（次级光线）的使用
4. 学习 GPU 上如何将递归算法改写为循环迭代
5. 理解 Shadow Acne（自相交）问题及解决方法

---

# 三、项目结构

```
.
├── main.py        # 主程序
└── README.md      # 实验说明
```

程序只有一个 Python 文件。

主要包括：

- 场景建立
- 光线求交
- 光照计算
- 光线弹射
- GUI 控制

---

# 四、场景组成

实验场景包含三个物体：

## 1. 无限大地板

位置：

```
y = -1
```

法线：

```
(0,1,0)
```

材质：

- 漫反射
- 棋盘格纹理

棋盘格通过交点坐标判断：

```
floor(x*2)+floor(z*2)
```

奇偶性决定颜色：

- 灰色
- 白色

无需贴图即可生成无限棋盘。

---

## 2. 红色漫反射球

位置：

```
(-1.2,0,0)
```

半径：

```
1
```

材质：

Diffuse

颜色：

```
(0.8,0.1,0.1)
```

---

## 3. 银色镜面球

位置：

```
(1.2,0,0)
```

半径：

```
1
```

材质：

Mirror

颜色：

```
(0.9,0.9,0.9)
```

镜面球不会直接着色，而是继续发射反射光线。

---

# 五、核心算法

## 1. 光线求交（Intersection）

程序分别实现：

### 球体求交

采用解析几何公式：

```
(ro + t*rd - center)^2 = radius^2
```

最终得到二次方程：

```
t² + bt + c = 0
```

计算判别式：

```
Δ = b² - 4c
```

若：

```
Δ > 0
```

则存在交点。

---

### 平面求交

对于：

```
y = plane_y
```

直接计算：

```
t = (plane_y-ro.y)/rd.y
```

即可得到交点距离。

---

## 2. 场景求交

函数：

```
scene_intersect()
```

依次检测：

- 红球
- 镜面球
- 地板

记录距离最近的交点。

返回：

- 最近距离 t
- 法线 N
- 颜色
- 材质编号

---

## 3. 材质系统

程序采用简单 Material ID。

```
MAT_DIFFUSE = 0
MAT_MIRROR  = 1
```

不同材质执行不同逻辑：

Diffuse：

- 计算光照
- 结束追踪

Mirror：

- 计算反射方向
- 继续追踪下一条光线

---

# 六、Whitted 光线追踪流程

每个像素都会发射一条 Primary Ray。

```
Camera
   │
   ▼
Primary Ray
   │
   ▼
Scene
```

之后根据材质分两种情况。

---

## 漫反射物体

计算：

- 环境光
- 漫反射
- 阴影

然后结束。

```
Primary Ray
      │
      ▼
Diffuse
      │
      ▼
Phong Lighting
      │
      ▼
End
```

---

## 镜面物体

继续生成新的 Reflection Ray。

```
Primary Ray
      │
      ▼
Mirror
      │
      ▼
Reflection Ray
      │
      ▼
Scene
```

直到：

- 打到漫反射物体
- 超过最大反射次数

---

# 七、GPU 迭代式光线追踪

传统 CPU Ray Tracing：

```
Trace(ray)
    Trace(reflection)
        Trace(reflection)
            ...
```

属于递归。

GPU 不擅长递归。

因此程序采用：

```
for bounce in range(max_bounces):
```

不断更新：

```
ro
rd
```

代替递归。

流程如下：

```
Primary Ray

↓

Hit Object

↓

Mirror ?

↓

Yes

↓

Reflect

↓

继续循环

↓

Diffuse ?

↓

Yes

↓

计算光照

↓

Break
```

这种方法更加适合 GPU 并行计算。

---

# 八、镜面反射

反射方向采用：

```
R = I - 2(I·N)N
```

程序实现：

```python
reflect(I,N)
```

反射后更新：

```
ro = p + N*1e-4
rd = reflect(rd,N)
```

同时更新能量：

```
throughput *= 0.8
```

表示每次反射都会损失部分能量。

---

# 九、光线吞吐量（Throughput）

程序使用：

```python
throughput
```

表示光线剩余能量。

初始化：

```
1.0
```

每经过一次镜面反射：

```
throughput *= 0.8
```

最终颜色：

```
final_color += throughput * lighting
```

这样能够模拟真实反射中能量逐渐衰减。

---

# 十、硬阴影（Hard Shadow）

对于每个漫反射交点：

向光源发射 Shadow Ray。

```
Surface

   │

Shadow Ray

   │

Light
```

若：

Shadow Ray 在到达光源之前撞到其它物体，则：

```
In Shadow
```

仅保留环境光：

```
ambient
```

否则：

计算：

```
diffuse
```

最终得到硬阴影效果。

---

# 十一、Shadow Acne（自相交）

若直接从交点继续发射光线：

```
ro = p
```

由于浮点误差，射线会再次撞到自身。

产生：

- 黑色噪点
- 满屏阴影

解决方法：

所有 Secondary Ray 起点偏移：

```
ro = p + N * 1e-4
```

包括：

- Reflection Ray
- Shadow Ray

这种技术称为：

```
Ray Bias
```

是光线追踪中最经典的技巧之一。

---

# 十二、Phong 光照模型

程序采用简单 Phong 光照：

```
Color

=

Ambient

+

Diffuse
```

其中：

环境光：

```
0.2 * objectColor
```

漫反射：

```
0.8 * max(dot(N,L),0)
```

最终：

```
Lighting = Ambient + Diffuse
```

若处于阴影中：

仅保留：

```
Ambient
```

---

# 十三、GUI 交互

程序提供实时控制窗口。

支持：

## 光源位置

```
Light X
Light Y
Light Z
```

可实时观察：

- 阴影移动
- 光照变化
- 镜面反射变化

---

## 最大反射次数

```
Max Bounces
```

范围：

```
1~5
```

不同参数效果：

### Bounce = 1

只有主光线。

镜面球没有镜中世界。

---

### Bounce = 2

镜面球开始出现一次反射。

---

### Bounce = 3~5

可以观察更多反射细节。

---

# 十四、运行方式

安装依赖：

```bash
pip install taichi
```

运行程序：

```bash
python main.py
```

程序启动后即可打开实时窗口。

---

# 十五、实验效果

本实验最终实现：

- 无限棋盘地板
- 红色漫反射球
- 银色镜面球
- GPU 光线追踪
- 实时硬阴影
- 理想镜面反射
- 光线弹射
- 光源实时移动
- 最大反射次数实时调节

整个渲染过程全部运行于 GPU，实现了较好的实时渲染效果。

---

# 十六、实验总结

本实验实现了一个完整的 Whitted-Style 光线追踪系统，掌握了光线追踪的基本流程，包括光线求交、材质分类、Phong 光照、硬阴影检测以及理想镜面反射。同时，将传统递归式光线追踪改写为 GPU 更适合执行的迭代方式，并通过引入光线偏移（Ray Bias）解决了 Shadow Acne 自相交问题。通过交互式 UI，可以实时调整光源位置和反射次数，直观观察阴影和镜面反射效果，加深了对光线传播和 GPU 并行渲染思想的理解。
