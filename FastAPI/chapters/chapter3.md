# Chapter3 エンドポイントの作成

ここでは、ユーザーデータを作成、取得、更新、削除を行うエンドポイントを作成してきます。初めに FastAPI のパラメータ取得方法を解説しますが、公式サイトで既に知っている方は、スキップしてください。

## 目次

- [Chapter3 エンドポイントの作成](#chapter3-エンドポイントの作成)
  - [目次](#目次)
  - [パラメータの取得(スキップ可)](#パラメータの取得スキップ可)
    - [パスパラメータ](#パスパラメータ)
    - [クエリパラメータ](#クエリパラメータ)
    - [リクエストボディ](#リクエストボディ)
  - [作成するエンドポイント](#作成するエンドポイント)
  - [スキーマの定義](#スキーマの定義)
  - [パスオペレーション関数の定義](#パスオペレーション関数の定義)
  - [Prev: Chapter2 ディレクトリ構成](#prev-chapter2-ディレクトリ構成)

## パラメータの取得(スキップ可)

FastAPI におけるパスパラメータ、クエリパラメータ、リクエストボディの取得の仕方について説明します。[公式サイトのチュートリアル](https://fastapi.tiangolo.com/ja/tutorial/)でも分かりやすく説明されているので、公式サイトも参照してください。ここでは、簡単に説明します。

- 公式サイトのリンク
  - [パスパラメータ](https://fastapi.tiangolo.com/ja/tutorial/path-params/)
  - [クエリパラメータ](https://fastapi.tiangolo.com/ja/tutorial/query-params/)
  - [リクエストボディ](https://fastapi.tiangolo.com/ja/tutorial/body/)

### パスパラメータ

パスパラメータは、format 文字列と同様のシンタックスで取得できます。`app/api/endpoints/root.py`を以下のように編集してみましょう。

`app/endpoints/root.py`

```python
from fastapi import APIRouter

router = APIRouter()


@router.get("/")
def root():
    return {"API": "Active"}

@router.get("/items/{item_id}")
def read_item(item_id):
    return {"item_id": item_id}
```

サーバーを起動(`uvicorn app.main:app --reload --port 8000`実行)後、[http://127.0.0.1:8000/items/foo](http://127.0.0.1:8000/items/foo)にアクセスすると以下のようなレスポンスが表示されるはずです。

```json
{ "item_id": "foo" }
```

また、以下のようにすることで、パスパラメータの型を宣言することができ、その型に自動的に変換されます。

```python
@router.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}
```

サーバーを起動後、[http://127.0.0.1:8000/items/3](http://127.0.0.1:8000/items/3)にアクセスすると以下のようなレスポンスが得られます。

```json
{ "item_id": 3 }
```

型を宣言すると、バリデーションも行うので、[http://127.0.0.1:8000/items/foo](http://127.0.0.1:8000/items/foo)にアクセスすると、以下のような HTTP エラーが表示されます。

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "item_id"],
      "msg": "Input should be a valid integer, unable to parse string as an integer",
      "input": "foo",
      "url": "https://errors.pydantic.dev/2.1/v/int_parsing"
    }
  ]
}
```

### クエリパラメータ

パスパラメータではないものをパスオペレーション関数の引数とすると、自動的にクエリパラメータとして解釈されます。
`app/endpoints/root.py`の末尾に以下を追加してみましょう。

```python
@router.get("/items/")
def read_items(skip: int = 0, limit: int = 10):
    fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]
    return fake_items_db[skip : skip + limit]
```

次の URL にだと

```url
http://127.0.0.1:8000/items/?skip=1&limit=20
```

クエリパラメータは、`skip`は`1`、`limit`は`20`と解釈されます。また、今回は型も宣言されているので、バリデーションも行われます。ちなみにリクエストに対するレスポンスは、以下のようになります。

```json
[{ "item_name": "Bar" }, { "item_name": "Baz" }]
```

### リクエストボディ

クライアント (ブラウザなど) から API にデータを送信する必要があるとき、データを リクエストボディ (request body) として送ります。

FastAPI では、リクエストボディは pydantic を利用して型を定義します。この型を定義は、スキーマと呼び`app/schema`ディレクトリにファイルを作っていきます。以下のファイルを作成しましょう。

`app/schema/root.py`

```python
from pydantic import BaseModel


class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
```

また、`app/api/endpoints/root.py`で定義したスキーマをインポートし、パスオペレーション関数を追加しましょう。

`app/api/endpoints/root.py`

```python
from app.schemas.root import Item

@router.post("/items/")
def create_item(item: Item):
    return item
```

さて、このエンドポイントを試すためには、リクエストボディを記述する必要があります。その場合は Swagger UI を使いましょう。サーバーを起動後、[http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)にアクセスしましょう。先ほど作成したエンドポイント`POST` `/items/`を開き、`Try it out`ボタンを押しましょう。以下のような画面になるので、リクエストボディを自由に編集することができます。自分の好きなパラメータで試してみましょう。

![SwaggerUI Request body](..\images\ch3_swagger_ui_request_body.png)

## 作成するエンドポイント

ここからは、ユーザーデータの追加、取得、更新、削除を行うエンドポイントを作成します。 ユーザーデータは、本来 DB に保存しますが、それは後で行います。

- `POST` `/users`
  ユーザーデータの作成
- `GET` `/users`
  ユーザーデータの取得
- `PUT` `/users/{singin_id}`
  ユーザーデータの更新
- `DELETE` `/users/{sinin_id}`
  ユーザーデータの削除

それでは、まずパスオペレーション関数を定義していきましょう。今回作成するエンドポイントは、ユーザー関連のエンドポイントなので、共通のパス`/users`をつけています。この場合、ルーターをまとめる際に`prefix`で定義します。パスオペレーション関数では、`/users`を除いた部分のみ定義します。以下のファイルを作成してください。

`app/api/endpoints/users.py`

```python
from fastapi import APIRouter

router = APIRouter()


@router.post("")
def create_user():
    pass


@router.get("")
def read_user():
    pass


@router.put("/{signin_id}")
def update_user():
    pass


@router.delete("/{signin_id}")
def delete_user():
    pass
```

このルーターを`prefix = "/users"`で追加します。`app/api/api.py`を以下のように編集してください。

`app/api/api.py`

```python
from fastapi import APIRouter

from app.api.endpoints import root, users

api_router = APIRouter()
api_router.include_router(root.router, tags=["test"])
api_router.include_router(users.router, tags=["users"], prefix="/users")
```

これで、一度サーバーを起動して、[Swagger UI](http://127.0.0.1:8000/docs) を見てみましょう。

![SwaggerUI Request body](..\images\ch3_swagger_ui_users.png)

新しく定義されたエンドポイントが追加されているはずです。

## スキーマの定義

パスオペレーション関数の具体的な定義を書く前に、リクエストボディやレスポンスボディのスキーマを定義していきます。

[リクエストボディ](#リクエストボディ)でも説明しましたが、FastAPI では、Pydantic を利用して API のリクエストとレスポンスの型のバリデーションを行います。この型定義のことをスキーマと呼んでいます。

さて、リクエストボディが必要になるエンドポイントは、以下２つです。

- `POST` `/users`
  ユーザーデータの作成
- `PUT` `/users/{singin_id}`
  ユーザーデータの更新

スキーマをそれぞれ`UserCreate`、`UserUpdate`として定義します。スキーマの定義は、`app/schema`ディレクトリで行います。以下のファイルを追加してください。

`app/schema/user.py`

```python
from typing import Optional

from pydantic import BaseModel


class UserCreate(BaseModel):
    signin_id: str
    password: str
    name: Optional[str] = ""
    role: Optional[str] = "User"


class UserUpdate(BaseModel):
    signin_id: str | None = None
    password: str | None = None
    name: str | None = None
```

また、レスポンスボディとして返すスキーマも定義します。password をレスポンスとして返すわけにはいかないので、password 以外の閲覧してもいいようなデータをレスポンスとして返しましょう。`app/schema/user.py`に以下のスキーマを追加してください。

`app/schema/user.py`

```python
class UserResponse(BaseModel):
    signin_id: str
    name: str
    role: str
```

ここで、定義したスキーマをパスオペレーション関数で使っていくのですが、パスオペレーション関数などで、import した際に、このクラスがスキーマであることを認識したいです。今後、DB などの連携を行うと DB のクラスなども import するため、スキーマなのか DB のクラスなのか混乱させないためです。

そのため、他のファイルでは、`schemas.UserCreate`という参照の仕方をしたいです。これを行うには、`app/schemas/__init__.py`を以下のように編集してください。

`app/schemas/__init__.py`

```python
from .user import UserCreate, UserUpdate, UserResponse
```

## パスオペレーション関数の定義

ここから、パスオペレーション関数の定義をしていきましょう。ユーザーデータは本来、DB に格納しますが、ここでは、疑似的な DB としてリストに保持することとします。以下のように`app/api/endpoints/users.py`に疑似的な DB を書きます。

`app/api/endpoints/users.py`

```python
# 　疑似的なDB
from pydantic import BaseModel


class User(BaseModel):
    signin_id: str
    password: str
    name: str
    role: str


fake_user_db = [User(signin_id="tarou", password="tarou", name="太郎", role="User")]
```

そして、`app/api/endpoints/users.py`にて以下を追加し、スキーマを import しておきましょう。

`app/api/endpoints/users.py`

```python
from app import schemas
```

それでは、エンドポイントを順に書き換えて行きましょう。

- `POST` `/users`
  ユーザーデータの作成

  ```python
  @router.post("", response_model=schemas.UserResponse)
  def create_user(user_create: schemas.UserCreate):
      user = User(**user_create.dict())
      fake_user_db.append(user)
      return user
  ```

- `GET` `/users`
  ユーザーデータの取得

  ```python
  from typing import List

  @router.get("", response_model=List[schemas.UserResponse])
  def read_user(skip: int = 0, limit: int = 100):
      return fake_user_db[skip : skip + limit]
  ```

- `PUT` `/users/{singin_id}`
  ユーザーデータの更新

  ```python
  @router.put("/{signin_id}", response_model=schemas.UserResponse)
  def update_user(signin_id: str, update_user: schemas.UserUpdate):
      update_dict = {}
      for column, value in update_user:
          if value is not None:
              update_dict[column] = value

    　user_ids = [user.signin_id for user in fake_user_db]
      if signin_id not in user_ids:
          raise HTTPException(status_code=404, detail="User not found")

      index = user_ids.index(signin_id)
      new_user = User(**fake_user_db[index].dict())
      for key in update_dict:
          setattr(new_user, key, update_dict[key])
      fake_user_db[index] = new_user

        return new_user
  ```

- `DELETE` `/users/{sinin_id}`
  ユーザーデータの削除

  ```python
  @router.delete("/{signin_id}", response_model=None)
  def delete_user(signin_id: str):
      user_ids = [user.signin_id for user in fake_user_db]
      if signin_id not in user_ids:
          raise HTTPException(status_code=404, detail="User not found")

      index = user_ids.index(signin_id)
      del fake_user_db[index]

      return None
  ```

これで完了です。レスポンスのスキーマは、`@router.get(...)`などにキーワード引数`response_model`として渡しています。ユーザーデータの取得を例として挙げると、以下の通りです。

```python
@router.get("", response_model=List[schemas.UserResponse])
def read_user(skip: int = 0, limit: int = 100):
    return fake_user_db[skip : skip + limit]
```

`response_model`を指定してあげると、自動的にスキーマに変換してくれます。上の関数で、`fake_user_db`は、`List[User]`という型ですが、`response_model`を定義したことにより、`List[schemas.UserResponse]`に変換されてレスポンスが返されます。

[SwaggerUI](http://127.0.0.1:8000/docs)で、いろいろ試して動作を確認してみましょう。

## [Prev: Chapter2 ディレクトリ構成](../chapters/chapter2.md)
