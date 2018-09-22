---
title: ReplicationController（レプリケーション・コントローラ）
content_template: templates/concept
weight: 20
---

{{% capture overview %}}

{{< note >}}
**メモ：**  複製（レプリケーション）をセットアップするには、[`Deployment`（デプロイメント）](/jp/docs/concepts/workloads/controllers/deployment/) の使用が [`ReplicaSet`（レプリカ・セット）](/jp/docs/concepts/workloads/controllers/replicaset/) よりも推奨される手法です。
{{< /note >}}

_ReplicationController_ （レプリケーション・コントローラ）は指定した数のポッド複製（レプリカ）の常時実行を確実に行います。
言い換えると、ReplicationController はポッドまたはポッドと同質のものを常に起動かつ利用可能にします。

{{% /capture %}}


{{% capture body %}}

### ReplicationController の挙動 {#how-a-replicationcontroller-works}

大量のポッドが存在すると、ReplicationController は余分なポッドを終了します。
あまりにも少なければ、ReplicationController は追加ポッドを起動します。
手動でポッドを作成するのとは異なり、ポッドの維持は ReplicationController によって行われ、もしも障害があれば自動的に置き換えますし、自動的に削除や終了を処理します。
たとえば、ノードがカーネルのアップグレードのような破壊的なメンテナスをした後に、ポッドを自動的に再作成します。
そのため、アプリケーションが必要なのはポッド１つだとしても、ReplicationController を使うべきでしょう。
ReplicationController はプロセス・スーパーバイザと似ていますが、１つのノード上の個々のプロセスを監視するスーパーバイザとは異なり、ReplicationController は複数のノードを横断する複数のポッドを監視（スーパーバイズ）します。

ReplicationController は議論において頻繁に「rc」や「rcs」と省略されます。
また、同様に kubectl コマンドでもショートカットとして使えます。

簡単な利用例は、１つの ReplicationController オブジェクトを作成し、直ちにポッドの中で１つのインスタンスを確実に実行することです。
より複雑な利用例は、ウェブサーバのようなサービスを複製するために、複数の全く同じ複製（レプリカ）を実行します。

## ReplicationController 実行例 {#running-an-example-replicationcontroller}

こちらはnginx ウェブサーバの３つのコピーを ReplicationController で設定する例です。

{{< codenew file="controllers/replication.yaml" >}}

サンプルのジョブを実行するには、こちらのコマンドを実行し、サンプルファイルをダウンロードして実行します：

```shell
$ kubectl create -f https://k8s.io/examples/controllers/replication.yaml
replicationcontroller "nginx" created
```

ReplicationController の状態を確認するにはこちらのコマンドを使います:

```shell
$ kubectl describe replicationcontrollers/nginx
Name:        nginx
Namespace:   default
Selector:    app=nginx
Labels:      app=nginx
Annotations:    <none>
Replicas:    3 current / 3 desired
Pods Status: 0 Running / 3 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       app=nginx
  Containers:
   nginx:
    Image:              nginx
    Port:               80/TCP
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen       LastSeen     Count    From                        SubobjectPath    Type      Reason              Message
  ---------       --------     -----    ----                        -------------    ----      ------              -------
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-qrm3m
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-3ntk0
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-4ok8v
```

こちらでは、３つのポッドが作成されましたが、まだ起動中ではありません。
おそらく、イメージを取得しているからでしょう。もうしばらく待ってから同じコマンドを実行すると、次のように表示されるでしょう:

```shell
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

マシン上で ReplicationController に所属している全ポッドの一覧を表示するには、次のようなコマンドを使えます：

```shell
$ pods=$(kubectl get pods --selector=app=nginx --output=jsonpath={.items..metadata.name})
echo $pods
nginx-3ntk0 nginx-4ok8v nginx-qrm3m
```

ここでは、セレクタは ReplicationController と同じセレクタにしています。
`kubectl describe` の出力で見られますが `repliation.yaml` とは違う形式になっています。
`--output=jsonpath` オプションを指定して、各ポッドごとの名前をリストにして表示できます。

## ReplicationController Spec を書くには {#writing-a-replicationcontroller-spec}

他すべての Kubernetes 設定と同様に、 ReplicationController には `apiVersion` 、 `kind` 、 `metadata` フィールドが必要です。
設定情報ファイルの役割に関する一般的な情報は、[オブジェクト管理](/jp/docs/concepts/overview/object-management-kubectl/overview/) をご覧ください。

また、ReplicationController には  [`.spec` セレクション](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status) が必要です。

### ポッド・テンプレート {#pod-template}

`.spec.template` に最低限必要なフィールドは `.spec` です。

`.spec.template` は [ポッド・テンプレート](/jp/docs/concepts/workloads/pods/pod-overview/#pod-templates) です。
これは [ポッド](/jp/docs/concepts/workloads/pods/pod/) と完全に同じスキーマですが、階層化されておらず、 `apiVersion` や `kind` を持ちません。

ポッドに対して必要なフィールドとしては、ReplicationController のポッドテンプレートでは、適切なラベルと適切な再起動方針（restart policy）の指定が必須です。
ラベルの場合、他のコントローラと重複しないようにする必要があります。
[ポッド・セレクタ](#pod-selector) をご覧ください。

[`.spec.template.spec.restartPolicy`](/jp/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) のデフォルトが指定されていなければ、常に `Always` が許可されたものとみなします。

ローカル・コンテナの再起動については、ReplicationControllers はノード上のエージェントに権限を委譲しています。
例えば [Kubelet](/jp/doc/admin/kubelet/) や Docker です。

### ReplicationController 上のラベル {#labels-on-the-replicationcontroller}

ReplicationController は自身のラベル（ `.metadata.labels`）を持てます。
たいていは、`.spec.template.metadata.labels` と同じものを設定します。
つまり `.metadata.labels` を指定しなければ、デフォルトで `.spec.template.metadata.labels` となります。
しかしながら、異なったものを指定できますし、 `.metadata.labels` は ReplicationController の挙動に対して影響を与えません。

### ポッド・セレクタ {#pod-selector}

`.spec.selector` フィールドは  [ラベル・セレクタ](/jp/docs/concepts/overview/working-with-objects/labels/#label-selectors) です。
ReplicationController はセレクタが一致するラベルを持つポッドすべてを管理します。
これはポッド間の違いについては認識しません。つまり、ポッドが他人またはプロセスによって作成されたか削除されたかは識別しません。
これにより、ReplicationController は実行中のポッドに影響を与えることなく置き換えが可能です。

`.spec.template.metadata.labels` を指定する場合は、 `.spec.selector` と一緒にする必要があります。そうしないと API から拒否されます。
もしも `.spec.selector` を指定しなければ、`.spec.template.metadata.labels` がデフォルトとなります。

また、通常はラベルがセレクタと一致するポッドを作成すべきではありません。
どうしても必要があれば、他の ReplicationController を直接使うか、ジョブのような他のコントローラを使います。
そうしておけば、ReplicationController は他のポッドが作成されたと考えます。
こうすることで、Kubernetes は作成したものを停止しません。

もしも、最終的に複数のコントローラを使うことになれば、セレクタが重複するでしょう。
そのような場合は、自分自身で削除を管理する必要があります（詳細は [以下](#working-with-replicationcontrollers)）。

### 複数の複製{#multiple-replicas}

`.spec.replicas` の指定によって、並列に実行すべきポッドの数を指定できます。
ポッド数の設定はいつでも上下できますので、複製（レプリカ）を常に増加または減少できます。
あるいは、ポッドに対する丁寧なシャットダウンや、早期の複製の作成が可能です。

`.spec.replicas` を指定しなければ、デフォルトは 1 になります。

## ReplicationControllers と連携 {#working-with-replicationcontrollers}

### ReplicationControllers と中にあるポッドを削除 {#deleting-a-replicationcontroller-and-its-pods}

ReplicationController と中にあるポッドを削除するには、 [`kubectl delete`](/jp/docs/reference/generated/kubectl/kubectl-commands#delete) を使います。
Kubectl は ReplicationController で規模を０に変更できます。そして、ReplicationController 自身を削除する前に、各ポッドの削除のために待機します。
もしも kubectl コマンドが中断されれば、再起動できます。

REST API や go クライアント・ライブラリを使う場合は、このステップを明示的に行う必要があります（複製数を０にし、ポッドの削除まで待機し、それから ReplicationController を削除）。

### ReplicationController のみ削除 {#deleting-just-a-replicationcontroller}

あらゆるポッドに影響を与えず ReplicationController だけを削除できます。

kubectl を使い、[`kubectl delete`](/jp/docs/reference/generated/kubectl/kubectl-commands#delete) のオプションに `--cascade=false` を指定します。

REST API や go クライアント・ライブラリを使う場合は、単純に ReplicationController オブジェクトを削除するだけです。

一度元からある（オリジナルの） ReplicationController  を削除すると、新しいものを作成して置き換え可能です。
新旧の `.spec.selector` が同じで有る限り、新しいポッドは古いポッドのものを受け入れます（採用します）。
しかしながら、異なったポッド・テンプレートを使わない限り、既存のポッドを新しいものと一致させるには大変ではありません。
ポッドを新しい spec に更新するには管理された手法 [ローリング・アップデート](#rolling-update) を使います。

### ポッドを ReplicationController から独立（分離） {#isolating-pods-from-a-replicationcontroller}

ポッドを ReplicationController の対象から外したい時は、ポッドに割り当てられているラベルを変更します。
この手法はサービスのデバッグやデータ修復等のためにポッドを削除するために役立つでしょう。
ポッドをこの手法で削除すると、自動的に置き換えられます（前提として複製数も変更していない場合）。

## 共通の利用パターン {#common-usage-patterns}

### 再スケジュール {#rescheduling}

先ほど言及したように、１つのポッドを実行し続けたい場合、あるいは 1000 に増やしたい場合でも、ReplicationController は指定したポッド数が確実に存在するようにします。
たとえ、ノード障害やポッドの終了が発生してもです（例えば、他のコントローラ・エージェントによる影響を受けてもです）。

### 規模変更（スケーリング） {#scaling}

ReplicationController は複製数を増減して規模変更（スケール）をしやすいようにします。
複製数の変更は手動だけでなく、オートスケーリング（自動規模調整）制御エージェントによっても行えます。
その場合は単純に `replicas` （複製）フィールドを更新するだけです。

### ローリング・アップデート（逐次更新） {#rolling-update}

ReplicationController はサービスに対して１つ１つのポッドを置き換えるローリング・アップデート（逐次更新）を調整するために設計されています。

[#1353](http://issue.k8s.io/1353) で説明があるように、推奨する手法は、新しい ReplicationController が１つの複製があるなら、新しいものをスケールし(+1)、それから古いコントローラの削除(-1)するのを１つ１つ行います。
古いコントローラを削除後、複製数は０になります。
この予測可能な更新は、ポッドの集まりは、予期せぬ障害が起こったとしても行えます。

理想的には、ローリング・アップデート（逐次更新）コントローラは、アプリケーションの読込性を報告しながら、常に十分な数のポッドを確実に維持します。

２つの ReplicationController でポッドを作成するには、少なくとも１つはラベルを変える必要があります。
ラベルではポッドで使われる主なコンテナのイメージ・タグなどです。
これは、イメージのローリング・アップデート（逐次更新）のために使うのが典型的です。

ローリング・アップデート（逐次更新）はクライアント・ツール [`kubectl rolling-update`](/jp/docs/reference/generated/kubectl/kubectl-commands#rolling-update) で実装されています。
より明確な例は  [`kubectl rolling-update` タスク](/jp/docs/tasks/run-application/rolling-update-replication-controller/) をご覧ください。

### 複数のリリース・トラック（Multiple release tracks） {#multiple-release-tracks}

アプリケーションの複数のリリースを提供するにあたって、ローリング・アップデート（逐次更新）を実行できるのに加えて、長期間にわたって複数のリリースを継続するのは一般的です。
あるいは、継続的に複数のリリース・トラック（軌道）を使います。トラック（軌道）はラベルとは異なります。

例として、サービスがすべてのポッドが  `tier in (frontend), environment in (prod)` （「フロントエンド」層の「プロダクション」環境）と仮定します。
この層は10の複製されたポッドによって構成されています。
ここで、構成要素（コンポーネント）の新しいバージョン「canary」（カナリア）を有効にしたいとします。
ReplicationController を使って `replicas` （複製）の集まりから、 9 つあるラベルを `tier=frontend, environment=prod, track=stable` とします（層＝フロントエンド、環境＝プロダクション、トラック＝stable）に設定します。
そして、ReplicationController では `replicas` をカナリア用に 1 を設定し、ラベルは  `tier=frontend, environment=prod, track=canary` とします（層＝フロントエンド、環境＝プロダクション、トラック＝canary）。
これでサービスを canary と canary ではないポッドを扱います。
しかし ReplicationControllers を分けているため、混乱を引き起こす可能性があるため、テストを行い、結果の監視などが必要になります。

### ReplicationController をサービスに使う {#using-replicationcontrollers-with-services}

複数の ReplicationController を１つのサービスの後ろに添えられます。たとえば、トラフィックを古いバージョンに流しますが、いくつかを新しいバージョンに流すことが可能です。

ReplicationController は自分自身を決して停止させません。
しかし、長期間にわたってサービスとしての稼働は予想されていません。
サービスはポッドの集合であり、複数の ReplicationController によって管理されるものです。
そのため、多くの ReplicationController が作成され、サービスのライフタイムを越えると破棄される可能性があります（例えば、サービスを動かしながら、ポッドの性能をアップデートする場合）。
サービス自身とクライアントの両方が ReplicationController の存在を記憶しません。これはサービスのポッドを運用（維持：maintain）するためです。

## 複製（レプリケーション）用のプログラムを書く {#writing-programs-for-replication}

ReplicationController によって作成されるポッドとは、代替可能かつ意味的にはどちらも同一なものを意図していますが、時間が経てば経つほど、設定ファイルは異質なものになる場合があるでしょう。
これは複製されたステートレスなサーバの用途と明確に一致します。しかしまた、ReplicationController はマスタ選出（master-elected）、sharded（断片化）、ワーカーをプールするアプリケーションを運用するためにも利用できます。
このようなアプリケーションは動的な処理の割り当て機構を使うべきです。例えば [RabbitMQ ワーク・キュー（work queues）(https://www.rabbitmq.com/tutorials/tutorial-two-python.html) などです。
従って、ポッドに対して静的な／一度きりの設定のカスタマイズは行うべきではありません。これは、アンチ・パターンと考えられます。
あらゆるポッドに対するカスタマイズとは、リソースの水平オートスケーリング（例：CPU やメモリ）のような場合は、他のオンライン制御プロセスによって行われるべきであり、ReplicationController 自身では行われるべきではありません。

## ReplicationController の責務 {#responsibilities-of-the-replicationcontroller}

ReplicationController がシンプルに保証するのは、ラベル・セレクタに一致する必要な数のポッドを準備し、それらが操作可能なことです。
現時点では、停止したポッドのみをカウント対象から除外できます。
将来的には  [readiness（準備）](http://issue.k8s.io/620)や他のシステムで利用可能な情報でカウントできる可能性があります。
そのために私たちは複製ポリシーを越えた制御を行うかもしれません。また私たちは外部のクライアントが使うためのイベント発行、これは独自に洗練されたものに置き換える、あるいは、ポリシーをスケールダウンできるようにするつもりです。

ReplicationController が持つと考えられる責任範囲とは、永久に狭いままです。
自身では読込性診断や生存性診断を処理しません。
それよりもむしろ、オートスケーリングを処理するため、外部のオートスケーラによって `replicas` フィールドが変更可能な管理ができるように意図しています（[#492](http://issue.k8s.io/492)で議論中）。
私たちはスケジューリング・ポリシーを追加しないつもりです（たとえば、ReplicationController に対する [spreading（スプレッディング）](http://issue.k8s.io/367#issuecomment-48428019) ）。
ポッドであれば現在指定したテンプレートに一致するものしか制御しないどころか、オートサイジングや他の自動化プロセスに対する抽象化はありません。
同様に、コレクションのデッドライン、順番の依存性、設定情報の拡張、その他の機能が上げられます。
まっさらなポッドを作成時の要素や機構については、計画しています（([#170](http://issue.k8s.io/170)）。

ReplicationController によって構築ブロック・プリミティブを構成可能にするつもりです。
私たちは高レベルの API やツールがそれらの上で、あるいはユーザが役立つ他の補完的なプリミティブを将来的に導入する可能性があります
「マクロ」操作が現在サポートされているのは kubectl （run、scale、rolling-update）であり、これらの例は実証実験中（proof-of-concept）です。
たとえば、私たちは [Asgard](http://techblog.netflix.com/2012/06/asgard-web-based-cloud-management-and.html) のようなものが ReplicationControllers 、オートスケーラ、サービス、スケジューリング方針（ポリシー）、カナリア、等を管理するのを想像しています。

## API オブジェクト {#api-object}

レプリケーション・コントローラは Kubernetes REST API におけるトップ・レベルのリソースです。
API オブジェクトの詳細に関しては [ReplicationController API object](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#replicationcontroller-v1-core) をご覧ください。

## ReplicationController の代替 {#alternatives-to-replicationcontroller}

### ReplicaSet（レプリカセット）

[`ReplicaSet`](/jp/docs/concepts/workloads/controllers/replicaset/) は次世代の ReplicationController であり、新しい[set-based label selector（セットに基づくラベル・セレクタ）](/jp/docs/concepts/overview/working-with-objects/labels/#set-based-requirement)をサポートします。
ポッドの作成、削除、更新を調整（オーケストレート）する仕組みとして主に使われるのは [`Deployment`](/jp/docs/concepts/workloads/controllers/deployment/) です。

### Deployment（デプロイメント）（推奨） {#deployment-recommended}

[`Deployment`](/jp/docs/concepts/workloads/controllers/deployment/) は高レベルの API オブジェクトであり、根底にあるのはレプリカ・セットとそれらのポッドであり、これらが `kubectl rolling-update` と似たような手法を提供します。
ローリング・アップデート（逐次更新）機能を必要とする場合、Deployment が推奨されています。
これは `kubectl rolling-update` と同様に、これらは宣言型であり、サーバ・サイドであり、追加機能を備えるからです。

### Bare Pods（ベア・ポッド） {#barepods}

### Job（ジョブ） {#job}

ポッドに対して ReplicationController の代わりに  [`Job`](/docs/concepts/jobs/run-to-completion-finite-workloads/)  を使う方法は、自身で終了する挙動が期待されます（つまり、バッチ・ジョブです）。

### DamonSet（デーモンセット） {#daemonset}

ポッドに対して ReplicationController の代わりに [`DaemonSet`](/docs/concepts/workloads/controllers/daemonset/) を使う方法は、マシン・レベルの機能、たとえばマシン監視やマシンのログ記録を提供します。
各ポッドはマシンのライフタイムに強く結びつくライフタイム（生存期間）を持ちます。つまり、ポッドに必要なのはマシン上で他のポッドの起動前に実行することであり、マシンが再起動またはシャットダウンなど利用できなくなる前に、安全に停止できるようにします。

## 詳しい情報 {#for-more-information}

[ステートレス API レプリケーション・コントローラの実行](/jp/docs/tutorials/stateless-application/run-stateless-ap-replication-controller/) をご覧ください。


{{% /capture %}}


