---
title: "Rust製TOMLフォーマッター・リンターのTaploのススメ"
emoji: "⚙️"
type: "tech"
topics: ["TOML", "vscode"]
published: true
---

# はじめに

昨今設定ファイルにtoml形式を求めるツール群が増えてきたように感じます。

tomlファイルを扱っていくうえで
- keyをアルファベット順に並べたい
- valueの位置を縦方向に揃えたい

などの書式を整えたいことがあると思います。

```toml
# before
[table]
aaa = "bbb"
eee = "fff"
ccccc = "ddd"

# after
[table]
aaa   = "bbb"
ccccc = "ddd"
eee   = "fff"
```

そこで、今回TOMLファイルの書式周りを整えてくれるRust製ツールであるTaploを紹介したいと思います。

# Taploとは

Taploは、Rust製のTOML言語サーバー・フォーマッター・リンターのツールです。
Taploを知らない方でも、VSCode拡張「[Even Better TOML](https://marketplace.visualstudio.com/items?itemName=tamasfe.even-better-toml)」の内部で使用されているツールといえば、馴染みのある方も多いかもしれません。

- 公式サイト：https://taplo.tamasfe.dev/
- GitHubリポジトリ：https://github.com/tamasfe/taplo

# インストール

Taploのインストールには複数の方法があります。
詳細は[ドキュメント](https://taplo.tamasfe.dev/cli/introduction.html)を参照してください

## brew経由でのダウンロード
```sh
brew install taplo
```

## Cargoを使用したインストール

```bash
cargo install taplo-cli --locked
```

## npm経由でのインストール
```bash
npm install -g @taplo/cli
```

npxを使用して直接実行することもできます

```bash
npx @taplo/cli format
```

# セットアップ
## taplo.tomlの設定

プロジェクトルートに`taplo.toml`（または`.taplo.toml`）を配置することで、Taploの動作を細かくカスタマイズできます。

### 基本的な設定例

```toml
# taplo.toml

[formatting]
align_entries       = true   # エントリを垂直に整列
allowed_blank_lines = 1      # 許可される連続空白行の最大数
array_auto_collapse = false  # 配列が1行に収まる場合でも自動的に縮小しない
indent_string       = "    " # インデントに使用する文字列（スペース4つ）
reorder_keys        = true   # キーをアルファベット順に並び替え
```

### フォーマッタ設定の詳細
Taploの特徴として、書式に関して設定ファイルを通して、細かく制御することができます。
詳細は[公式ドキュメント](https://taplo.tamasfe.dev/configuration/formatter-options.html)を参照してください。

#### align_entries
- **説明**: エントリを垂直に整列させます。テーブルヘッダー、コメント、空白行で区切られたエントリは整列されません。
- **デフォルト**: false
- **例**:
```toml
# align_entries = true
[table]
name     = "value"
longname = "value"

# align_entries = false
[table]
name = "value"
longname = "value"
```

#### align_comments
- **説明**: エントリや配列要素の後に続くコメントを垂直に整列させます。
- **デフォルト**: true
- **例**:
```toml
# align_comments = true
name = "value"     # コメント1
longname = "value" # コメント2

# align_comments = false
name = "value" # コメント1
longname = "value" # コメント2
```

#### array_trailing_comma
- **説明**: 複数行配列に末尾カンマを追加します。
- **デフォルト**: true
- **例**:
```toml
# array_trailing_comma = true
array = [
    "item1",
    "item2",
]

# array_trailing_comma = false
array = [
    "item1",
    "item2"
]
```

#### array_auto_expand
- **説明**: 配列がcolumn_width文字を超える場合、自動的に複数行に展開します。
- **デフォルト**: true
- **例**:
```toml
# array_auto_expand = true (column_width = 20)
array = [
    "long_item_name_1",
    "long_item_name_2"
]

# array_auto_expand = false
array = ["long_item_name_1", "long_item_name_2"]
```

#### array_auto_collapse
- **説明**: 配列が1行に収まる場合、自動的に縮小します。
- **デフォルト**: true
- **例**:
```toml
# array_auto_collapse = true
array = ["a", "b", "c"]

# array_auto_collapse = false
array = [
    "a",
    "b",
    "c"
]
```

#### compact_arrays
- **説明**: 単一行配列の内側の空白パディングを省略します。
- **デフォルト**: true
- **例**:
```toml
# compact_arrays = true
array = ["a", "b", "c"]

# compact_arrays = false
array = [ "a", "b", "c" ]
```

#### compact_inline_tables
- **説明**: インラインテーブルの内側の空白パディングを省略します。
- **デフォルト**: false
- **例**:
```toml
# compact_inline_tables = true
table = {key1="value1", key2="value2"}

# compact_inline_tables = false
table = { key1 = "value1", key2 = "value2" }
```

#### inline_table_expand
- **説明**: インラインテーブル内の値（配列など）を展開します。
- **デフォルト**: true
- **例**:
```toml
# inline_table_expand = true
table = {
    name = "value",
    array = [
        "item1",
        "item2"
    ]
}

# inline_table_expand = false
table = { name = "value", array = ["item1", "item2"] }
```

#### compact_entries
- **説明**: `=`の周りの空白を省略します。
- **デフォルト**: false
- **例**:
```toml
# compact_entries = true
key="value"

# compact_entries = false
key = "value"
```

#### column_width
- **説明**: 配列が複数行に展開される目標最大列幅です。
- **デフォルト**: 80
- **例**:
```toml
# column_width = 20
array = [
    "long_item_name"
]

# column_width = 80
array = ["long_item_name"]
```

#### indent_tables
- **説明**: サブテーブルが順番通りに来る場合、インデントします。
- **デフォルト**: false
- **例**:
```toml
# indent_tables = true
[table]
    [table.subtable]
    key = "value"

# indent_tables = false
[table]
[table.subtable]
key = "value"
```

#### indent_entries
- **説明**: テーブル下のエントリをインデントします。
- **デフォルト**: false
- **例**:
```toml
# indent_entries = true
[table]
    key = "value"

# indent_entries = false
[table]
key = "value"
```

#### indent_string
- **説明**: インデントに使用する文字列。タブやスペースが推奨されますが、技術的には任意の文字列が使用可能です。
- **デフォルト**: 2つのスペース (" ")
- **例**:
```toml
# indent_string = "    " (4つのスペース)
[table]
    key = "value"

# indent_string = "\t" (タブ)
[table]
	key = "value"
```

#### trailing_newline
- **説明**: ソースの末尾に改行を追加します。
- **デフォルト**: true
- **例**:
```toml
# trailing_newline = true
key = "value"


# trailing_newline = false
key = "value"
```

#### reorder_keys
- **説明**: 空白行で区切られていないキーをアルファベット順に並び替えます。
- **デフォルト**: false
- **例**:
```toml
# reorder_keys = true
[table]
aaa = "value"
bbb = "value"
ccc = "value"

# reorder_keys = false
[table]
ccc = "value"
aaa = "value"
bbb = "value"
```

#### reorder_arrays
- **説明**: 空白行で区切られていない配列値をアルファベット順に並び替えます。
- **デフォルト**: false
- **例**:
```toml
# reorder_arrays = true
array = ["aaa", "bbb", "ccc"]

# reorder_arrays = false
array = ["ccc", "aaa", "bbb"]
```

#### reorder_inline_tables
- **説明**: インラインテーブルをアルファベット順に並び替えます。
- **デフォルト**: false
- **例**:
```toml
# reorder_inline_tables = true
table = { aaa = "value", bbb = "value" }

# reorder_inline_tables = false
table = { bbb = "value", aaa = "value" }
```

#### allowed_blank_lines
- **説明**: 許可される連続空白行の最大数です。
- **デフォルト**: 2
- **例**:
```toml
# allowed_blank_lines = 1
[table1]
key = "value"

[table2]
key = "value"

# allowed_blank_lines = 0
[table1]
key = "value"
[table2]
key = "value"
```

#### crlf
- **説明**: CRLF改行コードを使用します。
- **デフォルト**: false
- **例**:
```toml
# crlf = true
key = "value"\r\n

# crlf = false
key = "value"\n
```

### その他の設定
#### [Include](https://taplo.tamasfe.dev/configuration/file.html#include)/[Exclude](https://taplo.tamasfe.dev/configuration/file.html#exclude)
特定のファイルやディレクトリをフォーマットやリンティングの対象にする、もしくは、除外することができます。
特定ファイルではTaploの対象にしたくない場合に有効な設定です。

#### [Rule](https://taplo.tamasfe.dev/configuration/file.html#rules)
TOMLファイルのあるテーブル(`[xxx]`)に対しては個別にルールを有効、無効にしたいといった場合が多々出てくると思います。

よくあるのが、基本は`reorder_arrays`で配列要素をアルファベット順に並び替えたいが、特定の配列に対しては並び替えたくないといった場合です。
そのような場合に、`rule`を使用して、特定のテーブルに対して、ルールを有効、無効にすることができます。

```toml
[formatting]
reorder_arrays = true

[[rule]]
include = ["pyproject.toml"]
keys = ["tool.pdm.dev-dependencies"]

[rule.formatting]
reorder_arrays = false
```


# 使い方

## CLI

TaploはCLIとして、TOMLファイルのリント、フォーマットを行うことができます。
npmやCargo以外のプロジェクトのタスクランナーとして追加する際には、別途グローバルにインストールする前提でTaploコマンドを用いるか、もしくは、npxのコマンドで実行するのがいいと思います。

```bash
# リント
taplo format --check

# フォーマット
taplo format

# 単一ファイルのフォーマット
taplo format pyproject.toml

# npxを使用して実行
npx @taplo/cli format
```

## エディタ統合
TaploはLSPとして、使用することもできます。
ここではVSCodeの設定について説明します。

### VS Code
VS Codeでは「Even Better TOML」拡張をインストールすることで、 上記taplo.tomlの設定を応じて、自動でフォーマットを行うことができます。

- プロジェクトのルートディレクトリに`taplo.toml`を配置します。
- `.vscode/settings.json`に以下の設定を追加します。

```json
// settings.json
{
  "[toml]": {
    "editor.defaultFormatter": "tamasfe.even-better-toml",
    "editor.formatOnSave": true
  },
  // "evenBetterToml.taplo.configFile.path": "taplo.toml" //ルートディレクトリ以外で使用したい場合は個別で設定
}
```

ここで、ハマりがちなのが一度エディタ起動後に`taplo.toml`の内容を変更したとしても、`onSave`時のフォーマットにはすぐに反映されないため、もし、`taplo.toml`の内容を変更した場合は、VSCodeを再起動などして、`taplo.toml`を再読み込みする必要がある点注意してください。（N敗）

## CI
TaploをCI上でも実行することで書式の自動検知を行うことができます。


### GitHub Actions での例

```yaml
name: TOML Lint
on: [push, pull_request]

jobs:
  toml-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: lint TOML
        run: npx @taplo/cli format --check .
```


# 補足情報

## Taploプロジェクトの現状と他ツール

これまでTaploを紹介してきましたが、Taploの現状についても触れておきます。
TaploはTOMLのリンター・フォーマッター・LSPサーバーのツールとして一定の人気を集めましたが、ツール作者はここ数年は開発から退き、メンテナー・オーナーから退くことを発表しています。
そのため、Taploプロジェクトは今後GitHub Organization化し、複数のメンテナーで継続していくことが計画されていますが、現時点ではまだ具体的な移行は行われておらず、今後の動向には注視が必要です。
cf.https://github.com/tamasfe/taplo/issues/715

上記のような状況ではありますが、Taploがあまり知られていない・比較的薄い形での使用用途での導入・離脱が容易であると思われるため、将来的なことを考慮しても導入する価値があると思い、今回Taploを紹介させてもらいました。

また、最近新しいツールとして、[tombi](https://tombi-toml.github.io/tombi)というツールも登場しております。
こちらはゼロコンフィグを謳っているように、Taploと比べてフォーマット周りの設定の自由度は低いですが、

今現在も継続的に開発が行われており、LSPによる補完だけあればよかったり、整形に関する細かい設定を必要としない方には、tombiを検討してみるのも良いかもしれません。（私も検討中です）

# まとめ
昨今TOMLファイルをいじる機会が多くなっていますが、その際に書式・整形周りについて気になっている方はTaploを一度使ってみてはいかがでしょうか。
