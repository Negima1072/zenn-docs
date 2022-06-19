---
title: "[新サーバー対応版]ニコニコ動画のコメントに関するTips"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python","niconico","nicovideo","api","ニコニコ動画"]
published: false
---

# はじめに
2022年6月ニコ動のコメントサーバーに大幅な仕様変更がありました。
新しく`nvcomment.nicovideo.jp/v1/threads`というAPIが使われるようになり、今まで使用していた`nvcomment.nicovideo.jp/legacy/api.json/thread`というAPIはURLからわかる通りレガシー版のため今後使用できなくなる可能性があります。
また本記事ではAPIのアクセス例としてPythonでのコードを記載しています。

# TL;DR
- コメントサーバーからコメント一覧を取得する
- コメント取得のTips
- 通常コメントを投稿する

# 準備
コメントサーバーにリクエストを送る際、ログイン情報が必要な場合があります。その場合以下の2つ方法でセッションを取得してください。ログインが必要な際は項の名前に`ログイン必須`と記載しています。

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

### 動画ページを見る
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