---
title: コンセプト
main_menu: true
weight: 40
---

コンセプトの章では、Kubernetesの一部や、Kubernetesがクラスタを表現するために使用する抽象概念について学び、Kubernetesがどのように動作するのかを深く理解することを助けます。

## 全体

Kubernetesで作業をするためには、 *Kubernetes APIオブジェクト* を使用してクラスタの *状態* を記述します。どのようなアプリケーションやワークロードを動作させたいのか、どのようなコンテナイメージを使うのか、レプリカの数、どのようなネットワークやディスクリソースがあるのか、などです。通常は`kubectl`というコマンドラインインターフェースを使用し、Kubernetes APIでオブジェクトを作成することによって目的の状態を設定します。また、Kubernetes APIを直接し要すして目的の状態を設定、変更することも出来ます。

一度目的の状態を設定したら、*Kubernetes Control Plane*はクラスタの現在の状態を目的の状態に一致させるように動作します。そのようにすることで、Kubernetesはコンテナの開始や再起動、レプリカ数のスケールなどといった操作を自動で実行します。Kubernetes Control Planeは、クラスタ上で実行されている一連のプロセスで構成されています:

* **Kubernetes Master** は、マスターノードとして指定された、クラスタ内の単一ノード上で実行されている3つのプロセスの集合です。 3つのプロセスとは、[kube-apiserver](/docs/admin/kube-apiserver/)、[kube-controller-manager](/docs/admin/kube-controller-manager/)、[kube-scheduler](/docs/admin/kube-scheduler/)です。
* マスターノードではない、クラスタ上の個別のノードは2つのプロセスを実行します:
  * **[kubelet](/docs/admin/kubelet/)**は、マスターノードと通信をします。
  * **[kube-proxy](/docs/admin/kube-proxy/)**は,各ノード上のKubernetesネットワーキングサービス実現するネットワークプロキシです。

## Kubernetesオブジェクト

Kubernetesはシステムの状態を表現するため、デプロイされたコンテナアプリケーションやワークロード、それらに紐付けられたネットワークやディスクリソース、その他クラスタが何をしているかについての情報など、多くの抽象化を含んでいます。これらの抽象概念はKubernetes APIのオブジェクトで表現されます。詳細は[Kubernetes Objects overview](/docs/concepts/abstractions/overview/)を見てください。

基本的なKubernetesオブジェクトは以下のものを含みます:

* [Pod](/docs/concepts/workloads/pods/pod-overview/)
* [Service](/docs/concepts/services-networking/service/)
* [Volume](/docs/concepts/storage/volumes/)
* [Namespace](/docs/concepts/overview/working-with-objects/namespaces/)

加えて、Kubernetesはコントローラと呼ばれる、高度な抽象概念を持っています。コントーらは基本オブジェクトの上に構築され、追加の機能性や利便性を提供します。これらは、次のものを含みます:

* [ReplicaSet](/docs/concepts/workloads/controllers/replicaset/)
* [Deployment](/docs/concepts/workloads/controllers/deployment/)
* [StatefulSet](/docs/concepts/workloads/controllers/statefulset/)
* [DaemonSet](/docs/concepts/workloads/controllers/daemonset/)
* [Job](/docs/concepts/workloads/controllers/jobs-run-to-completion/)

## Kubernetes Control Plane

Kubernetes Masterやkubeletプロセスなどのような、Kubernetes Control Planeは、Kubernetesがどのようにクラスタと通信を行うかを制御します。Control Planeはシステム中のすべてのKubernetesオブジェクトに関するレコードを保守し、オブジェクトの状態を管理するために継続的な管理ループを実行します。任意の時点で、Control Planeの制御ループはクラスタ内の変更に応答し、システム内のすべてのオブジェクトの状態を、目的の状態と一致させるように動作します。

例えば、Deploymentオブジェクトを作成するためには、Kubernetes APIを使用してシステムの新しい目的状態を提供します。Kubernetes Control Planeはオブジェクトの作成を記録し、必要なアプリケーションを起動してクラスタノードにスケジューリングすることでクラスタの状態を目的の状態に一致させます。

### Kubernetes Master

The Kubernetes master is responsible for maintaining the desired state for your cluster. When you interact with Kubernetes, such as by using the `kubectl` command-line interface, you're communicating with your cluster's Kubernetes master.

> The "master" refers to a collection of processes managing the cluster state.  Typically these processes are all run on a single node in the cluster, and this node is also referred to as the master. The master can also be replicated for availability and redundancy.

### Kubernetes Nodes

The nodes in a cluster are the machines (VMs, physical servers, etc) that run your applications and cloud workflows. The Kubernetes master controls each node; you'll rarely interact with nodes directly.

#### Object Metadata


* [Annotations](/docs/concepts/overview/working-with-objects/annotations/)


### What's next

コンセプトページを書きたい場合、コンセプトページとコンセプトテンプレートの情報を得るため、
[Using Page Templates](/docs/home/contribute/page-templates/)を見てください。
