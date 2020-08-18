# MerossControl
MerossスマートプラグをRaspberry Piからコントロールする
Controlling Meross smart-plug from Raspberry Pi

スマートプラグと呼ばれるデバイスはたくさんありますが、そのほとんどは（全部ではないにしても）製造業者のサーバーと通信することによって、その機能を実現しています。
これにはいくつかの懸念点があります。
- 製造業者のサーバーがダウンしているときには使えない。
- インターネットに接続できなければ使えない。
- 人に知られたくない個人情報を製造業者に知られてしまうかもしれない。
There are many so-called smart-plug devices, but many (if not all) of them are communicating with manufacturer's server to perform their functions.
But that raises some concerns:
- You cannot use them when manufacturer's server is down.
- You cannot use them if you lose internet connectivity.
- The manufacturer might capture your sensitive personal information.

ですから、それらのデバイスを（製造業者のサーバーとではなく）自宅のローカルなサーバーと通信させることができれば、もっと安心できるはずです。
しかし、そんなことは可能なのでしょうか？
はい、Merossデバイスであれば、Raspberry Piと通信させることが可能です！
So, if you can use these devices by letting them communicate with your local server (not with manufacturer's server), that would make your life better.
But is that possible?
Yes, if you have a Meross device, you can make it communicate with your Raspberry Pi!

そのやり方を説明しましょう。
I will show you how to do that.

## Step 1.
必要なパッケージをインストール
Install necessary packates:
```
$ sudo apt-get update
$ sudo apt-get install mosquitto mosquitto-clients
$ sudo apt-get install ssl-cert
$ sudo apt-get install npm
```

## Step 2.
MerossデバイスはMQTT通信にTLSv1.1を利用する。
Raspberry Pi OSではデフォルトでTLSv1.2以上を要求するので、/etc/ssl/openssl.cnfの最後から2行目の`MinProtocol = TLSv1.2`を`MinProtocol = TLSv1.1`と書き換える。
Meross devices use TLSv1.1 for MQTT communication.
By default, Raspberry Pi OS requires TLSv1.2 or later, so change the penultimate line of /etc/ssl/openssl.cnf, which is `MinProtocol = TLSv1.2`, to `MinProtocol = TLSv1.1`.

TLS通信に利用する証明書を作成する（実験目的には自己署名証明書で大丈夫だが、ホームネットワーク外部と通信する場合には正式な証明書の取得を検討してほしい）。
Create a certificate for use in TLS communication (self-signed certificate is OK for experiments, but consider obtaining official certificates if you are to communicate with outside of your home network):
```
$ cd /etc/mosquitto/certs
$ sudo make-ssl-cert /usr/share/ssl-cert/ssleay.cnf mqtt_server.pem
$ sudo chown pi.pi mqtt_server.pem
```
ホスト名には`localhost`、別名には`IP:192.168.X.Y`と入力する。ここで192.168.X.YはRaspberry PiのIPアドレス。
Enter `localhost` for hostname and `IP:192.168.X.Y` for altname, where 192.168.X.Y is the IP address of your Raspberry Pi.

## Step 3.
以下の内容を/etc/mosquitto/conf.d/meross.confに書き込む。
Create the file /etc/mosquitto/conf.d/meross.conf with the content:
```
port 1883
listener 8883

capath /etc/ssl/certs

certfile /etc/mosquitto/certs/mqtt_server.pem
keyfile /etc/mosquitto/certs/mqtt_server.pem

tls_version tlsv1.1
```

あるいは、当リポジトリのmeross.confを/etc/mosquitto/conf.dにコピーしてもよい。
Or, copy meross.conf of this repository to /etc/mosquitto/conf.d:
```
$ sudo cp meross.conf /etc/mosquitto/conf.d
```

## Step 04.
mosquittoデーモンを再起動し、ログを監視しておく。
Restart the mosquitto daemon and watch the log:
```
$ sudo systemctl restart mosquitto
$ sudo tail -f /var/log/mosquitto/mosquitto.log
```

または、mosquittoデーモンを停止し、デバッグモードでコマンドラインから実行する。
Or, stop the mosquitto daemon and invoke from command line in debug mode:
```
$ sudo systemctl stop mosquitto
$ mosquitto -v -c /etc/mosquitto/conf.d/meross.conf
```

## Step 5.
別途ターミナルウィンドウを開き、bytespider氏作成のmerossツールを取得する。
Open another terminal window, and retrieve the meross tool made by bytespider:
```
$ git clone https://github.com/bytespider/Meross
```

## Step 6.
Meross/bin/srcディレクトリへ移動し、`npm install`を実行して依存するパッケージをインストールする。
Change directory to bin/src and run `npm install` to resolve dependencies:
```
$ cd Meross/bin/src
$ npm install
```

## Step 7.
ここで、Merossデバイスをコンセントに差し込む。
MerossのLEDは緑とオレンジが交互に点滅しているはず。
消灯している場合には、Merossデバイスのボタンを短く一度押す。
別の色で点灯している場合には、LEDがオレンジ色に点灯するまで（約5秒間）ボタンを押し続ける。
At this point, plug your Meross device into an AC receptacle.
The LED on the Meross device should be alternating between green and orange.
If it is unlit, give a brief press to the button on the Meross device.
If it is lighting in other colors, press and hold the button (about five seconds) until the LED lights in orange color.

## Step 8.
Raspberry Piデスクトップの右上にあるWiFiアイコンをクリックすると、`Meross_SW_XXXX`のようなAP名が表示されるはず。
そのAPに接続する。
When you click the WiFi icon in the top-right corner of the Raspberry Pi Desktop, you should see an AP name like `Meross_SW_XXXX`.
Connect to that AP.

この状態で、IPアドレス10.10.10.1にMerossデバイスが見えている。
以下のようにmerossツールを実行し、結果は後で使うのでファイルに保存しておく。
Now you can access your Meross device with the IP address 10.10.10.1.
Run the meross tool as shown below, with the resulting string saved for later use:
```
$ ./meross info --gateway 10.10.10.1 | tee ~/meross-info.log
```
以下のように表示された中で、「from:」で始まる行の「2004216982025625189748e1e91a59bc」という32桁の16進数が、このMerossデバイスのIDだ。
The 32-digit hexadecimal number "2004216982025625189748e1e91a59bc", shown in the line beginning "from:", is the ID of your Meross device.
```
Getting info about device with IP 10.10.10.1
sending payload { header:
   { method: 'GET',
     namespace: 'Appliance.System.All',
     messageId: '1' },
  payload: {} }
{ header:
   { messageId: '1',
     namespace: 'Appliance.System.All',
     method: 'GETACK',
     payloadVersion: 1,
     from: '/appliance/2004216982025625189748e1e91a59bc/publish',
     timestamp: 517,
     timestampMs: 652,
     sign: '81c8727c62e800be708dbf37c4695dff' },
  payload: { all: { system: [Object], digest: [Object] } } }
```

## Step 9.
再度merossツールを、今度はsetupモードで実行する。
Run the meross tool again with setup mode:
```
$ ./meross setup --gateway 10.10.10.1 --wifi-ssid myssid --wifi-pass mypass --mqtt 192.168.X.Y:8883 | tee ~/meross-setup.log
```
ここでmyssidはホームWiFiネットワークのSSID、mypassはWiFiパスワード、192.168.X.YはRaspberry Piの（ホームWiFiネットワーク上の）IPアドレス。
Merossデバイスはカチッと音を立てて再起動し、LEDが緑色で点滅するはず。
そうならなかった場合は、何か（おそらくWiFiパスワード）が間違っていたので、ボタンを5秒間押し続けて再度やり直す。
Where myssid is the SSID of your home WiFi network, mypass is the WiFi password and 192.168.X.Y is the IP address of your Raspberry Pi (on the home WiFi network).
You should hear a click sound and see your Meross device rebooting with the LED flashing green.
If not, something (probably WiFi password) was wrong. Press and hold the button five seconds and try again.

## Step 10.
Raspberry PiをホームWiFiネットワークに再接続する。
しばらくするとMerossデバイスがMQTTブローカー（mosquittoデーモン）に接続し、LEDが緑色に点灯するはず。
同時にmosquittoのログに`New device`と表示されるはず。おめでとう、これでMQTT接続は成功だ！
Reconnect your Raspberry Pi to your home WiFi network.
After a while, your Meross device will be connecting to the MQTT broker (mosquitto daemon) and its LED will light solid green.
At the same time, you can see `new device found` entry in the mosquitto log. Congratulations, you successfully made a MQTT connection!

## Step 11.
もうひとつターミナルウィンドウを開き、MQTTサブスクライバーを起動してみよう。
Open another terminal window, and run a MQTT subscriber there:
```
$ mosquitto_sub -d -t "/#"
```
ここで「#」はマルチレベルのワイルドカードなので、すべてのトピックにサブスクライブしていることになる。
where "#" is a multi-level wildcard character, so you are subscribing to all topics.

あるいは、TLSで接続したければ、次のようにすることもできる。
Or, if you prefer TLS connection, you can:
```
$ sudo mosquitto_sub -d -t "/#" --cafile /etc/mosquitto/certs/mqtt_server.pem
```

## Step 12.
MQTTサブスクライバーを起動した状態で、Merossデバイスのスイッチを押してみよう。
一度押すとカチッと音がしてLEDが緑色に点灯し、もう一度押すと消灯するはずだ。
同時に、MQTTサブスクライバーからは以下のように表示されるだろう。
With the MQTT subscriber running, press the switch on the Meross device.
You should hear a click sound and see the LED lighting green.
At the same time, the MQTT subscriber should emit messages like the following:
```
{"header":{"messageId":"61c2c80bc240c7531079e6c4cb994fc9","namespace":"Appliance.Control.ToggleX","method":"PUSH","payloadVersion":1,"from":"/appliance/2004214517004725189748e1e91a62d2/publish","timestamp":1594369211,"timestampMs":856,"sign":"69fd2b2ffe7b132779b3b304453e1ad5"},"payload":{"togglex":[{"channel":0,"onoff":1,"lmTime":1594369210}]}}
{"header":{"messageId":"e97cf55a7b024a23d5a9ee5fc7b1035a","namespace":"Appliance.Control.ToggleX","method":"PUSH","payloadVersion":1,"from":"/appliance/2004214517004725189748e1e91a62d2/publish","timestamp":1594369215,"timestampMs":930,"sign":"2759bd1ab683a40459dce866f91b97ea"},"payload":{"togglex":[{"channel":0,"onoff":0,"lmTime":1594369214}]}}
```
ここで大事なのは、最後のほうにある`"onoff":1`あるいは`"onoff":0`だ。
これがスイッチのオン・オフ状態を示している。
The most important part is `"onoff":1` or `"onoff":0` at the last of the message.
This shows the status of the switch, namely ON or OFF.

## Step 13.
今度はRaspberry Pi上のMQTTパブリッシャーから、MerossデバイスへMQTTメッセージを送ってみよう。
MQTTメッセージの送信（パブリッシュ）は、mosquitto_pubというコマンドを使って行える。
しかしMerossにメッセージを受け付けてもらうには、タイムスタンプや署名を付加する必要があるので、ちょっと大変だ。
そういったメッセージを自動的に組み立ててMerossデバイスへ送付する`meross_control`というシェルスクリプトを作成してあるので、ぜひ使ってみてほしい。
Next, let's send an MQTT message to the Meross device, from the MQTT publisher on your Raspberry Pi.
You can send MQTT messages using a command called mosquitto_pub.
But the Meross is somewhat meticulous, in the sense that it requires a timestamp and signature, etc.
I have written a shell script called `meross_control`, which constructs a message according to such requirement.

当リポジトリのファイル`meross_control`をエディターで開き、最初のほうにあるMEROSS_DEVICEIDを、ステップ8で求めた実際のMerossデバイスのIDに設定する。
Open the file `meross_control` in this repository with your favorite editor, and set MEROSS_DEVICEID to your Meross device ID, obtained in the Step 8.

それから、以下のコマンドを実行してみよう。カチッと音がして、MerossデバイスのLEDが緑色に点灯するはずだ。
Then, run the following command. You should hear a clicking sound and see the LED lighting green on the Meross device.
```
$ ./meross_control 1
```

また、以下のコマンドを実行するとMerossデバイスのLEDが消灯するはずだ。
And when you run the folloing command, the LED should go off.
```
$ ./meross_control 0
```

Node-REDから「exec」ノードを使ってこのコマンドを実行すれば、Merossデバイスの動作を自動化することもできる。
If you use "exec" node of Node-RED to run this command, you can automate operation of the Meross device.