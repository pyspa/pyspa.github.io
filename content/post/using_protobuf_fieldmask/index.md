---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Netflixにおける実用的なAPI設計: gRPCとFieldMask"
subtitle: ""
summary: ""
authors: ['shirou']
tags: []
categories: []
date: 2021-10-17T02:53:58Z
lastmod: 2021-10-17T02:53:58Z
featured: false
draft: false
---
Netflix Tech BlogのgRPC APIに関する以下の２つの記事に感銘を受けたので、ここにその概要を日本語で記します。
<del>(めんどくさかったので)</del>翻訳の許可は取ってませんが、再構成してますし元のJavaではなくPythonで書き直していますので、容赦して下さい…

- [Practical API Design at Netflix, Part 1: Using Protobuf FieldMask](https://netflixtechblog.com/practical-api-design-at-netflix-part-1-using-protobuf-fieldmask-35cfdc606518)
- [Practical API Design at Netflix, Part 2: Protobuf FieldMask for Mutation Operations](https://netflixtechblog.com/practical-api-design-at-netflix-part-2-protobuf-fieldmask-for-mutation-operations-2e75e1d230e4)


# まとめ

- gRPCでは、FieldMaskをうまく使うことで、必要な情報だけ取得したりあるいは与えたりしたりできまっせ

# 第一部

まずField Maskをどのように使うかを述べています。

## 背景

Remote Callというものは、そもそもコストがかかるものです。そのコストはなるべく減らしたいですし、不必要なフィールドが乗ってたりするとそのコストはさらに高くなってしまいます。

[GraphQL](https://graphql.org/)ではField Selectorを使うことで、必要なフィールドだけを取得しています。JSON:APIでは[Sparse Fieldsets](https://jsonapi.org/format/#fetching-sparse-fieldsets)が知られています。

では、gRPCはどのようにしてフィールド数を削減すればよいのでしょうか。Netflixで使用しているのが[Field Mask](https://developers.google.com/protocol-buffers/docs/reference/csharp/class/google/protobuf/well-known-types/field-mask)です。

## Protobuf FieldMask

FieldMask自体は単にstringの配列というごくありふれた定義です。

```proto
message FieldMask {
  // The set of field mask paths.
  repeated string paths = 1;
}
```

そして、FieldMask自体は、[Protocol BufferでWell Known Type](https://developers.google.com/protocol-buffers/docs/reference/python-generated#fieldmask)にあるぐらい、標準で定義されています。といっても、[Goでは存在しないのですが](https://stackoverflow.com/questions/64739310/how-to-use-protocol-buffer-fieldmask-in-go)。

肝心なのは、このFieldMaskに関してさまざまなツールが用意されている、ということです。


## 例: Netflix Studio Production

例としてNetflix Studio Content Production(ここでのProductionは本番環境という意味ではなく、映画製作とかの意味)のサービスをあげます。

```proto
// Contains Production-related information  
message Production {
    string id                         = 1;
    string title                      = 2;
    ProductionFormat format           = 3;
    repeated ProductionScript scripts = 4;
    ProductionSchedule schedule       = 5;
    // ... more fields
}

service ProductionService {
  // returns Production by ID
  rpc GetProduction (GetProductionRequest) returns (GetProductionResponse);
}

message GetProductionRequest {
  string production_id = 1;
}

message GetProductionResponse {
  Production production = 1;
}
```

`GetProductionService` が提供する `GetProduction` メソッドを呼ぶと `Production` の情報が返ってきます。

ところで、NetflixはMicro Serviceで構築されているので、ProductionServiceの裏にはさらに別のサービスがあります。

```
client --> Production Service <---+---> Schedule Service
                                  |
                                  +---> Script Service
```

つまり、Production Serviceはレスポンスに載せる情報を埋めるために別のサービスに対して問い合わせなければならないわけです。

もしクライアントがscheduleやscriptに関する情報が要らなかったとしたら、わざわざScheduleサービスに問い合わせる必要がなくなるので、その分返答は速くなります。
またレスポンスのサイズも小さくできます。

{{< figure src="using_protobuf_fieldmask-Page-3.drawio.svg" caption="FieldMaskの効果" >}}

### FieldMaskをRequestに追加

これを実現するために、 `FieldMask` をRequestに追加します。

```proto
import "google/protobuf/field_mask.proto";

message GetProductionRequest {
  string production_id                 = 1;
  google.protobuf.FieldMask field_mask = 2;
}
```

### クライアント側実装

クライアント側では、以下のようにしてfield_maskに必要な情報のみを指定します。

```python
import grpc
from google.protobuf import field_mask_pb2

import service_pb2_grpc
from service_pb2 import GetProductionRequest

with grpc.insecure_channel("localhost:50051") as channel:
    stub = service_pb2_grpc.ProductionServiceStub(channel)
    req = GetProductionRequest(
        production_id="1",
        field_mask=field_mask_pb2.FieldMask(paths=["title", "schedule.last_updated_by.email"]),
    )
    stub.GetProduction(req)
```

pathsには`"schedule.last_updated_by.email"`というように、子のMessageのfieldも指定できます。

利便性のために、もしもfield_maskが設定されていなかった場合は全部のフィールドを返すようにします。

### サーバー側実装

サーバー側では [MergeMessage()](https://googleapis.dev/python/protobuf/latest/google/protobuf/field_mask_pb2.html#google.protobuf.field_mask_pb2.FieldMask.MergeMessage)を使います。

以下のように、リクエスト内にある"field_mask"の`MergeMessage()`を呼び出します。

```python
class ProductionServicer(service_pb2_grpc.ProductionServiceServicer):
    def GetProduction(self, request, context):
        # ここで例としてidも入れてProductionを作成
        production = Production(id="fooo", title="baaaa")
        # field mask自体はrequestから取得
        field_mask = request.field_mask

        # レスポンスの入れ物を作成
        new_production = Production()
        # maskの分のみをnew_productionにコピー
        field_mask.MergeMessage(production, new_production)

        return GetProductionResponse(production=new_production)
```

本来は`id`と`title`の両方が返ってくるように作成しましたが、`MergeMessage()` により、`title`のみが返ってきます。

ただ、これだとレスポンスに不要な情報を含めないことは実現できますが、不要なバックエンドを呼び出さない、ということはできません

そのためには、まずリクエストからField Maskを取り出します。そして `CanonicalFormFromMask()`を使うことで重複排除を行います。例えば、"schedule"と"schedule.last_updated_by"が指定されていた場合共通の要素である"schedule"だけにする、などです。

下のコードではCanonicalFormFromMaskをした後、pathsをfor文で回し、"schedule"が指定されていた場合は`get_schedule()` でscheduleを取得する、としています。指定されなかった場合は呼ばれませんのでその分の呼出コストが削減できます。

```python

schedule_descriptor = Production.DESCRIPTOR.fields_by_number[Production.SCHEDULE_FIELD_NUMBER]

class ProductionServicer(service_pb2_grpc.ProductionServiceServicer):
    def GetProduction(self, request, context):
...中略
        # CanonicalFormFromMask() で重複排除する
        canonical_mask = field_mask_pb2.FieldMask()
        canonical_mask.CanonicalFormFromMask(field_mask)

        # pathsをiterate
        for field in canonical_mask.paths:
            f = field.split(".", 1)  # .で分割。ただし2段階目まで

            # scheduleがmaskのどこかで指定されていた場合のみ、
            if schedule_descriptor.name in f:
                # scheduleを別マイクロサービスから取得する
                new_production.schedule = get_schedule()
```

"schdule"が指定されているかどうかを単純に文字列で比較してしまうと、フィールド名が変更された時に困ります。そのため、globalに定義している `schedule_descriptor` をfield_numberから作成し、その名前と比較しています。これについて次の項目で説明します。

### Field Nameを指定すべき？Field Numberを指定すべき？

proto定義には"field name"が存在していますが、実際にprotobufにエンコードされたメッセージの中にはfield nameは存在せず、"field number"のみが存在しています。これによりprotobufはメッセージのサイズを少なくしています。

図にするとこんな感じです。

{{< figure src="using_protobuf_fieldmask-Page-1.drawio.svg" caption="同じバージョンの例" >}}

これであれば問題はありません。しかし、サーバーとクライアントで読み込んでいるproto定義のバージョンが異なり、例えばクライアント側では"title_name"というfield_nameになっていたらどうでしょうか。

{{< figure src="using_protobuf_fieldmask-Page-2.drawio.svg" caption="違うバージョンの例" >}}

FieldMaskにはfield numberではなくfield nameを指定します。そのため、このようにfield nameが変わった場合、protobuf上では問題なく通信できたとしても、FieldMaskではそのfield nameが存在しないことになってしまいます。

この問題を解決するためにいくつか案があります。

- FieldMaskで指定しているfield nameを変えない
  - 単純ですが実際には常にできるとは限りません
- backendですべての古いfield nameに対応する
  - 後方互換性の問題を解決しますが、backendがすべてのfield nameの変更を追跡し続けなければいけません
- 古い名前を"deprecate"にし、名前の変更の代わりに新しいfieldを作成する
  - 例えば以下のようにfield number = 6として追加します。
  - この方式であれば今まで通りにコードを生成できます
  - [deprecated](https://developers.google.com/protocol-buffers/docs/proto3#options) optionを指定することでより分かりやすくなります。

```proto
message Production {
  string id = 1;
  string title = 2 [deprecated = true];  // 代わりに "title_name" を使用すること
  ProductionFormat format = 3;
  repeated ProductionScript scripts = 4;
  ProductionSchedule schedule = 5;
  string title_name = 6;
}
```

いずれの方法を取るにせよ、FieldMaskは提供するAPIにおける契約として重要であると認識しておく必要があります。


## pre-build(事前定義) field mask

これまで述べてきた例では、FieldMaskをクライアント側が指定していました。しかし、例えばAPIとしてライブラリを提供する時に事前に定義しておくと使う側として便利です。

```python
class ProductionFieldMasks:
    @classmethod
    def title_and_format_field_mask(self):
        mask = field_mask_pb2.FieldMask()
        # 本来であれば上記のようにField numberを使って定義する
        mask.FromJsonString("title,format")
        return mask
```

本来であればField numberを使って定義したいのですが、元記事のJavaにある`fromFieldNumbers()`関数がpythonにはないのでstringで定義しちゃっています。

### 情報の絞り込み

ここは元記事にはない部分です。例えば認証情報に応じて返答する情報を絞り込めます。

- ユーザーには通常の情報を
- 管理者には通常の情報に加えて管理用の情報を

渡したいという場合にも使えます。

以下は別途取得する"role"が"user"の場合はtitleだけしか渡さず"secret"については返さない、という例です。

```python
user_mask = field_mask_pb2.FieldMask()
user_mask.FromJsonString("title")  # userにはtitleしか渡さない

class ProductionServicer(service_pb2_grpc.ProductionServiceServicer):
    def GetProduction(self, request, context):
        # ここは共通処理。例えばDB内の情報まるごと取得するなど
        production = Production(id="fooo", title="baaaa", secret="super_secret")
        
        new_production = Production()
        if role == "user":
            user_mask.MergeMessage(production, new_production)
```

値はありませんが、proto定義から"secret"というfieldがあること自体は分かってしまいますので、その点は注意してください。

## 制限事項

- (今まで述べてきたとおり)FieldMaskを使用することでfield nameの変更に制限が生まれます
- `repeated`なfieldはFieldMaskのpathの最後に指定する必要があります。これはつまり、リスト内のメッセージの個々のサブフィールドを選択することはできません。
  - これは将来的には変更される可能性があります。最近承認されたGoogle API Improvement Proposal [AIP-161](https://google.aip.dev/161)には"repeated"なfieldに対するwildcardサポートが入っています。


# 第二部

第一部ではサーバーサイドでFieldMaskを使う使い方を紹介しました。[第二部](https://netflixtechblog.com/practical-api-design-at-netflix-part-2-protobuf-fieldmask-for-mutation-operations-2e75e1d230e4)では、FieldMaskをupdateやremoveといった変更操作に使用している例を紹介します。


## 例: Netflix Studio Production

今回も前回と同じProductionのserviceを使って紹介します。

## Productionの変更の詳細

productionのformatを変更するには、 `UpdateProductionFormat` という感じで指定する方法があります。

```proto
message UpdateProductionFormatRequest {
  string id = 1;
  ProductionFormat format = 2;
}

service ProductionService {
  rpc UpdateProductionFormat (UpdateProductionFormatRequest) 
      returns (UpdateProductionFormatResponse);
}
```

しかし、これを全fieldに対して個別のAPIを用意するのは非現実的です。

代わりに、`UpdateProduction`という一個のAPIを用意し、そのリクエスト中に全fieldを入れておく、という方式が考えられます。

```proto
service ProductionService {
  rpc UpdateProduction (UpdateProductionRequest) returns (UpdateProductionResponse);
}

message UpdateProductionRequest {
  Production production = 1;
}
```

このやり方の問題点は2つあります。1つはたとえformatなどの1つのfieldを更新したいだけであっても、clientはProductionのすべてのFieldを把握し、提供しなければならないことです。もう一つの問題は、Productionには多くのFieldがあるため、リクエストのサイズが大きくなってしまいます。

また、この方式ではFieldの値の削除ができません(shirou注:いやぁ、実装次第でできるのでは?)

では更新したいFieldだけ更新する方法を紹介します。

## 変更操作にFieldMaskを使う

2番目の方式を発展させたFieldMaskを使う方式を紹介します。まず、`ProductionUpdateOperation`というメッセージを定義し、その中に更新されるべき内容を入れておきます。仮に全部更新できるのであれば、元のProductionをそのまま使っても良いかも知れませんが、だいたいに置いて更新できるfieldを限定したいはずです。

さらに、リクエストに `update_mask` としてFieldMaskを追加します。

```proto
message ProductionUpdateOperation {
  string production_id = 1;
  string title = 2;
  ProductionFormat format = 3;
  ProductionSchedule schedule = 4;
  repeated ProductionScript scripts = 5;
  ... // その他の更新できるフィールド
}

message UpdateProductionRequest {
  // contains production ID and fields to be updated
  ProductionUpdateOperation update = 1;
  google.protobuf.FieldMask update_mask = 2;
}
```

client側はこれに沿ってリクエストを組み立て送ります。ProductionUpdateOperationには更新する内容だけを含めればよく、全部を含める必要はありません。

```python
    operation = ProductionUpdateOperation(
        production_id="1",
        format=ProductionFormat(format="foo"),
        # schedule.planned_launch_dateは含めてない、なので消される
    )
    req = UpdateProductionRequest(
        update=operation,
        update_mask=field_mask_pb2.FieldMask(
            paths=["title,schedule.planned_launch_date"]
        ),
    )
    print(stub.UpdateProduction(req))
```

この例では"format"の変更をProductionUpdateOperationに入れていますが、FieldMaskには指定がある`schedule.planned_launch_date` は含まれていません。これはつまり、削除されることを意味しています。


## Empty / Missing Field Mask

FieldMaskが設定されていない、あるいはpathがない場合更新操作はすべてのFieldに適用されます。これはつまりクライアント側からは全fieldを送らなければならないことになります。そうしないと前述のように未設定のfieldが削除されることになります。

この仕様は、新しいfieldを追加した時に難易度が高くなります。つまり、クライアント側が新しくfieldが追加されたことを把握していないと送信を忘れ、fieldが削除されることになってしまうからです。

この問題を解決するためには、FieldMaskを必ず指定するようにすると良いです。あるいは他の選択肢としては、リクエストに必ずバージョン番号を含めるようにし、古いバージョンに存在しないfieldをスキップする実装にすることです。

(shirou注:含まれていないと削除、というのはちょっと乱暴な気がするので削除だけ別APIにするとかほかのやりようはありそうです)


## まとめ

API設計者は、シンプルさを目指す一方で、APIを拡張・進化可能なものにしなければなりません。しかし、APIをシンプルかつ将来性のあるものにするのは難しいです。APIにFieldMaskを活用することで、シンプルさと柔軟性を両立させることができるかもしれません。