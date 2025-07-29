---
title: "GitHub CLIのおすすめの使い方"
emoji: "🐚"
type: "tech"
topics:
  - "github"
  - "cli"
  - "shell"
  - "gh"
  - "githubcli"
published: true
published_at: "2025-03-30 00:24"
---

# はじめに

今回GitHub CLI(ghコマンド)で私が普段使いしているスニペットを紹介したいと思います。【2025/4/13追記】

# 本記事の目的

GitHub CLIはGitHub操作をコマンドラインで行えるツールです。
https://cli.github.com/

GitHub CLIを用いることでブラウザを開いてカーソルを操作することなく、ターミナル上で特に定常作業などを効率的に操作を行うことができます。

・・・と言いたいところなのですが、特にターミナルでの操作に慣れていない人にはなかなか日常的には使われていない印象があります。

私はターミナル・CLIを愛用しており、できるだけカーソル操作することなくキーボード操作することを信条（誇張あり）として、GitHub CLIを色々いじってきました。

ドキュメントやGitHub CLI紹介記事では、コマンド内容・操作の基本的な紹介が多く、
特にCLIに慣れていない人にとっては活用イメージ・カスタマイズ例が湧きずらいかもしれないと思い、
今回、私が使用しているスニペットを紹介したいと考えました。

スニペット例から、GitHub CLI及びCLIやターミナルの便利さやカスタマイズする楽しさなどが伝われば幸いです！

# スニペットのカスタマイズの前知識

スニペットを構築する際に、よりスニペットコマンドを便利にするいくつかの前知識をいくつか記述します。

1. インタラクティブ(対話的)選択ツールの使用

GitHub CLIに限らず、CLI操作をより使いやすくするツールとして、あいまい検索(fuzzy finder)によるインタラクティブな選択を行えるツールを用いると良いです。
インタラクティブ選択とは以下のようなものになります。
![](https://storage.googleapis.com/zenn-user-upload/50342e4e33b7-20250330.gif)
インタラクティブに選択した結果内容を任意のコマンドに渡すことができ、対話的な形で自由度高くコマンドを生成することが出来ます。

ツール例としては、以下などがあります。

- [fzf](https://github.com/junegunn/fzf)

  - fzfのインストール

  詳細は[README](https://github.com/junegunn/fzf?tab=readme-ov-file#installation)を参照してください(以下、インストール例)

  ```sh
  brew install fzf
  ```

  ```sh
  source <(fzf --zsh)
  ```

- [peco](https://github.com/peco/peco)

以降のスニペット例ではfzfを用いて記述してきます。

2. GitHub CLIのオプションについて

GitHub CLIではオプションを指定することで、表示内容を色々カスタマイズすることができます。
オプション内容は[各コマンドのドキュメント](https://cli.github.com/manual/gh_pr_list)や`--help`を付けて実行することで確認することができます。
色々カスタマイズできる要素があるため、よりこだわりがある人は自分好みに設定するといいと思います。
(私のスニペット例は自分好みにカスタマイズしているのもあり、あくまでカスタマイズの一例として見ていただければと思います。)

```sh
~/Programming/type-challenges % gh pr list --help
List pull requests in a GitHub repository.

The search query syntax is documented here:
<https://docs.github.com/en/search-github/searching-on-github/searching-issues-and-pull-requests>

For more information about output formatting flags, see `gh help formatting`.

USAGE
  gh pr list [flags]

FLAGS
      --app string        Filter by GitHub App author
  -a, --assignee string   Filter by assignee
  -A, --author string     Filter by author
  -B, --base string       Filter by base branch
  -d, --draft             Filter by draft state
  -H, --head string       Filter by head branch
  -q, --jq expression     Filter JSON output using a jq expression
      --json fields       Output JSON with the specified fields
  -l, --label strings     Filter by label
  -L, --limit int         Maximum number of items to fetch (default 30)
  -S, --search query      Search pull requests with query
  -s, --state string      Filter by state: {open|closed|merged|all} (default "open")
  -t, --template string   Format JSON output using a Go template; see "gh help formatting"
  -w, --web               List pull requests in the web browser

INHERITED FLAGS
      --help                     Show help for command
  -R, --repo [HOST/]OWNER/REPO   Select another repository using the [HOST/]OWNER/REPO format

EXAMPLES
  List PRs authored by you
  $ gh pr list --author "@me"

  List only PRs with all of the given labels
  $ gh pr list --label bug --label "priority 1"

  Filter PRs using search syntax
  $ gh pr list --search "status:success review:required"

  Find a PR that introduced a given commit
  $ gh pr list --search "<SHA>" --state merged


LEARN MORE
  Use `gh <command> <subcommand> --help` for more information about a command.
  Read the manual at https://cli.github.com/manual


```

- `--json`オプションについて
  ghコマンドでは、`--json`でfieldを設定することで、デフォルトでは表示されないfieldやfieldの順番などをカスタマイズすることができます。
  表示できるfieldは試しに無効なfieldを設定することで有効なfieldを確認することができます。
  私は日付の表示を`YYYY-MM-DD hh:mm:ss`の表記にカスタマイズしたり、authorの情報なども表示して、その情報でもfilterできるようにカスタマイズしていることが多いです。

  詳細は[こちら](https://cli.github.com/manual/gh_help_formatting)

  ```sh
  ~/Programming/type-challenges % gh pr list --json "x"
  Unknown JSON field: "x"
  Available fields:
    additions
    assignees
    author
    autoMergeRequest
    baseRefName
    body
    changedFiles
    closed
    closedAt
    comments
    commits
    createdAt
    deletions
    files
    headRefName
    headRefOid
    headRepository
    headRepositoryOwner
    id
    isCrossRepository
    isDraft
    labels
    latestReviews
    maintainerCanModify
    mergeCommit
    mergeStateStatus
    mergeable
    mergedAt
    mergedBy
    milestone
    number
    potentialMergeCommit
    projectCards
    projectItems
    reactionGroups
    reviewDecision
    reviewRequests
    reviews
    state
    statusCheckRollup
    title
    updatedAt
    url
  ```

  - `--jq`オプションについて
    `--json`オプションでfieldをカスタマイズした後は、`--jq`オプションで整形することができます。
    `| jq`でパイプすることで整形している例も多くみられますが、パイプでつながずとも`--jq`で同様のことができるというTIPSです。

3. スニペットは略語展開（abbr,abbreviation）で設定
   略語展開(abbreviation)とは、シェル上で入力した特定の短縮文字をあらかじめ設定しておいた文字列に展開してくれるaliasのようなものです。
   @[card](https://zenn.dev/ryo246912/articles/161b1e339b4b5a)
   スニペットは略語展開として設定することで、より手軽に使用することができます。
   以下は、私が今現在おすすめしている[zabrze](https://github.com/Ryooooooga/zabrze)での設定例です。
   ```yaml
   - name: gh pr checkout
     abbr: ghpc
     snippet:
       gh pr checkout $(gh pr list --search "user-review-requested:@me" --limit 100
       --json number,title,author,state,isDraft,updatedAt,createdAt,headRefName
       --jq '
       ["no","title","author","state","draft","updatedAt","createdAt","branch"],
       ( .[] |
       [.number
       , .title[0:50]
       , .author.login
       , .state
       , (if .isDraft then "×" else "〇" end )
       , (.updatedAt | strptime("%Y-%m-%dT%H:%M:%SZ") | strftime("%Y/%m/%d %H:%M:%S"))
       , (.createdAt | strptime("%Y-%m-%dT%H:%M:%SZ") | strftime("%Y/%m/%d %H:%M:%S"))
       , .headRefName])
       | @tsv
       '
       | column -ts $'\t'
       | fzf --no-sort --header-lines=1
       | awk '{ print $1}')
   ```

# GitHub CLIのおすすめのコマンド・スニペット例

以下順不同で私がよく行うスニペット群を紹介したいと思います。

## PR操作

### [PR一覧を確認(`gh pr list`)](https://cli.github.com/manual/gh_pr_list)

PR一覧はブラウザでは`https://github.com/<name>/<repo>/pulls`で閲覧できますが、CLIでも確認することができます。

以下では、statusがOpenもしくはDraft状態のPRを最新の30件を取得するコマンドになります。

```sh
gh pr list
```

上記コマンドのオプションを色々設定した、私のスニペットは以下になります。
ghコマンドでは後述するコマンドである`gh pr checkout <pr_no>`などのように、PRNoを指定することが多いため、そのNoを一覧から確認したい際に使うことがあります。

```sh
gh pr list --author "" --assignee "" --search "" --state all --limit 100
  --json number,title,author,state,isDraft,updatedAt,createdAt,headRefName
  --jq '
    ["no","title","author","state","draft","updatedAt","createdAt","branch"],
    ( .[] |
    [.number
    , .title[0:50]
    , .author.login
    , .state
    , (if .isDraft then "◯" else "☓" end )
    , (.updatedAt | strptime("%Y-%m-%dT%H:%M:%SZ") | strftime("%Y/%m/%d %H:%M:%S"))
    , (.createdAt | strptime("%Y-%m-%dT%H:%M:%SZ") | strftime("%Y/%m/%d %H:%M:%S"))
    , .headRefName])
    | @tsv
  '
  | column -ts $'\t'
  | fzf --no-sort --header-lines=1
```

ポイントとしては、

- 以下オプションで最新のPRを100件取得しつつ、fzfでfilterする形で閲覧出来るようにしています。

  - `--state all`: closed/merged状態のPR含めて、全てのPRを表示するようにしています。
    - PRのステータスを特定の内容のみに制限したい場合は`--state merged`のように指定することもできます。
  - `--limit 100`: limitを設定しない場合は30件のみですが、最大数である100件取得するようにしています。

- 以下の検索オプションは通常は明示する必要がないのですが、追加で設定したい場合に設定できるように設けています。

  - `--author ""`: 作成者フィルタ
    - たとえば自身のPRのみに制限したい場合は`--author "@me"`のように指定することができます。
  - `--assignee ""`: 担当者フィルタ
  - `--search ""`: 追加の検索条件
    - より検索条件をカスタマイズしたい場合は、`--search "word in:title"`のように指定することでより詳細な検索を行うことも可能です。
      検索詳細は[こちら](https://docs.github.com/ja/search-github/searching-on-github/searching-issues-and-pull-requests)

- 出力内容をカスタマイズするために、`--json`オプションで追加表示したいfieldを指定するとともに、`--jq`コマンドで表示内容を整形しています。

  - `--json`: 取得するフィールドを指定
  - `--jq`: jqクエリで出力を整形

- 出力内容の表形式で見栄え良いように以下で整形しています。
  - `column -ts $'\t'`: タブ区切りを整形して表形式に変換
  - `fzf`: インタラクティブなフィルタリングツール
    - `--no-sort`: 元の順序を保持
    - `--header-lines=1`: 先頭行をヘッダーとして扱う

### [レビュー依頼されたPRにチェックアウト(`gh pr checkout`)](https://cli.github.com/manual/gh_pr_checkout)

私がよく使うスニペットです。
レビュー依頼されたPRに手軽にcheckoutすることができます。

```sh
gh pr checkout $(gh pr list --search "user-review-requested:@me" --limit 100
  --json number,title,author,state,isDraft,updatedAt,createdAt,headRefName
  --jq '
    ["no","title","author","state","draft","updatedAt","createdAt","branch"],
    ( .[] |
    [.number
    , .title[0:50]
    , .author.login
    , .state
    , (if .isDraft then "◯" else "☓" end )
    , (.updatedAt | strptime("%Y-%m-%dT%H:%M:%SZ") | strftime("%Y/%m/%d %H:%M:%S"))
    , (.createdAt | strptime("%Y-%m-%dT%H:%M:%SZ") | strftime("%Y/%m/%d %H:%M:%S"))
    , .headRefName])
    | @tsv
  '
  | column -ts $'\t'
  | fzf --no-sort --header-lines=1
  | awk '{ print $1}')
```

ポイントとしては、

- `gh pr checkout`コマンドは`gh pr checkout <pr_no>`とチェックアウトするPRNoを指定しないといけないです（他にはブランチ名など）
  PRNoを随時確認するのは大変のため、サブシェルで`$(gh pr list | fzf)`と用いることで、checkoutしたいPR一覧を表示させつつインタラクティブに選択できるようにしています。

- 内部コマンドで自分にレビュー依頼されているPR一覧を取得
  - `--search "user-review-requested:@me"`: 自分にレビュー依頼があるPRのみを対象
  - `awk '{ print $1}'`: 選択した行の最初のカラム(PR番号)を抽出

## GitHub Actionsの状況を確認

### [PRのCI状況を確認(`gh pr checks`)](https://cli.github.com/manual/gh_pr_checks)

これもよく使うコマンドで、HEADにいるブランチのPRのCI状況を監視することができます。
私の使い方としては、push後に、PRをレビュー依頼する前にCIが通っているかの確認に利用しており、コマンド実行が終了したら通知を送るようにしています。
通知には以下pluginなどが便利です。

- [zsh-auto-notify](https://github.com/MichaelAquilina/zsh-auto-notify)
- [zsh-notify](https://github.com/marzocchi/zsh-notify)

```sh
gh pr checks $(git symbolic-ref --short HEAD) --watch
```

- `--watch`オプションでコマンド終了まで監視
- `$(git symbolic-ref --short HEAD)`でHEADのブランチ名を指定

## レビュー依頼されているPRを確認

### [レビュー依頼・アサインされているPR・Issueを確認する(`gh status`)](https://cli.github.com/manual/gh_status)

意外と知られていないのが`gh status`コマンドです。
今レビュー依頼されているPRなんだっけ・・？というときに以下のコマンドを実行すると確認することができます。
手癖のように実行することで、レビュー依頼漏れなどを防ぐことができるのでおすすめです。

```sh
gh status
```

## GitHubのページをブラウザで開く

GitHub CLIを使うことで、ターミナルから直接HEADのリポジトリのページなどをブラウザで開くこともできます。
場合によってはブラウザで閲覧する方が見やすい方がある場合もあるかと思いますが、その場合もコマンド一発ですぐに開くことができます。

`gh browse`の以下は私がよく使う代表例ですが、他にもprojects・releases etc...なども指定して開くこともできます。

### [リポジトリをブラウザで開く(`gh browse`)](https://cli.github.com/manual/gh_browse)

```sh
gh browse
```

### 現在のHEADにいるブランチでリポジトリを開く

```sh
gh browse --branch $(git symbolic-ref --short HEAD)
```

### リポジトリのsettingsページをブラウザで開く

```sh
gh browse --settings
```

### 現在のHEADにいるブランチのPRをブラウザで開く

```sh
gh pr view -w
```

## まとめ

GitHub CLIを使いこなすことで、定番のGitHub作業内容を効率化できます。

最初は初期設定やスニペットの作成に慣れていないと戸惑うかもしれませんが、慣れると作業効率がとても上がると思います。

より便利なスニペット例がある場合は、コメントで教えていただけると嬉しいです！皆さんのghコマンド活用テクニックもぜひ共有してください。
（まだ他のスニペット例も全然記述できていない💦ので、私も随時更新していきたいと思います。）
