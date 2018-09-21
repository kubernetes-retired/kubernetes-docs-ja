---
title: Kubernetes オブジェクトを命令型コマンドで管理
content_template: templates/concept
weight: 20
---

{{% capture overview %}}
`kubectl` コマンドライン・ツールに組み込まれている命令型コマンド使い、Kubernetes オブジェクトを迅速に作成、更新、作成できます。
このドキュメントは、各コマンドの体系化した使い方と、どのようにして実行中のオブジェクトを管理するために使うかを説明します。
{{% /capture %}}

{{% capture body %}}

## トレードオフ {#trade-offs}

`kubectl` ツールは３種類のオブジェクト管理をサポートします：

* 命令型コマンド
* 命令型オブジェクト設定
* 宣言型オブジェクト設定

それぞれのオブジェクト管理の利点と欠点についての議論は、 [Kubernetes オブジェクト管理](/jp/docs/concepts/overview/object-management-kubectl/overview/) をご覧ください。

## オブジェクト作成の仕方 {#how-to-create-objects}

`kubectl` ツールは動詞で命令するコマンドをサポートします。
ほとんどの共通オブジェクト型の作成に対応しています。
Kubernetes オブジェクト型に不慣れなユーザも認識できるように、コマンドは名付けられています。

-  `run`（実行）：１つまたは複数のポッド内にコンテナを実行するために、新しいデプロイメント・オブジェクトを作成します。
-  `expose`（露出）：ポッドを横断して通信を負荷分散するために、新しいサービス・オブジェクトを作成します。
-  `autoscale`（オートスケール）：デプロイメントのような自動的に水平スケールを制御するために、新しいオートスケール・オブジェクトを作成します。

また、 `kubectl` ツールはオブジェクト型に依存する作成コマンドもサポートしています。
各コマンドは多くのオブジェクト型（objecttype）をサポートし、それぞれに明確な目的があります。
しかし、ユーザがオブジェクト型を作成するためには、それぞれのオブジェクト型の目的を理解する必要があります。

```
- `create <objecttype> [<subtype>] <instancename>`
```
- `create <オブジェクト型> [<サブ型>] <インスタンス名>`

オブジェクト型によってはサブ型を持ち、 `create` コマンドで指定できます。
たとえばサービスのオブジェクトには複数のサブ型があります。
サブ型に含まれるのは ClusterIP、LoadBalancer、NodePort です。
以下は NodePort サブ型があるサービスを作成する例です:

```shell
kubectl create service nodeport <自分のサービス名>
```

先の例にある `create service nodeport` コマンドでは、 `create service` コマンドというサブコマンドを呼び出しています。

サブコマンドがサポートしている引数やフラグを確認するには、 `-h` フラグを使えます。

```shell
kubectl create service nodeport -h
```

## オブジェクトを更新（アップデート）するには {#how-to-update-objects}

`kubectl` コマンドは、いくつかの共通する更新操作のために、動詞で命令するコマンドをサポートしています。
Kubernetes オブジェクト型に不慣れなユーザも認識できるように、コマンドは名前付けられています。

- `scale` （スケール）：コントローラがレプリカを数えて更新しながら、ポッドを追加や削除して水平スケールする。
- `annotate`（注釈）：オブジェクトにアノテーション（annotation）を追加・削除する。
- `label`（ラベル）：オブジェクトにラベルを追加・削除する。

また、 `kubectl` コマンドはオブジェクトの状態を示す更新に対応しています。この状態を設定するには、オブジェクト型によって設定するフィールドが異なります。

- `set` <フィールド>： オブジェクトの状態を設定する。


{{< note >}}
**メモ**：Kubernetes 1.5 までは、すべての動詞を示すコマンドが状態を示すコマンドとは関連付けされていませんでした。
{{< /note >}}

`kubectl` ツールは稼働オブジェクトを直接更新するための追加手法をサポートしています。
しかしながら、Kubernetes オブジェクト概念（スキーマ）に対する深い理解が必要です。

- `edit` （編集）：エディタで設定ファイルを開き、稼働オブジェクトの設定ファイルを直接編集する。
- `patch` （パッチ）：パッチ文字列を使い、稼働オブジェクトの特定フィールドを直接編集します。[API 仕様](https://git.k8s.io/community/contributors/devel/api-conventions.md#patch-operations) をご覧ください。

## オブジェクトを削除するには {#how-to-delete-objects}

クラスタからオブジェクトを削除するには `delete` （削除）コマンドが使えます。

```
- `delete <タイプ>/<名前>`
```

{{< note >}}
**メモ**：命令型コマンドと命令型オブジェクト設定の両方で `kubectl delete` が使えます。
違いは引数をコマンドで扱えるかどうかです。
`kubectl delete` を命令型コマンドとして使う場合は、削除対象のオブジェクトを引数で指定します。
次の例は nginx という名前のデプロイメント・オブジェクトを操作しています。
{{< /note >}}

```shell
kubectl delete deployment/nginx
```

## オブジェクトを表示するには {#how-to-view-an-object}

{{< comment >}}
TODO(pwittrock): Uncomment this when implemented.

You can use `kubectl view` to print specific fields of an object.

- `view`: Prints the value of a specific field of an object.

{{< /comment >}}


オブジェクトの情報を表示するには複数のコマンドがあります：

- `get`  （取得）：対象オブジェクトの基本情報を表示。 `get -h` でオプション一覧を表示。
- `describe`  （説明）：対象オブジェクトに関する全体的な詳細情報を表示。
- `logs`  （ログ）：ポッド内で実行しているコンテナの標準出力（stdout）と標準エラー（stderr）を表示。

## 以前に作成したオブジェクトを `set` コマンドで変更

オブジェクト・フィールドによっては、`create` コマンドの使用時にフラグを指定していない場合があります。
状況によっては、 `set` と `create`  を組み合わせて、オブジェクト作成時に指定がなかったフィールドの値を指定できます。
これを使えば、 `create` コマンドの出力をパイプして `set` コマンドに渡した後、再び `create` コマンドに戻せます。こちらに例があります。

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run | kubectl set selector --local -f - 'environment=qa' -o yaml | kubectl create -f -
```

1. `kubectl create service -o yaml --dry-run` コマンドはサービスの設定を作成しますが、Kubernetes API サーバに送信する代わりに YAML として標準出力します。
1. `kubectl set --local -f - -o yaml` コマンドは標準入力から設定ファイルを読み込み、変更を加えた設定ファイルを YAML として標準出力に書き出します。
1. `kubectl create -f -` コマンドで標準入力を経由して与えられた設定ファイルを使い、オブジェクトを作成します。

## 以前に作成したオブジェクトを `--edit` で編集 {#using-edit-to-modify-objects-before-creation}

既に作成したオブジェクトに対しては `kubectl create --edit` を使って億世を変更できます。
こちらが例です:

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run > /tmp/srv.yaml
kubectl create --edit -f /tmp/srv.yaml
```

1. `kubectl create service` コマンドはサービスのために設定情報を作成し、 `/tmp/srv.yaml` に保存します。
1. `kubectl create --edit` コマンドで設定情報ファイルを開き、以前に作成したオブジェクトの情報を編集します。

{{% /capture %}}

{{% capture whatsnext %}}
- [オブジェクト設定情報(命令型)を使って Kubernetes オブジェクトの管理](/jp/docs/concepts/overview/object-management-kubectl/imperative-config/)
- [オブジェクト設定情報(宣言型)を使って Kubernetes オブジェクトの管理](/jp/docs/concepts/overview/object-management-kubectl/declarative-config/)
- [Kubectl コマンド・リファレンス（参考資料）](/jp/docs/reference/generated/kubectl/kubectl/)
- [Kubernetes API リファレンス（参考資料）](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/)
{{% /capture %}}


