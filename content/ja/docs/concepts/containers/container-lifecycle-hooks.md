---
title: コンテナ・ライフサイクル・フック
content_template: templates/concept
weight: 30
---

{{% capture overview %}}
このページでは、コンテナの管理ライフサイクルにおいて、コードの実行トリガとなるイベント向けのコンテナ・ライフサイクル・フックを、kubelet がどのようにしてコンテナを管理しているか説明します。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## 概要 {#overview}

Angular のような多くのプログラミング言語フレームワークがコンポーネントのライフサイクル・フックを持っているのと同様に、Kubernetes はコンテナのライフサイクル・フック（hooks）を提供します。
フックにより、管理ライフサイクル上のイベントをコンテナが検出できるようにします。
また、適切なライフサイクル・フックが処理されると、コードを実行できるように実装されてます。

## コンテナ・フック {#container-hooks}

コンテナが外向きに公開するフックは２種類あります：

`PostStart`

このフックはコンテナの作成後、ただちに実行されます。しかし、コンテナの ENTRYPOINT に先立つ実行を保証するものではありません。ハンドラに対するパラメータは指定できません。

`PreStop`

このフックはコンテナの停止後、ただちに実行されます。これはブロッキング、つまり、同期処理のため、コンテナ削除のコール前（命令）に処理を終えた後での送信となります。ハンドラに対するパラメータは指定できません。

停止時の挙動の詳細については [ポッドの停止](/jp/docs/concepts/workloads/pods/pod/#termination-of-pods) に説明があります。

### フックを取り扱う実装 {#hook-handler-implementations}

コンテナがフックにアクセスできるのは、実装されている場合か、フックに対するハンドラが登録されている場合です。コンテナに対して実装可能なフック・ハンドラ（hook handler）は２種類です。

* exec（実行型） - コンテナの cgroup と名前空間の中で `pre-stop.sh` のような特定のコマンドを処理します。コマンドによって消費されるリソースは、コンテナに対するリソースに含まれるものとカウントされます。
# HTTP - コンテナ上にある特定のエンドポイントに対する HTTP リクエストを処理します。

### フックが扱う処理 {#hook-handler-execution}

コンテナ・ライフサイクル・マネジメント・フックが呼び出される時、Kubernetes 管理システムは対象となるフックに対して、コンテナに登録済みのハンドラを実行します。

フック・ハンドラは、コンテナを含むポッドの処理と同期して、ただちに実行されます。
つまり、 `PostStart` （訳者注：スタート前）フックとは、コンテナの ENTRYPOINT 実行とフック処理とが非同期です。
しかしながら、フック処理に時間がかかるか固まれば（ハングしたら）コンテナは `running` （実行中）の状態に到達できません。

この挙動は `PreStop` （訳者注：停止前）フックと似ています。もし実行中にフックの応答がなくなれば、Pod は `Terminating` 状態のままとなり、ポッドを終了させる `terminationGracePeriodSeconds` が経過した後に停止（kill）されます。
もし `PostStart` や `PreStop`  のフック処理に失敗すると、コンテナは停止（kill）させられます。

ユーザは可能な限りフック・ハンドラを軽量にすべきでしょう。
しかしながら、コンテナを停止する前に状態を保存したい時など、コマンド処理が長期間にわたるかどうかは、状況によります。

### フック・デリバリ（到達性）の保証 {#hook-delivery-guarantees}

フック・デリバリは *最低１回* の実行を意図しています。
つまり、 `PostStart` や `PreStop` のような指定されるイベントによって、フックは複数回呼び出される場合があります。
この処理が正確に行われるかどうかは、フックの実装次第です。

通常、１つの処理（デリバリ）のみが作成されます。たとえば、 HTTP フック・レシーバ（hook receiver）がダウンを検出してトラフィックを扱えなくなれば、再送は一切試みません。
他の稀な事例としては、２重に送信される場合も有り得ます。
たとえば、 kubelet がフック送信中に再起動した場合、kubelet が戻ってくるまでにフックが再送されるかもしれません。

### フック・ハンドラのデバッグ {#debugging-hook-handlers}

フック・ハンドラ用のログは Pod イベント内では表示されません。
もしも何らかの理由でハンドラに失敗した場合、イベントをブロードキャストします。
`PostStart` であれば `FailedPostStartHook` イベントを、 `PreStop` であれば `FailedPreStopHook` イベントです。
これらのイベントを表示するには `kubectl describe pod <ポッド名>` を実行します。
以下はコマンド実行後のイベント表示例です。

```
Events:
  FirstSeen    LastSeen    Count    From                            SubobjectPath        Type        Reason        Message
  ---------    --------    -----    ----                            -------------        --------    ------        -------
  1m        1m        1    {default-scheduler }                                Normal        Scheduled    Successfully assigned test-1730497541-cq1d2 to gke-test-cluster-default-pool-a07e5d30-siqd
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Pulling        pulling image "test:1.0"
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Created        Created container with docker id 5c6a256a2567; Security:[seccomp=unconfined]
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Pulled        Successfully pulled image "test:1.0"
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Started        Started container with docker id 5c6a256a2567
  38s        38s        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Killing        Killing container with docker id 5c6a256a2567: PostStart handler: Error executing in Docker Container: 1
  37s        37s        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Killing        Killing container with docker id 8df9fdfd7054: PostStart handler: Error executing in Docker Container: 1
  38s        37s        2    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}                Warning        FailedSync    Error syncing pod, skipping: failed to "StartContainer" for "main" with RunContainerError: "PostStart handler: Error executing in Docker Container: 1"
  1m         22s         2     {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Warning        FailedPostStartHook
```

{{% /capture %}}

{{% capture whatsnext %}}

* [コンテナ環境変数](/docs/concepts/containers/container-environment-variables/) について学ぶ。
* [コンテナ・ライフサイクル・イベントを取り扱う](/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/) ハンズオン練習。

{{% /capture %}}


