---
title: >-
  苦手なPythonだけどGemini CLIと強力タッグ！血糖測定器「FreeStyle Precision
  Neo」のデータ可視化デスクトップアプリを開発し、GitHub Actionsでインストーラー自動ビルドまで実現した話
tags:
  - Python
  - Gemini
  - pywebview
  - GitHubActions
  - 血糖値
private: false
updated_at: '2026-05-24T09:37:51+09:00'
id: 7a35242cd3569996182a
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

血糖測定器のデータをPCで管理したいけれど、**「公式ソフトウェアが最新のOSで動かなくなってしまった」「健康データ（ヘルスケアデータ）をクラウドではなく、安全なローカル環境だけで管理したい」**と悩んでいる方は多いのではないでしょうか？

そんな個人的な課題を解消するために開発したのが、オープンソースのデスクトップアプリケーション **[tw-precision-neo](https://github.com/twsnmp/tw-precision-neo)** です。

普段はGo言語やTypeScriptをメインに開発しており、**Pythonは正直苦手…**という筆者が、**GoogleのGemini CLI**という強力なパートナーと協力し、さらに**初めてGitHub Actionsを使ったWindows用の `.msi` インストーラー自動ビルド**に挑んだ開発エピソードをお届けします！

---

# 開発したソフトウェア：『tw-precision-neo』

今回開発した **tw-precision-neo** は、Abbott（アボット）社の血糖測定器**「FreeStyle Precision Neo（または海外版のOptium Neo）」**からMicro-USBケーブル経由で測定データを取得し、PC上で美しく可視化・管理できるローカルファーストなデスクトップアプリです。

| 血糖値の推移グラフ | ヒートマップ分析 |
| :---: | :---: |
| ![血糖値推移](https://raw.githubusercontent.com/twsnmp/tw-precision-neo/main/images/time.-ja.png) | ![ヒートマップ](https://raw.githubusercontent.com/twsnmp/tw-precision-neo/main/images/heatmap-ja.png) |

### 主な特徴と技術構成
* **プライバシー最優先**: 血糖データは一切クラウドや外部サーバーへ送信しません。データはすべてPC内のローカル **SQLite** データベースに保存されます。
* **最新OS対応**: Apple Silicon (M1/M2/M3) 搭載Mac、およびWindows 11に対応しています。
* **単位の自動変換**: 日本国内で一般的な `mg/dL` と、海外で使われる `mmol/L` の両方に対応。
* **インタラクティブな推移グラフ**: **ECharts** を使用し、スムーズでズーム可能な血糖値の時系列グラフを表示。
* **臨床指標の自動計算**: 管理状態の可視化に重要な **TIR（Time In Range: 目標範囲内時間）** や統計データを自動で算出します。

![統計情報](https://raw.githubusercontent.com/twsnmp/tw-precision-neo/main/images/stats-ja.png)

---

# エピソード1：Pythonにしか対応ライブラリがない！という「制約」から始まった開発

普段、私の個人開発（TWSNMPシリーズなど）は**Go言語**や**TypeScript**がメインです。Pythonはエコシステムやライブラリの扱いに慣れておらず、正直「苦手意識」がありました。

Go言語でサクッと作りたかったのですが、デバイスとの連携部分で大きな壁にぶつかりました。
**「FreeStyle Precision NeoとUSB経由で通信できる唯一のオープンソースライブラリが、Python用の `glucometerutils`（`fsprecisionneo` ドライバ内蔵）しか存在しない」** という現実です。

デバイス独自のUSB通信仕様を他言語でゼロから移植するのは、あまりに骨が折れます。「ここはもう、苦手だろうが何だろうがPythonで書くしかない！」と腹をくくり、Pythonでのデスクトップアプリ開発を決意しました。

---

# エピソード2：苦手なPython開発を劇的に変えてくれた「Gemini CLI」の存在

不慣れな言語での開発は、通常ならドキュメント探しや構文エラーのデバッグで膨大な時間を消費してしまいます。しかし、今回は最強の相棒である **Gemini CLI** が常に隣に伴走してくれました。

### Gemini CLIが助けてくれた具体的なポイント

#### 1. pywebview と Python/JS ブリッジの実装
軽量デスクトップフレームワーク **`pywebview`** を採用し、フロントエンドはHTML/CSS/JSでリッチに描画する方針にしました。
Python側からJS側へのデータ受け渡しや、非同期処理をまたぐブリッジ設計で悩んだ際、Gemini CLIに以下のように相談しました。

> 「Pythonのpywebviewを使って、JSからSQLiteにアクセスしてデータを取得し、EChartsでのグラフ描画用にフロントに返すためのセキュアなブリッジのコードを書いてください」

これによって、スレッド制御や例外処理を含んだ安定したブリッジコードが一瞬で完成しました。

#### 2. SQLiteによるローカル保存の設計
血糖値データ（測定日時、血糖値、血中ケトン体など）を保存するSQLiteのスキーマ設計や、初回起動時のテーブル自動生成処理もGeminiに設計してもらいました。最適なインデックス設定やSQLクエリも正確に出力されました。

#### 3. ECharts.js や DataTables.js との連携
「苦手なPython」でバックエンドはデータ処理とUSB通信に専念させ、フロントエンドはJavaScriptライブラリをフル活用しました。
EChartsを使った滑らかなズーム可能グラフ、DataTables.jsを使ったソート機能付き履歴テーブルなど、Python側から渡すJSONのフォーマット調整をGeminiが瞬時に行ってくれたため、統合も非常にスムーズでした。

> **💡 実感したこと**
> AIをペアプログラマーとして頼ることで、「使い慣れていない言語」であっても、言語特有の落とし穴（ライブラリ特有の記述ルールやスレッド競合など）を簡単に回避し、本職の言語と変わらないスピードでアプリケーションを完成させることができました。

---

# エピソード3：初めての挑戦！GitHub ActionsでWindows用インストーラーを自動ビルド

アプリケーションが完成した後は、配布用のパッケージング（インストーラーの作成）という課題が待っていました。
今回は、プラットフォームの特性やセキュリティ、ビルド方法に合わせて以下のようにデプロイ設計を行いました。

### Mac版とWindows版のパッケージング戦略

#### macOS版：ローカルで安全にビルド＆署名
macOSアプリはAppleによるコード署名（Developer ID Application）と公証（Notarization）が必要です。セキュリティ保持の観点から、署名に必要な秘密鍵を手元の安全なMacに閉じ込めるため、ローカルの `mise` タスク (`mise run release-mac`) を使って署名・公証を行い、使い慣れた `.pkg` 形式のインストーラーをビルドする形にしました。

#### Windows版：GitHub Actionsで自動ビルド
Windows向けには、リポジトリにプッシュすると **GitHub Actions** が動き、自動でパッケージングを行ってくれる仕組みに初めて挑戦しました。

1. リポジトリにコードがプッシュされるとワークフローが起動。
2. Windowsランナー (`windows-latest`) 上で、Python環境のセットアップとPyInstallerを用いたビルドが実行される。
3. 自動的にWindows用インストーラー作成ツールが動作し、インストールが簡単な **`.msi` 形式のインストーラー** およびポータブルな `.zip` パッケージを生成する。

初めて構築するインストーラー自動ビルドのGitHub Actionsワークフローは、依存パッケージの同梱漏れやパス指定エラーなどで何度も修正が必要になりましたが、その度にエラーログをGemini CLIに投げてトラブルシューティングを行い、無事に自動ビルドを稼働させることができました！

---

# おわりに ＆ インストール方法

苦手なPythonでの開発、そして初めてのインストーラー作成・GitHub Actions構築という技術的な挑戦がありましたが、**Gemini CLIというAIのサポート**のおかげで、アイデアから実用的なデスクトップアプリのリリースまで、挫折することなく駆け抜けることができました。

もし「FreeStyle Precision Neo/Optium Neo」を使っていて、最新OSでのローカルデータ管理に困っている方がいれば、ぜひダウンロードして使ってみてください！

### インストール方法

最新バージョンを [GitHub Releases](https://github.com/twsnmp/tw-precision-neo/releases) ページからダウンロードできます。

* **macOS**:
  `.pkg` インストーラーをダウンロードして実行してください。アプリはAppleによる公証(Notarized)を受けており、安全にインストール可能です。
  
* **Windows**:
  `.msi` インストーラーをダウンロードして実行するか、`.zip` ファイルをダウンロードして任意のフォルダに解凍してください。

### プロジェクトへのアクセス
オープンソースソフトウェア（OSS）として公開しています。バグレポートや機能提案、スター（★）もお待ちしております！

* **GitHubリポジトリ**: [twsnmp/tw-precision-neo](https://github.com/twsnmp/tw-precision-neo)

---
**参考リンク**
* [Abbott FreeStyle Precision Neo 公式](https://www.myfreestyle.jp/)
* [glucometerutils (GitHub)](https://github.com/glucometers-tech/glucometerutils)
* [pywebview 公式ドキュメント](https://pywebview.flowrl.com/)
