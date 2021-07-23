---
title: "[Python]ブクログに登録した本一覧を月別で出力してみる"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['python']
published: false
---

## やりたいこと

僕は普段ブクログというサービスを使って読んだ本を記録しています。ブクログに登録した書籍を月別で分類し、いつどんな本を読んでいたのかを整理したいと思いました。
Pythonを勉強中なので、Pythonを使ってこれを実装してみます。

https://booklog.jp/users/4165b902f43abd44

## 成果物

最終的なコードと出力ファイルは以下のGistに置きました。

https://gist.github.com/kenzo-tanaka/75aca94aa4033331cd5f135f5d467f74

https://gist.github.com/kenzo-tanaka/3ffb1aacdef3aac0ad941d0b25e49cc0

## ブクログAPI(非公式?)を叩いて書籍を取得する

`booklog api`などで調べると`http://api.booklog.jp/v2/json/<userId>?count=10000`に対してリクエストを送ると、ユーザーの本棚情報を取得できることが分かりました。

```py
import requests
import json

api_res=requests.get("http://api.booklog.jp/v2/json/4165b902f43abd44?count=10000")
json_res=json.loads(api_res.text)
json_res=json.loads(api_res.text)
```