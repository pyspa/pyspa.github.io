---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "RustでCloud Firestore"
subtitle: ""
summary: ""
authors: ["shirou"]
tags: ["rust"]
categories: []
date: 2021-08-28T13:09:15Z
lastmod: 2021-08-28T13:09:15Z
featured: false
draft: false
---
こんにちは。rust初心者の[shirou](https://twitter.com/r_rudi)です。

今回はなんとなくrustでGoogle CloudのCloud Firestoreを扱ってみたいと思います。

<!--more-->

# ライブラリ

firestore単独のライブラリは結構あります。

- [firestore-db-and-auth](https://crates.io/crates/firestore-db-and-auth)
- [firestore-sdk](https://crates.io/crates/firestore-sdk)

これらはfirestoreを使うだけならば良いと思いますが、今回はGoogle CloudのAPIから自動生成しているこのライブラリを使うことにします。

- [googleapis](https://crates.io/crates/googapis)

このライブラリの使い方に慣れることにより、Google CloudのほかのAPIを使えるようになるのではないか、という狙いがあります。

このライブラリは[tonic-build](https://github.com/hyperium/tonic/tree/master/tonic-build)を使ってAPI定義から自動生成しているものです。[tonic](https://github.com/hyperium/tonic)は活発に開発されているgRPCライブラリですので、今後も継続した更新が見込めます。たぶん。

# 準備

Cargo.tomlはこんな感じで書きました。もしかしたら不必要なものもあるかもしれません。

```
[dependencies]
googapis = { version = "0.5.0",  default-features = false, features = ["google-firestore-v1"] }
gouth = "^0.2.1"
tonic = { version = "^0.5.0", features = ["tls"] }
prost = "^0.8.0"
prost-types = "^0.8.0"
tokio = { version = "1.10.0", features = ["rt-multi-thread", "time", "fs", "macros"] }
```

googapisのfeaturesで使用したいAPIをバージョン込みで指定します。今回はgoogle-firestore-v1のみです。

# firestore client

では実装していきます。

## 宣言

まずは宣言です。説明の都合上、cargo fmtでの順番とは違ってますが、こんな感じになりました。

```rust
use googapis::{
    google::firestore::v1::{
        firestore_client::FirestoreClient, // これは必ず必要
        value::ValueType, Document, Value, // ドキュメント操作に必要。すなわちほぼ必ず必要
        // 以下は実行したいリクエストに応じて追加
        CreateDocumentRequest, // create_document用
        ListDocumentsRequest,  // list_document用
        RunQueryRequest, // run_query用
        StructuredQuery, // run_queryで必要
        run_query_request::QueryType, // run_queryメソッドで必要
        structured_query::{ // // run_queryメソッドで必要
            filter::FilterType, CollectionSelector, FieldFilter, FieldReference, Filter,
        },
    },
    CERTIFICATES, // 必ず必要
};
```

## service作成

まずFirestoreClientからServiceを作成します。
tls_configで指定するドメインとchannelで指定するAPI endpointは[このページ](https://cloud.google.com/firestore/docs/reference/rest?hl=ja)から取得します。Firestore以外のサービスを使用する場合も同様なページからendpointを調べる必要があります。

```rust
let tls_config = ClientTlsConfig::new()
    .ca_certificate(Certificate::from_pem(CERTIFICATES))
    .domain_name("firestore.googleapis.com");

let channel = Channel::from_static("https://firestore.googleapis.com")
    .tls_config(tls_config)?
    .connect()
    .await?;

let token = Token::new()?;
let mut service = FirestoreClient::with_interceptor(channel, move |mut reqRequest<()>| {
    let token = &*token.header_value().unwrap();
    let meta = MetadataValue::from_str(token).unwrap();
    req.metadata_mut().insert("authorization", meta);
    Ok(req)
});
```

ここで生成したserviceをAPI呼び出しで使っていきます。

## list_document

`ListDocumentsRequest`構造体を作成し、`list_documents`に与えます。指定しないフィールドは`..Default::default()`でデフォルト値を設定しておきます。

```rust
let response = service
    .list_documents(Request::new(ListDocumentsRequest {
        parent: String::from("projects/someproject/databases/(default)/documents/root/test"),
        collection_id: String::from("mycollection"),
        page_size: 100,
        ..Default::default()
    }))
    .await?;
let r = response.into_inner();
println!("RESPONSE={:?}", r);
```

responseに対して`into_inner()`を実行するとレスポンスの中身、すなわちlist_documentの中身が取れます。

## create_document

ドキュメントを作成するには`create_document`を使います。

まずは作成するDocumentを作ります。Sample構造体から作成する場合はこんな感じです。fieldsに`HashMap<String, Value>`を使って入れ込んでいきます。

```rust
struct Sample {
    pub title: String,
    pub url: String,  
}

impl Sample {
    fn to_document(&self) -> Option<Document> {
        let mut fields: HashMap<String, Value> = HashMap::new();
        fields.insert(
            String::from("title"),
            Value {
                value_type: Some(ValueType::StringValue(self.title.clone())),
            },
        );
        fields.insert(
            String::from("url"),
            Value {
                value_type: Some(ValueType::StringValue(self.url.clone())),
            },
        );
        Some(Document {
            fields: fields,
            ..Default::default()
        })
    }
}
```

こうして出来上がったDocumentを`create_document`で送り込みます。

```rust
let response = service
    .create_document(Request::new(CreateDocumentRequest {
        parent: String::from("projects/someproject/databases/(default)/documents/root/test"),
        collection_id: String::from("mycollection"),
        document_id: self.aaid.clone(),
        document: self.to_document(),
        ..Default::default()
    }))
    .await?;
println!("RESPONSE={:?}", response);
```

## run_query (クエリ)

他言語のSDKではこんな感じで簡単に書けるのですが、

```python
db.collection(u'cities').where(u'state', u'==', u'CA')
```

このrustライブラリではprimitiveな`run_query`を使わなければいけません。

まずはquery定義です。`Filter`でクエリの条件式を定義します。opはi32なので、コメント中のgRPC定義をもとに手動で指定します。

```rust
let filter = Filter {
    filter_type: Some(FilterType::FieldFilter(FieldFilter {
        field: Some(FieldReference {
            field_path: "url".to_string(),
        }),
        // op is gRPC enum. 
        // https://mechiru.github.io/googapis/src/googapis/up/genproto/google.firestore.v1.rs.html#268
        op: 5,
        value: Some(Value {
            value_type: Some(ValueType::StringValue(self.url.clone())),
        }),
    })),
};
```

作ったfilterを`StructuredQuery`に入れます。`from`は検索するコレクションを指定します。

```rust
let query = StructuredQuery {
    from: vec![CollectionSelector {
        collection_id: "test".to_string(),
        ..Default::default()
    }],
    r#where: Some(filter), // 注意: whereはkeywordなのでそのままでは使えない
    ..Default::default()
};
```

ここで重要なのが `r#where` です。定義では`where`なのですが、`where`はrustの予約語なのでそのままでは怒られます。ですので、escapeし`r#where`と指定します。

こうして出来上がったqueryを`run_query`に与えます。

```rust
let response = service
    .run_query(Request::new(RunQueryRequest {
        parent: String::from("projects/someproject/databases/(default)/documents/root/test"),
        query_type: Some(QueryType::StructuredQuery(query)),
        ..Default::default()
    }))
    .await?;
let r = response.into_inner().message().await;
println!("RESPONSE={:?}", r);
```

こんな感じで出てきます。

```
RESPONSE=Ok(Some(RunQueryResponse { 
    transaction: [], 
    document: Some(Document 
      { name: "projects/someproject/databases/(default)/documents/root/test/ErSfhFsR", 
        fields: {
          "url": Value { value_type: Some(StringValue("https://www.example.com")) },
          "title": Value { value_type: Some(StringValue("example")) },
        },
        create_time: Some(Timestamp { seconds: 1630152255, nanos: 882778000 }),
        update_time: Some(Timestamp { seconds: 1630152255, nanos: 882778000 })
      }),
    read_time: Some(Timestamp { seconds: 1630208712, nanos: 324490000 }), skipped_results: 0
}))
```

# まとめ

rust勉強の一環でCloud FirestoreのAPIを叩いてみました。ぶっちゃけ型定義が厳密でとてもめんどうなのでGolangなどのほうが楽ですね。
