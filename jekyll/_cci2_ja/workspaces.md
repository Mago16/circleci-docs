---
layout: classic-docs
title: "ワークスペースによるジョブ間のデータ共有"
description: "ここではワークスペースを使ってデータをワークフローのダウンストリームジョブで共有する方法を説明します。"
contentTags:
  platform:
    - クラウド
    - Server v4.x
    - Server v3.x
    - Server v2.x
---

各ワークフローには `workspace` がそれぞれ関連付けられています。 ワークスペースは、ワークフローが進むにつれてデータをダウンストリームジョブに転送するために使用されます。

ワークスペースを使って、各ワークフロー固有のデータやダウンストリームジョブで必要なデータを渡すことができます。 複数のブランチでジョブを実行するようなワークフローでは、ワークスペースを使ってデータを共有する必要がある場合があります。 また、テストコンテナで使われるコンパイル済みデータを含むプロジェクトにもワークスペースは役立ちます。

たとえば、`.jar` ファイルをビルドし、それをワークスペースに保存する `build` ジョブを含むプロジェクトの場合、 その`build`ジョブは、同時に実行されるテストジョブにファンアウトします。`integration-test`、`unit-test`、および`code-coverage` は、それぞれワークスペースをアタッチすることで jar ファイルにアクセスできます。

## はじめに
{: #overview }

ワークスペースは、追加のみが可能なデータストレージです。 ジョブはワークスペースに永続的にデータを保管することができます。 ワークスペースを使用すると、データはコンテナの外のストアにアーカイブされ保存されます。 With each addition to the workspace, a new layer is created in the store. ダウンストリームジョブは、コンテナのファイルシステムにそのワークスペースをアタッチできます。

ワークスペースをアタッチすると、ワークフロー内のアップストリームジョブの順序に基づいて、各レイヤーがダウンロードされ、アンパッケージ化されます。

ワークスペースに関する注意事項

* 各ワークフローには、それぞれに一時的なワークスペースが関連付けられています。 ワークスペースは、ジョブの実行中にビルドした固有のデータを、同じワークフローの他のジョブに渡すために使用されます。
* ジョブ内では、`persist_to_workspace` ステップを使用してワークスペースにファイルを追加でき、`attach_workspace` ステップを使用してワークスペースの内容をファイル システムにダウンロードできます。
* The workspace is additive only: jobs may add files to the workspace but cannot delete files from the workspace.
* 各ジョブでは、そのアップストリームのジョブによってワークスペースに追加された内容を参照することのみ可能です。
* ワークスペースをアタッチすると、アップストリームジョブがワークフローグラフに表示される順番で、各アップストリームジョブからの「レイヤー」が適用されます。 しかし、2 つのジョブを同時に実行すると、そのレイヤーの適用順序は確定できません。
* If multiple concurrent jobs persist, the same filename then attaching the workspace will error.
* If a workflow is re-run, it inherits the same workspace as the original workflow. 失敗したジョブを再度実行したときも、そのジョブは元のワークフローで実行したジョブと同じワークスペースの内容を使えることになります。

デフォルトのワークスペースの保存期間は 15 日間です。 保存期間は、[CircleCI Web アプリ](https://app.circleci.com/)の **Plan > Usage Controls** からカスタマイズ可能です。 現在、設定できる保存期間の最大値が 15 日間となっています。

![ワークスペースのデータ フロー](/docs/assets/img/docs/workspaces.png)

## ワークスペースの設定
{: #workspace-configuration }

ジョブで作られたデータを保存して他のジョブでも使えるよう、`persist_to_workspace` キーをジョブに追加します。 `persist_to_workspace` の `paths:` プロパティに記述されたディレクトリ・ファイルは、`root` キーで指定しているディレクトリの相対パスとなる一時 Workspace にアップロードされます。 その後、ファイルとディレクトリはアップロードされ、続くジョブで (および Workflow の再実行時に) 利用できるようにします。

カスタマイズされたストレージ設定の場合、ワークスペースのカスタマイズ設定に `persist_to_workspace` がデフォルトで設定されます。 カスタマイズ設定がない場合は、`persist_to_workspace` が 15 日間デフォルト設定となります。

`attach_workspace` キーをセットして、保存されたデータを取得できるようにします。 The following `.circleci/config.yml` file defines two jobs where the `downstream` job uses the artifact of the `flow` job. Workflow はシーケンシャルのため、`downstream` ジョブの処理がスタートする前に `flow` ジョブが終了していなければなりません。

{:.tab.workspaces.Cloud}
```yaml
version: 2.1

executors:
  my-executor:
    docker:
      - image: buildpack-deps:jessie
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    working_directory: /tmp

jobs:
  flow:
    executor: my-executor
    steps:
      - run: mkdir -p workspace
      - run: echo "Hello, world!" > workspace/echo-output

      # Persist the specified paths (workspace/echo-output) into the workspace for use in downstream job.
      - persist_to_workspace:
          # Must be an absolute path, or relative path from working_directory. This is a directory on the container which is
          # taken to be the root directory of the workspace.
          root: workspace
          # Must be relative path from root
          paths:
            - echo-output

  downstream:
    executor: my-executor
    steps:
      - attach_workspace:
          # Must be absolute path or relative path from working_directory
          at: /tmp/workspace

      - run: |
          if [[ `cat /tmp/workspace/echo-output` == "Hello, world!" ]]; then
            echo "It worked!";
          else
            echo "Nope!"; exit 1
          fi

workflows:
  btd:
    jobs:
      - flow
      - downstream:
          requires:
            - flow
```

{:.tab.workspaces.Server_3}
```yaml
version: 2.1

executors:
  my-executor:
    docker:
      - image: buildpack-deps:jessie
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    working_directory: /tmp

jobs:
  flow:
    executor: my-executor
    steps:
      - run: mkdir -p workspace
      - run: echo "Hello, world!" > workspace/echo-output

      # Persist the specified paths (workspace/echo-output) into the workspace for use in downstream job.
      - persist_to_workspace:
          # Must be an absolute path, or relative path from working_directory. This is a directory on the container which is
          # taken to be the root directory of the workspace.
          root: workspace
          # Must be relative path from root
          paths:
            - echo-output

  downstream:
    executor: my-executor
    steps:
      - attach_workspace:
          # Must be absolute path or relative path from working_directory
          at: /tmp/workspace

      - run: |
          if [[ `cat /tmp/workspace/echo-output` == "Hello, world!" ]]; then
            echo "It worked!";
          else
            echo "Nope!"; exit 1
          fi

workflows:
  btd:
    jobs:
      - flow
      - downstream:
          requires:
            - flow
```

{:.tab.workspaces.Server_2}
```yaml
version: 2

executors:
  my-executor:
    docker:
      - image: buildpack-deps:jessie
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    working_directory: /tmp

jobs:
  flow:
    executor: my-executor
    steps:
      - run: mkdir -p workspace
      - run: echo "Hello, world!" > workspace/echo-output

      # Persist the specified paths (workspace/echo-output) into the workspace for use in downstream job.
      - persist_to_workspace:
          # Must be an absolute path, or relative path from working_directory. This is a directory on the container which is
          # taken to be the root directory of the workspace.
          root: workspace
          # Must be relative path from root
          paths:
            - echo-output

  downstream:
    executor: my-executor
    steps:
      - attach_workspace:
          # Must be absolute path or relative path from working_directory
          at: /tmp/workspace

      - run: |
          if [[ `cat /tmp/workspace/echo-output` == "Hello, world!" ]]; then
            echo "It worked!";
          else
            echo "Nope!"; exit 1
          fi

workflows:
  btd:
    jobs:
      - flow
      - downstream:
          requires:
            - flow
```

For a live example of using workspaces to pass data between build and deploy jobs, see the [`.circleci/config.yml`](https://github.com/circleci/circleci-docs/blob/master/.circleci/config.yml) that is configured to build the CircleCI documentation.

ワークスペース、キャッシュ、およびアーティファクトの使用に関する概念的な情報については、ブログ記事「[ワークフローでデータを保持するには: キャッシュ、アーティファクト、ワークスペース活用のヒント](https://circleci.com/blog/persisting-data-in-workflows-when-to-use-caching-artifacts-and-workspaces/)」を参照してください。

## ワークスペースストレージのカスタマイズ
{: #workspaces-and-self-hosted-runner }

セルフホストランナーを使用する場合、プランに含まれるネットワークとストレージ使用量には制限があります。 ワークスペースに関連するアクションには、ネットワークとストレージの使用が発生するものがあります。 お客様の使用量が制限を超えた場合、料金が発生します。

ワークスペースを長期間保存すると、ストレージコストに影響が及ぶため、アーティファクトを保存する理由を明確にすることをお勧めします。 多くのプロジェクトでは、ワークスペースを保存する利点は、失敗したビルドの再実行ができることです。 ビルドが成功したら、そのワークスペースは不要になります。 ニーズに合う場合は、ワークスペースのストレージ保存期間を短く設定することを推奨します。

[CircleCI Web アプリ](https://app.circleci.com/)で **Plan > Usage Controls** に移動し、ワークスペースのストレージ使用量や保存期間をカスタマイズすることができます。 For information on managing network and storage usage, see the [Persisting Data](/docs/persist-data/#managing-network-and-storage-use) page.

## ワークスペースの最適化
{: #workspace-usage-optimization }

`persist_to_workspace` を使用する際は、パスとファイルを定義することが重要です。 定義しないと、ストレージが大幅に増加する場合があります。 パスとファイルは以下の構文を使って指定します。

```yml
- persist_to_workspace:
    root: /tmp/dir
    paths:
      - foo/bar
      - baz
```

## 関連項目
{: #see-also }

- For conceptual and usage information on Workflows, see the [Using Workflows to Orchestrate Jobs](/docs/workflows/) page.
- [データの永続化](/docs/persist-data/)
- [依存関係のキャッシュ](/docs/caching/)
- [キャッシュ戦略](/docs/caching-strategy/)
- [アーティファクト](/docs/artifacts/)
- [最適化の概要](/docs/optimizations/)
