---
title: Chapter4 DBとの連携
---

<!-- omit in toc -->
# DBとの連携

ここでは、ユーザーデータを実際に DB に置いていきます。SQLAlchemy という ORM ライブラリを使います。DB には PostgreSQL を使用します。

<!-- omit in toc -->
## 目次

- [ライブラリのインストール](#ライブラリのインストール)
- [設定ファイルの追加](#設定ファイルの追加)
- [DB 接続クラス](#db-接続クラス)
- [DB モデルの定義](#db-モデルの定義)
- [DB マイグレーション(sqlalchemy)](#db-マイグレーションsqlalchemy)
  - [注意点](#注意点)
- [CRUDsの実装](#crudsの実装)
- [テスト用ユーザーデータの登録（スキップ可）](#テスト用ユーザーデータの登録スキップ可)
- [usersエンドポイントの実装](#usersエンドポイントの実装)
  - [POST /users](#post-users)
  - [GET /users](#get-users)
  - [PUT /users/{signin\_id}](#put-userssignin_id)
  - [DELETE /users/{signin\_id}](#delete-userssignin_id)
  - [テスト](#テスト)
- [Next: Chapter5 セキュリティの実装](#next-chapter5-セキュリティの実装)
- [Prev: Chapter3 エンドポイントの作成](#prev-chapter3-エンドポイントの作成)

## ライブラリのインストール

ORM ライブラリである `sqlalchemy` をインストールします。

```bash
pip install sqlalchemy
```

`sqlalchemy`とは DB に DB に接続するライブラリが必要になります。DB によって必要なライブラリは異なるので、PostgreSQL 以外の DB を利用する場合は、各自調べてください。

PostgreSQL では、`psycopg2`が必要になります。ただし、`psycopg2`は C 言語ラッパーであるため、別途 C コンパイラが必要になります。そのためスタンドアローンなパッケージとして提供されている`psycopg2-binary`を利用します。

```bash
pip install psycopg2-binary
```

また、DBにユーザー情報を登録するので、その際にパスワードのハッシュ化を行います。パスワードのハッシュ化には、`bcrypt`を使用します。以下のコマンドでインストールしてください。

```bash
pip install bcrypt
```

※DB については、各自用意してください。ここでは、PostgreSQL を利用し、`test`という名前の DB を作成しています。

## 設定ファイルの追加

SQLAlchemy の engine 作成時に必要な URI は、設定ファイルに保存します。設定ファイルは、`app/core/config.py`となります。以下のファイルを作成しましょう。

`app/core/config.py`

```python
from pydantic import BaseModel, PostgresDsn


class Settings(BaseModel):
    SQLALCHEMY_DATABASE_URI: PostgresDsn = "postgresql://postgres:postgres@localhost:5432/test"


settings = Settings()
```

今後、設定値を追加する場合は、このファイルの`Setting`クラスに追加していきます。設定値を利用する場合は、以下のようにしてください。

```python
from app.core.config import settings

# 利用
settings.SQLALCHEMY_DATABASE_URI
```

URI は、[公式のドキュメント](https://docs.sqlalchemy.org/en/20/core/engines.html)などを参考にしてください。以下のような形式になります。

```
dialect+driver://username:password@host:port/database
```

## DB 接続クラス

DB 接続のためにクラスは、`app/db/session.py`に記述します。以下のファイルを作成しましょう。

`app/db/session.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.core.config import settings


engine = create_engine(settings.SQLALCHEMY_DATABASE_URI, pool_pre_ping=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

次にパスオペレーション関数が使う DB のセッションを取得する関数を定義していきます。以下のような`app/api/deps.py`を作成しましょう。

`app/api/deps.py`

```python
from typing import Generator

from app.db.session import SessionLocal


def get_db() -> Generator:
    try:
        db = SessionLocal()
        yield db
    finally:
        db.close()
```

実際にパスオペレーション関数で利用する際は、以下のように関数の引数として追加します。

```python
from fastapi import Depends
from sqlalchemy.orm import Session

from app.api.deps import get_db

@router.get("")
def test(db: Session = Depends(get_db)):
    pass
```

## DB モデルの定義

ここから、SQLAlchemy を使って DB モデルを定義していきます。通常、DB モデルは、`sqlalchemy.orm`の`declarative_base`を使い、`Base = declarative_base()`とした`Base`クラスを継承します。ここでは、DB モデルのテーブル名をクラス名から自動的に作るように以下のように`Base`クラスを`app/db/base_class.py`に定義します。

`app/db/base_class.py`

```python
from typing import Any

from sqlalchemy.ext.declarative import as_declarative, declared_attr


@as_declarative()
class Base:
    id: Any
    __name__: str

    @declared_attr
    def __tablename__(cls) -> str:
        return cls.__name__.lower()
```

次は、この Base クラスを継承してユーザーの DB モデルを定義しましょう。DB モデルは、`app/models`ディレクトリにファイルを追加していきます。`app/models/user.py`を追加しましょう。

`app/models/user.py`

```python
from sqlalchemy import Column, Integer, String

from app.db.base_class import Base


class User(Base):
    id = Column(Integer, primary_key=True, index=True)
    signin_id = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    name = Column(String, index=True)
    role = Column(String, default="User")
```

Chapter3 では、疑似的な DB には、パスワードをそのまま保存していましたが、本来はハッシュ化して保存します。DB には password はそのまま保存せず、ハッシュ化したパスワードを保存するので、カラム名は`hashed_password`としています。

また、パスオペレーション関数などで、このクラスを扱う場合、スキーマとモデルの区別を瞬時にできるように`models.User`という記述をおこないたいです。そのために`app/models/__init__.py`を以下のように編集しましょう。

`app/models/__init__.py`

```python
from .user import User
```

## DB マイグレーション(sqlalchemy)

作成した ORM モデルを利用して、DB マイグレーション用のスクリプト`app/migrate.py`を作成します。

`app/migrate.py`

```python
from sqlalchemy import create_engine

from app.db.base_class import Base
from app.core.config import settings
import app.models


def reset_database(engine):
    Base.metadata.drop_all(bind=engine)
    Base.metadata.create_all(bind=engine)


if __name__ == "__main__":
    engine = create_engine(settings.SQLALCHEMY_DATABASE_URI, echo=True)
    reset_database(engine=engine)
```

それでは、`app/migrate.py`を実行してみましょう。このときに DB も起動してください。

```bash
python -m app.migrate
```

各自の DB に合った方法で、以下のような SQL 文を実行して、user テーブルが作られているか確認しましょう。PostgreSQL では、`user`は予約語なので、`"user"`となることに注意してください。

```sql
SELECT * FROM "user";
```

### 注意点
このマイグレーションは、同じ名前のテーブルがすでに定義されている場合、そのテーブルを削除してテーブルを作り直しています。

開発途中で、テーブルの定義が変わった時に、すでにテーブルに保存されているデータを引き継いで欲しいことがあるかと思います。その際、このマイグレーションでは対応ができません。そこで、`alembic`というマイグレーション用のライブラリを使うことで、データの引き継ぎをしつつ、マイグレーションを行うことができるようになります。
これについては、後の章で紹介します。

## CRUDsの実装
ここでは、DBのCRUD(IO処理)を記述していきます。CRUDとは、永続的なデータを取り扱うソフトウェアに要求される4つの基本機能である、データの作成（Create）、読み出し（Read）、更新（Update）、削除（Delete）の頭文字を繋げた言葉です。`app/crud`ディレクトリにファイルを作成します。
ここでは、CRUD操作のベースとなるクラスを定義します。以下のファイルを作成してください。
※難しい場合は、個別に(UserのCreate、Read、Update、Deleteの４つ)CRUDの関数を実装してください。

`app/crud/base.py`

```python
from typing import Any, Dict, Generic, List, Optional, Type, TypeVar

from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel
from sqlalchemy.orm import Session

from app.db.base_class import Base

ModelType = TypeVar("ModelType", bound=Base)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)


class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
    def __init__(self, model: Type[ModelType]):
        """
        CRUD object with default methods to Create, Read, Update, Delete (CRUD).

        **Parameters**

        * `model`: A SQLAlchemy model class
        * `schema`: A Pydantic model (schema) class
        """
        self.model = model

    def create(self, db: Session, obj_in: CreateSchemaType) -> ModelType:
        obj_in_data = jsonable_encoder(obj_in)
        db_obj = self.model(**obj_in_data)  # type: ignore
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def read(self, db: Session, id: Any) -> Optional[ModelType]:
        return db.query(self.model).filter(self.model.id == id).first()

    def read_multi(
        self, db: Session, skip: int = 0, limit: int = 100
    ) -> List[ModelType]:
        return db.query(self.model).offset(skip).limit(limit).all()

    def update(
        self,
        db: Session,
        db_obj: ModelType,
        obj_in: UpdateSchemaType | Dict[str, Any],
    ) -> ModelType:
        obj_data = jsonable_encoder(db_obj)
        if isinstance(obj_in, dict):
            update_data = obj_in
        else:
            update_data = obj_in.model_dump(exclude_unset=True)
        for field in obj_data:
            if field in update_data:
                setattr(db_obj, field, update_data[field])
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def delete(self, db: Session, id: int) -> ModelType:
        obj = db.query(self.model).get(id)
        db.delete(obj)
        db.commit()
        return obj
```

それぞれのモデルに対して、CRUD操作を行うクラスを定義するのですが、そのクラスは、この`CRUDBase`クラスを継承し、必要に応じて、CRUD操作をオーバーライドしたり、新しくメソッドを定義したりします。

それでは、`User`モデルに対するCRUD操作を行うクラスを定義しましょう。DBでは、Userのパスワードは、そのまま保存せず、ハッシュ化して保存します。そのため、CreateやUpdateの際、リクエストの`password`をハッシュ化し、`hashed_password`に保存する必要があります。また、Readの際もDBのidではなく、`signin_id`で読み込めるようにします。

`User`モデルのCRUD操作を行うクラスは、以下のようになります。ファイルを作成してください。

`app/crud/user.py`

```python
from typing import Dict, Any, Optional

from sqlalchemy.orm import Session

from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate

from app.crud.base import CRUDBase


class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
    def read_by_signin_id(self, db: Session, signin_id: str) -> Optional[User]:
        return db.query(User).filter(User.signin_id == signin_id).first()

    def create(self, db: Session, user_create: UserCreate) -> User:
        # userはpasswordをhashed passwordにするので、CRUDBaseのcreateはオーバーライド
        user_create_dict = self.__hash_password(user_create)
        db_obj = self.model(**user_create_dict)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def update(self, db: Session, user_update: UserUpdate, db_obj: User) -> User:
        # userはpasswordをhashed passwordにするので、CRUDBaseのupdateはオーバーライド
        user_update_dict = self.__hash_password(user_update)

        db_obj = super().update(db, db_obj, user_update_dict)
        return db_obj

    def __hash_password(self, user_schema: UserCreate | UserUpdate) -> Dict[str, Any]:
        user_dict: Dict[str, Any] = {}
        for field, value in user_schema:
            if value is None:
                continue
            if field == "password":
                user_dict["hashed_password"] = value + "password"
            else:
                user_dict[field] = value
        return user_dict


user = CRUDUser(User)

```

このCRUD操作をパスオペレーション関数で使う際に、CRUD操作であることを即座にわかるようにしたいので、`crud.user.read(...)`のようにしたいです。そのため、`app/crud/__init__.py`を以下のように編集します。

`app/crud/__init__.py`
```python
from .user import user
```

ここでは、パスワードのハッシュ化が仮の物になっているので、次はパスワードのハッシュ化を実装します。パスワードのハッシュ化やこの後実装するJson Web Tokenの発行などは、`app/core/security.py`に記述します。パスワードのハッシュ化と同時にパスワードの検証も実装します。以下のファイルを作成してください。

`app/core/security.py`
```python
import bcrypt


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

ここで作成した`get_password_hash`を`CRUDUser`で使います。`app/crud/user.py`に定義した`CRUDUser`クラスの`__hash_password`メソッドを以下のように編集してください。

`app/crud/user.py`
```python
from app.core.security import get_password_hash


class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
    ...

    def __hash_password(self, user_schema: UserCreate | UserUpdate) -> Dict[str, Any]:
        user_dict: Dict[str, Any] = {}
        for field, value in user_schema:
            if value is None:
                continue
            if field == "password":
                user_dict["hashed_password"] = get_password_hash(value)
            else:
                user_dict[field] = value
        return user_dict
```

## テスト用ユーザーデータの登録（スキップ可）

パスオペレーション関数の実装の前に、テスト用にユーザーを登録するスクリプトを作成しましょう。先ほど実装した`CRUDUser`を使います。以下のファイルを作成してください。

`app/create_initial_data.py`
```python
from app.db.session import SessionLocal
from app.schemas.user import UserCreate
from app import crud


initial_data = [
    UserCreate(signin_id="tarou", password="tarou", name="太郎"),
    UserCreate(signin_id="john", password="john"),
    UserCreate(signin_id="admin", password="password", role="Admin"),
]


def init() -> None:
    db = SessionLocal()
    for data in initial_data:
        crud.user.create(db, data)


if __name__ == "__main__":
    init()
```

それでは、実行してみましょう。FastAPIのサーバーは立てる必要はありません。DBのみで良いです。

```bash
python -m app.create_initial_data
```

実行後、反映されているか以下のSQL文を実行して確認してみましょう。

```sql
SELECT * FROM "user";
```

## usersエンドポイントの実装

ここからは、Chapter3で作成したエンドポイントをDBと連携したものに書き換えていきましょう。

まず、`app/schemas/user.py`に定義したレスポンススキーマ`UserResponse`を以下のように変更します。

`app/schemas/user.py`
```python
from pydantic import ConfigDict

class UserResponse(BaseModel):
    signin_id: str
    name: str
    role: Literal["Admin", "User"]

    model_config = ConfigDict(from_attributes=True)
```

こうすることで、DBのモデルである`User`をスキーマの`UserResponse`に変換できるようになります。それでは、それぞれのエンドポイントを書き換えていきましょう。ここからは、`app/api/endpoints/users.py`に定義した擬似的なDB`fake_user_db`は使わないので、削除してください。また、同じファイルで定義している`User`も使わないので、これも削除してください。

`app/api/endpoints/users.py`

```python
# 以下は削除。
from pydantic import BaseModel


class User(BaseModel):
    signin_id: str
    password: str
    name: str
    role: str


fake_user_db = [User(signin_id="tarou", password="tarou", name="太郎", role="User")]

```

ここからは、DBとの接続も行うので、`models`、`crud`が必要になります。また、それぞれのパスオペレーション関数の引数に`db: Session = Depends(get_db)`が必要となります。importを以下のように編集してください。

```python
from typing import List

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app.api.deps import get_db
from app import crud, schemas, models
```

`app/api/endpoints/users.py`に定義したパスオペレーション関数をそれぞれ編集してください。

### POST /users

```python
@router.post("", response_model=schemas.UserResponse)
def create_user(
    user_create: schemas.UserCreate,
    db: Session = Depends(get_db),
):
    user = crud.user.read_by_signin_id(db, user_create.signin_id)
    if user:
        raise HTTPException(
            status_code=400,
            detail="The id already exists in the system.",
        )
    user = crud.user.create(db, user_create)
    return user
```

### GET /users

```python
@router.get("", response_model=List[schemas.UserResponse])
def read_all_users(
    db: Session = Depends(get_db),
    skip: int = 0,
    limit: int = 100,
):
    users = crud.user.read_multi(db, skip=skip, limit=limit)
    return users
```


### PUT /users/{signin_id}

```python
@router.put("/{signin_id}", response_model=schemas.UserResponse)
def update_user(
    signin_id: str,
    user_update: schemas.UserUpdate,
    db: Session = Depends(get_db),
):
    db_obj = crud.user.read_by_signin_id(db, signin_id)
    if db_obj is None:
        raise HTTPException(status_code=404, detail="User not found")
    user = crud.user.update(db, user_update, db_obj)
    return user
```

### DELETE /users/{signin_id}

```python
@router.delete("/{signin_id}", response_model=None)
def delete_user(
    signin_id: str,
    db: Session = Depends(get_db),
):
    user = crud.user.read_by_signin_id(db, signin_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    crud.user.delete(db, user.id)
```

### テスト
全てのパスオペレーション関数を書き換えたら、サーバーを起動し、[Swagger UI](http://127.0.0.1:8000/docs)で色々試してみましょう。

## [Next: Chapter5 セキュリティの実装](../chapters/chapter5.md)

## [Prev: Chapter3 エンドポイントの作成](../chapters/chapter3.md)