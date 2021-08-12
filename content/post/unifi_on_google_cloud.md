---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Unifi Controller on Google Cloud"
subtitle: ""
summary: ""
authors: ['shirou']
tags: ['unifi', 'gcp']
categories: []
date: 2021-08-11T12:40:45Z
lastmod: 2021-08-11T12:40:45Z
featured: false
draft: false
---
自宅ではUniFiでネットワークを構築しています。狭い家ですし、VPNも貼ってないしセグメントを分けたりもしてないので、全く無意味で完全に趣味です。

<!--more-->

UniFiネットワークでは、UniFi controllerというソフトウェアを使って設定します。これを常時稼働しておけば外から状態を監視したり、設定したりできるのです。

が、家の中だけだし、そもそもここ最近は常時家にいるのでUniFi controllerは自分のPC上で必要なときだけ立ち上げていました。それで十分なのですが、新しいPCに移行しようとした時に失敗してデバイスをファクトリーリセットするしかなくなったので、これを機会にクラウド上で動かすようにしようと思ったわけです。

ちなみに失敗したということをより正確に言うと、[以下のような手順](https://help.ui.com/hc/en-us/articles/115002869188-UniFi-Migrating-sites-with-Site-Export-Wizard)で実行したのですが、

1. 古いcontrollerでバックアップを取る
2. 新しいcontrollerでバックアップからリストア
3. 古いcontrollerからデバイスの紐付けを新しいcontrollerに移行指示
4. 新しいcontrollerでログインして移行確認

ところが、4で新しいcontrollerでログインしようとすると、本来は移行されているはずのアカウント情報がなく(mongodbの中を覗いて確認した)、無理やり追加してもDB上で不整合があるらしくPermission Deniedと言われるようになりました。こうなるともうデバイスをファクトリーリセットするしかできなくなります。

# Google Cloud (以下GCP)

GCPでは米国内リージョンのE2-microであれば無料枠があるのでこれを使うことにします。米国内リージョンの中で一番日本に近そうなus-west1(オレゴン)を選択します。
新しいプロジェクトを作り、E2-micro(2 vCPU, 1GB メモリ)でインスタンスを立ち上げます。

unifi controllerのDocker imageもあるのでコンテナで立ち上げても良いのですが、ホスト側でもいろいろやりたいことがあるかもしれないので今のところは普通のインスタンスとして立ち上げます。

## UniFiのインストール

ansible galaxyにあるのでさっくりとインストールできると思ったら、

- unifiはmongodbを使っている
- mongodbの4未満じゃないといけない
- debian 10は通常では4系統をインストールしてしまう

という依存性地獄がありました。ということで、mongodb 3.6.23を指定してインストールします。[参考](https://itblog.webdigg.org/425-how-to-install-unifi-controller-on-debian-10-buster/)

```
- name: Install UniFi controller in Google Cloud instance
  hosts: all
  become: yes
  tasks:
    - name: install mongodb 3.6 apt key
      apt_key:
        keyserver: "hkp://keyserver.ubuntu.com:80"
        id: "58712A2291FA4AD5"
    - name: install mongodb apt repository
      apt_repository:
        repo: deb [arch=amd64] https://repo.mongodb.org/apt/debian stretch/mongodb-org/3.6 main
    - name: install ubnt apt repository
      apt_repository:
        repo: deb [trusted=yes arch=amd64] https://apt.lecomte.at/repacks/debian/ buster ubiquiti
    - name: install java and unifi controller
      apt:
        pkg:
          - openjdk-8-jre-headless
          - unifi
```

その後、ファイアウォールルールを設定し、以下のポートを開ければOKです。

- TCP: 8080(通知用), 8443(UI用)
- UDP: 3478 (STUN用)

なお、UniFi controllerに8443でアクセスしても、ui.comのアカウントでログインできないかもしれません。その場合、 `update-ca-certificates -f` をしてunifi serviceを再起動してみてください。

## Dynamic DNS

固定IPを契約しても良いのですが、せっかくなので[mydns.jp](https://mydns.jp)を使ってDynamic DNSで名前解決するようにします。詳細は省略しますが、以下のserviceとtimerを定義して一日一回起動するようにします。ansible的にはtemplatesモジュールを使うといいですね。

```
[Unit]
Description=mydns update

[Service]
Environment=MASTER_ID={{mydns__master_id}}
Environment=PASSWORD={{mydns__password}}
Type=oneshot
ExecStart=/usr/bin/sh -c 'curl -q -u ${MASTER_ID}:${PASSWORD} https://www.mydns.jp/login.html'

[Install]
WantedBy=multi-user.target
```

# Device登録

## factory reset

前述の通りmigrateに失敗している状態なので、device自体はどこかに登録されています。これを消すには[factory reset](https://help.ui.com/hc/en-us/articles/205143490-UniFi-How-to-Reset-Devices-to-Factory-Defaults)をカマスしかありません。

USG(Security Gateway)にsshで入り、 `mca-ctrl -t dump-cfg > mca-config.backup` とバックアップを取っておきます。`show configration` の方が良いのかもしれません。

その後、

```
sudo syswrapper.sh restore-default
```

してファクトリーリセットします。

## デバイス登録

しばらく経つと再起動が終わるので、 `user=ubnt/password=ubnt` のデフォルトユーザーでsshで入ります。その後、以下のコマンドを叩きます。(URLは適宜変更してください)

```
set-inform http://unifi.foo.mydns.jp:8080/inform
```

これにより、GCP上に登録したUniFi controller上に「追加保留中」としてUSGが表示されるはずです。あとは「追加」をすれば完了です！

Switch等のconsoleがないものは後ろにあるresetボタンを5秒以上押してリセットします。


これで完了です。

オレオレ証明書で動いているので、Let's encryptを入れたりすると良いかもしれませんが、まあそれはまた今度ということで。


UniFi controllerからはこんな感じでトラフィックが見れます。たのしい！

![トラフィック](../../images/unifi_controller.png)