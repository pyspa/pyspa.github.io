---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "お題: パスワードマネージャなに使ってる？"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date: 2021-08-12T15:41:07Z
lastmod: 2021-08-12T15:41:07Z
featured: false
draft: false
---

なにつかってる？

- [1password](https://1password.com/jp/)
  - 1password便利だよ
  - 結構前からサブスクだよね
  - mac, winに加えて最近linux版まであるから
  - password managerのUIって、普段Browserでススッと入れてくれたり、スマホでささっと入力してくれたりであって、あのデカい管理画面なんて基本的に見ないから、あっちが残念なことになってもあんまり関係ないと思う
  - スマホのやつも快適に動くよ
  - windowsフォームアプリ使わされてきたやつらにとってはElectronがきれいすぎてビビるね
    - Electron は作る側にとってはかなり苦痛ではある
  - 1pass、ずっとローカルvault使っていて買い切りで粘っている
    - 1passずっと買い切りで使ってきたけど、そろそろfamily共有したくてサブスクに乗り換えようかな
  - うちは1passのサブスクでファミリー共有してるよ
    - こどもがスマホつかいだしたらファミリーにしよ

- [bitwarden](https://bitwarden.com/)無料だしいいですよ
  - bitwardenです
  - 同じくbitwardenです。
  - bitwardenはfreeであるし、$3.33/Monthで6人まで共有できる

- ファミリー共有って何が便利なんですか？あんまりイメージがついてなくて
  - おれも共有したいものはないな
  - 水道局のサイトとかは共有してますね
    - そう。そういう公共系のやつとか、マンションのサイトとかでアカウント共有せざるを得ないやつもあるよね…
    - マンションのサイトは、駐車場とかルームの予約とか、理事会議事録見たりとかそういうのに使う。
    - 子供のアカウント情報は夫婦で共有しておかないと、どちらかしか対応できなくなってしまう
  - 1passwordのファミリー、子供のパスワード管理を夫婦Vaultと家族Vaultで使い分けてます。便利
  - 情報共有というよりかは、複数人まとめてライセンス出来るのが嬉しい。
    - 個別に買うよりディスカウントが効くというのがメリット？
  - あと死んだ場合とか引き継げる機能とかあるといいなあ


- パスワード無料のとこに預け続けるのもなんかやだな
  - bitwardenはその辺OSSだから自分でホストしてしまえるので……。
  - パスワード預け先を自分でホスティングとか絶対やりたくないわw 罰ゲームやんって
  - むしろ自分でホストしたいんですがw
  - サービスが潰れても逃げ道がある、ぐらいの認識かなあ。積極的に運用はしたくないw
  - 人に預けるのすら相当抵抗があってお金を払って預けるなんて……。

- dropbox + [keepassxc](https://keepassxc.org/)
  - 人に預けるのに抵抗があったのでkeepassxをDropboxで共有という運用
    - [KeePassDroid](https://play.google.com/store/apps/details?id=com.android.keepass&hl=ja&gl=US) がよく出来てる。
  - だけどめんどくさくなってbitwardenに移行した

- 自前でハッシュ関数から生成する方法でやってる (マスターパスワードは記憶しておくだけ)
  - *nix 系だけですけど GPG 使って管理するやつありますよ
    - https://www.passwordstore.org/
  - うーん自作できちゃったから必要性がないっすねえ…
  - 1passいまcli版もあるよ

- めんどくさくて自作ツールでまとめて管理しちゃってる

- safari (実態としては icloud keychain? ) 使ってる
  - 次のバージョンからだいぶ賢くなるっぽいよ
  - 2faの管理もできるようになるし
  - OS標準でできるのは強そう
  - [chrome拡張もあるっぽい](https://iphone-mania.jp/news-344797/)

- 1passwordにも2faの管理機能付いたよね
  - bitwardenの2FAは無料だとOTPしか使えない。有料アカウントだとyubikey使える
  - 昨今2FAがmustなサービスも結構あって、そういうのを共有するためには2FA対応が必須
  - 2FAはAuthyに分離してるなぁ

- パスワードマネージャーの2FAの管理ってなにするん？
  - フォームsubmitしたあと2faのコードを勝手にクリップボードにコピーしてくれる
  - 2faのinputに勝手に突っ込んでくれるパターンもある

## まとめ

- 1password と bitwardenの一騎打ちだった