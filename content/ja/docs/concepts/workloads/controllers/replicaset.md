---
title: ReplicaSet（レプリカセット）
content_template: templates/concept
weight: 10
---

{{% capture overview %}}
ReplicaSet（レプリカセット）は次世代のレプリケーション・コントローラです。
_ReplicaSet_ と [_Replication Controller（レプリケーション・コントローラ）_](/jp/docs/concepts/workloads/controllers/replicationcontroller/) との相違点は、セレクタのサポートのみです。
Replication Controller がサポートするのは品質ベースのセレクタ要件であるのに対して、新しい設定ベース（set-based）セレクタの必要条件は [ラベル・ユーザ・ガイド](/jp/docs/concepts/overview/working-with-objects/labels/#label-selectors) に記述があります。

{{% /capture %}}

{{% capture body %}}
## ReplicaSet の使い方 {#how-to-use-a-replicaset}

ほとんどの  [`kubectl`](/jp/docs/reference/kubectl/kubectl/) コマンドは Replication Controller と ReplicaSet もサポートします。
例外の１つは [`rolling-update`](/jp/docs/reference/generated/kubectl/kubectl-commands#rolling-update) （ローリング・アップデート）コマンドです。ローリング・アップデート機能を使いたい場合は、かわりに Deployment（デプロイメント）の使用を検討してください。
また、[`rolling-update`](/jp/docs/reference/generated/kubectl/kubectl-commands#rolling-update) （ローリング・アップデート）コマンドコマンドは命令型として扱われるのに対し、Deployment（デプロイメント）では宣言型です。
そのため、私たちが推奨するのは Deployment（デプロイメント）を通して [`rollout`](/docs/reference/generated/kubectl/kubectl-commands#rollout) （ロールアウト）コマンドをご利用ください。

ReplicaSet の利用は独立的であり、ポッドの作成、削除、更新を調整（オーケストレート）するための仕組みとしては [Deployments](/jp/docs/concepts/workloads/controllers/deployment/) （デプロイメント）が主に使われます。
Deployment （デプロイメント）の利用にあたっては、既に作成した ReplicaSet に対する心配は不要です。
Deployment は自身と ReplicaSet の両方を管理します。

## ReplicaSet の利用時 {#when-to-use-a-replicaset}

ReplicaSet は、指定したポッドの複製数が常に実行中にするのを保証します。
しかしながら、Deployment は ReplicaSet を管理する高レベルの概念であり、ポッドに対する宣言型の更新は、他の多くの便利な機能をもたらします。
そのため、私たちは ReplicaSet を直接使うのではなく Deployment の使用を推奨します。
ただし、カスタム・アップデート・オーケストレーションが必要ではない場合と、すべてを更新する必要がない場合を除きます。

つまり、ReplicaSet オブジェクトを操作する必要は一切ありません。
そのかわり、Deployment を使って、アプリケーションの spec セクションで定義します。

## 例 {#example}

{{< codenew file="controllers/frontend.yaml" >}}

このマニフェストを `frontend.yaml` に保存し、Kubernetes クラスタに送信すると、定義した ReplicaSet をポッドに作成し、管理できるようになります。

```shell
$ kubectl create -f http://k8s.io/examples/controllers/frontend.yaml
replicaset "frontend" created
$ kubectl describe rs/frontend
Name:		frontend
Namespace:	default
Selector:	tier=frontend,tier in (frontend)
Labels:		app=guestbook
		tier=frontend
Annotations:	<none>
Replicas:	3 current / 3 desired
Pods Status:	3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       app=guestbook
                tier=frontend
  Containers:
   php-redis:
    Image:      gcr.io/google_samples/gb-frontend:v3
    Port:       80/TCP
    Requests:
      cpu:      100m
      memory:   100Mi
    Environment:
      GET_HOSTS_FROM:   dns
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen    LastSeen    Count    From                SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----                -------------    --------    ------            -------
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-qhloh
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-dnjpy
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-9si5l
$ kubectl get pods
NAME             READY     STATUS    RESTARTS   AGE
frontend-9si5l   1/1       Running   0          1m
frontend-dnjpy   1/1       Running   0          1m
frontend-qhloh   1/1       Running   0          1m
```

## ReplicaSet Spec を書く {#writing-a-replicaset-spec}

他のすべての Kubernetes API オブジェクトと同様に、ReplicaSet には `apiVersion` 、`kind`、 `metadata`  フィールドが必要です。
マニフェストの扱い方に関する一般的な情報は [object management using kubectl](/jp/docs/concepts/overview/object-management-kubectl/overview/) をご覧ください。

また、ReplicaSet には [`.spec` セレクション](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status)が必要です。

### Pod テンプレート（Pod Template） {#pod-template}

`.spec.template` は `.spec` で唯一必須のフィールドです。
`.spec.template` は [ポッド template](/jp/docs/concepts/workloads/pods/pod-overview/#pod-templates) です。
これは厳密には [ポッド](/jp/docs/concepts/workloads/pods/pod/) と同じスキーマですが、ネスト化（階層化）しているのと、 `apiVersion` や `kind` を持たないのが違います。

ポッドの必須フィールドに付け加えると、ReplicaSet 内のポッド・テンプレートには適切なラベルと適切な再起動方針（restart policy）の指定が必須です。

ラベルに関しては、他のコントローラと重複しないようにする必要があります。
詳しい情報は[ポッド・セレクタ（pod selector）](#pod-selector)をご覧ください。

[再起動方針（restart policy）](/jp/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) については、許可されている値は `.spec.template.spec.restartPolicy`  に対してのみであり、デフォルトは `Always` （常時）です。

ローカル・コンテナの再起動については、ReplicaSet はノード上のエージェントに権限を委譲しています。
例えば [Kubelet](/jp/docs/admin/kubelet/) や Docker です。

### ポッド・セレクタ（Pod Selector）{#pod-selector}

`.spec.selector` フィールドは[ラベル・セレクタ](/jp/docs/concepts/overview/working-with-objects/labels/)です。
ReplicaSet はすべてのポッドの管理を、ラベルとラベルに一致するセレクタで行います。
セレクタはポッドの作成または削除時に、ポッドが人もしくはプロセスが作成または削除するかどうかを区別しません。
これにより、ReplicaSet は実行しているポッドの影響を受けることなく、置き換え（replace）可能です。

`.spec.template.metadata.labels` は `.spec.selector` と一致する必要があります。
一致しない場合は API によって拒否されます。

Kubernetes 1.9 の API バージョン `apps/v1` では、ReplicaSet kind が現在のバージョンではデフォルトで有効化されています。
API バージョン `apps/v1beta2` は廃止済みです。

また、ポッドの作成にあたって、このセレクタに一致するラベルのポッドを重複して指定できません。
さらに、他の ReplicaSet や、他の Deployment などのコントローラ等とも重複できません。
もしも重複すると、ReplicaSet は他のポッドを作成したと認識します。
Kubernetes はこのような作業を止められません。

もしも複数のコントローラが存在してしまうと、セレクタが重複してしまい、削除を自分自身で行う必要が出てきます。

### ReplicaSet 上のラベル {#labels-on-a-replicaset}

ReplicaSet は自分自身のラベルを持てます（`.metadata.labels`）。
典型的なのは `.spec.template.metadata.labels` と同じものをセットする方法です。
しかしながら、これらは別々のものとして認識されるため、ReplicaSet の挙動に影響を与えないためには `.metadata.labels` を使います。

### 複製（レプリカ：Replicas） {#replicas}

いくつのポッドを同時に実行すべきかを `.spec.replicas` で指定できます。
実行中の数は常に増減する可能性がありますが、もしも複製が増えたり減ったりしたら、ポッドは丁寧にシャットダウンするか、早々に複製を起動します。

`.spec.replicas` を指定しなければ、デフォルトの 1 として扱われます。

## ReplicaSets を使う {#working-with-replicasets}

### ReplicaSet と Pod の削除 {#deleting-a-replicaset-and-its-pods}

ReplicaSet とポッドのすべてを削除するには、[`kubectl delete`](/jp/docs/reference/generated/kubectl/kubectl-commands#delete) を使います。
Kubectl は ReplicaSet をゼロにスケール（規模変更）し、各ポッドが削除されるまで待機したら、ReplicaSet 自身を削除します。
kubectl コマンドが中断されると、再起動されます。

REST API や go クライアント・ライブラリを使用する場合は、各ステップを明示的に行う必要があります（複製を 0 にスケールし、ポッドの削除まで待機し、その後 ReplicaSet を削除）。

### ReplicaSet で削除 {deleting-just-a-replicaset}

[`kubectl delete`](/jp/docs/reference/generated/kubectl/kubectl-commands#delete) に `--cascade=false` オプションを指定すると、あらゆるポッドに影響を与えず ReplicaSet を削除できます。

REST API や go クライアント・ライブラリを使う場合は、単純に ReplicaSet オブジェクトを削除します。

オリジナルの ReplicaSetを削除したら、新しい ReplicaSet を作成し、置き換えできます。
新旧どちらの `.spec.selector` が同じであれば、新しい ReplicaSet が古いほうのポッドを扱えます。
しかしながら、既存のポットと新しいポッドが異なるポッド・テンプレートの場合、何ら影響を及ぼせません。
ポッドを新しい spec で管理できるように更新するには、 [ローリング・アップデート](#rolling-update) を使います。

### ポッドを他の ReplicaSet から独立（Isolating）する {#isolating-pods-from-a-replicaset}

ポッドを ReplicaSet の対象から削除するには、ラベルの変更で行える場合があります。
このテクニックはサービスのデバッグ用やデータ修復用など、ポッドを削除するために使えるでしょう。
この方法によって削除すると、自動的に新しいものが置き換えられます（複製の数は変更していないとみなした場合）。

### ReplicaSet のスケーリング {#scalling-replicaset}

ReplicaSet は `.spec.replicas` フィールドをシンプルに変更するだけで、簡単にスケールアップまたはダウンができます。
ReplicaSet コントローラはポッドを希望する数の維持を確実に行い、一致するラベル・セレクタを利用可能かつ操作可能にします。

### ReplicaSet を水平ポッド・オートスケーラのターゲットとする {replicaset-as-an-horizontal-pod-autoscaler-target}

また、ReplicaSet は [水平ポッド・オートスケーラ(HPA)](/jp/docs/tasks/run-application/horizontal-pod-autoscale/) の対象（ターゲット）にもなれます。
これにより、ReplicaSet は HPA によってオートスケールされるようになります。
こちらにあるのは以前の例で作成した ReplicaSet を HPA のターゲットとなるようにしたサンプルです。

{{< codenew file="controllers/hpa-rs.yaml" >}}

このマニフェストを `hpa-rs.yaml` として保存し、 Kubernetes クラスタに対して送信すると、定義した HPA が作成され、ターゲットとなる ReplicaSet は複製されたポッドの CPU 使用率に応じてオートスケールします。

```shell
kubectl create -f https://k8s.io/examples/controllers/hpa-rs.yaml
```

あるいは、 `kubectl autocale` コマンドを使って同じ処理をします（そして、こちらが簡単です！）。

```shell
kubectl autoscale rs frontend
```

## ReplicaSet の代替 {#alternatives-to-replicaset}

### Deployment（デプロイメント）（推奨） {#deployment-recommended}

[`Deployment（デプロイメント）`](/jp/docs/concepts/workloads/controllers/deployment/) は高レベルの API オブジェクトであり、これを使って基礎となる ReplicaSet とそれらのポッドを `kubectl rolling-update` のように簡単に更新します。
Deployment はローリング・アップデートを機能的に使いたい時に推奨されます。
なぜなら、 `kubectl rolling-update` とは異なり、こちらは宣言型であり、サーバ側での処理（サーバサイド）で、かつ、追加機能があります。
ステートレス名アプリケーションを実行するために Deployment を使う場合は [Deployment でステートレスなアプリケーションを実行する](/jp/docs/tasks/run-application/run-stateless-application-deployment/) をご覧ください。

### ベア・ポッド （Bare Pod） {#bare-pods}

ユーザが直接ポッドを作成するのとは異なり、ReplicaSet がポッドを置き換えます。
これはノードの障害や、カーネルのアップグレードのようなノードの分断的なメンテナンスなどの状況下など、あらゆる理由によってポッドの削除または停止を ReplicaSet が行います。
そのため、アプリケーションが必要となるのが１つのポッドだとしても、私たちは ReplicaSet の使用を推奨します。
プロセスのスーパーバイザと同じようなものと考えると、1つのノード上で個々のプロセスを管理するのではなく、複数のノードに横断する複数のポッドをスーパーバイズ（監督）するようなものです。
ReplicaSet はローカルにあるコンテナの再起動ですら、ノード上の何らかのコントローラに委任します（例：Kubelet や Docker）。

### ジョブ（Job） {#job}

ポッドに ReplicaSet の代わりに [`Job（ジョブ）`](/jp/docs/concepts/jobs/run-to-completion-finite-workloads/) を使う方法です。
これは各々が自身を終了（terminate）するのが期待されます（つまり、バッチ・ジョブです）。

### DaemonSet（デーモンセット） {#daemonset}

ポッドに対して ReplicaSet の代わりに [`DaemonSet`（デーモンセット）](/jp/docs/concepts/workloads/controllers/daemonset/) を使う方法です。
こちらはマシンの監視やマシンのログ記録など、マシン・レベルの機能を提供します。
各ポッドはマシンのライフタイムに強く結びつくライフタイム（生存期間）を持ちます。つまり、ポッドに必要なのはマシン上で他のポッドの起動前に実行することであり、マシンが再起動またはシャットダウンなど利用できなくなる前に、安全に停止できるようにします。

{{% /capture %}}


