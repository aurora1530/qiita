---
title: SSHの公開鍵認証の仕組み（RFC準拠）
tags:
  - SSH
private: false
updated_at: '2025-02-13T20:56:08+09:00'
id: 24a43667d31f1818d689
organization_url_name: null
slide: false
ignorePublish: false
---
# SSHの公開鍵認証の仕組み

SSHの公開鍵認証をRFC準拠で説明します。


公開鍵認証では、秘密鍵の所有を証明することが認証として機能します。  
証明は、クライアントが秘密鍵で署名を作成し、サーバーがその秘密鍵に対応する公開鍵を使って署名を検証することで行われます。

## 公開鍵認証の流れ

署名によって認証を行うには、次の二段階が必要です。

1. 署名の検証に使う公開鍵とユーザーを紐づける。
2. ユーザーが秘密鍵を使って署名を作成し、サーバーがその署名を事前にユーザーに紐づけられた公開鍵で検証する。

これは公開鍵認証のよくある説明ですが、実際にはこの流れを実現するためにはさらに細かい手順が必要です。

1. 公開鍵とユーザーを紐づける
    - 公開鍵認証では、ユーザーと公開鍵との関連付けが前提となります。もしこの紐付けがなければ、後続の署名検証（手順2）の意義は失われてしまいます。ただし、この紐付けを行うためには、あらかじめ何らかの方法でユーザーを認証し、その上で公開鍵を登録する必要があります。なお、このユーザー認証および公開鍵登録の具体的な手続きについては、SSH の RFC には規定されていません。
        - たとえば、GitHub では、Web サイト上でのログイン認証を経た後に公開鍵を登録する仕組みが採用されています。
2. 署名の検証
    - 署名によってクライアントが秘密鍵の所有者であることを証明するためには、次の2点が重要です。
    1. 秘密鍵の安全な管理
        - クライアントは秘密鍵を適切に保護し、第三者に漏洩しないよう努める必要があります（例：パスフレーズを設定）
    2. 署名対象のデータがそのセッション固有のものであること
    電子署名の有効性を保証するため、署名対象となるデータは各セッションごとに一意でなければなりません。もし毎回同一のデータに対して署名が行われると、クライアントまたはサーバーが任意に操作可能な値が署名対象となる可能性があり、その結果、リプレイ攻撃が成立するリスクが生じます。SSH では、セッション識別子を署名対象データに含めることで、各セッションごとに固有のデータとし、リプレイ攻撃を防止しています。なお、このセッション識別子は、鍵交換時に使用する双方の公開鍵やその他の値をハッシュ化して算出されるため、どちらか一方が自由に操作することはできません。

### SSH の公開鍵認証の流れ

実際にクライアントとサーバー間で送り合う値を見ていきましょう。

##### 1. まず、クライアントからサーバーに対して以下の値が送信されます。

※いきなり手順3の署名を送ることもできます。

このメッセージにより、公開鍵認証で認証を行うこと・公開鍵アルゴリズムを指定します。

```plaintext
byte: SSH_MSG_USERAUTH_REQUEST（認証リクエストを表す値）
string: ユーザー名
string: 認証後に使用するサービス名（例："ssh-connection"）
string: 認証方法名(公開鍵認証では "publickey" が使われる)
bool: False
string: 公開鍵アルゴリズム名
string: 公開鍵のバイナリデータ
```

認証には任意の公開鍵アルゴリズムを利用できますが、サーバーがサポートしていない場合は認証が失敗します。が、あまりそんなことを意識することはないでしょう。

また、複数の公開鍵を登録していたとしても、ここで公開鍵のバイナリデータを指定することで、どの公開鍵を使って認証を行うかを指定できます（というよりは、サーバー側がそれを認識できる、と言ったほうが正確かもしれません）。

##### 2. 次に、サーバーからクライアントに対して以下の値が送信されます。

```plaintext
byte: SSH_MSG_USERAUTH_PK_OK（公開鍵認証が許可されたことを表す値）
string: 公開鍵アルゴリズム名（クライアントが指定したもの）
string: 公開鍵のバイナリデータ（クライアントが指定したもの）
```

##### 3. クライアントは、秘密鍵を使って署名を作成し、サーバーに送信します。

まずメッセージの全体像を示します。

```plaintext
byte: SSH_MSG_USERAUTH_REQUEST（認証リクエストを表す値）
string: ユーザー名
string: 認証後に使用するサービス名（例："ssh-connection"）
string: 認証方法名（"publickey"）
bool: True
string: 公開鍵アルゴリズム名
string: 公開鍵のバイナリデータ
string: 署名
```

署名は、以下の値への署名です。

```plaintext
string: セッション識別子（後述します）
byte: SSH_MSG_USERAUTH_REQUEST（認証リクエストを表す値）
string: ユーザー名
string: 認証後に使用するサービス名（例："ssh-connection"）
string: 認証方法名（"publickey"）
bool: True
string: 公開鍵アルゴリズム名
string: 公開鍵のバイナリデータ
```

##### 4. 最後に、サーバーはクライアントから受け取った署名を検証します。

検証に使う公開鍵は事前にクライアントと紐づけられているものです。署名の検証に成功すれば、認証が成功となります。

成功・失敗それぞれでサーバーからクライアントに対して以下の値が送信されます。

```plaintext
byte: SSH_MSG_USERAUTH_SUCCESS（認証成功を表す値）
```

```plaintext
byte: SSH_MSG_USERAUTH_FAILURE（認証失敗を表す値）
```

### セッション識別子

セッション識別子は SSH の鍵交換後に生成される値で、そのセッションにおいて一意な値です。

具体的には、以下の値を連結したものをハッシュ化したものです。

```plaintext
string: クライアントの識別子（プロトコルバージョン文字列：例 SSH-2.0-OpenSSH_9.8）
string: サーバーの識別子（同上）
string: クライアントからサーバーへ送ったSSH_MSG_KEXINITメッセージのペイロード（後述します）
string: サーバーからクライアントへ送ったSSH_MSG_KEXINITメッセージのペイロード（後述します）
string: サーバーのホストキー（サーバー認証に使う公開鍵）
mpint: 鍵交換でクライアントが生成した共有値
mpint: 鍵交換でサーバーが生成した共有値
mpint: 鍵交換によって生成された共有鍵
```

<details><summary>補足</summary>

ただし、これは DH鍵交換を利用する場合で、ECDH(E)だと下から2~3番目がどちらもstringになります。

また、RSA鍵交換（あまり使われないですが）の場合、下から2~3番目が下記の値になります。

```plaintext
string: サーバの RSA 公開鍵
string 共有鍵を RSA で暗号化した値
```

</details>

### SSH_MSG_KEXINIT

SSH_MSG_KEXINIT は SSH で使う各種暗号アルゴリズムを合意するためのメッセージです。 　
公開鍵認証では、前述したとおりこのメッセージのペイロードがセッション識別子の生成に使われます。  
また、その生成に使うハッシュ関数は、下記の `kex_algorithms` で合意されたもの（鍵交換に使うハッシュ関数）です。  

今回の話では下記の情報はあまり重要ではありません。

```plaintext
byte: SSH_MSG_KEXINIT（鍵交換の初期化メッセージを表す値）
byte[16]: ランダムバイト列
name-list: kex_algorithms（鍵交換アルゴリズム）
name-list: server_host_key_algorithms（サーバーのホストキーのアルゴリズム）
name-list: encryption_algorithms_client_to_server（クライアントからサーバーへの暗号化アルゴリズム）
name-list: encryption_algorithms_server_to_client（サーバーからクライアントへの暗号化アルゴリズム）
name-list: mac_algorithms_client_to_server（クライアントからサーバーへのメッセージ認証コードアルゴリズム）
name-list: mac_algorithms_server_to_client（サーバーからクライアントへのメッセージ認証コードアルゴリズム）
name-list: compression_algorithms_client_to_server（クライアントからサーバーへの圧縮アルゴリズム）
name-list: compression_algorithms_server_to_client（サーバーからクライアントへの圧縮アルゴリズム）
name-list: languages_client_to_server（クライアントからサーバーへの言語）
name-list: languages_server_to_client（サーバーからクライアントへの言語）
boolean: first_kex_packet_follows(この次に推測された鍵交換メッセージが続くかどうか)
uint32: 0（将来拡張機能のためのフィールド）
```

### mpint

mpintとは、多倍長整数（大きな整数）を表すためのデータ型です。

## 参考文献

- [RFC4251 The Secure Shell (SSH) Protocol Architecture](https://www.rfc-editor.org/info/rfc4251)
- [RFC4252 The Secure Shell (SSH) Authentication Protocol](https://www.rfc-editor.org/info/rfc4252)
- [RFC4253 The Secure Shell (SSH) Transport Layer Protocol](https://www.rfc-editor.org/info/rfc4253)
- [RFC4432 RSA Key Exchange for the Secure Shell (SSH) Transport Layer Protocol](https://www.rfc-editor.org/info/rfc4432)
- [RFC5656 Elliptic Curve Algorithm Integration in the Secure Shell Transport Layer](https://www.rfc-editor.org/info/rfc5656)
