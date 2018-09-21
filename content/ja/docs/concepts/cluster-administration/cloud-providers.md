---
title: クラウド・プロバイダ
content_template: templates/concept
weight: 30
---

{{% capture overview %}}
このページでは、特定のクラウド・プロバイダ上で Kubernetes の動作を管理する方法を紹介します。
{{% /capture %}}

{{% capture body %}}
## AWS
このセクションは、Amazon Web Services 上で Kubernetes を実行するにあたって、使う可能性がある全ての設定について説明します。

### ノード名 {#node-name}

AWS クラウド・プロバイダは、AWS インスタンスのプライベート DNS 名を、Kubernetes ノード・オブジェクトの名前として使います。

### ロードバランサ {#lad-balancers}

[外部ロードバランサ](/jp/docs/tasks/access-application-cluster/create-external-load-balancer/) を設定し、以下のようにアノテーションを設定変更することで、AWS の特定機能を使えるようになります。


```yaml
apiVersion: v1
kind: Service
metadata:
  name: example
  namespace: kube-system
  labels:
    run: example
  annotations:
     service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:xx-xxxx-x:xxxxxxxxx:xxxxxxx/xxxxx-xxxx-xxxx-xxxx-xxxxxxxxx #replace this value
     service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 5556
    protocol: TCP
  selector:
    app: example
```
AWS のロードバランサ・サービスに対して異なる設定を適用するには  _annotations_ を使います。
以下の説明は AWS ELB をサポートするアノテーションです。

* `service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval`: アクセスログの送信間隔を指定するのに使います。
* `service.beta.kubernetes.io/aws-load-balancer-access-log-enabled`: サービスに対するアクセスログを有効または無効にするのに使います。
* `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name`: アクセスログの s3 バケット名の指定に使います。
* `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix`: アクセスログの s3 バケット・プリフィックスの指定に使います。
* `service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags`: サービス上での ELB 追加タグとして記録されるキーバリューのペアを、カンマ区切りで指定します。例：`"Key1=Val1,Key2=Val2,KeyNoVal1=,KeyNoVal2"`。
* `service.beta.kubernetes.io/aws-load-balancer-backend-protocol`: サービス上で使うための、バックエンド（ポッド）の背後にいるリスナーが使うプロトコルを指定します。もし `http` （デフォルト）や `https`  であれば、HTTPS リスナーは作成されると、通信を切断し、ヘッダをパースします。もしも `ssl` や `tcp` を設定すると、 "raw"（そのままの） SSL リスナを使います。もしも `http` と `aws-load-balancer-ssl-cert` を設定して使っていなくても、その後の HTTP リスナーによって使われます
* `service.beta.kubernetes.io/aws-load-balancer-ssl-cert`: サービス上で安全なリスナー要求のために使われます。値は有効な証明書お ARN です。詳しくは [ELB Listener Config](http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/elb-listener-config.html) CertARN にある IAM または CM 証明書 ARN、例： `arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012` をご覧ください。
* `service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled`: サービス上で接続の排出（ドレイニング：draining）を有効にするか無効にするかを設定します。
* `service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout`: サービス上で接続の排出タイムアウトの指定に使います。
* `service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout`: サービス上でアイドル（何もしない）接続のタイムアウトの指定に使います。
* `service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled`: サービス上でクロスゾーンの負荷分散を有効化もしくは無効化を設定します。
* `service.beta.kubernetes.io/aws-load-balancer-extra-security-groups`: サービス上で作成した ELB に追加する追加セキュリティ・グループの指定で使います。
* `service.beta.kubernetes.io/aws-load-balancer-internal`: サービス上で内部の ELB として指定したい場合に使います。
* `service.beta.kubernetes.io/aws-load-balancer-proxy-protocol`: サービス上の ELB 上で有効にしたい proxy プロトコルの指定に使います。現時点では `*` の値しか受け付けません。つまり、全ての ELB バックエンド上のプロトコルが有効デス。将来的にはバックエンドを特定のプロキシ・プロトコルのみに絞られるように調整するつもりです。
* `service.beta.kubernetes.io/aws-load-balancer-ssl-ports`: SSL/HTTPS リスナが巣買うためのポートを、カンマ区切りで指定します。デフォルトは `*` （すべて）です。

AWS の注記については、[aws.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/cloudprovider/providers/aws/aws.go) のコメント欄をご覧ください。

## Azure

### ノード名 {#node-name}

Azure クラウド・プロバイダがノードのホスト名に使うのは（kubeｪtによって決められるか、 `--hostname-override` で上書きできます）、Kubernetes ﾉｰﾄﾞ/オブジェクトとしての名前です。Kubernetes ノード名は Azure VM 名と一致する必要があるのでご注意ください。

## CloudStack

### ノード名 {#node-name}

CloudStack クラウド・プロバイダがノードのホスト名に使うのは（kubeｪtによって決められるか、 `--hostname-override` で上書きできます）、Kubernetes ﾉｰﾄﾞ/オブジェクトとしての名前です。Kubernetes ノード名は CloudStack VM 名と一致する必要があるのでご注意ください。


(TODO)
