---
title: Chapter5 セキュリティの実装
mermaid: true
---

<!-- omit in toc -->
# セキュリティの実装

ここでは、Json Web Tokenを利用したセキュリティを実装します。

<!-- omit in toc -->
## 目次

- [Json Web Token(JWT)とは](#json-web-tokenjwtとは)
  - [ヘッダー](#ヘッダー)
  - [ペイロード](#ペイロード)
  - [署名](#署名)
- [フロントエンド、APIの認証の流れ](#フロントエンドapiの認証の流れ)
- [パッケージのインストール](#パッケージのインストール)
- [JWTを作成する関数の実装](#jwtを作成する関数の実装)
- [JWTを発行するエンドポイントの実装](#jwtを発行するエンドポイントの実装)
- [パスオペレーション関数：依存関係の追加](#パスオペレーション関数依存関係の追加)
- [ユーザーロールによるエンドポイントの制限](#ユーザーロールによるエンドポイントの制限)
- [(付録)JWTをHttpOnlyなCookieに保存するパターン](#付録jwtをhttponlyなcookieに保存するパターン)
  - [JWTをHttpOnlyなCookieに保存させるエンドポイントの実装](#jwtをhttponlyなcookieに保存させるエンドポイントの実装)
  - [Cookieに保存されたJWTの取得](#cookieに保存されたjwtの取得)
  - [Cookieの削除(ログアウト処理)](#cookieの削除ログアウト処理)
  - [Swagger UIの動作を戻す](#swagger-uiの動作を戻す)
- [おまけ](#おまけ)
- [Prev: Chapter4 DBとの連携](#prev-chapter4-dbとの連携)

## Json Web Token(JWT)とは
JSON Web Token（ジェイソン・ウェブ・トークン）は、JSONデータに署名や暗号化を施す方法を定めた[オープン標準](https://datatracker.ietf.org/doc/html/rfc7519)である。(from [Wikipedia](https://ja.wikipedia.org/wiki/JSON_Web_Token))

JWTには、任意の情報を保持することができます。例えば、ユーザーIDやトークンの有効期限などの情報を持たせることができ、フロントエンドがそのトークンを持っていれば、認証済みであることの証明に使えます。

トークンは、秘密鍵によって署名されており、悪意を持ったユーザーがトークンを書き換えたとしても、署名が一致しないので、修正されたことを検知できます。

JWTは、ヘッダー、ペイロード、署名の3要素から成ります。ペイロードに任意の情報を持たせることができます。

※JWTの説明は、FastAPI公式サイトの[パスワード（およびハッシュ化）によるOAuth2、JWTトークンによるBearer](https://fastapi.tiangolo.com/ja/tutorial/security/oauth2-jwt/)にもあります。

### ヘッダー

```json
{
  "alg" : "HS256",
  "typ" : "JWT"
}
```

署名生成に使用したアルゴリズムを格納します。左記のHS256は、このトークンがHMAC-SHA256で署名されていることを示します。

### ペイロード
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

認証情報などを格納します。

### 署名

```
HMAC-SHA256(
 base64urlEncoding(header) + '.' +
 base64urlEncoding(payload),
 secret
)
```

トークン検証用の署名。署名はヘッダーとペイロードをBase64urlエンコーディングしてピリオドで結合したものから生成します。署名はヘッダーで指定された暗号化アルゴリズムにより生成されます。 


## フロントエンド、APIの認証の流れ
フロントエンドを考慮した認証の流れは以下のようになります。

- ユーザーは、フロントエンドで**ユーザー名**と**パスワード**を入力し、Enterを押します。
- フロントエンドがAPIのトークンを発行するURLにリクエストを送ります。
- APIは、**ユーザー名**と**パスワード**を確認し、正しいものであれば、**トークン**を返します。
- フロントエンドは、**トークン**を一時的に保存します。
- フロントエンドが、認証が必要なURLにリクエストを送る場合、**Authorization**ヘッダに**トークン**を付与して、リクエストを送ります。
- APIは、トークンとリクエストを受け取り、トークンが正しいものであれば、レスポンスを返します。

```mermaid
sequenceDiagram
    participant front as フロントエンド
    participant api as API
    front->>api: ユーザー名＋パスワード
    api->>api: ユーザー認証
    api->>front: JWT
    front->>api: JWT+リクエスト
    api->>api: JWT確認
    api->>front: レスポンス
```

※ここの内容は、FastAPI公式サイトのチュートリアルの[セキュリティ](https://fastapi.tiangolo.com/ja/tutorial/security/first-steps/)にも記述してあります。

## パッケージのインストール

ここでは、フォームデータを扱うため、`python-multipart`が必要になります。以下のコマンドでインストールしてください。
```bash
pip install python-multipart
```

JWTの発行には、`python-jose`を使用します。また、暗号を扱うためのパッケージが追加で必要となるので、ここでは、推奨されている`cryptography`を使用します。以下のコマンドでインストールできます。
```bash
pip install "python-jose[cryptography]"
```

## JWTを作成する関数の実装

まず、JWTに持たせるトークンのペイロードのスキーマを定義します。ここでは、ユーザーIDを持たせることとします。以下の`app/schemas/token.py`を作成してください。トークンの有効期限は、必ず持たせることとするので、ここには、記述しなくてよいです。

`app/schemas/token.py`

```python
from pydantic import BaseModel


class TokenPayload(BaseModel):
    sub: str
```

スキーマを新しく定義したら、合わせて`app/schemas/__init__.py`も編集しましょう。以下の記述を追加してください。

`app/schemas/__init__.py`
```python
from .token import TokenPayload
```

次に、トークンの有効期限をトークン発行後、何分とするかの設定を記述します。ここでは、60分に設定します。また、JWTの署名に利用する秘密鍵も設定します。`app/core/config.py`を以下のように編集してください。

`app/core/config.py`
```python
import secrets
from pydantic import BaseModel, PostgresDsn


class Settings(BaseModel):
    SQLALCHEMY_DATABASE_URI: PostgresDsn = (
        "postgresql://postgres:postgres@localhost:5000/test"
    )
    SECRET_KEY: str = secrets.token_urlsafe(32)
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60


settings = Settings()

```

**注意：** ここで、`SECRET_KEY`は、`secrets.token_urlsafe(32)`としていますが、デプロイの際は、`secrets.token_urlsafe(32)`の実行結果を設定するようにしてください。このままだと、APIを起動するごとに`SECRET_KEY`が変わることになり、APIがステートフルになってしまいます。そうなるとロードバランサを導入した際など複数のプロセスで動作させた時に、不都合が生じます。


さて、次にJWTを作成する関数を実装しましょう。この関数は、`app/core/security.py`に記述します。以下ように変更してください。

```python
from datetime import datetime, timedelta, UTC

import bcrypt
from jose import jwt

from app import schemas
from app.core.config import settings

ALGORITHM = "HS256"


def create_access_token(
    payload: schemas.TokenPayload,
    expires_delta: timedelta | None = None,
    return_expire=False,
):
    if expires_delta:
        exp = datetime.now(UTC) + expires_delta
    else:
        exp = datetime.now(UTC) + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )

    to_encode = {key: value for key, value in payload}
    to_encode.update({"exp": exp})

    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=ALGORITHM)

    if return_expire:
        return encoded_jwt, exp
    else:
        return encoded_jwt


def get_password_hash(password: str) -> str:
    pwd_bytes = password.encode("utf-8")
    salt = bcrypt.gensalt()
    hashed_password = bcrypt.hashpw(password=pwd_bytes, salt=salt)
    return hashed_password.decode("utf-8")


def verify_password(plain_password: str, hashed_password: str) -> bool:
    pwd_bytes = plain_password.encode("utf-8")
    hashed_pwd_bytes = hashed_password.encode("utf-8")
    return bcrypt.checkpw(password=pwd_bytes, hashed_password=hashed_pwd_bytes)
```


## JWTを発行するエンドポイントの実装
ここでは、`POST` `/login/token`にJWTを発行するエンドポイントを実装します。

まず、トークンの発行にあたり、ユーザー認証を行う必要があります。そこで`app/crud/user.py`の`CRUDUser`クラスにユーザー認証を行うメソッドを追加しましょう。このメソッドで`signin_id`と`password`が正しいか確認します。`app/crud/user.py`の`CRUDUser`に以下のメソッドを追加してください。

`app/crud/user.py`

```python
from app.core.security import verify_password

class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
  ...

  def authenticate(
        self, db: Session, signin_id: str, password: str
    ) -> Optional[User]:
        user = self.read_by_signin_id(db, signin_id=signin_id)
        if not user:
            return None
        if not verify_password(password, user.hashed_password):
            return None
        return user
```

次はレスポンスとして返却するトークンのスキーマを定義しましょう。`app/schemas/token.py`に以下を追加してください。

`app/schemas/token.py`
```python
class Token(BaseModel):
    access_token: str
    token_type: str
```

ここでも、新しくスキーマを定義したので、合わせて`app/schemas/__init__.py`も編集します。以下の記述を追加してください。

`app/schemas/__init__.py`
```python
from .token import TokenPayload, Token
```

最後にパスオペレーション関数の定義です。`app/api/endpoints/auth.py`を作成してください。

`app/api/endpoints/auth.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from app.api.deps import get_db
from app.core import security
from app import schemas, crud

router = APIRouter()


@router.post("/login/token", response_model=schemas.Token)
def login(
    db: Session = Depends(get_db), form_data: OAuth2PasswordRequestForm = Depends()
):
    user = crud.user.authenticate(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=400, detail="Incorrect id or password")
    payload = schemas.TokenPayload(sub=str(user.id))
    access_token = security.create_access_token(payload)
    return {"access_token": access_token, "token_type": "bearer"}
```

作成したパスオペレーション関数をapiに含めましょう。`app/api/api.py`を以下のように編集してください。

`app/api/api.py`

```python
from fastapi import APIRouter

from app.api.endpoints import root, users, auth

api_router = APIRouter()
api_router.include_router(root.router, tags=["test"])
api_router.include_router(auth.router, tags=["auth"])
api_router.include_router(users.router, tags=["users"], prefix="/users")
```

それでは、[SwaggerUI](http://localhost:8000/docs)でトークンが発行できるか確かめてみましょう。サーバーを起動`uvicorn app.main:app --reload --port 8000`し、[SwaggerUI](http://localhost:8000/docs)にアクセスしてください。

`POST` `/login/token`を試してみましょう。`Try it out`を押し、`username`、`password`を入力し、`Execute`を押してみましょう。以下のように`access_token`が返されれば、成功です。
![JWT endpoint](../images/ch5_swagger_ui_token.png)

IDやパスワードを間違えていると以下のようなレスポンスになるはずです。

![Bad Request](../images/ch5_bad_request.png)

## パスオペレーション関数：依存関係の追加

ここからは、パスオペレーション関数がトークンを受け取り、トークンの検証を行えるようにしましょう。トークンを検証し、トークンのユーザーを返す関数を作成します。この関数は、パスオペレーション関数に依存関係として加えるので、`app/api/deps.py`に記述します。以下のようにファイルを編集してください。

`app/api/deps.py`

```python
from typing import Generator

from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer
from jose import jwt
from pydantic import ValidationError
from sqlalchemy.orm import Session

from app.db.session import SessionLocal
from app import models, schemas, crud
from app.core.config import settings
from app.core import security


oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login/token")


def get_db() -> Generator:
    try:
        db = SessionLocal()
        yield db
    finally:
        db.close()


def get_current_user(
    db: Session = Depends(get_db), token: str = Depends(oauth2_scheme)
) -> models.User:
    try:
        payload = jwt.decode(
            token, settings.SECRET_KEY, algorithms=[security.ALGORITHM]
        )
        token_data = schemas.TokenPayload(**payload)
    except (jwt.JWTError, ValidationError):
        raise HTTPException(
            status_code=403,
            detail="Could not validate credentials",
        )
    user = crud.user.read(db, id=token_data.sub)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

パスオペレーション関数の引数に`OAuth2PasswordBearer`のインスタンスである`oauth2_scheme`を使って`token: str = Depends(oauth2_scheme)`とすることで、リクエストの`Authorization`ヘッダーを探しに行き、その値が`Bearer`とトークンを含んでいるかどうかチェックし、トークンを`str`として返します。

もし`Authorization`ヘッダーが見つからなかったり、値が`Bearer`トークンを持っていなかったりすると、401 ステータスコードエラー (`UNAUTHORIZED`) で直接応答します。

また、`OAuth2PasswordBearer`の引数に`tokenUrl`にJWTを発行するURLを追加しておくと、Swagger UIに認証フォームが追加され、JWTによるユーザー認証のテストが行えるので、ここでは、`oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login/token")`としています。

この`get_current_user`は、パスオペレーション関数の引数に以下のように加えることで、そのエンドポイントには、認証が必要になります。

```python
@router.get()
def test(current_user = Depends(get_current_user)):
    pass
```

それでは、`app/api/endpoints/users.py`のパスオペレーション関数に、JWTによる認証を付けましょう。パスオペレーション関数の引数に以下を加えてください。

```python
current_user: models.User = Depends(get_current_user)
```

変更後の`app/api/endpoints/users.py`は以下のようになります。

`app/api/endpoints/users.py`
```python
from typing import List

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app.api.deps import get_db, get_current_user
from app import crud, schemas, models

router = APIRouter()


@router.post("", response_model=schemas.UserResponse)
def create_user(
    user_create: schemas.UserCreate,
    db: Session = Depends(get_db),
    current_user: models.User = Depends(get_current_user),
):
    user = crud.user.read_by_signin_id(db, user_create.signin_id)
    if user:
        raise HTTPException(
            status_code=400,
            detail="The id already exists in the system.",
        )
    user = crud.user.create(db, user_create)
    return user


@router.get("", response_model=List[schemas.UserResponse])
def read_all_users(
    db: Session = Depends(get_db),
    skip: int = 0,
    limit: int = 100,
    current_user: models.User = Depends(get_current_user),
):
    users = crud.user.read_multi(db, skip=skip, limit=limit)
    return users


@router.put("/{signin_id}", response_model=schemas.UserResponse)
def update_user(
    signin_id: str,
    user_update: schemas.UserUpdate,
    db: Session = Depends(get_db),
    current_user: models.User = Depends(get_current_user),
):
    db_obj = crud.user.read_by_signin_id(db, signin_id)
    if db_obj is None:
        raise HTTPException(status_code=404, detail="User not found")
    user = crud.user.update(db, user_update, db_obj)
    return user


@router.delete("/{signin_id}", response_model=None)
def delete_user(
    signin_id: str,
    db: Session = Depends(get_db),
    current_user: models.User = Depends(get_current_user),
):
    user = crud.user.read_by_signin_id(db, signin_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    crud.user.delete(db, user.id)
```

サーバーを起動し、[Swagger UI](http://127.0.0.1:8000/docs)で見てみましょう。

![Swagger UI after JWT](../images/ch5_swagger_ui_after_jwt.png)

上部に`Authorize`ボタンが追加され、JWTの認証を付けたエンドポイントには、鍵のアイコンが付いていることが分かります。

実際に試してみましょう。鍵のアイコンがついたエンドポイントを試してみると、`Unauthorized`となり、レスポンスは以下のようになっているはずです。

![Unauthorized](../images/ch5_unauthorized.png)

それでは、鍵のアイコンが付いたエンドポイントは、JWTが必要です。`Authorize`ボタンを押してください。以下のようなフォームが表示されるので、`username`と`password`を入力して、フォーム下の`Authorize`ボタンを押してください。

![oauth2 form](../images/ch5_form_modal.png)

認証に成功すると、モーダル画面が以下のようになります。

![oauth2 form after login](../images/ch5_modal_after_login.png)

これで、鍵のアイコンが付いたエンドポイントも使えるようになりました。`GET` `users`を試してみましょう。今度は、うまく実行できるはずです。

`Execute`後の`Responses`にある`Curl`を見てみましょう。`Curl`は、HTTPリクエストを送るコマンドです。`curl`の後にリクエストの内容が含まれています。そこを見てみると次のような`Authorization`ヘッダーが追加されていることが分かります。

```bash
curl -X 'GET' \
  'http://localhost:8000/users?skip=0&limit=100' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjEsImV4cCI6MTY5Nzk1NTE4OH0.yxjWDw7QRmXbG32ntVoSH7EUEvlRbxE5AW1gkX2bl1A'
```

Swagger UIの機能で、先ほどのモーダル画面から取得したトークンをリクエストの`Authorization`ヘッダーに自動的に追加しています。

## ユーザーロールによるエンドポイントの制限

ここでは、管理者のみが扱えるエンドポイントを実装します。ユーザーの作成、削除、任意のユーザー情報の更新は、管理者のみが使えるものにしたいです。

実装方法は、様々です。すぐ実装できるのは、それぞれのパスオペレーション関数で`current_user.role == "Admin"`の判定を行い、`False`であれば、`raise HTTPException`とすれば、できそうです。

実際に、この方法でも実装することができますが、使いまわしが容易で、ロジックが分離されているように実装するため、パスオペレーション関数の依存関係として、`get_current_user`ではなく、`get_current_admin_user`を実装しましょう。`get_current_admin_user`では、トークンに記述されているIDのユーザー情報をDBから読み込み、そのユーザーがadminであるか判定を行います。

まず、ユーザーがアドミンであるかを判定するメソッドを`app/curd/user.py`の`CRUDUser`クラスに加えましょう。以下のメソッドを`CRUDUser`クラスに追加してください。

`app/crud/user.py`

```python
class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
    ...

    def is_admin(self, user: User) -> bool:
        return user.role == "Admin"
```

`app/api/deps.py`に`get_current_admin_user`を定義しましょう。`app/api/deps.py`に以下の関数を追加してください。

`app/api/deps.py`
```python
def get_current_admin_user(current_user: models.User = Depends(get_current_user)):
    if not crud.user.is_admin(current_user):
        raise HTTPException(
            status_code=400, detail="The user doesn't have enough privileges"
        )
    return current_user
```

管理者のみが使えるエンドポイントのパスオペレーション関数の依存関係を編集しましょう。編集内容は、簡単で、`Depends(get_current_user)`を`Depends(get_current_admin_user)`に変更するだけです。

最終的に`app/api/ednpoints/users.py`は、以下のようになります。

```python
from typing import List

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app.api.deps import get_db, get_current_user, get_current_admin_user
from app import crud, schemas, models

router = APIRouter()


@router.post("", response_model=schemas.UserResponse)
def create_user(
    user_create: schemas.UserCreate,
    db: Session = Depends(get_db),
    current_user: models.User = Depends(get_current_admin_user),
):
    user = crud.user.read_by_signin_id(db, user_create.signin_id)
    if user:
        raise HTTPException(
            status_code=400,
            detail="The id already exists in the system.",
        )
    user = crud.user.create(db, user_create)
    return user


@router.get("", response_model=List[schemas.UserResponse])
def read_all_users(
    db: Session = Depends(get_db),
    skip: int = 0,
    limit: int = 100,
    current_user: models.User = Depends(get_current_user),
):
    users = crud.user.read_multi(db, skip=skip, limit=limit)
    return users


@router.put("/{signin_id}", response_model=schemas.UserResponse)
def update_user(
    signin_id: str,
    user_update: schemas.UserUpdate,
    db: Session = Depends(get_db),
    current_user: models.User = Depends(get_current_admin_user),
):
    db_obj = crud.user.read_by_signin_id(db, signin_id)
    if db_obj is None:
        raise HTTPException(status_code=404, detail="User not found")
    user = crud.user.update(db, user_update, db_obj)
    return user


@router.delete("/{signin_id}", response_model=None)
def delete_user(
    signin_id: str,
    db: Session = Depends(get_db),
    current_user: models.User = Depends(get_current_admin_user),
):
    user = crud.user.read_by_signin_id(db, signin_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    crud.user.delete(db, user.id)
```

## (付録)JWTをHttpOnlyなCookieに保存するパターン
JWTをフロントエンドのどこに保持するか？は、よく話題に上がりますが、その中でもHttpOnlyなCookieに保存するというパターンが多いように感じます。

HttpOnlyなCookieは、Javascriptから操作することはできないので、バックエンド側で対応が必要になってきます。

そこで、ここでは、FastAPIでJWTをHttpOnlyなCookieに保存させるエンドポイントの実装と、Cookieに保存されたJWTを取得する方法について説明します。

### JWTをHttpOnlyなCookieに保存させるエンドポイントの実装
フロントエンドにCookieを保存させるためには、レスポンスヘッダーの[Set-Cookie](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Set-Cookie)を用います。

FastAPIでレスポンスヘッダーを編集するには、パスオペレーション関数の引数に`response: Response`を設定し、この`response`に対して操作を行うことで、編集することができます。

```python
from fastapi import APIRouter, Response

router = APIRouter()

@router.get("/headers")
def get_headers(response: Response):
    # ヘッダーの設定
    response.headers["X-Cat-Dog"] = "alone in the world"
    # Set-Cookieヘッダーの設定
    response.set_cookie(key="access_token", value="sample token")
    return None
```

それでは、JWTをHttpOnly Cookieに保存させるエンドポイントを実装しましょう。`app/api/endpoints/auth.py`に記述していきます。

また、Cookieには有効期限も設定できるので、JWTと同じ有効期限を設定します。

`app/api/endpoints/auth.py`
```python
# 追加
from fastapi import Response

@router.post("/login", response_model=None)
def login_cookie(
    response: Response,
    db: Session = Depends(get_db),
    form_data: OAuth2PasswordRequestForm = Depends(),
):
    user = crud.user.authenticate(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=400, detail="Incorrect id or password")
    payload = schemas.TokenPayload(sub=str(user.id))
    access_token, expire = security.create_access_token(payload, return_expire=True)
    response.set_cookie(
        key="access_token",
        value=f"Bearer {access_token}",
        httponly=True,
        expires=expire,
    )
    return None
```

サーバーを起動し、[Swagger UI](http://127.0.0.1:8000/docs)で動作を確認してみましょう。

`auth`タグの`/login`を実際に試してみてください。

![loginのリクエストボディ](../images/ch5_swagger_ui_jwt_cookie_request.png)

レスポンスは次のようになります。

![loginのレスポンス](../images/ch5_swagger_ui_jwt_cookie_response.png)

ここで、Chromeの検証ツールを使ってCookieの中身を見てみましょう。
検証ツールのApplicationタブ内でCookieの確認ができます。

![クッキーの中身](../images/ch5_show_cookie.png)

### Cookieに保存されたJWTの取得

次にCookieに保存されたJWTの取得方法です。HttpOnly Cookieに保存されたJWTは、リクエストヘッダーのCookieに設定されます。

FastAPIでリクエストヘッダーのCookieを読むためには、パスオペレーション関数の引数に`request: Request`を設定し、`request.cookies`から取得できます。また、`request.cookies`は`dict[str, str]`なので、`request.cookies.get("access_token")`とすることで、valueを取得できます。

```python
@router.get("/cookie")
def cookie(request: Request):
    # cookiesはdict[str, str]なので、以下のように取得できる
    access_token = request.cookies.get("access_token")
    # access_token = "Bearer eyJhbGciOiJIUzI1NiIsInR5cC..."
    return None
```

また`access_token`は、`f"{scheme} {param}"`の形式で保存されており、JWTのデコードを行うのは、`{param}`の部分です。そのため、`scheme`と`param`に分解する必要があります。これは、FastAPIに実装があるので、それを使いましょう。

```python
from fastapi.security.utils import get_authorization_scheme_param

@router.get("/cookie")
def cookie(request: Request):
    access_token = request.cookies.get("access_token")
    scheme, param = get_authorization_scheme_param(access_token)
    # scheme -> "Bearer", param -> "eyJhbGciOiJIUzI1NiIsInR5cC..."
    return None
```

認証が必要なエンドポイントで毎回この処理をやるわけにもいかないので、`Depeneds`の中に入れれる関数を作っていきましょう。また、`access_token`が取得できない時や、`scheme`が`Bearer`でない場合に、`HTTPException`となるようにしましょう。

`app/api/deps.py`に以下の関数を追加してください。

`app/api/deps.py`
```python
# 追加
from fastapi import Request
from fastapi.security.utils import get_authorization_scheme_param

def get_access_token_from_cookie(request: Request):
    access_token = request.cookies.get("access_token")
    scheme, param = get_authorization_scheme_param(access_token)
    if not access_token or scheme.lower() != "bearer":
        raise HTTPException(
            status_code=401,
            detail="Not authenticated",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return param
```

これまで、`oauth2_scheme`を使っていた関数`get_current_user`を書き換えてください。

`app/api/deps.py`
```python
# 引数を変更
def get_current_user(
    db: Session = Depends(get_db), token: str = Depends(get_access_token_from_cookie)
) -> models.User:
    ...
```

これで、Cookieに保存されたJWTを取得し、このJWTを使ってログインユーザーを取得するようになりました。

サーバーを起動し、[Swagger UI](http://127.0.0.1:8000/docs)で動作を確認してみましょう。

最初にログインを実行し、その後で`users`タグのエンドポイントを試してみてください。

### Cookieの削除(ログアウト処理)
次にログアウト処理として、Cookieから`access_token`を削除させるエンドポイントを作成しましょう。

このエンドポイントの実装は簡単です。Cookieに保存させるエンドポイントは、`response.set_cookie`で行いましたが、削除は`response.delete_cookie`で行います。

`app/api/endpoints/auth.py`に以下のパスオペレーション関数を追加してください。

```python
# 追加
from app.api.deps import get_current_user
from app import models

@router.post("/logout", response_model=None)
def logout(response: Response, current_user: models.User = Depends(get_current_user)):
    response.delete_cookie("access_token")
    return None
```

このエンドポイントも試してみて、Chromeの検証ツールでCookieが削除されているか確認してみましょう。

### Swagger UIの動作を戻す
ここまで、JWTをHttpOnlyなCookieに保存させるパターンを実装してきました。実装後、[Swagger UI](http://127.0.0.1:8000/docs)を見て気づいた方もいるかもしれませんが、これまでSwagger UIにあった`Authorization`のボタンや鍵のアイコンが無くなっています。

これは、Swagger UIがJWTをHttpOnlyなCookieに保存させるパターンに対応していないためです。HttpOnly CookieはJavascriptからアクセスできないため、JWTの状態をUIに反映できず、対応ができません。

ここでは、Swagger UIの`Authorization`のボタンや鍵のアイコンを残しつつ、JWTをHttpOnlyなCookieに保存させるエンドポイントも残す実装を行なっていきます。

**注意：** この実装では、JWTがクッキーにある場合でも、Authorizationヘッダーにある場合でも動作するようになります。

これは、FastAPI公式のissue:[Cookie based JWT tokens #480](https://github.com/tiangolo/fastapi/issues/480)にあったコードを参考にしたものです。

WTをHttpOnlyなCookieに保存させるパターンの実装前、JWTは、`OAuth2PasswordBearer`を使って取得していました。この`OAuth2PasswordBearer`に似たようなクラスを定義し、元の動作にプラスして、cookieのJWTも取得するようにします。

`app/core/security.py`に以下のクラスを追加してください。

```python
# 追加
from typing import Optional
from fastapi import Request, HTTPException
from fastapi.security import OAuth2
from fastapi.security.utils import get_authorization_scheme_param
from fastapi.openapi.models import OAuthFlows as OAuthFlowsModel


class OAuth2PasswordBearerWithCookie(OAuth2):
    def __init__(
        self,
        tokenUrl: str,
        scheme_name: str = None,
        scopes: dict = None,
        auto_error: bool = True,
    ):
        if not scopes:
            scopes = {}
        flows = OAuthFlowsModel(password={"tokenUrl": tokenUrl, "scopes": scopes})
        super().__init__(flows=flows, scheme_name=scheme_name, auto_error=auto_error)

    async def __call__(self, request: Request) -> Optional[str]:
        authorization: str = request.headers.get("Authorization")
        if not authorization:
            authorization: str = request.cookies.get("access_token")

        scheme, param = get_authorization_scheme_param(authorization)
        if not authorization or scheme.lower() != "bearer":
            if self.auto_error:
                raise HTTPException(
                    status_code=401,
                    detail="Not authenticated",
                    headers={"WWW-Authenticate": "Bearer"},
                )
            else:
                return None

        return param
```

`app/api/deps.py`の`oauth2_scheme`を以下のように変更しましょう。

```python
from app.core.security import OAuth2PasswordBearerWithCookie

oauth2_scheme = OAuth2PasswordBearerWithCookie(tokenUrl="login/token")
```

次に`app/api/deps.py`の`get_current_user`でJWTを取得する際に、`oauth2_scheme`を使うように変更しましょう。

```python
def get_current_user(
    db: Session = Depends(get_db), token: str = Depends(oauth2_scheme)
) -> models.User:
    ...
```

これで完了です。サーバーを起動し、[Swagger UI](http://127.0.0.1:8000/docs)で動作を確認してみましょう。

最初にCookieを削除するようにしてください。これでSwagger UIの認証も使えますし、CookieにJWTを保存するパターンでも使えます。

## おまけ
一般ユーザーが自分自身のデータを取得するエンドポイント`GET` `/users/myself`や、自分自身のデータを更新できるエンドポイント`PUT` `/users/myself`を作成してみましょう。

答えは以下。

---

`app/api/endpoints/users.py`に以下のパスオペレーション関数を追加

```python
@router.get("/myself", response_model=schemas.UserResponse)
def read_myself(
    current_user: models.User = Depends(get_current_user),
):
    return current_user


@router.put("/myself", response_model=schemas.UserResponse)
def update_myself(
    user_update: schemas.UserUpdate,
    db: Session = Depends(get_db),
    current_user: models.User = Depends(get_current_user),
):
    user = crud.user.update(db, user_update, current_user)
    return user
```

## [Prev: Chapter4 DBとの連携](../chapters/chapter4.md)