---
title: Chapter0 FastAPI 開発環境構築
---

<!-- omit in toc -->
# FastAPI 開発環境構築

<!-- omit in toc -->
## 目次
- [Pythonの仮装環境構築](#pythonの仮装環境構築)
  - [Miniconda](#miniconda)
    - [Condaの基本的な使い方](#condaの基本的な使い方)
  - [poetry (推奨)](#poetry-推奨)
    - [poetryの基本的な使い方](#poetryの基本的な使い方)
- [Visual Studio Code　(エディタ)](#visual-studio-codeエディタ)
    - [VSCodeの拡張機能](#vscodeの拡張機能)
- [Next: Chapter1 FastAPI はじめのステップ](#next-chapter1-fastapi-はじめのステップ)


## Pythonの仮装環境構築
Pythonは、開発するものによってパッケージが異なりますし、Python自体のバージョンアップも頻繁に行われているので、開発する１つのプロジェクトまたはパッケージごとに、Pythonのバージョンとパッケージを管理することが望ましいです。

そのために、開発する１つのプロジェクトまたはパッケージごとPythonの仮装環境の構築を行います。仮想環境では、それぞれpythonのバージョンとインストールするパッケージを環境ごとに指定します。ここでは、Pythonの仮装環境を作るためにMinicondaを利用します。

仮想環境構築のやり方としては、`pyenv`と`venv`を使った形式も有名です。どちらでも問題ありませんので、好きなものを使ってください。

### Miniconda
[Miniconda](https://docs.anaconda.com/free/miniconda/)は、Anacondaの最小構成版です。Anacondaは、パッケージ管理や環境管理を簡単にするためのCondaに加えて、データサイエンスや機械学習向けのパッケージがプリインストールされており、Pythonの統合開発環境なども付随したプラットフォームです。

だたし、ここではCondaが使えれば十分ですので、Anacondaの最小構成版の[Miniconda](https://docs.anaconda.com/free/miniconda/)を利用します。

Minicondaの公式のインストール方法は[こちら](https://docs.anaconda.com/free/miniconda/miniconda-install/)です。
コマンドでもインストールできます。インストール方法は[こちら](https://docs.anaconda.com/free/miniconda/#quick-command-line-install)です。

Minicondaインストール後、この演習のために使う仮装環境を作っておきましょう。以下のコマンドで行います。`{仮想環境名}`は自分で好きなように設定してください。

```shell
conda create -n {仮想環境名} python=3.12
```

#### Condaの基本的な使い方
- 仮装環境の作成
  ```shell
  conda create -n {仮想環境名} python=X.XX.X
  ```
- 仮装環境の有効化・無効化
  - 有効化
    ```shell
    conda activate {仮想環境名}
    ```
  - 無効化
    ```shell
    conda deactivate
    ```
- 仮装環境の一覧表示
  ```shell
  conda info -e
  ```
- 仮装環境の削除
  ```shell
  conda remove -n {仮想環境名} --all
  ```

作成した仮想環境ごとにパッケージのインストールを行います。パッケージは、pythonに付属しているpipでも、condaでも可能です。
- パッケージのインストール
  - pip
    ```shell
    pip install {パッケージ名}
    ```
  - conda
    ```shell
    conda install {パッケージ名}
    ```
- パッケージのアップデート
  - pip
    ```shell
    pip install -U {パッケージ名}
    ```
  - conda
    ```shell
    conda update {パッケージ名}
    ```
- パッケージのアンインストール
  - pip
    ```shell
    pip uninstall {パッケージ名}
    ```
  - conda
    ```shell
    conda remove {パッケージ名}
    ```

### poetry (推奨)
仮装環境ごとに`pip`や`conda`を用いてパッケージのインストールを行っても良いですが、複数人で開発する際、利用しているパッケージの共有が少々面倒です。
複数人で開発を行う場合、パッケージ管理ツールとして、`poetry`を使うことを推奨します。

`poetry`は、`pip`と同じようにPythonのパッケージマネージャーですが、パッケージ管理ファイル`pyproject.toml`を生成してくれるので、そのファイルがあれば、作成者以外の人もコマンド１つで、同じパッケージをインストールできます。

また、`poetry`は、インストールするパッケージの依存関係も解決してくれるので、`pip`よりも厳密にパッケージのバージョン管理ができます。

condaで作成した仮想環境の場合、`poetry`は、pipでインストールできます。以下のコマンドでインストールしましょう。

```shell
pip install poetry
```

#### poetryの基本的な使い方
- プロジェクトの初期化
  - 管理するプロジェクトのディレクトリで、以下のコマンドを実行します。パッケージ管理ファイル`pyproject.toml`が生成されます。
  ```shell
  poetry init
  ```
- パッケージの追加
  - 新しく必要になったパッケージは、以下のコマンドでインストールします。
  ```shell
  poetry add {パッケージ名}
  ```
- pyproject.tomlのパッケージのインストール
  - pyproject.tomlに記述してあるパッケージを全てインストールします。
  ```shell
  poetry install --no-root
  ```
- 環境でのコマンド実行
  - poetryで作成した環境でのコマンド実行は、以下のように行います。
  ```shell
  poetry run {コマンド}
  ```
  - 実行例
  ```shell
  poetry run python main.py
  ```


## Visual Studio Code　(エディタ)
以下のURLよりインストール
もし、すでに使い慣れているPython用のエディタがあれば、そちらを使ってください。

https://code.visualstudio.com/

#### VSCodeの拡張機能
すでに類似の機能を持ったものを使っている場合などは、インストールする必要はありません。

- Python (必須)
  - Pythonのコード編集が楽になる拡張機能
  - URL：[https://marketplace.visualstudio.com/items?itemName=ms-python.python](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
  - 識別子：ms-python.python
- Black Formatter
  - [Black](https://github.com/psf/black)を利用したPythonのフォーマッタ
  - [https://marketplace.visualstudio.com/items?itemName=ms-python.black-formatter](https://marketplace.visualstudio.com/items?itemName=ms-python.black-formatter)
  - 識別子：ms-python.black-formatter


## [Next: Chapter1 FastAPI はじめのステップ](../chapters/chapter1.md)
