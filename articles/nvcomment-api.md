---
title: "[新サーバー対応版]ニコニコ動画のコメント取得に関するTips"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python","niconico","nicovideo","api","ニコニコ動画"]
published: true
---

# はじめに
2022年6月ニコ動のコメントサーバーに大幅な仕様変更がありました。
新しく`nvcomment.nicovideo.jp/v1/threads`というAPIが使われるようになり、今まで使用していた`nvcomment.nicovideo.jp/legacy/api.json/thread`というAPIはURLからわかる通りレガシー版のため今後使用できなくなる可能性があります。
また本記事ではAPIのアクセス例としてPythonでのコードを記載しています。

# TL;DR
- コメントサーバーからコメント一覧を取得する
- 過去ログのコメントを取得する
- 通常コメントを投稿する

# 準備
コメントサーバーにリクエストを送る際、ログイン情報が必要な場合があります。その場合以下の2つ方法でセッションを取得してください。ログインが必要な際は項の名前に`ログイン必須`と記載しています。
ログインしていなくてもコメントの取得はできるので飛ばしてもらっても構いません。

## ブラウザから取得する
Chromeでの方法を記載しますが他のブラウザでも同じようにできると思います。
1. デベロッパーツールを開きアプリケーションタブを選択する。
2. ストレージ欄のCookieで`https://nicovideo.jp`を選択する
3. `user_session`の値部分をコピーする
これがユーザーのセッション情報なのでこれをリクエストをする際に設定してください。

## POSTをして取得する
下記記事の「1. ニコ動にログインする」を参考にしてください。
https://zenn.dev/negima1072/articles/howto-upload-nicovideo-by-api#1.-%E3%83%8B%E3%82%B3%E5%8B%95%E3%81%AB%E3%83%AD%E3%82%B0%E3%82%A4%E3%83%B3%E3%81%99%E3%82%8B

# 通常コメントを取得する
通常コメントを取得する方法を順に説明していきます。

## 1. 動画情報を取得する
まず動画情報を取得します。
必要なのは`nvComment`の部分だけです。
リクエストのレスポンス全体を見たい場合は以下のGistを参考にしてください。
https://gist.github.com/Negima1072/8d3abe48e9c2690b459da9cefcd0e89c#file-movie-dt-sm9-json

### APIサーバーを叩く(ログイン必須)
こちらはログイン情報が必須です。
```python
import requests, random, string

movieId = "[動画ID(ex. sm9]"

headers = {
  "X-Frontend-Id": "6",
  "X-Frontend-Version": "0"
}

actionTrackId = \
  "".join(random.choice(string.ascii_letters + string.digits) for _ in range(10)) \
  + "_" \
  + str(random.randrange(10**(12),10**13))

url = "https://www.nicovideo.jp/api/watch/v3/{}?actionTrackId={}" \
  .format(movieId, actionTrackId)

res = requests.post(url, headers=headers).json()

nvComment = res["data"]["comment"]["nvComment"]
```

### APIサーバーを叩く(ログイン不必要)
こちらはログイン情報が必要ないAPIです
※新しく見つけたので更新しました。
```python
import requests, random, string

movieId = "[動画ID(ex. sm9]"

headers = {
  "X-Frontend-Id": "6",
  "X-Frontend-Version": "0"
}

actionTrackId = \
  "".join(random.choice(string.ascii_letters + string.digits) for _ in range(10)) \
  + "_" \
  + str(random.randrange(10**(12),10**13))

url = "https://www.nicovideo.jp/api/watch/v3_guest/{}?actionTrackId={}" \
  .format(movieId, actionTrackId)

res = requests.post(url, headers=headers).json()

nvComment = res["data"]["comment"]["nvComment"]
```

#### 動画ページを見る
こちらはログイン情報がなくてもアクセスすることができます。
```python
from bs4 import BeautifulSoup
import requests
import json

movieId = "[動画ID(ex. sm9]"

source = requests.get("https://nicovideo.jp/watch/{}".format(movieId))
soup = BeautifulSoup(source.text, "html.parser")
elem = soup.select_one("#js-initial-watch-data")
js = json.loads(elem.get("data-api-data"))

nvComment = js["comment"]["nvComment"]
```

## 2. コメントサーバーにPOSTする
次にコメントサーバーにPOSTにコメント一覧を取得します。
```python
import requests

headers = {
  "X-Frontend-Id": "6",
  "X-Frontend-Version": "0",
  "Content-Type": "application/json"
}

params = {
  "params": nvComment["params"],
  "additionals": {},
  "threadKey": nvComment["threadKey"]
}

url = nvComment["server"] + "/v1/threads"

res = requests.post(url, json.dumps(params), headers=headers).json()
```

### レスポンス
リクエストのレスポンス全体を見たい場合は以下のGistを参考にしてください。
https://gist.github.com/Negima1072/7fd0f7a3be5eae97e553c46c5066e1b2#file-comment-dt-sm19560942-json
レスポンスの概形は以下のとおりです。

```json
{
  "meta": {
    "status": 200
  },
  "data": {
    "globalComments": [
      {
        "id": "1355259729",
        "count": 19696 //すべてのサーバーでの合計コメント数
      }
    ],
    "threads": [
      {
        "id": "1355259729", //スレッドID
        "fork": "owner", //ラベル
        "commentCount": 10, //スレッド内でのコメント数
        "comments": [...] //コメント一覧
      },
      {
        "id": "1355259729",
        "fork": "main",
        "commentCount": 10,
        "comments": [
          ...
        ]
      },
      {
        "id": "1355259729",
        "fork": "easy",
        "commentCount": 10,
        "comments": [...]
      }
    ]
  }
}
```

#### コメントレスポンス
`res.data.threads[n].comments[n]`の概形です。
```json
{
  "id": "874878644481122398", //コメントID
  "no": 16925, //動画内でのコメント番号
  "vposMs": 682300, //動画内の投稿場所
  "body": "コメントだよ", //投稿内容
  "commands": [ //コマンドのリスト
    "184",
    "device:3DS"
  ],
  "userId": "1MoG8ntWStspL0v-Wg5rdAbbn_4", //ユーザーID
  "isPremium": false, //投稿者がプレミアムかどうか
  "score": 0, //NGスコア
  "postedAt": "2014-11-14T19:51:38+09:00", //投稿された日時
  "nicoruCount": 0, //にこられた数
  "nicoruId": null, //不明
  "source": "leaf", //ソースがどこか
  "isMyPost": false //自分の投稿か
}
```
`source`についての詳細です。
|source|説明|
|------|----|
|truck |直近1000コメント|
|leaf  |1分おきに100コメントを配置すす場合truckで足りない部分を補完したコメント|

## 注意
これでコメント一覧を取得できました。やったね！！
とは行きません。このAPIはレートリミットがついています。
1分間に60リクエストまでなので気をつけましょう。

# 過去ログのコメントを取得する
さて、最近（動画上に表示される）のコメントの取得はできるようになりました。次は過去ログの取得です。

## コメントサーバーにPOSTする
通常コメントの`params.additionals`に以下のように加えます。

```python
import requests

unixTime = 1655558820 #コメントを取得したいUNIX時間

headers = {
  "X-Frontend-Id": "6",
  "X-Frontend-Version": "0",
  "Content-Type": "application/json"
}

params = {
  "params": nvComment["params"],
  "additionals": {
    "when": unixTime
  },
  "threadKey": nvComment["threadKey"]
}

url = nvComment["server"] + "/v1/threads"

res = requests.post(url, json.dumps(params), headers=headers).json()
```

## スレッドキーの再取得

`threadKey`は１０分もしないうちに有効期限が切れてしまうため、それ以降にコメントを取得するには再度上記のv3APIを叩くか、keyを再取得する必要があります。

```python
import requests

videoId = "sm9"

headers = {
  "X-Frontend-Id": "6",
  "X-Frontend-Version": "0",
  "Content-Type": "application/json"
}

res = requests.get("https://nvapi.nicovideo.jp/v1/comment/keys/thread?videoId="+videoId, headers=headers).json()

threadKey = res["data"]["threadKey"]
```

ちなみに`threadKey`はJWTエンコードされていて、デコードすると`threadId`や有効期限、ログインしている状態で取得した場合はユーザーIDを見ることができます。

# 通常コメントを投稿する(ログイン必須)
次はコメントを投稿していきましょう。

## 1. PostKeyを取得する
まずはPostKeyを取得します。

```python
import requests

threadId = nvComment["params"]["targets"][n]["id"]

url = "https://nvapi.nicovideo.jp/v1/comment/keys/post?threadId={}".format(threadId)

res = requests.get(url, headers=headers).json()

postkey = res["data"]["postKey"]
```

以下のレスポンスが帰ってきます。
```json
{
  "meta": {
    "status": 200
  },
  "data": {
    "postKey": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJqdGkiOiI2MmF*************6ZJhs5QKSMtBoyXrVEwpnd8UwZulf6ZE-lHJx2kSrmZ3kY4kW4rtUbT2_9f0aHQ"
  }
}
```

## 2. コメントサーバーにPOSTする
コメントサーバーにコメントデータをPOSTします。

```python
import requests

headers = {
  "X-Frontend-Id": "6",
  "X-Frontend-Version": "0",
  "Content-Type": "application/json"
}

params = {
  "videoId": "so30413239", #動画ID
  "commands": [ #コマンドリスト
    "small"
  ],
  "body": "わーい", #コメント内容
  "vposMs": 4000, #コメント投稿時間
  "postKey": postKey #Postkey
}

threadId = nvComment["params"]["targets"][n]["id"]

url = "https://nvcomment.nicovideo.jp/v1/threads/{}/comments".format(threadId)

res = requests.post(url, json.dumps(params), headers=headers).json()
```

以下のレスポンスが帰ってきます
```json
{
  "meta": {
    "status": 201 //Created
  },
  "data": {
    "no": 2550510, //動画内のコメント番号
    "id": "988079784753934336" //ユニークなコメントID
  }
}
```

# まとめ
適当にまとめてみましたがわかりにくい！！
なにか問題があればprかコメ欄でお願いします。

#### 追記
ニコ動アプリもニコ生アプリもNicoBoxもログインするかゲストアカウントを作成するかで必ずユーザーデータが紐付けられている上、ログインしなくても見れるPCブラウザではHTMLパースでしか取得できない。ゲストアカウントはAPIで動画情報取得できないのかー
モバイル版のブラウザのデータ見てたときー＞はぁなんだよv3_guestって！！！（泣）