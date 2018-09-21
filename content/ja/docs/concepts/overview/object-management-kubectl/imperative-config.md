---
title: Kubernetes オブジェクト管理に命令型の設定ファイルを使う
content_template: templates/concept
weight: 30
---

{{% capture overview %}}
`kubectl` コマンドライン・ツールを使うと、YAML または JSON で書かれたオブジェクト設定情報の記述に従い、Kubernetes オブジェクトを作成、更新、削除できます。
このドキュメントは設定情報ファイルを使ったオブジェクトの定義と管理方法を説明します。

{{% /capture %}}

{{% capture body %}}

## トレードオフ {#trade-offs}

`kubectl` ツールは３種類のオブジェクト管理をサポートしています:

* 命令型コマンド
* 命令型オブジェクト設定情報
* 宣言型オブジェクト設定情報

それぞれのオブジェクト管理手法の利点と欠点に関する議論は、[Kubernetes オブジェクト管理](/jp/docs/concepts/overview/object-management-kubectl/overview/) をご覧ください。

## オブジェクトの作成方法 {#how-to-create-objects}

`kubectl create -f` を使い、設定情報ファイルにあるオブジェクトを作成できます。
詳細は [kubernetes API リファレンス（参考情報）](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/) をご覧ください。

- `kubectl create -f <ファイル名|url>`

## オブジェクトの更新方法 {#how-to-update-objects}

{{< warning >}}
**警告：** オブジェクトの更新に `replace` （置換）コマンドを使うと、設定情報ファイル内に spec （仕様）の記述がないパーツをすべて削除します。
そのため、サービス型 `LoadBalancer` のようなクラスタによって管理される部分を specs のオブジェクトで管理すべきではないでしょう。
この `externalIPs` フィールドは設定情報ファイルとは独立して管理すべきです。
独立して管理するフィールドは `replace` によって削除されるのを防ぐため、別の設定情報ファイルにコピーしておく必要があります。
{{< /warning >}}

`kubectl replace -f` を使い、設定情報ファイルに従って稼働中のオブジェクトを更新します。

- `kubectl replace -f <ファイル名|url>`

## オブジェクトの削除方法 {#how-to-delete-object}

`kubectl delete -f` を使って設定情報ファイルに記述があるオブジェクトを削除します。

- `kubectl delete -f <ファイル名|url>`

## オブジェクトを表示するには {#how-to-view-an-object}

 `kubectl get -f` を使い、設定情報ファイル内に記述があるオブジェクトの情報を表示できます。

- `kubectl get -f <ファイル名|url> -o yaml`

`-o yaml` フラグを指定すると、オブジェクト設定情報のすべてを表示します。 `kubectl get -h` でオプションの一覧を表示します。

## 制限事項 {#limitations}

`create` 、 `replace` 、 `delete` コマンドは各オブジェクトの設定情報を完全に定義し、それを設定情報ファイルに記録している場合のみ動作します。
しかし、稼働中のオブジェクトが更新するか、更新したものを設定情報ファイルに反映しなければ、更新を次回実行すると `replace` （置換）が処理されます。
これによって起こるのは、 HorizontalPodAutoscaler（水平ポッド・スケーラ）のようなコントローラは稼働中のオブジェクトに対して直接更新を試みます。
以下は例です:

1. オブジェクトを構成情報ファイルから作成します。
1. （ファイルではない）別の方法を使い、オブジェクトのフィールドをいくつか変更します。
1. 設定情報ファイルでオブジェクトを置き換えます（replace）。手順２で作成した別の方法による変更は消えます。

同じオブジェクトを複数回書けるようにするには、オブジェクトの管理に `kubectl apply` で管理する必要があります。

## 設定情報を保存せずに URL でオブジェクトの作成と更新{#creating-and-editing-an-object-from-a-url-without-saving-the-configuration}

オブジェクト構成情報ファイルの URL があると想定します。
`kubectl create --edit` を使い、既に作成したオブジェクトに対する設定情報を変更可能です。
これが特に役立つのはチュートリアルとタスクです。
読み手に対し、設定情報ファイルを使って変更内容を指示できます。

```sh
kubectl create -f <url> --edit
```

## 命令型コマンドから命令型オブジェクト設定への移行 {#migrating-from-imperative-commands-to-imperative-object-configuration}

命令型コマンドから命令型オブジェクト設定に移行するには、いくつかの手動手順があります。

1. 稼働中のオブジェクトを設定情報ファイルに出力（export）します。
```sh
kubectl get <種類>/<名前> -o yaml --export > <種類>_<名前>.yaml
```

1. オブジェクト設定情報ファイルから、手動で status フィールドを削除します。

1. 以降のオブジェクト管理には `replace` のみを使います。
```sh
kubectl replace -f <種類>_<名前>.yaml
```

## コントローラ・セレクタとポッド・テンプレートの定義 {#defining-controller-selectors-and-podtemplate-labels}

{{< warning >}}
**警告：** コントローラ上のセレクタの更新は、非常に非推奨です。
{{< /warning >}}

推奨する手法は、全く手を加えない（イミュータブルな）ポッド・テンプレート（PodTemplate）ラベルを１つだけ定義し、何も意味を持たないコントローラ・セレクタのために使う方法です。

サンプル・ラベル:

```yaml
selector:
  matchLabels:
      controller-selector: "extensions/v1beta1/deployment/nginx"
template:
  metadata:
    labels:
      controller-selector: "extensions/v1beta1/deployment/nginx"
```

{{% /capture %}}

{{% capture whatsnext %}}
- [Kubernetes オブジェクトを宣言型コマンドで管理](/jp/docs/concepts/overview/object-management-kubectl/imperative-command/)
- [Kubernetes オブジェクトを設定叙法（宣言型）で管理](/jp/docs/concepts/overview/object-management-kubectl/declarative-config/)
- [Kubectl コマンドリファレンス（参考情報）](/jp/docs/reference/generated/kubectl/kubectl/)
- [Kubernetes API リファレンス（参考情報）](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/)

{{% /capture %}}


