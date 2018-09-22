---
title: Deployments（デプロイメント）
content_template: templates/concept
weight: 30
---

{{% capture overview %}}

_Deployment（デプロイメント）_ コントローラは  [ポッド](/jp/docs/concepts/workloads/pods/pod/)  と[ReplicaSets（レプリカ・セット）](/docs/concepts/workloads/controllers/replicaset/) に対して宣言型の更新機能を提供します。

Deployment オブジェクトで _desired state（期待状態）_ を記述すると、デプロイメント・コントローラは現在の状態を期待状態へ変更するような調整を制御します。
新しい ReplicaSets を作成するため、Deployment を定義できます。あるいは、Deployment から既存のものを削除し、全てのリソースが新しい Deployment を採用するようにします。

{{< note >}}
**メモ：** Deployment によって所有されている ReplicaSet は自分自身で管理すべきではありません。
全ての利用例は Deployment オブジェクトによって操作されるのが前提です。

{{< /note >}}

{{% /capture %}}


{{% capture body %}}

## 利用例 {#use-case}

以下は典型的な Deployment の利用例です：

* [ReplicaSet を展開する Deployment を作成](#creating-a-deployment)。ReplicaSet はバックグラウンドでポッドを作成します。成功したかどうかを確認するには、ロールアウト（rollout）のステータスを確認します。
* [ポッドの新しい状態宣言](#updating-a-deployment) を Deployment の PodTemplateSpec で更新します。新しい ReplicaSet is が作成され、Deployment はポッドが古い ReplicaSet から新しいものへ移動する調整を管理します。それぞれの新しい ReplicaSet は Deployment のリビジョンによって更新されます。
* もしも Deployment の現在の状態を取得できなければ [直前の Deployment リビジョンにロールバック（巻き戻す）](#rolling-back-a-deployment)します。それぞれのロールバックは Deployment のリビジョンを更新します。
* [より高い負荷に対応するため Deployment をスケールアップする](#scaling-a-deployment)。
* [Deploymentの一次停止](#pausing-and-resuming-a-deployment) を、PodTemplateSpec による複数の変更を適用するため、そして新しいものが展開（ロールアウト）したら再開します。
* スタックをロールアウトする時に、状態を表示する（インディケータ）ため[Deployment のステータスを使う](#deployment-status)。
* [以前の ReplicaSet をクリーンアップ（掃除）する](#clean-up-policy) that you don't need anymore.

## Deployment の作成 {#ceating-a-deployment}

以下は Deployment の例です。`nginx` ポッドが３つある ReplicaSet を作成します。

{{< codenew file="controllers/nginx-deployment.yaml" >}}

この例では:

* `nginx-deployment` という名前の Deployment を作成する。 `.metadata.name` フィールドで名前が指定されている。
* Deployment は `replicasf` フィールドで指定されている３つの複製ポッドを作成する。
* Deployment が管理対象のポッドを識別するための、 `selector` フィールドを定義する。今回の例では、ポッド・テンプレート内でのラベル定義（ `app: nginx`）を指定するのみ。しかしながら、ポッド・テンプレート自身のルールを満たす限り、より高度な選択ルールも必要であれば指定できます。
* ポッド・テンプレートの仕様（specification） 、すなわち `.template.spec` フィールドで、ポッドが１つのコンテナを実行するのを示します。コンテナは `nginx` であり、 [Docker Hub](https://hub.coker.com/) イメージのバージョン 1.7.9 を使います。
* Deployment はポッドのためにポート  80 を開きます。

{{< note >}}
**メモ：**  `matchLables` はマップの {キー,値}のペアです。`matchLables` 内の１つの {キー,値} は、は `matchExpressions` の要素と同等であり、このキーにあるフィールドは "key" として、演算子は "In" であり、配列に入っている値が "value" です（key フィールドに入っているものが value です）。必要条件は AND化です。
{{< /note >}}

`template` フィールドは以下の指示を含みます：

* ポッドは `app: nginx` のラベルを付ける
* １つのコンテナを作成し、 `nginx` と名付ける
* `nginx` イメージのバージョン `1.7.9` を実行
* ポート `80` を開き、コンテナがトラフィックの送受信を可能とする

この Deployment を作成するには、以下のコマンドを実行します：

```shell
kubectl create -f  https://k8s.io/examples/controllers/nginx-deployment.yaml
```

{{< note >}}
**メモ：**  コマンドに `--record` を追加出来ます。これはリソースを作成または更新しても、現在のコマンドをアノテーションに記録します。これは各 Deployment の改訂（リビジョン：）ごと調査用のコマンドを実行するなど、将来的なレビューに役立ちます。
{{< /note >}}


次に `kubectl get deployments` を実行します。出力結果は以下のようになります：

```shell
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```

クラスタ内の Deployment を調べるために、以下のフィールドが表示されています:

* `NAME` はクラスタ内の Deployment の名前を一覧表示
* `DISIRED`  はアプリケーションの期待する _複製_ 数（replica数）を表示します。これは Deployment の作成時に指定したものです。これが _期待状態（desired state）_ です。
* `CURRENT`  は現在実行中の複製（レプリカ）を表示します。
* `UP-TO-DATE` は期待状態に到達するため、いくつのレプリカを実行中か表示します。
* `AVAILABLE` はアプリケーションの複製（レプリカ）をユーザがいくつ使えるか表示します。
* `AGE` はアプリケーションが実行中になってから経過した時間を表示します。

各フィールドの値が、Deployment 仕様（Deployment spec）における値に相当しますのでご注意ください。

* `.spec.replicas` フィールドによると、期待する複製（レプリカ）数は３。
* `.status.replica` フィールドによると、現在の複製数は０。
* `.status.updatedReplicas` フィールドによると、最新の（up-to-date）複製数は０。
* `.status.availableReplicas` フィールドによると、利用できる複製数は０。

Deployment の展開ステータスを表示するには、 `kubectl rollout status deployment/nginx-deployment` を実行します。このコマンドは以下の出力を返します。

```shell
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```
（展開が完了するまで待機中。３つのうち２つの新しい服し枝が更新中……デプロイメント "nginx-deployment" は展開に成功）

`kubectl get deployments` を数秒後に再び実行します:

```shell
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           18s
```

Deployment で３つの複製が全て作成されたのに注目します。
また、複製は最新（up-to-date）（ここには最新のポッド・テンプレートも含みます）かつ利用可能（available）（Deployment の `.spec.minReadySeconds` フィールドにある最新のポッドのステータス値が Ready＝準備完了のもの ）です。

デプロイメントによって作成された ReplicaSet (`rs`) を表示するには、 `kubectl get rc` を実行します。

```shell
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-2035384211   3         3         3       18s
```

ReplicaSet によって作成される名前は、常に `{デプロイメント名}-{ポッド・テンプレート-ハッシュ値}` の形式なのがわかります。
ハッシュ値はデプロイメントの作成時に自動的に生成されるものです。

各ポッドには自動的に生成されたラベルが割り当てられます。表示するには `kubectl get pods --show-labels` を実行すると、以下の出力結果が戻ります:

```shell
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-2035384211-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
```

作成した ReplicaSet は、常に３つの `nginx` ポッド実行を確保します。

{{< note >}}
**メモ：** デプロイメントでは適切なセレクタとポッド・テンプレート・ラベルの指定を必ず行ってください（この例では `app:nginx` です）。
ラベルやセレクタは他のコントローラと重複しないようにしてください（他のデプロイメントや StatefulSets も含みます）。
Kubernetes は重複による停止は行いません。
そして、もし複数のコントローラが重複するセレクタを持つ場合、各コントローラは衝突もしくは予期しない挙動を起こす可能性があります
{{< /note >}}

### ポッド・テンプレートのハッシュ値ラベル

{{< note >}}
**メモ：** このラベルを変更しないでください。
{{< /note >}}

`pod-template-hash`ラベルはデプロイメント・コントローラにより、デプロイメントによって作成または採用された全ての ReplicaSet に対して追加されます。

このラベルはデプロイメントの子 ReplicaSet と重複しないようにします。
ReplicaSet の `PodTemplate` のハッシュをもとに作成されるものです。
得られたハッシュを使い、ラベルの値として指定します。ReplicaSet が持っているであろうラベルの値は、 ReplicaSet セレクタ、ポッド・テンプレート・ラベル、そして、あらゆる既存のポッドに対して追加されます。

## デプロイメントの更新 {#updating-a-deployment}

{{< note >}}
**メモ：**  デプロイメントのポッド・テンプレートが変更された時のみ（ここでは `.spec.template` ）、デプロイメントを展開（rollout）する処理開始（トリガ）となります。
たとえば、テンプレートのラベルまたはコンテナ・イメージの更新です。
デプロイメントのスケーリングなど、その他の変更時は展開を開始しません。
{{< /note >}}

nginx ポッドを `nginx:1.7.9` を使うイメージから`nginx:1.9.1` イメージへ更新したいと仮定しましょう。

```shell
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated
```

あるいは、デプロイメントを編集（`edit`）して、  `.spec.template.spec.containers[0].image` を `nginx:1.7.9` から `1.9.1` に変更します。

```shell
$ kubectl edit deployment/nginx-deployment
deployment "nginx-deployment" edited
```
展開状態を表示するには、次のように実行します:

```shell
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

展開に成功したら、デプロイメントの状態を取得（ `get` ）したいでしょう：

```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           36s
```

最新の（up-to-date）複製数が示すのは、最新の設定情報によって、デプロイメントの複製数が変更されたことです。
現在の複製数（current replicas）が示すのは、このデプロイメントが管理している合計複製数のうち、現在利用できる複製の数です。

`kubectl get rs` を実行すると、デプロイメントがポッドを更新するため、新しい ReplicaSet を作成して３つの複製にスケールします。
同様に古い ReplicaSet の複製を０に変更するためスケールダウンします。


```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s
```

`get pods` を実行すると、表示されているのは新しいポッドのみです:

```shell
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

次に、これらのポッドを更新します。必要なのはデプロイメントのポッド・テンプレートの再編集のみです。

デプロイメントは更新によって停止するポッド数が一定数に収まるようにします。
デフォルトでは、（訳者注：停止中となるポッドの数は）ポッド期待値の 25% に収まるようにします（最大で 25% 利用できなくなります）。

また、デプロイメントは作成済みのポッド数を一定数維持します。
デフォルトでは、ポッド期待値の 25% 以上を（訳者注：常に実行中のポッドを一定数確保しながら）実行するようにします（25% が設定可能な上限です）。

たとえば、先ほどのデプロイメントを詳しく見ると、まず始めに新しいポッドを作成し、その後に古いポッドを作成し、それから新しいポッドを作成するのが分かります。
十分な数の新しいポッドが起動しない、古いポッドを削除しません。そして、十分な数の古いポッドを停止（kill）するまで、新しいポッドを作成しません。
これにより、利用可能なポッド数は最小２つであり、合計のポッド数は最大で４です。


```shell
$ kubectl describe deployments
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 30 Nov 2017 10:56:25 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.9.1
    Port:         80/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-1564180365 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
  Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
```

こちらで分かるのは、まずデプロイメントを作成し、デプロイメントは ReplicaSet（nginx-deployment-2035384211）を作成,し、直接３つの複製（レプリカ）にスケールアップします。
デプロイメントの更新によって、新しい ReplicaSet（nginx-deployment-1564180365）を１つ作成し、古い ReplicaSet を２にスケールダウンします。
その結果、少なくとも２つのポッドが利用可能であり、同時に４つのポッドが同時に利用できるようになります。
同じローリング・アップデート方針（strategy）に従って、新旧 ReplicaSet のスケールアップとダウンが続きます。
最終的には新しい ReplicaSet で３つの複製（レプリカ）が利用可能となり、古い ReplicaSet は０にスケールダウンします。

### ロールオーバ（rollover：載せ替え）（別名、実行しながら同時更新）{#rollover-aka-multiple-update-in-flight}

デプロイメント・コントローラによって新しいデプロイメント・オブジェクトが見つけられるたびに、既存の ReplicaSet で行えることが何もなければ、ReplicaSet は期待する（望ましい）ポッドを作り出します。
既存の ReplicaSet が制御するポッドは `.spec.selector` のラベルに一致するものですが、 `.spec.template` に一致しないものはスケールダウンします。
最終的には新しい ReplicaSet は `.spec.replicas` までスケールし、古い ReplicaSetは０に規模変更（スケールします。

もしも既存のロールアウトが進行中にデプロイメントを更新すると、デプロイメントは新しい ReplicaSet を更新の一部として作成し、スケールアップを開始します。
これによって、以前にスケールアップした ReplicaSet はロールオーバー（載せ替え）となるため、そのためには、古い ReplicaSet の一覧に追加され、いずれスケールダウンが始まります。

たとえば、 `nginx:1.7.9` の複製５つを作成するデプロイメントを作成したと仮定します。
`nginx:1.9.1` の複製を５つ作成するようデプロイメントを更新しようとしますが、 `nginx:1.7.9` が３つしか作成されていません。
このような場合、デプロイメントは直ちに作成済みの `nginx:1.7.9` ポッドを停止開始します。
そして、 `nginx:1.9.1` の作成を開始します。方針を変えず、以前に作成した方針は変更せず、 `nginx:1.7.9` の複製が５つになるのを待ちません。

### ラベル・セレクタの更新 {#label-selector-updates}

一般的にラベル・セレクタの更新を行うべきではなく、事前にセレクタを計画しておくのを推奨します。
ラベル・セレクタの更新処理が必要な場合は、どのようなときも、重大な注意を払いながら、予想されうるすべてを確実に把握する必要があります。

{{< note >}}
**メモ：** API バージョン `apps/v1` では、デプロイメントのラベルセレクタを作成後は変更できません（イミュータブルです）。
{{< /note >}}

* セレクタを追加するには、デプロイメント spec 内のポッド・テンプレート・ラベルに新しいラベルの追加も必要です。そうしなければ、整合性（バリデーション）エラーを返します。この変更にあたって、重複は不可能です。新しいセレクタは古いセレクタによって作成された ReplicaSet とポッドを選択できません。つまり、古い ReplicaSet で作成されたものだけでなく、新しい ReplicaSet で作成されいたものすべてが孤立します（訳者注：操作できなくなります）。
* セレクタの更新 - つまり、セレクタ・キーにある既存の値の変更を意味します。この結果、先ほどの説明と同じ挙動となります。
* セレクタの削除 - これは、デプロイメントのセレクタから既存のキーの削除を意味します。つまり、ポッド・テンプレートのラベルには、変更に必要なものが何もなくなります。ReplicaSet はどれも孤立状態となり、新しい ReplicaSet も作成できなくなります。しかし、既存のポッドや ReplicaSet に対するラベルの削除は可能です。

## デプロイメントの巻き戻し（ローリング・バック） {#rolling-back-deployment}

たまにはデプロイメントを巻き戻し（ロールバック）したい場合があるでしょう。
たとえば、クラッシュがループするなど、デプロイメントが安定しない場合です。
デフォルトでは、デプロイメントすべての展開（ロールアウト）履歴がシステム上に保管されるため、必要があればいつでも巻き戻せます（変更履歴の上限は変更可能です）。

{{< note >}}
**メモ：** デプロイメントの展開（ロールアウト）が行われると、デプロイメントの履歴（リビジョン）が作成されます。
つまり、新しい履歴（リビジョン）が作成されるのは、デプロイメントのポッド・テンプレート（ `.spec.template` ）変更時のみです。
たとえば、テンプレートのラベルやコンテナ・イメージを更新したとします。
他の更新、たとえばデプロイメントのスケーリング（規模変更など）では、デプロイメントの履歴（リビジョン）を作成しません。
そのため、擬似的な手動もしくはオート・スケーリングを可能とします。
つまり、以前の履歴（リビジョン）にロールバックすると、デプロイメントのポッド・テンプレートの部分のみがロールバックされるのです。
{{< /note >}}

デプロイメントの更新にあたって入力間違いがあったと仮定します。イメージ名を `1.9.1` ではなく、 `nginx:1.91` としてしまいました:

```shell
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment "nginx-deployment" image updated
```

展開は固まってしまいます。

```shell
$ kubectl rollout status deployments nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
```

このような展開状態が表示されたら、 Ctrl-C を入力して中断します。
展開が固まる件の詳細については [こちら](#deployment-status) をご覧ください。

また、古い複製（nginx-deployment-1564180365 と nginx-deployment-2035384211）の両方が見えます。
また、新しい複製（nginx-deployment-3066724191）は２つです。

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   2         2         0       25s
nginx-deployment-2035384211   0         0         0       36s
nginx-deployment-3066724191   2         2         2       6s
```

作成されたポッドを調べると、新しい ReplicaSet によって作成された２つのポッドがありますが、イメージの取得ループで固まっています。

```shell
$ kubectl get pods
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
nginx-deployment-3066724191-eocby   0/1       ImagePullBackOff   0          6s
```

{{< note >}}
**メモ：** デプロイメント・コントローラは状態が悪いロールアウト（展開）を自動的に停止し、新しい ReplicaSet のスケールアップも停止します。
この挙動はローリング・アップデートのパラメータ（`maxUnavailable` を明確に）指定がある場合、そのパラメータに依存します。
Kubernetes はデフォルトで値が 1 のため、設定時にパラメータの指定がなければ `.spec.replicas` も 1 になります。
そのため、デプロイメントはデフォルトでは 100% 利用できなくなる可能性があります。
これは将来の Kubernetes では修正されます。
{{< /note >}}

```shell
$ kubectl describe deployment
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       2 updated | 3 total | 2 available | 2 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     nginx-deployment-1564180365 (2/2 replicas created)
NewReplicaSet:      nginx-deployment-3066724191 (2/2 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-1564180365 to 2
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 2
```

こちらを修正するには、デプロイメントを安定版である以前のバージョンに巻き戻す必要があります。

### デプロイメントの展開履歴を確認 {#checking-rollout-history-of-a-deployment}

まず、このデプロイメントの履歴（リビジョン）を確認します:

```shell
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f https://k8s.io/examples/controllers/nginx-deployment.yaml --record
2           kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```

デプロイメントの作成時に `--record` を使って記録するコマンドを指定したため、各履歴ごとの変更箇所を簡単に確認できます。

各履歴の更なる詳細を表示するには、次のように実行します:

```shell
$ kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
  Containers:
   nginx:
    Image:      nginx:1.9.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```

### 以前の履歴に巻き戻す（ロールバック） {#rolling-back-to-a-previous-revision}

ここでは現在の展開を取り消し、以前の履歴に巻き戻すのを決めました。

```shell
$ kubectl rollout undo deployment/nginx-deployment
deployment "nginx-deployment" rolled back
```

あるいは、 `--to-revision` で特定の履歴（リビジョン）にロールバックもできます：

```shell
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment "nginx-deployment" rolled back
```

展開（ロールアウト）に関連する詳しい情報は、 [`kubectl rollout`](/jp/docs/reference/generated/kubectl/kubectl-commands#rollout) をご覧ください。

これでデプロイメントは以前の安定版に巻き戻りました。
`DeploymentRollback` イベントを見ると、デプロイメント・コントローラによって作成されたリビジョン２に巻き戻ったのが分かります。

```shell
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           30m

$ kubectl describe deployment
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       3 updated | 3 total | 3 available | 0 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     <none>
NewReplicaSet:      nginx-deployment-1564180365 (3/3 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  30m       30m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-1564180365 to 2
  2m        2m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-3066724191 to 0
  2m        2m          1       {deployment-controller }                Normal      DeploymentRollback  Rolled back deployment "nginx-deployment" to revision 2
  29m       2m          2       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
```

## デプロイメントのスケーリング {#scaling-a-deployment}

以下のコマンドを使ってデプロイメントをスケールできます：

```shell
$ kubectl scale deployment nginx-deployment --replicas=10
deployment "nginx-deployment" scaled
```

クラスタ内で [水平ポッド・オートスケーリング（horizontal pod autoscaling）](/jp/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) が有効と仮定すると、デプロイメントに対してオートスケーラ（autoscaler）をセットアップできます。
そして、稼働しているポッドの CPU 使用率に基づいて動くよう、ポッドの最小数と最大数を選択できます。

```shell
$ kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu-percent=80
deployment "nginx-deployment" autoscaled
```

### プロモーショナル（比例）・スケーリング {#proomtional-scaling}

RollingUpdate（ローリング・アップデート）デプロイメントは、アプリケーションの複数バージョン同時実行をサポートします。
自分でもしくはオートスケーラによって、ロールアウト中に（進行中もしくは一次停止中だとしても）RollingUpdate デプロイメントをスケールする場合、移行のリスク（危険性）があるため、デプロイメント・コントローラは追加の複製と既存のアクティブな ReplicaSet （ポッドとレプリカセット）バランスを取ります。
これを *「プロモーショナル（比例）・スケーリング」（promotional scaling）* と呼びます。

たとえば、10の複製があるデプロイメントを実行中で、 [maxSurge](#max-surge)=3 、 [maxUnavailable](#max-unavailable)=2 とします。

```shell
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s
```

クラスタの内部で解決不可能な新しいイメージに更新します。

```shell
$ kubectl set image deploy/nginx-deployment nginx=nginx:sometag
deployment "nginx-deployment" image updated
```

イメージの更新が始まり、レプリカセット nginx-deployment-1989198191 を新規に展開（ロールアウト開始）しますが、先ほど言及したように、 `maxUnavailable` によってブロックされます。

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```

それから、デプロイメントに対する新しいスケーリング要求が発生します。
オートスケーラはデプロイメントの複製を 15 に増やします。
デプロイメント・コントローラは、どこに新しい５つの複製を追加するか決める必要があります。
もしも、プロポーショナル（比例）・スケーリングを使用していなければ、５つすべてを新しいレプリカセットに追加します。
プロポーショナル（比例）・スケーリングに対応していれば、追加の複製を全てのレプリカセットを横断して展開します。
比率が大きければレプリカ・セットは複製を増やし、比率が低ければレプリカ・セットは複製を減らします。
複製が余ったら、複製が多いレプリカ・セットに追加されます。
レプリカ・セットのレプリカがゼロであれば、スケールアップしません。

先ほどの例では、３つの複製は古いレプリカ・セットに追加され、２つの福祉が新しいレプリカセットに追加されます。
この展開作業は、最終的には全ての複製が新しいレプリカ・セットに移動し、すべての複製が正常（healthy）になったとみなされます。

```shell
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```

## デプロイメントの一次停止と再開 {#pausing-and-resuming-a-deployment}

１つまたは複数の更新をトリガとして、デプロイメントの一次停止（pause）と再開（resume）が可能です。
これによって、不要なロールアウトをトリガとすることなく、一次停止と再開を複数適用できるようにします。

たとえば、デプロイメントを作成していたとします。

```shell
$ kubectl get deploy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     3         3         3            3           1m
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         1m
```

一次停止するには以下のコマンドを実行します：

```shell
$ kubectl rollout pause deployment/nginx-deployment
deployment "nginx-deployment" paused
```

それから、デプロイメントのイメージを更新します。

```shell
$ kubectl set image deploy/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated
```

新しいロールアウトは開始されないのに注目します：

```shell
$ kubectl rollout history deploy/nginx-deployment
deployments "nginx"
REVISION  CHANGE-CAUSE
1   <none>

$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         2m
```

必要があれば、どれだけでも更新できます。たとえば、リソースを更新するために使うには：

```shell
$ kubectl set resources deployment nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
deployment "nginx-deployment" resource requirements updated
```

デプロイメントの初期状態は、機能によって一次停止されていたとしても、デプロイメントに対する新しい更新はデプロイメントが停止している限り何ら影響を与えません。

最終的には、デプロイメントは再開し、新しいレプリカ・セットが全ての新しい更新を見つけ出します。


```shell
$ kubectl rollout resume deploy/nginx-deployment
deployment "nginx" resumed
$ kubectl get rs -w
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   2         2         2         2m
nginx-3926361531   2         2         0         6s
nginx-3926361531   2         2         1         18s
nginx-2142116321   1         2         2         2m
nginx-2142116321   1         2         2         2m
nginx-3926361531   3         2         1         18s
nginx-3926361531   3         2         1         18s
nginx-2142116321   1         1         1         2m
nginx-3926361531   3         3         1         18s
nginx-3926361531   3         3         2         19s
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         20s
^C
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         28s
```

{{< note >}}
**メモ：** 一時停止中のデプロイメントは再開しない限り、巻き戻し（ロールバック）できません。
{{< /note >}}

## デプロイメント状態（ステータス）  {#deployment-status}

デプロイメントは、そのライフサイクルで様々な状態に遷移します。
新しいレプリカ・セットの展開に応じて、[完了](#complete-deployment) または[処理失敗](#failed-deployment)といった [プロセッシング（processing：手続き／処理）](#progressing-deployment) が可能です。

### デプロイメントの進行中（Progressing Deployment） {#progressing-deployment}

以下のタスクのうち１つでも処理する時、Kubernetes はデプロイメントを _progressing（進行中）_　にマークします：

* デプロイメントは新しいレプリカ・セットを作成
* デプロイメントは最も新しいレプリカ・セットにスケールアップ
* デプロイメントは古いレプリカ・セットにスケールダウン
* 新しいポッドが利用可能になる（少なくとも [MinReadySeconds（最小待機秒（](#min-ready-seconds) を待機）

デプロイメントの進捗は `kubectl rollout status` で監視できます。

### デプロイメントの完了（Complete Deployment） {#complete-deployment}

以下の特徴があれば、Kubernetes はデプロイメントを _complete（完了）_ とマークします。

* デプロイメントに関連付けられたすべての複製が指定されたバージョンに更新されている。つまり、リクエスト済みのあらゆる更新が完了。
* デプロイメントに関連付けられたすべての複製（レプリカ）が利用可能。
* デプロイメントに対する古い複製が一切動作していない。

デプロイメントが完了したかどうかは `kubectl rollout status` を使って確認できます。
もしもロールアウトが完了していたら、 `kubectl rollout status` はゼロの終了コードを返します。

```shell
$ kubectl rollout status deploy/nginx-deployment
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx" successfully rolled out
$ echo $?
0
```

### デプロイメント失敗（Failed Deployment） {#failed-deployment}

最も新しいレプリカ・セットが完了しなければ、デプロイメントはデプロイの試みに行き詰まります。
以下の要素のいくつかによって、このような状況が発生します：

* 容量制限（クォータ）が足りない
* 読込性診断の失敗
* イメージ取得（pull）エラー
* 権限（パーミッション）が足りない
* 範囲の制限
* アプリケーション・ランタイムの設定ミス

この状況を検出する１つの方法は、デプロイメント spec でデッドライン・パラメータ（deadline parameter）の指定 （[`.spec.progressDeadlineSeconds`](#progress-deadline-seconds)） です。
`.spec.progressDeadlineSeconds` が意味するのは、デプロイメント処理の失速が始まってから、デプロイメント・コントローラが待機する秒数です。

以下の `kubectl` コマンドは spec に `progressDeadlineSeconds` を指定します。
これは、デプロイメントの進捗残りが10分以下になれば、がコントローラに対して報告させます。

```shell
$ kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
deployment "nginx-deployment" patched
```

デッドラインに到達すると、デプロイメント・コントローラはデプロイメントの `.status.conditions` に対して `DeploymentCondition` と以下の属性を追加します。

* Type=Progressing（処理進行）
* Status=False（失敗）
* Reason=ProgressDeadlineExceeded（処理がデッドラインに到達）

ステータス状況に関する詳しい状況は [Kubernetes API conventions](https://git.k8s.io/community/contributors/devel/api-conventions.md#typical-status-properties) をご覧ください。

{{< note >}}
**メモ：** Kubernetes は固まったままのデプロイメントに対して何も行動を起こさず、ステータス状況で `Reason=ProgressDeadlineExceeded` を示すだけです。
高度なオーケストレータであれば、これをうまく利用し、適切に処理するでしょう。
たとえば、デプロイメントを以前のバージョンに巻き戻すなどです。
{{< /note >}}

{{< note >}}
**メモ：** もしもデプロイメントを一次停止（pause）すると、Kubernetes は指定したデッドラインに対する進捗を確認しません。
ロールアウトの途中でデプロイメントの一次停止を安全に行い、状況のトリガがデッドラインに到達するまでに再開（resume）できます。
{{< /note >}}

デプロイメントに対しては一時的なエラーが出るかもしれません。
それは、指定したタイムアウト時間を経過したか、他の何らかのエラーの発生によるものです。
これらのエラーは短期的なもの（transient）として扱われます。

```shell
$ kubectl describe deployment nginx-deployment
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

`kubectl get deployment nginx-deployment -o yaml` を実行すると、デプロイメントのステータスはこのようになります：

```
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

最終的には、デプロイメント進捗デッドラインに到達すると、Kubernetes は状態（ステータス）と進捗状況（Progressing condition）の理由を更新します：

```
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     False   ProgressDeadlineExceeded
  ReplicaFailure  True    FailedCreate
```

デプロイメントのスケールダウンにより、不十分なクォータ（容量割り当て）があれば問題を発生します。
これは実行中の他のコントローラをスケールダウンしてしまうか、あるいは、自分の名前空間内でクォータ（容量割り当て）を増やすと発生します。
クォータ状態とデプロイメント・コントローラを充足するには、デプロイメントの展開（ロールアウト）を完了します。
そうすると、デプロイメントのステータスは成功状態（ `Status=True` and `Reason=NewReplicaSetAvailable`）へと更新されます。

```
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
```

`Type=Available` にある `Status=True` の意味は、デプロイメントが最小の可用性を持っています。
最小の可用性を決めるのは、デプロイメント方針（strategy：ストラテジ）で指定するパラメータです。
`Type=Progressing` にある `Status=True` の意味は、デプロイメントが展開（ロールアウト）の途中であり処理が進行中か、あるいは、進捗の完了に成功し、最小限必要な新しい複製が利用可能となっています（個々の状況における理由をご覧ください。今回の例では `Reason=NewReplicaSetAvailable` は、デプロイメントの完了を意味します）。

デプロイメントの進行が失敗したかどうかを調べるには、 `kubectl rollout status` を使います。
`kubectl rollout status` が０以外の終了コードを返す場合、デプロイメントは処理のデッドラインへ到達しています。

```shell
$ kubectl rollout status deploy/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "nginx" exceeded its progress deadline
$ echo $?
1
```

### デプロイメントが失敗した場合の操作 {#operating-on-a-failed-deployment}

全てのアクションはデプロイメントに完全適用されるだけでなく、デプロイメントへの適用失敗もあります。
スケールアップ・ダウンや以前のバージョンに戻したり、一次停止にあたっては、デプロイメント・ポッド・テンプレートの調整が必要になります。

## クリーンアップ方針 {#clean-up-policy}

デプロイメントで古いレプリカ・セットをいくつ保持する（retain）かは、デプロイメントの `.spec.revisionHistoryLimit` フィールドで指定できます。
何も処理が発生しなければ、ガベージコレクション（garbage-collected）がバックグラウンドで進行します。
デフォルトでは 10 です。

{{< note >}}
**メモ** このフィールドを０と明示すると、デプロイメントの履歴もクリーンアップします。その結果、デプロイメントはロールバックできなくなります。

{{< /note >}}

### 使用例

### カナリア・デプロイメント（Canary Deployemnt{}）

ユーザまたはデプロイメントによって用いられるサーバのサブセット（一部）を展開（ロールアウト）するにあたり、複数のデプロイメントを作成できます。
それぞれのリリースごとに複数のデプロイメントを作成できます。カナリア・パターンの詳細んついては [リソースの管理](/jp/docs/concepts/cluster-administration/manage-deployment/#canary-deployments) をご覧ください。

## デプロイメント spec を書く {#writing-a-deployment-spec}

他の Kubernetes 設定と同様に、デプロイメントは `apiVersion` 、 `kind` 、 `metadata` フィールドが必要です。
設定ファイルに関する共通情報については、 [アプリケーションのデプロイ](/jp/docs/tutorials/stateless-application/run-stateless-application-deployment/)、 [リソース管理のために kubectl を使う) コンテナの設定のドキュメントをご覧ください。

また、デプロイメントには [`.spec` セクション](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status) が必要です。

### ポッド・テンプレート {#pod-template}

`.spec.template` は `.spec` で唯一必要なフィールドです。

`.spec.template` は [ポッド・テンプレート](/jp/docs/concepts/workloads/pods/pod-overview/#pod-templates) です。
実際には [ポッド](/jp/docs/concepts/workloads/pods/pod/) と同じスキーマですが、ネスト化でき `apiVersion` や `kind` を持たないのを除外します。

ポッドに対する必要なフィールドに加え、デプロイメントでのポッド・テンプレートには適切なラベルと適切な再起動ポリシーの指定が必須です。
ラベルについては、他のコントローラとは重複しないようにしてください。詳細は [セレクタ](#selector) をご覧ください。

[`.spec.template.spec.restartPolicy`](/jp/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) を指定しなければ、デフォルトでは常に `Always` が指定されたものと同等になります。

### 複製（レプリカ） #replicas

`.spec.replicas` は期待するポッド（desired Pods）の数を指定する、オプションのフィールドです。デフォルトは１です。

### セレクタ {#selector}

`.spec.selector` はオプションのフィールドであり、デプロイメントにおいて対象となるポッドを [ラベル・セレクタ（label selector）](/jp/docs/concepts/overview/working-with-objects/labels/) を指定します。

`.spec.selector` は `.spec.template.metadata.labels` と一致する必要があります。そうでなければ、API によって拒否されます。

API バージョン `apps/v1` では、 `.spec.selector` と `.metadata.labels` がセットされないため、デフォルトで `.spec.template.metadata.labels` が設定されます。
そのため、これらは明示する必要があります。
また、`apps/v1` では `.spec.selector` はデプロイメントの作成後に変更できない（イミュータブル）のでご注意ください。

デプロイメントはポッドによって終了させられる場合があります。
それは、デプロイメントのラベルがセレクタと一致するときや、`.spec.template` とテンプレートのセレクタが一致しない場合、 ポッドが  `.spec.replicas` の合計数を上回る場合です。
期待数よりもポッドの数が少なくなれば、 `.spec.template` に従って新しいポッドを立ち上げます。

{{< note >}}
**メモ:** セレクタと一致するラベルを作成すべきではありません。それだけでなく、直接または他のデプロイメントによる作成、レプリカ・セットやレプリケーション・コントローラなどによって作成されるデプロイメントもです。
もし作成すると、１つめに作成したデプロイメントが他のポッドを作成したと認識してしまいます。
Kubernetes はこれを止められません。
{{< /note >}}

もしもセレクタが重複する複数のコントローラが有る場合には、コントローラがお互いに衝突し、正しい動作をしない可能性があります。

### ストラテジ（方針） {#strategy}

`.spec.strategy` とは、古いポッドを新しいポッドに置き換えるときに使うストラテジ（方針）です。
`.spec.strategy.type` は、デフォルト値 "Recreate"、"RollingUpdate"、"RollingUpdate" を更新可能です。

#### デプロイメントの再作成 {#recreate-deployment}

`.spec.strategy.type==Recreate` を指定すると、全ての既存のポッドは、新しいポッドを作成する前に削除します。

### ローリング・アップデート・デプロイメント {#rolling-update-deployment}

デプロイメントは `.spec.strategy.type==RollingUpdate` の指定があれば、ポッドを[ローリング・アップデート](/jp/docs/tasks/run-application/rolling-update-replication-controller/)でアップデートします。
ローリング・アップデートの処理を管理するために、 `maxUnavailable` と `maxSurge` を指定できます。

##### 最大利用不可数（Max Unavailable） {#max-unavailable}

`.spec.strategy.rollingUpdate.maxUnavailable` はアップデート処理中に利用できなくなるポッドの最大数を指定するための、オプションのフィールドです。
この値は絶対値（例、5）か期待ポッドのパーセンテージ（例、10% ）で指定します。
パーセンテージによって計算される絶対値は、端数を切り捨てます。
もし `.spec.strategy.rollingUpdate.maxSurge` が 0 であっても、この値は 0 になりません。
デフォルトの値は 25% です。

たとえば、この値を 30% に指定すると、ローリング・アップデートを開始すると、古いレプリカ・セットは期待ポッドの 70% まで直ちにスケールダウンします。
新しいポッドの準備が整えば、古いレプリカ・セットは更にスケールダウンします。
あわせて、新しいレプリカ・セットがスケールアップします。
この時、利用可能なポット数の合計は、アップデート期間中は常に期待ポッド数の少なくとも 70% を維持し続けようとします。

### Max Surge（最大サージ／超過） {#mas-surge}

`.spec.strategy.rollingUpdate.maxSurge` はポッドの期待数を越えて作成可能な最大ポッド数を指定するための、オプションのフィールドです。
この値は絶対値（例、5）か期待ポッドのパーセンテージ（例、10% ）で指定します。
パーセンテージによって計算される絶対値は、端数を切り捨てます。
もし `MaxUnavailable` が 0 であっても、この値は 0 になりません。
デフォルトの値は 25% です。

たとえば、この値を 30% に指定すると、ローリング・アップデートを開始すると、新しいレプリカ・セットは新旧ポッドの合計数が、期待ポッドの 130% に到達するまで直ちにスケールアップします。
古いポッドが停止（kill）されれば、新しいレプリカセットはさらにスケールアップします。
この時、利用可能なポット数の合計は、アップデート期間中は常に期待ポッド数の最大で 130% を維持し続けようとします。


### 進捗デッドライン秒（Progress Deadline Seconds） {#progress-deadline-seconds}

`.spec.progressDeadlineSeconds` はデプロイメントが [処理に失敗（failed progressing（](#failed-deployment)  したとシステムが報告を返すまで、デプロイメントが進捗するのを待つ秒数を指定するための、オプションのフィールドです。
状態を表示すると、`Type=Progressing` と `Status=False` 、および `Reason=ProgressDeadlineExceeded` がリソースのステータスに現れます。
デプロイメント・コントローラはデプロイメントの再試行を維持します。
将来的には、自動ロールバックが実装され、デプロイメント・コントローラはデプロイメントのロールバックを状況の変化に応じて行えるようになります。

このフィールドを指定するにあたっては、 `.spec.minReadySeconds` よりも大きな値が必要です。

### 最小待機秒（Min Ready Seconds）{#min-ready-seconds}

`.spec.minReadySeconds` は最近作成されたポッドがコンテナのキャッシュのために待機し、準備が整ったとみなす最小の秒数を指定するオプションのフィールドです。
これはデフォルトでは 0 です（ポッドは間もなく利用可能になるとみなします）。
ポッドの準備が整ったとみなす条件については、  [Container Probes](/jp/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) をご覧ください。

### ロールバック先（Rollback To） {#rollback-to}

フィールド  `.spec.rollbackTo` は API バージョン `extensions/v1beta1` と `apps/v1beta1` で廃止されました。
そして、 `apps/v1beta2` 移行ではサポートしなくなりました。
そのかわりに、 `kubectl rollout undo` によってサポートされた [以前のバージョンへのロールバック](#rolling-back-to-a-previous-revision) を使うべきです。

### リビジョン履歴上限（Revision History Limit） {#revision-history-limit}

デプロイメントのリビジョン履歴はコントローラのレプリカ・セットに保管されます。

`.spec.revisionHisoryLimit` は巻き戻し（ロールバック）できるように保持し続ける古い ReplicaSet の数を指定する、オプション・フィールドです。
この理想的な値は、新しいデプロイメントによる繰り返しと安定性に依存します。
もしこのフィールドの値を指定しなければ、全ての古いレプリカセットはデフォルトで維持されますが、 `etcd` のリソースを消費し、 `kubectl get rs` の出力から混み合っているのがわかります。
各デプロイメントのリビジョンを保管する設定はレプリカ・セットにあります。
つまり、古いレプリカ・セットが削除されれば、そこに格納されているデプロイメントのリビジョンにロールバックできなくなります。

さらに具体的に書くと、このフィールドをゼロにすると、全ての古いレプリカ・セットは０複製となり、クリーンアップされるのを意味します。この例では、リビジョン履歴がクリーンアップされるまで、新しいデプロイメントのロールアウトは完了しません。

### 一次停止（Paused） {#paused}

`.spec.paused` はデプロイメントの一次停止と再会のためのオプション・ブール演算（boolean）です。
一次停止したデプロイメントと停止していないデプロイメントとの違いとは、一次停止している限り新しいロールアウトをトリガとしません。
デプロイメントの作成時、デフォルトでは一次停止しません。

## デプロイメントの代替 {#alternative-to-deployments}

### kubectl rolling update

[`kubectl rolling update`](/jp/docs/reference/generated/kubectl/kubectl-commands#rolling-update) はポッドとレプリケーション・コントローラを同様の手法で更新します。
しかし、デプロイメントを使う方が推奨です。なぜならば、こちらは宣言型であり、サーバサイドであり、ローリング・アップデート後もあらゆる以前のバージョンに巻き戻せるロールバックのような追加機能があるからです。

{{% /capture %}}


