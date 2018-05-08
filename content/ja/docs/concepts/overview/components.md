---
reviewers:
- lavalamp
title: Kubernetesコンポーネント
content_template: templates/concept
---

{{% capture overview %}}
このドキュメントでは、動作するKubernetes clusterに必要なバイナリコンポーネントについて説明します。
{{% /capture %}}

{{% capture body %}}
## Masterコンポーネント

MasterコンポーネントはクラスタのControl Planeを提供します。Masterコンポーネントはクラスタについての全体的な決定(スケジューリング)を行ったり、クラスタのイベント(replication controllerの'replicas'フィールドが充足されていないときに新しいポッドを作成する等)を検知し、対応したりします。

Masterコンポーネントはクラスタ内のいずれのマシンでも実行可能です。
しかし、単純化のために、セットアップスクリプトは通常、すべてのマスターコンポーネントを同じマシン上で実行し、そのマシン上ではユーザのコンテナを実行しません。
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

cloud-controller-manager allows cloud vendors code and the Kubernetes core to evolve independent of each other. In prior releases, the core Kubernetes code was dependent upon cloud-provider-specific code for functionality. In future releases, code specific to cloud vendors should be maintained by the cloud vendor themselves, and linked to cloud-controller-manager while running Kubernetes.

The following controllers have cloud provider dependencies:

  * Node Controller: For checking the cloud provider to determine if a node has been deleted in the cloud after it stops responding
  * Route Controller: For setting up routes in the underlying cloud infrastructure
  * Service Controller: For creating, updating and deleting cloud provider load balancers
  * Volume Controller: For creating, attaching, and mounting volumes, and interacting with the cloud provider to orchestrate volumes

## Nodeコンポーネント

Nodeコンポーネントはすべてのノードで動作し、podの実行とKubernetes実行環境の提供を維持します。

### kubelet

{{< glossary_definition term_id="kubelet" length="all" >}}

### kube-proxy

[kube-proxy](/docs/admin/kube-proxy/) enables the Kubernetes service abstraction by maintaining
network rules on the host and performing connection forwarding.

### Container Runtime

The container runtime is the software that is responsible for running containers. Kubernetes supports several runtimes: [Docker](http://www.docker.com), [rkt](https://coreos.com/rkt/), [runc](https://github.com/opencontainers/runc) and any OCI [runtime-spec](https://github.com/opencontainers/runtime-spec) implementation.

## Addons

Addons are pods and services that implement cluster features. The pods may be managed
by Deployments, ReplicationControllers, and so on. Namespaced addon objects are created in
the `kube-system` namespace.

Selected addons are described below, for an extended list of available addons please see [Addons](/docs/concepts/cluster-administration/addons/).

### DNS

While the other addons are not strictly required, all Kubernetes clusters should have [cluster DNS](/docs/concepts/services-networking/dns-pod-service/), as many examples rely on it.

Cluster DNS is a DNS server, in addition to the other DNS server(s) in your environment, which serves DNS records for Kubernetes services.

Containers started by Kubernetes automatically include this DNS server in their DNS searches.

### Web UI (Dashboard)

[Dashboard](/docs/tasks/access-application-cluster/web-ui-dashboard/) is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage and troubleshoot applications running in the cluster, as well as the cluster itself.

### Container Resource Monitoring

[Container Resource Monitoring](/docs/tasks/debug-application-cluster/resource-usage-monitoring/) records generic time-series metrics
about containers in a central database, and provides a UI for browsing that data.

### Cluster-level Logging

A [Cluster-level logging](/docs/concepts/cluster-administration/logging/) mechanism is responsible for
saving container logs to a central log store with search/browsing interface.

{{% /capture %}}
