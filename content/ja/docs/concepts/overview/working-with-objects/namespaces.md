---
title: 名前空間
content_template: templates/concept
weight: 30
---

{{% capture overview %}}
Kubernetes は同じ物理クラスタ上で複数の仮想クラスタをサポートします。
これらの仮想クラスタを名前空間（namespace）と呼びます。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}
## 複数の名前空間を使う場合 {#when-to-user-multiple-namespaces}

名前空間（namespaces）の利用を想定しているのは、多くの利用者が複数のチームやプロジェクトを広く横断している環境です。
利用者が数人から10人のクラスタでは、名前空間の作成や検討する必要はほとんどないでしょう。
名前空間が必要な機能が必要になった場合に、名前空間の利用を始めます。

名前空間が提供するのは名前の範囲（scope for namess）です。
名前空間内ではリソース名がユニークである必要がありますが、名前空間を越えません。

名前空間とは複数のユーザ間でクラスタのリソースを（[リソース・クォータ](/jp/docs/concepts/policy/resource-quotas/)を経由して）分割するための手段です。

Kubernetes の今後のバージョンでは、同じ名前空間にあるオブジェクトが、同じアクセス制御ポリシーを持つのがデフォルトになります。

同じソフトウェアの異なるバージョンを使うなど、わずかに異なるリソースのために名前空間を使う必要はありません。
そのような場合は同じ名前空間内でも [ラベル](/docs/concepts/overview/working-with-objects/labels/) を使ってリソースを区別します。

## 名前空間の動作 {#working-with-namespaces}

名前空間の作成と定義については、[名前空間の管理者ガイド・ドキュメント](/jp/docs/tasks/administer-cluster/namespaces/) に説明があります。

### 名前空間の表示 {#viewing-namespaces}

クラスタで使っている名前空間の一覧を表示できます:

```shell
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d
kube-public   Active    1d
```

Kubernetes は初期に３つの名前空間を開始します。

   * `default` オブジェクトが一切ないデフォルトの名前空間。
   * `kube-system` Kubernetes システムによって作成されるオブジェクト用の名前空間。
   * `kube-public` 自動的に作成され、すべてのユーザ（認証されていないユーザも含む）が読み込み可能な名前空間。この名前空間はクラスタが使うために予約されています。用途はクラスタ全体を通してパブリックに表示かつ読み込み可能な同一リソースを使う場合です。この名前空間がパブリックに見えるのは利便性のためであり、（利用は）必須ではありません。

### リクエストのために名前空間を設定 {#setting-the-namespace-for-a-request}

リクエストのために一時的な名前空間を設定するには、 `--namespace` フラグを使います。

例:

```shell
$ kubectl --namespace=<insert-namespace-name-here> run nginx --image=nginx
$ kubectl --namespace=<insert-namespace-name-here> get pods
```

### 優先する名前空間の設定 {#setting-the-namespace-preference}

kubectl コマンドに続くすべての名前空間が変わらないように保存するには、次のように実行します。

```shell
$ kubectl config set-context $(kubectl config current-context) --namespace=<insert-namespace-name-here>
# 確認
$ kubectl config view | grep namespace:
```

## 名前空間と DNS {#namespaces-and-dns}

[サービス](/jp/docs/concepts/services-networking/service/) を作成すると、適切な [DNS エントリ](/jp/docs/concepts/services-networking/dns-pod-service/) が作成されます。
このエントリは `<サービス名>.<名前空間名>.svc.cluster.local` の形式です。
これが意味するのは、 `<サービス名>` にあたるのがコンテナで使われる箇所で、ローカルな名前空間におけるサービスの名前解決に使います。
これは展開（デプロイメント）、ステージング、本番といった、複数の名前空間を通して同じ設定を使う場合に便利です。
もしも名前空間を越えて到達したい場合は、完全修飾ドメイン名（FQDN）を使う必要があります。

## オブジェクトが名前空間にあるとは限らない {#not-all-obujects-are-in-a-namespace}

多くの Kubernetes リソース（例：ポッド、サービス、レプリケーション・コントローラ、等）は同じ名前空間内にあります。しかしながら、名前空間リソースそのものが名前空間にはありません。
また、 [ノード](/docs/concepts/architecture/nodes/) や一貫性ボリュームといった低レベル・リソースは名前空間内にはありません。

{{% /capture %}}