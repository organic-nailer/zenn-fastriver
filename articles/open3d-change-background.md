---
title: "Open3Dで背景色を変える"
emoji: "🦎"
type: "tech"
topics: ["open3d", "python"]
published: true
---

Open3DのVisualizer、便利なんですが背景が白いので目に刺さりがち...。背景色を変更して目を保護したいと思います。

# 通常

```python
import open3d as o3d

bunny = o3d.data.BunnyMesh()
mesh = o3d.io.read_triangle_mesh(bunny.path)
mesh.compute_vertex_normals()

o3d.visualization.draw_geometries([mesh])
```

`open3d.visualization.draw_geometries()`でオブジェクトを表示するのは上記のようなコードですが、これだと背景が白に固定されます。背景色のオプションは存在しません。

![](/images/open3d-change-background-2023-03-15-11-08-47.png)

# Visualizerを用いてレンダリング

では背景色のオプションがどこに存在するかというと、`visualization.RenderOption`内にあります。

http://www.open3d.org/docs/release/python_api/open3d.visualization.RenderOption.html#open3d.visualization.RenderOption.background_color

RenderOptionは`Visualizer.get_render_option()`からオブジェクトを得ることができます。つまり`draw_geometries()`での表示ではなくVisualizerを用いたレンダリングが必要になることがわかります。

http://www.open3d.org/docs/release/python_api/open3d.visualization.Visualizer.html#open3d.visualization.Visualizer.get_render_option

`draw_geometries()`とほぼ等価な挙動をVisualizerで書き直すと以下のようになります。

```python
vis = o3d.visualization.Visualizer()
vis.create_window()

vis.add_geometry(mesh)

while vis.poll_events():
    vis.update_renderer()

vis.destroy_window()
```

`open3d.visualization.Visualizer()`でインスタンスを作成後、`Visualizer.create_window()`で表示用のウィンドウを生成します。また表示したいオブジェクトは`Visualizer.add_geometry()`で1つづ指定します。
後ろが少し複雑です。ウィンドウのインタラクションを維持しながら表示を続けるために、`Visualizer.poll_events()`を待ちながら`Visualizer.update_renderer()`を繰り返します。`poll_events()`は例えばウィンドウが閉じられるとfalseを返すようになります。

Visualizerオブジェクトにアクセスできるようになったので、RenderOptionを弄れば背景色の変更が可能です。

```python
import open3d as o3d
import numpy as np

bunny = o3d.data.BunnyMesh()
mesh = o3d.io.read_triangle_mesh(bunny.path)
mesh.compute_vertex_normals()

vis = o3d.visualization.Visualizer()
vis.create_window()

render_option = vis.get_render_option()
render_option.background_color = np.asarray([0, 0, 0.2])

vis.add_geometry(mesh)

while vis.poll_events():
    vis.update_renderer()

vis.destroy_window()
```

![](/images/open3d-change-background-2023-03-15-11-18-49.png)

> notebook上でVisualizerを使うと稀によくGLFWのエラーが出てしまうので注意！
