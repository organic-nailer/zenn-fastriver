---
title: "Flutter WebをLAN内の他のデバイスでデバッグする"
emoji: "🧭"
type: "tech"
topics: ["flutter", "web"]
published: true
---

Flutter Webを開発したときに、開発機以外でも動かしたい。
スマホや他のPCでも動かしたい。実機で。

# 解法

https://stackoverflow.com/questions/56967885/how-to-change-the-default-web-server-ip127-0-0-1-of-flutter-web-app

`flutter run`するときに、hostnameとportを以下のように指定すると、外部からもアクセスできるようになります。

```
> flutter run -d web-server --web-hostname=0.0.0.0 --web-port=50505
```

`0.0.0.0`の部分には、開発機のローカルIPを入れます。`ipconfig`などのコマンドから調べてください。

最初に実行すると、Windowsだとfirewallの設定が出てくるので、許可します。

![](/images/flutter-web-debug-lan-2022-11-30-17-41-46.png)

Web Serverが立ち上がったら、同じLAN内のデバイスから`http://0.0.0.0:50505`にアクセスしましょう。

```
> flutter run -d web-server --web-hostname=192.168.1.9 --web-port=50505
Launching lib\main.dart on Web Server in debug mode...
Waiting for connection from debug service on Web Server...         20.8s
lib\main.dart is being served at http://192.168.1.9:50505
The web-server device requires the Dart Debug Chrome extension for debugging. Consider using the Chrome or Edge devices
for an improved development workflow.

🔥  To hot restart changes while running, press "r" or "R".
For a more detailed help message, press "h". To quit, press "q".
```

![](/images/flutter-web-debug-lan-2022-11-30-17-52-04.png)

スマホのブラウザから入れました！

> hot restartは効かない
