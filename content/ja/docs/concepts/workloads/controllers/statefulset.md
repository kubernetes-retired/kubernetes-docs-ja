---
title: StatefulSets（ステートフル・セット）
content_template: templates/concept
weight: 40
---

{{% capture overview %}}
StatefulSet はステートフルなアプリケーションを管理するために使うワークロード API オブジェクトです。

{{< note >}}
**メモ：** StatefulSets はバージョン 1.9 から安定版（GA）です。
{{< /note >}}

ポッド（Pod） の集まりの展開（デプロイ）と拡大縮小（スケーリング）を管理し、 かつ、順序づけ（ordering）と一意性（uniqueness）の保証 をポッドに対して行います。

Deployment（デプロイメント） のように、StatefulSet は同一のコンテナ spec をベースにしているポッドを管理します。Deployment と違うのは、StatefulSet は各ポッドの厄介な自己同一性を管理します。ポッドは同じ spec から作成されていますが、お互いを交換できません。つまり、それぞれのポッドは再スケジューリングの間でも一貫した識別子を持ちます。

StetefulSet は他のあらゆる Controller（コントローラ）と同じパターンの元で稼働します。StatefulSet オブジェクト で期待状態（desired state）を定義し、 現在の状態からそこ（期待状態）に到達するよう必要な更新を StatefulSet コントローラ が行います。
{{% /capture %}}

{{% capture body %}}

## StatefulSet を使う {#using-statefulsets}

StatefulSet は以下の１つもしくは複数の要件があるアプリケーションに有益です。

* 安定した、ユニークなネットワーク識別子
* 安定した、持続的ストレージ（保管領域）
* 手入れの行き届いた、丁寧なデプロイメント（展開）とスケーリング（規模変更）
* 手入れの行き届いた、丁寧な削除と停止
* 手入れの行き届いた、自動化ローリング・アップデート（逐次更新）

前述のとおり、ポッドの（再）スケーリングにおいて、安定性と持続性（persistence）とは同義語です。
もしも、アプリケーションが安定した識別子、規則正しいデプロイメント（展開）、削除、スケーリング（規模変更）を必要としないのであれば、
アプリケーションのデプロイはコントローラを使うべきでしょう。
コントローラはステートレス（状態を持たない）な複製（レプリカ）の集まりを提供します。
ステートレス（状態を持たない）が必要であれば、[Deployment（デプロイメント）](/jp/docs/concepts/workloads/controllers/deployment/) や [ReplicaSet（レプリカ・セット）](/jp/docs/concepts/workloads/controllers/replicaset/) のようなコントローラのほうが適しているでしょう。

## 制限事項 {#Limitations}

* StatefulSet は 1.9 未満ではベータ・リソースであり、1.5 未満の Kubernetes では利用できません。
* 全てのアルファ/ベータのリソースと同様に、apiserver に渡すオプションで `--runtime-config` を指定すると、StatefulSet を無効化できます。
* ポッドに対して与えられるストレージは、 `storage class` の要求に基づく  [PersistentVolume Provisioner](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/persistent-volume-provisioning/README.md) によって供給（プロビジョン）されるか、管理者によって予め供給（プロビジョニング）済みか、どちらかの必要があります。
* StatefulSet の削除やスケーリング・ダウンでは、StatefulSet に関連付けられたボリュームを削除*しません* 。これはデータの安全性を確保するために行うものであり、StatefulSet リソースに関連する全てのリソースを自動的に切り離すよりも重要です。
* StatefulSet は、ポッドをネットワーク上で識別する役割ために現時点で [Headless Service](/jp/docs/concepts/services-networking/service/#headless-services) が必要です。あなたはサービスの作成に責任を持つ必要があります。

## 構成要素（コンポーネント） {#components}

以下の例は StatefulSet の構成要素を示します。

* nginx という名前の Headless Service（ヘッドレス・サービス） を、ネットワーク・ドメインの管理に使う
* web という名前の StatefulSet は、nginx コンテナの複製を３つを、それぞれユニークなポッドとして起動するSpec（仕様）があります。
* volumeClaimTemplates は PersistentVolume プロビジョナ（供給機能）による [PersistentVolumes（持続ボリューム）](/jp/docs/concepts/storage/persistent-volumes/)  を使う安定したストレージ（記憶領域）を提供します。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

## ポッド・セレクタ（Pod Selector）{#pod-selector}

StatefulSet の `.spec.selector` フィールドは、自身の `.spec.template.metadata.labels` ラベルと一致するように指定が必要です。
Kubernetes 1.8 未満では、 `.spec.selector` フィールドの省略はデフォルトでした。1.8 以降のバージョンでは、ポッド・セレクタの指定と一致しなければ、StatefulSet 作成時に整合性エラー（validation error）が起こり、作成に失敗します。

## ポッド同一性（Pod Identity） {#pod-identity}

StatefulSet のポッドはユニークな同一性（unique identity）があります。これを構成するのは順序、安定したネットワーク同一性、安定したストレージです。
識別子はポッドに張り付いているもので、どのノードに（再）スケジュールされても変わりません。

### オリジナル（原型）インデックス（Original Index） {#original-index}

StatefulSet に N の複製があるとすると、StatefulSet 内の各ポッドにはオリジナル（原型）としての整数が、0 から N-1 まで割り当てられます。これはセットを超えてユニークです。

### 安定したネットワーク ID （Stable Network ID） {#stable-network-id}

StatefulSet 内の各ポッドに提供するホスト名（hostname）は、StatefulSet の名前からと、ポッドのオリジナル（原型）からです。
作成されるホスト名のパターンは `$(statefulsetの名前)-$(オリジナル)` です。
戦術の例では、３つのポッドを作成しました。名前は `web-0、web-1、web-2` です。
StatefulSet は [ヘッドレス・サービス（Headless Service）](/jp/docs/concepts/services-networking/service/#headless-services) を使ってポッドのドメインを制御します。
このサービスによって管理されるドメインが取る形式は、 `$(サービス名).$(名前空間).svc.cluster.local` です。
「cluster.local」はクラスタンおドメインです。
各ポッドが作成されると、ここには DNS サブドメインに一致する名前をとり、形式は `$(ポッド名).$(管理対象のサービス・ドメイン)` となります。管理対象のサービスとは、StatefulSet の `serviceName` （サービス名）フィールドで定義されています。

こちらにあるのは、クラスタ・ドメイン、サービス名、StatefulSet 名を選んだ例です。そして、StatefulSet のポッドに対して、DNS 名がどのような影響を与えているかです。

クラスタ/ドメイン | サービス (名前空間/名前) | StatefulSet (名前空間/名前)  | StatefulSet ドメイン  | ポッド DNS | ポッド ホスト名 |
-------------- | ----------------- | ----------------- | -------------- | ------- | ------------ |
 cluster.local | default/nginx     | default/web       | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
 cluster.local | foo/nginx         | foo/web           | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local     | web-{0..N-1} |
 kube.local    | foo/nginx         | foo/web           | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local        | web-{0..N-1} |

クラスタ・ドメインが [設定されていなければ](/jp/docs/concepts/services-networking/dns-pod-service/#how-it-works)  `cluster.local` に設定されるのでご注意ください。

### 安定したストレージ（Stable Storage） {#stable-storage}

Kubernetes は、それぞれの VolumeClaimTemplate （ボリューム要求テンプレート）に対して、１つの [PersistentVolume（持続型ボリューム）](/jp/docs/concepts/storage/persistent-volumes/) を割り当てます。
先述の nginx の例では、各ポッドは１つの PersistentVolume に `my-storage-class` の StorageClass（ストレージ・クラス）を与えられ、1Gib の容量が供給されます。
もし StorageClass を指定しなければ、デフォルトの StorageClass が使われます。
ポッドがノード内に（再）スケジュールされた時、 `volumeMounts` は PersistentVolume Claim（持続ボリューム要求）に関連付けられた PersistentVolume（持続ボリューム）をマウントします。
注意するのは、PersistentVolume（持続ボリューム）が関連付けられているのはポットに対する PersistentVolume Claim（持続ボリューム要求）であり、これはポッドや StatefulSet が削除されても削除されません。
削除をするには手動で行う必要があります。

### ポッド名ラベル（Pod Name Label）{#pod-name-label}

StatefulSet コントローラがポッドを作成すると、 `statefulset.kubernetes.io/pod-name` ラベルを追加します。
これはポッドの名前を追加します。
このラベルによって、StatefulSet が特定のポッドに対してサービス割り当て（アタッチする）可能になります。

## デプロイメントとスケーリング保証 {#deployment-and-scaling-guarantees

* StatefulSet に N 複製（レプリカ）があり、ポッドがデプロイされると、 {0..N-1} まで順番に連続して作成される
* ポッドの削除が始まると、削除は逆順 {N-1..0} で削除される
* スケーリング作業がポッドに適用される前に、全ての先行作業（predecessor）が実行され、準備が整っている必要がある
* ポッドが削除される前に、その後継者（successor）が完全にシャットダウンしている必要がある

StetefulSet では `pod.Spec.TerminationGracePeriodSeconds` を 0 に指定すべきではありません。
この実行は安全ではなく、とても思い留めるべきです。
詳しい説明については  [StatefulSet ポッドの強制削除](/jp/docs/tasks/run-application/force-delete-stateful-set-pod/) を参照ください。

先ほど作成した nginx の例では、３つのポッドが web-0、web-1、web-2 の順番で展開（デプロイ）されます。
web-1 は web-0 が [実行中かつ待機](/jp/docs/user-guide/pod-states/) にならないと展開しませんし、web-2 は web1 が実行中かつ待機にならないと展開しません。
もし、web-0 の展開に失敗し、後の web-1 が実行中かつ待機になったとしても、web-2 が起動していなければ、web-0 の起動が完了しない限り web-2 は実行中かつ待機にはなりません。

もしもユーザが StatefulSet にパッチをあてて（追加で）デプロイするような場合、 `repicas=1` であれば、 web-2 がまずはじめに削除されます。
web-2 が完全にシャットダウンして削除されるまで、web-1 は停止（terminate）されません。
もしも web-2 が削除されて完全にシャットダウンされたあとに web-0 で障害が起こると、web-1 の削除よりも優先されます。web-1 は web-0 が実行・待機中となるまで削除されません。

### ポッド管理（Pod Managemement）方針 {#pod-management-policies}

Kubernetes 1.7 以降では、StatefulSet はユニーク性と同一性の保証が `.spec.podManagementPolicy` フィールドの指定によって、確実な順序づけ（ordering）を緩和できるようになりました。

#### OrderedReady ポッド管理 {#orderedready-pod-management}

`OrderedReady` ポッド管理とは StatefulSet のためのデフォルトです。
挙動の実装については [前述]](#deployment-and-scaling-guarantees) の通りに説明があります。

#### Parallel（並列）ポッド管理 {#parallel-pod-management}

`Parallel` （並列）ポッド管理は、StatefulSet コントローラに対して、ポッドの起動と停止を並列に行うように伝えます。
また、他のポッドの起動や停止に対する優先度に拘わらず、完全な完了を待たずに、ポッドの起動や削除を行います。

## 更新ストラテジ（Update Strategies） {#update-strategies}

Kubernetes 1.7 以降では、 StatefulSet の `.spec.updateStrategy`  フィールドによって、コントローラ、ラベル、リソース要求・制限、StatefulSet 内のポッドに対する自動化に関して、自動的なローリング・アップデート（逐次更新）の調整や無効化が行えます。

### On Delete（削除した状態で）{#on-delete}

`OnDelete` 更新ストラテジの実装は、過去（1.6 以前）の挙動です。
StatefulSet の `.spec.updateStrategy.type` を `OnDelete` に指定すると、StatefulSet コントローラは StatefulSet 内のポッドを自動的に更新しません。
コントローラが新しいポッドを削除できるようにするには、ユーザはポッドを手動で削除する必要があります。StatefulSet の `.spec.template` によって設定が反映されます。

### Rolling Updates（逐次更新） {#rolling-updates}

`RollingUpdate` 更新ストラテジの実装は、StatefulSet 内のポッドに対して自動的なローリング・アップデート（逐次更新）をもたらします。
これは `.spec.updateStrategy`  を未定義な場合のデフォルトのストラテジです。
StatefulSet の `.spec.updateStrategy.type` を `RollingUpdate` にすると、StatefulSet コントローラは StatefulSet 内のポッドを削除および再作成します。
処理の進行はポッドを削除する順番と同じであり、各ポッドを同時に削除する場合と同じです（最も大きなオリジナルから小さなものへ）。
ポッドの更新に先立って、ポッドが実行中かつ待機になるまで待機します。

#### パーティション {#partitions}

`RollingUpdate` 更新ストラテジでは `.spec.updateStrategy.rollingUpdate.partition` の指定によって分割して行えます。
パーティションが指定されると、全てのポッドが順序づけされ、 StatefulSet の `.spec.template` が更新されると、パーティションと一致するか大きいものが更新されます。
順序づけられたポッドが、パーティションよりも小さければ更新されませんし、削除されたとしても、以前のバージョンのものを再作成します。
ほとんどの場合、パーティションを使う必要はありません。
しかし、ステージごとの更新や、カナリア方式のロールアウトや、段階的なロールアウトを処理するには役立つでしょう。


{{% /capture %}}
{{% capture whatsnext %}}

* [ステートフルなアプリケーション](/jp/docs/tutorials/stateful-application/basic-stateful-set/) 例をフォロー。
* [Cassandra を StatefulSet でデプロイする](/jp/docs/tutorials/stateful-application/cassandra/) 例をフォロー。

{{% /capture %}}

