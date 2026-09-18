---
title: Go言語製ツールにローカルLLM（tensai）を組み込んで「完全オフライン動作するAI内蔵ネットワーク管理」を実現した話
tags:
  - Go
  - LLM
  - tensai
  - ネットワーク監視
  - 機械学習
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# はじめに

ネットワーク機器の監視やサーバーのログ分析において、生成AI（LLM）のサポートは非常に強力です。
「このSyslogエラーはどういう意味？」「昨夜発生したトラフィック急増の原因候補は？」「このログを検知する正規表現はどう書けばいい？」といった疑問に対し、AIは的確なアドバイスを返してくれます。

しかし、ネットワーク管理やログ分析でAIを活用しようとすると、必ず大きな壁にぶつかります。

1. **セキュリティと機密性**: IPアドレスやホスト名、機器構成、個人情報（PII）を含むログを外部クラウドAPI（OpenAIやGeminiなど）に送信できない（閉域網や社内規定の壁）。
2. **ローカルLLMの導入ハードル**: Ollamaなどを別途インストール・常駐・ポート設定してもらうのは、一般ユーザーや現場の運用担当者にとって敷居が高い。

**「外部クラウドも不要、別途Ollama等のサーバーを立てる必要もなく、アプリ自身がローカルLLMを直接内蔵してワンストップで動かせないか？」**

この課題を解決するため、自作のネットワーク監視・ログ分析ツール群（**TWSNMP FK**, **TWSLA**, **TWLogAIAN**, **twlogeye**）に、mattn氏が開発するGo言語製推論エンジン **[tensai](https://github.com/mattn/tensai)** を直接組み込みました。

本記事では、Go言語アプリにローカルLLMをインプロセスで組み込んだ構成、**モデルやGPUライブラリ（wgpu-native）をアプリ自身が自動ダウンロード・管理する仕組み**、そして**ログやレポートのAI解説**といった実践的な機能について紹介します。

---

# tensaiパッケージによるローカルLLMの直接組み込み

## なぜ tensai なのか？

Go言語でローカルLLMを動かすアプローチとしては、llama.cppのCgoバインディングやOllamaとのHTTP通信が一般的でした。しかし、Cgoによるビルドの複雑化や外部プロセスの依存は、クロスプラットフォーム配布（Windows/macOS/Linux）において大きな障壁となります。

mattn氏が開発されている **`github.com/mattn/tensai`** は、Go言語から軽量に扱える機械学習・ニューラルネットワーク・LLM推論エンジンです。

* **GGUFモデルのネイティブロード・推論**: 量子化されたGGUFフォーマットのLLMを直接読み込んで推論を実行可能。
* **WebGPU（wgpu）によるGPUアクセラレーション**: Metal（macOS）、Vulkan / Direct3D 12（Windows）、Vulkan（Linux）を活用したGPU高速化に対応。
* **ニューラルネットワーク機能**: AutoencoderやLSTMなどのレイヤー・モデル定義、学習・推論もサポート。

これをGoアプリケーションの依存関係（`go get github.com/mattn/tensai`）として組み込むことで、**Goの単一プロセス内で完全に完結する「AI内蔵ネットワーク管理ソフト」** が実現しました。

---

# 今回ローカルAIを組み込んだツール群

今回、以下の4つのツールに `tensai` によるローカルAI連携を組み込みました。

| ツール名 | 種別 | tensaiの活用ポイント |
|---|---|---|
| **[TWSNMP FK](https://github.com/twsnmp/twsnmpfk)** | 統合監視デスクトップGUI (Wails) | ノード総合診断、Syslogポーリング作成、レポート要約、Ping/Smokeping診断、モデル・GPU管理GUI |
| **[TWSLA](https://github.com/twsnmp/twsla)** | 高速CLIログ分析ツール (Cobra/TUI) | `twsla ai` によるログ傾向・エラー分析、TUI画面での個別ログ解説（`e`キー）と全体要約（`a`キー）、モデル管理CLI |
| **[TWLogAIAN](https://github.com/twsnmp/TWLogAIAN)** | ログ分析デスクトップアプリ (Wails) | 検索結果ログのAI要約・トラブルシューティング、対話型Ask AI機能、異常検知 |
| **[twlogeye](https://github.com/twsnmp/twlogeye)** | 軽量ログサーバー＆監視 (Badger/Parquet) | tensaiのニューラルネットワーク（Autoencoder, LSTM）を用いたリアルタイム多変量ログ異常検知 |

---

# ポイント1: モデルとGPUライブラリの自動ダウンロード機能

ローカルLLMを内蔵する上で最大の問題となるのが、**「モデルファイル（数百MB〜数GB）」や「GPUアクセラレーション用ライブラリ」をどうやってユーザー環境に届けるか** です。
インストーラーにすべて同梱すると配布サイズが巨大化してしまいます。

そこで、各ツールに **「必要なモデルやGPUライブラリをオンデマンドでダウンロードして自動構成するマネージャー機能」** を実装しました。

### 1. GGUFモデルのプリセット管理とダウンロード

ネットワーク監視・ログ解析の実務で扱いやすい軽量モデル（0.5B〜1.5Bクラス）を中心に、Hugging Faceからのワンクリック/コマンドダウンロードに対応させました。

**主なプリセットモデル例:**
* **`qwen2.5-coder-0.5b` / `1.5b`**: ログ構造やコード、設定スクリプトの解析に強い。
* **`qwen2.5-0.5b` / `1.5b`**: 日本語を含む汎用対話・要約が高品質。
* **`smollm2-360m` / `1.7b`**: 超軽量（360Mは約380MB）で、CPU環境でも超高速に動作。
* **`deepseek-r1-1.5b`**: `<think>` による推論ステップを踏む推論特化型モデル。

#### CLIツール（TWSLA）での操作例:
```terminal
# ハードウェア状況（GPU利用可否など）の確認
$ twsla model status

# プリセットモデル一覧の表示
$ twsla model presets

# プリセットからモデルをダウンロード
$ twsla model download qwen2.5-coder-0.5b

# ダウンロード済みモデルの一覧
$ twsla model list
```

#### GUIツール（TWSNMP FK / TWLogAIAN）での操作:
設定画面内の「モデル管理」ダイアログから、進捗バーを見ながらワンクリックでモデルの取得・切り替え・削除が行えます。

![TWSNMP FKのLLMプロバイダ設定画面](https://raw.githubusercontent.com/twsnmp/twsnmpfk/main/docs/images/ja/llm_setting.png)
*▲ LLMプロバイダで「Local (tensai)」を選択（モデル管理ボタンが有効化）*

![TWSNMP FKのローカルモデル管理ダイアログ](https://raw.githubusercontent.com/twsnmp/twsnmpfk/main/docs/images/ja/llm_local_model.png)
*▲ ローカルモデル管理ダイアログ：推奨プリセットモデルのワンクリックダウンロードやGPUライブラリの管理が可能*

### 2. GPUアクセラレーション（wgpu-native）の自動配備

GPUによる高速推論を有効化するには `wgpu-native` の共有ライブラリが必要です。しかし、ユーザーに「GitHubから対応するOS・アーキテクチャのdylib/dll/soを探して配置してください」と求めるのは酷です。

そこで、アプリ実行環境のOS（`runtime.GOOS`）とCPU（`runtime.GOARCH`）を自動判別し、公式リリース（`github.com/gfx-rs/wgpu-native`）のZIPアーカイブから必要なライブラリファイルだけを展開・配置するダウンローダーを実装しました。

```go
// OS・アーキテクチャに応じたアセットを自動特定
switch runtime.GOOS {
case "darwin":
    libFileName = "libwgpu_native.dylib"
    if runtime.GOARCH == "arm64" { zipName = "wgpu-macos-aarch64-release.zip" }
case "linux":
    libFileName = "libwgpu_native.so"
    // ...
case "windows":
    libFileName = "wgpu_native.dll"
    // ...
}
```

CLIなら `twsla model download-gpu`、GUIならダイアログの「GPUライブラリをダウンロード」ボタンを押すだけで、即座にMetal/Vulkan/Direct3D 12によるGPU推論が有効化されます。

---

# ポイント2: ログやレポートのAI解説・アシスト機能

内蔵されたローカルLLMは、ネットワーク運用の現場で具体的にどう役立つのか？
以下のような機能として各ツールに組み込んでいます。

## 1. 難解な個別ログの即座解説

SyslogやSNMP Trap、Windowsイベントログには、英語のエラーメッセージや数字のエラーコードが並びます。
TWSLAのTUI画面やTWSNMP FKのログ一覧で気になる行を選択してAI解説を呼び出すと、ローカルLLMが以下を瞬時に提示します。

* **ログの意味**: 何が起きたのか（認証失敗、インターフェースDown、リソース枯渇など）
* **考えられる原因**: 設定ミス、ハード故障、外部からの不正アクセス試行など
* **推奨される初動対応**: 次に確認すべきコマンドやログ、確認手順

```
【AIによるログ解説例】
ログ: "SSHD: Failed password for invalid user admin from 192.168.1.50 port 49152 ssh2"
解説: 
外部ホスト(192.168.1.50)から存在しないユーザー 'admin' に対するSSH総当たり攻撃の試行が検知されました。
対応: 当該IPアドレスの接続制限（ファイアウォール/TCP Wrapper）およびSSHの公開鍵認証への制限を推奨します。
```

※TWSLAでは、送信前にIPアドレスやメールアドレス等の個人情報（PII）を自動マスキングするフィルターも備えており、安全性を二重に高めています。

![TWSLAでのAIログ解説](https://raw.githubusercontent.com/twsnmp/twsla/main/images/ai.png)
*▲ TWSLAのTUIログ検索画面で選択したログをAIが即座に解説*

![TWSNMP FKのイベントログAI解説](https://raw.githubusercontent.com/twsnmp/twsnmpfk/main/docs/images/ja/eventlog_ai.png)
*▲ TWSNMP FKのイベントログ画面で選択したログのAI解説*

![TWSNMP FKのノードAI総合診断](https://raw.githubusercontent.com/twsnmp/twsnmpfk/main/docs/images/ja/node_ai_diag.png)
*▲ ノードの状態や直近24時間のログをまとめてAIが総合診断*

## 2. 大量ログや各種レポートの要約・インサイト抽出

単一のログだけでなく、集計レポート全体の傾向分析もAIが担当します。

* **TWSLAの全体分析（`a`キー / `twsla ai`）**:
  検索でヒットした大量のログ（最大数十〜数百件のサンプリング）から、主要なエラーパターンや発生傾向、異常な急増（スパイク）を抽出してサマリーを生成します。
* **TWSNMP FK のレポート解説**:
  Syslog、NetFlow/sFlow、ARP、MQTT、障害通知などのレポート画面で「AI解説」を押すだけで、上位ホストやプロトコル分布の傾向、管理者が注目すべき不審な動きを箇条書きで分かりやすく整理します。
* **TWLogAIAN の対話型 Ask AI & レポート解説**:
  取り込んだログに対して自然言語で質問したり、各種統計レポートのAI解説を行えます。

![TWSNMP FK レポート画面のAI要約・解説](https://raw.githubusercontent.com/twsnmp/twsnmpfk/main/docs/images/ja/report_ai_summary.png)
*▲ TWSNMP FK レポート画面で「AI解説」を実行した様子*

![TWLogAIANのAsk AI画面](https://raw.githubusercontent.com/twsnmp/TWLogAIAN/main/docs/images/ja/ask_ai_screen.png)
*▲ TWLogAIANのAsk AI画面（ログを元に対話形式でトラブルシューティング）*

![TWLogAIANのレポートAI解説](https://raw.githubusercontent.com/twsnmp/TWLogAIAN/main/docs/images/ja/report_ai_explain.png)
*▲ TWLogAIANでのレポートAI解説*

## 3. Syslogからのポーリング監視自動作成

TWSNMP FK では、過去に届いたSyslogを選択して「AIで監視を作成」を実行すると、可変部分（プロセスIDやポート番号、送信元IP）をAIが自動的にワイルドカード・正規表現化（`.*` や `\d+`）し、TWSNMP FK のポーリングスクリプト（Otto JavaScript VM）に合わせた判定ルール（`count == 0` など）を自動生成してくれます。

![Syslogからのポーリング作成AIアシスト](https://raw.githubusercontent.com/twsnmp/twsnmpfk/main/docs/images/ja/syslog_polling_ai.png)
*▲ ログを選択してAIが正規表現と判定スクリプトを自動生成する画面*

## 4. ネットワーク品質・経路遅延のAI診断（Ping / MTR）

PINGやMTRによるパケットロスやジッター、ホップ別遅延の測定結果をワンクリックでAIに送り、ネットワーク品質やボトルネックを診断させることも可能です。

![TWSNMP FKのPing/MTR AI解説](https://raw.githubusercontent.com/twsnmp/twsnmpfk/main/docs/images/ja/map_ping_mtr_ai.png)
*▲ MTR経路測定のAI解説画面*

---

# ローカルLLM組み込みの所感・メリット

実際にツール群へ `tensai` を組み込んで運用してみて感じたメリットは以下の通りです。

1. **「完全オフライン・ゼロコンフィグ」の安心感**:
   閉域網のサーバー室や顧客先ネットワークでも、一切の外部通信なしにAI機能がフルに動作します。セキュリティ審査の厳しい現場でも胸を張って導入できます。
2. **小規模モデルの限界と将来への期待**:
   現在一般的なPC環境で軽快に動く0.5B〜1.5Bクラスの超小規模モデルでは、簡単なキーワード解釈や単一ログの概要提示程度はできても、複雑な障害ログの文脈理解や的確な原因推論においては、**残念ながら現時点では実用的とは言えません**。指示通りに動かなかったり、回答が物足りないケースも多々あります。
   しかし、PCハードウェア（NPUや統合GPU）の進化やモデルの軽量・高効率化は凄まじい勢いで進んでいます。将来、手元の一般的なPCでも7B〜8Bクラス以上の中規模・大規模モデルが快適に動作するようになれば、この「ツール内蔵型ローカルAI」の真価が本格的に発揮されるはずで、今後の進化に大いに期待しています。
3. **GPUアクセラレーションでストレスのない速度**:
   `wgpu-native` を使ったMetal（MシリーズMac）やVulkan/D3D12での推論は非常に高速で、小型モデルであれば数秒〜十数秒程度でスラスラと回答が返ってきます。内蔵推論エンジンの仕組み自体の手応えは十分得られました。

---

# おわりに

AIを組み込んだソフトウェアを開発する際、「クラウドAPIを呼び出す」または「ユーザーにOllama環境を用意してもらう」以外の第3の選択肢として、**「Go言語＋tensaiによるインプロセス内蔵」** は極めて実用的で魅力的なアプローチだと実感しました。

今回ご紹介した機能は、各ツールの最新バージョンで実際にお試しいただけます。

* **TWSNMP FK**: [GitHub リポジトリ](https://github.com/twsnmp/twsnmpfk)
* **TWSLA**: [GitHub リポジトリ](https://github.com/twsnmp/twsla)
* **TWLogAIAN**: [GitHub リポジトリ](https://github.com/twsnmp/TWLogAIAN)
* **twlogeye**: [GitHub リポジトリ](https://github.com/twsnmp/twlogeye)
* **tensai (mattn氏)**: [github.com/mattn/tensai](https://github.com/mattn/tensai)

ネットワーク管理やログ分析の現場で「閉域網だけどAIを活用したい」「スタンドアロンで動くAIツールを作りたい」と考えている方の参考になれば幸いです！
