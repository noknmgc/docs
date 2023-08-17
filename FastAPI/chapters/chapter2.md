# ディレクトリ構成

ここでは、FastAPI が配布しているテンプレート[フルスタック FastAPI PostgreSQL](https://github.com/tiangolo/full-stack-fastapi-postgresql)のディレクトリ構成を参考にし、ディレクトリを作成していきます。

## 目次

- [ディレクトリ構成](#ディレクトリ構成)
  - [目次](#目次)
  - [ディレクトリ作成](#ディレクトリ作成)
  - [各ディレクトリについて(スキップ可)](#各ディレクトリについてスキップ可)
    - [api](#api)
    - [core](#core)
    - [crud](#crud)
    - [db](#db)
    - [models](#models)
    - [schemas](#schemas)
  - [Chapter1 のリファクタリング](#chapter1-のリファクタリング)
    - [router を使ったファイル分け](#router-を使ったファイル分け)
    - [動作確認](#動作確認)

## ディレクトリ作成

`app`ディレクトリ配下に`api`、`core`、`crud`、`db`、`models`、`schemas`ディレクトリを作成します。さらに`api`ディレクトリ配下に`endpoints`ディレクトリを作成してください。ディレクトリ構成は、以下のようになります。

```
app
├ api
│  └ endpoints
├ core
├ crud
├ db
├ models
├ schemas
├ __init__.py
└ main.py
```

また、各ディレクトリに`__init__.py`を作成してください。中身が空で大丈夫です。

```
app
├ api
│  ├ endpoints
│  │  └ __init__.py
│  └ __init__.py
├ core
│  └ __init__.py
├ crud
│  └ __init__.py
├ db
│  └ __init__.py
├ models
│  └ __init__.py
├ schemas
│  └ __init__.py
├ __init__.py
└ main.py
```

## 各ディレクトリについて(スキップ可)

各ディレクトリの役割について記述します。

### api

FastAPI のパスオペレーション関数を定義します。

### core

`config.py`と`security.py`を作成します。`config.py`には DB の URI や、暗号化に使う KEY など設定値を記述します。`security.py`には、パスワードのハッシュ化や Json Web Token の発行を行う関数を定義します。

### crud

SQLAlchemy の CRUD 操作について記述します。

### db

`base.py`と`session.py`を作成します。`base.py`には、SQLAlchmey のモデルベースクラス(Base)を定義します。ここで定義した Base を継承して ORM を扱えるモデルクラスを定義します。`session.py`には、DB エンジンとセッションを作るクラスを sessionmaker で作ります。

### models

`db/base.py`で定義した Base を継承して SQLAlchemy のモデルクラスを定義します。

### schemas

FastAPI で利用する API のリクエストとレスポンスの型を定義します。

## Chapter1 のリファクタリング

Chapter1 に記述したコードをこのディレクトリ構成に合うようにリファクタリングしていきます。

### router を使ったファイル分け

Chapter1 で`main.py`に記述したパスオペレーション関数は、`app/api/endpoints`配下にファイルを作成し、定義していきます。Chapter1 では、`app = FastAPI()`とし`app`にパスオペレーション関数を追加していきましたが、ファイルを分ける場合は、`APIRouter`を使います。

`app/api/endpoints`配下に以下のファイルを記述しましょう。
`app/api/endpoints/root.py`

```python
from fastapi import APIRouter

router = APIRouter()


@router.get("/")
def root():
    return {"API": "Active"}
```

今後、ここにエンドポイントの種類ごとにファイルを追加していきます。そこで定義した router を 1 つにまとめる記述を`app/api/api.py`にします。以下のようなファイルを作成しましょう。

`app/api/api.py`

```python
from fastapi import APIRouter

from app.api.endpoints import root

api_router = APIRouter()
api_router.include_router(root.router, tags=["test"])
```

Chapter1 で作成した`app/main.py`は、`app/api/api.py`で 1 つにまとめられた router を`app = FastAPI()`に加えます。以下のように変更します。

`app/main.py`

```python
from fastapi import FastAPI

from app.api.api import api_router

app = FastAPI()

app.include_router(api_router)
```

### 動作確認

chapter1 と同じように`app`ディレクトリがあるディレクトリで、以下のコマンドを実行し、サーバーを起動します。

```bash
uvicorn app.main:app --reload --port 8000
```

ブラウザで[http://127.0.0.1:8000](http://127.0.0.1:8000)を開きます。次のような JSON レスポンスが表示されれば、リファクタリング完了です。

```json
{ "API": "Active" }
```

Chapter1 と同じように[Swagger UI](http://127.0.0.1:8000/docs)を開いてみましょう。

![SwaggerUI](..\images\ch2_swagger_ui.png)

Chapter1 とほぼ同じ画面が表示されているかと思いますが、1 箇所**default**から**test**に変わったところがあると思います。これは、`app/api/api.py`で定義した`api_router.include_router`の引数の`tags=["test"]`によって設定されています。機能ごとに tags を設定するとこのようにドキュメントも整理されるので、tags は書くようにしましょう。
