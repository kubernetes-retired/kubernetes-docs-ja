---
title: イメージ
content_template: templates/concept
weight: 10
---

{{% capture overview %}}

Kubernetes ポッドから参照される前に、Docker イメージの作成とレジストリへの送信ができます。

コンテナの `image` 属性（プロパティ）には `docker` コマンドと同じ構文をサポートしており、プライベート・レジストリやタグに関するコマンドも含みます。


{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## イメージの更新 {#updating-images}

デフォルトの取得（pull）ポリシーが `IfNotPresent` （無指定）の場合、既にイメージが存在していれば、Kubelet はイメージの取得を省略します。常に取得を強制したい場合は、いかのいずれかによって行えます：

- コンテナの  `imagePullPolicy` を `Always` （常時）に指定する
- 使用するイメージに対して `:latest` （最新）タグを使う
- [AlwaysPullImages](/jp/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages) （常にイメージを取得）を許可するコントローラを有効化する。

もしもイメージのタグを指定しなければ、イメージ取得ポリシーが `Always` であれば、 `:latest` が指定されたものとみなされます。

`:latest` タグの使用は避けるべきなのでご注意ください。詳細は [設定情報のベスト・プラクティス](/jp/docs/concepts/configuration/overview/#container-images) をご覧ください。

## プライベート・レジストリを使う {#using-a-private-registry}

プライベート・レジストリからイメージを読み込むには、鍵（キー）が必要になる場合があります。
証明書を取得するには、複数の方法があります：

  - Google Container Registry を使う場合
    - クラスタごとに
    - Google Compute Engine または Google Kubernetes Engine で自動的に設定
    - すべてのポッドがプロジェクトのプライベート・レジストリを読み込める
  - AWS EC2 Container Registry (ECR) を使う場合
    - ECR リポジトリに対するアクセス制御に IAM ロールとポリシーを使う
    - ECR ログイン証明書を自動的にリフレッシュ（再設定）する
  - Azure Container Registry (ACR) を使う場合
  - プライベート・レジストリの認証のためにノードを調整する場合
    - すべてのポッドが設定したプライベート・レジストリから読み込み可能
    - クラスタ管理者によるノードの設定変更が必要
  - 事前にイメージを取得する場合
    - すべてのポッドはノード上にキャッシュされたあらゆるイメージを利用できる
    - すべてのノードをセットアップするために root 権限が必要
  - ポッド上で ImagePullSecrets を指定
    - 自身のキーを持っているポッドのみがプライベート・レジストリにアクセスできる。各オプションの詳細については後述。

### Google Container Registry を使う場合 {#using-google-container-registry}

Google Compute Engine (GCE) を使う場合、Kubernetes は  [Google Container Registry (GCR)](https://cloud.google.com/tools/container-registry/) をネイティブにサポートします。
もしクラスタを GCE や Google Kubernetes Engine 上で実行している場合は、単純に完全なイメージ名を使います（例： gcr.io/my_project/image:tag ）

クラスタ内のすべてのポッドが、このレジストリにあるイメージを読み込めます。

Kubelet は GCR との認証にインスタンスの Google サービス・アカウントを使います。インスタンスに対するサービス・アカウントが `https://www.googleapis.com/auth/devstorage.read_only` であれば、プロジェクトの CGR から取得（pull）はできますが、送信（push）できません。

### AWS EC2 Container Registry を使う場合 {#using-aws-ec2-container-registry} 

ノードが AWS EC2 インスタンスの場合は、Kubernetes は [AWS EC2 Container Registry](https://aws.amazon.com/ecr/) をネイティブにサポートします。

ポッドの定義を簡単にするにはフル・イメージ名を使います（例： `ACCOUNT.dkr.ecr.REGION.amazonaws.com/imagename:tag`）。

ECS レジストリ内にあるイメージのすべてを、クラスタ内のユーザであれば誰でもポッドを作成し、ポッドを実行できます。

kubelet は ECR 証明書情報を定期的に取得し、再読み込み（リフレッシュ）します。そのためには、以下の権限が必要です：

- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:GetDownloadUrlForLayer`
- `ecr:GetRepositoryPolicy`
- `ecr:DescribeRepositories`
- `ecr:ListImages`
- `ecr:BatchGetImage`

必要条件:

- kubelet バージョン `v1.2.0` 以上の使用が必要です（例： `/usr/bin/kubelet --version=true` を実行します）。
- ノードがリージョン A にあり、レジストリが異なるリージョン B にある場合は、バージョン `v1.3.0` 以上が必要です。

トラブルシューティング:

- 前述の必要条件すべてを確認します。
- $REGION（例 `us-west-2` など）認証情報を自分の PC に準備します。認証情報を使ってホストに SSH 接続し、 Docker を手動で起動します。正しく動作しますか？
- kubelet が `--cloud-provider=aws` で動作しているか確認します。
- kubelet ログ（例： `journalctl -u kubelet` ）から、次のような行を確認します：
  - `plugins.go:56] Registering credential provider: aws-ecr-key`
  - `provider.go:91] Refreshing cache for provider: *aws_credentials.ecrProvider`

(TODO)

{{% /capture %}}


