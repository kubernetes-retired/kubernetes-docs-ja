---
title: Kubernetes オブジェクト管理に宣言型の設定情報ファイルを使う
content_template: templates/concept
weight: 40
---

{{% capture overview %}}
ディレクトリ内の設定情報ファイルに複数のオブジェクトを書くことで、Kubernetes オブジェクトの作成・更新・削除ができます。
また、 `kubectl apply` を使うと必要に応じてオブジェクトの作成と更新ができます。
この手法では、変更がオブジェクト設定情報ファイルには反映されず、稼働中のオブジェクトが書き込まれたデータを保持します。
{{% /capture %}}

{{% capture body %}}

## トレードオフ {#trade-offs}

`kubectl` ツールは３つのオブジェクト管理をサポートします。

* 命令型コマンド
* 命令型オブジェクト設定情報
* 宣言型オブジェクト設定情報

それぞれのオブジェクト管理手法の利点と欠点に関する議論は、[Kubernetes オブジェクト管理](/jp/docs/concepts/overview/object-management-kubectl/overview/) をご覧ください。

## 始める前に {#before-you-begin}

宣言型オブジェクト設定情報には、Kubernetes オブジェクト定義と設定情報に関するしっかりとした理解が必要です。
準備ができていなければ、以下のドキュメントを読み終えてください。

- [Kubernetes オブジェクトを命令型コマンドで管理](/jp/docs/concepts/overview/object-management-kubectl/imperative-command/)
- [Kubernetes オブジェクトを設定情報ファイルを使って命令型で管理](/docs/concepts/overview/object-management-kubectl/imperative-config/)

この文章では、用語の定義を次のようにしています:

- *オブジェクト設定情報ファイル（object configuration file） / 設定情報ファイル（configuration file）* ：Kubernetes オブジェクトに関する設定情報を定義したファイルです。このトピックでは設定情報ファイルを `kubectl apply` に渡す方法法を紹介します。設定情報ファイルは、通常 Git のようなソース管理システム（source control）に保管します。
* *稼働オブジェクト設定情報（live object configuration） / 稼働設定情報（live configuration） * ：オブジェクトの稼働設定情報の値とは、 Kubernetes クラスタ側から見えるものです。これらは Kubernetes クラスタ・ストレージ（記憶装置）、通常は etcd に保管されます。
* *宣言型設定情報の作者（declarative configuration writer） / 宣言型の作者（declarative writer）* ：ソフトウェア構成要素（コンポーネント）の稼働オブジェクトを更新する人です。このトピックで扱うのは、この作者が参照するためであり、オブジェクト設定情報ファイルを変更して `kubectl apply` を実行して設定を反映します。

## オブジェクトの作成方法 {#how-to-create-objects}

`kubectl apply` を使い、指定したディレクトリ内にある設定情報ファイルで定義した全オブジェクトを作成します。
ただし既に存在しているものは除外します。

```shell
kubectl apply -f <ディレクトリ>/
```

各オブジェクトには `kubectl.kubernetes.io/last-applied-configuration: '{...}'` アノテーション（注釈）が設定されています。アノテーション（注釈）にはオブジェクト作成時に使われたオブジェクト設定情報ファイルの内容を含んでいます。

{{< note >}}
**メモ：** `-R` フラグはディレクトリを再帰的に処理します（訳者注：下層にあるディレクトリもまとめて処理します）。
{{< /note >}}

こちらがオブジェクト構成情報ファイルの例です:

{{< code file="simple_deployment.yaml" >}}

`kubectl apply` を使ってオブジェクトを作成します:

```shell
kubectl apply -f https://k8s.io/docs/concepts/overview/object-management-kubectl/simple_deployment.yaml
```

`kubectl get` を使って設定情報を表示します:

```shell
kubectl get -f https://k8s.io/docs/concepts/overview/object-management-kubectl/simple_deployment.yaml -o yaml
```

出力が表示しているのは  `kubectl.kubernetes.io/last-applied-configuration` アノテーション（注釈）が稼働設定情報に書き込まれた時のものであり、設定情報ファイルと一致します:


```yaml
kind: Deployment
metadata:
  annotations:
    # ...
    # This is the json representation of simple_deployment.yaml
    # It was written by kubectl apply when the object was created
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

## オブジェクトの更新方法 {#how-to-update-objects}

`kubectl apply` を使って、ディレクトリで定義したすべてのオブジェクトの更新もできます。更新対象には既存のオブジェクトも含みます。この方法は以下のように進めます：

1. 稼働設定情報に含まれる設定情報ファイルに記載されたフィールドをセットする。
1. 稼働設定情報に含まれる設定情報ファイルから、不要なフィールドを削除する。

```shell
kubectl apply -f <ディレクトリ>/
```

{{< note >}}
**メモ：** `-R` フラグを追加するとディレクトリを再帰的に処理します。
{{< /note >}}

こちらは設定情報ファイルの例です:

{{< code file="simple_deployment.yaml" >}}

`kubectl apply` を使ってオブジェクトを作成します:

```shell
kubectl apply -f https://k8s.io/docs/concepts/overview/object-management-kubectl/simple_deployment.yaml
```

{{< note >}}
**メモ：** インストール用途のため、ディレクトリではなく１つの設定情報ファイルを参照するコマンドを事前に実行します。
{{< /note >}}

`kubectl get` を使って稼働設定情報を表示します。

```shell
kubectl get -f https://k8s.io/docs/concepts/overview/object-management-kubectl/simple_deployment.yaml -o yaml
```

出力が表示しているのは  `kubectl.kubernetes.io/last-applied-configuration` アノテーション（注釈）が稼働設定情報に書き込まれた時のものであり、設定情報ファイルと一致します:

```yaml
kind: Deployment
metadata:
  annotations:
    # ...
    # This is the json representation of simple_deployment.yaml
    # It was written by kubectl apply when the object was created
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

`kubectl scale` を使い、稼働設定情報の `replicas` フィールドを直接更新します。
ここでは `kubectl apply` を使いません：

```shell
kubectl scale deployment/nginx-deployment --replicas=2
```

`kubectl get` を使い、稼働設定情報を表示します。

```shell
kubectl get -f https://k8s.io/docs/concepts/overview/object-management-kubectl/simple_deployment.yaml -o yaml
```

出力が表示するのは、 `replicas` フィールドが 2 に設定されており、 `last-applied-configuration` （最後に適用された変更）アノテーションには `replicas` を含みません：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # note that the annotation does not contain replicas
    # because it was not updated through apply
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  replicas: 2 # written by scale
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        # ...
        name: nginx
        ports:
        - containerPort: 80
      # ...
```

`simple_deployment.yaml` 設定情報ファイルを更新し、イメージを `nginx:1.7.9` から `nginx:1.11.9` に変更します。
そして、 `minReadySeconds` フィールドを削除します。

{{< code file="update_deployment.yaml" >}}

設定情ファイルに対する変更を反映します。

```shell
kubectl apply -f https://k8s.io/docs/concepts/overview/object-management-kubectl/update_deployment.yaml
```

`kubectl get` を使って稼働設定情報を表示します:

```
kubectl get -f https://k8s.io/docs/concepts/overview/object-management-kubectl/simple_deployment.yaml -o yaml
```

出力が表示するのは、以下の稼働設定情報に対する変更です:

- `replicas` （複製数）フィールドは `kubectl scale` で設定された 2 の値を保持。設定情報ファイルでは省略されている可能性がある。
- `image` （イメージ）フィールドは `nginx:1.7.9` から `nginx:1.11.9` に変更。
- `last-applied-configuration` （最後に適用された変更）アノテーション（注釈）は、新しいイメージに更新されている。
- `minReadySeconds` （最小待機秒）フィールどがクリアされる。
- `last-applied-configuration` （最後に適用された変更）アノテーション（注釈）には `minReadySeconds` フィールドが無くなる。


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # The annotation contains the updated image to nginx 1.11.9,
    # but does not contain the updated replicas to 2
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.11.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
    # ...
spec:
  replicas: 2 # Set by `kubectl scale`.  Ignored by `kubectl apply`.
  # minReadySeconds cleared by `kubectl apply`
  # ...
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.11.9 # Set by `kubectl apply`
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

{{< warning >}}
**警告：** ``kubectl apply` は命令型オブジェクト設定コマンドの `create` および `replace` との混在をサポートしていません。
これは `create` と `replace` は  `kubectl.kubernetes.io/last-applied-configuration` を保持しないからです。 
`kubectl apply` は数を更新するために使います。
{{< /warning >}}

## オブジェクトの削除方法 {#how-to-delete-object}

オブジェクトを削除するには `kubectl apply` で管理する２つの方法があります。

### 推奨： `kubectl delete -f <filename>` {#recommended}

オブジェクトの削除には、命令型コマンドを使って直接削除する手法が推奨されています。これは何を削除するかの対象が明確だからです。また、ユーザの意図しない削除をより減らすためです。

```shell
kubectl delete -f <ファイル名>
```

### 別の方法：: `kubectl apply -f <ディレクトリ/> --prune -l your=label` {#alternative}

何を行おうとしているか理解している場合のみ使ってください：

{{< warning >}}
**警告：**  `kubectl apply --prone` はアルファです。今後のリリースでは後方互換が変更される場合があります。
{{< /warning >}}

{{< warning >}}
**警告：** このコマンドを使うにあたり、意図せずオブジェクトを削除しないよう、注意が必要です。
{{< /warning >}}

`kubectl delete` で削除するオブジェクトを指定する代わりに、ディレクトリから設定情報ファイルを削除してから  `kubectl apply` を使っての削除もできます。
`--prune` クエリを指定すると、API サーバはオブジェクト設定譲歩宇ファイルと現在のオブジェクト設定が一致するかどうかを比較し、すべてのオブジェクトから一致するラベルを返します。
もしもオブジェクトとクエリ（問い合わせ）が一致し、ディレクトリ内の設定情報ファイルと一致しなければ、また `last-applied-configuration` アノテーション（注釈）を持っていれば削除されます。

{{< comment >}}
TODO(pwittrock): We need to change the behavior to prevent the user from running apply on subdirectories unintentionally.
{{< /comment >}}


```shell
kubectl apply -f <ディレクトリ/> --prune -l <ラベル>
```


{{< warning >}}
**警告：**  prune の実行を指定すべきなのは、オブジェクト設定情報ファイルを含むルートディレクトリです。
サブディレクトリに対して実行すると、 `-l <ラベル>` で指定したラベル選択クエリの結果が一致しないため、意図しないオブジェクトの削除を引き起こします。
そのため、サブディレクトリでは実行しないでください。
{{< /warning >}}

## オブジェクトを表示するには {#how-to-viewan-object}

`kubectl get` に `-o yaml` を指定して、稼働オブジェクトの設定情報を表示できます：

```shell
kubectl get -f <ファイル名|url> -o yaml
```

## 差分の確認と変更を反映するには {#how-apply-calculates-differences-and-merges-changes}

{{< caution >}}
**注意：** *patch* （パッチ）とは更新作業であり、オブジェクト全体ではなく、オブジェクトの特定のフィールドを対象としています。
これにより、オブジェクトをまず読み込むのではなく、オブジェクト上の特定フィールドにあるセットのみを更新できます。
{{< /caution >}}

(TODO)

{{% /capture %}}


