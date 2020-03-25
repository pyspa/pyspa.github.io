---
title: "pyspaブログを構成する要素技術"
date: 2020-03-22T13:13:06+09:00
tags: ["Hugo", "GitHub", "Cloud Build", "Firebase"]
authors: ["taichi"]
---

皆さんはじめまして。pyspaの方からやってきました太一です。

突然ですけども、ブログを継続的に書き続けるというのは、かなり大変なことですよね。

そこで、複数人で単一のブログを運営すれば、もう少し効率的かつ継続的にブログを書けるようになる筈だとpyspaでは考えました。

今回はこのブログにおける最初のエントリとして、このブログを構成する要素技術を説明していきます。

あわせて、他のpyspaメンバーがエントリを記述する際のサンプルとなることを意図してこのエントリを書いています。

<!--more-->

<!-- TOC -->

- [1. はじめに](#1-はじめに)
- [2. Hugo](#2-hugo)
- [3. GitHub](#3-github)
- [4. Cloud Build](#4-cloud-build)
  - [4.1. GitHub連携における注意点](#41-github連携における注意点)
    - [4.1.1. submoduleの対応](#411-submoduleの対応)
    - [4.1.2. 手動ビルドとイベントトリガビルドの差分解消](#412-手動ビルドとイベントトリガビルドの差分解消)
  - [4.2. ビルド中に使うツールをコンテナ化する](#42-ビルド中に使うツールをコンテナ化する)
  - [4.3. Firebaseのデプロイにおけるサービスアカウントへの権限付与](#43-firebaseのデプロイにおけるサービスアカウントへの権限付与)
  - [4.4. Cloud Buildに対する要望](#44-cloud-buildに対する要望)
    - [4.4.1. ビルド構成ファイルのsecretsセクションでSecret Managerをサポートして欲しい](#441-ビルド構成ファイルのsecretsセクションでsecret-managerをサポートして欲しい)
    - [4.4.2. 定期実行トリガーをもっと簡単に構成したい](#442-定期実行トリガーをもっと簡単に構成したい)
    - [4.4.3. ステップに条件節を記述できるようにして欲しい](#443-ステップに条件節を記述できるようにして欲しい)
    - [4.4.4. GitHubとの連携においてビルドトリガーの名前が分かるようにして欲しい](#444-githubとの連携においてビルドトリガーの名前が分かるようにして欲しい)
- [5. Firebase Hosting](#5-firebase-hosting)
  - [5.1. HTTPヘッダの付与](#51-httpヘッダの付与)
- [6. まとめ](#6-まとめ)
  - [6.1. ToCに関する補足](#61-tocに関する補足)

<!-- /TOC -->

# 1. はじめに

ざっくり言うと、pyspaブログは、Markdownでエントリを書いたら、GitHubにPRしてマージされるとCloud Build上でHugoが動いて、その結果をFirebase Hostingにデプロイしています。

{{<figure src="/img/how-to-make-this-site/pyspablog.svg" title="全体構成" >}}

このエントリでは、単にマニュアルを読むだけでは分かり辛かったことや、マニュアルには書いてない部分について説明していきます。

# 2. Hugo

サーバプロセス上で動的にコンテンツを生成して返さず、事前に全てをHTMLファイルとして書き出すstatic site generatorと総称されるツールがあります。

僕がそういう機能があるツールを初めて使ったのは恐らく2000年代前半に流行ったMovable Typeなんだろうけど、もう完全に忘れてしまいました。

その後、2008年くらいにJekyllがブームになってstatic site generatorというカテゴリが成立しました。

それまでは、コンテンツの静的ファイル化はパフォーマンス改善のためのオプションでした。

しかし、Jekyllはその機能だけに絞ってアプリケーションとしての動作を単純化したところが、白眉だったと考えています。

だから、今のstatic site generatorは、大体がJekyllの影響を受けていますので、実装技術の違いはあれど動作は大体同じです。

じゃあどうやって選ぶのかと言うと、僕の場合は探した時に使いたいテーマがあるやつを選択します。

今回は、サラッとした感じのテーマがすぐに見つかったのでHugoにしてみました。

他の候補は、[StaticGen](https://www.staticgen.com/) というサイトによくまとまっていますよ。

# 3. GitHub

コミュニティ共有の資源置き場としては、大抵の人がGoogleアカウントかMicrosoftアカウントは持っていますので、Google DriveかOneDriveが使い易いですよね。

ただ、pyspaはソフトウェアエンジニアが多くgitベースのCIツールを前提にしたワークフローが受け入れられ易いのでGitHubを使っています。

この取組みを始めるより前にorganizationとしてpyspaが存在していたことも大きいですね。

このサイトが置いてあるGitHubのリポジトリはここです。

* [pyspa/pyspa.github.io](https://github.com/pyspa/pyspa.github.io)

GitHubにはGitHub Pagesという機能がありHTMLをホスティングしてくれます。

ただ、今回は僕の学習のため、少し無理目にCloud BuildとFirebase Hostingを使っています。

# 4. Cloud Build

Cloud BuildはGoogle Cloudの一部としてサービス提供されているCI/CD SaaSです。

類似サービスとしては、AWSだとCode Buildで、AzureだとAzure Pipelinesですね。

僕の個人的な趣味の話で言うとCircle CIが好きですが、最近はGitHub Actionsを結構使っています。

## 4.1. GitHub連携における注意点

Cloud Buildはトリガーを構成するだけでGitHubのpushイベントを拾ってビルドします。

* [Running builds on GitHub](https://cloud.google.com/cloud-build/docs/automating-builds/run-builds-on-github?hl=ja)

Cloud BuildとGitHubを接続した時の根源的な問題は、`.git` ディレクトリを何故かビルドサーバ上にコピーしてこないことです。

それによって、今回のユースケースではいくつかの問題が発生しました。問題とその対応策について説明していきます。

### 4.1.1. submoduleの対応

Hugoではテーマをインストールする際には、submoduleとしてチェックアウトすることが推奨されています。

ところが、Cloud Buildではsubmoduleをチェックアウトしません。

つまり、アプリケーションのビルドがsubmoduleに含まれている何かに依存していると失敗します。

これ自体は別にそういう仕様だと言われれば納得できるものです。リポジトリ固有の事情を鑑みずにどこまでsubmoduleを辿るのが妥当なのか仕様を決めるのは難しいですし。

それならsubmoduleのチェックアウトを自分で宣言するだけです。例えば、`git submodule update --init --recursive` みたいなコマンドを実行します。

しかしながら、既に説明した通りCloud Buildでは`.git`ディレクトリをビルドサーバ上にコピーして来ないので、無情にも以下のようなエラーで失敗します。

```
Step #1: fatal: not a git repository (or any parent up to mount point /)
Step #1: Stopping at filesystem boundary (GIT_DISCOVERY_ACROSS_FILESYSTEM not set).
```

仕方ないので、ビルド構成ファイル上でcloneすることになります。

{{<highlight yaml>}}
steps:
  - name: gcr.io/cloud-builders/git
    args: ['clone', '--recurse-submodules', '--depth', '1', 'https://github.com/matsuyoshi30/harbor.git', 'themes/harbor']
{{</highlight>}}

今回はsubmoduleが一つしかありませんしパブリックリポジトリなので、そんなに手間ではありませんけども、これがプライベートリポジトリだったり、submoduleが増えてきたらどうなるんでしょうね。

何らかの方法でビルド時に、SSH鍵をビルドサーバ上に配置するんでしょうか？

### 4.1.2. 手動ビルドとイベントトリガビルドの差分解消

試行錯誤している時は、一々リモートのGitHubにpushするのはわずらわしいですからローカルマシンのリソースを直接Cloud Buildにアップロードしてビルドしたいですよね。

Cloud Buildでは、開発に使っているローカルマシンから `gcloud builds submit` コマンドを実行することでビルドジョブを開始できます。

ビルドジョブを実行する経路が違うだけで、ビルド対象となるファイルやディレクトリに違いがあると効率よく開発が出来ません。

そこで、GitHubのpushから実行されるビルドジョブとローカルから実行するビルドジョブの差分を無くすために`.gcloudignore`ファイルを用意します。

このファイルは、`.gitignore`ファイルと記述方法は同じなのですけども、一点だけ明確に違うものがあります。

それが、既存の`.gitignore`ファイルに記述されている内容を取り込むための `#!include:.gitignore` という部分です。

大抵の場合、これさえ書いておけばビルド実行経路における違いはほとんどが無くなります。

しかし、すでに説明した通りサーバ上でsubmoduleにあたる部分は、ビルドサーバ上でcloneする必要があります。

よって、ローカルからビルドジョブを実行する際には、そのディレクトリがアップロードされないよう`.gcloudignore`ファイルに記述します。

今回の場合は、こうなりました。

```
#!include:.gitignore

themes/*
```

## 4.2. ビルド中に使うツールをコンテナ化する

Cloud Buildのステップは、全て指定したDockerコンテナイメージを`docker run`することで実行していきます。

よって、ビルド中に使うツールの類は全てDockerコンテナイメージとして、いずれかのコンテナレジストリに登録されていなければなりません。

pyspaブログのビルドでは、firebase-toolsとHugoを使いますので、これらをコンテナイメージ化しています。

例えば、Hugoをコンテナイメージ化するにはまずDockerfile.hugoとして以下のような内容のファイルを作ります。

{{<highlight dockerfile>}}
FROM busybox AS build-env
ENV HUGO_VERSION=0.67.1
RUN wget "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz"
RUN tar zxf "hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz"

FROM gcr.io/distroless/cc
ENTRYPOINT ["/hugo"]
COPY --from=build-env /hugo /
{{</highlight>}}

その上で、以下のようなcloudbuild.yamlを作ります。

{{<highlight yaml>}}
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: [ 'build', '-f', 'tools/Dockerfile.hugo', '-t', 'gcr.io/$PROJECT_ID/hugo', 'tools' ]
images:
  - 'gcr.io/$PROJECT_ID/hugo'
{{</highlight>}}

最後は、以下のようにローカルから直接ビルドジョブを実行します。

{{<highlight sh>}}
gcloud builds submit --config tools/cloudbuild.yaml
{{</highlight>}}


こうすることで自分のプロジェクトに付属してくるコンテナレジストリにDockerコンテナイメージを登録します。

ちなみに、`$PROJECT_ID`という部分はCloud Buildが自動的に現在のプロジェクトIDに置き換えてくれますので、このままコピペしてあなたのプロジェクトでも使えますよ。

面倒ならメインのビルドプロセスの中に、このステップを入れてしまっても良いのですが、GitHubにpushするたびにコンテナイメージをデプロイするのではビルドが遅くなってしまうので望ましくありません。

こうやって見ると、Cloud BuildはDockerのコンテナイメージをビルドしてデプロイするには非常に便利なサービスですね。

余談ですけども、マルチステージビルドが前提になるとは言えベースイメージとしての[distroless](https://github.com/GoogleContainerTools/distroless)は本当に最高です。

皆さん是非使いましょう。僕が仕事で使っているDockerイメージは基本的にdistrolessをベースイメージにしています。

## 4.3. Firebaseのデプロイにおけるサービスアカウントへの権限付与

2020/03/21 現在、以下のドキュメントには誤りがあります。

* [Deploying to Firebase](https://cloud.google.com/cloud-build/docs/deploying-builds/deploy-firebase)

冒頭でCloud BuildのサービスアカウントにFirebase Adminの権限を付与しているので、`firebase-tools` の `deploy` コマンドを実行する際に`--token`オプションは必要ありません。

インターネットを検索すると、KMSを使ってFirebaseへのアクセストークンを暗号化するような記事が散見されますが、これは以下のPRで既に解決済みの問題に対するワークアラウンドです。

* [Use public APIs where possible, address service account gaps.](https://github.com/firebase/firebase-tools/pull/1463)

## 4.4. Cloud Buildに対する要望

pyspaブログの構築は非常に単純なビルドパイプラインですが、Cloud Buildへの要望が見えてきたので、それを少しまとめておきます。

### 4.4.1. ビルド構成ファイルのsecretsセクションでSecret Managerをサポートして欲しい

現状、機密性のある情報を扱いたい場合、KMSによって暗号化した上でBASE64エンコードしたものをビルド構成ファイルに記述するということになっています。

* [暗号化されたリソースの使用](https://cloud.google.com/cloud-build/docs/securing-builds/use-encrypted-secrets-credentials)

僕としては、このアプローチは全く許容できません。

何故なら、暗号化してあるとしても機密情報が記載されているビルド構成ファイルは機密情報として扱わざるをえないからです。

機密情報として扱うなら、そのファイルを読み書きできるユーザを出来る限り制限することが望ましいですし、その読み書きは丁寧に記録して監査しなくてはなりません。

しかし、ビルド構成ファイルは本来プロジェクトの参加者全員が自由に閲覧できることが望ましいものです。

ビルド構成ファイルへの読み書きを著しく制限するとビルドパイプラインに関連する問題を解決できるプロジェクト参加者もまた著しく制限されることになります。

そのような状態は決して望ましくありません。

Google CloudにはSecret Managerという機密情報を保存するためのサービスが既に存在しているのですからCloud Buildのビルド構成ファイルで、それを直接参照したいですね。

なお、ビルドトリガーを構成する際に、環境変数を指定できますが同様の問題があります。

Cloud Buildにおけるアクセス制御としてビルドトリガーの構成画面だけ見せないとか、ビルド履歴だけ見せるといった機能はありません。

例えば、AWSのCode BuildではAWS Secrets Managerを直接参照する構文がサポートされています。

* [Build specification reference for CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)

完全に余談ですけども、GCPのサービスは Secret Managerで、AWSのサービスは Secrets Managerです。会議室内に紛れ込んだ裏切り者を見つける際にご活用ください。

### 4.4.2. 定期実行トリガーをもっと簡単に構成したい

アプリケーションを継続的にデプロイしていくためには、GitHubへのpushだけをトリガーにビルドするだけでは足りません、日次や週次で実行するタスクもあります。

例えば、使っているツールのアップデートや、アプリケーションが依存するライブラリの脆弱性情報のチェックなどです。

現状のCloud Buildでそれを実現するには、まずGitHubへのpushからの起動を意図しないビルドトリガーを構成します。

その後、Cloud SchedulerのcronジョブでHTTPターゲットを指定した上で、Cloud BuildのREST APIを呼び出します。

より具体的な手順は以下のページで確認できます。

* [Trigger Google Cloud Build with Google Cloud Scheduler periodically](https://stackoverflow.com/questions/57681367/trigger-google-cloud-build-with-google-cloud-scheduler-periodically)

この手順は、どう考えてもやりたい事に対してかかる手間が見合っていません。

ビルド構成ファイルなり、Cloud BuildのWeb UIなりでもっと簡単に定期実行トリガーを構成したいですね。

### 4.4.3. ステップに条件節を記述できるようにして欲しい

現状では、ステップの実行条件を記述する方法がないので、それぞれのDockerコンテナイメージの中でそれを記述する必要があります。

これによって、複数のステップをまとめたエラー処理をビルド構成ファイルの中に記述することが出来ません。

例えば、複数のステップが存在する場合に、そのいずれかが失敗したらSlackに通知するといった単純なケースでも、全てのステップ用にDockerコンテナイメージを作って、そのエラー処理を記述する必要があります。

ビルド構成ファイルのスコープにおけるエラー処理を記述するため、複数のステップをまたがって参照できる変数領域を実現した上で、それぞれのステップで実行条件を記述できるようして欲しいですね。

### 4.4.4. GitHubとの連携においてビルドトリガーの名前が分かるようにして欲しい

具体的には、PRの画面やproceted branchを構成するGitHubの画面から見えるCloud BuildのビルドトリガーがUUIDのようなおおよそ人間には認識し辛い文字列になっているのです。

{{<figure src="/img/how-to-make-this-site/branch_protection.png" title="Branch protection" >}}

pyspaブログではPR時にとりあえずエラーが無い事だけを確認するだけのビルドプロセスと、マージされた後にデプロイするビルドプロセスを定義してあります。

Cloud Build側の画面で確認するとこういう風になっています。

{{<figure src="/img/how-to-make-this-site/build_triggers.png" title="Build Triggers" >}}

問題は、GitHub側に表示されているIDとCloud Build側に表示されている名前を対応付ける方法がCloud BuildのWeb UIによって提供されていないことです。

その対応関係をどうやって調べるのかと言うと、以下のコマンドを実行します。

{{<highlight sh>}}
gcloud beta builds triggers list
{{</highlight>}}

そうすると、以下のような標準出力が得られるので正確な情報が分かるのです。

```
---
createTime: '2020-03-20T13:08:23.744833084Z'
filename: build-only.yaml
github:
  name: pyspa.github.io
  owner: pyspa
  push:
    branch: .*
id: fab5671f-ab73-42ba-9b8c-xxxxxxxxx
name: build-only
---
createTime: '2020-03-20T05:49:57.279972779Z'
description: ??????????? push
github:
  name: pyspa.github.io
  owner: pyspa
  push:
    branch: ^master$
id: 4171dd20-f1a8-4f71-8651-1xxxxxxxxx
name: blog-pyspa-pushtrigger
tags:
- github-default-push-trigger
```

`?` に文字化けしている description も地味にいい味を出していますよね。この部分には2byte文字が使われています。

# 5. Firebase Hosting

最後は皆さんが見ているこのファイルがホストされているFirebase Hostingについてです。

これについては、あまり語ることは無いのですが、一点だけ説明しておきます。

## 5.1. HTTPヘッダの付与

pyspaブログには、動的な要素はないため難しいことは特に何もありません。

他のサイトから悪用されないようにするため、幾つかのレスポンスヘッダーを全てのレスポンスに付与しています。

Firebaseは任意のレスポンスヘッダーをクライアントに送信する機能がありますのでそれを使います。

以下の内容はpyspaブログで使っているfirebase.jsonです。

{{<highlight json>}}
{
  "hosting": {
    "public": "public",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "headers": [
      {
        "source": "**",
        "headers": [
          {
            "key": "X-Content-Type-Options",
            "value": "nosniff"
          },
          {
            "key": "X-Frame-Options",
            "value": "DENY"
          },
          {
            "key": "X-XSS-Protection",
            "value": "1; mode=block"
          },
          {
            "key": "Referrer-Policy",
            "value": "same-origin"
          }
        ]
      }
    ]
  }
}
{{</highlight>}}

極めて簡潔にやりたいことが実現できていますね。

# 6. まとめ

pyspaブログは、多くのソフトウェアエンジニアが日常的に使っているもので構成されています。

このエントリではCloud Buildを利用するあたって必要になるワークアラウンドを中心にpyspaブログを構成する要素技術について説明しました。

大抵のCI/CDサービスがビルドステップの基礎技術としては、シェルコマンドを採用しています。

これに対して、Cloud Buildでは全てのビルドステップが`docker run`です。これは、極めて柔軟性が高い一方で複雑さのある方式です。

それは明らかな優位性であるとともに、他のCI/CDサービスに比べると分かり辛い部分であるように感じます。

Cloud Buildが皆さんに受け入れられていくのか、そうでないのか分かりませんが、僕は引き続き時々使ってみようと考えています。

## 6.1. ToCに関する補足

このエントリのToCは、[Markdown TOC](https://marketplace.visualstudio.com/items?itemName=AlanWalk.markdown-toc)を使っています。
