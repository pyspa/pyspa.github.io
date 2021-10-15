---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Unifi Controllerその後 2021年10月バージョン"
subtitle: ""
summary: ""
authors: ['shirou']
tags: ['unifi', 'gcp']
categories: []
date: 2021-10-11T12:40:45Z
lastmod: 2021-10-11T12:40:45Z
featured: false
draft: false
---
[前回](https://blog.pyspa.org/post/unifi_on_google_cloud/) からUniFi ContollerをGoogle Cloudで動かしています。その続報です。

- Preemptible化
- Let's Encrypt化
- UniFiでのコマンド集

# Preemptible化

Google Cloud上に建てたはいいのですが、無料のe2-microだとしばらくするとJavaがメモリを食いつぶしてノードまるごと死ぬ、ということが分かってきました。

しょうがないので、一個上のe2-smallを使おうかと思いましたが、お金がかかる。少しかかるのはしょうがないが、なるべく少なくしたい。ということで、e2-smallの ["Preemptible VM"](https://cloud.google.com/compute/docs/instances/preemptible) を使うことにしました。

Preemptible は最高24時間しか使えず、たびたび再起動します。その代わり激安で、e2-smallで$3/月で使えます。

ストレージはmongodbで多少データが欠けても全然問題ないのですが、再起動のたびにIPアドレスが変わります。そうなるとmydnsに登録してあるIPアドレスが
使えなくなってしまうので、以下のようにmydnsのtimerをセットしました。

```
[Timer]
OnBootSec=3min   # 起動後3分後
OnUnitActiveSec=6h  # 起動している限り6時間ごと
```

これで再起動してもmydnsでのIPアドレスが更新されるので、問題なくなりました。

(追記: mydnsもしょっちゅう更新が失敗するので、結局静的IPアドレスを用意しました)


# Let's Encrypt化

後述しますが、オレオレ証明書だと困ったことが分かったので[この記事](https://lazyadmin.nl/home-network/unifi-controller-ssl-certificate/)を参考に、UniFi controllerをLet's Encrypt化します。

といっても、この記事の実態は[この人のGist](https://github.com/stevejenkins/unifi-linux-utils/blob/master/unifi_ssl_import.sh) です。


## 1. certbotで証明書作成

事前にLet's encryptから検証としてhttpでアクセスしてくるので、80番を許可しておきます。

```
sudo apt install certbot
sudo certbot certonly --standalone -d unifi.alphawind.mydns.jp
```

## 2. gistからツールを取得、設定

その後、上述のgistを取得し、設定を変更します。 

- `UNIFI_HOSTNAME`
- `UNIFI_DIR` のあたりをuncomment
- `PRIV_KEY` のあたりを設定

気をつけるべき点としては、 `password` の設定は変更しない、ということです。これはunificontrollerのJavaが持つcert用なので、特に変更する必要はありません。というか変更してしまうと証明書が入れられなくなってしまいます。

## 3. ツール実行

```
$ sudo bash unifi_ssl_import.sh
Starting UniFi Controller SSL Import...
Running in Let's Encrypt Mode...
Inspecting current SSL certificate...
Updated SSL certificate available. Proceeding with import...
....
```

途中unifi controllerの再起動が入るのでちょっと時間かかります。

最後に `Done` と出れば成功です。

## 4. certbotをsystemdに登録

こんな感じのserviceを登録して、一週間に一回とかで起動させてあげれば良いです。

```
[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --quiet --agree-tos --post-hook "service unifi restart"
```