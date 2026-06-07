---
title: Windows標準のSNMPエージェントをSNMPv3対応させる「twsnmpv3proxy」のお試し版をリリースしました
tags:
  - Windows
  - snmp
  - Go
  - セキュリティ対策
  - TWSNMP
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

以前の記事で「[CiscoがSNMPを廃止するという噂の真相](https://qiita.com/twsnmp/items/19a123ec43f0c2143d0a)」や「[実践編：Cisco推奨のセキュリティ強度を実現するSNMPv3設定（NET-SNMP & TWSNMP）](https://qiita.com/twsnmp/items/809467d27f226c508f5e)」を紹介しました。

そこでの結論は、**「これからのSNMP監視はセキュリティの観点から SNMPv3（authPriv: 認証＋暗号化）一択である」** ということでした。

しかし、ここで多くのシステム管理者やインフラエンジニアが直面する大きな課題があります。

**「Windows標準のSNMPエージェントは、SNMPv3に対応していない（SNMPv1/v2cのみ）」**

MicrosoftはWindows標準のSNMPサービスをすでに非推奨（Deprecated）としており、今後機能がアップデートされてSNMPv3に対応する見込みは極めて低いです。
だからといって、監視のために各Windowsサーバーに別の複雑なSNMPエージェント（Net-SNMP等）を個別にインストール・設定して回るのは、運用コストの面から現実的ではありません。

そこで、Windows標準のSNMPエージェントをそのまま活かしつつ、外部との通信だけをセキュアなSNMPv3に変換するプロキシサーバー **「twsnmpv3proxy」** を開発しました！最初のお試し版（v0.0.1）をリリースしましたので紹介します。

https://github.com/twsnmp/twsnmpv3proxy

---

# twsnmpv3proxy の仕組み

仕組みはいたってシンプルです。Windowsサーバーの内部でプロキシとして動作します。

1. 外部の監視マネージャー（TWSNMP FC/FKなど）から、暗号化・認証された **SNMPv3 (authPriv)** リクエストを `twsnmpv3proxy` が受信します。
2. `twsnmpv3proxy` はそのリクエストを復号・検証し、対応する **SNMPv2c** リクエストに変換して、同じWindowsサーバーのローカル（127.0.0.1）で動いているWindows標準SNMPエージェントに中継します。
3. Windows標準SNMPエージェントから返ってきた SNMPv2c 応答を、`twsnmpv3proxy` が再び SNMPv3 (authPriv) に暗号化・署名して監視マネージャーへ返します。

```text
[監視マネージャー (SNMPv3 authPriv)]
             │
             │ (インターネット/LAN経由の暗号化・認証通信)
             ▼
┌──────────────── Windows Server ────────────────┐
│                                                │
│  [twsnmpv3proxy] (UDP:161ポートで待機)         │
│            │                                   │
│            │ (127.0.0.1 ローカル経由の v2c)     │
│            ▼                                   │
│  [Windows標準SNMPサービス] (UDP:1161ポート等)  │
│                                                │
└────────────────────────────────────────────────┘
```

この仕組みにより、外部のネットワーク上には安全なSNMPv3パケットしか流れないため、セキュリティポリシーの厳しい環境でも安全にWindowsサーバーを監視できるようになります。

---

# 主な機能と特徴

* **セキュアなSNMPv3対応（gosnmp準拠）**
  * 認証アルゴリズム： MD5, SHA, SHA-224, SHA-256, SHA-384, SHA-512
  * 暗号アルゴリズム： DES, AES, AES-192, AES-256 など
  * Ciscoや主要ベンダーが推奨する「SHA-256 + AES-256」の最高セキュリティ設定（authPriv）にも完全に対応しています。
* **主要なSNMP操作のサポート**
  * `GET`, `GETNEXT`, `GETBULK` の主要なリクエストに対応しています。
* **Windowsサービス対応**
  * Windows環境下でOSのサービスとして登録、起動、停止、解除が可能です。サーバー起動時に自動で立ち上がるため、運用の手間がかかりません。
* **自動EngineID生成・永続化**
  * SNMPv3で必須となる「EngineID」を未設定時に自動生成（IANA企業番号 `17862` ＋ランダム値）します。
  * 生成されたEngineIDや起動回数（Boots）は設定ファイル（`config.ini`）に保存されて永続化されるため、再起動しても同じEngineIDを維持できます。
* **接続自動回復**
  * 背後のWindows標準SNMPエージェントとの通信エラー（ソケット切断やBroken pipe等）が発生した場合、自動でソケットを再作成し、指数バックオフウェイトを挟んで最大5回リトライします。

---

# クイックスタート (お試し方法)

Go環境があれば、簡単に動作検証ができる「統合テストモード」を搭載しています。

`-test` 引数を指定して実行すると、ローカルに擬似的なSNMPv2cエージェント（ポート1162）とSNMPv3プロキシ（ポート1161）を同時に立ち上げ、自動的にテストクエリを実行して結果を検証します。

```bash
# 統合テストの実行
go run . -test
```

ビルド済みの実行ファイルを使用する場合は、以下のように実行できます。

```cmd
twsnmpv3proxy.exe -test
```

---

# 設定ファイル (config.ini) の例

設定ファイルは以下のように直感的に記述できます。

```ini
# プロキシの動作設定
proxy_port = 161

# SNMPv3 USM ユーザー設定
v3_user = myuser
v3_auth_proto = SHA-256
v3_auth_pass = authpass123
v3_priv_proto = AES-256
v3_priv_pass = privpass123

# 転送先（Windows標準SNMPサービスなど）の設定
agent_address = 127.0.0.1:1161
agent_community = public
```

### Windows標準SNMPエージェントとの共存方法
通常、SNMPはポート `161` を使用します。
Windows標準のSNMPエージェントのポート番号をレジストリなどで `1161` に変更し、`twsnmpv3proxy` をポート `161` で起動して外部からのSNMPv3通信を待ち受ける設定にすることで、マネージャー側（監視ソフト）の設定を変更することなくスムーズにSNMPv3へ移行可能です。

---

# おわりに

Windows標準SNMPの非推奨化に伴い、「監視をどうセキュアに継続するか」に悩んでいた方も多いのではないでしょうか。
`twsnmpv3proxy` は、既存のエージェントを活かしながら最小限の手間で安全な監視環境を実現するために開発しました。

まだ最初のお試し版（v0.0.1）をリリースしたばかりの段階ですので、うまく動作した報告や、「ここが動かない」「こんな機能がほしい」といったフィードバックをいただけると非常に励みになります！

バグレポートやご意見は、ぜひGitHubのIssueへお寄せください。

https://github.com/twsnmp/twsnmpv3proxy
