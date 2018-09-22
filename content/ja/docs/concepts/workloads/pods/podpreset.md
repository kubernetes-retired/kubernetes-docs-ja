---
title: ポッド・プリセット
content_template: templates/concept
weight: 50
---

{{% capture overview %}}
このページは PodPreset（ポッド・プリセット）の概要を紹介します。
PodPreset とは、特定の情報をポッド作成時に投入するオブジェクトです。
情報に含められるのはシークレット（機密情報）、ボリューム、ボリューム・マウント、環境変数です。
{{% /capture %}}

{{< toc >}}

{{% capture body %}}
## ポッド・プリセットの理解 {#understanding-pod-presets}

`PodPreset` （ポッド・プリセット）とは API リソースであり、ポッドの作成時に追加のランタイム環境変数を投入します。
[ラベル・セレクタ](/jp/docs/concepts/overview/working-with-objects/labels/#label-selectors) を使い、特定のポッドに対して適用するポッド・プリセットを指定します。

ポッド・プリセットを使うと、ポッド・テンプレートの作者はポッドごとに全ての情報を明示する必要がなくなります。
ポッド・テンプレートの作者がこの方法を使うと、詳細な記述が不要な情報を対象となるサービスから削除できます。

背景に関する詳しい情報は、 [PodPreset の設計提案](https://git.k8s.io/community/contributors/design-proposals/service-catalog/pod-preset.md) をご覧ください。

## 動作内容 {#how-it-works}

Kubernetes が提供する入場制御（アドミッション・コントローラ：admission controller）（`PodPreset`）は、有効にすると、ポッド作成要求があれば、ポッド・プリセットを作成します。
ポッド作成要求が発生すると、システムは以下の処理をします：

1. 使うために、全ての利用可能な `PodPresets` を集めます。
1. ラベル・セレクタは、作成しようとしているポット上に `PodPreset` に一致するラベルがあるかどうか調べます。
1. 作成しようとしているポッド内に、 `PodPreset` にある様々なリソース定義の統合（マージ）を試みます。
1. エラーがあれば、ポッドに対する統合エラーがイベントに記録され、ポッドには `PodPreset` で指定したリソース投入を行いません。
1. 変更した Pod spec を目立つようにアノテーション（注釈）をつけます。これは `PodPreset` いよって変更されたものです。アノテーションの形式は    `podpreset.admission.kubernetes.io/podpreset-<ポッド・プリセット名>: "<リソース・バージョン>"` です。

各ポッドがゼロもしくは PodPreset を複数もつ場合があります。
つまり、各 `PodPreset` にはゼロもしくは複数のポッドが適用される可能性があります。
もし `PodPreset` が１つもしくは複数のポッドに割り当てられる場合、Kubernetes はポッド内にある全てのコンテナのコンテナ spec を変更します。
この変更には `Volume` も含みますので、Kubernetes はポッドの spec を変更します。

{{< note >}}
**メモ：** ポッド・プリセットは然るべき時にポッド spec の `.spec.containers` フィールドを変更できる能力があります。
ポッド・プリセットに *No* リソースを定義すると（訳者注：何も定義しなければ）、ポッド・プリセットは `initContainers` フィールドに適用されます。
{{< /note >}}

### 特定のポッドに対するポッド・プリセットを無効化 {#disable-pod-preset-for-a-specific-pod}

ポッドに対しては、通知の無いあらゆるポッドのプリセット変更を行いたくない場合があるでしょう。
そのような場合、ポッド Spec にアノテーション（注釈）を `podpreset.admission.kubernetes.io/exclude: "true"` のような形式で追加できます。

## ポッド・プリセットの有効化 {#enable-pod-preset}

ポッド・プリセットをクラスタ内で使うためには、以下の手順を確実に行う必要があります

1. API タイプ `settings.k8s.io/v1alpha1/podpreset` を有効化します。たとえば、API サーバに対するオプションとして  `--runtime-config` に `settings.k8s.io/v1alpha1=true`  を指定します。
1. アドミッション・コントローラ `PodPreset` を有効化します。有効化するための１つの方法は、API サーバに対するオプション `--enable-admission-plugins` の値に `PodPreset` を含めます。
1. 使用する名前空間内で `PodPreset` オブジェクトの作成時に、ポッド・プリセットを定義します。

{{% /capture %}}

{{% capture whatsnext %}}
* [PodPreset を使ってポッドにデータを投入](/jp/docs/tasks/inject-data-application/podpreset/)

{{% /capture %}}


