---
title: SSHの公開鍵認証の仕組み
tags:
  - ''
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# SSHの公開鍵認証の仕組み

公開鍵認証では、秘密鍵の所有を証明することが認証として機能します。  
証明は、クライアントが秘密鍵で署名を作成し、サーバーがその秘密鍵に対応する公開鍵を使って署名を検証することで行われます。

## 公開鍵認証の流れ

署名によって認証を行うには、次の二段階が必要です。

1. 署名の検証に使う公開鍵とユーザーを紐づける。
2. ユーザーが秘密鍵を使って署名を作成し、サーバーがその署名を事前にユーザーに紐づけられた公開鍵で検証する。

それぞれ注意点があります。

1. 公開鍵とユーザーを紐づける
    - 当たり前の話ですが、これがなければ手順2の意味がありません。しかし、これを行うには何らかの方法でユーザーを認証し、かつその上で公開鍵を登録する必要があります。この方法は RFC には定められていません。
    - 例えば、GitHub では Web ページから公開鍵を登録します。（冗長な説明をすれば、この方法は https を使って通信の機密性・完全性・真正性を担保し、そしてアプリケーション層でユーザー認証を行ったうえで公開鍵を登録します。）
2. 署名の検証
    - 署名の検証によって相手が（この場合ではクライアントが）秘密鍵の所有者であることの証明とするには、2-1) クライアントが秘密鍵を安全に保持していること 2-2) 署名対象のデータがそのセッション固有のものであること が必要です。2-1 はクライアントの責任であり、パスフレーズを設定するなどして対処します。2-2 はこのような電子署名を使った認証で重要です。毎回同じデータだったり、クライアント（or サーバー）が操作可能な値を署名対象とすると、リプレイ攻撃を成立させる要因になります。SSH ではセッション識別子を使うことでこれを防いでいます。

### SSH の公開鍵認証の流れ

実際にクライアントとサーバー間で送り合う値を見ていきましょう。

1. まず、クライアントからサーバーに対して以下の値が送信されます。

このメッセージにより、公開鍵認証で認証を行うこと・公開鍵アルゴリズムを指定します。

```plaintext
byte: SSH_MSG_USERAUTH_REQUEST（認証リクエストを表す値）
string: ユーザー名
string: サービス名
string: 認証方法名(公開鍵認証では "publickey" が使われる)
bool: False
string: 公開鍵アルゴリズム名
string: 公開鍵のバイナリデータ
```

認証には任意の公開鍵アルゴリズムを利用できますが、サーバーがサポートしていない場合は認証が失敗します。が、あまりそんなことを意識することはないでしょう。

また、複数の公開鍵を登録していたとしても、ここで公開鍵のバイナリデータを指定することで、どの公開鍵を使って認証を行うかを指定できます（というよりは、サーバー側がそれを認識できる、と言ったほうが正確かもしれません）。

2. 次に、サーバーからクライアントに対して以下の値が送信されます。

```plaintext
byte: SSH_MSG_USERAUTH_PK_OK（公開鍵認証が許可されたことを表す値）
string: 公開鍵アルゴリズム名（クライアントが指定したもの）
string: 公開鍵のバイナリデータ（クライアントが指定したもの）
```

3. クライアントは、秘密鍵を使って署名を作成し、サーバーに送信します。

まずメッセージの全体像を示します。

```plaintext
byte: SSH_MSG_USERAUTH_REQUEST（認証リクエストを表す値）
string: ユーザー名
string: サービス名
string: 認証方法名（"publickey"）
bool: True
string: 公開鍵アルゴリズム名
string: 公開鍵のバイナリデータ
string: 署名
```

署名は、秘密鍵を使って以下の値の以下の順での署名です。

```plaintext
string: セッション識別子（後述します）
byte: SSH_MSG_USERAUTH_REQUEST（認証リクエストを表す値）
string: ユーザー名
string: サービス名
string: 認証方法名（"publickey"）
bool: True
string: 公開鍵アルゴリズム名
string: 公開鍵のバイナリデータ
```

4. 最後に、サーバーはクライアントから受け取った署名を検証します。

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
string: クライアントの識別子
string: サーバーの識別子
string: クライアントからサーバーへ送ったSSH_MSG_KEXINITメッセージのペイロード（後述します）
string: サーバーからクライアントへ送ったSSH_MSG_KEXINITメッセージのペイロード（後述します）
mpint: サーバーのホストキー（サーバー認証に使う公開鍵）
mpint: 鍵交換でクライアントが生成した共有値
mpint: 鍵交換でサーバーが生成した共有値
mpint: 鍵交換によって生成された共有鍵
```

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
