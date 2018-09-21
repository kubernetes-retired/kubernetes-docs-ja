---
title: 名前
content_template: templates/concept
weight: 20
---

{{% capture overview %}}
Kubernetes REST API のすべてのオブジェクトは、名前と UID によって明確に識別します。

ユニークではないユーザが指定する属性のためには、Kubernetes は [ラベル（label）](/jp/docs/concepts/overview/working-with-objects/labels/) と [アノテーション（annotations）](/jp/docs/concepts/overview/working-with-objects/annotations/) を提供しています

名前と UID に対する厳密な文法規則は [identifiers design doc](https://git.k8s.io/community/contributors/design-proposals/architecture/identifiers.md) をご覧ください。
{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## 名前 {#names}

クライアントが提供する文字列であり、オブジェクトを `/api/v1/pods/some-name` のようなリソース URL で参照します。

特定のオブジェクトに対して、名前を指定できるのは１度だけです。しかし、オブジェクトを削除した後であれば、同じ名前を持つ新しいオブジェクトを作成できます。

慣例として、Kubernetes のリソース名は最大 253 文字で、アルファベットの小文字、 `-` 、 `.` を含みますが、特定のリソースでは特別な制限を持ちます

## UID {#uids}

Kubernetes はユニークにオブジェクトを識別するため、システムが生成した文字列を使います。

作成されたすべてのオブジェクトは Kubernetes クラスタ上に存在し続ける間、個別に UID を持ちます。
これは似たような古いものと区別するためです。

{{% /capture %}}