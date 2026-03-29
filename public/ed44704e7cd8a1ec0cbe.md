---
title: 'OllamaとWeaviateを使ってAI(LLM,RAG)によるログ分析をやってみた'
tags:
  - Go
  - ログ分析
  - LLM
  - ollama
  - Weaviate
private: false
updated_at: '2025-04-19T18:31:25+09:00'
id: ed44704e7cd8a1ec0cbe
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

OllamaとWeaviateを使ってログをAIで分析する実験をしてみました。
その実験で作ったプログラムの解説です。

最終的にログ分析ツールに

![2025-04-08_06-50-18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3874862/26423f02-8128-43c1-bf33-1c3bcea1514c.png)

のような構成で組み込みました。


# Ollamaの準備

Ollamaは、LLMをローカル環境で動かすためのソフトです。

https://github.com/ollama/ollama

Windows/Mac OS/Linux/Dockerの環境で動作します。
私は、Mac OS版をダウンロードしてインストールしました。


# Ollamaのモデルの準備

Ollamaには、以下のコマンドでモデルをダウンロードしておきます。

```terminal
$ollama pull nomic-embed-text
$ollama pull llama3.2
```


# Weaviateの準備

Weaviateは、AIで使えるベクトルDBです。

https://weaviate.io/

このWeaviateのローカル環境のQuick Start

https://weaviate.io/developers/weaviate/quickstart/local

の最初のほうの説明を実施すれば環境が作れます。

私は、Dokcer composeで起動しました。

```yaml:docker-compose.yml 
---
services:
  weaviate:
    command:
    - --host
    - 0.0.0.0
    - --port
    - '8080'
    - --scheme
    - http
    image: cr.weaviate.io/semitechnologies/weaviate:1.30.0
    ports:
    - 8080:8080
    - 50052:50051
    volumes:
    - weaviate_data:/var/lib/weaviate
    restart: on-failure:0
    environment:
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
      ENABLE_API_BASED_MODULES: 'true'
      ENABLE_MODULES: 'text2vec-ollama,generative-ollama'
      CLUSTER_HOSTNAME: 'node1'
volumes:
  weaviate_data:
...
```

次のコマンドで起動できます。

```terminal
$docker-compose up -d
```

これで、WeaviateのURL
http://localhost:8080
を通して、weaviateのDBとOllamaを利用できます。


# AI連携プログラムの説明

## 基本的な処理

基本的な処理は

- weaviateへ接続
- コレクションの作成
- ログをファイルからコレクションへ登録
- ログの類似検索
- RAGによる生成AIへの質問
- コレクションの削除

です。
先ほどのWeaviateのローカル環境のQuick Startの解説に従えば作れます。

## ライブラリ

私は、Go言語のライブラリ

https://weaviate.io/developers/weaviate/client-libraries/go

を使用しました。


## weaviateへの接続

```go
	cfg := weaviate.Config{
		Host:   "localhost:8080",
		Scheme: "http",
	}
	client, err := weaviate.NewClient(cfg)
```

の処理で、weaviateと通信可能なclient変数を作成します。
ローカルのテスト環境なので、暗号化と認証はしません。

## weaviateのコレクション作成

weaviateのDBにログを登録するコレクションを作成します。

```go
		// コレクションの定義
		classObj := &models.Class{
			Class:      "Logs",
			Vectorizer: "text2vec-ollama",
			ModuleConfig: map[string]interface{}{
				"text2vec-ollama": map[string]interface{}{ // Configure the Ollama embedding integration
					"apiEndpoint": "http://host.docker.internal:11434", // Allow Weaviate from within a Docker container to contact your Ollama instance
					"model":       "nomic-embed-text",                  // The model to use
				},
				"generative-ollama": map[string]interface{}{ // Configure the Ollama generative integration
					"apiEndpoint": "http://host.docker.internal:11434", // Allow Weaviate from within a Docker container to contact your Ollama instance
					"model":       "llama3.2",                          // The model to use
				},
			},
		}

		// コレクションをWeaviateへ登録
		err = client.Schema().ClassCreator().WithClass(classObj).Do(context.Background())
		if err != nil {
			log.Fatalln(err)
		}

```

ここで、WeaviateからOllamaへ接続するためにURLを指定します。

```
"apiEndpoint": "http://host.docker.internal:11434
```

ベクトル化のモデル`nomic-embed-text`と生成AIのモデル`llama3.2`を指定します。

## ログをコレクションに登録する

syslogなどのテキスト形式のログを読み込んで作成してWeaviateのコレクションに登録します。

```go
		objects := []*models.Object{}
		r, err := os.Open(os.Args[2])
		if err != nil {
			log.Fatalln(err)
		}
		tg, err := timegrinder.New(timegrinder.Config{
			EnableLeftMostSeed: true,
		})
		if err != nil {
			log.Fatalln(err)
		}
		tg.SetLocalTime()
		scanner := bufio.NewScanner(r)
		c := 0
		for scanner.Scan() {
			l := scanner.Text()
			ts, ok, _ := tg.Extract([]byte(l))
			if !ok {
				continue
			}
			objects = append(objects, &models.Object{
				Class: "Logs",
				Properties: map[string]any{
					"timestamp": ts,
					"log":       l,
				},
			})
			c++
			if len(objects) >= 100 {
				batchImport(client, objects)
				objects = []*models.Object{}
			}
		}
		if len(objects) > 0 {
			batchImport(client, objects)
		}

func batchImport(client *weaviate.Client, objects []*models.Object) {
	st := time.Now()
	batchRes, err := client.Batch().ObjectsBatcher().WithObjects(objects...).Do(context.Background())
	if err != nil {
		log.Fatalln(err)
	}
	for _, res := range batchRes {
		if res.Result.Errors != nil {
			log.Fatalln(res.Result.Errors.Error)
		}
	}
	log.Printf("batch end len=%d dur=%v", len(objects), time.Since(st))
}

```

まとめて登録できますが、データが多すぎるとエラーになるので、100件単位で登録するような
処理になっています。
登録するログは、ログの行とタイムスタンプです。私の環境では、１秒間に１００件ぐらいの
速度で登録できました。


##　類似検索

Weaviateのコレクションから入力したキーワードに類似したログを検索できます。
その処理は

```go
		log.Println("search")
		ctx := context.Background()

		response, err := client.GraphQL().Get().
			WithClassName("Logs").
			WithFields(
				graphql.Field{Name: "timestamp"},
				graphql.Field{Name: "log"},
			).
			WithNearText(client.GraphQL().NearTextArgBuilder().
				WithConcepts(os.Args[1:])).
			WithLimit(2).
			Do(ctx)

		if err != nil {
			log.Fatalln(err)
		}
		j, _ := json.MarshalIndent(response, "", "  ")
		fmt.Println(string(j))

```
です。

キーワードにtestを指定して実行すれば、

```json
{
  "data": {
    "Get": {
      "Logs": [
        {
          "log": "Jul  1 05:02:26 combo sshd(pam_unix)[21691]: session closed for user test",
          "timestamp": "2024-07-01T05:02:26+09:00"
        },
        {
          "log": "Jul  1 05:02:26 combo sshd(pam_unix)[21689]: session closed for user test",
          "timestamp": "2024-07-01T05:02:26+09:00"
        }
      ]
    }
  }
}
```
のような結果が帰ってきます。

## 生成AIに質問する

いわゆるRAGで生成AIにログについて質問する処理は、

```go
		log.Println("rag")
		ctx := context.Background()

		generatePrompt := os.Args[1]

		gs := graphql.NewGenerativeSearch().GroupedResult(generatePrompt)

		response, err := client.GraphQL().Get().
			WithClassName("Logs").
			WithFields(
				graphql.Field{Name: "timestamp"},
				graphql.Field{Name: "log"},
			).
			WithGenerativeSearch(gs).
			WithNearText(client.GraphQL().NearTextArgBuilder().
				WithConcepts(os.Args[2:])).
			WithLimit(2).
			Do(ctx)

		if err != nil {
			log.Fatalln(err)
		}
		j, _ := json.MarshalIndent(response, "", "  ")
		fmt.Println(string(j))
		r, err := jsonpath.Get("$..groupedResult", response.Data["Get"])
		if err != nil {
			log.Fatalln(err)
		}
		fmt.Println(r)

```

のような感じでです。
ログの概要を聞いてみると

```
[These two log entries appear to be related to a SSH (Secure Shell) session that was closed on July 1st, at 05:02:26. Here's a breakdown of the information:

**Entry 1:** ` combo sshd(pam_unix)[21689]: session closed for user test`

* The first entry indicates that a SSH session was closed for the user "test". The `[21689]` part suggests that this is a process ID (PID) associated with the SSH daemon.
* The `combo` command likely refers to a logging or monitoring tool, such as the `syslog-ng` or `rsyslog` system.

**Entry 2:** ` combo sshd(pam_unix)[21691]: session closed for user test`

* This entry is similar to the first one, but with a different PID (`[21691]`) and likely indicates that another SSH session was also closed for the same user "test".

In summary, these log entries indicate that two separate SSH sessions were closed for the user "test" on July 1st, at approximately 05:02:26.]
```

何か言ってくれます。

## コレクションを削除する

コレクションを削除する処理は

```go
		log.Println("delete collection")
		if err := client.Schema().ClassDeleter().WithClassName("Logs").Do(context.Background()); err != nil {
			if status, ok := err.(*fault.WeaviateClientError); ok && status.StatusCode != http.StatusBadRequest {
				log.Fatalln(err)
			}
		}

```
です。`Logs`がコレクションの名前です。


# ソースコード

作ったソースコードの全体は

```go

package main

import (
	"bufio"
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"time"

	"github.com/PaesslerAG/jsonpath"
	"github.com/gravwell/gravwell/v3/timegrinder"
	"github.com/weaviate/weaviate-go-client/v4/weaviate"
	"github.com/weaviate/weaviate-go-client/v4/weaviate/fault"
	"github.com/weaviate/weaviate-go-client/v4/weaviate/graphql"
	"github.com/weaviate/weaviate/entities/models"
)

func main() {
	if len(os.Args) < 2 {
		log.Fatalln("usage log cmd <args>")
	}
	cfg := weaviate.Config{
		Host:   "localhost:8080",
		Scheme: "http",
	}
	client, err := weaviate.NewClient(cfg)
	if os.Args[1] == "import" {
		if len(os.Args) < 3 {
			log.Fatalln("log import <logfile>")
		}
		log.Println("add collection")
		if err != nil {
			log.Fatalln(err)
		}

		// Define the collection
		classObj := &models.Class{
			Class:      "Logs",
			Vectorizer: "text2vec-ollama",
			ModuleConfig: map[string]interface{}{
				"text2vec-ollama": map[string]interface{}{ // Configure the Ollama embedding integration
					"apiEndpoint": "http://host.docker.internal:11434", // Allow Weaviate from within a Docker container to contact your Ollama instance
					"model":       "nomic-embed-text",                  // The model to use
				},
				"generative-ollama": map[string]interface{}{ // Configure the Ollama generative integration
					"apiEndpoint": "http://host.docker.internal:11434", // Allow Weaviate from within a Docker container to contact your Ollama instance
					"model":       "llama3.2",                          // The model to use
				},
			},
		}

		// Add the collection
		err = client.Schema().ClassCreator().WithClass(classObj).Do(context.Background())
		if err != nil {
			log.Fatalln(err)
		}
		// Retrieve the data
		log.Println("start import log")
		st := time.Now()
		objects := []*models.Object{}
		r, err := os.Open(os.Args[2])
		if err != nil {
			log.Fatalln(err)
		}
		tg, err := timegrinder.New(timegrinder.Config{
			EnableLeftMostSeed: true,
		})
		if err != nil {
			log.Fatalln(err)
		}
		tg.SetLocalTime()
		scanner := bufio.NewScanner(r)
		c := 0
		for scanner.Scan() {
			l := scanner.Text()
			ts, ok, _ := tg.Extract([]byte(l))
			if !ok {
				continue
			}
			objects = append(objects, &models.Object{
				Class: "Logs",
				Properties: map[string]any{
					"timestamp": ts,
					"log":       l,
				},
			})
			c++
			if len(objects) >= 100 {
				batchImport(client, objects)
				objects = []*models.Object{}
			}
		}
		if len(objects) > 0 {
			batchImport(client, objects)
		}
		log.Printf("end import count=%d dur=%v", c, time.Since(st))
	} else if os.Args[1] == "destroy" {
		log.Println("delete collection")
		if err := client.Schema().ClassDeleter().WithClassName("Logs").Do(context.Background()); err != nil {
			if status, ok := err.(*fault.WeaviateClientError); ok && status.StatusCode != http.StatusBadRequest {
				log.Fatalln(err)
			}
		}
	} else if os.Args[1] == "search" {
		if len(os.Args) < 3 {
			log.Fatalln("log search <key>...")
		}
		log.Println("search")
		ctx := context.Background()

		response, err := client.GraphQL().Get().
			WithClassName("Logs").
			WithFields(
				graphql.Field{Name: "timestamp"},
				graphql.Field{Name: "log"},
			).
			WithNearText(client.GraphQL().NearTextArgBuilder().
				WithConcepts(os.Args[1:])).
			WithLimit(2).
			Do(ctx)

		if err != nil {
			log.Fatalln(err)
		}
		j, _ := json.MarshalIndent(response, "", "  ")
		fmt.Println(string(j))

	} else if os.Args[1] == "rag" {
		if len(os.Args) < 4 {
			log.Fatalln("log rag <prompt> <key>...")
		}
		log.Println("rag")
		ctx := context.Background()

		generatePrompt := os.Args[1]

		gs := graphql.NewGenerativeSearch().GroupedResult(generatePrompt)

		response, err := client.GraphQL().Get().
			WithClassName("Logs").
			WithFields(
				graphql.Field{Name: "timestamp"},
				graphql.Field{Name: "log"},
			).
			WithGenerativeSearch(gs).
			WithNearText(client.GraphQL().NearTextArgBuilder().
				WithConcepts(os.Args[2:])).
			WithLimit(2).
			Do(ctx)

		if err != nil {
			log.Fatalln(err)
		}
		j, _ := json.MarshalIndent(response, "", "  ")
		fmt.Println(string(j))
		r, err := jsonpath.Get("$..groupedResult", response.Data["Get"])
		if err != nil {
			log.Fatalln(err)
		}
		fmt.Println(r)
	}
}

func batchImport(client *weaviate.Client, objects []*models.Object) {
	st := time.Now()
	batchRes, err := client.Batch().ObjectsBatcher().WithObjects(objects...).Do(context.Background())
	if err != nil {
		log.Fatalln(err)
	}
	for _, res := range batchRes {
		if res.Result.Errors != nil {
			log.Fatalln(res.Result.Errors.Error)
		}
	}
	log.Printf("batch end len=%d dur=%v", len(objects), time.Since(st))
}


```

です。


# 余談

この記事で実験したプログラムは、

https://twsnmp.github.io/TWLogAIAN/


https://github.com/twsnmp/TWLogAIAN

に組み込みました。

https://github.com/twsnmp/twsla

にも組み込んでいます。

どちらも改善の余地があります。
