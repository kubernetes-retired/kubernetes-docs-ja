---
title: Kubernetesコンポーネント
content_template: templates/concept
weight: 20
---

{{% capture overview %}}
このドキュメントでは、動作するKubernetesクラスタに必要なバイナリコンポーネントについて説明します。
{{% /capture %}}

{{% capture body %}}
## Masterコンポーネント

MasterコンポーネントはクラスタのControl Planeを提供します。Masterコンポーネントはクラスタについての全体的な決定(スケジューリング)を行ったり、クラスタのイベント(replication controllerの'replicas'フィールドが充足されていないときに新しいポッドを作成する等)を検知し、対応したりします。

Masterコンポーネントはクラスタ内のいずれのマシンでも実行可能です。
しかし、単純化のために、セットアップスクリプトは通常、すべてのMasterコンポーネントを同じマシン上で実行し、そのマシン上ではユーザのコンテナを実行しません。
マルチマスターVMの例を構成するために、[Building High-Availability Clusters](/docs/admin/high-availability/)を見てください。

### kube-apiserver

{{< glossary_definition term_id="kube-apiserver" length="all" >}}

### etcd

{{< glossary_definition term_id="etcd" length="all" >}}

### kube-scheduler

{{< glossary_definition term_id="kube-scheduler" length="all" >}}

### kube-controller-manager

{{< glossary_definition term_id="kube-controller-manager" length="all" >}}

以下のコンポーネントを含みます:

  * Node Controller: ノードがダウンしたときに検知し、対応する責務があります。
  * Replication Controller: システム上のすべてのreplication controllerオブジェクトで正しい数のpodを維持する責務があります。
  * Endpoints Controller:　Endpointsオブジェクトを生成します(つまり、ServiceとPodを結合します)。
  * Service Account & Token Controllers: 新しいnamespaceのための既定のアカウントとAPIアクセストークンを作成します。

### cloud-controller-manager

[cloud-controller-manager](/docs/tasks/administer-cluster/running-cloud-controller/)は、基盤となるクラウドプロバイダーと対話するコントローラを実行します。cloud-controller-managerはKubernetes 1.6ではアルファ機能です。

cloud-controller-managerはcloud-provider特有の制御ループのみ実行します。kube-controller-managerにおけるこれらの制御ループは無効化する必要があります。無効化するには、kube-controller-managerを起動する際に、`--cloud-provider`フラグを`external`に設定します。

cloud-controller-managerはクラウドベンダーのコードとKubernetesコアを互いに独立して進化させることを可能にします。以前のリリースでは、コアKubernetesコードはクラウドプロバイダ固有の機能コードに依存していました。将来のリリースでは、クラウドベンダー特有のコードはクラウドベンダー自身によって管理され、cloud-controller-managerにリンクして使用します。

次のコントローラはクラウドプロバイダに依存しています:

  * Node Controller: ノードが停止した後、クラウド上でノードが削除されたかどうかを調べるため
  * Route Controller: 基盤クラウドインフラ上でのルーティングを設定するため
  * Service Controller: クラウドプロバイダのロードバランサを作成、更新、削除するため
  * Volume Controller: ボリュームを作成、接続、マウントしたり、クラウドプロバイダと対話してボリュームを構成するため

## Nodeコンポーネント

Nodeコンポーネントはすべてのノードで動作し、podの実行とKubernetes実行環境の提供を維持します。

### kubelet

{{< glossary_definition term_id="kubelet" length="all" >}}

### kube-proxy

[kube-proxy](/docs/admin/kube-proxy/)はホストのネットワークルールを管理したり、接続転送を実行することでKubernetesサービスの抽象化を可能にします。

### コンテナランタイム

コンテナランタイムはコンテナを実行する責務を負うソフトウェアです。Kubernetesは複数のランタイムをサポートしています: [Docker](http://www.docker.com)、[rkt](https://coreos.com/rkt/)、[runc](https://github.com/opencontainers/runc)及びOCI([runtime-spec](https://github.com/opencontainers/runtime-spec))を実装した任意のランタイム

## アドオン

アドオンはクラスタの機能を実装するpodやserviceです。そのpodはdeploymentやreplicationController等によって管理されます。名前空間を区切られたアドオンは`kube-system`という名前空間に作成されます。

いくつかのアドオンは以下に説明しています。有効なアドオンの一覧は[Addons](/docs/concepts/cluster-administration/addons/)ページを見てください。

### DNS

その他のアドオンは厳密に必須とはされていませんが、すべてのKubernetesクラスタは[cluster DNS](/docs/concepts/services-networking/dns-pod-service/)を持っているべきです。

Cluster DNSはDNSサーバで、他のDNSサーバに加え、KubernetesのserviceのDNSレコードを返します。

Kubernetesで実行されたコンテナは自動的にこのDNSサーバに含まれます。

### Web UI (Dashboard)

[Dashboard](/docs/tasks/access-application-cluster/web-ui-dashboard/)は汎用のKubernetesクラスタのWebUIです。クラスタ自身を含む、クラスタ上で動作しているアプリケーションを管理したりトラブルシューティングを行ったりするのに役立ちます。

### Container Resource Monitoring

[Container Resource Monitoring](/docs/tasks/debug-application-cluster/resource-usage-monitoring/)はコンテナについての時系列メトリクスを中央DBに格納し、それらを閲覧するためのUIを提供します。

### Cluster-level Logging

[Cluster-level logging](/docs/concepts/cluster-administration/logging/)はコンテナのログを中央ログストアに保存し、検索・ブラウジングインターフェースを提供する機構です。

{{% /capture %}}
