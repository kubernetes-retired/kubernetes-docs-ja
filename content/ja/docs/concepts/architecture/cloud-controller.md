---
title: クラウド・コントローラ・マネージャの基礎概念
content_template: templates/concept
weight: 30
---

{{% capture overview %}}
クラウド・コントローラ・マネージャ（CCM）の概念とは（プログラムとしてのバイナリと混同しないでください）、本来はクラウド固有のベンダー・コードと Kubernetes コアがお互い独立して開発を進められるようにするためでした。
クラウド・コントローラ・マネージャは Kubernetes コントローラ・マネージャ、API サーバ、スケジューラのような、他のマスタ構成要素と一緒に実行します。
また、Kubernetes アドオンとしても起動できるため、Kubernetes 上で実行する場合もあります。

クラウド・コントローラ・マネージャの設計はプラグイン機構に基づいた設計です。
そのため、クラウド事業者が Kubernetes と統合するにはプラグインを使うのが簡単です。

このドキュメントが扱うのはクラウド・コントローラ・マネージャの背後にある概念と、関連する機能に関する詳細を提供します。

ここは Kubernetes クラスタの構造（アーキテクチャ）を使いますが、クラウド・コントローラ・マネージャについては扱いません。

![Pre CCM Kube Arch](/images/docs/pre-ccm-arch.png)

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## 設計 {#design}

先ほどの図にあるように、Kubernetes とクラウド事業者は複数の異なる構成要素（コンポーネント）を通して統合されています。

* Kubelet
* Kubernetes コントローラ・マネージャ
* Kubernetes API サーバ

先ほどの３つの構成要素から、クラウドに１つの統合地点を作成するために、クラウドに依存する仕組み（ロジック）全てを CCM が集約します。

![CCM Kube Arch](/images/docs/post-ccm-arch.png)

## CCM の構成要素 {#components-of-the-ccm}

CCM は Kubernetes コントローラ・マネージャ（KCM）の複数ある機能から切り離されており、独立した機能として動作します。
特に、クラウドに依存する KCM 内のコントローラからは切り離されています。
KCM は以下のクラウドに依存する制御ループがあります：

 * ノード・コントローラ（Node controller）
 * ボリューム・コントローラ（Volume controller）
 * 径路・コントローラ（Route controller）
 * サービス・コントローラ（Service Controller）

バージョン 1.9 では、CCM は先の一覧から、以下のコントローラを実行します。

 * ノード・コントローラ（Node controller）
 * 径路・コントローラ（Route controller）
 * サービス・コントローラ（Service Controller）

さらに、PersistentVolumeLabels と呼ぶ別のコントローラも動かします。
このコントローラは GCP と AWS クラウド上で、 PersistentVolumes（持続ボリューム） 上にゾーンとリージョンのラベルを設定する役割があります。

{{< note >}}
**メモ:** ボリューム・コントローラは CCM の一部としてではなく、意図的に選択する必要があります。
複雑さに伴う原因と、ベンダ固有のボリューム・ロジックを抽象化して切り離す既存の努力により、ボリューム・コントローラは CCM によって移動しないのが決定されました。

{{< /note >}}
本来はボリュームをサポートするために CCM を使う計画でした。
これはプラガブル（取り付け・取り外し可能）なボリュームをサポートするために、柔軟なボリュームを使うためでした。
ですが、CSI として知られている競合する取り組みは、柔軟なものへと置き換える計画がされました。

これらの動きを考慮し、CSI が利用可能になるよりも前に、私たちはただちに活動を停止しました。

## CCM の機能 {#functions-of-the-ccm}

CCM は Kubernetes の構成要素から、クラウド事業者に依存する機能を継承します。
このセクションは構造化された各構成要素に基づいた構成です。

### 1. Kubernetes コントローラ・マネージャ

CCM 機能の大部分は、KCM によって提供されます。
以前のセクションで言及したように、CCM は以下の制御ループを扱います:

* ノード・コントローラ（Node controller）
* 径路（Route）・コントローラ（controller ）
* サービス・コントローラ（Service controller）
* 持続ボリューム・ラベル（PersistentVolumeLabels controller）

#### ノード・コントローラ（Node controller） {#node-controller}

クラウド事業者からクラスタ内で実行しているノードに関する情報を取得するために、ノード・コントローラはノードの初期化に責任を持ちます。
ノード・コントローラは以下の機能を処理します：

1. クラウド固有のゾーンやリージョンのラベルと共に、ノードを初期化します。
2. クラウド固有のインスタンスの詳細と共に、ノードを初期化します。たとえば、種類（タイプ）や容量（サイズ）です。
3. ノードのネットワーク・アドレスとホスト名を取得します。
4. ノードの反応が無くなれば、クラウドでノードがクラウドから削除されていないかどうかを調べます。もしもクラウドからノードが削除されていれば、Kubernetes ノード・オブジェクトを削除します。

#### 径路コントローラ（Route controller）{#route-controller}

径路コントローラはクラウド内で適切な径路設定をする責任があります。
これにより、Kubernetes クラスタ内で異なったノード上にあるコンテナが、お互いに通信可能となります。
径路コントローラが適用できるのは Google Compute Engine クラスタのみです。

#### サービス・コントローラ（Service Controller） {#service-controller}

サービス・コントローラは、サービスの作成、更新、削除のイベントを受け付ける（listening）責任を持ちます。
Kubernetes 内にあるサービスの現在の状態（current state）に基づき、クラウド負荷分散装置（ロードバランサ）（ELB や Google LB）に Kubernetes 内のサービス状況を反映するための設定をします。
さらに、クラウド負荷分散装置（ロードバランサ）が更新されても、サービスのバックエンドを安全にします（確保します）。

#### 持続ボリューム・ラベル・コントローラ（PersistentVolumeLabels controller）{#persistentvolumelabels-controller}

持続ボリューム・ラベル・コントローラは、ボリュームの作成時に AWS EBS/GCE PD ボリューム上にラベルを適用します。
ボリュームに対して設定したラベルは、ユーザの必要があれば手動で削除する必要があります。

これらのラベルは、本質的にポッドのスケジューリングのためです。
各ボリュームは、リージョンまたはゾーン内でのみボリュームが動作するように制限されています。

持続ボリューム・ラベル・コントローラはCCM によって特別に作成されます。
つまり、CCM によって作成されるまで、これは存在しません。
この処理を行う PV ラベリング論理（ロジック）は、Kubernetes PI サーバ（以前はアドミッション・コントローラでした）から CCM に移動しました。
これは KCM 上では実行されません。

### 2. Kubelet

ノード・コントローラには kubelet のクラウドに依存する機能を含んでいます。
CCM が導入される以前は、kubelet はノードの初期化に責任があり、ここにクラウド固有の詳細も含んでいました。たとえば IP アドレス、リージョンやゾーン、ラベル、インフタンス型などの情報です。
CCM の導入によって、初期化作業は kubelet から CCM へと移行しました。

この新しいモデルでは、kubelet がノードを初期化するにあたり、クラウド固有の情報を含みません。
しかし、新しく作成されたノードに対してはテイントを追加します。
これは CCM がクラウド固有の情報と共にノードを初期化するまでは、ノードがスケジュール不可能とするものです。
その後、テイントを削除します。

### 3. Kubernetes API サーバ

持続ボリューム・ラベル・コントローラの、 Kubernetes API サーバのクラウドに依存する機能を、前述の通り CCM に移行しました。

## プラグイン構造 {#plugin-mechanism}

クラウド・コントローラ・マネージャは Go インターフェースを使い、あらゆるクラウドを差し込んで（plugged in）実装します。
特に、CloudProvider インターフェースを使うものは[こちら](https://github.com/kubernetes/kubernetes/blob/master/pkg/cloudprovider/cloud.go) に定義されています。

４つの共有コントローラの実装に関するハイライトは、前述の通りです。
そして、いくつかの足場（scaffolding）は共有クラウドプロバイダ・インターフェースと一緒に Kubernetes コアの中にあります。
クラウド事業者に対する特別な実装は、コアの外で構築されており、実装のためのインターフェースはコアの中で定義されています。

プラグイン開発に関する詳細は、[クラウド・コントローラ・マネージャの開発](/jp/docs/tasks/administer-cluster/developing-cloud-controller-manager/) をご覧ください。

## 承認（Authorization）{#authorization}

このセクションは CCM が処理する動作によって、様々な API オブジェクト上でのアクセスに必要なものを掘り下げます（ブレイクダウンします）。

### ノード・コントローラ {#node-controller}

ノード・コントローラが操作するのはノード・オブジェクトのみです。
ノード・コントローラはノード・オブジェクトに対する get、list、create、update、patch、watch、delete に対するフル・アクセスが必要です。

v1/Node: 

- Get
- List
- Create
- Update
- Patch
- Watch
- Delete

### 径路コントローラ {#route-controller}

径路コントローラはノード・オブジェクトの作成と適切な径路設定を受け付けます。
これにはノード・オブジェクトに対する get アクセスが必要です。

v1/Node: 

- Get

### サービス・コントローラ {#service-controller}

サービス・コントローラはサービス・オブジェクトの create、update、delete イベントと、これらサービスのエンドポイントに対する適切な設定を受け付けます。

サービスにアクセスするためには、list と watch 権限が必要です。
サービスを更新するには、patch と update 権限が必要です。

サービスのエンドポイントをセットアップするには、create、list、get、watch、update 権限が必要です。

v1/Service:

- List
- Get
- Watch
- Patch
- Update

### 持続ボリューム・ラベル・コントローラ {#persistentvolumelabels-controller}

持続ボリューム・ラベル・コントローラは、持続ボリューム（PV）上での create イベントと update を受け付けます。
このコントローラは持続ボリュームに対する get と update 権限が必要です。

v1/PersistentVolume:

- Get
- List
- Watch
- Update

### その他 {#others}

CCM のコア実装に対する必要な権限は create イベントです。安全な作業のためには ServiceAccount（サービス・アカウント）を作成する権限が必要です。

v1/Event:

- Create
- Patch
- Update

v1/ServiceAccount:

- Create

CCM に対する RBAC ClusterRole は以下のようなものです:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloud-controller-manager
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - create
  - get
  - list
  - watch
  - update
```

## ベンダー実装 {#vender-implementations}

以下のクラウド事業者は CCM を実装しています：

* Digital Ocean
* [Oracle](https://github.com/oracle/oci-cloud-controller-manager)
* Azure
* GCE
* AWS

## クラスタ管理 {#cluster-administration}

CCM の設定と実行に関する全面的な手順は [こちら](/jp/docs/tasks/administer-cluster/running-cloud-controller/#cloud-controller-manager) にあります。


{{% /capture %}}
