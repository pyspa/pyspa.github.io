---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "SBOMイントロダクション"
subtitle: ""
summary: ""
authors: ["shirou"]
tags: []
categories: []
date: 2022-04-22T12:27:36Z
lastmod: 2022-04-22T12:27:36Z
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
# SBOMのお話


C:
> OSS管理の国際規格ってどういうことです?

B:
> む？Openchainの話ですが
> 雑に説明します。
> ある製品(ソフトウェア、ハードウェア問わず)に含まれている、ソフトウェアのサプライチェーン、ライセンスなどを適切に管理しましょうという活動です。
> それを主導しているLinux Foundationの組織が[Openchain](https://www.openchainproject.org/)。そして、その活動が世界でも活発な場所がドイツと日本。
> これを実現するために、OpenchainというISO標準ができました。
> これはソフトウェアの取り扱いに関して、企業はどういう体制を作らねばならないかという標準です。

A:
> 使う側ががんばるだけなら大いに結構ですがね

B:
> また、この中でソフトウェアに関してそのソフトウェアがどういったソフトウェアを含めるかのフォーマットを定める必要がありました。
> これを定めた標準があってSPDXというものがあります。
> これを決めている団体もOpenchainの団体のサブグループだったりします。

A:
> pythonメタデータはSPDXはライセンスのshort nameを参照するように推奨しております

B:
> 今後ありそうな方向としてはOpenchainの標準を取得していない団体のソフトウェアの納品を断るみたいなことになりえます
> バイデン大統領などがSBOMをサイバーセキュリティ施策のなかの1つとして掲げたりしています。
> 昨今のサプライチェーン攻撃などを受けての流れですね。

> と、まぁ、つまり使う側がライセンスや発注先、発注元今まで適当にやりすぎたツケを払ってねと言ってるだけでもある
> GPLだっつてるのに勝手に製品に組み込んだ挙句ソース開示拒否みたいな
> OSSを公開する側にとってはどちらかというとやりやすくなるはず
> だって搾取される可能性がなくなるのだから

A: 
> まあライセンスさえちゃんとしてればっていうのでいいならこれまででいいのかね

B: 
> SBOMというのを企業は扱わないといけないのがコストとして今後掛かってきますね
> その辺の管理ツールはまだまだこれからですが時が解決するでしょう

A: 
> [deps.dev](https://deps.dev/) もそこらへん意識したものだったんだな

B: 
> そこらへんのライセンス表記がどこまで信じられるものか？というのが多分大きな企業は気にしているところですね
> MITって書いてあったけど実は嘘とか、本人気が付いてなかったけど実はGPLのものがあったとか
> OSS公開する側も、そのOSSが利用しているOSSのライセンスとかをちゃんと気にしてねっていう話でしかないんですが

A: 
> OSSを公開するときはOSSを使ってもいるのだ

B: 
> あとはライセンスの表記を適当にするのを辞めるのだ
> ということくらいです
> BSDと一言だけ書いてるみたいなｗ
> 義務とかちゃんと守るのだみたいな

C:
> はーんなるほど

B: 
> ライセンス条文張ってあるけど、どう見ても義務を守ってないやつとか企業は採用できないし

B: 
> んで、こういう活動を見て

C:
> つまりOSS開発者の99%ぐらいは現時点では従っていない国際規格なのね。

B: 
> いや
> Linux Kernelが従っているのでSPDXの表記は
> なので活動自体は支持されている
> SBOMという製品の部品表みたいなのは産業界の要請で出てきているから、これから普及するというかTOYOTAなりが担いでるし事実上のデファクトでもある

A: 
> https://peps.python.org/pep-0639/
> peps.python.orgpeps.python.org
> PEP 639 – Improving License Clarity with Better Package Metadata | peps.python.org
> Python Enhancement Proposals (PEPs)
> PythonではSPDXの名前を尊重せぇという話になりました
> あ、よく見たらこのPEPは新しいやつだ
> まあpythonパッケージングでもSPDX表記への対応が進んでいますよ

A: 
> 好きに書いてた LICENSE フィールドから LICENSE-EXPRESSION フィールドにspdx identifiers書くことになったんだな

B: 
> 素晴らしいですね

A: 
> じゃあこの話題はSBOMと絡めて話すべきことなんだな。ふむふむ。

C:
> とはいえ、デジュールじゃんというのは間違いないと思う
> 現在進行形でnpm moduleの何割が従ってるのよって言う

B: 
> うーん。そこに誤解がありそう

A: 
> ISCってなってるのが多いからたいがいじゃないの

B: 
> SPDX identifierに従ってないものは、それはそれ。協力してくれたらうれしい。

D:
> データベースが有ることと、データベースに機械的に取り込めるメタデータを持つことは別ってことであってる? 

B: 
> SPDXという団体が全部悪いな

A: 
> プレゼンス悪いよ

B: 
> SPDX identifier => ライセンスの表記統一しようぜ
> SPDX/ SPDX Lite => SBOMの規格統一しようぜ
> 前者のidentifierは従ってくれたらうれしいというやつで、守ってないとかいうのが普通の状態。

A: 
> さきほどのPEPはspdx identifierだけでなくspdx expressionまで含めたもののようです。まあ LICENSE-EXPRESSION って名前だもんな。

B: 
> 後者のSBOMの規格に関しては、紆余曲折あってSPDXを大手が担いだから今後SBOMというものが出てくるときはほぼSPDXになる。

A: 
> https://spdx.dev/specifications/
> 2.2やでもう
> SBOMは基準とかそういうので、それを満たすのはまずSPDXでリファレンスしてんぜっていうことかな

B: 
> SBOMは
> - ソフトウェアの名前とかバージョンなどを一位に定めるもの
> - ライセンス
> - 依存関係
> の集合体なんですよね
> この中をどう埋めるか、何で埋めるか？というのはまだ確定できていない

A: 
> [SBOMとは何ですか？ - The Linux Foundation](https://www.linuxfoundation.jp/blog/2021/07/what-is-an-sbom/)
> をみて、構成管理表と、理解した

B: 
> 依存関係は仕様として書けるけど

C:
> いちOSSデベロッパとしては何をどうすればいいんでしょう？

A: 
> ちゃんとライセンス書く

B: 
> ライセンスをちゃんと書く。以上
> ライセンス違反をしない

C:
> 特定のフォーマットに従って依存OSSを列挙するとかは要らない？

B: 
> 全然関係ないです
> それをやる必要があるのは製品を市場に流通させる責任を持つ人

A: 
> ちゃんと = spdx identifierを使いましょう。使ってるライブラリのライセンスもちゃんと確認して矛盾とか起きないようにしておきましょう。
> くらいかと

C:
> リポジトリのルートにLICENSEファイルを置いてOSSのいい感じのライセンスを設定しましょうと

D:
> - ライセンスを宣言する
> - (できれば) SPDX Identifier 準拠の(マシンリーダブルな形式の?)ライセンス定義をする
> ってことかなあと理解した

B: 
> なので、Cさんが社員として製品を出すときに求められることはあり得る未来です。

A: 
> なのでOSS公開する人としてはこれまでちゃんとしてるならそのままですよねと

C:
> とあるOSSは全行僕が手書きしました
> ライセンスはApache2.0のはず。依存はOpenSSLのみ。

B:
> そのOSSを見ましたが、残念なお知らせとしては
> こういう感じにヘッダにライセンスがあったりなかったりするのは困るという

D:
> 「部品」って言葉が引っかかりやすいのかねえ。

A:
> https://spdx.org/licenses/
> この名前使ってねっと

E:
> [deps.dev](https://deps.dev/) での怒られに対応していけば良さそうな気がしている。

C:
> なるほど？
> 抜けがあったか

B:
> なので、このライブラリを採用する場合は、作者に確認する必要があるので、いつの日か連絡が来る
> 全部書くのが面倒くさいから1行で済むSPDX identifierが便利ですよっていう

C:
> ふーむ

A:
> [licenser - Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ymotongpoo.licenser)
> これを。

D:
> 大企業の部品表に載せたいので、SPDX identifier 形式のライセンス定義を書け、って言われるとちょっと身構える気持ちはあります。

B:
> ライセンスの面倒なところは新しくバージョン切っても古いバージョンには波及しないことですね

A:
> SPDX identifier 形式と身構えても結構普通の文字列ですよ

B:
> // SPDX-License-Identifier: GPL-2.0-only
> これだけ
> 例はこれ [linux/cache.c at master · torvalds/linux](https://github.com/torvalds/linux/blob/master/fs/9p/cache.c)

D:
> たぶんこれが package.json だけだとライセンス違反が怖いので、マシンリーダブルなライセンス定義を書いてくれって言われると、それくらいなら…とガードを下げそう

A:
> そうかtrove classifierのLICENSEカテゴリどうなるんだろな
> _spdx expression便利やな

D:
> 大企業とか部品表っていうキーワードで身構えちゃいそうだなあ、ぐらい
> ちなみに cpython はファイル単位でのライセンス表記はやめてるっぽいすよ
> [cpython/typing.py at main · python/cpython](https://github.com/python/cpython/blob/main/Lib/typing.py)


A:
> あったりなかったりじゃなければいいんじゃないの

D:
> なるほど?

C:
> その1行を全ファイルの先頭に入れればいいんです？

B:
> それだけでばっちりです

C:
> よっしゃ

C:
> onlyってのが引っかかるな
> どうせ1行で簡単に書けるなら

B:
> どれを選ぶかは
> https://spdx.org/licenses/
> これを参照です
> あ、Aさんが張ってたか

A:
> GPL-2.0-Onlyは3.0を含まないっていう感じのidentifier なので Apache 2.0ならそのままのはず
> [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) だな

B:
> ちなみにファイルごとに在ったらいいのは、一部コピペで使う場合とかもありうるし、Cとかだと極端な話makeのオプションで含めるソースが変わってライセンスが違うかもしれないからなど

A:
> spdxってRDFなんだな

B:
> そうです

A:
> 各種のエコシステムから変換できるようなOWLを定義すべきなのかなこれ
> あでもそもそも既存のエコシステムのパッケージングメタデータがトリプルが取れるように形式化されているかどうかからか

B:
> Jsonもいけるようになってた気もします

A:
> んでも意味論はあるわけっしょ

B:
> [spdx-spec/spdx-schema.json at master · spdx/spdx-spec](https://github.com/spdx/spdx-spec/blob/master/schemas/spdx-schema.json)
> [spdx-spec/spdx-ontology.owl.xml at master · spdx/spdx-spec](https://github.com/spdx/spdx-spec/blob/master/ontology/spdx-ontology.owl.xml)
> この辺ですかね
> ちなみに日本企業の尽力の結果(多分)、ExcelでもOKですよ

E:
> Excel！

B:
> [Japan-WG-General/License-Info-Exchange/SPDX-Lite-sample at master · OpenChain-Project/Japan-WG-General](https://github.com/OpenChain-Project/Japan-WG-General/tree/master/License-Info-Exchange/SPDX-Lite-sample)
> [Japan-WG-General/SPDX-Lite-sample_ffmpeg.xlsx at master · OpenChain-Project/Japan-WG-General](https://github.com/OpenChain-Project/Japan-WG-General/blob/master/License-Info-Exchange/SPDX-Lite-sample/ffmpeg/SPDX-Lite-sample_ffmpeg.xlsx)
