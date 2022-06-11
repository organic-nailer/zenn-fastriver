---
title: "Open3Dの点群を切り出す"
emoji: "✂"
type: "tech"
topics: ["open3d", "python"]
published: true
---

Open3Dで点群(PointCloud)を切り取ろうと思った際、公式のチュートリアルに用意されているCropは意味がよくわからない。

http://www.open3d.org/docs/release/tutorial/geometry/pointcloud.html#Crop-point-cloud

調べたところいくつか方法が見つかったので紹介します。

準備として50x50x50の立方体をPointCloudで作っておきます。

```python
import open3d as o3d
import numpy as np

line = np.linspace(0, 50, 51, dtype=np.float64)

points = []
colors = []
for x in line:
    for y in line:
        for z in line:
            points.append([x,y,z])
            colors.append([x/50, y/50, z/50])

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

o3d.visualization.draw_geometries([pcd])
```

![](/images/2022-06-11-21-46-51.png)

これでカラーな立方体が表示できました。色の定義から分かるように、x軸が赤、y軸が緑、z軸が青に対応しています。
よって黒の頂点が(0,0,0)、青の頂点が(0,0,1)です。

# 1. 軸に沿った直方体で切り取る

`PointCloud.crop(bounding_box: AxisAlignedBoundingBox)`を使います。

http://www.open3d.org/docs/release/python_api/open3d.geometry.PointCloud.html#open3d.geometry.PointCloud.crop

http://www.open3d.org/docs/release/python_api/open3d.geometry.AxisAlignedBoundingBox.html#open3d.geometry.AxisAlignedBoundingBox

AxisAlignedBoundingBoxは、以下のように一つ目の引数に**原点に一番近い頂点**、二つ目に**原点から一番遠い頂点**の座標を指定します。
ここでいうと、原点に一番近い頂点が(45,10,15)の5x10x20の直方体を定義していることになります。

作成したBoundingBoxをpcd.crop()の引数に入れるとその部分の点群のみが抽出されます。

```python
bb = o3d.geometry.AxisAlignedBoundingBox(
    np.array([[45],[10],[15]]),
    np.array([[50],[20],[35]]),
)

cropped_pcd = pcd.crop(bb)

o3d.visualization.draw_geometries([cropped_pcd])
```

![](/images/2022-06-11-21-45-34.png)

赤っぽいですが想定通りの直方体が現れました。

# 2. 任意の直方体で切り取る

軸に沿わない形の直方体で切り取りたい場合(斜めにしたいなど)はバウンディングボックスに`OrientedBoundingBox`が使えます。

http://www.open3d.org/docs/release/python_api/open3d.geometry.OrientedBoundingBox.html#open3d.geometry.OrientedBoundingBox

こちらは直方体の中心、回転行列、各辺の長さの3つを指定するようです。
回転行列は詳しくは分かりませんが、一行目がx軸の向き、二行目がy軸、三行目がz軸のベクトルになるようにするようです(それぞれ直交)。

```python
obb = o3d.geometry.OrientedBoundingBox(
    np.array([[25],[25],[25]]),
    np.array([[0.5,0.1,-0.1],[-0.1,0.5,0.1],[0.1,-0.1,0.5]]),
    np.array([[15],[5],[25]]),
)

cropped_pcd = pcd.crop(obb)

o3d.visualization.draw_geometries([cropped_pcd])
```

![](/images/2022-06-11-21-44-24.png)

斜めに切り取っているので断面も斜めになっていることが確認できます。

# 3. 2次元パスと軸で切り取る

もう一つ、平面上に書いたパスと直交する軸の範囲を指定して角柱のような形状で切り取る方法があります。
これは`visualization.SelectionPolygonVolume`を使うことで実現できます。

http://www.open3d.org/docs/release/python_api/open3d.visualization.SelectionPolygonVolume.html

まず、SelectionPolygonVolumeオブジェクトを作成し、軸をorthogonal_axisに格納します。以下ではZ軸を指定しています。これはXYZどれかから選べます。
次に軸のどの範囲で切り取るかをaxis_minとaxis_maxで指定します。

そしてbounding_polygonに平面(ここではXY平面)で切り取るパスを指定します。使うのは2次元ですが三次元の点でパスを作らないとエラーになるので注意しましょう(軸方向の値は無視されます)。

その後SelectionPolygonVolume.crop_point_cloud()に切り取りたいPointCloudを入れるとパスと軸の範囲のPointCloudのみ抽出してくれます。

```python
corners = np.array([
    [25, 45, 0],
    [40, 5, 0],
    [25, 20, 0],
    [10, 5, 0],
], dtype=np.float64)

vol = o3d.visualization.SelectionPolygonVolume()
vol.orthogonal_axis = "Z"
vol.axis_max = 50
vol.axis_min = 10

vol.bounding_polygon = o3d.utility.Vector3dVector(corners)

cropped_pcd = vol.crop_point_cloud(pcd)
```

![](/images/2022-06-11-21-31-44.png)

XY平面に書いた矢印の形がZ方向に切り取られていることが分かります。

2と3は自由度がまちまちなので適当に使い分けないといけませんね。
