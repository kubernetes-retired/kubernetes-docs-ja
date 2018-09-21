---
title: クラスタ管理概要
content_template: templates/concept
weight: 10
---

{{% capture overview %}}
クラスタ管理概要の対象は、Kubernetes クラスタを構築または管理をする方です。
核心となる Kubernetes [概念](/jp/docs/concepts/) をある程度熟知しているのが前提です。
{{% /capture %}}

{{% capture body %}}
## クラスタの設計 {#planning-a-cluster}

Kubernetes クラスタをどのように設計（計画）、セットアップ（構築）、設定するかの例は、[適切な解決策（ソリューション）の選択](/jp/docs/setup/pick-right-solution/) にあるガイドをご覧ください。

ガイドを選ぶ前に、いくつかの検討事項があります:

 - 自分のコンピュータ上で Kubernetes を試してみるだけですか？ 高可用性や複数ノードのクラスタを構築したいですか？ 必要に応じて最適なディストリビューションを選択してください。
 - **高可用性の設計をしている場合は** 、 [複数ゾーンでのクラスタ](/jp/docs/concepts/cluster-administration/federation/) 設定について学んでください。
 -  [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) や **自分自身でクラスタをホスティング（所有）**  して、**ホステッド（所有型） Kubernetes クラスタ** を使いますか？
 - クラスタは **オン・プレミス（on-premises）** ですか、 **クラウド（IaaS）内** ですか？ Kubernetes はハイブリッド／クラスタを直接サポートしません。そのかわりに、自分自身で複数のクラスタをセットアップ（構築）できます。
 - **Kubernetes をオンプレミスで設定する場合** 、どの [ネットワーク形成モデル](/jp/docs/concepts/cluster-administration/networking/) が一番適切なのか検討します。
 - **"ベア・メタル" ハードウェア** あるいは **仮想マシン（VM）** のどちらで Kubernetes を実行しようとしていますか？
 - **単位クラスタを動かしたいだけ**  ですか？ それとも **Kubernetes プロジェクトのコードをアクティブに展開（デプロイ）** するつもりですか？ 後者でしたら、活発に開発されているディストリビューションを選択します。ディストリビューションによってはバイナリ・リリースしか提供されていませんが、様々な選択肢が提供されています。
 - クラスタを実行に必要な [構成要素（コンポーネント）](/jp/docs/concepts/overview/components/) に自分自身が習熟している。

メモ：ディストリビューションは全てが活発に維持（メンテナンス）されているわけではありません。
最近の Kubernetes バージョンをテスト済みのディストリビューションを選んでください。

既に Salt を導入しているガイドを使っている場合は、 [Salt で Kubernetes を設定](/jp/docs/admin/salt/) をご覧ください。

## クラスタ管理 {#managing-a-cluster}

* [クラスタの管理](/jp/docs/tasks/administer-cluster/cluster-management/) には、クラスタのライフサイクルに関連するトピックの記載が複数あります。
たとえば、新しいクラスタを作成、クラスタのマスタとワーカ・ノードを更新、ノードのメンテナンス対応（例：カーネル更新）、実行中のクラスタで Kubernetes API のバージョン更新です。

* [ノードの管理](/docs/concepts/nodes/node/) 方法を学びます。

* 共有されたクラスタで [リソース制限（quota）](/jp/docs/concepts/policy/resource-quotas/) の設定と管理方法を学びます。

## クラスタを安全に {#securing-a-cluster}

* [証明書（Certificates）](/jp/docs/concepts/cluster-administration/certificates/) は、別々のツール・チェーンを使った証明書の作成手順を説明します。
* [Kubernetes コンテナ環境変数](/jp/docs/concepts/containers/container-environment-variables/) は、Kubernetes ノード上で Kubelet が管理するコンテナに対する環境変数について説明します。
* [Kubernetes API に対するアクセス制御](/jp/docs/reference/access-authn-authz/controlling-access//) は、ユーザとサービス・アカウントに対する権限の設定方法を説明します。
* [認証（Authenticating）](/jp/docs/reference/access-authn-authz/authentication/) は様々な認証オプションを含む Kubernetes の認証を説明します。
* [権限付与（Authorization）](/jp/docs/reference/access-authn-authz/authorization/) は認証とは異なり、制御は HTTP コールを処理します。
* [承認コントローラ（Admission Controller）を使う](/jp/docs/reference/access-authn-authz/admission-controllers/) は、認証および権限付与後、Kubernetes API サーバでプラグインのリクエストを受けるのを説明します。
* [Kubernetes クラスタで sysctl を使う](/jp/docs/concepts/cluster-administration/sysctl-cluster/) は、 `sysctl` コマンドライン・ツールでカーネル・パラメータを設定する方法を節シマします。
* [監査](/jp/docs/tasks/debug-application-cluster/audit/) は、Kubernetes の監査ログをどのようにして扱うかを説明します。

## kubelet を安全に {#securing-the-kubelet}

  * [マスタ・ノード間の通信](/jp/docs/concepts/architecture/master-node-communication/)
  * [TLS 初期構築（ブートストラッピング）](/jp/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)
  * [Kubelet 認証/権限付与](/jp/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/)

## オプションのクラスタ・サービス {#optional-cluster-services}

* [DNS 統合（インテグレーション）](/jp/docs/concepts/services-networking/dns-pod-service/) では、Kubernetes サービスに DNS 名で直接名前する方法を説明します。
* [クラスタ動作のログ出力と監視](/jp/docs/concepts/cluster-administration/logging/) では、Kubernetes のログ出力がどのように機能するかと、どのようにこれを実装しているかを説明します。

{{% /capture %}}


