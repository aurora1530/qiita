---
title: iron-session の仕組み
tags:
  - JavaScript
private: true
updated_at: '2025-02-24T15:17:04+09:00'
id: 015cdef0fc9c8033c949
organization_url_name: null
slide: false
ignorePublish: false
---
# iron-session の仕組み

iron-session([https://github.com/vvo/iron-session](https://github.com/vvo/iron-session))とは、Cookieを使ったセッション管理を行うためのJavaScriptライブラリです。  

iron-sessionではセッション情報がCookieに保存されるため、ステートレスなセッション管理が可能です。保存される情報は暗号化され、かつ署名（正確にはMAC）が付与されます。

本記事では、[iron-session](https://github.com/vvo/iron-session)および内部で使用されている[iron-webcrypto](https://github.com/brc-dd/iron-webcrypto)のソースコードに基づいて、iron-sessionの詳しい仕組みについて解説します。

※ どちらも2025/02/24時点の最新バージョンを元に解説しています。

## 概要

暗号化は**AES-256-CBC**、署名（MAC）は**HMAC-SHA256**を使用しています。  
また、それぞれで使う鍵はサーバで決めたパスワードと内部でランダムに生成したソルトから**PBKDF2**を使用して生成されます。

Cookieには暗号化されたセッション情報だけでなく、MAC値や鍵の生成に使ったソルトも保存されます。

## iron-session の使い方

iron-sessionは、以下のようにして使うことができます。

```typescript
const session = await getIronSession(req, res, sessionOptions); // セッション情報の取得、初期化

session.key = 'value'; // セッション情報の設定
const value = session.key; // セッション情報の取得

session.save(); // セッション情報の保存（resのCookieにセッション情報が保存される）

session.destroy(); // セッション情報の破棄
```

`sessionOptions`で以下のようなオプションを指定できます。

```typescript
interface sessionOptions {
  password: string | Record<string,string>; // 暗号化や署名に使用するパスワード。
  // string型だけでなく、{1: '~~', 2: '~~'}のようなオブジェクトも指定可能。最も大きな数字のキーが使用される。
  cookieName: string, // Cookieの名前
  ttl?: number, // セッション情報の有効期限（秒）。Cookieのmax-ageも同時に設定される。
  cookieOptions?: CookieOptions, // Cookieに設定するオプション。
}
```

## iron-session の大まかな仕組み

`getIronSession` および `session.save()` でどのような処理が行われているのか、その大まかな流れを見ていきます。  

### getIronSession

1. まず、`getIronSession` が呼び出されると、Cookieから`cookieName` で指定した名前のCookieを取得します。そのようなCookieが存在しない場合、3. に進みます。
2. 取得したCookieの値から、有効期限・MAC値を検証し、その後復号します。
3. `save()` や `destroy()` といったメソッドをオブジェクトに生やし、`session` オブジェクトを返します。

### session.save()

1. `session`オブジェクトを`JSON.stringify`でシリアライズした文字列を暗号化・MAC付与をします。
2. その文字列をCookieに保存します。

## seal / unseal の仕組み

暗号化・MAC付与の処理は、`iron-webcrypto` の `seal` / `unseal` 関数によって行われます。  
それぞれの仕組みを見ていきます。

前提として、一連の処理で使われる`password` は、`sessionOptions.password` の値を使用します。オブジェクトで指定された場合、最も大きな数字のキーの値が使用されます。

※ 以下の説明で暗号化アルゴリズムを**AES-256-CBC**だとしていますが、iron-webcrypto自体は**AES-128-CTR**も使用可能です。iron-sessionでは**AES-256-CBC**を使用しているため、こちらに焦点を当てて説明します。

### seal

#### 1. 暗号化

まず最初に、セッション情報を暗号化します。  
暗号化に使う鍵は **PBKDF2** を使って`password` から生成されます。イテレーションは`1`固定、ソルトは`256`ビットのランダムな値で、内部で`crypto.getRandomValues` を使って生成されます。  
また、暗号化に使うIV（Initialization Vector）も`128`ビットのランダムな値としてソルトと同様に生成されます。`save()`の度に異なる鍵が生成されるため、セッション情報が暗号化されるたびに異なる暗号化結果が得られます。

そして、これらの鍵・IVを使って、**AES-256-CBC** で暗号化を行います。暗号化対象のセッション情報は、`session`を`JSON.stringify` でシリアライズした文字列です。

#### 2. MAC付与

次に、暗号化されたセッション情報に署名（MAC）を付与します。  

暗号化のときと同様に鍵を生成し、**HMAC-SHA256** でMACを計算します。MACは以下の文字列に対して計算されます。

```javascript
`${macPrefix}*${passwordId}*${encrypt key salt}*${iv}*${base64urlEncode(encryptedData)}*${expiration}`
```

- **macPrefix**: `Fe26.${macFormatVersion}` 現在、`macFormatVersion` は`2`です。
- **passwordId**: `password` がオブジェクトで指定された場合、最も大きな数字のキーの値が使用されます。文字列の場合は`1`が使用されます。
- **encrypt key salt**: 暗号化に使った鍵のソルト
- **iv**: 暗号化に使ったIV
- **encryptedData**: 暗号化されたセッション情報
- **expiration**: セッション情報の有効期限

#### 3. 出力

最後に、以下の値を出力します。

```javascript
`${macBasedString}*${mac key salt}*${base64urlEncode(mac)}`
```

- **macBasedString**: 2でMAC計算を行ったデータ
- **mac key salt**: MAC計算に使った鍵のソルト
- **mac**: 計算されたMAC

#### 4. 保存

sealの説明は3で終わりですが、Cookieには3の出力を更に以下のように連結して保存します。

```javascript
`${seal}~${currentMajorVersion}`
```

- **seal**: 3で出力した値
- **currentMajorVersion**: `2` iron-sessionのアップデートで仕組みが変わっても処理できるようにするためのバージョン情報

### unseal

unsealは以下のような流れで行われます。

1. セッション情報から`*`で区切られた各値を取得します。
2. `expiration` が現在時刻より過去の場合、セッション情報は無効として処理を終了します。
3. `macBasedString` に対してMACを計算し、計算されたMACと`mac` を比較します。一致しない場合、セッション情報は無効として処理を終了します。
4. `macBasedString` を復号し、セッション情報を取得します。
5. セッション情報を`JSON.parse` でパースし、`session`オブジェクトを返します。

## まとめ

- 暗号化は**AES-256-CBC**、署名（MAC）は**HMAC-SHA256**を使用
- 鍵はパスワードから**PBKDF2**を使用して生成。ソルトを使い保存の度に異なる鍵が生成される。

## 参考

- iron-session([https://github.com/vvo/iron-session](https://github.com/vvo/iron-session))
- iron-webcrypto([https://github.com/brc-dd/iron-webcrypto](https://github.com/brc-dd/iron-webcrypto))
