---
title: ポッドのライフサイクル
content_template: templates/concept
weight: 30
---

{{% capture overview %}}

{{< comment >}}Updated: 4/14/2015{{< /comment >}}
{{< comment >}}Edited and moved to Concepts section: 2/2/17{{< /comment >}}

このページはコンテナのライフサイクルについて説明します。

{{% /capture %}}


{{% capture body %}}

## ポッドの段階 {#pod-phase}

ポッドの `status` （状態）フィールドは [PodStatus（ポッド・ステータス）](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podstatus-v1-core) オブジェクトであり、 `phase` （段階：フェーズ）フィールドがあります。

ポッドの段階（phase）とはシンプルであり、ポッドのライフサイクルにおいてはハイレベルな位置にまとめられています。
段階とは、コンテナやポッド状態を包括的に監視できるようにするのを目的としていません。
また、包括的なマシン状態のためでもありません。

ポッドの段階で表示される値の数字と意味は、厳密に固定されています。
ここでドキュメント化されているもの以外で、ポッドにおける  `phase` （段階：フェーズ）の値に関する記述はありません。

以下が `phase` として取り得る値です：

値 | 説明
:-----|:-----------
`Pending`（保留中） | ポッドは Kubernetes システムに受諾されましたが、１つまたは複数のコンテナ・イメージ作成が完了していません。これに含まれるのはスケジュールされる前の時間だけでなく、ネットワークを越えてイメージをダウンロードするのにかかる時間も含むので、時間がかかる場合があります。
`Running`（実行中） | ポッドがノードに割り当てられ、全てのコンテナが作成された状態です。少なくとも１つのコンテナが実行中であるか、あるいはプロセスが開始中または再起動中です。
`Succeeded`（成功） | ポッドの全てのコンテナが処理終了（terminated）に成功し、再起動しません。
`Failed`（失敗） | ポッド内の全てのコンテナの処理終了に失敗し、少なくとも１つのコンテナの処理終了が失敗しました。つまり、コンテナは０以外のステータスで異常終了したか、システムによって終了させられました。
`Unknown`（不明） | 何らかの理由によって、ポッドから状態を取得できません。典型的なのはポットのホストとの通信がエラーです。

## ポッドの状態 {#pod-conditions}

ポッドには PodStatus があります。これはポッドにある [PodConditions](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podcondition-v1-core) 配列を通してであり、そうでなければ得られません。PodCondition 配列の各要素は６つのフィールドを持つでしょう：

* `lastProbeTime` フィールドが提供するのは、ポッドの状態を調べた直近のタイムスタンプです。
* `lastTransitionTime` フィールドが提供するのは、あるポッドのステータスが別のステータスに移行した、直近のタイムスタンプです。
* `message` フィールドが提供するのは、人間が読める移行に関する詳細なメッセージを表示します。

ポッドには PodStatus があり、これには [PodConditions](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podcondition-v1-core) の配列があります。PodCondition 配列の各要素には `type` フィールドと `status` フィールドがあります。 `type` フィールドには、次の値の文字列がある可能性があります：

* `PodScheduled`（ポッドがスケジュール済み）: ポッドがノードにスケジュールされた
* `Ready`（待機）: ポットが全ての対象サービスを負荷分散プールに追加して、リクエストを処理可能な状態
* `Initialized`（初期化）: 全ての [コンテナ初期化（init containers）](/jp/docs/concepts/workloads/pods/init-containers) 処理に成功
* `Unschedulable`（スケジュール不可）: スケジューラがポッドを適切に処理できない。他のコンテナによってリソースが欠乏しているなど
* `ContainersReady`（コンテナ準備完了）: ポッド内の全コンテナの準備が完了

`status` （状態）フィールドは "``True" か "`Flase`" か "`Unknown`" の文字列です。

## コンテナ調査（Container probes） {#container-probes}

[Probe（調査）](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#probe-v1-core) とは  [kubelet](/jp/docs/admin/kubelet/) がコンテナ上で行う定期的な性能診断です。診断の処理は kubelet がコンテナに実装された [Handlerアンドラ）](https://godoc.org/k8s.io/kubernetes/pkg/api/v1#Handler) を呼び出します。ハンドラは3種類あります：

* [ExecAction](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#execaction-v1-core)（実行アクション）：コンテナ内で指定されたコマンドを実行します。コマンドがステータス・コードが 0 で終了すると、診断に成功したとみなします。

* [TCPSocketAction](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#tcpsocketaction-v1-core)（TCP ソケット・アクション）：コンテナ IP アドレス上の特定ポートに対して TCP チェックを実行します。ポートがオープンであれば（開いていれば）、診断に成功したとみなします。

* [HTTPGetAction](/jpdocs/reference/generated/kubernetes-api/{{< param "version" >}}/#httpgetaction-v1-core)（HTTP 取得アクション）：コンテナ IP アドレス上の特定ポートとパスに対する HTTP Get （取得）リクエストを実行します。200 ～ 400 までのステータス・コードを返す応答があれば、診断に成功したとみなします。

各調査には３つの結果があります:

* Success（成功）：コンテナは診断に合格。
* Failure:(（失敗）：コンテナは診断に不合格。
* Unknown（不明）：診断が失敗したため、何も行動を起こせない。

オプションにより、kubelet は実行中のコンテナ上で、２種類の診断と対応ができます。

* `livenessProbe`（生存性診断）：コンテナが実行中かどうかを示します。もし も生存性診断に失敗すると、kubelet はコンテナを強制停止（kill）し、[再起動方針](#restart-policy) に従ってコンテナを処理します。もしもコンテナが生存性診断を提供しない場合、デフォルトの状態は `Success` （成功）です。

* `readinessProbe`（読込性診断）：はコンテナがサービス・リクエストに対応できるかどうかを示します。もしも読込性診断に失敗すると、全サービスのエンドポイントに一致するポッドがあれば、エンドポイント・コントローラはポッドの IP アドレスを削除します。読込性診断のデフォルト値は `Failuer` （失敗）です。もしコンテナが読込性診断を提供しない場合、デフォルトの状態は `Success` （成功）です。

### いつ生存性診断や読込性診断を使うべきですか？ {#when-should-you-use-liveness-or-readiness-probes}

コンテナ内のプロセスが自分自身でクラッシュできる場合は、問題が発生すると不健全（unhealthy）になります。
そのため生存性診断は不要です。言い換えれば、kubelet はポッドの `restartPolicy`（再起動方針）に従って自動的に適切な処理を行うよういします。

診断に失敗した場合、コンテナを停止して再起動したければ、生存性診断を設定し、 `restartPolicy`（再起動方針）を Always（常時） か OnFailuer （障害時）に指定します。

ポッドに対する診断が成功する時のみトラフィックを送信開始したい場合は、読込性診断を指定します。
この場合、読込性診断は生存性診断と同じように見えるかもしれません。
しかし、spec 上で読込性診断の指定が存在していれば、トラフィックを受診していなくてもポッドを開始し、診断に成功した後のみトラフィックを受診するという意味があります。

もしもコンテナが大きなデータを読み込む処理を必要とする場合は、設定ファイルや起動中のマイグレーションで読込性診断を指定してください。

コンテナが自分自身をメンテナンスでダウン可能にしたければ、読込性診断を指定します。
これは、特定のエンドポイントに対する読込性の確認を設定するものであり、製造性診断とは異なります。

ポッドの削除時にドレイン（drain：排出）要求をしたい場合は、読込性診断を指定する必要がありませんのでご注意ください。
削除時は、読込性診断があるかどうかに拘わらず、ポッドが自動的に読込性の状態を unready （準備ができていない）に変えるからです。
ポッド内のコンテナが待機している間中、ポッドはずっと unready のまま停止します。

生存性診断または読込性診断をセットアップする更なる情報は、[ポッドの生存性および読込性診断](/jp/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) をご覧ください。

## ポッドとコンテナの状態 {#pod-and-container-status}

ポッド・コンテナ状態に関する詳しい情報は
[PodStatus（ポッド状態）](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podstatus-v1-core)
と
[ContainerStatus（コンテナ状態）](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#containerstatus-v1-core).
をご覧ください。
ポット状態として表示される情報は
[ContainerState（コンテナ状態）](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#containerstatus-v1-core)
に依存しますのでご注意ください。

<!--
## ポッド待機ゲート（Pod readiness gate）{#pod-readiness-gate}

{{< feature-state for_k8s_version="v1.11" state="alpha" >}}

ポッドに対して拡張性を付与するため、外部のフィードバックまたはシグナルを `PodStatus` （ポッド状態）に投入できる機能、Kubernetes 1.11 では [Pod ready++](https://github.com/kubernetes/community/blob/master/keps/sig-network/0007-pod-ready%2B%2B.md) という名称で機能が導入されました。
`PodSpec` 内で新しいフィールド `ReadinessGate` （待機ゲート）を利用して、追加の状況（condition）を指定すると、ポッドの準備中に評価できるようになります。
もしも Kubernetes がポッドの `status.condition` フィールド内に対象の状況（condition）がなければ、状態のデフォルトは "`False`" です。以下が例になります：


```yaml
Kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready  # こちらの初期の PodCondition
      status: "True"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"   # こちらは追加した PodCondition
      status: "False"
      lastProbeTIme: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

新しいポッド状態の指定は Kubernetel [ラベル・キー書式](/jp/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)に従う必要があります。
`kubectl patch` コマンドはオブジェクト状態に対するパッチをまだサポートしていないため、新しい Pod 状態を設定するには、[KubeClient ライブラリ](/jp/docs/reference/using-api/client-librarie/) の１つを使い `PATCH` アクションを投入する必要があります。

新しいポッド状態を導入するには、ポッドが以下両方の状態が真（True）の時 **のみ** 評価します：

* ポッド内のコンテナ全てが準備完了
* 全ての `ReadinessGate`（待機中ゲート）状態 が `True` に指定

ポッドに対する読込性評価を調整するために、新しいポッド状態 `ContainersReady` を古いポッド状態 `Ready` から取得します。

"Pod Ready++" はアルファ機能のため、利用するには `PodReadinessGates` [待機ゲート](/jp/docs/reference/command-line-tools-reference/feature-gates/) 機能を `True` に設定する必要があります。
-->

## 再起動方針 {#restart-policy}

PodSpec にある `restartPolicy` （再起動方針）フィールドの値に入るのは、Always （常に再起動）か OnFailure （障害時に再起動）か Never （再起動しない）です。
デフォルトは Always です。
`restartPolicy`  が適用されるのは、ポッド内の全てのコンテナです。
`restartPolicy` はコンテナを再起動するために、同じノード上にある kubelet が参照します。
終了したコンテナは kubelet によって自動的に再起動されます。
この再起動（のタイミング）は簡単な指数関数的に遅延（10秒、20秒、40秒）します。
上限は5分です。
また実行に成功すると10分後にリセットされます。
[Pod ドキュメント](/jp/docs/concepts/workloads/pods/pod/#durability-of-pods-or-lack-thereof) に議論があるように、ポッドはノードと共に稼働しますので、ポッドは他のノードに再配置されません。

## ポッドの生存期間（ライフタイム） {#pod-lifetime}

通常、ポッドは何らかによって破棄されない限り、消滅しません。破棄するのは人間かコントローラです。この規則には唯一の例外があります。それはポッドの `phase` （段階）が成功か失敗が一定時間（マスタの `terminated-pod-gc-threshold` によって決定）経過すると、期限切れとなり自動的にポッドが破棄されます。

3種類のコントローラで使えます:

- 例えばバッチ処理などで、ポッドに [ジョブ（Job）](/jp/docs/concepts/jobs/run-to-completion-finite-workloads/) を使って停止できるようにします。ポッドに対するジョブの `restartPolicy`（再起動方針）は OnFailure（障害時のみ） か Never （再起動しない）です。

- 例えばウェブサーバで、ポッドに [ReplicationController](/jp/docs/concepts/workloads/controllers/replicationcontroller/)、[ReplicaSet](/jp/docs/concepts/workloads/controllers/replicaset/)、 [Deployment](/jp/docs/concepts/workloads/controllers/deployment/) を使い、停止しないようにします。ポッドに対する ReplicationController は `restartPolicy` が Awlays（常時）のみ適用されます。

- ポッドに [DaemonSet](/jp/docs/concepts/workloads/controllers/daemonset/) を使うのは、マシンごとにシステム・サービスを提供するので、マシンごとにポッドを実行する必要があるからです。

３種類のコントローラは PodTemplate （ポッド・テンプレート）を含みます。
推奨するのは、直接自分でポッドを作成するのではなく、 適切なコントローラを作成して、コントローラにポッドを作成させる方法です。
これはマシン障害発生時にポッドには回復力はありません。
しかし、コントローラには回復力があるからです。

ノードが停止するかクラスタから切り離されると、Kubernetes は対象となる全てのポッドの `phase` を Failed（障害）に変更します。

## 例 {#examples}

### 高度な生存性診断の例 {#advanced-liveness-probe-example}

生存性診断は kubelet によって実行されるため、kubelet 名前空間の中で全ての要求が作成されます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - args:
    - /server
    image: k8s.gcr.io/liveness
    livenessProbe:
      httpGet:
        # when "host" is not defined, "PodIP" will be used
        # host: my-host
        # when "scheme" is not defined, "HTTP" scheme will be used. Only "HTTP" and "HTTPS" are allowed
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      timeoutSeconds: 1
    name: liveness
```

### 状態の例 {#example-states}

   * ポッドを実行中で１つのコンテナがある。コンテナの終了が成功する。
     * 完了したイベントを記録する。
     * もし `restartPolicy` （再起動方針）が：
       * Always（常時）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * OnFailure（障害時に再起動）の場合：ポッドの `phase`  は Succeeded（成功）になる。
       * Never（再起動しない）の場合：ポッドの `phase`  は Succeeded（成功）になる。

   * ポッドを実行中で１つのコンテナがある。コンテナの終了が失敗する。
     * 失敗したイベントを記録する。
     * もし `restartPolicy` （再起動方針）が：
       * Always（常時）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * OnFailure（障害時に再起動）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * Never（再起動しない）の場合：ポッドの `phase`  は Failed（失敗）になる。

   * ポッドを実行中で２つのコンテナがある。コンテナ１の終了が失敗する。
     * 失敗したイベントを記録する。
     * もし `restartPolicy` （再起動方針）が：
       * Always（常時）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * OnFailure（障害時に再起動）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）にまま。
       * Never（再起動しない）の場合：コンテナを再起動せず、ポッドの `phase`  は Running（実行中）のまま。
     * もしコンテナ１が実行中ではなく、コンテナ２が終了した場合：
     * 失敗したイベントを記録する。
     * もし `restartPolicy` （再起動方針）が：
       * Always（常時）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * OnFailure（障害時に再起動）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * Never（再起動しない）の場合：ポッドの `phase`  は Failed（失敗）になる。

   * ポッドを実行中で１つのコンテナがある。コンテナでメモリ不足が発生する。
     * コンテナは障害によって停止（terminate）する。
     * OOM イベントを記録する。
     * もし `restartPolicy` （再起動方針）が：
       * Always（常時）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * OnFailure（障害時に再起動）の場合：コンテナを再起動し、ポッドの `phase` は Running（実行中）のまま。
       * Never（再起動しない）の場合：イベントの失敗を記録し、ポッドの `phase`  は Failed（失敗）になる。

   * ポッドを実行中で、ディスクが故障する。
     * 全てのコンテナを停止。
     * 適切なイベントを記録。
     * ポッドの `phase` は Failed（障害）になる。
     * コントローラが稼働中であれば、ポッドをどこかで再作成する。

   * ポッドが実行中で、ノードがセグメント・アウトする。
     * ノード・コントローラはタイムアウトを待つ。
     * ノード・コントローラはポッドの `phase` を Failed （失敗）に設定。
     * コントローラが稼働中であれば、ポッドをどこかで再作成する。

{{% /capture %}}


{{% capture whatsnext %}}

* ハンズオンを試す  [コンテナ・ライフサイクル・イベントの扱いを割り当て](/jp/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/).

* ハンズオンを試す
  [生存性または読込性診断の設定](/jp/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/).

* [コンテナ・ライフサイクル・フック](/jp/docs/concepts/containers/container-lifecycle-hooks/) について学ぶ。


{{% /capture %}}



