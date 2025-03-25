### 事前準備
もともとraspberry pi zero 2wを購入しましたが、wifiのスピードが遅くて、耐えられないため、
2021年に購入したraspberry pi 4Bを再び探し出して、使うことにしました。

### 構築手順
1. teslausbのイメージをsd cardに焼く、イメージは`https://github.com/marcone/teslausb`からダウンロードする。
2. `teslausb_setup_variables.conf`をsd cardのbootディレクトリに置く
3. sd cardをraspberry pi 4bに入れて、raspberry 4bをpcと繋ぐ
4. `ssh pi@teslausb.local`でraspberry pi 4bに入る(初期パスワードはraspberry)
　繋がるまで時間がかかるので、繋がるまでリトライする
6. `tail -f /boot/teslausb-headless-setup.log`で初期設定の状況を監視
7. 初期設定が終わると自動的に再起動される
8. `ssh pi@teslausb.local`でraspberry pi 4bに入る(初期パスワードはraspberry)
9. rsync転送のため、sshキーを作成してrsyncサーバーに転送
    ```
    sudo -i
    bin/remountfs_rw
    ssh-keygen #nameやpasswardなど入力する欄が出てくるが、全部無視してenterを押してください
    ssh-copy-id user@archiveserver #自身のものに変えてください
    ```
10. raspberryからパスワードなしでsshでrsyncサーバーに入れることを確認
    ```
    ssh user@archiveserver
    exit
    ```
11. raspberryのデフォルトパスワードを変更
    ```
    passwd pi
    reboot
    ```

構築手順は以下を参考に作成しました。rsyncにしたのはraspberry pi zero 2wで検証する時に、cifsにした場合、ときどき一部のファイルが同期されないことがあったからです。
https://github.com/marcone/teslausb/blob/main-dev/doc/OneStepSetup.md
https://github.com/marcone/teslausb/blob/main-dev/doc%2FSetupRSync.md

### 役に立つFAQ
Q: How do I change the TeslaUSB configuration?  
A: some (but not all) configuration settings can be changed by editing /root/teslausb_setup_variables.conf and re-running /root/bin/setup-teslausb. To make the root partition writeable so you can make changes, run /root/bin/remountfs_rw first. To edit the configuration file, use nano /root/teslausb_setup_variables.conf
```
sudo -i
/root/bin/remountfs_rw
vi teslausb_setup_variables.conf
/root/bin/setup-teslausb
reboot
```
注意すべきなのは`/root/bin/setup-teslausb`を実行したら、CAMやBOOMBOXなども再作成されるので、中身のファイルは失われます。  
なので、基本的に設定変更はイメージの焼き直しからでいいかなと思います。

Q: Do I need to manually clean up the recording drive?  
A: No. While recent recordings are not deleted immediately after archiving (so that there's always something to watch in the in-car viewer), TeslaUSB manages the space on the sd card or other storage so that it doesn't fill up.

全部のFAQは以下となります。  
https://github.com/marcone/teslausb/wiki/FAQ

### [Discord](https://discord.gg/2WgJFrFc)で開発者のかこの回答から拾ったもの
yes, it will keep taking snapshots, and deleting older snapshots when free space is less than 10G+3%.  
clipsをアーカイブしたらすぐteslausbから削除するのではなく、容量が圧迫したら削除される仕様になっている。

TeslaUSB keeps a list of files that it has previously archived, so if you delete from the server but it still exists on the Pi, it won't be archived again  


### トラブルシュート
1. sd cardを焼き直して再度sshしてエラーになる場合は、~/.ssh/known_hostsにあるteslausb.localから始まる行を消してください。
2. rsyncの接続情報を設定する前に、ちゃんとPCからパスワードなしでrsyncサーバーにファイルを転送できることをrsyncコマンドで確認したほうがいいです。1回目は大体ミスが入ります。
3. telegramのbotに通知する設定をする場合は、BotFatherからもらったtokenの先頭にbotをつけて設定ファイルに入れてください。
4. tesla apiにアクセスできず、ずっとretryするようになったら、新しいrefresh tokenを生成して、/root/teslausb_setup_variables.confに入れて、rebootしてください。teslausbを焼き直すたびに、refresh tokenは新しいものに変えた方がいいです。
5. アーカイブの時間があまりにも長くなってしまうと、車をwake状態にするためのAPI呼び出しは失敗になる場合があります。特に解決する方法はないです。

### 日本の電波法について
5GHz帯を利用したWIFI通信はW52とW53は屋外利用禁止になっています。ただし、W56は屋外利用可能です。  
したがって、teslausbと家のルーター間のWIFI通信をW56にした方が安心です。  
やり方は主にルーターのほうにあります。W56をサポートするルーターであれば、WIFI 5GHzの設定周りで100以上のチャンネルを設定できます。  
以下はW56とチャンネルの対応関係です。  
https://www.aterm.jp/function/guide4/wireless_cmx/list/iee-dual/m01_m12a.html

ルーター側の設定が終わると、もし自分のWIFIは2.4GHzと5GHzを同じSSIDにしたい場合、teslausb側の設定を修正する必要があります。  
`/etc/wpa_supplicant/wpa_supplicant.conf`にあるnetworkに以下のようにbssidの追加指定ができます。  
このbssidを指定することで、必ず5GHzのほうにアクセスするようになります。指定しないと、電波の強い2.4Ghzに接続してしまいがちです。  
network={
ssid=”XXXXXXXX”
bssid=xx:xx:xx:xx:xx:xx
psk=”your_wifi_password”
key_mgmt=WPA-PSK
}

bssidの確認ですが、MacOSの場合、optionを押しながら、右上のWIFIアイコンを左クリックすると出てきます。  
また上記設定はteslausbの初期設定の後に行う方法です。  
もし、初期設定の中で一緒に行うようにしたければ、`/boot/wpa_supplicant.conf.sample`に正しい設定を入れた上で、`wpa_supplicant.conf`にリネームすればいいです。

設定がうまくいけば、最終的に以下のようにW56で繋がっていることを確認できます。freqのところです。  
```
pi@teslausb:~ $ iw dev wlan0 link
Connected to xx:xx:xx:xx:xx:xx (on wlan0)
SSID: XXXXXXXXXX
freq: 5500
RX: 94046 bytes (631 packets)
TX: 32096 bytes (252 packets)
signal: -71 dBm
rx bitrate: 292.5 MBit/s
tx bitrate: 292.5 MBit/s

bss flags:
dtim period: 1
beacon int: 100
```

### ロック音
自分の好きなロック音を追加しました。  
CAMのルートディレクトリにそのままのファイル名で入れて、おもちゃ箱→ブームボックスに入って、ロック音をONにして、音楽をUSBにします。

### CAM_SIZEという設定の決め方について
デフォルトは40GBになります。  
teslaの仕様だと、recentclipsは常に最新の1時間しか保持しないですが、
teslausbは容量が許す限り、recentclipsをずっと保持し続けます。
保持する方法はtesla車が見えない領域に過去のrecentclipsを移動します。tesla車はその動きに気づきません。
これを理解した上で、CAM_SIZEについて考えると、256GBのSDカードでCAM_SIZEが40GBにする場合、tesla車は40GBしか見えません。
直近1時間のrecentclips、savedclips、sentryclipsは全部この40GBに保存します。
NASへのアーカイブの間隔が長くて、アーカイブする前に40GBに達してしまうと、teslausbは古いものから削除するので、一部の記録がロストすることになります。
逆にtesla車が見えない領域は216GBくらいあるので、recentclipsはたくさん残すことができます。
なので、従来のドライブレコーダーと同じような使い方がしたければ、CAM_SIZEをデフォルトの40にして、savedclipsとsentryclipsに気にしません。
何かあったら、teslausbのweb interfaceで過去のrecentclipsを調べます。tesla車が見えないところに保存しているので、tesla車では直近1時間の物しか見えないです。
もしteslaのDashCamの使い方がしたければ、CAM_SIZEをできるだけ大きくして、recentclipsは1時間以前の物は気にしません。
ただし、何か記録したときは自身の操作でsavedclipsを作ったり、セントリーモードをよく使って、sentryclipsをたくさん保存します。

私はどっちも極端だなあと思って、256GBのSDカードにおいて、CAM_SIZEを128GBにしました。