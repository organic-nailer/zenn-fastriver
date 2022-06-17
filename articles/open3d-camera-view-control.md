---
title: "Open3Dのカメラの取り扱い ViewControl編"
emoji: "📷"
type: "tech"
topics: ["open3d", "python"]
published: true
---

Open3Dで3Dをどこから見るか、というカメラの取り扱いについて調べたのでまとめました。

# カメラとは？

![](/images/open3d-camera-view-control-2022-06-17-14-58-04.png)

> https://jp.mathworks.com/help/vision/ug/camera-calibration.html より

Open3Dではデータが3次元空間に存在しますが、表示する画面は2次元平面です。3次元空間の物体を2次元平面上に射影するのがカメラの役割になります。

![](/images/open3d-camera-view-control-2022-06-17-15-10-32.png)

Open3Dでカメラを制御する一つの方法として、ViewControlオブジェクトを使うというものがあります。

http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html

ViewControl内では、上の図のような5つのパラメータを設定することでカメラの位置と向きを決定しています。

## lookat

カメラが3次元上のどの点に照準を合わせるかを設定します。点を指定するのでx,y,zの3つの数字が必要です。

## front

lookatの点から見てどの向きにカメラが存在するか、というベクトルです。カメラの向いている方向とは逆向きなので注意が必要になります。こちらもベクトルなのでx,y,zの3つの数字が必要です。

## up

カメラの上方向がどちらに向いているかというベクトルです。実際にはfrontと直交している必要はなく、front軸を固定したときにカメラの上方向とこのベクトルが一番近くなるように調整されます。こちらもベクトルなのでx,y,zの3つの数字が必要です。

## zoom

lookatの点とカメラの距離を指定するパラメータです。距離なのでスカラー値をとります。ただし指定した数字が何を示しているかは不明です。

lookat・front・up・zoomの4つのパラメータでカメラの位置と向きが決定されます。それぞれのパラメータは個別に設定することが可能で、1つのパラメータを変更したときに他のパラメータは変化しないようにカメラが移動します。例えばupのみを変更した場合、カメラは同じ位置でカメラのz軸を中心に回転するのみです。

## fov

fovはField of Viewの略で、日本語だと視野角や画角などと言います。カメラがどの程度広く撮影するかという指標です。

# 相対的にカメラを移動する

初期状態のカメラに対し、マウス操作時と同じように回転や拡大縮小を行うメソッドがいくつか用意されています。

- [rotate(x,y,xo=0.0,yo=0.0)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.rotate)
  - xとyで指定した向きにViewを回転する
  - xに正の数を指定すると左に、yに正の数を指定すると上に回転する
  - マウスでドラッグした時と同じ挙動になっている
  - xoとyoはどのように使うのか不明
- [camera_local_rotate(x,y,xo=0.0,yo=0.0)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.camera_local_rotate)
  - xとyで指定した向きにカメラが回転する
  - xに正の数を指定すると右を、yに正の数を指定すると下を向く
  - 首を回したような挙動
  - xoとyoはどのように使うのか不明
- [translate(x,y,xo=0.0,yo=0.0)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.translate)
  - xとyで指定した方向に移動する
  - xに正の数を指定するとカメラが左に、yに正の数を指定するとカメラが上に移動する
  - ctrlを押しながらドラッグした時と同じ挙動
  - xoとyoはどのように使うのか不明
- [scale(scale)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.scale)
  - 現在の位置から近づくか遠ざかるかを指定する
  - scaleに正の数を指定すると遠ざかる
  - ホイール回した時と同じ挙動

以下のコードから実際の使い方を見ることができます。

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

def rotate_left_view(vis):
    """Viewを左に回転する"""
    ctr = vis.get_view_control()
    ctr.rotate(10.0, 0.0)
    return False

def rotate_right_view(vis):
    """Viewを右に回転する"""
    ctr = vis.get_view_control()
    ctr.rotate(-10.0, 0.0)
    return False

def pan_up_view(vis):
    """カメラを上に向ける"""
    ctr = vis.get_view_control()
    ctr.camera_local_rotate(0.0, -10.0)
    return False

def pan_down_view(vis):
    """カメラを下に向ける"""
    ctr = vis.get_view_control()
    ctr.camera_local_rotate(0.0, 10.0)
    return False

def scale_up_view(vis):
    """ズームアウトする"""
    ctr = vis.get_view_control()
    ctr.scale(1)
    return False

def scale_down_view(vis):
    """ズームインする"""
    ctr = vis.get_view_control()
    ctr.scale(-1)
    return False

def translate_left_view(vis):
    """カメラを左に移動する"""
    ctr = vis.get_view_control()
    ctr.translate(10.0, 0.0)
    return False

def translate_right_view(vis):
    """カメラを右に移動する"""
    ctr = vis.get_view_control()
    ctr.translate(-10.0, 0.0)
    return False

def translate_up_view(vis):
    """カメラを上に移動する"""
    ctr = vis.get_view_control()
    ctr.translate(0.0, 10.0)
    return False

def translate_down_view(vis):
    """カメラを下に移動する"""
    ctr = vis.get_view_control()
    ctr.translate(0.0, -10.0)
    return False

key_callback = {}
key_callback[ord("F")] = rotate_left_view
key_callback[ord("H")] = rotate_right_view
key_callback[ord("T")] = pan_up_view
key_callback[ord("G")] = pan_down_view
key_callback[ord("Z")] = scale_up_view
key_callback[ord("X")] = scale_down_view
key_callback[ord("A")] = translate_left_view
key_callback[ord("D")] = translate_right_view
key_callback[ord("W")] = translate_up_view
key_callback[ord("S")] = translate_down_view

o3d.visualization.draw_geometries_with_key_callbacks([pcd],key_callback)
```

まず準備として、50x50x50の立方体を埋める点群にXYZに対応したRGBの色をつけたものを用意しています。

最後の行ではdraw_geometries_with_key_callbacks()を使っています。これは画面にgeometryを表示するだけでなくキー入力のコールバックを設定することが可能です。

http://www.open3d.org/docs/release/python_api/open3d.visualization.draw_geometries_with_key_callbacks.html

コールバックはopen3d.visualization.Visualizerを引数にとり、boolを返します。
Visualizer.get_view_control()から現在の表示系のViewControlオブジェクトにアクセスできます。

# FOVを設定する

現在の画角設定にアクセスする場合は、[get_field_of_view()](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.get_field_of_view)が、画角を変更したい場合は[change_field_of_view()](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.change_field_of_view)が利用できます。

FOVについては[公式のチュートリアル](http://www.open3d.org/docs/latest/tutorial/visualization/customized_visualization.html#change-field-of-view)に説明があるのですが、微妙に間違ったことが書いてあるようです。

FOVは5～90までの値をとり、数値が大きいほど広い画角となっています。しかし表示を見ると数値は特に角度(degree)を示している訳ではないように思います。
デフォルトの値は60です。

change_field_of_view()は現在の値から相対的に画角を設定します。stepは実数をとり、変更後のfovは

$$
FOV_{new} = FOV_{prev} + step \cdot 5
$$

のようになります。範囲に収まりきらない場合は最小値ないしは最大値に設定されます。

また$FOV=5$に設定すると平行投影モードになり、CADのように立方体の各辺が平行に表示されるようになります。

![](/images/open3d-camera-view-control-2022-06-17-18-31-11.png)

# カメラを好きな位置に配置する

それぞれのパラメータを設定することも可能です。1つのパラメータを変更した場合でも、他のパラメータの値は変わらないようにうまく移動します。

- [set_lookat(lookat)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.set_lookat)
  - lookatは3行1列のndarray
  - `[[x],[y],[z]]`
- [set_front(front)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.set_front)
  - frontは3行1列のndarray
- [set_up(up)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.set_up)
  - upは3行1列のndarray
- [set_zoom(zoom)](http://www.open3d.org/docs/release/python_api/open3d.visualization.ViewControl.html#open3d.visualization.ViewControl.set_zoom)
  - zoomは実数
  - 値を大きくするほど距離が離れる(値が何なのかは不明)

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

def look_at_view(vis):
    ctr = vis.get_view_control()
    ctr.set_lookat(np.array([0, 0, 0]))
    return True

def front_view(vis):
    ctr = vis.get_view_control()
    ctr.set_front(np.array([0, 0, 1]))
    return True

def up_view(vis):
    ctr = vis.get_view_control()
    ctr.set_up(np.array([0, 1, 0]))
    return True

def zoom_view(vis):
    ctr = vis.get_view_control()
    ctr.set_zoom(1)
    return True

key_callback = {}
key_callback[ord("L")] = look_at_view
key_callback[ord("F")] = front_view
key_callback[ord("U")] = up_view
key_callback[ord("Z")] = zoom_view

o3d.visualization.draw_geometries_with_key_callbacks([pcd],key_callback)
```

しかし、その場でカメラを30°回したいなどの一部の表現はViewControlでは難しいという問題があります。その場合は、ピンホールカメラモデルを使ってカメラの位置を表現する手法が別に存在しているのでそちらを使うことができます。


