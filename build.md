---
SPDX-License-Identifier: Community-Spec-1.0
SPDX-FileCopyrightText: Copyright 2024 The SPDX Contributors
---

# 仕様ウェブサイトの構築

仕様ウェブサイトは次にフローで構築される：

```text
  +-------------------+
  |[spdx-3-model]     |
  | +- model/        ---- Constrained-Markdown files -+
  | +- model.drawio  -----------------+               |
  +-------------------+               |               |
                                      |               |
  +-------------------+               v               |
  |[spdx-spec]        |            draw.io            |
  | +- docs/          |            (manual)           v
  | |  +- annexes/    |               |          spec-parser
  | |  +- front/      |               |               |
  | |  +- images/  <---- PNG images --+               |
  | |  +- licenses/   |                               |
  | |  +- model/   <----- Processed Markdown files ---+
  | |  +- rdf/     <----- RDF files ------------------+
  | |  +- *.md        |
  | |  +- index.md    |
  | +- mkdocs.yml     |
  +-------------------+
          |
       mkdocs
          |
          v
  +-------------------+
  | HTML website      |
  | +- annexes/       |
  | +- ...            |
  | +- *.md           |
  | +- index.html     |
  +-------------------+
```

## 1. 前提条件

GitやPythonとは別に、[MkDocs](http://mkdocs.org)を仕様ウェブサイトを構築するマシンにインストールする必要がある。
[installation instructions](http://www.mkdocs.org/#installation)を参考にインストールを行う。

`mkdocs.yml` は MkDocsのためのコンフィグレーションファイルである。

<!--
[WeasyPrint](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html#installation)
is also required for generating PDF files. To enable PDF generation, set the
`ENABLE_PDF_EXPORT` environment variable to `1`.
-->

## 2. 入力ファイルの取得

次に、モデルファイル、他の仕様ファイル（本文、付録、前付け、ライセンス）やモデルパーサーを次のレポジトリをクローンして取得する：
[`spdx/spdx-3-model`](https://github.com/spdx/spdx-3-model),
[`spdx/spdx-spec`](https://github.com/spdx/spdx-spec), および
[`spdx/spec-parser`](https://github.com/spdx/spec-parser)
が `spdx-3-model`, `spdx-spec`, `spec-parser`それぞれに対応:

```shell
git clone https://github.com/spdx/spdx-3-model.git
git clone https://github.com/spdx/spdx-spec.git
git clone https://github.com/spdx/spec-parser.git
```

次のPythonの前提条件ファイルをインストールする:

```shell
pip3 install -r spdx-spec/requirements.txt
pip3 install -r spec-parser/requirements.txt
```

## 3. モデルファイルの処理 (マークダウン and RDF)

*仕様のモデルに関する部分以外（例えば、本文や付録）のみをレビューする場合は、*
*[step 4]([#4-html%E3%81%AE%E6%A7%8B%E7%AF%89)にスキップしてよい。*

`spdx/spdx-3-model` レポジトリのモデルファイルは制約付きマークダウンフォーマットで書かれている。
`spec-parser` によりこれらのモデルファイルが処理され、MkDocs用のオントロジーファイルと最終的なマークダウンファイルが生成される。

`spec-parser` はさらに、最終的なマークダウンファイルの自動フォーマットを行う。
例えば、"Properties"の リストをテーブルに変換する。

### 3.1 spec-parserによるモデルファイルの生成

プリプロセスされたモデルファイルのフォーマットを確認し、MkDocs用に準備するために、次のコマンドを実行する：

```shell
python3 spec-parser/main.py spdx-3-model/model parser_output
```

*(もし `parser_output` がすでに存在する場合は、`spec-parser` によって上書きされることはない)*

このコマンドの実行により`parser_output` ディレクトリにファイルが作られる。
サブディレクトリのうち、次に特に注目する：

- `parser_output/mkdocs` - MkDocs用に処理されたマークダウンファイル:
  これらのファイル (`.md` 拡張子) は複数のサブディレクトリに格納され、
  次のステップで、MkDocsによって処理されることになる。
- `parser_output/rdf` - オントロジー (RDF) ファイル:
  これらのファイル (`spdx-context.jsonld`, `spdx-model.json-ld`, `spdx-model.n3`,
  `spdx-model.pretty-xml`,`spdx-model.ttl`, `spdx-model.xml`など)は利用可能な状態にある

- `parser_output/mkdocs`: MkDocs用に処理されたマークダウンファイル(`.md`) 。
  これらのファイルは次のステップで、MkDocsによって利用される。
- `parser_output/rdf`:  `spdx-context.jsonld`, `spdx-model.json-ld`, `spdx-model.ttl`などを含むオントロジー (RDF) ファイル。
  これらのファイルは利用可能である。

さらに`parser_output/model-files.yml` ファイルが生成される。
このファイルは、`parser_output/mkdocs`内のファイルのリストを含み、
後で、MkDocsのコンフィグレーションに利用される。

### 3.2 生成されたファイルのコピー

生成されたマークダウンファイルとオントロジーファイルをを`docs/`ディレクトリにコピーする：

```shell
cp -R parser_output/mkdocs spdx-spec/docs/model 
cp -R parser_output/rdf spdx-spec/docs/rdf
```

MkDocsに新しいマークダウンファイルを確実に認識させるために、
`parser_output/model-files.yml`のモデルファイルリストを
`spdx-spec/mkdocs.yml`のMkDocsコンフィグレーションファイルに加える。

```shell
spdx-spec/bin/make-mkdocs-config.sh \
  -b spdx-spec/mkdocs.yml \
  -m parser_output/model-files.yml \
  -f spdx-spec/mkdocs-full.yml
```

完全なMkDocsコンフィグレーションファイルが`spdx-spec/mkdocs-full.yml`となる。

## 4. HTMLの構築

全ての仕様とモデルのファイルを準備し、
MkDocsによって、ウェブサイトを組み立てる。

*注意: 以下のすべてのコマンドは [step 3.2](#32-%E7%94%9F%E6%88%90%E3%81%95%E3%82%8C%E3%81%9F%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%81%AE%E3%82%B3%E3%83%94%E3%83%BC)で生成されるコンフィグレーションファイル、モデルファイルリスト、`mkdocs-full.yml`を利用する。*
*仕様のモデルに関する部分以外（例えば、本文や付録）のみをレビューする場合（step 3をスキップしているので）*
*`mkdocs.yml` を代わりに利用すること。*

次のコマンドは`spdx-spec/`ディレクトリ内で実行する：

- ウェブブラウザーで仕様をプレビューする場合:

  ```shell
  mkdocs serve --config-file mkdocs-full.yml
  ```

- 静的HTMLサイトを構築する場合:

  ```shell
  mkdocs build --config-file mkdocs-full.yml
  ```

- デバッグメッセージや　詳細情報を出力する場合:

  ```shell
  mkdocs build --verbose --config-file mkdocs-full.yml
  ```

## 5. ウェブサイトのコンフィグレーション

ウェブサイトの追加調整を行うためにコンフィグレーションを修正することができる。

`spdx-spec/` ディレクトリに`mkdocs.yml`ファイルがある。
これが、MkDocs用のコンフィグレーションファイルである。

例えば、このファイル内のサイトの名称やメインURL(正規URL）といったウェブサイトの詳細をカスタマイズすることができる。

ナビゲーションバーにページを加えるためには、`nav:`セクションにそのファイル名をリストする。
このセクション内のファイル名の順序によってナビゲーションバー内のページの順序が決まる。

## 6. spdx.github.ioの仕様のバージョン 

<https://spdx.github.io/spdx-spec/>のSPDX仕様は
[`.github/workflows/publish_v3.yml`](.github/workflows/publish_v3.yml)にあるワークフローで構築される。
このワークフローは MkDocsを用いたドキュメントの複数のバージョンを発行するために
[mike](https://github.com/jimporter/mike)を利用する。

発行されたバージョン、タイトル、別名は`gh-pages`ブランチにある
[versions.json](https://github.com/spdx/spdx-spec/blob/gh-pages/versions.json)
にリストされている。
これらのバージョンはウェブサイトのバージョン選択ドロップダウンによって表示できる。
GitHubワークフローファイルにある`name: Deploy and set aliases`ステップでタイトル、別名を定める。

特定の仕様バージョンのローカルテストには、mike必要ない。
