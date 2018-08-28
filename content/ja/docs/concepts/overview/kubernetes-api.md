---
title: KubernetesのAPI
content_template: templates/concept
weight: 30
---

{{% capture overview %}}

API規約全体については[API conventions doc](https://git.k8s.io/community/contributors/devel/api-conventions.md)に記載されています。

APIのエンドポイント、リソースタイプ、サンプルは[API Reference](/docs/reference)に記載されています。

APIへのリモートアクセスについては、[access doc](/docs/admin/accessing-the-api)で議論されています。

Kubernetes APIはシステムの宣言的な設定スキーマも提供しています。[kubectl](/docs/reference/kubectl/overview)コマンドラインツールは、APIオブジェクトを作成、更新、削除、取得のために使用することができます。

KubernetesはAPIリソースの状態をシリアライズしたものも保存しています。(現在は[etcd](https://coreos.com/docs/distributed-configuration/getting-started-with-etcd/))。

Kubernetes自体はAPIを介して相互に作用する複数のコンポーネントに分解することができます。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## APIの変更

経験上、成功したシステムはどんなものであれ、新しい使用方法や、既存のシステムの変更によって成長、あるいは変更の必要が生じてきます。なので、KubernetesのAPIも継続的に変更・成長して行くだろうと想定しています。しかしながら、私たちは長い期間の間、既存のクライアントとの互換性を壊すつもりはありません。総合的に、新しいAPIリソースや新しいリソースのフィールドは頻繁に追加される可能性があります。リソースやフィールドの削除については、[API deprecation policy](/docs/reference/using-api/deprecation-policy/)に従う必要があります。

互換性のある変更の構成と、どのようにAPIを変更するのかについては、 [API change document](https://git.k8s.io/community/contributors/devel/api_changes.md)で詳しく説明しています。

## OpenAPIとSwagger定義

完全なAPIの詳細は[Swagger v1.2](http://swagger.io/)と[OpenAPI](https://www.openapis.org/)で文書化されています。KubernetesのAPIサーバ("master"のことです)はSwagger v1.2のAPI仕様書を`/swaggerapi`で公開しています。

Kubernetes 1.10からは、OpenAPI仕様書が単一の`/openapi/v2`というエンドポイントで提供されています。フォーマット毎のエンドポイント(`/swagger.json`, `/swagger-2.0.0.json`, `/swagger-2.0.0.pb-v1`, `/swagger-2.0.0.pb-v1.gz`)は非推奨で、Kubernetes 1.14で削除されます。

形式はHTTPヘッダで指定できます:

  ヘッダ | 値
------ | ---------------
Accept | `application/json`, `application/com.github.proto-openapi.spec.v2@v1.0+protobuf` (`*/*`の規定のContent-Typeは`application/json`で、このヘッダは渡しません)
Accept-Encoding | `gzip` (このヘッダを指定しないこともできます)

**OpenAPI仕様書取得の例**:

  Before 1.10 | Starting with Kubernetes 1.10
----------- | -----------------------------
GET /swagger.json | GET /openapi/v2 **Accept**: application/json
GET /swagger-2.0.0.pb-v1 | GET /openapi/v2 **Accept**: application/com.github.proto-openapi.spec.v2@v1.0+protobuf
GET /swagger-2.0.0.pb-v1.gz | GET /openapi/v2 **Accept**: application/com.github.proto-openapi.spec.v2@v1.0+protobuf **Accept-Encoding**: gzip


Kubernetesはクラスタ内部のコミュニケーションで主に使用するために、Protobufベースのシリアライゼーションフォーマットを実装しています。これは[design proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/protobuf.md)で文書化されており、各スキーマに対するIDLファイルはAPIオブジェクトを定義しているGoのパッケージ内にあります。

## API versioning

フィールドの削除やリソースの再構成をわかりやすくするため、Kubernetesは`/api/v1`や`/apis/extensions/v1beta1`等のような個別のパスで表現される、複数のAPIバージョンをサポートしています。

リソースやフィールドレベルでは無く、APIレベルでバージョンを選択し、APIが明確で一貫した動作をし、削除された、あるいは実験的なAPIへのアクセスを制御できるようにしました。
JSONおよびProtobufシリアライゼーションスキーマはスキーマの変更に関して、同様のガイドラインに従い、以下すべての説明は両方のフォーマットに適用されます。

APIのバージョニングと、ソフトウェアのバージョニングは間接的にしか関連していないことに注意してください。[API and release
versioning proposal](https://git.k8s.io/community/contributors/design-proposals/release/versioning.md)では、APIバージョニングとソフトウェアバージョニングの関係について説明しています。

異なるAPIバージョンは、異なるレベルの安定性と、異なるレベルのサポートを意味しています。それぞれのレベルの詳細な基準は[API Changes documentation](https://git.k8s.io/community/contributors/devel/api_changes.md#alpha-beta-and-stable-versions)で説明されています。以下に要約を掲載します:

- Alphaレベル:
  - バージョン名称に`alpha`を含む (例:`v1alpha1`)。
  - バグを含んでいるかもしれません。機能を有効化することで、バグが表面化する可能性があります。デフォルトでは無効化されています。
  - 予告なしにサポートが打ち切られる場合があります。
  - 予告なしに互換性のない変更が加えられる場合があります。
  - 長期にわたり使用すると、サポートが無く、リスクが上昇するため、短期的なテストクラスタでのみ使用することを推奨します。
- Betaレベル:
  - バージョン名称に`beta`を含む (例:`v2beta3`)。
  - コードは十分テストされています。 機能を有効化することは安全だと考えられ、デフォルトで有効です。
  - 機能全体のサポートが無くなることはないでしょうが、細かい部分は変更になるかもしれません。
  - その後のbetaやstableリリースでは、互換性のない方法でオブジェクトのスキーマの意味論が変更される可能性があります。その場合、次のバージョンへの移行方法を提供します。この手順は、APIオブジェクトの削除、編集、再作成を必要とするかもしれません。また、編集作業はいくらか考えなければならないかもしれません。これにより、その機能に依存するアプリケーションのダウンタイムが必要となる場合があります。
  - 将来的なリリースで互換性のない変更が加えられる可能性があるため、ビジネスクリティカルではない用途でのみ使用することを推奨します。個別にアップグレードされる可能性のある複数のクラスタを持っている場合、この制限はある程度緩和できるかもしれません。
  - **beta機能を使って、フィードバックをください！一度betaを終了すると、それ以上の変更を加えるのは現実的ではない可能性があります**
- Stableレベル:
  - `X`が整数であるような、`vX`という名称のバージョン。
  - 安定バージョンの機能は多くの後続バージョンでも使用できます。

## APIグループ

KubernetesのAPIを拡張することを容易にするため、[*APIグループ*](https://git.k8s.io/community/contributors/design-proposals/api-machinery/api-group.md)を実装しています。
APIグループはRESTパスとシリアライズされたオブジェクトの`apiVersion`フィールドで特定されます。

現時点で、いくつかのAPIグループが使用されています:

1. *core*グループ。 *legacyグループ*として参照されることもあります。RESTパス`/api/v1`にあり、`apiVersion: v1`を使用します。

1. 名前付きのグループは`/apis/$GROUP_NAME/$VERSION`というRESTパスにあり、, `apiVersion: $GROUP_NAME/$VERSION`を使用します。
   (e.g. `apiVersion: batch/v1`).  APIグループの完全なリストは、[Kubernetes API reference](/docs/reference/)にあります。


[custom resources](/docs/concepts/api-extension/custom-resources/)を使用してAPIを拡張するための、2つのサポートされたパスがあります:

1. [CustomResourceDefinition](/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/)は、非常に基本的なCRUDが必要なユーザ向けです。
1. 近日公開予定: Kubernetes APIの完全なセマンティクスが必要なユーザは独自のapiserverを実装でき、[aggregator](https://git.k8s.io/community/contributors/design-proposals/api-machinery/aggregated-api-servers.md)を使用することで、シームレスにクライアントで使用できます。


## APIグループを有効化する

特定のリソースおよびAPIグループはデフォルトで有効になっています。APIグループは、apiserverで`--runtime-config`を設定することで有効または無効にすることができます。
`--runtime-config`はカンマ区切りの値を指定できます。例えば、batch/v1を無効化するには`--runtime-config=batch/v1=false`を、batch/v2alpha1を有効化するには `--runtime-config=batch/v2alpha1`を設定します。
このフラグはapiserverの実行時設定としてカンマで区切られたkey=value形式の値のセットを受け入れます。

重要: APIグループまたはリソースを有効化あるいは無効化するには、`--runtime-config`の設定を取得するため、apiserverとcontroller-managerの再起動が必要です。

## グループ内のリソースを有効化する

DaemonSet、Deployment、HorizontalPodAutoscaler、Ingress、Job、ReplicaSetはデフォルトで有効です。
その他の拡張リソースはapiserverで`--runtime-config`を設定して有効化することができますう。`--runtime-config`はカンマ区切りの値で指定できます。例えば、DeploymentとIngressを無効化するには`--runtime-config=extensions/v1beta1/deployments=false,extensions/v1beta1/ingress=false`を指定します。

{{% /capture %}}
