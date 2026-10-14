---
title: 'Go言語でPKI入門（勝手に復刻）: ACME拡張編（DNS-01・IPアドレス・端末構成証明）'
tags:
  - Go
  - pki
  - acme
  - ゼロトラスト
  - セキュリティ
private: false
updated_at: '2026-10-14T15:20:01+09:00'
id: cfe997182e5cb8a3913b
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

前回のACME編では、RFC 8555の基本仕様と、Echoフレームワークを用いたACMEサーバーの基本実装（アカウント登録、オーダー管理、HTTP-01チャレンジの検証、証明書発行など）を解説しました。

今回はその続編として、実際の運用やネットワーク管理ツール（[TWSNMP FK](https://github.com/twsnmp/twsnmpfk)）への組み込みにおいて直面した**ACMEの拡張機能**について解説します。

具体的には以下の3つの拡張を取り上げます。

1. **DNS-01 チャレンジ**: ワイルドカード証明書や80番ポートを開放できない環境への対応
2. **IPアドレス識別子（RFC 8738）**: ドメイン名を持たない社内機器やIoTデバイスへの証明書発行
3. **端末構成証明（Device Attestation / device-attest-01）**: AppleデバイスやYubiKeyのハードウェアセキュリティチップと連携したゼロトラスト証明書発行

---

# 1. DNS-01 チャレンジの実装

## なぜ DNS-01 が必要か？

前回の `http-01` チャレンジは、Webサーバーの `/.well-known/acme-challenge/<token>` にGETリクエストを投げて検証する最も一般的な方法でした。
しかし、以下のケースでは `http-01` を利用できません。

- **ワイルドカード証明書（`*.example.com`）**: RFC 8555上、ワイルドカード証明書は `dns-01` による検証が必須と規定されています。
- **内部ネットワークやファイアウォール内**: 外部から80番ポートへのHTTPアクセスが許可されていない機器。

## DNS-01 の仕様

`dns-01` チャレンジでは、ドメインのDNSに以下のTXTレコードを登録して検証します。

- **レコード名**: `_acme-challenge.<ドメイン名>`
- **レコード値**: `Base64URL(SHA-256(KeyAuthorization))`

クライアントがDNSにTXTレコードを配置した後、ACMEサーバーに対してチャレンジ検証要求を送付します。サーバー側はDNS検索（TXTレコードルックアップ）を行い、期待されるハッシュ値が存在すれば検証成功とします。

## Goでの実装例

Go言語の `net.LookupTXT` を利用して検証する実装です。
ワイルドカードドメインの場合、`_acme-challenge.*.example.com` ではなく `_acme-challenge.example.com` を検索する必要があるため、ドメイン名のプレフィックス `*.` をトリムします。

```go
func dns01Validate(acc *Account, ch *Challenge) error {
	// ワイルドカードドメインの場合はプレフィックス "*. " を除去
	domain := strings.TrimPrefix(ch.Value, "*.")

	// _acme-challenge.<domain> のTXTレコードを名前解決
	txtRecords, err := net.LookupTXT("_acme-challenge." + domain)
	if err != nil {
		return fmt.Errorf("error looking up TXT records for domain %s: %w", domain, err)
	}

	// 期待されるKeyAuthorization（Token + アカウント公開鍵Thumbprint）の算出
	expectedKeyAuth, err := KeyAuthorization(ch.Token, acc.Key)
	if err != nil {
		return err
	}

	// SHA-256ハッシュをBase64-URL（パディングなし）エンコード
	h := sha256.Sum256([]byte(expectedKeyAuth))
	expected := base64.RawURLEncoding.EncodeToString(h[:])

	// 取得したTXTレコード群の中に期待値が存在するか確認
	var found bool
	for _, r := range txtRecords {
		if r == expected {
			found = true
			break
		}
	}
	if !found {
		return fmt.Errorf("dns TXT record does not match; expected %s, got %v", expected, txtRecords)
	}

	ch.Status = "valid"
	ch.ValidatedAt = time.Now().UTC().Format(time.RFC3339)
	return nil
}
```

---

# 2. IPアドレス識別子への対応（RFC 8738）

## ドメイン名のないネットワーク機器に証明書を発行したい

LAN内のルーターやスイッチ、管理サーバーなどでは、DNS名が割り当てられておらず、IPアドレス（例: `192.168.1.1`）で直接HTTPS管理画面にアクセスすることが多々あります。

RFC 8555の当初の仕様では、識別子（Identifier）の `type` は `dns` のみでしたが、**RFC 8738 (Automated Certificate Management Environment (ACME) IP Identifier Validation Extension)** により `type: "ip"` が標準化されました。

## 実装のポイント

オーダー作成時（`/new-order`）に、Identifierの種別として `dns` だけでなく `ip` を受け入れられるようにします。

```go
type Identifier struct {
	Type  string `json:"type"`  // "dns" または "ip"
	Value string `json:"value"` // "example.com" または "192.168.1.10"
}
```

サーバー側のオーダー処理では、以下のように判定します。

```go
switch identifier.Type {
case "dns":
	chTypes = append(chTypes, "http-01", "tls-alpn-01", "dns-01")
case "ip":
	// IPアドレスの場合は http-01 または tls-alpn-01 を提供
	parsedIP := net.ParseIP(identifier.Value)
	if parsedIP == nil {
		return c.JSON(http.StatusBadRequest, &acmeError{
			Type:   "urn:ietf:params:acme:error:rejectedIdentifier",
			Detail: "invalid IP address identifier",
		})
	}
	chTypes = append(chTypes, "http-01", "tls-alpn-01")
}
```

そして、CSR（証明書署名要求）を処理して証明書を発行するフェーズ（`/order/:id/finalize`）で、CSRの `IPAddresses` に対象IPが含まれていることを検証し、発行するX.509証明書の `IPAddresses`（SAN拡張）に正しく設定します。

---

# 3. 端末構成証明（Device Attestation）の実装

## ゼロトラスト時代の課題：秘密鍵は本当にその端末にあるか？

ACMEは「そのドメインやIPを制御できているか」を検証する仕組みですが、**「証明書の秘密鍵が正規のデバイス内のセキュアチップ（TPM / Secure Enclave / YubiKey）で生成され、外部にエクスポートできない状態にあるか」**までは検証できません。

私用のPCや不正な端末からACME経由で証明書をリクエストされても、ドメイン検証さえ通れば証明書が発行されてしまいます。

これを解決するのが、現在IETFで標準化が進められている **ACME Device Attestation (`draft-ietf-acme-device-attest`)** です。
Appleの **Managed Device Attestation**（iOS 16 / macOS 14以降）や **YubiKey PIV**（Smallstep仕様）などで採用されています。

## `device-attest-01` の仕組み

1. クライアントはハードウェア（Secure EnclaveやYubiKey）内で秘密鍵ペアを生成し、ハードウェアベンダーの認証局が署名した「構成証明書（Attestation Certificate）」を取得します。
2. ACMEサーバーに対して `permanent-identifier` または `hardware-module` 識別子でオーダーを作成します。
3. チャレンジ `device-attest-01` に対して、CBORエンコードされた構成証明オブジェクト（`attObj`）を送信します。
4. ACMEサーバーは以下の検証を行います：
   - 構成証明書の署名チェーンがAppleやYubicoの公式ルートCAに繋がっているか
   - 構成証明書に含まれる公開鍵と、チャレンジで署名された公開鍵が一致するか
   - デバイスの固有識別子（シリアル番号やUDIDなど、**RFC 4043 PermanentIdentifier**）を取得
5. 最終的なCSR提出時（finalize）に、CSR内の公開鍵が先ほどAttestationで検証された公開鍵と同一であるかを検証します。

```
[クライアント端末]                              [ACMEサーバー]
 Secure Enclave で鍵生成
        |
        |--- 1. new-order (permanent-identifier) --->|
        |<-- 2. challenge (device-attest-01) --------|
        |
        |--- 3. challenge 応答 (CBOR attObj) -------->|
        |                                              |-- Apple/Yubico RootCAチェーン検証
        |                                              |-- RFC 4043 PermanentIdentifier抽出
        |<-- 4. challenge valid ----------------------|
        |
        |--- 5. finalize (CSR) ---------------------->|
        |                                              |-- CSR公開鍵が構成証明鍵と一致するか検証
        |                                              |-- SANにPermanentIdentifierを埋め込み署名
        |<-- 6. 証明書発行 (PKCS#7/PEM) --------------|
```

## Go言語での実装ハイライト

### ① CBOR形式の Attestation Object デコード

Attestation Object は WebAuthn と類似の CBOR（RFC 8949）フォーマットで送られてきます。
フォーマット `fmt` には Apple用の `"apple"` や Smallstep/YubiKey用の `"step"` などがあります。

```go
type attestationObject struct {
	Format       string         // "apple", "step" など
	AttStatement map[string]any // 証明書チェーン (x5c) や署名データ
	AuthData     []byte
}

type deviceAttestPayload struct {
	AttObj string `json:"attObj"` // Base64エンコードされたCBORデータ
}
```

### ② 証明書チェーンの検証

ベンダーの公開ルート証明書（Apple Enterprise Attestation Root CA や Yubico PIV Root CA）を `x509.CertPool` に登録し、クライアントから送られた `x5c` チェーンを検証します。

```go
func verifyAttestationChain(certs []*x509.Certificate, roots *x509.CertPool) error {
	if len(certs) == 0 {
		return fmt.Errorf("empty attestation certificate chain")
	}
	opts := x509.VerifyOptions{
		Roots:         roots,
		Intermediates: x509.NewCertPool(),
		KeyUsages:     []x509.ExtKeyUsage{x509.ExtKeyUsageAny},
	}
	for _, ic := range certs[1:] {
		opts.Intermediates.AddCert(ic)
	}
	_, err := certs[0].Verify(opts)
	return err
}
```

### ③ RFC 4043 PermanentIdentifier の抽出とSAN拡張

Appleデバイスの場合、Attestation証明書の拡張領域（OID `1.2.840.113635.100.8.2` 等）やSubject、あるいはYubiKeyのシリアル番号からデバイスのハードウェアUUID/シリアル番号を取り出すことができます。

この識別子を、発行するクライアント証明書の **SAN（Subject Alternative Name）OtherName: RFC 4043 PermanentIdentifier**（OID: `1.3.6.1.5.5.7.8.3`）として埋め込みます。

```go
var oidPermanentIdentifier = asn1.ObjectIdentifier{1, 3, 6, 1, 5, 5, 7, 8, 3}

// PermanentIdentifier のASN.1構造
type permanentIdentifier struct {
	IdentifierValue string                 `asn1:"utf8,optional"`
	Assigner        asn1.ObjectIdentifier `asn1:"optional"`
}
```

これを発行証明書の拡張（`pkix.Extension`）に追加することで、証明書自体が「この端末のセキュアチップに紐づいた証明書である」ことを明確に示すことができます。

### ④ CSRの公開鍵とのフィンガープリント突合

finalize時に送られてくるCSRの公開鍵が、構成証明された公開鍵と完全に一致するかをSHA-256フィンガープリントで突合します。

```go
attestedFP := az.Fingerprint // device-attest-01 で検証済みの鍵指紋
csrFP, err := publicKeyFingerprint(fr.csr.PublicKey)
if err != nil {
	return c.JSON(http.StatusBadRequest, err)
}

// 固定時間比較で一致を確認
if subtle.ConstantTimeCompare([]byte(attestedFP), []byte(csrFP)) != 1 {
	return c.JSON(http.StatusBadRequest, &acmeError{
		Type:   "urn:ietf:params:acme:error:badCSR",
		Detail: "CSR public key does not match attested public key",
	})
}
```

---

# 4. TWSNMP FK での活用とゼロトラスト連携

[TWSNMP FK](https://github.com/twsnmp/twsnmpfk) では、今回解説したACME拡張を活用して以下のようなネットワーク管理連携を行っています。

1. **管理対象ノードとの自動照合**:
   - ACMEで証明書を要求してきた端末のIPアドレス、DNS名、または PermanentIdentifier（シリアル番号）をマップ上の登録ノード情報と照合。
   - 未知の端末からのリクエストは自動的に拒否（Reject）し、イベントログに記録。
2. **社内端末のゼロトラスト管理**:
   - 管理者がノードの「説明」や「公開鍵」フィールドにMac/iPad/YubiKeyのシリアル番号を登録しておくだけで、端末キッティング時にACME経由で安全に証明書を自動配備。

---

# まとめ

ACMEを拡張することで、従来のWebサーバー向けSSL証明書発行にとどまらず、**ワイルドカード運用**、**社内IPアドレス証明書**、そして**ハードウェア構成証明（Device Attestation）によるゼロトラスト端末認証**までを単一のACMEサーバー基盤で実現できるようになります。

次回（続編第5弾）は、ネットワーク機器や組込みIoT機器のデファクトスタンダードである **EST（RFC 7030 / Enrollment over Secure Transport）** のGo言語による実装について解説します！

### 参考ソースコード
実装の全容は TWSNMP FK のリポジトリで公開しています。

- https://github.com/twsnmp/twsnmpfk （`pki/acme.go`, `pki/ca.go`）
