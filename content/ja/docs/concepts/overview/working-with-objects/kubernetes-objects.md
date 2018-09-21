---
title: Kubernetes オブジェクトの理解
content_template: templates/concept
weight: 10
---

{{% capture overview %}}
このページは Kubernetes オブジェクトが Kubernetes API でどのような役割を持つのかを説明します。
そして、これらを `.yaml` 形式でも表現できます。
{{% /capture %}}

{{% capture body %}}
## Kubernetes オブジェクトの理解 {#understanding-kubernetes-object}

Kubernetes システム内では、*Kubernetes オブジェクト* とは永続的な存在です。
Kubernetes はこれらの存在を使い、クラスタの状態を説明します。
特に、これらで説明できるのは、以下の通りです。

* 実行中のコンテナ化されたアプリケーションは何か（そして、どのノードか）
* これらアプリケーションが利用可能な資源（リソース）
* 再起動ポリシー、更新、耐障害性（fault-tolerance）のようなアプリケーションがどのような挙動なのかに関連するポリシー。

Kubernetes オブジェクトとは「意図の記録」（record of intent）です。
ひとたびオブジェクトを作成したら、Kubernetes システムはオブジェクトが確実に存在し続けるよう、絶え間なく動作します。
オブジェクトの作成によって、Kubernetes システムに対してクラスタのワークロード（作業負荷）をどうしたいのかを効率的に伝えられます。
すなわち、これがクラスタの **期待状態（desired state）** です。

Kubernetes オブジェクトを取り扱うには、たとえばオブジェクトを作成、変更、削除をするには [Kubernetes API](/jp/docs/concepts/overview/kubernetes-api/) を使う必要があります。
例として、 `kubectl` コマンドライン・インターフェースを使うとすると、CLI は必要になる Kubernetes API コールを作成します。
また、[クライアント・ライブラリ](/jp/docs/reference/using-api/client-libraries/)のどれかを使う自分のプログラムを用い、Kubernetes API で直接の操作も可能です。

### オブジェクト仕様と状態 {#object-spec-and-status}

すべての Kubernetes オブジェクトには２つの入れ子になった（ネストした）オブジェクト・フィールドを含んでおり、これでオブジェクトの設定を管理します。
これはオブジェクトの *spec（仕様）* とオブジェクトの *status（状態）* です。 
*spec* とは、指定が必須であり、オブジェクトに対する *期待状態（desired state）* を記述します。
*status* にはオブジェクトの *実際の状態（actual state）* を記述し、こちらは Kubernetes システムによって提供および更新されます。
あらゆる時間があれば、Kubernetes はコントロール・プレーンの管理を活性化し、オブジェクトの実際の状態が、指定した期待状態と一致するようにします。

たとえば、Kubernetes Deployment （デプロイメント：配置）とは、クラスタ上で実行するアプリケーションを表すオブジェクトです。
Deployment（デプロイメント）の作成時には、Deploymet spec（配置仕様）を設定し、アプリケーションを稼働するのに３つの複製（レプリカ）が必要であると指定をするでしょう。
Kubernetes システムはこの Deplloyment spec を読み込み、任意アプリケーションの３つのインスタンス（実体）を起動します。
そして、状態が指定した spec （仕様）と一致するように更新し続けるのです。
もしもインスタンスのいずれかに障害が起これば（状態が変われば）、Kubernetes システムは仕様と状態の違いに対応し、正しい状態を保つ役割があります。
今回の障害が発生した例であれば、別のインスタンスを起動します。

オブジェクト仕様（object spec）、状態（status）、メタデータ（metadata）に関する詳しい情報は [Kubernetes API 仕様](https://git.k8s.io/community/contributors/devel/api-conventions.md)をご覧ください。

### Kubernetes オブジェクトの記述 {#describing-a-kubernetes-object}

Kubernetes でオブジェクトの作成時、オブジェクト仕様（spec）の指定が必要です。
これは期待状態の記述だけでなく、オブジェクトに関する基本的な情報（名前など）も記述します。
Kubernetes API を使ってオブジェクトを作成すると（直接、あるいは `kuvectl` を経由するかのどちらか）、API リクエストにはリクエスト・ボディに情報を JSON として含める必要があります。
**ほとんどの場合、 .yaml ファイルに書かれた情報を `kubectl` で送ります。
**  `kubectl` は API リクエストの作成時に、情報を変換します。

こちらは `.yaml` ファイル例です。
リクエスト・フィールドと Kubernetes Deployment に対するオブジェクト仕様が見えます：

{{< code file="nginx-deployment.yaml" >}}

Deployment の作成に前述の `.yaml` ファイルを作成する１つの方法は、 `kubectl` コマンドライン・インターフェースで [`kubectl create`](/jp/docs/reference/generated/kubectl/kubectl-commands#create) コマンドを使い、 `.yaml` ファイルを引数として渡します。
以下は例です:

```shell
$ kubectl create -f https://k8s.io/docs/concepts/overview/working-with-objects/nginx-deployment.yaml --record
```

この出力は次のようになります:

```shell
deployment "nginx-deployment" created
```

### 必要なフィールド {#required-fields}

Kubernetes オブジェクトを作成するための `.yaml` ファイルでは、以下フィールドの値に対する指定が必要です:

* `apiVersion` - このオブジェクトを作成するために、どのバージョンの Kubernetes API を使うか
* `kind` - 作成したいオブジェクトの種類は何か
* `metadata` - オブジェクトを一意に（ユニークに）識別するために役立つデータであり、 `name` 文字列、UID、オプションで `namespace` を含む

また、オブジェクトの `spec` フィールドも指定する必要があります。オブジェクト 'spec' の厳密な書式は各 Kubernetes オブジェクトとは異なり、そのオブジェクトの仕様をネスト化したフィールドも含みます。に Kubernetes を作って作成可能なすべてのオブジェクトに対する spec 書式（フォーマット）を探すには、 [Kubernetes API リファレンス](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/) が役立つでしょう。たとえば、`Pod` オブジェクトに対する  `spec` 書式は [こちら](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podspec-v1-core) にあります。また、 `Deployment` オブジェクトに対する `spec` 書式は[こちら](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#deploymentspec-v1-apps)にあります。

{{% /capture %}}

{{% capture whatsnext %}}
* [Pod](/jp/docs/concepts/workloads/pods/pod-overview/) のような、最も重要な基本 Kubernetes オブジェクトについて学びましょう。
{{% /capture %}}


