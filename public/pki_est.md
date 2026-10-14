---
title: 'Go言語でPKI入門（勝手に復刻）: EST（RFC 7030）編'
tags:
  - Go
  - pki
  - est
  - RFC7030
  - Network
private: false
updated_at: '2026-10-14T15:36:26+09:00'
id: e0f3d67bf9a7ab09fb9b
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

Go言語でPKI入門（勝手に復刻版）シリーズの続編です。

- [第1回：CA編](https://qiita.com/twsnmp/items/60ba568f0daca8bae915)
- [第2回：CRL/OCSP編](https://qiita.com/twsnmp/items/0457d5b9ff91d037b202)
- [第3回：ACME編](https://qiita.com/twsnmp/items/bfa2254031a4f2f6b17c)
- [第4回：ACME拡張編（DNS-01・IPアドレス・端末構成証明）](https://qiita.com/twsnmp/items/cfe997182e5cb8a3913b)

前回までの記事では、Webサーバーやエンドポイント端末で広く使われるACME（RFC 8555）プロトコルとその拡張を解説しました。

今回は、ルーターやスイッチなどのネットワーク機器（Cisco、Juniper、Fortinet等）や、産業用・組込みIoTデバイスの世界でデファクトスタンダードとなっている**EST（Enrollment over Secure Transport / RFC 7030）**をGo言語で実装する方法を解説します。

---

# 1. EST（RFC 7030）とは？

## SCEPの課題とESTの登場

ネットワーク機器の証明書自動登録プロトコルとしては、古くから **SCEP（Simple Certificate Enrollment Protocol）** が使われてきました。
しかしSCEPには以下の課題がありました：

- HTTP上で独自に暗号化・署名（PKCS#7ラッパー）を行っており、プロトコルが複雑
- 最新の暗号アルゴリズム（楕円曲線暗号/ECCなど）のサポートが後手に回りがち
- 初回登録時の事前共有鍵（Challenge Password）の安全な配送が難しい

これらを近代化するために策定されたのが **EST（RFC 7030）** です。

## ESTの主な特徴

1. **HTTPS（TLS）前提のシンプル設計**:
   - 通信全体の暗号化・改ざん防止はTLSに任せるため、独自プロトコルの複雑さがありません。
2. **RESTfulなURL体系**:
   - `/.well-known/est/` 以下のパスにGET/POSTするだけです。
3. **柔軟な認証方式**:
   - **初回登録時**: ユーザー名／パスワードによるHTTP Basic認証（TLSで保護）
   - **更新時**: 既存のクライアント証明書を用いた **mTLS（相互TLS認証）**
4. **標準的なデータ形式**:
   - CSRは **PKCS#10**（Base64）
   - 返却される証明書は **PKCS#7 (`certs-only`)**（Base64）

---

# 2. ESTの主要エンドポイント

ESTサーバーが提供すべき主要なAPIは以下の通りです。
オプションで `/.well-known/est/<任意のラベル>/...` のようにマルチテナントや用途別のラベルを挟むことも可能です。

| メソッド | パス | 役割 | レスポンス形式 |
| :--- | :--- | :--- | :--- |
| `GET` | `/.well-known/est/cacerts` | Root CAおよび中間CA証明書チェーンの取得 | PKCS#7 (`certs-only`) Base64 |
| `POST` | `/.well-known/est/simpleenroll` | 新規証明書の発行（初回登録） | PKCS#7 (`certs-only`) Base64 |
| `POST` | `/.well-known/est/simplereenroll` | 既存証明書の更新 | PKCS#7 (`certs-only`) Base64 |
| `GET` | `/.well-known/est/csrattrs` | CSR生成時に要求する属性（任意） | ASN.1 Csrattrs |

---

# 3. Go言語によるESTサーバーの実装

Echoフレームワークを使った実装例で解説します。

## ルーティングの登録

標準パスとラベル付きパスの双方に対応させます。

```go
func registerESTRoutes(e *echo.Echo) {
	e.GET("/.well-known/est/cacerts", estCACerts)
	e.GET("/.well-known/est/:label/cacerts", estCACerts)
	e.POST("/.well-known/est/simpleenroll", estSimpleEnroll)
	e.POST("/.well-known/est/:label/simpleenroll", estSimpleEnroll)
	e.POST("/.well-known/est/simplereenroll", estSimpleReenroll)
	e.POST("/.well-known/est/:label/simplereenroll", estSimpleReenroll)
}
```

## ① CA証明書の配布 (`/cacerts`)

RFC 7030 Section 4.1 に基づき、CA証明書をPKCS#7の `certs-only` 構造（署名者や暗号化データのない純粋な証明書コンテナ）にし、Base64エンコードして返却します。

Content-Typeには `application/pkcs7-mime; smime-type=certs-only` を指定します。

GoでPKCS#7を扱うには、`github.com/smallstep/scep` に含まれる `DegenerateCertificates` 関数を利用すると非常に簡潔に生成できます。

```go
const estPKCS7MimeType = "application/pkcs7-mime; smime-type=certs-only"

func estCACerts(c echo.Context) error {
	ca, err := x509.ParseCertificate(rootCACertificate)
	if err != nil {
		return c.String(http.StatusInternalServerError, err.Error())
	}

	// 証明書をPKCS#7 certs-only 形式のバイナリに変換
	p7Bytes, err := scep.DegenerateCertificates([]*x509.Certificate{ca})
	if err != nil {
		return c.String(http.StatusInternalServerError, err.Error())
	}

	return sendESTPKCS7Response(c, p7Bytes)
}

func sendESTPKCS7Response(c echo.Context, p7Der []byte) error {
	c.Response().Header().Set("Content-Type", estPKCS7MimeType)
	c.Response().Header().Set("Content-Transfer-Encoding", "base64")
	
	// RFC 7030 に従いBase64エンコードして返却
	encoded := base64.StdEncoding.EncodeToString(p7Der)
	return c.String(http.StatusOK, encoded)
}
```

## ② 証明書発行 (`/simpleenroll`) と更新 (`/simplereenroll`)

クライアントから送られてくるPOSTボディには、Base64エンコードされた PKCS#10 CSR が含まれています。

処理の流れは以下の通りです：

1. **CSRのデコードと検証**:
   - Base64デコード（URL-safe/Standard両対応、途中の改行除去）
   - `x509.ParseCertificateRequest` でパース
   - `csr.CheckSignature()` でCSR自身の署名検証
2. **クライアント認証（mTLS または HTTP Basic認証）**:
   - リクエストのTLS接続にクライアント証明書が含まれていれば検証（mTLS）
   - 証明書がない場合は `Authorization: Basic ...` ヘッダーを検証
   - 未認証なら `401 Unauthorized` と `WWW-Authenticate: Basic realm="EST"` を返却
3. **証明書の発行と返却**:
   - CAでCSRに署名してX.509証明書を生成
   - PKCS#7 Base64形式にラップして返却

```go
func handleESTEnroll(c echo.Context, isReenroll bool) error {
	body, err := io.ReadAll(io.LimitReader(c.Request().Body, 1<<20)) // 1MB制限
	if err != nil {
		return c.String(http.StatusBadRequest, err.Error())
	}

	// Base64デコード
	csrBytes, err := decodeESTCSR(body)
	if err != nil {
		return c.String(http.StatusBadRequest, "invalid CSR format")
	}

	// CSRのパースと署名検証
	csr, err := x509.ParseCertificateRequest(csrBytes)
	if err != nil || csr.CheckSignature() != nil {
		return c.String(http.StatusBadRequest, "invalid CSR signature")
	}

	// 対象機器（ノード）の特定
	node := findNodeForCSR(csr, c.RealIP())
	if node == nil {
		return c.String(http.StatusBadRequest, "node not found")
	}

	// 認証チェック（mTLS または HTTP Basic認証）
	authMethod, err := authenticateESTRequest(c, node, isReenroll)
	if err != nil {
		// ※libest対策：ヘッダーのキー文字列の大文字小文字を維持してセット
		c.Response().Header()["WWW-Authenticate"] = []string{`Basic realm="EST"`}
		return c.String(http.StatusUnauthorized, err.Error())
	}

	// CAによる証明書発行
	crtBytes, err := issueCertFromCSR(csr)
	if err != nil {
		return c.String(http.StatusInternalServerError, err.Error())
	}
	crt, _ := x509.ParseCertificate(crtBytes)

	// PKCS#7 Base64 で返却
	p7, _ := scep.DegenerateCertificates([]*x509.Certificate{crt})
	return sendESTPKCS7Response(c, p7)
}
```

## ③ mTLSとHTTP Basic認証の分岐ロジック

ESTの素晴らしい点は、初回登録と更新のセキュリティモデルが綺麗に分かれているところです。

- **新規登録時（初回）**: まだ証明書を持っていないため、機器に事前設定された管理者パスワード等による HTTP Basic認証 を利用。
- **更新時（Re-enroll）**: まもなく期限切れになる既存の有効なクライアント証明書を提示して mTLS で認証。

Go言語の `net/http` / Echo では以下のように判定できます。

```go
func authenticateESTRequest(c echo.Context, node *Node, isReenroll bool) (string, error) {
	// 1. mTLSクライアント証明書が提示されている場合
	if reqTLS := c.Request().TLS; reqTLS != nil && len(reqTLS.PeerCertificates) > 0 {
		clientCert := reqTLS.PeerCertificates[0]
		
		// 自局CAで署名された有効な証明書か検証
		opts := x509.VerifyOptions{
			Roots: rootCAPool,
			KeyUsages: []x509.ExtKeyUsage{x509.ExtKeyUsageClientAuth},
		}
		if _, err := clientCert.Verify(opts); err != nil {
			return "", fmt.Errorf("mTLS verification failed: %w", err)
		}
		// 失効チェック（CRL/OCSP）やノード所有権チェック...
		return "mtls", nil
	}

	// 2. クライアント証明書がない場合は HTTP Basic 認証
	user, pass, hasBasic := c.Request().BasicAuth()
	if !hasBasic {
		return "", fmt.Errorf("authentication required (mTLS or HTTP Basic)")
	}

	// ノードの登録パスワードと突合
	if subtle.ConstantTimeCompare([]byte(pass), []byte(node.Password)) != 1 {
		return "", fmt.Errorf("invalid password")
	}
	return "basic", nil
}
```

---

# 4. 実装上のTips & 落とし穴

### ① C言語製クライアント（`libest`）との互換性問題
Ciscoが公開しているオープンソースのEST参照実装 `libest` では、401応答を処理する際に以下のような判定を行っています。

```c
// libest のソースコード抜粋例
if (strncmp(header, "WWW-Authenticate", 16) == 0) ...
```

一般的なGoのHTTPライブラリ（`Header.Set` や HTTP/2）はヘッダー名をキャメルケース（`Www-Authenticate`）に正規化することがあります。これによって `libest` がヘッダーを認識できず認証エラーになるケースがあります。
対策として、Goの `Header` マップに直接大文字小文字を指定して書き込む手法が有効です。

```go
c.Response().Header()["WWW-Authenticate"] = []string{`Basic realm="EST"`}
```

### ② PKCS#10 CSRのBase64デコード
クライアントによっては、PEMヘッダー（`-----BEGIN CERTIFICATE REQUEST-----`）を付けたまま送ってくるもの、純粋なBase64文字列を送ってくるもの、途中に改行（`\r\n`）を含むものなど様々です。
Go側で受信する際は、PEMヘッダーの有無判定と空白・改行文字の除去を行ってからBase64デコードを行うと堅牢になります。

---

# 5. curlによる動作テスト

実装したESTサーバーは `curl` や `openssl` で簡単にテストできます。

### CA証明書の取得
```bash
curl -k https://<server-ip>:8443/.well-known/est/cacerts | base64 -d | openssl pkcs7 -inform DER -print_certs -text
```

### 新規証明書の発行（HTTP Basic認証）
```bash
# 秘密鍵とCSRを作成
openssl ecparam -name prime256v1 -genkey -out client.key
openssl req -new -key client.key -out client.csr -subj "/CN=router01.local"

# CSRをBase64化してPOST（Basic認証）
cat client.csr | openssl req -outform DER | base64 | \
  curl -k -X POST https://<server-ip>:8443/.well-known/est/simpleenroll \
    -u "admin:secretpassword" \
    -H "Content-Type: application/pkcs10" \
    --data-binary @- | base64 -d | openssl pkcs7 -inform DER -print_certs -out client.crt
```

---

# まとめ

ACMEがWebホスティングの世界で圧倒的な地位を築いたのに対し、**EST（RFC 7030）** はエンタープライズのネットワーク機器やIoT/産業機器分野における標準プロトコルとして広く普及しています。

プロトコル自体がHTTPS上に構築されているため、Go言語の `net/http` や `echo` を使えば、非常にシンプルかつ安全にサーバーを実装できます。

社内ネットワーク管理ツール（[TWSNMP FK](https://github.com/twsnmp/twsnmpfk)）のようなソフトウェアに組み込むことで、ネットワークに接続されたスイッチやルーターへゼロタッチで証明書を配備できるようになります。

### 参考ソースコード
実装の全容は TWSNMP FK のリポジトリで公開しています。

- https://github.com/twsnmp/twsnmpfk （`pki/est.go`, `pki/ca.go`）
