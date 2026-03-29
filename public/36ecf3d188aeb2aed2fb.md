---
title: Go言語のベンチマーク結果をgo-echartsでグラフ表示するプログラムの紹介
tags:
  - Go
  - benchmark
  - ECharts
private: false
updated_at: '2024-12-13T07:30:02+09:00'
id: 36ecf3d188aeb2aed2fb
organization_url_name: null
slide: false
ignorePublish: false
---
#　はじめに

Go言語のベンチマーク結果をグラフにするプログラムを作ってみました。その紹介です。


![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3874862/e36c984d-4852-d340-dcf8-36292721653a.png)


のようなHTMLのグラフを作成できます。
HTMLのJavaScriptで表示しているので、インタラクティブに表示を切り替えできます。

![2024-12-13_06-54-00 (1).gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3874862/9a3b5fcc-f9b7-b1ed-01a8-3120e414aa72.gif)

# 表示するGo言語のベンチマーク結果

```terminal
$go test -bench .  -benchmem
goos: darwin
goarch: arm64
pkg: lotusdb_test
Benchmark_bbolt_seq-10       	  493197	      2673 ns/op	    8609 B/op	      57 allocs/op
Benchmark_lotusdb_seq-10     	  433036	      2941 ns/op	    3434 B/op	      40 allocs/op
Benchmark_badger_seq-10      	  785300	      1590 ns/op	    1647 B/op	      18 allocs/op
Benchmark_bbolt_rand-10      	  155701	     48440 ns/op	    6417 B/op	      49 allocs/op
Benchmark_lotusdb_rand-10    	  394960	      3468 ns/op	    3100 B/op	      41 allocs/op
Benchmark_badger_rand-10     	  610329	      1819 ns/op	    1545 B/op	      18 allocs/op
PASS
ok  	lotusdb_test	15.933s
```

 のような結果からns/opとB/op,alllocs/opの値をグラフ化します。

# グラフ表示のパッケージ

グラフ表示に使うパッケージは、

https://github.com/go-echarts/go-echarts

です。


# プログラム

グラフ化するプログラムは

```go
package main

import (
	"bufio"
	"os"
	"strconv"
	"strings"

	"github.com/go-echarts/go-echarts/v2/charts"
	"github.com/go-echarts/go-echarts/v2/opts"
)

func main() {
	benchs := []string{}
	times := []opts.BarData{}
	bytes := []opts.BarData{}
	allocs := []opts.BarData{}
	goos := ""
	goarch := ""
	pkg := ""
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		a := strings.Fields(scanner.Text())
		switch len(a) {
		case 2:
			switch a[0] {
			case "goos:":
				goos = a[1]
			case "goarch:":
				goarch = a[1]
			case "pkg:":
				pkg = a[1]
			}
		case 8:
			if a[7] != "allocs/op" {
				continue
			}
			if v, err := strconv.Atoi(a[6]); err == nil {
				allocs = append(allocs, opts.BarData{Value: v})
			} else {
				allocs = append(allocs, opts.BarData{Value: 0})
			}
			if v, err := strconv.Atoi(a[4]); err == nil {
				bytes = append(bytes, opts.BarData{Value: v})
			} else {
				bytes = append(bytes, opts.BarData{Value: 0})
			}
			fallthrough
		case 4:
			if a[3] != "ns/op" {
				continue
			}
			b := strings.SplitN(a[0], "-", 2)
			benchs = append(benchs, strings.Replace(b[0], "Benchmark_", "", 1))
			if v, err := strconv.Atoi(a[2]); err == nil {
				times = append(times, opts.BarData{Value: v})
			} else {
				times = append(bytes, opts.BarData{Value: 0})
			}
		}
	}
	if err := scanner.Err(); err != nil {
		panic(err)
	}
	bar := charts.NewBar()
	bar.SetGlobalOptions(charts.WithTitleOpts(opts.Title{
		Title:    pkg + " Bechmark",
		Subtitle: "GOOS:" + goos + " GOARCH:" + goarch,
	}))
	bar.SetXAxis(benchs).
		AddSeries("ns/op", times).
		AddSeries("B/op", bytes).
		AddSeries("allocs/op", allocs).
		XYReversal()
	bar.Render(os.Stdout)
}
```
です。
標準入力からベンチマーク結果を読み込んで、タイトルやグラフのデータを取得した後、横棒グラフを作成して標準出力にHTMLとして出力しています。

# 使い方

```terminal

$go test -bench .  -benchmem  | gobench2chart > t.html

```

# 余談

このプログラムは、

https://note.com/twsnmp/n/nd90978e728c0

の開発中にキーバリューストアパッケージの書き込みを比較するために実施したベンチマーク結果をグラフにするために作りました。
