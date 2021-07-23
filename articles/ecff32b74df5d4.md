---
title: "[Python]ブクログに登録した本一覧を月別で出力してみる"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['python']
published: true
---

## やりたいこと

僕は普段ブクログというサービスを使って読んだ本を記録しています。ブクログに登録した書籍を月別で分類し、いつどんな本を読んでいたのかを整理したいと思いました。
Pythonを勉強中なので、Pythonを使ってこれを実装してみます。

https://booklog.jp/users/4165b902f43abd44

## 成果物

最終的なコードと出力ファイルは以下のGistに置きました。

https://gist.github.com/kenzo-tanaka/75aca94aa4033331cd5f135f5d467f74

https://gist.github.com/kenzo-tanaka/3ffb1aacdef3aac0ad941d0b25e49cc0

## ブクログAPIを叩いて書籍を取得する

`booklog api`などで調べると`http://api.booklog.jp/v2/json/<userId>?count=10000`に対してリクエストを送ると、ユーザーの本棚情報を取得できることが分かりました。
このAPIは公式のドキュメントなども見当たらなかったので、動かして試してみます。

```py:booklog.py
import requests
import json

api_res=requests.get("http://api.booklog.jp/v2/json/4165b902f43abd44?count=10000")
json_res=json.loads(api_res.text)

print(json_res)
```

```shell
$ python3 booklog.py
[{'url': 'https://booklog.jp/users/4165b902f43abd44/archives/1/487311893X', 'title': '初めてのGraphQL ―Webサービスを作って学ぶ新世代API', 'image': 'https://m.media-amazon.com/images/I/51Z8dT721zL._SL75_.jpg', 'catalog': 'book'}, ...]
```

レスポンスとして取得した情報として、以下が使えそうです。

- url: ユーザーごとの書籍詳細ページ
- title: 書籍のタイトル

逆にブクログに書籍をいつ登録したのかの登録日が取れていないので、これを取得する必要があります。

## 書籍詳細ページからスクレイピングで登録日を取得

上記APIから取得したURLがユーザーごとの書籍詳細ページになっていて、ここに「本棚登録日」という項目があるので、これをスクレイピングで取得します。

![](/images/booklog/img1.png)

```py:booklog.py
from bs4 import BeautifulSoup
import requests

html = requests.get('https://booklog.jp/users/4165b902f43abd44/archives/1/487311893X')
soup = BeautifulSoup(html.content, "html.parser")
register_date=soup.find(class_='read-day-status-area').find('span').text
print(register_date)
```

```shell
$ python3 booklog.py
2021年7月22日
```

同じ要領でAmazonのリンクも取得できるので、取得します。

```py:booklog.py
from bs4 import BeautifulSoup
import requests

html = requests.get('https://booklog.jp/users/4165b902f43abd44/archives/1/487311893X')
soup = BeautifulSoup(html.content, "html.parser")
amazon_link=soup.find(class_='itemInfoElm').find('a').get('href')
print(amazon_link)
```

```shell
$ python3 test.py
https://www.amazon.co.jp/dp/487311893X?tag=booklogjp-default-22&linkCode=ogi&th=1&psc=1
```

## メソッドに分割

色々と処理が増えてきたので、メソッドに処理を分割してみます。

```py:booklog.py
import requests
from bs4 import BeautifulSoup
import json

def get_soup(url):
    html = requests.get(url)
    soup = BeautifulSoup(html.content, "html.parser")
    return soup

def get_amazon_link(book):
    soup = get_soup(book['url'])
    amazon_link=soup.find(class_='itemInfoElm').find('a').get('href')
    return amazon_link

def get_register_date(book):
    soup = get_soup(book['url'])
    register_date=soup.find(class_='read-day-status-area').find('span').text
    return register_date

def main():
    api_res=requests.get("http://api.booklog.jp/v2/json/4165b902f43abd44?count=10000")
    json_res=json.loads(api_res.text)
    books=json_res['books']

    for book in books:
        book['amazon_link'] = get_amazon_link(book)
        book['register_date'] = get_register_date(book)

if __name__ == "__main__":
    main()
```

## 月毎で分類できるよう日付を整形

最終的に月で分類していくために、`groupby`関数を使いたいと思います。`groupby`を使うには、グルーピングしたいキーでソートする必要があります。
https://techblog.recochoku.jp/5117

なので、`register_date`に入っている「2021年7月22日」のような文字列を「202107」などに整形していこうと思います。

`strptime()`を使えば文字列から日付へ変換ができるので、これを使います。

ブクログをスクレイピングして取得した日付は「2021年7月22日」のように月に0が含まれません。正しくソートするためには、0埋めをしないといけないため`f'{date_dt.month:02}'`と書いています。

```py:booklog.py
# 省略...

def format_date(date_str):
    date_format = '%Y年%m月%d日'
    # 2021-07-22 00:00:00 のように変換
    date_dt = datetime.datetime.strptime(date_str, date_format)

    year=date_dt.year

    # 0埋めで2文字
    month=f'{date_dt.month:02}'

    date=f'{year}{month}'
    return date

def main():
    api_res=requests.get("http://api.booklog.jp/v2/json/4165b902f43abd44?count=10000")
    json_res=json.loads(api_res.text)
    books=json_res['books']

    for book in books:
        book['amazon_link'] = get_amazon_link(book)
        book['register_date'] = format_date(get_register_date(book))

if __name__ == "__main__":
    main()
```

参考:
https://note.nkmk.me/python-datetime-usage/
https://www.atmarkit.co.jp/ait/articles/2008/21/news017.html


## 最終結果をマークダウンに書き出す

あとは、分類しながら出力するだけです。

```py:booklog.py
def main():
    api_res=requests.get("http://api.booklog.jp/v2/json/4165b902f43abd44?count=10000")
    json_res=json.loads(api_res.text)
    books=json_res['books']

    # amazonリンクと登録日を取得
    for book in books:
        book['amazon_link'] = get_amazon_link(book)
        book['register_date'] = format_date(get_register_date(book))

    f=open('books.md', 'w')
    books.sort(key=lambda b: int(b['register_date'])) # 登録年月でソート
    for key, group in groupby(books, key=lambda b: b['register_date']):
        f.write(f'\n## {key}\n\n') # 登録年月を見出しにする
        for book in group:
            title=book['title']
            url=book['amazon_link']
            f.write(f'- [{title}]({url})\n') # 書籍をリンクで出力

    f.close()
```

## 終わりに

今年もたくさん本を読みたいです。