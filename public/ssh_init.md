---
title: SSHで暗号通信を確立するまでの仕組みをRFC準拠で解説
tags:
  - SSH
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# SSHの暗号通信の仕組み

SSH の接続開始から暗号化通信の確立までの流れを RFC 準拠で解説します。

この記事はクライアントとサーバが送り合う値を詳しく見ていくことが目的で、暗号技術の基礎知識を前提としています。  

## 基礎知識

SSH は、ホスト（サーバ）認証・機密性・完全性を提供するプロトコルです。  

- 公開鍵暗号（暗号化、鍵交換、電子署名）
- 共通鍵暗号
- MAC（メッセージ認証コード）
- ハッシュ関数
を組み合わせることで、これらの機能を提供します。

※ ECDH/DH鍵交換を使う場合は前方秘匿性も提供されます。


これからSSHの仕組みを説明するにあたって、以下の用語を理解しておく必要があります。

### ホストキー

ホストは電子署名の秘密鍵・公開鍵ペア（ホストキー）を少なくとも1つ所有していなければなりません。

ホストキーは、**ホスト認証**（クライアントが正しいサーバと通信していることの確認）のため、鍵交換中に使用されます。その仕組み上、**クライアントはホストキーの公開鍵を事前に知っていなくてはなりません**。  

この公開鍵とサーバの紐づけには、二種類の方法があります。

1. クライアントが公開鍵とサーバ名（IPアドレスやFQDNなど）を紐づけたローカルのDBを持つ（例：`~/.ssh/known_hosts`）。
2. CA（認証局）による署名付き公開鍵証明書を使う。

1については、そもそもそのDBをどのように構築するかという問題があります。  
一度でも正しい相手と接続できればその公開鍵をDBに保存しておけばよいですが、初回接続時は何かしら別の安全な手段を利用しなくてはなりません。

例えば GitHubの場合、公開鍵のフィンガープリントがWeb上で公開されています。

- [GitHubの公開鍵フィンガープリント](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)

HTTPS経由で入手できるため、このフィンガープリントを信頼できるものとして扱うことができます。  
※「HTTPS経由だから」という言い方はTLSを理解している方でないと正確な意味が伝わりきらないですが、ここはSSHの解説記事なのでこれ以上踏み込みません。

これ以降、公開鍵・秘密鍵という言葉がよく出てきますが、ホストキーのそれを指す場合は「ホストキーの公開鍵」「ホストキーの秘密鍵」というように表現します。  

## 接続のセットアップ

SSHはクライアント側から接続を開始します。

### プロトコルバージョンの交換

まず、お互いが識別子を送信しあう必要があります。識別子とは、以下の形式の文字列です。

```plaintext
SSH-protoversion-softwareversion SP comments CR LF
```

例：`SSH-2.0-OpenSSH_9.8`

- `protoversion`: 2025年2月現在、最新のプロトコルバージョンは `2.0` です。
- `comments`: これはオプションです。もし`comments`がある場合は、`SP`（スペース）で`softwareversion`と区切られる必要があります。
- `CR`: キャリッジリターン
- `LF`: ラインフィード

`CR`より前の部分**はクライアント・サーバそれぞれの識別子として扱われ**、鍵交換で利用されます。

クライアント側はこの識別子のみを送信しますが、サーバはこれ以外の情報も一緒に送信することができます。大抵は次で説明する`SSH_MSG_KEXINIT`メッセージを一緒に送信します。

このプロトコルバージョンの交換を終えたら、鍵交換が始まります。

### 鍵交換

鍵交換は、暗号化・MAC付与（or AEAD）で使う鍵の材料である共有シークレットを共有するための手順です。鍵交換の後、共有シークレットからハッシュ関数を使って鍵を生成します。  
また、**ホスト認証もここで同時に**行われます。
※クライアント認証は暗号化通信の確立後に行われます。クライアント認証の公開鍵認証の仕組みについては、拙記事[SSHの公開鍵認証の仕組み（RFC準拠）](https://qiita.com/aurora1530/items/24a43667d31f1818d689)をご覧ください。

#### アルゴリズム交渉

鍵交換は、まずお互いにサポートするアルゴリズムのリストを送信しあうことから始まります。  

```plaintext
byte: SSH_MSG_KEXINIT（鍵交換の開始メッセージを表す値）
byte[16]: cookie（ランダムバイト列）
name-list: kex_algorithms（鍵交換アルゴリズム）
name-list: server_host_key_algorithms（サーバのホストキーのアルゴリズム）
name-list: encryption_algorithms_client_to_server（クライアントからサーバへの暗号化アルゴリズム）
name-list: encryption_algorithms_server_to_client（サーバからクライアントへの暗号化アルゴリズム）
name-list: mac_algorithms_client_to_server（クライアントからサーバへのメッセージ認証コードアルゴリズム）
name-list: mac_algorithms_server_to_client（サーバからクライアントへのメッセージ認証コードアルゴリズム）
name-list: compression_algorithms_client_to_server（クライアントからサーバへの圧縮アルゴリズム）
name-list: compression_algorithms_server_to_client（サーバからクライアントへの圧縮アルゴリズム）
name-list: languages_client_to_server（クライアントからサーバへの言語）
name-list: languages_server_to_client（サーバからクライアントへの言語）
boolean: first_kex_packet_follows(この次に推測された鍵交換メッセージが続くかどうか)
uint32: 0（将来拡張機能のためのフィールド）
```

name-listは、カンマ区切りで複数のアルゴリズムを優先度が高いものから列挙します。  

`cookie`は、送信者が生成したランダムな値です。これは、この後生成する**鍵**や**セッション識別子**を**クライアント・サーバのどちらもコントロールできないように**するために必要です。
※ `SSH_MSG_KEXINIT`メッセージはセッション識別子や後述するExchange Hashの生成に使われます。鍵はセッション識別子とExchange Hashを使います。

このメッセージの後、鍵交換アルゴリズムが実行されます。

以降、何かハッシュ計算を行う場合は`kex-algorithms`で合意されたハッシュ関数を使います。
※ 鍵交換はホスト認証のためにハッシュ関数を使います。そのため、`diffie-hellman-~~-sha256`や`rsa2048-sha256`のようにアルゴリズム名でハッシュ関数が指定されています。

#### 鍵交換（ECDH）

ここでは、ECDH（Elliptic Curve Diffie-Hellman）鍵交換を例に取ります。  
DH鍵交換の場合も大体同じです。  

交換アルゴリズムの概要は以下の通りです（RFC5656より）。

```plaintext
Client                                                Server
------                                                ------
使い捨ての鍵ペアを生成
SSH_MSG_KEX_ECDH_INIT  -------------->

                                 受け取った鍵が正当なものか検証
                                        使い捨ての鍵ペアを生成
                                                 共有シークレットを計算
                                    Exchange Hashと署名を生成
                       <------------- SSH_MSG_KEX_ECDH_REPLY
受け取った公開鍵が正当なものか検証
受け取ったホストキーの公開鍵がサーバのものと一致するか検証
共有シークレットを計算
Exchange Hashを生成
署名を検証
```

詳しく見ていきます。

まず、クライアントはECDHの秘密鍵`S_C`と公開鍵`Q_C`を生成します。  
そして、以下の形式のメッセージをサーバに送信します。

```plaintext
byte: SSH_MSG_KEX_ECDH_INIT（ECDH鍵交換メッセージの開始を表す値）
string: Q_C（クライアントの公開鍵）
```

サーバは、クライアントの公開鍵が有効かどうかを検証し、自身のECDHの秘密鍵`S_S`と公開鍵`Q_S`を生成します。

<details><summary>「公開鍵を検証」の意味</summary>

ECDH鍵交換の場合、公開鍵は楕円曲線上の点です。この点が有効なものであることを、以下の仕様の3.2.2の手順に従って検証します。

- Standards for Efficient Cryptography Group (SECG), "SEC 1: Elliptic Curve Cryptography", Version 2.0, May 2009. [Online]. Available: <https://www.secg.org/sec1-v2.pdf>

DH鍵交換の場合は、公開鍵が$[1,p-1]$の範囲にあるかどうかを検証します。$p$はDH鍵交換で使われる素数で、公開値（RFC3526で指定されている）です。  

---

</details>

そして、共有シークレット`K`をクライアントの公開鍵`Q_C`とサーバの秘密鍵`S_S`を使って計算します。

その後、Exchange Hash`H`を以下の連結のハッシュ値として計算します。
※ このExchange Hashはこの後よく登場します。

```plaintext
string: クライアントの識別子（プロトコルバージョン文字列）
string: サーバの識別子（同上）
string: クライアントからサーバへ送ったSSH_MSG_KEXINITメッセージのペイロード
string: サーバからクライアントへ送ったSSH_MSG_KEXINITメッセージのペイロード
string: サーバのホストキー
string: Q_C
string: Q_S
mpint: K
```

そして、**この`H`に対して**ホストキーの秘密鍵を使って署名`signature`を生成します。

最後に、サーバは以下の形式のメッセージをクライアントに送信します。

```plaintext
byte: SSH_MSG_KEX_ECDH_REPLY（ECDH鍵交換の返信メッセージを表す値）
string: ホストキーの公開鍵
string: Q_S
string: signature
```

クライアントは、まず受け取った公開鍵が有効かどうかを検証します。

次に、**受け取ったホストキーの公開鍵がサーバの正しいホストキーの公開鍵であること**を検証します（例えば、前述したローカルDB`~/.ssh/known_hosts`を使って）。

そして、共有シークレット`K`と Exchange Hash`H`を生成します。  
最後に、ホストキーの公開鍵を使って署名`signature`を検証します。検証が成功すれば、正しいサーバと通信していることが確認できます。

<details><summary>なぜ「正しいサーバと通信している」といえるのか</summary>

署名対象に**鍵交換の公開鍵**（クライアント・サーバ両方）と**共有シークレット**が含まれているからです。これにより、署名を行った相手に対して**間違いなく鍵交換が成立している**ことが分かります。鍵交換が成立すれば、**これから暗号通信を行う相手が正しいサーバである**ということが保証されます。
※ お互いの識別子や`SSH_MSG_KEXINIT`、ホストキーの公開鍵も署名対象に含まれるので、これらのやり取りも正しい相手と行えていたことが保証されます。
※ 繰り返しになりますが、「サーバの正しい公開鍵を知っていること」が前提です。

また、これは鍵交換の欠点を補います。ECDH/DH鍵交換や後述するRSA鍵配送は優れた鍵交換手法ですが、これらの手法は鍵交換の相手が正しいものであることを保証しませんが、署名を組み合わせることでこの問題が解決されます。

</details>

<details><summary>鍵交換（RSA鍵交換の場合）</summary>

あまりRSA鍵交換は使われないですが、**一応まだ非推奨ではない（MAY）** ので説明します。  
みんな大好きな「公開鍵で鍵を暗号化して、秘密鍵で復号する」やつです。このタイプの鍵交換を「使われていない」と言いきれなくさせる要因の一つなので、いい加減[SHOULD NOT]にしてほしいですね。する理由がまだないのでしょうが、、、

ここでのRSAは鍵交換のためのRSAで、暗号化なのでRSAES-OAEP[[RFC3447]](https://www.rfc-editor.org/info/rfc3447)のことです。以降、RSAES-OAEPは単にRSAと呼びます。
また、現在SSHで使える鍵長は2048ビット、ハッシュ関数はSHA-256ですので、これも前提とします。  

この鍵交換では、まずサーバが下記の形式のメッセージをクライアントに送信します。

```plaintext
byte: SSH_MSG_KEXRSA_PUBKEY（RSA鍵交換メッセージの開始を表す値）
string: サーバのホストキーの公開鍵
string: K_T（RSA公開鍵）
```

次に、クライアントは$0<=K<2^{LEN(key) - 2 * LEN(hash) - 49}$を満たす乱数`K`を生成します。  
※ RSAES-OAEPで暗号化可能なサイズより少し小さいですが、これは`K`が`mpint`エンコードされており、このエンコーディングで少しデータサイズが増えるためです。

ここで、`LEN(key)`はRSAの鍵長（2048ビット）、`LEN(hash)`はハッシュ関数の出力長（SHA-256なので256ビット）です。  

その後、`K_T`を使って`K`を暗号化し、以下の形式のメッセージをサーバに送信します。

```plaintext
byte: SSH_MSG_KEXRSA_SECRET（RSA鍵交換メッセージの続きを表す値）
string: 暗号化されたk
```

サーバは、受け取った暗号文をRSA秘密鍵で復号し、共有シークレット`K`を取得します。

その後、以下の値の連結をハッシュ化したものをExchange Hash`H`とします。

```plaintext
string: クライアントの識別子（プロトコルバージョン文字列）
string: サーバの識別子（同上）
string: クライアントからサーバへ送ったSSH_MSG_KEXINITメッセージのペイロード
string: サーバからクライアントへ送ったSSH_MSG_KEXINITメッセージのペイロード
string: サーバのホストキー
string: RSA公開鍵
string: 暗号化されたk
mpint: K（共有シークレット）
```

そして、この`H`に対してホストキーの秘密鍵を使って署名`signature`を生成し、以下の形式のメッセージをクライアントに送信します。

```plaintext
byte: SSH_MSG_KEXRSA_DONE （RSA鍵交換の終了を表す値）
string: signature
```

クライアントは`H`を計算し、サーバから受け取った署名`signature`を検証します。検証が成功すれば、正しいサーバと通信していることが確認できます。

---

</details>

#### 鍵交換からの出力

鍵交換により、共有シークレット`K`と Exchange Hash`H`がそれぞれで生成されました。  
セッションの**最初の**鍵交換で作られた`H`はセッション識別子です。後述しますが、鍵交換はセッション中に複数回行われる場合があります。それでも**セッション識別子は変わりません**。

`K`はまだ鍵の「素」であり、このままでは使いません（そもそも鍵の数が足りません）。  
ここから`K`や`H`、ハッシュ関数を使って鍵を生成します。

#### 鍵生成

鍵生成は、SSH独自の方法で行われます。

以降、`K`を共有シークレット、`HASH`を`kex_algorithms`で合意されたハッシュ関数、`H`をExchange Hash、`session_id`をセッション識別子とします。

生成しなくてはならない値は、

- 暗号化で使うIV（Initialization Vector）
- 暗号化の鍵
- MACの鍵

を**クライアントからサーバへの通信**と**サーバからクライアントへの通信**のそれぞれの分です。
※ ただし、AES-128-GCMやChaCha20-Poly1305のようなAEADではMAC用の鍵を使いません。

以下のようにハッシュ関数を使って生成します。

- IV(server to client): `HASH(K || H || "A" || session_id)`
- IV(client to server): `HASH(K || H || "B" || session_id)`
- encryption key(server to client): `HASH(K || H || "C" || session_id)`
- encryption key(client to server): `HASH(K || H || "D" || session_id)`
- MAC key(server to client): `HASH(K || H || "E" || session_id)`
- MAC key(client to server): `HASH(K || H || "F" || session_id)`

"A","B",...は一文字のASCII文字です。

必要な長さが出力より短い場合、先頭から必要な長さ分を使います。  

逆に、必要な長さが出力より長い場合、以下のように連結してハッシュ化した`Key`を使います。

```plaintext
Key := K1 || K2 || K3 || ...
K1 := HASH(K || H || X || session_id) 
(Xの部分は、必要とされる鍵で使われた文字。server to clientのIVなら"A")
K2 := HASH(K || H || K1)
K3 := HASH(K || H || K2)
...
```

#### 鍵交換の終了

鍵交換を終えたら、以下のメッセージを送り合い、以降の通信は暗号化・MAC付きで行われます。

```plaintext
byte: SSH_MSG_NEWKEYS（新しい鍵を使うことを表す値）
```

#### 再鍵交換

RFC4253によると、送信データの1GBごと または 接続時間の1時間ごと のいずれか早い方で鍵交換をもう一度行うことが推奨されています。  

鍵交換中は、交換の開始時に有効な暗号化を利用して実行されます。  
SSH_MSG_NEWKEYSが送信されたら、新しい暗号化・MACが使われるようになります。

## 参考文献

- [RFC 3526 - More Modular Exponential (MODP) Diffie-Hellman groups for Internet Key Exchange (IKE)](https://www.rfc-editor.org/info/rfc3526)
- [RFC 4251 The Secure Shell (SSH) Protocol Architecture](https://www.rfc-editor.org/info/rfc4251)
- [RFC 4253 The Secure Shell (SSH) Transport Layer Protocol](https://www.rfc-editor.org/info/rfc4253)
- [RFC 4432 RSA Key Exchange for the Secure Shell (SSH) Transport Layer Protocol](https://www.rfc-editor.org/info/rfc4432)
- [RFC 5656 - Elliptic Curve Algorithm Integration in the Secure Shell Transport Layer](https://www.rfc-editor.org/info/rfc5656)
- [RFC 5647 - AES Galois Counter Mode for the Secure Shell Transport Layer Protocol](https://www.rfc-editor.org/info/rfc5647)
- [RFC 8268 - More Modular Exponentiation (MODP) Diffie-Hellman (DH) Key Exchange (KEX) Groups for Secure Shell (SSH)](https://www.rfc-editor.org/info/rfc8268)
- [The chacha20-poly1305@openssh.com authenticated encryption cipher draft-josefsson-ssh-chacha20-poly1305-openssh-00](https://datatracker.ietf.org/doc/html/draft-josefsson-ssh-chacha20-poly1305-openssh-00#section-4)
- [IANA Secure Shell (SSH) Protocol Parameters](https://www.iana.org/assignments/ssh-parameters/ssh-parameters.xhtml)
