---
title: "GitHub ActionsでのGoのキャッシュ戦略を考えてみた"
emoji: "⚙️"
type: "tech"
topics: ["go","cicd","githubactions","tech"]
published: true
---

# はじめに
最近Go環境のCI構築を見直す機会があり、その際にGoのキャッシュ制御周りを色々と調整してみました。

Go環境における、CIでのキャッシュ設定について、情報が断片的であったため、現時点での自分の整理を兼ねて記事にまとめてみました。
なお、以降の内容は、主にGitHub Actionsを用いたWebアプリケーション開発の場合を主軸においています。

# 要点
先に、本記事の要点をまとめます。

- `actions/setup-go`では、Goのversion及び`go.mod`が同じ場合はキャッシュキーは一様になります
- `actions/setup-go`を用いた複数のワークフローがあり、キャッシュキーが同じ場合、キャッシュされる内容は複数のワークフローのうち、最初に保存されたもの(早いもの勝ち)になります
  - 同時に複数ワークフローを稼働した場合、モジュール・ビルドキャッシュが比較的軽量で短時間で終了するワークフローが先にキャッシュされることが多々あります
  - 本来長時間かかるワークフローに対して、できるだけキャッシュを効かせて実行時間の短縮を図りたい一方で、軽量なキャッシュが先に保存され、キャッシュを有効活用できないことが多々あります
- キャッシュ内容を細かく制御したい場合は、`actions/setup-go`でのキャッシュ制御は利用せず、`actions/cache`などを用いて自前でキャッシュキーを割り振ることで制御することが望ましいです
  - キャッシュの保存・復元を更に細かく制御したい場合は、`actions/cache/restore`などで復元のみ行うようにするなどの工夫の余地もあります

# 問題: setup-goのキャッシュと複数ワークフローの競合
goプロジェクトにおいて、アプリケーションのビルド・lint・テストなどを異なるワークフローで設定してCIを行っている場合があるかと思います。
また、場合によっては、CI上からgoコードをビルドして単発のコマンド実行を行ったりということもあるかと思います。

このような場合、ワークフローの構成としては、以下のように、ビルド・lint・テスト・単発コマンドなどの複数ワークフローが用意されており、mainブランチへのマージをトリガーに複数のワークフローが同時に実行されることがあるかと思います。
- ビルド：`go build <所定のパス>`を実行
- lint：`lint`や`go build ./...`を実行
- test：`go test ./... -race`を実行
- command：`go run <所定のパス>`を実行

例.
```yaml
# .github/workflows/build.yml
name: Build
on:
  push:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
      - name: Build
        run: go build app/main.go

# .github/workflows/lint.yml
name: Lint
on:
  push:
    branches:
      - main
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
      - name: go build
        run: go build ./...
      - name: golangci-lint
        uses: golangci/golangci-lint-action@971e284b6050e8a5849b72094c50ab08da042db8 # v6.1.1
        with:
          version: v1.52.2

# .github/workflows/test.yml
name: Test
on:
  push:
    branches:
      - main
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
      - name: go test
        run: go test ./... -race

# .github/workflows/command.yml
name: Command
on:
  workflow_dispatch:
jobs:
  command:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
      - name: Run command
        run: go run cmd/main.go
```

それぞれのワークフローで必要とされるモジュールやビルド時の引数(環境変数)、ビルドキャッシュが異なり、利用側としてはできるだけ時間のかかるワークフローについてはキャッシュを保存・活用して、CIの実行時間を短縮したいと考えることが多いかと思います。

しかし、`setup-go`のキャッシュ機構を利用した場合、上記のように複数ワークフローを同時に動かした場合、ちょっとした不都合が生じます。

`setup-go`のキャッシュキーは[`setup-go-${platform}-${arch}-${linuxVersion}go-${versionSpec}-${fileHash}`](https://github.com/actions/setup-go/blob/v6.0.0/src/cache-restore.ts#L35)(tag: v6.0.0時点)となっており、Goのversion及び`go.mod`の内容が同じであれば、キャッシュキーは同じになります。

そのため、時間のかかるワークフローにおけるキャッシュを保存することでキャッシュヒット率を上げ、次回以降のCIの実行時間を短縮したいと考えます。

しかし、キャッシュキーは一様故に、他の比較的軽量なワークフローでのキャッシュが先に保存されてしまうことがあります。結果として、時間のかかるワークフローに対してキャッシュがあまり効かず、CIの実行時間があまり短縮されないことがあります。

# 対応策: GoのCIにおけるキャッシュ戦略

[`actions/setup-go`では、キャッシュキーやrestore/saveを細かく制御できるようにする予定はありません](https://github.com/actions/setup-go/blob/v6.0.0/docs/adrs/0000-caching-dependencies.md)。
> We don't pursue the goal to provide wide customization of caching in scope of actions/setup-go action. The purpose of this integration is covering ~90% of basic use-cases. If user needs flexible customization, we should advice them to use actions/cache directly.

そのため、キャッシュ戦略としては、以下のようなものが考えられます。
- ワークフローを分けずに、1つのワークフロー内で完結させる
- ワークフローやジョブのトリガーを調整して、ワークフローの実行順を制御・調整する
- 当該ワークフロー(ビルド・テストなど)とは別の部分で、キャッシュ周りを調整する専用のワークフローやスクリプトなどでキャッシュを管理する
- `actions/setup-go`のキャッシュは利用せず、`actions/cache`でキャッシュキーやsave/restoreを細かく制御する

ワークフローを分けずに1つのワークフロー内で完結させる方法は、キャッシュキーの競合が起きないため、キャッシュを有効活用できる一方で、並列で動かさない分ワークフローの実行時間が長くなるというデメリットがあります。
また、ビルド・単体コマンドとlint・テストなどを同じワークフローにまとめるのは、役割分担の観点からも難しい場合があります。

同様に、ワークフローやジョブのトリガーを調整することである程度キャッシュ保存タイミングの制御も可能ですが、mainブランチへのマージのタイミングでビルド・テスト・lintを同時トリガーしたいケースが往々にしてあるため、細かい調整は難しい事が多いです。

そこで、今回は、`actions/setup-go`でのキャッシュ機構は利用せず、`actions/cache`でキャッシュを細かく制御する方法を検討しました。

## 採用した方法: `actions/cache`での細かい制御

少しずつ調整を加えながら検討した結果、以下のような形にしました。

- `actions/setup-go`の`cache: false`を指定して、`actions/setup-go`のキャッシュを無効化する
- `actions/cache`を用いて、キャッシュを細かく制御する
  - キャッシュキーにワークフロー名やジョブ名を含めることで、ワークフローごとにキャッシュキーを分けるようにする
    - 例えば、`go-<hash>-build`、`go-<hash>-lint`、`go-<hash>-test`のようにする
  - キャッシュ効果が薄いワークフローの場合は、restoreのみ行う(`actions/cache/restore`)ようにする

先程のワークフロー例を基に修正した場合は、以下のようになります。

例.
```yaml
# .github/workflows/build.yml
name: Build
on:
  push:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        id: setup-go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
          cache: false
      - name: Cache Go modules
        uses: actions/cache@1bd1e32a3bdc45362d1e726936510720a7c30a57 # v4.2.0
        with:
          path: |
            ~/.cache/go-build
            ~/.go/pkg/mod
          key: go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-build # キャッシュキーを分ける
          restore-keys: |
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-
      - name: Build
        run: go build app/main.go

# .github/workflows/lint.yml
name: Lint
on:
  push:
    branches:
      - main
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        id: setup-go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
          cache: false
      - name: Cache Go modules
        uses: actions/cache@1bd1e32a3bdc45362d1e726936510720a7c30a57 # v4.2.0
        with:
          path: |
            ~/.cache/go-build
            ~/.go/pkg/mod
          key: go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-lint # キャッシュキーを分ける
          restore-keys: |
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-
      - name: go build
        run: go build ./...
      - name: golangci-lint
        uses: golangci/golangci-lint-action@971e284b6050e8a5849b72094c50ab08da042db8 # v6.1.1
        with:
          version: v1.52.2

# .github/workflows/test.yml
name: Test
on:
  push:
    branches:
      - main
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        id: setup-go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
          cache: false
      - name: Cache Go modules
        uses: actions/cache@1bd1e32a3bdc45362d1e726936510720a7c30a57 # v4.2.0
        with:
          path: |
            ~/.cache/go-build
            ~/.go/pkg/mod
          key: go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-test # キャッシュキーを分ける
          restore-keys: |
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-
      - name: go test
        run: go test ./... -race

# .github/workflows/command.yml
name: Command
on:
  workflow_dispatch:
jobs:
  command:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Set up Go
        id: setup-go
        uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version-file: 'go.mod'
          cache: false
      - name: Cache Go modules
        uses: actions/cache/restore@1bd1e32a3bdc45362d1e726936510720a7c30a57 # v4.2.0
        with:
          path: |
            ~/.cache/go-build
            ~/.go/pkg/mod
          key: go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-build # buildのキャッシュを利用
          restore-keys: |
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-
            go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-
      - name: Run command
        run: go run cmd/main.go
```

キャッシュキーの分け方について、
- 今回はビルド・lint・testで分けました。
  - テストでは、-raceオプションをつけており、ビルドキャッシュが異なるため、キャッシュキーを分けました。
  - lintは、`go build ./...`を実行しており、比較的実行時間が長いこともあり、できるだけキャッシュを活用したいため、今回はビルドキャッシュキーを分けました。
- キャッシュキーには、goのversion(`${{ steps.setup-go.outputs.go-version }}`)と`${{ hashFiles('**/go.sum') }}`を含めつつ、末尾にjob名を入れる形にしました。
  - job名を含んだキーがヒットしない場合は`go-${{ runner.os }}-${{ steps.setup-go.outputs.go-version }}-${{ hashFiles('**/go.sum') }}-`などの他のキャッシュキーをヒットさせることである程度キャッシュを活用できることを想定しています。
- 単体コマンドなど軽量ワークフローの場合は、
  - 他のワークフローのキャッシュを使い回すことで十分であることが多かったため、他のワークフローのキャッシュキーを設定しています。
  - また当該ワークフローのキャッシュを保存しても他では有効的に活用できないため、`actions/cache/restore`を用いることで、restoreのみ行うように設定したりしています。以前軽量ワークフローが他のワークフローと同時にトリガーされてしまい、軽量ワークフローのキャッシュが先に保存されてしまうことがあったため、明示的にrestoreのみ行うようにしています。

# まとめ
GoのCIにおけるキャッシュ戦略について、自分なりに整理してみました。

変更前後を比較すると、[`actions/setup-go`のみでキャッシュ保存できるようになった](https://github.com/actions/setup-go/tree/v6.0.0?tab=readme-ov-file#v4)現時点で、あえて`actions/cache`を用いて自前で制御するのは少し手間な形にも感じられますが、できるだけキャッシュを意図通り制御してCI実行時間を短縮することを考えた場合、今現在はこのような形に整理しています。
