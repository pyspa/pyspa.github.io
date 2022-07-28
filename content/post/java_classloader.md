---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Javaのローダーとかクラスパス回り"
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


# Javaのローダーとかクラスパス回りが分からない話

## class ファイルについて

classファイルには一つのクラスしか定義できない。
内部クラスなどにも対応してクラスファイルを作る必要があり、単一のソースコードから複数のclassファイルが出来ることもある。

https://docs.oracle.com/javase/specs/jvms/se18/html/jvms-4.html#jvms-4.1

> This chapter describes the class file format of the Java Virtual Machine. 
> Each class file contains the definition of a single class, interface, or module.

``` java
import java.io.Serializable;

public class TestInnerOuterClass {
    class TestInnerChild{

    }

    Serializable annoymousTest = new Serializable() {
    };
}
```

こういう内部クラス付きのソースを、javac に食わせると

``` shell
-rw-r--r-- 1 sakura sakura 428  7月 28 10:48 'TestInnerOuterClass$1.class'
-rw-r--r-- 1 sakura sakura 346  7月 28 10:48 'TestInnerOuterClass$TestInnerChild.class'
-rw-r--r-- 1 sakura sakura 473  7月 28 10:48  TestInnerOuterClass.class
-rw-r--r-- 1 sakura sakura 162  7月 28 10:48  TestInnerOuterClass.java
```

また、クラスファイルにある`$`はクラス名と内部クラスの名前を分離するためのセパレータ的なものである。
(同じ名前のクラスは作れないので)

``` shell
javap -v クラスファイル
```

などとすることで、classファイルの中身を読むことができる。


## Javaのクラスローダとかの話

Clojureだとreplがあって既存の関数をじゃんじゃん上書きしているが、それって大変。
https://blog.bobuhiro11.net/2014/05-20-clojure-reload.html

Javaにはクラスローダが3種類ある
https://qiita.com/ysk24ok/items/57daed08592f7f8d1bea

#### bootstrap class loader

bootstrap class は `jre/lib/rt.jar` や `jre/lib/` にあるものを読みこむ。
JVM付属のもの。JVM起動時に実行される。

####  extension class loader

`jre/lib/ext` のクラスを読み込む

#### user class loader

上2つの実行後、classpathなどを使ってclassファイルを読み込む。
ユーザが独自に拡張して、自分のプログラムから呼びだすことも可能である。


### ディレクトリとパッケージとかの関係について

https://docs.oracle.com/javase/specs/jvms/se18/html/jvms-5.html#jvms-5.3.1
> Typically, a class or interface will be represented using a file in a hierarchical file system, and the name of the class 
> or interface will be encoded in the pathname of the file to aid in locating it.

ディレクトリパスとパッケージ名を一致させる挙動なのは、上のデフォルトのuser class loaderの挙動がそうなっているからである。
foo.bar.baz.SampleClass があったときに、<classpath>/foo/bar/baz/SampleClass.class を探しにいくということになる。

### jarファイルについて

classファイルをまとめたもの。zip形式で `unzip` で展開できる。
MANIFEST.MF、ResourceBundleなどと一緒に仕様が定められている。
https://docs.oracle.com/javase/7/docs/technotes/guides/jar/jar.html
https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ResourceBundle.html

`java -classpath foo.jar` で読むとき `java -jar foo.jar` で動作が異なる。

jarファイルを読むと、classpath的にjarファイルを展開した時のclassfilesのトップがjavaのclasspathにあるような動作になる(多分)。
`-jar`オプションを使った場合は、MANIFEST.MF などを読みmainの場所などを特定して実行してくれる。

java -jarの時は、ちょっと特別な動作をします。MANIFEST.MFを固定値として読みに行って、そこに書いてあるメタデータを使います。
あれのなかにclasspathを指定したりもできる。.propertiesファイルに関する仕組みはこのへん。

jarファイルの作りかた
https://www.ne.jp/asahi/hishidama/home/tech/java/jar.html


## クラスローダを自分で呼び出して使うことも出来る

johtaniさんが実装している例
https://github.com/johtani/check-dictionary

この辺り
https://github.com/johtani/check-dictionary/blob/master/src/main/java/info/johtani/misc/ja/dictionary/compare/RestrictedURLClassLoader.java

## java 16の話

`<JAVA_HOME>/lib` の下を確認してみると、jarファイルやclassファイルが無く、soファイルばかりがある。
`modules` というファイルがあり、これに格納されている。
`jimage extract modules`することで展開でき、この中に起動時に読み込むclassファイルたちを確認できた。

java 8 はまた違ってそう。


