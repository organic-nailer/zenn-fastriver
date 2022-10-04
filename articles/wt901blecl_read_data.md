---
title: "WT901BLECLにBLE接続してデータを読み取る"
emoji: "🚣"
type: "tech"
topics: ["imu", "bluetooth", "python"]
published: true
---

![](/images/wt901blecl_read_data-2022-10-04-18-18-32.png)

https://www.wit-motion.com/9-axis/witmotion-bluetooth-ble-5-0.html

WT901BLECLは中国WitMotionの開発する安価なBluetooth Low Energy(BLE)対応IMUです。

Windows11とPythonを使ってセンサの値を読み取れるようにしてみましょう。

# PCと接続する

WT901の電源を入れると自動でペアリングモードになるので、PCの設定の[Bluetooth & Devices] > [Add Device]からペアリングします。

![](/images/wt901blecl-2022-10-04-17-16-36.png)

「WT901BLE67」が見つかると思うのでこれを選びます。

![](/images/wt901blecl-2022-10-04-17-17-38.png)

その後デバイス一覧にPairedと表示されていれば接続ができています。

![](/images/wt901blecl-2022-10-04-17-19-08.png)

# ライブラリのインストール

WindowsでBLEを扱うには、[Bleak](https://github.com/hbldh/bleak)が利用できます。

BleakのAPI docs:

https://bleak.readthedocs.io/en/latest/

:::message
PythonのBLEライブラリは他にも[PyBluez](https://github.com/pybluez/pybluez)や[bluepy](https://github.com/IanHarvey/bluepy)がありますが、いずれもWindowsに対応していません。
:::

Bleakはpipからインストールできます。

```
> pip install bleak
```

# MACアドレスの確認

Bleakでクライアントと通信するためにはクライアントのMACアドレスが必要になるため、Pythonで接続しているクライアント一覧を取得します。

```python:scan_ble_devices.py
import asyncio
from bleak import BleakScanner

async def run():
    devices = await BleakScanner.discover()
    for d in devices:
        print(f"address: {d.address}, name: {d.name}, uuid: {d.metadata['uuids']}")

loop = asyncio.get_event_loop()
loop.run_until_complete(run())
```

これを実行すると現在PCに接続しているBLEデバイスの一覧が表示されます。IMUが接続されていればnameが「WT901BLE67」の行があるはずです。その行のアドレス、今回は「EA:78:B5:4D:E3:21」を控えておきましょう。

```
> python scan_ble_devices.py
address: C7:64:C1:C7:64:C2, name: WM2, uuid: ['0000fd81-0000-1000-8000-00805f9b34fb']
address: D6:C4:38:A6:EA:C7, name: None, uuid: ['cba20d00-224d-11e6-9fb8-0002a5d5c51b']
address: EA:78:B5:4D:E3:21, name: WT901BLE67, uuid: ['0000ffe5-0000-1000-8000-00805f9a34fb']
address: D2:2E:9D:D5:BC:05, name: None, uuid: ['cba20d00-224d-11e6-9fb8-0002a5d5c51b']
...
```

:::message
恐らくMACアドレスはIMUごとに固有の値が割り振られています。
:::

# notify用のUUIDの確認

BLEではCharacteristicという名前で1つのデバイスにいくつかの通信チャンネルを持っています。BLEの構造については以下の記事がわかりやすいです。

https://monomonotech.jp/kurage/webbluetooth/ble_guide.html

それぞれのCharacteristicはユニークなアドレスとしてUUIDが振られており、UUIDを指定して通信を行うことができます。
今回探すのはセンサデータが送られてくるnotify用のCharacteristicです。

```python:get_device_uuids.py
import asyncio
from bleak import BleakClient

address = "EA:78:B5:4D:E3:21" # scan_ble_devices.pyで取得したMACアドレスを指定

async def run(address, loop):
    async with BleakClient(address, loop=loop) as client:
        x = client.is_connected
        print("Connected: {0}".format(x))
        for service in client.services:
            print(f"{service.uuid}: {service.description}: {[f'{c.properties},{c.uuid}' for c in service.characteristics]}")

loop = asyncio.get_event_loop()
loop.run_until_complete(run(address, loop))
```

Characteristicの情報を得るため、まずは先程調べたMACアドレスを使ってBleakClientというオブジェクトを作ります。BleakClientはasyncioに対応しているのでasync withを使っています。
次にBleakClient.is_connectedで接続状態を確認しています。
複数のCharacteristicを格納するフォルダのようなものとしてServiceがあります。Service一覧はBleakClient.servicesから得られます。Characteristicはservice.characteristicsにlistで格納されています。

上のコードを実行すると、クライアントの持つCharacteristicのUUIDが一覧で表示されます。

```
> python get_device_uuids.py
Connected: True
00001800-0000-1000-8000-00805f9b34fb: Generic Access Profile: ["['read', 'write'],00002a00-0000-1000-8000-00805f9b34fb", "['read'],00002a01-0000-1000-8000-00805f9b34fb", "['read'],00002a04-0000-1000-8000
-00805f9b34fb", "['read'],00002aa6-0000-1000-8000-00805f9b34fb"]
00001801-0000-1000-8000-00805f9b34fb: Generic Attribute Profile: []
0000ffe5-0000-1000-8000-00805f9a34fb: Unknown: ["['write-without-response', 'write'],0000ffe9-0000-1000-8000-00805f9a34fb", "['notify'],0000ffe4-0000-1000-8000-00805f9a34fb"]
```

上2つはセンサの値の通信とは関係なさそうなので一旦無視します。一番下のnotifyというプロパティ名を持つCharacteristicのUUID(0000ffe4から始まるもの)が知りたかったものになります。

:::message
BLEのUUIDは全ての同じ型のセンサと共通なので、調べず決め打ちで問題ないと思います
:::

# 値を読み取る

さて、これで必要な情報が揃ったため実際にセンサの値を読んでみましょう。

```python:read_values.py
import asyncio
from bleak import BleakClient

mac_address = "EA:78:B5:4D:E3:21"

def notification_handler(sender, data: bytearray):
    decoded = [int.from_bytes(data[i:i+2], byteorder='little', signed=True) for i in range(2, len(data), 2)]
    ax    = decoded[0] / 32768.0 * 16 * 9.8
    ay    = decoded[1] / 32768.0 * 16 * 9.8
    az    = decoded[2] / 32768.0 * 16 * 9.8
    wx    = decoded[3] / 32768.0 * 2000
    wy    = decoded[4] / 32768.0 * 2000
    wz    = decoded[5] / 32768.0 * 2000
    roll  = decoded[6] / 32768.0 * 180
    pitch = decoded[7] / 32768.0 * 180
    yaw   = decoded[8] / 32768.0 * 180
    print(f"ax: {ax:.3f}, ay: {ay:.3f}, az: {az:.3f}, wx: {wx:.3f}, wy: {wy:.3f}, wz: {wz:.3f}, roll: {roll:.3f}, pitch: {pitch:.3f}, yaw: {yaw:.3f}")

async def run(address, loop):
    async with BleakClient(address, loop=loop) as client:
        x = await client.is_connected()
        print("Connected: {0}".format(x))
        await client.start_notify("0000ffe4-0000-1000-8000-00805f9a34fb", notification_handler)

        while True:
            await asyncio.sleep(1)

loop = asyncio.get_event_loop()
loop.run_until_complete(run(mac_address, loop))
```

まずは先程と同様にMACアドレスからBleakClientを作成します。そしてCharacteristicからの通知を受け取るため、BleakClient.start_notify()を利用します。

https://bleak.readthedocs.io/en/latest/api/client.html#bleak.BleakClient.start_notify

引数の1つ目に先程調べたUUID、2つ目にcallbackを設定します。

プログラムの実行が終わると通知も止まってしまうため、後ろで無限ループを挟んで終わらないようにしています。

## バイト列をパースする

センサのデータはcallbackの第2引数にbytearrayの形式で流れてきます。データの中身は公式のDataSheetに記載があります。

https://drive.google.com/file/d/1vpPqaVdSLzlPhd9YFE3bjj0f75XtiSyO/view

![](/images/wt901blecl-2022-10-04-18-09-22.png)

![](/images/wt901blecl-2022-10-04-18-09-58.png)

全体の長さは20byteで、最初の2byteはヘッダとフラグ、他は2byteずつ9種類のセンサ値が入っています。形式はsigned shortです。

上のコードを実行すると0.1秒ごとにセンサの値を読み取ることができていることが確認できます。

```
> python read_value.py 
Connected: True
ax: -0.158, ay: 0.096, az: 10.145, wx: 0.000, wy: -0.061, wz: -1.099, roll: 0.286, pitch: 0.335, yaw: -77.536
ax: -0.096, ay: 0.062, az: 10.154, wx: -0.122, wy: -0.366, wz: -5.920, roll: 0.198, pitch: 0.170, yaw: -75.844
ax: -0.110, ay: 0.100, az: 10.154, wx: 0.793, wy: 1.343, wz: -5.005, roll: 0.308, pitch: 0.401, yaw: -74.366
ax: -0.172, ay: 0.091, az: 10.154, wx: -0.366, wy: -0.366, wz: -3.662, roll: 0.363, pitch: 0.478, yaw: -73.158
ax: -0.110, ay: 0.067, az: 10.164, wx: 0.000, wy: -0.183, wz: -0.061, roll: 0.324, pitch: 0.439, yaw: -71.417
...
```

# 利用例

https://github.com/Keio-CSG/witmotion_visualizer

@[tweet](https://twitter.com/Fastriver_org/status/1575664578259980288)

# 参考

https://atatat.hatenablog.com/entry/2020/07/09/003000
