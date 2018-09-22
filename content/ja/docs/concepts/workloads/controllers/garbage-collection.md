---
title: ガベージ・コレクション
content_template: templates/concept
weight: 60
---

{{% capture overview %}}

Kubernetes ガベージ・コレクタ（garbage collector）の役割は、かつては所有者がいたものの、現在は所有者がいないオブジェクトの削除です。


**メモ：** ガベージ・コレクションはベータ機能であり、デフォルトで有効なのは Kubernetes 1.4 以降です。

{{% /capture %}}


{{% capture body %}}

## 所有者と従属 {#owners-and-dependents}

いくつかの Kubernetes オブジェクトは他のオブジェクトの所有者です。
たとえば、ReplicaSet はポッドの集まりの所有者です。
所有しているオブジェクトは、所有者オブジェクトの *dependents（従属）* と呼びます。
それぞれの従属オブジェクト（dependent object）は `metadata.ownerReferences` （メタデータ.所有者参照）フィールドを持ちます。これは所有しているオブジェクトを示します。

時々、Kubernetes は `ownerReference` （所有者参照）の値を自動的に設定します。
たとえば、ReplicaSet の作成時、Kubernetes は自動的に ReplicaSet 内の各ポッドの `ownerReference` フィールドを更新します。
Kubernetes 1.8 では、オブジェクトの作成時または採用時（adopted）に  `ownerReference` の値が ReplicationController、ReplicaSet、StatefulSet、Deployment、Job、CronJob によって自動的に設定されます。

また、自分で所有者と従属に関する指定を、手動で `ownerReference` フィールドの更新もできます。

こちらの設定ファイルは３つのポッドを持つ ReplicaSet です。

{{< codenew file="controllers/replicaset.yaml" >}}

もし ReplicaSet を作成し、ポッドのメタデータを参照すると、OwnerReferences フィールドが見えるでしょう。

```shell
kubectl create -f https://k8s.io/examples/controllers/replicaset.yaml
kubectl get pods --output=yaml
```

出力結果から、ポッドの所有者が `myrepset` という名称の ReplicaSet だと分かります。

```shell
apiVersion: v1
kind: Pod
metadata:
  ...
  ownerReferences:
  - apiVersion: apps/v1
    controller: true
    blockOwnerDeletion: true
    kind: ReplicaSet
    name: my-repset
    uid: d9607e19-f88f-11e6-a518-42010a800195
  ...
```

## ガベージ・コレクタが削除する従属を制御するには {#controlling-how-the-garbage-collector-deletes-dependents}

オブジェクトの削除時、オブジェクトの従属を限定して、自動的な削除を行えます。
従属の自動的な削除を *cascading deletion（連鎖削除／カスケーディング削除）* と呼びます。
*cascading deletion（連鎖削除）*には *background（バックグラウンド）* と *foreground （フォアグラウンド）* があります。

オブジェクトの削除時に従属を自動自動的に削除しなければ、従属は *orphaned（孤立化）*  すると呼ばれます。

### フォアグラウンド連鎖削除（Foreground cascading deletion） {#forground-cascading-deletion}

*フォアグラウンド連鎖削除* において、元になるオブジェクトがまず先に "deletion in progress"（削除進行中） 状態となります。
"deletion progress"  状態においては、以下の状態が true（正）です：

 * オブジェクトは REST API を経由して見えるまま
 * オブジェクトの `deletionTimestamp` がセットされる
 * オブジェクトの `metadata.finalizers` には "foregroundDeletion" の値が含まれる

一度 "deletion in progress" 状態がセットされると、ガベージ・コレクションはオブジェクトの従属を削除します。
ガベージ・コレクタがすべての "blocking" （ブロックされた）従属（ `ownerReference.blockOwnerDeletion=true` のオブジェクト）を削除すると、所有者オブジェクトを削除します。

"foregroundDeletion" は `ownerReference.blockOwnerDeletion` ブロックを持つ従属のみが、所有者オブジェクトの削除対象となりますのでご注意ください。
Kubernetes バージョン 1.7 は [admission controller（承認コントローラ）](/jp/docs/admin/admission-controllers/#ownerreferencespermissionenforcement) を追加しました。
これは所有者オブジェクト上の削除権限を、`blockOwnerDeletion`  が true をベースとしたユーザアクセスを制御します。
そのため、所有者オブジェクトの削除をしても、その後の従属は削除が許可されません。

オブジェクトの `ownerReferences` フィールドはコントローラ（Deployment や ReplicaSet）によって設定されますが、
blockOwnerDeletion は自動的に設定されるもので、ユーザが手動でこのフィールドを変更する必要はありません。

### バックグラウンド連鎖削除（Background cascading deletion） {#background-cascading-deletion}

*background cascading deletion（バックグラウンド連鎖削除）* では、Kubernetes は所有者オブジェクトを直ちに削除し、ガベージ・コレクタがバックグラウンドでその従属を削除します。

### 連鎖削除ポリシーの設定 {#setting-the-cascading-deletion-policy}

連鎖削除ポリシーを設定するには、 `propagationPolicy` （伝搬方針）フィールドにある `deleteOptions`  （削除オプション）引数を、オブジェクトの削除時に設定します。
ここで設定できる値は "Orphan" （孤立）、 "" （フォアグラウンド）、 "" （バックグラウンド）です。

Kubernetes 1.9 よりも前のバージョンでは、多くのコントローラ・リソースにおけるデフォルトのガベージ・コレクション方針は `orphan` （孤立）でした。
これには ReplicationController、ReplicaSet、StatefulSet、DaemonSet、Deployment を含みます。
`extensions/v1beta1`、 `apps/v1beta1`、 `apps/v1beta2` グループ・バージョンでは、項目の指定がなければ、従属オブジェクトはデフォルトで孤立します。
Kubernetes 1.9 では、 `apps/v1` のグループ・バージョンすべてが、デフォルトで従属オブジェクトが削除されます。

こちらはバックグラウンドで従属を削除する例です:

```shell
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
-H "Content-Type: application/json"
```

こちらはフォアグラウンドで従属を削除する例です:

```shell
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
-H "Content-Type: application/json"
```
こちらは孤立した従属の例です:

```shell
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
-H "Content-Type: application/json"
```

また、kubectl も連鎖削除をサポートします。
kubectl を使って従属を自動的に削除するには、 `--cascade` を true に設定します。
従属を孤立させるには、 `--cascade` を false にします。
`--cascade` のデフォルト値は true です。

こちらは ReplicaSet の従属を孤立化する例です:

```shell
kubectl delete replicaset my-repset --cascade=false
```

### デプロイメント上における追加注記 {#additional-note-on-deployments}

デプロイメントを連鎖削除するときは、 `propagationPolicy: Foreground` にするのが *必須* です。
作成した Replicaset の削除だけでなく、ポッドも削除する必要があります。
_propagationPolicy_ （伝搬方針）を使わなければ、ReplicaSet のみが削除され、ポッドは孤立化します。
詳しい情報は [kubeadm/#149](https://github.com/kubernetes/kubeadm/issues/149#issuecomment-284766613) をご覧ください。

## 衆知の問題 {#known-issue}

[#26120](https://github.com/kubernetes/kubernetes/issues/26120) を追跡ください。

{{% /capture %}}


{{% capture whatsnext %}}

[設計文書1](https://git.k8s.io/community/contributors/design-proposals/api-machinery/garbage-collection.md)

[設計文章2](https://git.k8s.io/community/contributors/design-proposals/api-machinery/synchronous-garbage-collection.md)

{{% /capture %}}



