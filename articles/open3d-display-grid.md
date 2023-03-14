---
title: "Open3Dでグリッドを表示する(LineSet)"
emoji: "🌐"
type: "tech"
topics: ["open3d", "python"]
published: true
---

Open3DのVisualizerでは標準にグリッドを表示する機能がありません。しかしセンサ出力点群などの場合はグリッドを表示できたほうが点群の位置などがわかりやすくなると思うので、自分でグリッドを作ってみます。

![](/images/open3d-grid-2023-03-14-23-00-00.png)

> 例えばVeloViewでは上のようにXY平面上のグリッドが表示される

# Open3Dで線を引く

Open3DのVisualizer上に線を引くのにはgeometry.LineSetを用います。

http://www.open3d.org/docs/release/python_api/open3d.geometry.LineSet.html

```python
import open3d as o3d
import numpy as np

line_set = o3d.geometry.LineSet()

points = np.array([
    [0, 0, 0],
    [2, 0, 0],
    [3, 2, 0],
    [1, 4, 0],
    [-1,2, 0],
])
line_set.points = o3d.utility.Vector3dVector(points)

lines = np.array([
    [0, 2],
    [2, 4],
    [4, 1],
    [1, 3],
    [3, 0],
])
line_set.lines = o3d.utility.Vector2iVector(lines)

line_set.paint_uniform_color([1, 0, 0])

o3d.visualization.draw_geometries([line_set])
```

![](/images/open3d-grid-2023-03-14-23-14-27.png)

LineSetでは、まずpointsプロパティに引きたい線の始点・終点となる点を設定します。形式は(点数, 3)のndarrayをutility.Vector3dVectorで変換したもので、xyzの3次元で設定すればよいです。

次にlinesプロパティにてどの点と点を結ぶかを指定します。形式は(線の数, 2)のndarrayをutility.Vector2iVectorで変換したもので、始点と終点の点のindexを指定すると線分で結んでくれます。例えば`[1,3]`であればpointsに設定した点の1番目と3番目を線で結びます。上記の例では円状に設置した点を1つおきに線で結んでいるため、星の形状になっています。

# グリッドを作る

このLineSetを用いてグリッドを作りましょう。XY平面に線と同心円を書いてみます。

```python
def gen_grid(pitch: float, length: int) -> o3d.geometry.LineSet:
    line_set = o3d.geometry.LineSet()
    max_value = length * pitch
    x = np.arange(-max_value, max_value+pitch, pitch)
    x = np.repeat(x, 2)
    y = np.full_like(x, -max_value)
    y[::2] = max_value
    z = np.zeros_like(x)
    points_Y = np.vstack((x, y, z)).T

    y = np.arange(-max_value, max_value+pitch, pitch)
    y = np.repeat(y, 2)
    x = np.full_like(y, -max_value)
    x[::2] = max_value
    z = np.zeros_like(y)
    points_X = np.vstack((x, y, z)).T

    points = np.vstack((points_X, points_Y))
    line_set.points = o3d.utility.Vector3dVector(points)
    lines = np.arange(points.shape[0]).reshape(-1, 2)
    line_set.lines = o3d.utility.Vector2iVector(lines)

    line_set.paint_uniform_color((0.5, 0.5, 0.5))

    return line_set

def gen_circle(r: float, points: int = 100) -> o3d.geometry.LineSet:
    theta = np.linspace(0, 2*np.pi, points)
    x = r * np.cos(theta)
    y = r * np.sin(theta)
    z = np.zeros_like(x)

    points = np.vstack((x, y, z)).T
    line_set = o3d.geometry.LineSet()
    line_set.points = o3d.utility.Vector3dVector(points)
    lines = np.zeros((points.shape[0]-1, 2), dtype=int)
    lines[:, 0] = np.arange(points.shape[0]-1)
    lines[:, 1] = np.arange(points.shape[0]-1) + 1
    line_set.lines = o3d.utility.Vector2iVector(lines)

    line_set.paint_uniform_color((0.5, 0.5, 0.5))

    return line_set

line_set_r150 = gen_circle(150)
line_set_r10 = gen_circle(100)
line_set_r5 = gen_circle(50)
line_set_r1 = gen_circle(10)
line_set_grid = gen_grid(10, 15)


o3d.visualization.draw_geometries([line_set_grid, line_set_r1, line_set_r5, line_set_r10, line_set_r150])
```

![](/images/open3d-display-grid-2023-03-14-23-53-33.png)

できました😊。
