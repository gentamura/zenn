---
title: "reviewdogとはじめるGitHub ActionsとESLint"
emoji: "🐶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['reviewdog', 'eslint', 'githubactions']
published: false
---
reviewdog🐕とたくさん遊ぶ機会があったので、まとめてみました。

# メモ

- eslint のルール変更で、eslint がエラーに引っかかる状況であっても、reviewdog では指摘されない
  - なぜなら file 自体に変更点がないから

- `-level` フラグは `github-pr-review` では無視される
  - `warnings` と `error` の両方で、レビューコメントが反映する
  - `-level` フラグは `github-check` と `github-pr-check` のとき有効

- `github-check`、`github-pr-check` はレビューコメントができない。check 対象がある場合は、actions のタブで anotations で表示される
- `github-pr-review` では、pr に含まれる commit のみレビューコメントの対象になる。


# reviewdog利用シナリオ

- ESLint で JS ファイルをチェックし、GitHub の PR 上で、レビューコメントをもらう

## GitHub Actionsについて

### イベント

`on` で指定する。

#### pull_request
https://docs.github.com/en/actions/reference/events-that-trigger-workflows#pull_request
以下のタイミングで CI が実行される。

- [opened]Pull Request を作成したタイミング
- [synchronize]PR に Push したタイミング
- [reopened]Pull Request を再オープンしたタイミング


#### push

commit を push したタイミングで実行される。

# reviewdogの機能抜粋

## Reporter
GitHub 関連の reporter は、以下の通り。

1. Reporter: github-check
1. Reporter: github-pr-check
1. Reporter: github-pr-review


**reporterごとの動作対象一覧**

| `対象のcommit sha` / `-reporter` | **`github-check`** | **`github-pr-check`** |**`github-pr-review`**  |
| - | - | - | - |
| Commit（PRに含まれないCommit） | OK | ❌ | ❌ |
| Pull Request（PRに含まれるCommit） | OK | OK | OK |


**CommitにLinter対象ファイルが含まれているかどうかの動作一覧**

| `Commitのファイル` / `-reporter` | **`github-check`** | **`github-pr-check`** |**`github-pr-review`**  |
| - | - | - | - |
| [Commit] Linter対象ファイルが存在する | すべて指摘 | ❌ | ❌ |
| [Commit] Linter対象ファイルが存在しない | すべて指摘 | ❌ | ❌ |
| [Pull Request] Linter対象ファイルが存在する | すべて指摘 | [1] | [1] |
| [Pull Request] Linter対象ファイルが存在しない | すべて指摘 | ❌ | ❌ |
- [1]commit に含まれているファイルのみ指摘

### 1. Reporter: github-check
linter の実行結果にあわせ、GitHub 上で anotation してくれる。

対象は以下の通り。

- Commit
- Pull request

#### 実例
##### GitHub Actions

###### fail_on_error: true
check は実行され、新規・既存エラーすべてが、`Findings` に補足され、error になる。



###### fail_on_error: false (default)
check は実行され、既存のエラーがあれば、`Filtered findings` の中で補足される。default の filter の `added` に合致しないため、`Findings` には補足されず、error にもならない。

=> error にならず、CI が止まらないのは正しいが、`Findings` と `Filtered findings` に入る条件がまだちょっと違うかも。うーん。

![](./img/github.com_gentamura_hello-reviewdog_actions_runs_868377581.png)

##### reviewdog CLI
reviewdog を cli で利用するためには、reviewdog の token を設定する必要がある。

1. token を取得するためには、GitHub App に reviewdog を追加し、専用ページから token を取得する。

- [GitHub App - reviewdog](https://github.com/apps/reviewdog)
- [gentamura/hello-reviewdog - reviewdog](https://reviewdog.app/gh/gentamura/hello-reviewdog)

2. `REVIEWDOG_TOKEN` に加え、必要な環境変数を追加する。

```
CI_REPO_OWNER=gentamura
CI_REPO_NAME=hello-reviewdog
CI_COMMIT=***
REVIEWDOG_TOKEN=***
```

> CI_COMMITはregular sha-1 hashでshort sha-1 hashはNG。short hashのときは、以下のエラーがでます。

```
reviewdog: POST https://api.github.com/repos/gentamura/hello-reviewdog/pulls/12/reviews: 422 Unprocessable Entity [{Resource: Field: Code: Message:Variable $commitOID of type GitObjectID was provided invalid value}]
```

3. 以下のように実行する

###### [error] REVIEWDOG_TOKENが設定されていない場合のエラー

```
hello-reviewdog on  main [!?] is 📦 v1.0.0 via ⬢ v15.0.1
❯ npm run eslint -- -f checkstyle . | reviewdog -f=checkstyle -name="eslint" -reporter=github-check
reviewdog: post failed for eslint: status=401: The access token not provided. Get token from https://reviewdog.app/gh/gentamura/hello-reviewdog

```

###### [success] format: checkstyleにてsuccess

```
hello-reviewdog on  main [!?] is 📦 v1.0.0 via ⬢ v15.0.1
❯ npm run eslint -- -f checkstyle . | reviewdog -f=checkstyle -name="eslint" -reporter=github-check
2021/05/21 20:21:47 [eslint] reported: https://github.com/gentamura/hello-reviewdog/runs/2638782339 (conclusion=failure)
```
> **Info:** format: checkstyleのときは、npmコマンドで実行してもOK。

https://github.com/gentamura/hello-reviewdog/runs/2638782339 にアクセスすると、以下のような表示になる。

![](./img/github.com_gentamura_hello-reviewdog_runs_1.png)
###### [success] format: rdjsonにてsuccess

```
❯ $(npm bin)/eslint src/ -f rdjson | reviewdog -f=rdjson -reporter=github-check
{"ruleId":"no-constant-condition","severity":2,"message":"Unexpected constant condition.","line":10,"column":7,"nodeType":"Literal","messageId":"unexpected","endLine":10,"endColumn":11}
```

![](./img/github.com_gentamura_hello-reviewdog_runs_2.png)

> **Warnings:** 以下のようにeslintをnpmを経由して実行すると、rdjson以外の文字列が表示されるため、reviewdogのparseが失敗するので注意が必要。

```
hello-reviewdog on  add-bar [!?] is 📦 v1.0.0 via ⬢ v15.0.1
❯ npm run eslint -- -f rdjson | reviewdog -f=rdjson -reporter=github-checkt
reviewdog: parse error: failed to unmarshal rdjson (DiagnosticResult): proto: syntax error (line 2:1): invalid value >
```

上記のエラーは、

```
hello-reviewdog on  add-bar is 📦 v1.0.0 via ⬢ v15.0.1
❯ npm run eslint -- -f rdjson

> hello-reviewdog@1.0.0 eslint
> eslint src/ "-f" "rdjson"

{"source":{"name":"eslint","url":"https://eslint.org/"},"diagnostics":[...]}
```
のように、npm run を経由することにより、

```
> hello-reviewdog@1.0.0 eslint
> eslint src/ "-f" "rdjson"
```

のように、余計なデータが出力されて、reviewdog に渡り、その値を parse していることが原因。

次のように、余計な log がでないように直接 eslint を叩いてあげると OK。

```
hello-reviewdog on  add-bar is 📦 v1.0.0 via ⬢ v15.0.1
❯ $(npm bin)/eslint src/ -f rdjson
{"source":{"name":"eslint","url":"https://eslint.org/"},"diagnostics":[...]}
```

### 2. Reporter: github-pr-check
linter の実行結果にあわせ、GitHub 上で anotation してくれる。

対象は以下の通り。

- Pull request

※ PR ではなく、Commit の sha を指定して、実行すると以下のエラーが発生。

```
hello-reviewdog on  add-bar [!] is 📦 v1.0.0 via ⬢ v15.0.1
❯ $(npm bin)/eslint src/ -f rdjson | reviewdog -f=rdjson -reporter=github-pr-check -guess
reviewdog: PullRequest not found, query: type:pr state:open repo:gentamura/hello-reviewdog head:add-bar babf561967664dc8a2fd492d07781b68eea4aa6e
reviewdog: this is not PullRequest build.
```
##### GitHub Actions

- check は実行され、既存のエラーがあれば、`Filtered findings` の中で補足される。default の filter の `added` に合致しないため、`Findings` には補足されず、error にもならない。
- `on: push` の場合は、`reviewdog: this is not PullRequest build.` でエラーとなり、正常に動作しない。

##### reviewdog CLI


### 3. Reporter: github-pr-review

linter の実行結果にあわせ、GitHub 上でレビューコメントしてくれる。

対象は以下の通り。

- Pull request

※ PR ではなく、Commit の sha を指定して、実行すると以下のエラーが発生。

```
hello-reviewdog on  add-bar [!] is 📦 v1.0.0 via ⬢ v15.0.1
❯ $(npm bin)/eslint src/ -f rdjson | reviewdog -f=rdjson -reporter=github-pr-review -guess
reviewdog: PullRequest not found, query: type:pr state:open repo:gentamura/hello-reviewdog head:add-bar babf561967664dc8a2fd492d07781b68eea4aa6e
reviewdog: this is not PullRequest build.
```

##### GitHub Actions

- `on: push` の場合は、`reviewdog: this is not PullRequest build.` でエラーとなり、正常に動作しない。

###### レビューコメントが追加される条件

- Linter でエラーが出たファイルを新規追加したとき
- Linter でエラーが出ている既存ファイルを更新した

**filter added**

- 追加項目がなければ、skip。設定されている filter に関わらず、added の条件に当たらなければ、なにも反応がなく、GitHub Actions のコンソールにも何も表示されない。

**filter nofilter**

GitHub Actions のコンソールに、ESLint の検査結果がすべて表示される。
そのため、fail_on_error が true であれば、CI が失敗になる。
added の条件にあたるファイルがなければ、Review コメントは表示されない。

##### reviewdog CLI

指定する commit hash は、PR 内に含まれていれば OK。
指定した commit 以外の commit も Linter の対象になっていれば、レビューコメントが追加される。

###### error

-guess を渡さないとエラー。

```
hello-reviewdog on  add-baz is 📦 v1.0.0 via ⬢ v15.0.1
❯ npm run eslint -- -f checkstyle . | reviewdog -f=checkstyle -name="eslint" -reporter=github-pr-review
reviewdog: this is not PullRequest build.
```

```
hello-reviewdog on  add-bar is 📦 v1.0.0 via ⬢ v15.0.1
❯ $(npm bin)/eslint src/ -f rdjson | reviewdog -f=rdjson -reporter=github-pr-review
reviewdog: this is not PullRequest build.
```

###### success

Token 発行者のアカウントで review コメントが入る。

```
hello-reviewdog on  add-baz is 📦 v1.0.0 via ⬢ v15.0.1
❯ npm run eslint -- -f checkstyle . | reviewdog -f=checkstyle -name="eslint" -reporter=github-pr-review -guess
/Users/gentamura/workspece/project/hello-reviewdog/src/index_7.js:2:7: error: Unexpected constant condition. (no-constant-condition) (eslint.rules.no-constant-condition)
/Users/gentamura/workspece/project/hello-reviewdog/src/index_7.js:6:7: error: Unexpected constant condition. (no-constant-condition) (eslint.rules.no-constant-condition)
/Users/gentamura/workspece/project/hello-reviewdog/src/index_7.js:10:7: error: Unexpected constant condition. (no-constant-condition) (eslint.rules.no-constant-condition)
```
![](./img/github.com_gentamura_hello-reviewdog_pull_13.png)


```
hello-reviewdog on  add-baz is 📦 v1.0.0 via ⬢ v15.0.1
❯ $(npm bin)/eslint src/ -f rdjson | reviewdog -f=rdjson -reporter=github-pr-review -guess
{"ruleId":"no-constant-condition","severity":2,"message":"Unexpected constant condition.","line":2,"column":7,"nodeType":"Literal","messageId":"unexpected","endLine":2,"endColumn":11}
{"ruleId":"no-constant-condition","severity":2,"message":"Unexpected constant condition.","line":6,"column":7,"nodeType":"Literal","messageId":"unexpected","endLine":6,"endColumn":11}
{"ruleId":"no-constant-condition","severity":2,"message":"Unexpected constant condition.","line":10,"column":7,"nodeType":"Literal","messageId":"unexpected","endLine":10,"endColumn":11}
```
![](./img/github.com_gentamura_hello-reviewdog_pull_14.png)
