---
title: Chapter6 Alembicを使ったマイグレーション
---

<!-- omit in toc -->
# Alembicを使ったマイグレーション

ここでは、[Alembic](https://alembic.sqlalchemy.org/en/latest/)を利用したより実践的なマイグレーションを実装します。

<!-- omit in toc -->
## 目次
- [Alembicとは](#alembicとは)
- [Alembicの使い方](#alembicの使い方)
  - [初回導入](#初回導入)
    - [セットアップ](#セットアップ)
    - [`app/db/base.py`作成](#appdbbasepy作成)
    - [`env.py`修正](#envpy修正)
    - [マイグレーション](#マイグレーション)
  - [変更時](#変更時)
    - [通常時（自動で検出できる変更）](#通常時自動で検出できる変更)
    - [自動で検出できない変更のとき](#自動で検出できない変更のとき)
    - [具体的に修正が必要なケース](#具体的に修正が必要なケース)
      - [列名を変更したとき](#列名を変更したとき)
      - [nullable=Falseの列を追加したとき](#nullablefalseの列を追加したとき)
  - [ダウングレードしたいとき](#ダウングレードしたいとき)
  - [その他コマンド](#その他コマンド)
- [AlembicをFastAPIのプロジェクトで使ってみる](#alembicをfastapiのプロジェクトで使ってみる)
  - [セットアップ](#セットアップ-1)
  - [マイグレーション](#マイグレーション-1)
    - [準備](#準備)
    - [スクリプト生成](#スクリプト生成)
    - [マイグレーション実行](#マイグレーション実行)
    - [テーブル定義変更](#テーブル定義変更)
- [まとめ](#まとめ)
- [Prev: Chapter5 セキュリティの実装](#prev-chapter5-セキュリティの実装)


## Alembicとは
システム開発の際、開発途中でDBの定義を変える必要が出てくるシチュエーションは、様々あります。（設計時点で考慮できていない項目があった、どうしても必要な追加機能が出てきたなど…）この時、SQLAlchemy単体のマイグレーションでは、テーブルを一旦全て削除して作り直すことになります。しかし、DBにはすでにテスト用のデータが入っており、できればデータを引き継ぎたいということは、誰しもが考えることかと思います。そんな時に活躍するのが、Alembicです。

AlembicとはSQLAlchemyと一緒に使うDBマイグレーションツールです。主に以下のような機能を提供してくれます。

- バージョン管理
- 定義変更の自動検出
- データ引き継ぎ

これらの機能のおかげで、データを引き継ぎつつ、DBのテーブル定義を変更することができます。また、定義変更後、定義を元に戻したくなった時も、Alembicを使えば簡単に行うことができます。SQLAlchemyを使う際は、ぜひ一緒に使って欲しいツールです。

## Alembicの使い方

ここから実際にAlembicの使い方を見ていきます。

### 初回導入
#### セットアップ
まず、以下コマンドでインストールしましょう。
```shell
pip install alembic
```

以下のコマンドでAlembicの初期化処理を行います。コマンド内の`"alembic"`は、好きな名前を入れることができます。
```shell
alembic init "alembic"
```
コマンド実行後、以下のようなディレクトリとファイルが生成されます。
```
.
├── alembic.ini
└── alembic
	├── README
	├── env.py
	├── script.py.mako
	└── versions
```

#### `app/db/base.py`作成
alembicがマイグレーションを行うSQLAlchemyをでマイグレーションを行うモデルを全て認識させるために、以下のファイルを作成します。

`app/db/base.py`
```python
from app.db.base_class import Base
from app.models import *
```

#### `env.py`修正
次に生成された`alembic/env.py`を修正します。変更箇所は、主に`Base.metadata`の指定と、DBのURI設定です。

編集例を以下に示します。変更がわかりやすいようにデフォルトで記述してあるコメントは削除しています。

`alembic/env.py`
```python
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Baseクラスを読み込む
from app.db.base import Base

# DBのURI指定のため、設定を読み込む
from app.core.config import settings

# Base.metadataを指定
target_metadata = Base.metadata


def run_migrations_offline() -> None:
    context.configure(
        url=settings.SQLALCHEMY_DATABASE_URI,  # DBのURIを設定
        target_metadata=target_metadata,
        literal_binds=True,
        compare_type=True,  # 列タイプの変更を検知したい場合、True
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    # sqlalchemy.urlを変更するため、configをはじめに編集
    configuration = config.get_section(config.config_ini_section, {})
    configuration["sqlalchemy.url"] = settings.SQLALCHEMY_DATABASE_URI
    connectable = engine_from_config(
        configuration,  # 編集したconfigを渡す
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,  # 列タイプの変更を検知したい場合は、True
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

#### マイグレーション
以下のコマンドでマイグレーションファイルを生成します。
`"message"`には、好きな文字列を入れることができます。変更点を端的に説明した文章などをいれましょう。

このコマンドを実行すると[revision id]_[message(_区切り)].pyという名称のファイルが生成されます。
```shell
alembic revision --autogenerate -m "message"
```

次のコマンドで、マイグレーションを実行します。
（コマンドの実行内容は、最新の変更を適用するものです。）
```shell
alembic upgrade head
```

### 変更時
alembicは、ある程度自動で変更を検出してくれます。
変更を検出できる場合とできない場合でマイグレーションのやり方を説明します。

alembicがどのような変更を検出してくれるのか詳しい説明は、次のリンクにあります。

(Auto Generating Migrations - Alembic's documentation!)[https://alembic.sqlalchemy.org/en/latest/autogenerate.html#what-does-autogenerate-detect-and-what-does-it-not-detect]

#### 通常時（自動で検出できる変更）
以下のコマンドで最新の状態になります。
```shell
alembic revision --autogenerate -m "message"
alembic upgrade head
```

alembicが自動で検出できる変更は以下のものです。
- テーブルの追加・削除
- 列の追加・削除
- 列のnullableの変更
- インデックスと明示的にしていされた一意制約の基本的な変更
- 外部キー制約の基本的な変更

また、オプションで以下の変更も検出できます。
- 列タイプの変更
- サーバーのデフォルトの変更

#### 自動で検出できない変更のとき

alembicが自動で検出できない変更は、以下のような場合です。
- テーブル名の変更（2つのテーブルの削除と追加になってしまう）
- 列名の変更（2つの列の削除と追加になってしまう）
- 匿名の制約(nameがつけられていないUniqueConstraint)
- ENUMがサポートされていないバックエンドを使用しているときに、Enumで定義された列がある場合

現在（2024/01/24）検出されないもの（今後のアップデートによっては検出可能になる）
- PRIMARY KEY, EXCLUDE, CHECKなど一部の独立した制約の追加と削除
- シーケンスの追加・削除

#### 具体的に修正が必要なケース
Alembicが自動で検知してくれない変更に関しては、自分でスクリプトを編集する必要があります。

ここでは、自動で検知してくれない具体的な一部のパターンについて、どのような修正をするべきかを紹介します。

##### 列名を変更したとき
これは、[自動で検出できない変更のとき](#自動で検出できない変更のとき)でも紹介したように、自動生成しても列名の削除と追加が行われるスクリプトが生成されてしまいます。
この場合は、自分で`upgrade`関数と`downgrade`関数を実行しましょう。

ベースとなるスクリプトは生成してもらいましょう。以下のコマンドを実行してください。
```shell
alembic revision -m "message"
```
以下ようなスクリプトが生成されます。

```python
"""test

Revision ID: 1fafad17d9ac
Revises: c413d4b1ff87
Create Date: 2024-01-24 14:10:58.880701

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = '1fafad17d9ac'
down_revision: Union[str, None] = 'c413d4b1ff87'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    pass


def downgrade() -> None:
    pass
```

このスクリプトの`upgrade`関数と`downgrade`関数を編集しましょう。
以下は、テーブル`my_table`のカラム`old_column_name`を`new_column_name`に変更した例です。

```python
def upgrade() -> None:
    op.alter_column(
        "my_table",
        "old_column_name",
        new_column_name="new_column_name",
        existing_type=sa.Integer(),
        existing_nullable=True,
    )


def downgrade() -> None:
    op.alter_column(
        "my_table",
        "new_column_name",
        new_column_name="old_column_name",
        existing_type=sa.Integer(),
        existing_nullable=True,
    )
```

##### nullable=Falseの列を追加したとき
まず、Columnの追加を行い、`alembic revision --autogenerate -m "message"`を実行した際、自動生成されたマイグレーションファイル内の`upgrade`関数は以下のようになっているはずです。

```python
def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column(
        "my_table",
        sa.Column(
            "my_column", sa.String(), nullable=False
        ),
    )
    # ### end Alembic commands ###
```

ただし、DBに既にデータが入っている状態で、このままマイグレーションを実行`alembic upgrade head`するとエラーになります。
その理由は、元々あったデータに追加された新しい列"my_column"が`NULL`になってしまうからです。

そのため、一度server_defaultを加えた状態で列の追加を行い、後でserver_defaultを削除しましょう。
`upgrade`関数を次のように変更してください。

```python
def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column(
        "my_table",
        sa.Column(
            "my_column", sa.String(), nullable=False, server_default="default"
        ),
    )
    op.alter_column("my_table", "my_column", server_default=None)
    # ### end Alembic commands ###

```

### ダウングレードしたいとき
バージョンを戻したいときは、まず戻したいバージョンのrevision idを調べます。
revision idは、マイグレーションファイルの先頭に付けられている英数字です。また、マイグレーションファイルの中にも記述されています。

マイグレーションファイル内の記述例
```python
"""rename task temporal resource column

Revision ID: d53e47758190
Revises: b2644f7c7d2b
Create Date: 2023-12-19 08:38:25.155965

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = "d53e47758190"
down_revision: Union[str, None] = "b2644f7c7d2b"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None
```

上記の例だとrevision idは、d53e47758190です。
revision idが判明すれば、以下のコマンドを実行するだけです。

```shell
alembic downgrade d53e47758190
```

また、初期化したい場合は、以下のコマンドを実行します。

```shell
alembic downgrade base
```

### その他コマンド
- 履歴の確認
```shell
alembic history
> fa02cad6f436 -> 9e7bef8dc48b (head), add create update at user table
> <base> -> fa02cad6f436, create initial tables
```

- 現在のリビジョン確認
```shell
alembic current
> INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
> INFO  [alembic.runtime.migration] Will assume transactional DDL.
> 9e7bef8dc48b (head)
```

- 特定リビジョンへのアップグレード
```shell
alembic upgrade {revision id}
```

実行例
```shell
alembic upgrade d53e47758190
```

## AlembicをFastAPIのプロジェクトで使ってみる

### セットアップ
以下の場所でセットアップを行ないます。
```
.
└── app
    ├── __init__.py
    ├── api
    ├── core
    ├── create_initial_data.py
    ├── crud
    ├── db
    ├── main.py
    ├── migrate.py
    ├── models
    └── schemas
```

以下のコマンドを実行しましょう。
```shell
pip install alembic
alembic init "alembic"
```

以下のようになります。
```
.
├── alembic
│   ├── README
│   ├── env.py
│   ├── script.py.mako
│   └── versions
├── alembic.ini
└── app
    ├── __init__.py
    ├── api
    ├── core
    ├── create_initial_data.py
    ├── crud
    ├── db
    ├── main.py
    ├── migrate.py
    ├── models
    └── schemas
```

次に`alembic/env.py`を修正しますが、この`alembic/env.py`に渡す`Base.metadata`ですが、Baseをインポートしたときに、Baseクラスを継承した全てのclass(テーブル定義)が読み込まれている必要があります。

そのため、以下のようなファイルを作成します。

`app/db/base.py`
```python
from app.db.base_class import Base

from app.models import *
```

`alembic/env.py`は、この`app/db/base.py`から`Base`を読み込むようにしましょう。

それでは、`alembic/env.py`を以下のように編集してください。

```python
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Baseクラスを読み込む
from app.db.base import Base

# DBのURI指定のため、設定を読み込む
from app.core.config import settings

# Base.metadataを指定
target_metadata = Base.metadata


def run_migrations_offline() -> None:
    context.configure(
        url=settings.SQLALCHEMY_DATABASE_URI,  # DBのURIを設定
        target_metadata=target_metadata,
        literal_binds=True,
        compare_type=True,  # 列タイプの変更を検知したい場合、True
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    # sqlalchemy.urlを変更するため、configをはじめに編集
    configuration = config.get_section(config.config_ini_section, {})
    configuration["sqlalchemy.url"] = settings.SQLALCHEMY_DATABASE_URI
    connectable = engine_from_config(
        configuration,  # 編集したconfigを渡す
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,  # 列タイプの変更を検知したい場合は、True
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### マイグレーション

それでは、実際にマイグレーションを実行してみましょう。

#### 準備
今あるDBはすでにマイグレーションが行われた状態なので、データベースからテーブルを削除しておきましょう。

[4章DBマイグレーション(sqlalchemy)](../chapters/chapter4.md#db-マイグレーションsqlalchemy)で作成した`app/migrate.py`からテーブルを作成する行を除き、
```python
from sqlalchemy import create_engine

from app.db.base_class import Base
from app.core.config import settings
import app.models


def reset_database(engine):
    Base.metadata.drop_all(bind=engine)
    # Base.metadata.create_all(bind=engine)


if __name__ == "__main__":
    engine = create_engine(settings.SQLALCHEMY_DATABASE_URI, echo=True)
    reset_database(engine=engine)

```

以下のコマンドで実行します。
```shell
python -m app.migrate
```

#### スクリプト生成
以下のコマンドでマイグレーションスクリプトを自動生成します。
```shell
alembic revision --autogenerate -m "fist"
```

実行後、`alembic/versions/{revision_id}_first.py`ファイル（`{revision_id}`はalembicが自動生成したID）が作成され、中身を確認すると以下のようになっています。

`alembic/versions/859d9b67806f_first.py`
```python
"""first

Revision ID: fb25e3007956
Revises: 
Create Date: 2024-05-27 09:31:57.261184

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = 'fb25e3007956'
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('user',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('signin_id', sa.String(), nullable=False),
    sa.Column('hashed_password', sa.String(), nullable=False),
    sa.Column('name', sa.String(), nullable=True),
    sa.Column('role', sa.String(), nullable=True),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_user_id'), 'user', ['id'], unique=False)
    op.create_index(op.f('ix_user_name'), 'user', ['name'], unique=False)
    op.create_index(op.f('ix_user_signin_id'), 'user', ['signin_id'], unique=True)
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f('ix_user_signin_id'), table_name='user')
    op.drop_index(op.f('ix_user_name'), table_name='user')
    op.drop_index(op.f('ix_user_id'), table_name='user')
    op.drop_table('user')
    # ### end Alembic commands ###

```

ファイルを見てみると、`upgrade`に`user`テーブルを作成するコマンドが指定され、逆に`downgrade`には、`user`テーブルを削除するコマンドが指定されていることがわかります。

そして、この時DBに`alembic_version`というテーブルが作成されます。これは、Alembicがバージョン管理に用いるテーブルで、現在どのrevisionのテーブル定義になっているのかを示します。今は、まだ何も入っていないはずです。

#### マイグレーション実行
それでは、マイグレーションを行います。以下のコマンドを実行してください。

```shell
alembic upgrade head
```

コマンド実行後、`user`テーブルが作成されます。また、`alembic_version`というテーブルにファイル名などにある`revision id`が格納されます。

`user`テーブル
```
 id | signin_id | hashed_password | name | role 
----+-----------+-----------------+------+------
(0 rows)
```

`alembic_version`テーブル
```
 version_num  
--------------
 fb25e3007956
(1 row)
```

#### テーブル定義変更
テーブル定義の変更も行ってみましょう。ここでは、`user`テーブルに年齢を入れる`age`を追加します。

はじめに、データの引き継ぎができるという点を確認するため、`user`テーブルにデータを入れましょう。
[4章テスト用ユーザーデータの登録](../chapters/chapter4.md#テスト用ユーザーデータの登録スキップ可)で作成したスクリプトを使用します。作成していない方は、以下の内容のファイルを作成してください。

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

このファイルを以下のコマンドで実行します。

```shell
python -m app.create_initial_data
```

コマンド実行後、`user`テーブルにデータが挿入されます。

```
 id | signin_id |                       hashed_password                        | name | role  
----+-----------+--------------------------------------------------------------+------+-------
  1 | tarou     | $2b$12$J07RmN1JNDb4Sggfc/Gruu7EDz.AJxlAgst5lGOw5BzypyCMny6J2 | 太郎 | User
  2 | john      | $2b$12$nhYtGcL7AmcMKixPCYG31.9wwxN1lxQsFqMf9viGgQ5I1UwG3CaX. |      | User
  3 | admin     | $2b$12$gPonJjMaVQzWpwTkT1R.MuaQDgrkCbWK6l8CBli0wjf6mYK.ToL1K |      | Admin
(3 rows)
```

次にSQLALchemyのモデルを変更します。`User`に`age`を追加しましょう。`app/models/user.py`を以下のように変更します。

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
    age = Column(Integer, nullable=False) # 追加

```

次にマイグレーションスクリプトを生成します。

```shell
alembic revision --autogenerate -m "add age to user"
```

実行後、以下の内容のファイルが生成されます。

`alembic/versions/bf537ef17652_add_age_to_user.py`
```python
"""add age to user

Revision ID: bf537ef17652
Revises: fb25e3007956
Create Date: 2024-05-27 11:04:26.365449

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = 'bf537ef17652'
down_revision: Union[str, None] = 'fb25e3007956'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column('user', sa.Column('age', sa.Integer(), nullable=False))
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_column('user', 'age')
    # ### end Alembic commands ###

```

マイグレーションを実行してみます。

```shell
alembic upgrade head
```

このマイグレーションは、エラーとなるはずです。これは、新しく追加した`age`は`nullable=False`で、列`age`を追加する際、もともとテーブルに入っているデータは`age=NULL`となってしまうためです。

これは、具体的に修正が必要なケースの[nullable=Falseの列を追加したとき](#nullablefalseの列を追加したとき)に当たります。

それでは、マイグレーションスクリプトを修正しましょう。

```python
"""add age to user

Revision ID: bf537ef17652
Revises: fb25e3007956
Create Date: 2024-05-27 11:04:26.365449

"""

from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = "bf537ef17652"
down_revision: Union[str, None] = "fb25e3007956"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column(
        "user",
        sa.Column(
            "age",
            sa.Integer(),
            nullable=False,
            server_default="25",  # 追加
        ),
    )
    op.alter_column("user", "age", server_default=None)
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_column("user", "age")
    # ### end Alembic commands ###

```

ここで`upgrade`関数が行っているのは、一旦`age`列を`default=25`として追加し、その後`user`テーブルの`age`列を`default=None`として、`default`の設定を取り除いています。

これで、マイグレーションが実行できるようになったので、以下のコマンドでマイグレーションを行いましょう。

```shell
alembic upgrade head
```

`user`テーブルを見てみると以下のように`age`列が追加され、もともとあったデータは`age=25`になっているはずです。

```
 id | signin_id |                       hashed_password                        | name | role  | age 
----+-----------+--------------------------------------------------------------+------+-------+-----
  1 | tarou     | $2b$12$J07RmN1JNDb4Sggfc/Gruu7EDz.AJxlAgst5lGOw5BzypyCMny6J2 | 太郎 | User  |  25
  2 | john      | $2b$12$nhYtGcL7AmcMKixPCYG31.9wwxN1lxQsFqMf9viGgQ5I1UwG3CaX. |      | User  |  25
  3 | admin     | $2b$12$gPonJjMaVQzWpwTkT1R.MuaQDgrkCbWK6l8CBli0wjf6mYK.ToL1K |      | Admin |  25
(3 rows)
```

## まとめ
以上のようにAlmebicを利用することで、テーブルの定義の変更が非常に楽になります。開発途中では、テーブルの定義の変更が入ることは避けられないことが多いです。SQLAlchemyだけでもマイグレーションはできますが、定義変更をするには不便です。実際の開発では是非Alembicを利用しましょう。


## [Prev: Chapter5 セキュリティの実装](../chapters/chapter5.md)
