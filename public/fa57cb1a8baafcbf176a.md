---
title: ' Go言語のgrokパッケージの進化版を試してみた'
tags:
  - Go
  - grok
private: false
updated_at: '2024-11-15T05:56:48+09:00'
id: fa57cb1a8baafcbf176a
organization_url_name: null
slide: false
ignorePublish: false
---
#　はじめに

ログなどのテキスト情報から情報を取り出すための技術にgrokというものがあります。最近grokを実装した新しいGo言語パッケージを発見したので昔からあるパッケージとの比較した記録を紹介します。

X（旧Twitter）で公開されているイーロン・マスク氏が設立した企業のxAI社が手掛ける対話型AIの話ではありません。

# grokの基本

grokはログなどのテキストから定義したパターンに一致する情報を識別子（変数名）付きで取り出すためのものです。
パターンの定義は、

```
%{SYNTAX:SEMANTIC}
```

のようにします。

SYNTAXは取り出したい情報のパターン名です。3.44という数値を取り出したければ`NUMBER`を指定します。192.168.1.1のようなIPを取り出したければ、`IP`を指定します。
SEMANTICは、取り出した情報の変数名（識別子）です。3.44がイベントの継続期間であれば、durationという変数名をつけます。192.168.1.1がアクセス元ならば、clientという変数名をつけます。

この例をgrokで記述すると

```
%{NUMBER:duration} %{IP:client}
```

のようになります。

```
3.44 192.168.1.1
```

というログからgrokパターンで情報を取り出すと
```
duration: 3.44
clinet: 192.168.1.1
```

のように変数名付きで扱えるようになります。
ログを分析する時に処理が、かなり楽になる仕組みです。


#　Go言語のgrokパッケージ

GitHUB検索すると3つ見つかります。

## たぶんオリジナル

https://github.com/vjeantet/grok


１０年前に開始して最終更新が４年前です。

最初に見つけて、[TWSNMPシリーズ](https://note.com/twsnmp/n/na5c62f59833d)の開発で使ったいるものです。



## トリバゴ？版

https://github.com/trivago/grok

ベンチマークの結果をみるとパターンの作成と並行処理を高速化しているようです。

１０年前にオリジナルからforkして最終更新は７年前です。


## elastic版


https://github.com/elastic/go-grok


今回見つけたものです。オリジナルからforkしたものではないようす。re2を使っているというような記述がありますが、不明です。

７ヶ月前に開始して最終更新が４ヶ月前です。


#  サンプルプログラム

オリジナルで紹介されているサンプルプログラムを試してみました。

## オリジナル

```go:main.go
package main

import (
	"fmt"

	"github.com/vjeantet/grok"
)

func main() {
	g, _ := grok.New()
	values, _ := g.Parse("%{COMMONAPACHELOG}", `127.0.0.1 - - [23/Apr/2014:22:58:32 +0200] "GET /index.php HTTP/1.1" 404 207`)
	for k, v := range values {
		fmt.Printf("%+15s: %s\n", k, v)
	}
}
```

COMMONAPACHELOGは、ApacheというWebサーバーのCOMMON形式のログに対応したgrokパターンです。

実行すると

```terminal
$go run main.go
             IP: 127.0.0.1
           IPV4: 127.0.0.1
       HOSTNAME:
           TIME: 22:58:32
         SECOND: 32
           verb: GET
      BASE10NUM: 207
COMMONAPACHELOG: 127.0.0.1 - - [23/Apr/2014:22:58:32 +0200] "GET /index.php HTTP/1.1" 404 207
           HOUR: 22
         MINUTE: 58
       clientip: 127.0.0.1
           USER: -
           auth: -
          ident: -
   EMAILADDRESS:
 EMAILLOCALPART:
           YEAR: 2014
            INT: +0200
        request: /index.php
           IPV6:
       USERNAME: -
      timestamp: 23/Apr/2014:22:58:32 +0200
          MONTH: Apr
     rawrequest:
          bytes: 207
       MONTHDAY: 23
    httpversion: 1.1

```

のような感じになります。
クライアントのIPなどは所得できています。応答コードが取得できないようです。


## elastic版

オリジナル版と同じサンプルを動かそうとすると多少修正が必要です。

```diff_go
package main

import (
	"fmt"

	"github.com/elastic/go-grok"
)

func main() {
-	g, _ := grok.New()
+	g, _ := grok.NewComplete()
+	g.Compile("%{COMMONAPACHELOG}", false)
-	values, _ := g.Parse("%{COMMONAPACHELOG}", `127.0.0.1 - - [23/Apr/2014:22:58:32 +0200] "GET /index.php HTTP/1.1" 404 207`)
+	values, _ := g.ParseString(`127.0.0.1 - - [23/Apr/2014:22:58:32 +0200] "GET /index.php HTTP/1.1" 404 207`)
	for k, v := range values {
		fmt.Printf("%+15s: %s\n", k, v)
	}
}

```

のようになります。

デフォルトには、COMMONAPACHELOGが含まれていないので、組み込まれている全パターンを読み込む

```go
g, _ := grok.NewComplete()
```
を使っています。


実行すると

```terminal
go run main.go
      timestamp: 23/Apr/2014:22:58:32 +0200
           YEAR: 2014
             IP: 127.0.0.1
          MONTH: Apr
   url.original: /index.php
      BASE10NUM: 1.1
           IPV4: 127.0.0.1
       MONTHDAY: 23
           TIME: 22:58:32
           HOUR: 22
         MINUTE: 58
            INT: +0200
   http.version: 1.1
http.response.body.size: 207
COMMONAPACHELOG: 127.0.0.1 - - [23/Apr/2014:22:58:32 +0200] "GET /index.php HTTP/1.1" 404 207
 source.address: 127.0.0.1
         SECOND: 32
http.request.method: GET
http.response.status_code: 404
HTTPD_COMMONLOG: 127.0.0.1 - - [23/Apr/2014:22:58:32 +0200] "GET /index.php HTTP/1.1" 404 207
```

のようになります。応答コード404も取得できています。


# ベンチマーク

elastic版のbenchmarksに、３種類のパッケージを比較できるベンチマークがあるので私のMACで試してみました。

```terminal
$go test -bench . -benchmem
go: downloading github.com/trivago/grok v1.0.0
goos: darwin
goarch: amd64
pkg: github.com/elastic/go-grok/benchmarks
cpu: Intel(R) Core(TM) i5-8500B CPU @ 3.00GHz
BenchmarkParseString-6                	    8294	    131559 ns/op	    4562 B/op	       5 allocs/op
BenchmarkParseStringRegexp-6          	    9661	    127712 ns/op	    3848 B/op	       3 allocs/op
BenchmarkParseStringTrivago-6         	    8534	    131852 ns/op	    4566 B/op	       5 allocs/op
BenchmarkParseStringVjeanet-6         	    8589	    134199 ns/op	    5979 B/op	       7 allocs/op
BenchmarkNestedParseString-6          	   23575	     50277 ns/op	    3452 B/op	       4 allocs/op
BenchmarkNestedParseStringTrivago-6   	   24788	     48061 ns/op	    3427 B/op	       4 allocs/op
BenchmarkNestedParseStringVjeanet-6   	   23722	     49186 ns/op	    4037 B/op	       5 allocs/op
BenchmarkTypedParseString-6           	   23742	     50472 ns/op	    3867 B/op	       9 allocs/op
BenchmarkTypedParseStringTrivago-6    	   24266	     50727 ns/op	    3473 B/op	       6 allocs/op
BenchmarkTypedParseStringVjeanet-6    	   22989	     52878 ns/op	    4197 B/op	      14 allocs/op
PASS

```

劇的に速くなるわけではないようですが、メモリー使用量が改善されるようです。


# 余談

最近開発しているログ分析ツール

https://note.com/twsnmp/n/nd90978e728c0

では、elastic版のgrokを使ってみました。
複雑なパターンを使う時には便利ですが、IPアドレスだけ取り出すような処理は、標準の正規表現パッケージで処理したほうが速いようです。

TWSNMP FCなどのほかのソフトでも使うか悩んでいます。



