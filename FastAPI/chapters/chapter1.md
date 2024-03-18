# Chapter1 FastAPI はじめのステップ

## 目次

- [Chapter1 FastAPI はじめのステップ](#chapter1-fastapi-はじめのステップ)
  - [目次](#目次)
  - [パッケージのインストール](#パッケージのインストール)
  - [FastAPI を使ってみる](#fastapi-を使ってみる)
    - [スクリプトの記述](#スクリプトの記述)
    - [サーバーの起動](#サーバーの起動)
    - [チェック](#チェック)
    - [対話的 API ドキュメント(Swagger UI)](#対話的-api-ドキュメントswagger-ui)
  - [(補足) def か async def か](#補足-def-か-async-def-か)
  - [Next: Chapter2 ディレクトリ構成](#next-chapter2-ディレクトリ構成)

## パッケージのインストール

以下のコマンドで FastAPI をインストールします。

```bash
pip install fastapi
```

また、サーバーとして利用できる uvicorn もインストールします。

```bash
pip install "uvicorn[standard]"
```

## FastAPI を使ってみる

### スクリプトの記述

まず、作業を行うディレクトリを作成してください。ここでは、`app`というディレクトリを作成します。ディレクトリに以下 2 つのファイルを作成してください。

`app/__init__.py`

```python

```

`app/main.py`

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def root():
    return {"message": "Hello World"}
```

ディレクトリの構成は、以下のようになります。

```
app
├ __init__.py
└ main.py
```

### サーバーの起動

`app`ディレクトリがあるディレクトリで、以下のコマンドを実行し、サーバーを起動します。

```bash
uvicorn app.main:app --reload --port 8000
```

### チェック

ブラウザで[http://127.0.0.1:8000](http://127.0.0.1:8000)を開きます。
次のような JSON レスポンスが表示されれば、成功です！

```json
{ "message": "Hello World" }
```

### 対話的 API ドキュメント(Swagger UI)

FastAPI では、`/docs`にアクセスすることで、自動生成されたドキュメントにアクセスすることができます。
ブラウザで[http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)にアクセスしてみましょう。次のような画面が表示されます。

![SwaggerUI](..\images\first_swagger_ui.png)

ここでは、自分が作成したエンドポイントを試すことができます。エンドポイントを開き、`Try it out`と書かれたボタンを押すと、パラメータを設定する画面が開き、`Execute`を押せば設定したパラメータでリクエストを送ることができます。

![SwaggerUI Execute](..\images\first_swagger_ui_execute.png)

## (補足) def か async def か

- async def を使うケース
  - async/await をサポートしているライブラリを利用する場合。
  - 他の何とも通信せず、応答を待つ必要がない場合。
- def を使うケース（ほとんどの場合は、こっち）
  - async/await をサポートしていないライブラリを利用する場合。
  - よく分からない場合。

公式サイトで[async def に関する解説](https://fastapi.tiangolo.com/ja/async/)があります。

## [Next: Chapter2 ディレクトリ構成](../chapters/chapter2.md)
