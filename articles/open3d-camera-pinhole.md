---
title: "Open3Dのカメラの取り扱い PinholeCamera編"
emoji: "📸"
type: "tech"
topics: ["open3d", "python"]
published: true
---

ViewControl編

https://zenn.dev/fastriver/articles/open3d-camera-view-control

Open3Dでカメラを扱う際、一つの手法としてViewControlから色々設定するというのがある(上記参照)のですが、表現が少々特殊なので30°回転するなどの細かいところに手が届かないという問題があります。
そこでOpen3Dでは、ピンホールカメラモデルというものを使ってカメラを細かく設定することができます。

# ピンホールカメラモデル

![](/images/open3d-camera-view-control-2022-06-17-14-58-04.png)

> https://jp.mathworks.com/help/vision/ug/camera-calibration.html より

ピンホールカメラとは、レンズを使わず1つの小さな穴だけが開いたカメラで、理想的には光線が一点で交差するためスクリーン上に反転した像を得ることができます。
理想的なものではありますが、コンピュータ上では問題ないのでこのピンホールカメラモデルによる2次元への射影がよく使われます。

https://qiita.com/harmegiddo/items/1d287c9a02e4b061287f

細かい説明は上の記事を参照していただきたいのですが、ピンホールカメラモデルはカメラの位置を表す外部パラメータとカメラの光学的な作りを表す内部パラメータの2つの行列で表現されます。

## 外部パラメータ

$$
\begin{pmatrix}
R | t \\
\end{pmatrix}
= 
\begin{pmatrix}
r_{11} & r_{12} & r_{13} & t_1 \\
r_{21} & r_{22} & r_{23} & t_2 \\
r_{31} & r_{32} & r_{33} & t_3 \\
\end{pmatrix}
$$

外部パラメータは上のような3行4列の行列で表され、左側が回転行列、右側が平行移動になっています。
外部パラメータにより世界座標を変換すると、まずカメラ座標に合うように回転されてから原点が一致するように平行移動されます。
やっていることはアフィン変換なのでOpen3D内ではこれに一行追加した4行4列で表現されています(下の一行を消せば一致)。

$$
\begin{pmatrix}
r_{11} & r_{12} & r_{13} & t_1 \\
r_{21} & r_{22} & r_{23} & t_2 \\
r_{31} & r_{32} & r_{33} & t_3 \\
0 & 0 & 0 & 1 \\
\end{pmatrix}
$$

## 内部パラメータ

$$
K = 
\begin{pmatrix}
f_x & s & c_x \\
0 & f_y & c_y \\
0 & 0 & 1 \\
\end{pmatrix}
$$

内部パラメータはカメラ座標からスクリーン上の座標に変換するための行列で、上のように表されます。
$f_x$,$f_y$はxy方向での焦点距離で、理想的には同じ値をとります。$c_x$,$c_y$はスクリーンの中心位置で、基本はスクリーンの大きさの半分の値が入ります。
$s$はせん断係数です。理想的なカメラでは0になります。

外部パラメータと内部パラメータを組み合わせると、3次元の世界座標(x,y,z)から2次元のスクリーン座標(X,Y)に変換することができるわけです。

$$
S \begin{pmatrix}
X \\ Y \\ 1
\end{pmatrix} = K 
\begin{pmatrix}
R | t \\
\end{pmatrix}
\begin{pmatrix}
x \\ y \\ z \\ 1
\end{pmatrix}
$$

# パラメータを取得する

ViewControlを使うとカメラの位置を弄ることができる、ということは以下の記事にて話しましたが、ViewControlとピンホールカメラモデルを相互に変換するメソッドが用意されています。

https://zenn.dev/fastriver/articles/open3d-camera-view-control

[ViewControl.convert_to_pinhole_camera_parameters()](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.convert_to_pinhole_camera_parameters)と[ViewControl.conver_from_pinhole_camera_parameters()](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.convert_from_pinhole_camera_parameters)を使うことで実現できます。

```python
import open3d as o3d
import numpy as np

scale = 50

line = np.linspace(0, scale, scale+1, dtype=np.float64)

points = []
colors = []
for x in line:
    for y in line:
        for z in line:
            points.append([x,y,z])
            colors.append([x/scale, y/scale, z/scale])

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

vis = o3d.visualization.Visualizer()
vis.create_window()
vis.add_geometry(pcd)

view_control = vis.get_view_control()
pinhole_parameters = view_control.convert_to_pinhole_camera_parameters()

print(pinhole_parameters.intrinsic.intrinsic_matrix)
print(pinhole_parameters.extrinsic)

vis.run()
```

ViewControl.convert_to_pinhole_camera_parameters()を呼び出すと、[open3d.camera.PinholeCameraParameters](http://www.open3d.org/docs/release/python_api/open3d.camera.PinholeCameraParameters.html#open3d.camera.PinholeCameraParameters)オブジェクトが得られます。外部パラメータはPinholeCameraParameters.extrinsicにあり、PinholeCameraParameters.intrinsicは[open3d.camera.PinholeCameraIntrinsic](http://www.open3d.org/docs/release/python_api/open3d.camera.PinholeCameraIntrinsic.html)オブジェクトのためさらにその中のPinholeCameraIntrinsic.intrinsic_matrixが内部パラメータになります。

# カメラを特定座標に設置する

カメラを特定座標に配置したい場合は、外部パラメータ内の平行移動のパラメータを弄る方法が考えられます。

$$
\begin{pmatrix}
r_{11} & r_{12} & r_{13} & t'_1 \\
r_{21} & r_{22} & r_{23} & t'_2 \\
r_{31} & r_{32} & r_{33} & t'_3 \\
0 & 0 & 0 & 1 \\
\end{pmatrix}
$$

このように外部パラメータを変更すると、カメラ座標から見て世界座標の原点が($t'_1$,$t'_2$,$t'_3$)の位置に表示されます。

カメラ座標は下の図のようにx軸が右方向、y軸が下方向、z軸が前方向を向いています。

![](/images/open3d-camera-pinhole-2022-06-18-14-58-43.png)

pythonで実装すると以下のようになります。

```python
import open3d as o3d
import numpy as np

scale = 50

line = np.linspace(0, scale, scale+1, dtype=np.float64)

points = []
colors = []
for x in line:
    for y in line:
        for z in line:
            points.append([x,y,z])
            colors.append([x/scale, y/scale, z/scale])

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

vis = o3d.visualization.Visualizer()
vis.create_window()
vis.add_geometry(pcd)

view_control = vis.get_view_control()

pinhole_parameters = view_control.convert_to_pinhole_camera_parameters()

extrinsic = pinhole_parameters.extrinsic.copy()
extrinsic[:3,3] = np.array([-50,50,200])

pinhole_parameters.extrinsic = extrinsic

view_control.convert_from_pinhole_camera_parameters(pinhole_parameters)

vis.run()
```

![](/images/open3d-camera-pinhole-2022-06-18-15-01-11.png)

カメラの前方向200で、左に50、下に50動いた部分に世界座標の原点を置くようにしたため50x50x50の立方体の頂点$(50,50,50)_{world}$が正面に見えるように表示されました。

どうにかすれば世界座標系の特定の位置にカメラを置くこともできると思いますが、実装方法が思い浮かばなかったので割愛します。

# オブジェクトを中心に特定角度回る

中心点を変更しないままオブジェクトの周りを特定角度回したい場合には、世界座標の時点で回転させてやればよいので外部パラメータの前に回転を加えます。

$$
\begin{pmatrix}
R & t \\
0 & 1 \\
\end{pmatrix}
\begin{pmatrix}
R'
\end{pmatrix}
\begin{pmatrix}
x \\ y \\ z \\ 1
\end{pmatrix}
$$

以下は水平方向に30°回転させようとしたコードです。回転行列はScipyのRotationクラスから簡単に作ることができます。

```python
import open3d as o3d
import numpy as np
from scipy.spatial.transform import Rotation

scale = 50

line = np.linspace(0, scale, scale+1, dtype=np.float64)

points = []
colors = []
for x in line:
    for y in line:
        for z in line:
            points.append([x,y,z])
            colors.append([x/scale, y/scale, z/scale])

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

vis = o3d.visualization.Visualizer()
vis.create_window()
vis.add_geometry(pcd)

view_control = vis.get_view_control()
view_control.set_zoom(1.5)
pinhole_parameters = view_control.convert_to_pinhole_camera_parameters()

rot = np.eye(4)
rot[:3,:3] = Rotation.from_euler('y', 30, degrees=True).as_dcm()

pinhole_parameters.extrinsic = np.dot(pinhole_parameters.extrinsic, rot)

view_control.convert_from_pinhole_camera_parameters(pinhole_parameters)

vis.run()
```

![](/images/open3d-camera-pinhole-2022-06-18-15-11-33.png)

元の位置から30°回転していることがわかります。

# カメラを特定角度回す

カメラの首振りをしたい場合もあると思います。その場合はカメラ座標で回転させればよい(中心はカメラ)です。

$$
\begin{pmatrix}
R'
\end{pmatrix}
\begin{pmatrix}
R & t \\
0 & 1 \\
\end{pmatrix}
\begin{pmatrix}
x \\ y \\ z \\ 1
\end{pmatrix}
$$

```python
import open3d as o3d
import numpy as np
from scipy.spatial.transform import Rotation

scale = 50

line = np.linspace(0, scale, scale+1, dtype=np.float64)

points = []
colors = []
for x in line:
    for y in line:
        for z in line:
            points.append([x,y,z])
            colors.append([x/scale, y/scale, z/scale])

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

vis = o3d.visualization.Visualizer()
vis.create_window()
vis.add_geometry(pcd)

view_control = vis.get_view_control()
view_control.set_zoom(1.5)
pinhole_parameters = view_control.convert_to_pinhole_camera_parameters()

rot = np.eye(4)
rot[:3,:3] = Rotation.from_euler('y', 30, degrees=True).as_dcm()

pinhole_parameters.extrinsic = np.dot(rot, pinhole_parameters.extrinsic)

view_control.convert_from_pinhole_camera_parameters(pinhole_parameters)

vis.run()
```

![](/images/open3d-camera-pinhole-2022-06-18-15-13-53.png)

30°左を向きました。

# 焦点距離を設定する

内部パラメータを弄れば焦点距離を正確に設定することが可能です。$f_x$と$f_y$を好きな値に書き換えればよい($f_x=f_y$)ことになります。

$$
K = 
\begin{pmatrix}
f_x & s & c_x \\
0 & f_y & c_y \\
0 & 0 & 1 \\
\end{pmatrix}
$$

以下が変更するコードになります。焦点距離は500以上の値を指定することができました。

```python
import open3d as o3d
import numpy as np

scale = 50

line = np.linspace(0, scale, scale+1, dtype=np.float64)

points = []
colors = []
for x in line:
    for y in line:
        for z in line:
            points.append([x,y,z])
            colors.append([x/scale, y/scale, z/scale])

pcd = o3d.geometry.PointCloud()
pcd.points = o3d.utility.Vector3dVector(points)
pcd.colors = o3d.utility.Vector3dVector(colors)

vis = o3d.visualization.Visualizer()
vis.create_window()
vis.add_geometry(pcd)

view_control = vis.get_view_control()
view_control.set_zoom(1.5)
view_control.rotate(100,100)

pinhole_parameters = view_control.convert_to_pinhole_camera_parameters()

intrinsic = pinhole_parameters.intrinsic.intrinsic_matrix.copy()
intrinsic[0,0] = 1500
intrinsic[1,1] = 1500

pinhole_parameters.intrinsic.intrinsic_matrix = intrinsic

view_control.convert_from_pinhole_camera_parameters(pinhole_parameters)

vis.run()
```

![](/images/open3d-camera-pinhole-2022-06-18-15-19-19.png)

焦点距離を小さくしたほうが大きな画角となり、広く映るようになっていることがわかります。
また当然ですが、焦点距離を無限大にすることはできないためピンホールカメラモデルで平行投影表示にすることはできません。その場合はViewControlのfovを変更するようにしましょう。

https://zenn.dev/fastriver/articles/open3d-camera-view-control
