---
title: "WT901BLECLにBLE接続して設定をいじる"
emoji: "🚤"
type: "tech"
topics: ["imu", "bluetooth", "python"]
published: true
---

WitMotionのWT901BLECLをBLEで接続したときに、設定をいじったり他のデータを取る方法を考えます。

BLE接続をする方法はこちら:

https://zenn.dev/fastriver/articles/wt901blecl_read_data

# 設定を送信する場所

設定の変更や指示などは、あるCharacteristicにデータを書き込むことで実現できます。
WT901BLECLの持つCharacteristic一覧は[こちら](https://zenn.dev/fastriver/articles/wt901blecl_read_data#notify%E7%94%A8%E3%81%AEuuid%E3%81%AE%E7%A2%BA%E8%AA%8D)から入手できることがわかっています。
その中でnotifyと同じServiceに含まれwriteコマンドを持つもの、つまり0000ffe9で始まるUUIDが設定用のCharacteristicです。

```
0000ffe5-0000-1000-8000-00805f9a34fb: Unknown: ["['write-without-response', 'write'],0000ffe9-0000-1000-8000-00805f9a34fb", "['notify'],0000ffe4-0000-1000-8000-00805f9a34fb"]
```

# データ送信間隔を変更する

デフォルトのセンサデータを送る間隔を変更することができます。

![](/images/wt901blecl_change_settings-2022-10-23-10-48-34.png)

設定データはビット列で構成され、接続されたBleakClientに対してwrite_gatt_char()を呼ぶことで指定のCharacteristicに書き込むことができます。
例えば送信間隔を1 Hzに変更したい場合、上に習って`FF AA 03 03 00`というビット列を用意します。PythonではByte型を使って下のように簡潔に表現可能です。

```python
await client.write_gatt_char("0000ffe9-0000-1000-8000-00805f9a34fb", b'\xFF\xAA\x03\x03\x00')
```

実際に動かすコードは以下のようになります。デフォルトでは10 Hzなので毎秒10回データが届きますが、設定変更すると毎秒1回に変化することがわかります。

```python
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
        is_connected = client.is_connected
        print(f"Connected: {is_connected}")
        await client.start_notify("0000ffe4-0000-1000-8000-00805f9a34fb", notification_handler)

        # return rateを1 Hzに設定
        await client.write_gatt_char("0000ffe9-0000-1000-8000-00805f9a34fb", b'\xFF\xAA\x03\x03\x00')

        while True:
            await asyncio.sleep(1)

loop = asyncio.get_event_loop()
loop.run_until_complete(run(mac_address, loop))
```

# キャリブレーション

IMU内のセンサー、特に地磁気センサーは環境により大きくずれるので、正確な値を得るためにはキャリブレーションが必要です。

![](/images/wt901blecl_change_settings-2022-10-23-11-08-48.png)

キャリブレーションは開始コードを書き込んでスタートし、終了コードを書き込んで終了することができます。
例えば地磁気センサーのキャリブレーションは以下のようにできます。

```python
# 地磁気キャリブレーション開始
await client.write_gatt_char("0000ffe9-0000-1000-8000-00805f9a34fb", b'\xFF\xAA\x01\x07\x00')

# 10秒間キャリブレーションする
await asyncio.sleep(10)

# キャリブレーション終了
await client.write_gatt_char("0000ffe9-0000-1000-8000-00805f9a34fb",b'\xFF\xAA\x01\x00\x00')
```

地磁気キャリブレーションは鉄や磁石から20cm以上離した状態で、3次元的に何周かIMUを回す必要があります。

# 特殊なデータを読み取る

デフォルトで送られてくるデータは、加速度・ジャイロ・角度の3つですが、レジスタの値を読み取るコマンドを使うことで他のデータにもアクセスすることができます。

![](/images/wt901blecl_change_settings-2022-10-23-11-20-25.png)

コマンドは`FF AA 27 XX 00`です。XXに読み取りたい最初のレジスタ番号を入れると、8つの連続したレジスタの値を返してくれます。
マニュアルにはMagnetic Field、Quaternion、Temperatureの読み方の記述があります。
このコマンドを書き込むと、notify用のCharacteristic(通常データと同じ)に1度だけレジスタの値を持つパケットが流れてきます。通常データとの区別は、2byte目のSignの値からできます(通常データは`55 61 ...`でレジスタデータは`55 71 ...`)。

実際に3種類のデータを読み取るコードは以下のようになります。

```python:wt901_data.py
import asyncio
from bleak import BleakClient
import numpy as np
import matplotlib.pyplot as plt

address = "F8:1B:3D:20:A9:FE" # scan_ble_devices.pyで取得したMACアドレスを指定

def notification_handler(sender, data):
    header_bit = data[0]
    assert header_bit == 0x55
    flag_bit = data[1] # 0x51 or 0x71
    if flag_bit == 0x61:
        # Default Data Packet
        decoded = [int.from_bytes(data[i:i + 2], byteorder='little', signed=True) for i in range(2, len(data), 2)]
        ax = decoded[0] / 32768.0 * 16 * 9.8
        ay = decoded[1] / 32768.0 * 16 * 9.8
        az = decoded[2] / 32768.0 * 16 * 9.8
        wx = decoded[3] / 32768.0 * 2000
        wy = decoded[4] / 32768.0 * 2000
        wz = decoded[5] / 32768.0 * 2000
        roll = decoded[6] / 32768.0 * 180
        pitch = decoded[7] / 32768.0 * 180
        yaw = decoded[8] / 32768.0 * 180
        print(f"ax:{ax}, ay:{ay}, az:{az}, wx:{wx}, wy:{wy}, wz:{wz}, roll:{roll}, pitch:{pitch}, yaw:{yaw}")
    elif flag_bit == 0x71:
        # Single Return Data Packet
        reg_l = data[2]
        reg_h = data[3]
        if reg_l == 0x3A and reg_h == 0x00:
            # Magnetic Field
            decoded = [int.from_bytes(data[i:i + 2], byteorder='little', signed=True) for i in range(4, 10, 2)]
            hx = decoded[0]
            hy = decoded[1]
            hz = decoded[2]
            print("Magnetic Field:", hx,hy,hz)
        elif reg_l == 0x51 and reg_h == 0x00:
            # Quaternion
            decoded = [int.from_bytes(data[i:i + 2], byteorder='little', signed=True) for i in range(4, 12, 2)]
            q0 = decoded[0] / 32768.0
            q1 = decoded[1] / 32768.0
            q2 = decoded[2] / 32768.0
            q3 = decoded[3] / 32768.0
            print("Quaternion:", q0,q1,q2,q3)
        elif reg_l == 0x40 and reg_h == 0x00:
            temperature = int.from_bytes(data[4:6], byteorder='little', signed=True) / 100
            print("Temperature:", temperature)
        else:
            raise Exception("Unknown data type")
    else:
        raise Exception("Unknown data type: ", data.hex())

async def run(address, loop):
    notify_id = "0000ffe4-0000-1000-8000-00805f9a34fb"
    write_id = "0000ffe9-0000-1000-8000-00805f9a34fb"

    async with BleakClient(address, loop=loop) as client:
        x = client.is_connected
        print("Connected: {0}".format(x))
        await client.start_notify(notify_id, notification_handler)
        # data rate -> 1 Hz
        await client.write_gatt_char(write_id, b'\xFF\xAA\x03\x03\x00')

        while True:
            await asyncio.sleep(0.2)
            # Magnetic Field
            await client.write_gatt_char(write_id,b'\xFF\xAA\x27\x3A\x00')
            await asyncio.sleep(0.2)
            # Quaternion
            await client.write_gatt_char(write_id,b'\xFF\xAA\x27\x51\x00')
            await asyncio.sleep(0.2)
            # Temperature
            await client.write_gatt_char(write_id,b'\xFF\xAA\x27\x40\x00')

loop = asyncio.get_event_loop()
loop.run_until_complete(run(address, loop))
```

```
> python wt901_data.py
Connected: True
Magnetic Field: 235 -540 -7
ax:-0.35888671875, ay:0.30146484375000004, az:9.771289062500001, wx:0.0, wy:0.0, wz:0.0, roll:1.7138671875, pitch:2.1148681640625, yaw:156.26953125
Quaternion: 0.20574951171875 -0.014984130859375 0.018463134765625 0.978271484375
Temperature: 21.13
Magnetic Field: 234 -540 -5
Quaternion: 0.20574951171875 -0.01483154296875 0.01849365234375 0.978271484375
ax:-0.3541015625, ay:0.2966796875, az:9.766503906250001, wx:0.0, wy:0.0, wz:0.0, roll:1.724853515625, pitch:2.098388671875, yaw:156.26953125
Temperature: 21.19
Quaternion: 0.20574951171875 -0.0147705078125 0.01849365234375 0.978271484375
Magnetic Field: 234 -542 -5
```

データシートを見ると、レジスタ番号とデータの対応表が載っているため他のデータでも同様にして入手できると考えられます。

# 参考

https://drive.google.com/file/d/1vpPqaVdSLzlPhd9YFE3bjj0f75XtiSyO/view
