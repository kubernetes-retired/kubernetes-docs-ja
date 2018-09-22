---
title: 初期化コンテナ
content_template: templates/concept
weight: 40
---

{{% capture overview %}}
このページは初期化コンテナ（Init コンテナ）の概要を紹介します。
初期化コンテナは特殊なコンテナであり、アプリケーション・コンテナを実行する前に実行し、アプリケーション・イメージに存在しないユーティリティやセットアップ・スクリプトも含められます。
{{% /capture %}}

{{< toc >}}

<!--
この機能は 1.6 でベータが外れました。
初期化コンテナ（Init Container）は PodSpec にある app `containers` アレイと並んで指定できます。
ベータの注釈値（beta annotation value）が尊重されるため、 PodSpec フィールドの値を上書きします。
しかしながら、1.6 と 1.7 で廃止されました。
1.8 ではアノテーションでサポートされないため、PodSpec フィールドでカバーする必要があります。
-->
{{% capture body %}}

## 初期化コンテナの理解 {#understanding-init-containers}

[ポッド](/jp/docs/concepts/workloads/pods/pod-overview/) はポッドの中でアプリケーションを実行する複数のコンテナを持てます。
また、これに加えて１つまたは複数の初期化コンテナ（Init Container）も持てます。これは、アプリケーション・コンテナを起動する前に実行するコンテナです。

初期化コンテナは、厳密には通常のコンテナと同じですが、以下の手が異なります:

* コンテナの実行は常に完了する。
* 次のコンテナを開始する前に、正常に完了している必要がある。

ポッド用の初期化コントローラで障害が起きれば、Kubernetes は Init コンテナが成功するまでポッドの再起動を繰り返します。
しかしながら、ポッドの `restartPolicy` （再起動方針）が Never（再起動しない）であれば、再起動を行いません。

初期化コンテナとしてコンテナを指定するには、PodSpec に app `containers` アレイと並んで `initContainers` フィールドを追加し、 アプリの `containers` アレイと並列に、 [コンテナ](/jp/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#container-v1-core) の JSON アレイのオブジェクト・タイプとして記述します。
初期化コンテナが返すステータスは、コンテナ・ステータスの配列として `.status.initContainerStatus` にあります（ `.status.containerStatuses` フィールドと似ています ）。

### 通常のコンテナとの違い {#differences-from-regular-containers}

初期化コンテナはアプリケーション・コンテナの全てのフィールドと機能をサポートします。
ここにはリソース上限、ボリューム、セキュリティ設定も含まれます。しかしながら、初期化コンテナに対するリソース要求と上限の取り扱いは極めて異なります。
違いは以下の [リソース](#resources) にドキュメント化しています。
また、初期化コンテナは読込性診断をサポートしません。
なぜなら、ポッドが利用可能になるのに先立って、初期化コンテナの実行を完了している必要があるからです。

ポッドに対して複数の初期化コンテナを指定すると、各コンテナは連続した順番で一度だけ実行されます。
次のコンテナを実行する前に、各コンテナは処理が完了している必要があります。
全ての初期化コンテナの実行が完了すると、kubernetes はポッドを初期化し、通常通りにアプリケーション・コンテナを実行します。


## 初期化コンテナは何のために使えますか？ {#what-can-init-containers-be-used-for}

初期化コンテナはアプリケーション・コンテナとは分離できるため、起動処理に関連するコードのために役立ちます:

* セキュリティ上の理由により、アプリケーション・コンテナ・イメージ含められないユーティリティを、初期化コンテナに含めて実行できます。
* セットアップのためにユーティリティやカスタム・コードを含められます。たとえば、イメージに `FROM` で作成するにあたり、他のイメージに含む必要の無いツール、たとえば `sed`、 `awk`、 `python`、 `dig` をセットアップ中に扱えます
* アプリケーション・イメージ構築担当者や開発担当者が、１つのアプリケーション・イメージを一緒に構築する必要が無く、独立して作業できます。
* Linux 名前空間を使うため、アプリケーション・コンテナごとに異なるファイルシステムを持てます。そのため、アプリケーション・コンテナがアクセスできないシークレットにアクセス可能にもできます。
* あらゆるアプリケーション・コンテナを実行する前に実行を完了する必要があるのに対して、アプリケーション・コンテナは並列に実行できます。そのため、アプリケーション・コンテナが起動するまで、何らかの状態に一致するまでブロック（待機）したり遅延するのを簡単にするために、初期化コンテナを使えます。

### 例 {#examples}

ここにあるのは初期化コンテナをどう使うかのアイディアです：

* 次のようなシェル・コマンドを使い、サービスが作成されるのを待ちます：

      for i in {1..100}; do sleep 1; if dig myservice; then exit 0; fi; done; exit 1

* 以下にある API のコマンドを使い、対象のポッドをリモート・サーバ上に登録します：

      curl -X POST http://$MANAGEMENT_SERVICE_HOST:$MANAGEMENT_SERVICE_PORT/register -d 'instance=$(<POD_NAME>)&ip=$(<POD_IP>)'

* アプリケーション・コンテナが起動する前に、 `sleep 60` のようなコマンドを使って待機します。
* ボリューム内に git リポジトリをクローンします。
* メインのアプリケーション・コンテナ用の設定ファイルを動的に生成するような、テンプレート・ツールの設定ファイルに値を入れ、実行します。たとえば、設定ファイルに POD_IP の値を入れた設定ファイルを準備しておき、メインのアプリケーション用が使う Jinja 用のファイルを生成します。

より詳しいサンプルの使い方は、[StatefulSets ドキュメント](/jp/docs/concepts/workloads/controllers/statefulset/)
と [プロダクション向けのポッド・ガイド](/jp/docs/tasks/configure-pod-container/configure-pod-initialization/) にあります。

### 初期化コンテナ使用例 {init-containers-in-use}

以下の YAML ファイルは Kubernetes 1.5 用にまとめたシンプルなポッドであり、２つの初期化コンテナがあります。
まずは `myservice` を待機し、次に `mydb` の処理を待ちます。どちらのコンテナも完了したら、ポッドを開始します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
  annotations:
    pod.beta.kubernetes.io/init-containers: '[
        {
            "name": "init-myservice",
            "image": "busybox",
            "command": ["sh", "-c", "until nslookup myservice; do echo waiting for myservice; sleep 2; done;"]
        },
        {
            "name": "init-mydb",
            "image": "busybox",
            "command": ["sh", "-c", "until nslookup mydb; do echo waiting for mydb; sleep 2; done;"]
        }
    ]'
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
```

こちらは Kubernetes 1.6 に対応した新しい構文です。
以前の古い構文も 1.6 と 1.7 で利用できます。新しい構文を利用できるのは 1.8 以上です。
初期化コンテナの宣言は `spec` に移動しました:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

1.5 構文は 1.6 でも動作しますが、1.6 構文の仕様を推奨します。
また、Kubernetes 1.6 では、初期化コンテナは API のフィールドでも作成できます。
ベータ・アノテーションは 1.6 と 1.7 までは尊重されていますが、1.8 以上ではサポートされていません。

以下の YAML ファイルは、 `mydb` と `myservice` サービスのアウトラインです:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
kind: Service
apiVersion: v1
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

このポッドは以下のコマンドで開始およびデバッグできます:

```shell
$ kubectl create -f myapp.yaml
pod "myapp-pod" created
$ kubectl get -f myapp.yaml
NAME        READY     STATUS     RESTARTS   AGE
myapp-pod   0/1       Init:0/2   0          6m
$ kubectl describe -f myapp.yaml
Name:          myapp-pod
Namespace:     default
[...]
Labels:        app=myapp
Status:        Pending
[...]
Init Containers:
  init-myservice:
[...]
    State:         Running
[...]
  init-mydb:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Containers:
  myapp-container:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Events:
  FirstSeen    LastSeen    Count    From                      SubObjectPath                           Type          Reason        Message
  ---------    --------    -----    ----                      -------------                           --------      ------        -------
  16s          16s         1        {default-scheduler }                                              Normal        Scheduled     Successfully assigned myapp-pod to 172.17.4.201
  16s          16s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulling       pulling image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulled        Successfully pulled image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Created       Created container with docker id 5ced34a04634; Security:[seccomp=unconfined]
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Started       Started container with docker id 5ced34a04634
$ kubectl logs myapp-pod -c init-myservice # Inspect the first init container
$ kubectl logs myapp-pod -c init-mydb      # Inspect the second init container
```

`mydb` と `myservice` サービスを開始すると、初期化コンテナの完了後に `myapp-pod` が作成されます：

```shell
$ kubectl create -f services.yaml
service "myservice" created
service "mydb" created
$ kubectl get -f myapp.yaml
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          9m
```

この例は非常にシンプルですが、皆さん自身が初期化コンテナを作成する時に参考となるでしょう。

## 挙動の詳細 {#detailed-behaivor}

ポッドの起動中、初期化コンテナは順番通りに起動した後、ネットワークとボリュームが初期化されます。
各コンテナを開始するには、前のコンテナは実行を完了している必要があります。
コンテナのランタイムが原因か障害による終了で起動できなければ、ポッドの `restartPolicy` （再起動方針）に従ってポッドの再起動を試みます。
しかしながら、ポッドの `restartPolicy` が `Always`（常時）に設定されていると、初期化コンテナは `RestartPolicy` （再起動方針）を OnFailure（障害時のみ）として扱います。

全ての初期化コンテナが成功するまで、ポッドは `Ready` （待機）になりません。
初期化コンテナのポートはサービスと関連付けられていません。
ポッドの初期化中は `Pending`（待機中）の状態ですが、 `Initilizing` （初期化）の状態も true となるべきでしょう。

ポッドが [再起動したら](#pod-restart-reasons)、全ての初期化コンテナを再実行する必要があります。

初期化コンテナの spec 変更は、コンテナ・イメージ・フィールドのみに制限されています。
初期化コンテナのイメージ・フィールドを反映するには、ポッドの再起動をしますのでご注意ください。

全ての初期化コンテナは再起動され、リトライされ、再実行されうるので、初期化コンテナのコードは冪等（idempotent）であるべきです。
特に、 `EnptyDirs` 上にファイルを書き出す場合は、出力するファイルが既に存在しうるものとして準備すべきでしょう。

初期化コンテナが持つ全てのフィールドは、アプリケーション・コンテナ上にあります。
しかしながら、Kubernetes は `readinessProbe` が使われるのを禁止します。
これは初期化コンテナが読込性を定義しても完了するかどうか確認できないからです。
この整合性（validation）の確認が必ず行われます。

ポッド上の `activeDeadlineSeconds`（アクティブ期限秒） とコンテナ上の `livenessProve` （生存性診断）によって、初期化コンテナの障害継続を防止します。
アクティブ期限（デッドライン）には初期化コンテナも含みます。

ポッド内の各アプリケーション・コンテナおよび初期化コンテナの名前は、ユニークである必要があります。
つまり、あらゆるコンテナが他と名前の重複があると、整合性のエラーとなります。

### リソース {#resources}

初期化コンテナの並べ替えの指定と実行には、以下のルールに従ってリソース使用が適用されます:

* 何らかの最も高いリソース要求か、全ての初期化コンテナで定義されている上限が、 *効率的な初期化要求/上限（effective init request/limit）* です
* ポッドのリソースに対する効率的な初期化要求/上限は、以下どちらかの高い方です：
  * 全てのアプリケーション・コンテナのリソース要求/上限の合計
  * リソースに対する効率的な初期化要求/上限
* 効率的な要求/上限に基づきスケジューリングが実施されます。つまり、初期化コンテナは初期化のためにリソースを予約できます。これはポッドの稼働中には使われないものも含みます。
* ポッドの効率的な QoS ティア（tier）は、初期化コンテナとアプリケーション・コンテナに対する QoS ティアと同じです。

クォータ（容量制限）と上限は、効率的なポッド要求と上限に基づいて適用されます。

ポッド・レベルの cgroup はスケジューラと同様に、効率的なポッド要求と上限に基づいて適用されます。

### ポッド再起動条件 {#pod-restart-reasons}

以下の条件に従って、初期化コンテナの再実行によるポッドは再起動（restart）が発生します；

* ユーザの PodSpec 更新は、初期化コンテナ・イメージの変更を引き起こします。アプリケーション・コンテナ・イメージの変更は、アプリケーション・コンテナのみ再起動します。
* ポッドの基盤（infrastructure）コンテナの再起動です。これは一般的では無く、ノードに対する何らかの root アクセスによる処理が行われた場合です。
* ポッド内の全てのコンテナが `restartPolicy` （再起動方針）が Always （常に）に設定された場合、再起動は強制され、ガベージ・コレクション（不要データの削除処理）によって初期化コンテナ管理の記録は破棄されます。

## サポートと互換性 {#support-and-compatibility}

apiserver バージョン 1.6.0 以上のクラスタは `.spec.initContainers` フィールドで初期化コンテナの使用をサポートします。
それよりも以前のバージョンでは、初期化コンテナの使用にはアルファまたはベータの注記がありました。
また、 `.spec.initContainers` フィールドはアルファおよびベータ注記を反映していますので、Kubernetes 1.3.0 以上では初期化コンテナを実行できます。
また、バージョン 1.6 以上の apiserver は安全にバージョン 1.5.x にロールバックでき、そのときに作成済みのポッドに対する初期化コンテナの機能性は失われません。

Kubernetes バージョン 1.8.0 以上の apiserver では、アルファおよびベータ注記のサポートが削除されました。 `.spec.initContainers` フィールドにおけるアノテーション重複も削除されています。

{{% /capture %}}


{{% capture whatsnext %}}
* [初期化コンテナがあるポッドを作成](/jp/docs/tasks/configure-pod-container/configure-pod-initialization/#creating-a-pod-that-has-an-init-container)

{{% /capture %}}



