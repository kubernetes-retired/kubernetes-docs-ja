---
title: ラベルとセレクタ
content_template: templates/concept
weight: 40
---

{{% capture overview %}}
_ラベル（Labels）_ とはキー・バリューの組み合わせであり、ポッドのようなオブジェクトに付けます。
ラベルを使う目的は、オブジェクトの属性を明確に識別するためです。
つまり、ユーザにとっては意味や関連性がありますが、コア・システムにとっては直接的な意味はありません。
オブジェクトの集まりを整理・選ぶためにラベルを使えます。
オブジェクトの作成時にラベルを付けられますし、後から何時でも追加や変更ができます。
各オブジェクトはキー・バリューのセットでラベルで定義できます。
各キーはオブジェクトごとにユニークな必要があります。

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

効果的なクエリ（問い合わせ）や監視で使うため、最終的にはラベルの一覧と逆引き一覧を UI や CLI 等で並べ替えたりグループ化したりに使えます。
識別する以外の用途でラベルを汚染するつもりはありません。
特に大規模なデータ構造においては尚更です。
識別以外の情報は [アノテーション（annotation）](/jp/docs/concepts/overview/working-with-objects/annotations/) を使って記録すべきです。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## 動機 {#motivation}

ラベルの利用により、ユーザは自分の組織構造システム・オブジェクトと緩やかに連動して関連付けられます。
これには、クライアント側では割り当てに関する情報を持つ必要はありません。

サービスのデプロイとバッチ処理パイプラインでは、頻繁に多次元要素を扱います（例：複数に分割したデプロイ、複数のリリース・トラック、ティアごとの複数のマイクロサービス）。
管理では分野横断的な運用が頻繁に求められます。
運用では、厳密な階層構造をカプセル化して表現するのではありません。
むしろ、ユーザではなく基盤（インフラストラクチャ）に基づき、特別に固定した階層構造を用います。

ラベルの例:

   * `"release" : "stable"`, `"release" : "canary"`
   * `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
   * `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
   * `"partition" : "customerA"`, `"partition" : "customerB"`
   * `"track" : "daily"`, `"track" : "weekly"`

これらはラベルとして一般的に使われる例です。
そのため、自分が使いやすいように自由に作成できます。
ラベルのキーは対象となるオブジェクト内ではユニークな必要がありますので、この点はご注意ください。

## 構文と文字セット {#syntax-and-character-set}

_ラベル_ はキー・バリューの組み合わせです。
有効なラベル・キーには２つの部分あります。
オプションのプリフィックスと名前を分けるのはスラッシュ（`/`）です。
名前の部分に必要なのは、63文字以内の文字列であり、冒頭・末尾が英数字（`[a-z0-9A-Z]`）です。
途中には、ダッシュ（`-`）、アンダースコア（`_`）、ドット（`.`）と英数字が使えます。
プレフィックスはオプションです。
もしも指定した場合は、プレフィックスは DNS サブドメインの必要があります。
つまり、DNS が続くラベルはドット（`.`）で区切る必要があり、合計で 253文字を越えられず、後にはスラッシュ（`/`）が続きます。
もしもプリフィックスがなければ、ラベル・キーはユーザにとってプライベートなものとみなされます。
システム構成要素は自動化されています（例：`kube-scheduler`、 `kube-controller-manager`、 `kube-apiserver`、 `kubectl`、あるいは他のサードパーティによる自動設定） 。
エンドユーザがオブジェクトにラベルを追加するには、プレフィックスの指定が必須です。
`kubernetes.io/` プレフィックスは Kubernetes の中心となる構成要素のために予約済みです。

有効なラベルの値は63文字以下の文字か、内容がからであるか、冒頭・末尾が英数字（`[a-z0-9A-Z]`）です。
途中には、ダッシュ（`-`）、アンダースコア（`_`）、ドット（`.`）と英数字が使えます。

## ラベル・セレクタ {#label-selectors}

[名前や UID](/jp/docs/concepts/overview/working-with-objects/names/) とは違い、ラベルは一意性を提供しません。
通常、多くのオブジェクトが同じラベルを持つのが予想されます。

_ラベル・セレクタ（label selector）_ を経由すると、クライアントやユーザはオブジェクトの集まりを識別できます。
ラベル・セレクタは Kubernetes におけるコアなグループ化単位（プリミティブ）です。

API は現時点で２種類のセレクタをサポートしています。
等号を使うもの（ _quality-based_）と、組み合わせを使うもの（ _set-based_ ）です。
ラベル・セレクタはコンマ記号で分割する複数の _必要条件_ で構成されます。
複数の必要条件のすべてをコンマ記号で記述しますが、これは論理演算子 _AND_ （`&&`）として扱われます。

空のラベル・セレクタ（使うには少なくとも0が必要）は、コレクション内にあるすべてのオブジェクトから選びます。

ヌル（null）ラベル・セレクタ（オプションのセレクタ・フィールドのみの可能性）はオブジェクトがないものを選びます。


{{< note >}}
**メモ**：２つのコレクタを持つラベル・セレクタは、名前空間内で重複できないだけでなく、お互いの競合もできません。
{{< /note >}}

### 等号を使う（ _Equality-based_ ）要件 {#equality-based-requirement}

ラベルのキーとバリューのフィルタに等号や不等号を使えます。
条件が一致するオブジェクトは、指定したラベル条件をすべて満たす必要があります。
追加のラベルがあっても同様です。
３つの演算子  `=` 、 `==` 、 `!=` が許可されています。
はじめの２つは等しい意味を表し（そして簡単にした表記です）、最後のは等しくない意味を表します。
例:

```
environment = production
tier != frontend
```

前者で選択するのは、すべてのリソース中からキーが `environment` であり、値が `production` と等しいものです。
後者で選択するのは、キーが `tier` に一致しますが、値が `frontend`ではないものかつ、すべてのリソースから `tier` キーがないものです。
`production` にあるリソースから `frontend` を除くフィルタを使うには、コンマ区切りを使います:`environment=production,tier!=frontend` 。

使用例としては、ポッドを稼働するノードの条件を指定するためです。
たとえば、以下はサンプル・ポッドが動作するノードは、ラベル  "`accelerator=nvidia-tesla-p100`" を選びます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

### セットを使う（ _Set-based_ ）要件 {#set-based-requirement}

セットを使う（ _Set-based_）要件により、セットした値に一致するキーでフィルタできます。
３種類の演算子 `in` 、 `notin` 、 `exist`（キー識別子のみ）をサポートします。
例:

```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

１つめの例では、すべてのリソースからキーが `environment` に一致し、かつ、値が `production` か `qa` のものを選びます。
２つめの例はすべてのリソースからキーが `tier` に一致し、値が `frontend` または `backend` ではないものを選ぶのと、すべてのリソースのうち `tier` キーのないラベルを持つものです。
３つめの例はすべてのリソースのうち、ラベルのキーに `partition` を含むものを選びますが、値があるかどうかはチェックしません。
４つめの例はすべてのリソースからラベルのキーに `partition` がないものを選びます。
擬似的にコンマ記号を _AND_ 演算子として使えます。
そのため、 `patition` キー（値が何であれ）と、 `environment` が `qa` でない条件を達成するには `partition,environment notin (qa)` を使います。
セットを使うラベル・セレクタ `environment=production` は、 `environment in (production)` と同じです。
つまり `!=` と `notin` は同じです。

セットを使う条件と等号を使うものは、組み合わせて利用できます。
例：`partition in (customerA, customerB),environment!=qa` 。


## API

### LIST と WATCH フィルタリング {#list-and-watch-filtering}

ラベル・セレクタで LIST （一覧表示）と WATCH （監視）の処理は、オブジェクト・セットの問い合わせパラメータを使ってフィルタできます。

  * _equality-based_ 必要条件: `?labelSelector=environment%3Dproduction,tier%3Dfrontend`
  * _set-based_ 必要条件: `?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`

ラベル・セレクタ・スタイルの両方を、 REST クライアントを通してリソースの一覧表示または監視のために使えます。
たとえば、 `kubectl` で `apiserver` を対象にして  _equality-based_ を利用する方法の１つは:

```shell
$ kubectl get pods -l environment=production,tier=frontend
```

あるいは _set-based_ 必要条件を使います:

```shell
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
```

既に言及している _set-based_ 必要条件については、記法が多彩です。
たとえば、 _OR_ 演算子を値に埋め込むには：

```shell
$ kubectl get pods -l 'environment in (production, qa)'
```

あるいは _exists_ 演算子で一致しないものに制限するには:

```shell
$ kubectl get pods -l 'environment,environment notin (frontend)'
```

## API オブジェクト設定リファレンス {#set-references-in-api-objects}

[`services`（サービス）](/jp/docs/concepts/services-networking/service/) と [`replicationcontrollers`（レプリケーション・コントローラ）](/jp/docs/concepts/workloads/controllers/replicationcontroller/) といった一部の Kubernetes オブジェクトは、ラベル・セレクタとして [ポッド](/jp/docs/concepts/workloads/pods/pod/) のような他のリソース指定も可能です。

#### サービスとレプリケーション・コントローラ {#service-and-replicationcontroller}

ポッドの集まりとは、ラベル・セレクタで `service` と定義したものが対象です。同様に、ポッドの集まりとはラベル・セレクタで   `replicationcontroller`  として定義したものが管理対象とも言えるでしょう。

ラベル・セレクタがサポートしているのは、 `json` や `yaml`  ファイルを使って割り当てる（mapする）オブジェクトか、_equality-based_ 必要条件のセレクタのみの両方です。

```json
"selector": {
    "component" : "redis",
}
```

または

```yaml
selector:
    component: redis
```

このセレクタ（個々の `json` か `yaml` 形式）は `component=redis` や  `component in (redis)` と同等です。

### set-based 必要条件をサポートするリソース {#resources-that-support-set-based-requirements}

[`Job`（ジョブ）](/jp/docs/concepts/jobs/run-to-completion-finite-workloads/)、 [`Deployment`（デプロイメント）](/jp/docs/concepts/workloads/controllers/deployment/)、 [`Replica Set`（レプリカ・セット）](/jp/docs/concepts/workloads/controllers/replicaset/)、 [`Daemon Set`（デーモン・セット）](/jp/docs/concepts/workloads/controllers/daemonset/) のような新しいリソースは、 _set-based_ 必要条件も同様にサポートします：


```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

`matchLables` は `{key.value}` の組み合わせを割り当て（マップ）します。
`matchLables` 中の `{key.value}` が割り当てる（マップする）のは、`matchExpressions` 要素の `key` フィールドを「key」として、 `operator` を「In」として、 `values` フィールドのアレイを「Value」として扱うのと同等です。
`matchExpressions`  はポッド・セレクタが必要とするリストです。
有効な演算子には Notin、Exist、DoesNotExist が含まれます。
In と notIn の場合は value に空ではない値が必ず入っている必要があります。
すべての必要条件に `matchLables` と `matchExpressions` が含まれており、 AND も一緒に使えます。
並べた順番で条件を満たす必要があります。

#### ノードのセットを選択 {#selecting-sets-of-nodes}

ラベルの使用例としては、どのポッドがスケジュール可能かどうか制限する方法があります。
詳細は [ノード選択](/jp/docs/concepts/configuration/assign-pod-node/) をご覧ください。

{{% /capture %}}