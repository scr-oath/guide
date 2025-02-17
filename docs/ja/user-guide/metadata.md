---
layout: main
title: Metadata
category: User Guide
menu: menu_ja
toc:
- title: Metadata
  url: "#metadata"
- title: Metadataとは?
  url: "#Metadataとは"
- title: デフォルトMetadata
  url: "#デフォルトMetadata"
- title: Metadataの操作
  url: "#Metadataの操作"
- title: 同一パイプライン
  url: "#同一パイプライン"
  subitem: true
- title: 外部パイプライン
  url: "#外部パイプライン"
  subitem: true
- title: プルリクエストコメント
  url: "#プルリクエストコメント"
  subitem: true
- title: プルリクエストチェック
  url: "#追加のプルリクエストチェック"
  subitem: true
- title: カバレッジとテスト結果
  url: "#カバレッジとテスト結果"
  subitem: true
- title: イベントラベル
  url: "#イベントラベル"
  subitem: true
- title: Slack通知
  url: "#slack通知"
  subitem: true
- title: ジョブベースのSlackメッセージ
  url: "#ジョブベースのslackメッセージ"
  subitem: level-2
- title: ジョブベースのSlackチャンネル
  url: "#ジョブベースのslackチャンネル"
  subitem: level-2
- title: ジョブベースで通知を最小化する設定
  url: "#ジョブベースで通知を最小化する設定"
  subitem: level-2      
---

# Metadata

## Metadataとは？

Metadataは [ビルド](../../about/appendix/domain#build) に関する情報を保持する key/value ストアです。Metadataは [steps](../../about/appendix/domain#step) 内で組み込まれている [meta CLI](https://github.com/screwdriver-cd/meta-cli) を利用することで、全てのビルドで更新と取得が可能です。

## デフォルトMetadata

Screwdriver.cdはデフォルトでMetadataに以下のキーを設定しています。

| キー | 説明 |
| --- | ----------- |
| build.buildId | ビルドのID |
| build.jobId | ビルドと紐付いているジョブのID |
| build.eventId | ビルドと紐付いているイベントのID |
| build.pipelineId | ビルドと紐付いているパイプラインのID |
| build.sha | ビルドが実行しているコミットのsha |
| build.jobName | ジョブ名 |
| commit.author | `avatar`, `name`, `url`, `username`を含むAuthor情報のオブジェクト |
| commit.committer | `avatar`, `name`, `url`, `username`を含むCommitter情報のオブジェクト |
| commit.message | コミットメッセージ |
| commit.url | コミットへのURL |
| commit.changedFiles | カンマ区切りの変更ファイルリスト<br>**注意**: UIから新たにイベントを開始した場合はコミットでトリガーされたことにならないので、この値は空になります |
| sd.tag.name | タグ名 |
| sd.release.id | リリースID |
| sd.release.name | リリース名 |
| sd.release.author | リリース |

## Metadataの操作

Screwdriver は meta store から情報を取得するためのシェルコマンド `meta get` と、meta store に情報を保存するためのシェルコマンド `meta set` を提供しています。

### 同一パイプライン

Screwdriverのビルドでは、同ビルドでセットされたMetadata、もしくは同イベントの以前のビルドでセットされたMetadataを取得することができます。

例: `build1` -> `build2` -> `build3`

`build2` のMetadataは、自身でセットしたMetadataと `build1` でセットしたMetadataを保持しています。

`build3` のMetadataは、 `build2` が持っていたMetadataを保持しています。 ( `build1` のMetadataも含む)

```bash
$ meta set example.coverage 99.95
$ meta get example.coverage
99.95
$ meta get example
{"coverage":99.95}
```

例:

```bash
$ meta set foo[2].bar[1] baz
$ meta get foo
[null,null,{"bar":[null,"baz"]}]
```

サンプルリポジトリ: <https://github.com/screwdriver-cd-test/workflow-metadata-example>

注意:

- `foo`がセットされていない場合に`meta get foo`を実行した場合、デフォルトで文字列の`null`を返します。

### 外部パイプライン

Screwdriverのビルドは外部トリガー元のジョブのMetadataにも `--external` フラグにトリガー元のジョブを指定することでアクセスすることができます。

例: `sd@123:publish` -> `build1` の時 `build1` のビルド内で:

```
$ meta get example --external sd@123:publish
{"coverage":99.95}
```

注意:

- `meta set` は外部パイプラインのジョブに対してはできません。
- もしビルドのトリガー元のジョブが `--external` で指定した外部パイプラインのジョブではなかった場合、meta はセットされません。
- `--external`で指定したパイプラインのジョブが、そのビルドをトリガーしていなかった場合は、最後に成功した外部ジョブの`meta`が取得されます。

### APIを使用する

`/v4/events`への`POST`リクエストのpayloadに設定することでも、イベントメタを設定することができます。

APIのエンドポイントについての詳細は、[APIのドキュメント](./api)を参照してください。

[イベントメタトリガーのサンプルリポジトリ](https://github.com/screwdriver-cd-test/event-meta-trigger-example)や、それに対応した[イベントメタのサンプルリポジトリ](https://github.com/screwdriver-cd-test/event-meta-example)も参考にしてください。

### プルリクエストコメント

> 注意：この機能は現在のところGithubプラグインでのみ使用可能です

Metadataを使用することで、ScrewdriverのビルドからGitのプルリクエストにコメントを書き込むことができます。コメントの内容はパイプラインのPRビルドから、metaのsummaryオブジェクトに書き込まれます。

プルリクエストにMetadataを書き出すには、`meta.summary`に必要な情報をセットするだけです。このデータはheadlessなGitのユーザからのコメントとして出てきます。

例として、カバレッジの説明を加えたい場合には、screwdriver.yamlは以下のようになります。

```yaml
jobs:
  main:
    steps:
      - comment: meta set meta.summary.coverage "Coverage increased by 15%"
```

以下の例のようにMarkdown記法で書くこともできます。

```yaml
jobs:
  main:
    steps:
      - comment: meta set meta.summary.markdown "this markdown comment is **bold** and *italic*"
```
これらの設定をすると、以下のようにGitにコメントがされます。

![PR comment](./../../user-guide/assets/pr-comment.png)

_注意: ビルドを複数実行すると、Gitの同じコメントを編集します。_

### 追加のプルリクエストチェック

> 注意：この機能は現在のところGithubプラグインでのみ使用可能です

プルリクエストビルドのより詳細な状態を知るために追加のステータスチェックをプルリクエストに加えることができます。

プルリクエストに追加のチェックをするには、`meta.status.<check>`にJSON形式で必要な情報を設定するだけでできます。このデータはGitのプルリクエストのチェックとして出てきます。

設定できるフィールドは以下の通りです。

| Key | Description |
| --- | ----------- |
| status (String) | チェックのステータス。次から一つ選びます (`SUCCESS`, `FAILURE`) |
| message (String) | チェックの説明 |
| url (String) | チェックのリンクのURL（デフォルトはビルドのリンク） |

例として、`findbugs`と`coverage`の2つの追加のチェックを加える場合、screwdriver.yamlは次のようになります。

```yaml
jobs:
  main:
    steps:
      - status: |
          meta set meta.status.findbugs '{"status":"FAILURE","message":"923 issues found. Previous count: 914 issues.","url":"http://findbugs.com"}'
          meta set meta.status.coverage '{"status":"SUCCESS","message":"Coverage is above 80%."}'
```

これらの設定は以下のようにGitのチェックになります。

![PR checks](./../../user-guide/assets/pr-checks.png)

### カバレッジとテスト結果

metadataを利用して、Screwdriverのビルドからビルドページにカバレッジの結果やテスト結果をそれらの生成物へのURLと一緒に取り込むことができます。ScrewdriverのUIはmetadata内の`tests.coverage`、 `tests.results`、 `tests.coverageUrl`、 `tests.resultsUrl`を読み込んで対応するものを表示させたり設定したりします。

screwdriver.yamlの例:

```yaml
jobs:
  main:
    steps:
      - set-coverage-and-test-results: |
          meta set tests.coverage 100 # カバレッジパーセンテージ数
          meta set tests.results 10/10 # 成功テスト数/全テスト数
          meta set tests.coverageUrl /test/coverageReport.html # ビルド成果物への相対パスで指定します
          meta set tests.resultsUrl /test/testReport.html # ビルド成果物への相対パスで指定します
```

> 注意: metadataはSonarQubeの結果を上書きします。

これらの設定により、ビルドページは次のようになります:

![coverage-meta](./../../user-guide/assets/coverage-meta.png)

### イベントラベル

metaのキーに`label`を指定するとイベントにラベルを付与することができます。このキーは[ロールバック](./FAQ.html#ビルドのロールバックを行うには？)するイベントの指定に役立ちます。

screwdriver.yamlの例:
```yaml
jobs:
  main:
    steps:
      - set-label: |
          meta set label VERSION_3.0 # 設定した値はイベントに紐づいてパイプラインページ上に表示されます
```

結果:
![Label](./../../user-guide/assets/label-meta.png)

### Slack通知

metaを利用することで[通知](./configuration/settings.html#slack)メッセージをカスタマイズすることができます。metaのキーは通知ブラグインごとに異なります。

#### 基本
Slack通知をするscrewdriver.yamlの例:
```yaml
jobs:
  main:
    steps:
      - meta: |
          meta set notification.slack.message "<@yoshwata> Hello Meta!"
```

Result:
![notification-meta](./../../user-guide/assets/notification-meta.png)

#### ジョブベースのSlackメッセージ

*注意* ジョブベースのSlack通知のメタデータは基本的な通知メッセージを上書きします。

メタ変数の構造は、`notification.slack.<jobname>.message`です。  
`<jobname>`をScrewdriver.cdのジョブ名に置き換えます。

特定のジョブをSlackメッセージで通知する例:
```yaml
jobs:
  main:
    steps:
      - meta: |
          meta set notification.slack.slack-notification-test.message "<@yoshwata> Hello Meta!"
```

Result:
![notification-meta](./../../user-guide/assets/notification-meta.png)

#### ジョブベースのSlackチャンネル
*注意*: ジョブベースのSlackチャンネルのメタデータは基本的なSlack通知チャンネルを上書きするだけです。[チャンネル通知](./configuration/settings#slack)設定の代わりになるものではありません。

メタ変数の構造は、`notification.slack.<jobName>.channels`です。
`<jobname>`をScrewdriver.cdのジョブ名に置き換えます。

コンマ区切りの文字列にすることで、複数のチャンネルを設定することができます。

`component`ジョブで失敗した時に別のSlackチャンネルへ通知するscrewdriver.yamlの例:
```yaml
shared:
    image: docker.ouroath.com:4443/x/y/z
    
    settings:
        slack:
            channels: [ main_channel ]
            statuses: [ FAILURE ]
jobs:
   component:
    steps:
      - meta: |
          meta set notification.slack.component.channels "fail_channel, prod_channel"
```
上記の例では、Slackの失敗通知のメッセージは`main_channel`へ送られる代わりに`fail_channel`と、`prod_channel`へ送られます。パイプラインの他のすべてのジョブは、`main_channel`へ通知します。

#### ジョブベースで通知を最小化する設定
ジョブベースのSlackの `minimized` メタの設定はデフォルトのSlack minimized設定を上書きします。

メタ変数の構造は、 `notification.slack.<jobName>.minimized` です。 `<jobName>` をScrewdriverのジョブ名に置き換えてください。

例えば、 `component` ジョブがスケジューラーによってトリガーされた場合に、最小化したSlackメッセージを投稿します:

```yaml
shared:
    image: docker.ouroath.com:4443/x/y/z
    
    settings:
        slack:
            channels: [ main_channel ]
            statuses: [ FAILURE ]
            minimized: false
jobs:
   component:
    steps:
      - meta: |
          if [[ $SD_SCHEDULED_BUILD == true ]]; then
             meta set notification.slack.component.minimized true
          fi
```
上記の例では、スケジューラーによってトリガーされた `component` ジョブでのSlack通知は `minimized` 形式で投稿されます。

