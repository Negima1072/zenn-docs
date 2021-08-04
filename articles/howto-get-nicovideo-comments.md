---
title: "最もかんたんにログインせずニコニコ動画のコメントを取得する"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["niconico","nicovideo","api","ニコニコ動画"]
published: true
---

ニコニコ動画のコメントサーバーが新サーバーに移行されたため一部コメント取得の方法が変わっていた為紹介します。
@[card](https://blog.nicovideo.jp/niconews/141893.html)

流れとしては
1. ニコニコ動画の動画ページHTMLから情報を取得
2. それを用いてコメントサーバーにGETリクエスト

**※注意　この方法ではログインしない代わりに、ニコるの数・最後にニコられた時間を取得できません。ニコるの情報を取得したい場合はログインしてPOST形式でリクエストしてください。↓参考(ただしリクエストURLに関しては要確認)**
https://qiita.com/tor4kichi/items/74939b49954d3e72d789

# 実践
## 1. リクエストするための情報を取得
ニコニコ動画の動画に関する情報は`#js-initial-watch-data`上の`data-api-data`属性にJSON形式であります。
![](/images/howto-get-nicovideo-comments/sc01.png)

ここで取得するものは`data-api-data`の`comment.threads`以下です。

```python
from bs4 import BeautifulSoup
import requests
import json

source = requests.get("https://nicovideo.jp/watch/sm○○")
soup = BeautifulSoup(source.text, "html.parser")
elem = soup.select_one("#js-initial-watch-data")
js = json.loads(elem.get("data-api-data"))
threads = js["comment"]["threads"]
```

`comment.threads`は以下のようにコメントの種類ごとにコメントサーバーへリクエストするための情報が格納されています。

```json
[{
    "id": 1173108780,
    "fork": 1,
    ~~~
    "label": "owner", //投稿者コメント
    "postkeyStatus": 0,
    "server": "https://nmsg.nicovideo.jp"
}, {
    "id": 1173108780,
    "fork": 0,
    ~~~
    "label": "default", //通常コメント
    "postkeyStatus": 0,
    "server": "https://nmsg.nicovideo.jp"
}, {
    "id": 1173108780,
    "fork": 2,
    ~~~
    "label": "easy", //かんたんコメント
    "postkeyStatus": 0,
    "server": "https://nmsg.nicovideo.jp"
}]
```

## 2. コメントサーバーにリクエストする
前項でリクエストするための情報を取得しました。 通常コメントだけ取得したいのならば`threads[].label=default`のものを、全て取得したいならforeachで行えばよいでしょう。今回は通常コメントのみを取得します。

リクエスト先のURLは`threads[].server+"/api.json/thread"`です。serverのURLは投稿日時によって変わるので**必ず**ここを参照してください。
またリクエストに必要なパラメータは以下です。
|パラメータ(*は必須)|説明|
|---|---|
|thread*|`threads[].id`|
|version*|20090904を指定|
|scores|レスポンスにNGスコア情報を追加するか否か(1=追加, 0=しない) 指定しないと0|
|fork|`threads[].fork` 指定しないと0|
|language|海外サーバーのコメントも取得できる 0=日本 1=英語 2=中国 指定しないと日本|
|res_from*|どこのコメントから取得するか指定 コメントNoでも最新からn件(最大最新1000件)でも取得できる ex.45809 -1000|
|when|UNIXタイムで時間を指定することでその当時のコメントを取得(過去ログ機能)|

レスポンスはJSON形式の配列で返ってきます。もしXML形式が良い場合はリクエスト先のURLを`api.json`でなくて`api`のみにすると良いでしょう。

```python
import requests

params = {
    "thread": threads[1]["id"],
    "version": "20090904",
    "scores": "1",
    "fork": threads[1]["fork"],
    "language": "0",
    "res_from": "-1000"
}
url = threads[1]["server"] + "/api.json/thread"

res = requests.get(url, params=params)
resjs = json.loads(res)
```

以下のようにthreadの情報、leafの情報、そして各コメント(chat)の情報が配列となって返ってきます。

```json
[
    {"thread": { //threadは必ず返ってくる
        "resultcode": 0,  //成功したかどうか 失敗の場合は1以上
        "thread": "1173108780",  //thread id
        "last_res": 5186301, //コメント数(一番最新のコメントNo)
        ~~~
    }},
    {"leaf": {~~~}},
    ~~~
    {"chat": {
        "thread": "1173108780",  //thread id
        "no": 5185302,  //コメントNo
        "vpos": 2909,  //投稿された場所(1/100秒 よって左は29.09秒)
        "date": 1627567608,  //投稿時間(UNIX)
        "date_usec": 673929,  //投稿時間マイクロ秒
        "score": -12350, //NGスコア
        "anonymity": 1, 
        "user_id": "2kNzkNewE7Y6DzwzeJhfB997mcE", 
        "mail": "green big 184", //コマンド
        "content": "▒▓████▓▒░悪霊退散▒▓████▓▒░悪霊退散▒▓████▓▒░悪霊退散▒▓████▓▒░悪霊退散▒▓████▓▒░悪霊退散▒▓████▓▒" //内容
    }}
    ~~~
]
```

## 全体のソースコード
```python
from bs4 import BeautifulSoup
import requests
import json

source = requests.get("https://nicovideo.jp/watch/sm○○")
soup = BeautifulSoup(source.text, "html.parser")
elem = soup.select_one("#js-initial-watch-data")
js = json.loads(elem.get("data-api-data"))
threads = js["comment"]["threads"]

params = {
    "thread": threads[1]["id"],
    "version": "20090904",
    "scores": "1",
    "fork": threads[1]["fork"],
    "language": "0",
    "res_from": "-1000"
}
url = threads[1]["server"] + "/api.json/thread"

res = requests.get(url, params=params)
resjs = json.loads(res.text)

chats = []
for el in resjs:
    if "chat" in el:
        chats.append(el["chat"])
```

# 参考
https://blog.nicovideo.jp/niconews/141893.html
https://dwango.github.io/niconico/jikken-housou/comment-server/
https://qiita.com/tor4kichi/items/74939b49954d3e72d789