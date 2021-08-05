---
title: "ニコ動で動画をAPIからアップロードする"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python","niconico","nicovideo","api","ニコニコ動画"]
published: true
---

ニコニコ動画の動画をAPIを用いて外部から投稿するやり方をまとめました。

手順としては
1. ニコニコ動画にログインしセッション情報を取得
2. 投稿リクエストを送信する
3. 動画をチャンクに分けてアップロードする
4. メタデータを送信する

**※注意　内部APIで本来非公開のものなのでいつ仕様変更があるかわかりません。自己責任で使用してください。**

# 実践
## 1. ニコ動にログインする
まずログインAPIにアクセスしメールアドレスとパスワードでログイン後、ユーザーIDとセッション情報を取得します。

```python
import requests

session = requests.session()
url = "https://secure.nicovideo.jp/secure/login?site=niconico"
params = {
    "mail": "example@example.com",
    "password": "password"
}

response=session.post(url, params=params)
user_session = session.cookies.get("user_session") #セッション情報
user_id = response.headers.get("x-niconico-id") #ユーザーID
```

## 2. 投稿リクエストを送信する
まずはニコ動側に動画を投稿したいという意思を伝えます。

```python
url = "https://www.upload.nicovideo.jp/v2/videos"
headers = {
    "X-Request-With":"N-garage"
}
response=session.post(url, headers=headers)
js = response.json()

videoid = str(js["data"]["id"]) #動画ID
```

これを実行すると以下のレスポンスを得られます。
```json
{
    "meta":{
        "status":201
    },
    "data":{
        "id":39130867,
        "videoId":"sm39130867", //動画のID
        "status":{ //各状況ステータス
            "postStatus":"CREATED",
            "videoFileStatus":"NOT-UPLOAD",
            "thumbnailStatus":"NOT-AVAILABLE",
            "inputStatus":"INCOMPLETE"
        }
    }
}
```

このときすでに何かしらを投稿中・エンコード中等の場合エラーが発生します。
ニコ動は同時に一つしかアップロードできないのでしょうがないですね。
```json
{"meta":{"error-code":"ID_ALREADY_EXISTS","status":409},"data":[{"id":39130867}]}
```

## 3.動画をアップロードする
### 動画をアップロードするURLを取得する
次に動画をアップロードする必要がありますがそのためのURLを取得します。

```python
url = "https://www.upload.nicovideo.jp/v2/videos/" + videoid + "/upload-chunk-stream"
response = session.post(url, headers=headers)
js = response.json()

chunkurl = "https://www.upload.nicovideo.jp" + js["data"]["url"]
```

これを実行すると以下のレスポンスを得られます。
```json
{
    "meta":{
        "status":201
    },
    "data":{
        "url":"/v2/videos/39130867/file/chunks", //URLのパス
        "token":"uI7v9brAK5irk8skTHywjZzyBJBD7lcTJkObMWvVjgm98TJ9TD48uqX9" //特に必要なし
    }
}
```

### 動画をアップロード
次に動画ファイルをサーバー上にアップロードします。
動画ファイルを10000000バイトごとに分割して送信しないといけないため複雑な処理になります。

```python
from http.cookiejar import CookieJar, Cookie
import uuid
import urllib
import urllib.request
import json

uuid = str(uuid.uuid4())

chunks = []
datalen = 0
with open("./downloads/movie.mp4", "rb") as f:
    while True:
        c = f.read(10000000)
        if len(c) == 0:
            break
        chunks.append(c)
        datalen += len(c)

boundary = "----WebKitFormBoundary" + "1234567890"

cj = CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cj))
c = Cookie(0, "user_session", user_session, None, False, ".nicovideo.jp", True, True, "/", True, False, "9999999999", False, None, None, {})
cj.set_cookie(c)


for i in range(len(chunks)):
    bdata = []
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqpartindex\"")
    bdata.append("")
    bdata.append(str(i))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqpartbyteoffset\"")
    bdata.append("")
    bdata.append(str(datalen))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqchunksize\"")
    bdata.append("")
    bdata.append(str(len(chunks[i])))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqtotalparts\"")
    bdata.append("")
    bdata.append(str(len(chunks)))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqtotalfilesize\"")
    bdata.append("")
    bdata.append(str(datalen))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqfilename\"")
    bdata.append("")
    bdata.append("movie.mp4")
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qquuid\"")
    bdata.append("")
    bdata.append(uuid)
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqfile\"; filename=\"blob\"")
    bdata.append("Content-Type: application/octet-stream")
    bdata.append("")
    bdataByted = "\n".join(bdata).encode("shift_jis")
    adata = '--' + boundary
    adataByted = adata.encode("shift_jis")
    data = bdataByted + b"\n" + chunks[i] + b"\n" + adataByted

    req = urllib.request.Request(chunkurl)
    req.add_header("Content-Type", "multipart/form-data; boundary=%s" % boundary)
    req.add_header("X-Request-With", "N-garage")
    req.add_header("X-Requested-With", "XMLHttpRequest")
    with opener.open(req, data) as response:
        js = json.loads(response.read().decode('utf8'))
```

このときアップロードに成功すると以下のJSONが返ってきます。
```json
{"success": true, "uuid": "382fda4e-bbf1-48af-8fcd-105a29f5376f"}
```

### アップロードの完了を報告する
次にすべてのchunkがアップロードし終わったことをサーバーに報告します。
```python
params = {
    "qquuid": uuid,
    "qqfilename": "movie.mp4",
    "qqtotalfilesize": str(datalen),
    "qqtotalparts": str(len(chunks))
}
res = session.post(chunkurl + "?done", params=params, headers=headers)
js = res.json()
```

これを実行すると以下のレスポンスを得られます。
```json
{
    "success":true,
    "uuid":"382fda4e-bbf1-48af-8fcd-105a29f5376f",
    "data":{
        "id":39130867,
        "cancelToken":"DpMOxAhI7KKNx6ViJOLBLnrby73b7WILr08zzdsfdsfhsoifs"
    }
}
```

## 4.メタデータを送信する
### サムネイルの位置を指定する
メタデータを送信する前にサムネイルの場所を指定します。これを実行しない場合サーバー側でランダムに選ばれた場所がサムネイルになります。
```python
url = "https://www.upload.nicovideo.jp/v2/videos/" + videoid + "/scene-thumbnails/0"
params = {
    "timing": 20263 #サムネイルの位置を1/100秒で指定(=20.263秒)
}
response = session.post(url, params=params, headers=headers)
js = response.json()
```

これを実行すると以下のレスポンスを得られます。
```json
{
    "meta":{
        "status":200
    },
    "data":{
        "key":"0",
        "timing":20263,
        "resolution":{
            "height":90,
            "width":182
        },
        "dataUri":"data:image\/jpeg;base64,\/9j\/4AAQSkZJ~~~~"
    }
}
```

### メタデータの送信
最後にメタデータ(動画情報)を送信します。
```python
params = {
	"id": videoid,
	"title": "movie", #動画のタイトル
	"detail": "asas", #動画の説明文(HTML可)
	"signature": {
		"display": False #共通説明文を表示するか
	},
	"publish": True, #公開するかしないか
	"publishTimer": {
		"use": False #公開タイマーを使うか使わないか
	},
	"community": {
		"belongs": True, #コミュニティーに所属させるかどうか
		"id": 10, #上がTrueの場合はコミュニティーID(co~~),Falseのときは空
		"communityOnly": True, #publish=TrueのときこれをTrueにするとコミュ限投稿になる
		"groupWorkFlag": False #投コメをコミュメンバーが編集できるかどうか
	},
	"tags": [ #タグの指定(最大6件)
		"tag1",
        "tag2"
	],
	"genreKey": "animal", #ジャンル("none"は未指定)
	"thumbnail": {
		"selectThumbnailIndex": "0",
		"aspectBias": "STANDARD", #動画サイズが16:9より横長のときはWIDE、縦長のときはTALL
		"cropMode": 0, #STANDART=0,TALL=1,WIDE=2
		"position": 0
	},
	"permissionSettings": {
		"allowNgShareFlag": True,
		"allowOutsidePlayerFlag": True,
		"allowUadFlag": True,
		"allowNicoliveFlag": True,
		"allowUserTranslateFlag": True,
		"allowRegularUserTagEditFlag": True
	},
	"notification": {
		"email": False
	},
	"excludeFromUploadList": False,
	"thanksMessage": {
		"isVisible": False, #いいねのお礼を表示するか
		"content": ""
	}
}
url = "https://www.upload.nicovideo.jp/v2/videos/" + videoid

headers["Content-Type"] = "application/json" #Content-Typeを追加
response = session.post(url, json.dumps(params), headers=headers)
js = response.json()
```

これで以下のレスポンスが帰ってきたら成功です
```json
{"meta":{"status":200}}
```

[アップロードページ](https://www.upload.nicovideo.jp/garage/videos)を確認し、新しい動画があることを確認してください。

## 全体のソースコード
```python
from http.cookiejar import CookieJar, Cookie
import uuid
import urllib
import urllib.request
import json
import requests

session = requests.session()
url = "https://secure.nicovideo.jp/secure/login?site=niconico"
params = {
    "mail": "example@example.com",
    "password": "password"
}

response=session.post(url, params=params)
user_session = session.cookies.get("user_session") #セッション情報
user_id = response.headers.get("x-niconico-id") #ユーザーID

url = "https://www.upload.nicovideo.jp/v2/videos"
headers = {
    "X-Request-With":"N-garage"
}
response=session.post(url, headers=headers)
js = response.json()

videoid = str(js["data"]["id"]) #動画ID

url = "https://www.upload.nicovideo.jp/v2/videos/" + videoid + "/upload-chunk-stream"
response = session.post(url, headers=headers)
js = response.json()

chunkurl = "https://www.upload.nicovideo.jp" + js["data"]["url"]

uuid = str(uuid.uuid4())

chunks = []
datalen = 0
with open("./downloads/movie.mp4", "rb") as f:
    while True:
        c = f.read(10000000)
        if len(c) == 0:
            break
        chunks.append(c)
        datalen += len(c)

boundary = "----WebKitFormBoundary" + "1234567890"

cj = CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cj))
c = Cookie(0, "user_session", user_session, None, False, ".nicovideo.jp", True, True, "/", True, False, "9999999999", False, None, None, {})
cj.set_cookie(c)


for i in range(len(chunks)):
    bdata = []
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqpartindex\"")
    bdata.append("")
    bdata.append(str(i))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqpartbyteoffset\"")
    bdata.append("")
    bdata.append(str(datalen))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqchunksize\"")
    bdata.append("")
    bdata.append(str(len(chunks[i])))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqtotalparts\"")
    bdata.append("")
    bdata.append(str(len(chunks)))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqtotalfilesize\"")
    bdata.append("")
    bdata.append(str(datalen))
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqfilename\"")
    bdata.append("")
    bdata.append("movie.mp4")
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qquuid\"")
    bdata.append("")
    bdata.append(uuid)
    bdata.append('--' + boundary)
    bdata.append("Content-Disposition: form-data; name=\"qqfile\"; filename=\"blob\"")
    bdata.append("Content-Type: application/octet-stream")
    bdata.append("")
    bdataByted = "\n".join(bdata).encode("shift_jis")
    adata = '--' + boundary
    adataByted = adata.encode("shift_jis")
    data = bdataByted + b"\n" + chunks[i] + b"\n" + adataByted

    req = urllib.request.Request(chunkurl)
    req.add_header("Content-Type", "multipart/form-data; boundary=%s" % boundary)
    req.add_header("X-Request-With", "N-garage")
    req.add_header("X-Requested-With", "XMLHttpRequest")
    with opener.open(req, data) as response:
        js = json.loads(response.read().decode('utf8'))

params = {
    "qquuid": uuid,
    "qqfilename": "movie.mp4",
    "qqtotalfilesize": str(datalen),
    "qqtotalparts": str(len(chunks))
}
res = session.post(chunkurl + "?done", params=params, headers=headers)
js = res.json()

url = "https://www.upload.nicovideo.jp/v2/videos/" + videoid + "/scene-thumbnails/0"
params = {
    "timing": 20263 #サムネイルの位置を1/100秒で指定(=20.263秒)
}
response = session.post(url, params=params, headers=headers)
js = response.json()

params = {
	"id": videoid,
	"title": "movie", #動画のタイトル
	"detail": "asas", #動画の説明文(HTML可)
	"signature": {
		"display": False #共通説明文を表示するか
	},
	"publish": True, #公開するかしないか
	"publishTimer": {
		"use": False #公開タイマーを使うか使わないか
	},
	"community": {
		"belongs": True, #コミュニティーに所属させるかどうか
		"id": 10, #上がTrueの場合はコミュニティーID(co~~),Falseのときは空
		"communityOnly": True, #publish=TrueのときこれをTrueにするとコミュ限投稿になる
		"groupWorkFlag": False #投コメをコミュメンバーが編集できるかどうか
	},
	"tags": [ #タグの指定(最大6件)
		"tag1",
        "tag2"
	],
	"genreKey": "animal", #ジャンル("none"は未指定)
	"thumbnail": {
		"selectThumbnailIndex": "0",
		"aspectBias": "STANDARD", #動画サイズが16:9より横長のときはWIDE、縦長のときはTALL
		"cropMode": 0, #STANDART=0,TALL=1,WIDE=2
		"position": 0
	},
	"permissionSettings": {
		"allowNgShareFlag": True,
		"allowOutsidePlayerFlag": True,
		"allowUadFlag": True,
		"allowNicoliveFlag": True,
		"allowUserTranslateFlag": True,
		"allowRegularUserTagEditFlag": True
	},
	"notification": {
		"email": False
	},
	"excludeFromUploadList": False,
	"thanksMessage": {
		"isVisible": False, #いいねのお礼を表示するか
		"content": ""
	}
}
url = "https://www.upload.nicovideo.jp/v2/videos/" + videoid

headers["Content-Type"] = "application/json" #Content-Typeを追加
response = session.post(url, json.dumps(params), headers=headers)
js = response.json()
```

# おまけ
おまけで動画を削除するサンプルを載せます。

## 動画を削除する
```python
headers = {
    "X-Request-With": "N-garage"
}
session.delete("https://nvapi.nicovideo.jp/v1/users/me/videos/sm"+videoid, headers=headers)
```

# 最後に
今回はPythonを使ってやりましたが、自分が最初にこのAPIを使用したのはC#で、今回記事用にわかりやすくするためにPythonに変換しました。
もっと良いやり方や指摘等があれば、[Twitter](https://twitter.com/Negima1072)かGithubのプルリク、コメント等でお願いいたします。