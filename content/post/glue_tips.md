---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "AWS Glue Job Pythonのちょっとした小技の備忘録"
subtitle: ""
summary: ""
authors: ['shirou']
tags: []
categories: []
date: 2021-09-10T11:54:59Z
lastmod: 2021-09-10T11:54:59Z
featured: false
draft: false
---
こんにちは。[shirou](https://twitter.com/r_rudi)です。
pyspaらしくたまにはpythonの話でも。

AWS Glue Python便利ですね。で、ちょっとハマったことがあったので、備忘録としてまとめておきます。知ってる人には常識かと思います。


## pythonライブラリを追加する

glue scriptにpythonのライブラリを追加するには、zipで固めてあげて、としなければならないと思っていました。
しかし、実はジョブパラメータの設定で、

- `--additional-python-modules`

というキーを作り、そのvalueとして

- `pyarrow,awswrangler,psycopg2-binary`

というようにカンマ区切りで複数のライブリ名を指定してあげれば勝手にpipを起動して、pypiからインストールしてくれます。
`scikit-learn==0.21.3` というような指定も可能でし、`s3://...`としてwhlなどを指定することも可能です。

詳細はここに記載されています。

https://docs.aws.amazon.com/glue/latest/dg/reduced-start-times-spark-etl-jobs.html


## Glue version 3でのエラーに対応する

Glue version 3でparquetに書き出そうとすると、以下のエラーが出ることがあります。

```
Caused by: org.apache.spark.SparkUpgradeException: You may get a different result due to the upgrading of Spark 3.0: writing dates before 1582-10-15 or timestamps before 1900-01-01T00:00:00Z into Parquet INT96 files can be dangerous, as the files may be read by Spark 2.x or legacy versions of Hive later, which uses a legacy hybrid calendar that is different from Spark 3.0+'s Proleptic Gregorian calendar. See more details in SPARK-31404. You can set spark.sql.legacy.parquet.int96RebaseModeInWrite to 'LEGACY' to rebase the datetime values w.r.t. the calendar difference during writing, to get maximum interoperability. Or set spark.sql.legacy.parquet.int96RebaseModeInWrite to 'CORRECTED' to write the datetime values as it is, if you are 100% sure that the written files will only be read by Spark 3.0+ or other systems that use Proleptic Gregorian calendar
```

これは、Glue version 3からSpark 3.0を使うようになり、Spark 3.0と2.0以前とは使用するparquetのバージョンが違うけど、ほんとにいいの？っていうエラーです。
本来は `CORRECTED` にして、移行していきたいのですが、2021年9月時点では、[stackoverflowのこの回答](https://stackoverflow.com/a/69040997)によると `data_frame.rdd.isEmpty()` をしたらfailするなどまだバグがありそうなので、ここは涙を飲んで `LEGACY` を指定します。

先程と同様にジョブパラメータに

- `--conf`

というキーを作り、値を

- `spark.sql.legacy.parquet.int96RebaseModeInWrite=LEGACY`

とします。また、複数のconfが必要な場合は

- `spark.sql.legacy.parquet.int96RebaseModeInWrite=LEGACY --conf spark.sql.legacy.parquet.datetimeRebaseModeInWrite=LEGACY`

というように、値の後にもう一度 `--conf` と入れて続けた値を突っ込みます。このあたりどうにかならないものか、とも思いますが仕方ない。


# まとめ

AWS Glueのちょっとした備忘録でした。ちゃお～