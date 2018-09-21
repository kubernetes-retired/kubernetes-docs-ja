---
title: コンテナ環境変数
content_template: templates/concept
weight: 20
---

{{% capture overview %}}
このページはコンテナで利用できるコンテナ環境変数について説明します。
{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## コンテナ環境変数 {#container-environment}

Kubernetes コンテナ環境変数は、コンテナに対して複数の重要なリソースを提供します：

- [イメージ](/jp/docs/concepts/containers/images/) と１つまたは複数の [ボリューム](/jp/docs/concepts/storage/volumes/) を組み合わせたファイルシステム。
- コンテナ自身に関する情報。
- クラスタ内の他のオブジェクトに関する情報。

### コンテナ情報 {#container-information}

コンテナの *ホスト名* とは、コンテナを実行しているポッドの名前です。
これは `hostname` コマンドを通してか、 libc の関数コール [`gethostname`](http://man7.org/linux/man-pages/man2/gethostname.2.html) で取得できます。

[downward API](/jp/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/) を通して、ポッド名と名前空間を環境変数として利用できます。

また、ユーザはポッドの定義時にもコンテナに対する環境変数を定義できます。
また、Docker イメージで指定されている環境変数と共に利用できます。

### クラスタ情報 {#cluster-infomation}

コンテナ作成時に実行中のサービス一覧は、コンテナの環境変数として利用できます。
これら環境変数の利用形態は、 Docker の link 構文に相当します。

サービス名 *foo*  がコンテナ名 *bar* に割り当てられている場合、以下の環境変数が定義されます：

```shell
FOO_SERVICE_HOST=<サービスを実行しているホスト>
FOO_SERVICE_PORT=<サービスを実行しているポート>
```

[DNS アドオン](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons/dns/) を有効化すると、サービスは専用の IP アドレスをもち、DNS を通してコンテナにアクセスできるようになります。

{{% /capture %}}

{{% capture whatsnext %}}

* [コンテナ・ライフサイクル・フック](/jp/docs/concepts/containers/container-lifecycle-hooks/) について学ぶ。
* [コンテナ・ライフサイクル・イベントを取り扱う](/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/) ハンズオン練習。


{{% /capture %}}


