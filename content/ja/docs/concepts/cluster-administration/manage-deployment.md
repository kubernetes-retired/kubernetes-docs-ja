---
title: リソース管理
content_template: templates/concept
weight: 40
---

{{% capture overview %}}

アプリケーションを展開（デプロイ）したら、サービスを経由して外に対する公開ができます。
そのためには、何をどうしたらよいのでしょうか？
Kubernetes が提供するのは、アプリケーションの展開に役立つツールであり、スケーリングと更新も含みませす。
共通している機能に関しては、[設定情報ファイル](/jp/docs/concepts/configuration/overview/) と [ラベル](/jp/docs/concepts/overview/working-with-objects/labels/) で更に深く扱っています。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## リソース設定情報の組織化 {#organizing-resource-configurations}

多くのアプリケーションは、デプロイメントやサービスといった、複数のリソース作成を必要とします。
複数のリソース管理は、同じファイルで（ YAML の `---` で分離 ）一緒にグループ化することで、単純化できます。
こちらは例です:

{{< code file="nginx-app.yaml" >}}

複数のリソースを作成するには、１つのリソース作成と同様に作成できます。

```shell
$ kubectl create -f https://k8s.io/docs/concepts/cluster-administration/nginx-app.yaml
service "my-nginx-svc" created
deployment "my-nginx" created
```

リソースはファイルに記述された順番で作成されます。
そのため、最初にサービスを指定するのがベストです。
そうしておけば、スケジューラはポッドに関連付けられたサービスを展開できるようになります。このサービスとはデプロイメントのようなコントローラによって作成されるものも含みます。

また、 `kubectl create` は複数の `-f` 引数を受け付けられます。

```shell
$ kubectl create -f https://k8s.io/docs/concepts/cluster-administration/nginx/nginx-svc.yaml -f https://k8s.io/docs/concepts/cluster-administration/nginx/nginx-deployment.yaml
```

そして、ここのファイルを追加する代わりに、ディレクトリも指定できます。

```shell
$ kubectl create -f https://k8s.io/docs/concepts/cluster-administration/nginx/
```

`kubectl` は末尾が `.yaml` や `.yml` や `.json` のあらゆるファイルを読み込めます。

実践において推奨するのは、同じマイクロサービスやアプリケーションに関連するリソースを同じファイルにおき、アプリケーションに関連するファイルのグループを同じディレクトリ内に置くことです。
アプリケーション層がお互いに DNS を使って結びついている場合は、全てのコンポーネントのスタックを一緒にして、シンプルに展開（デプロイ）できます。

また、URL で設定情報のソース（元）として指定できます。URL は GitHub でチェックした設定情報ファイルを、直接デプロイするのに手軽です。

```shell
$ kubectl create -f https://raw.githubusercontent.com/kubernetes/website/master/docs/concepts/cluster-administration/nginx-deployment.yaml
deployment "nginx-deployment" created
```

## kubectl で大量の操作 {#bulk-operations-in-kubectl}

```shell
$ kubectl delete -f https://k8s.io/docs/concepts/cluster-administration/nginx-app.yaml
deployment "my-nginx" deleted
service "my-nginx-svc" deleted
```

(TODO)