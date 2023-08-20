# Chapter4 DB との連携

ここでは、ユーザーデータを実際に DB に置いていきます。SQLAlchemy という ORM ライブラリを使います。DB には PostgreSQL を使用します。

## 目次

- [Chapter4 DB との連携](#chapter4-db-との連携)
  - [目次](#目次)
  - [ライブラリのインストール](#ライブラリのインストール)
  - [設定ファイルの追加](#設定ファイルの追加)
  - [DB 接続クラス](#db-接続クラス)
  - [DB モデルの定義](#db-モデルの定義)
  - [DB マイグレーション](#db-マイグレーション)

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

ここから、SQLAlchemy を使って DB モデルを定義していきます。通常、DB モデルは、`sqlalchemy.orm`の`declarative_base`を使い、`Base = declarative_base()`とした`Base`クラスを継承します。ここでは、DB モデルのテーブル名をクラス名から自動的に作るように以下のように`Base`クラスを`app/db/base.py`に定義します。

`app/db/base.py`

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

from app.db.base import Base


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

## DB マイグレーション

作成した ORM モデルを利用して、DB マイグレーション用のスクリプト`app/migrate.py`を作成します。

`app/migrate.py`

```python
from sqlalchemy import create_engine

from app.db.base import Base
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
