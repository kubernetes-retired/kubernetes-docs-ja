---
title: 証明書（Certificates）
content_template: templates/concept
weight: 20
---


{{% capture overview %}}

クライアント証明書認証を使う場合は、証明書を `easyrsa` 、 `openssl` 、 `cfssl`  を通して手動で作成できます。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

### easyrsa

**easyrsa**  はクラスタ用の証明書を自動的に生成できます。

1. easyrsa3 のパッチ済みバージョンをダウンロード、展開、初期化します。

        curl -LO https://storage.googleapis.com/kubernetes-release/easy-rsa/easy-rsa.tar.gz
        tar xzf easy-rsa.tar.gz
        cd easy-rsa-master/easyrsa3
        ./easyrsa init-pki
2. CA（証明機関）を作成します。（ `--batch` を設定すると自動モード、 `--req-cn` はデフォルトの CN を使います ）

        ./easyrsa --batch "--req-cn=${MASTER_IP}@`date +%s`" build-ca nopass
3. サーバ証明書と鍵を作成します。引数 `--subject-alt-name`  （別名の受け入れ）で、 API サーバになる可能性のある IP アドレスと DNS 名を設定します。 `MASTER_CLUSTER_IP` （マスタのクラスタ IP アドレス）には、通常はサービス CIDR にある最初の IP アドレスです。API サーバとコントローラ・マネージャ（構成）部品の両方のクラスタ範囲を、サービス CIDR の `--service-cluster-ip-range` 引数によって 指定します。引数 `--days` （日数）は、証明書の有効期限が切れてからの日数を指定します。また、以下の例では `cluster.local` をデフォルトの DNS ドメイン名とみなしています。
   
        ./easyrsa --subject-alt-name="IP:${MASTER_IP},"\
        "IP:${MASTER_CLUSTER_IP},"\
        "DNS:kubernetes,"\
        "DNS:kubernetes.default,"\
        "DNS:kubernetes.default.svc,"\
        "DNS:kubernetes.default.svc.cluster,"\
        "DNS:kubernetes.default.svc.cluster.local" \
        --days=10000 \
        build-server-full server nopass
4. `pki/ca.crt`、`pki/issued/server.crt`、 `pki/private/server.key` を自分のディレクトリにコピーします。

5. API サーバの起動パラメータ中に、以下のパラメータを埋めて追加します。

        --client-ca-file=/yourdirectory/ca.crt
        --tls-cert-file=/yourdirectory/server.crt
        --tls-private-key-file=/yourdirectory/server.key

### openssl

**openssl** は自分のクラスタ用の証明書を手動で作成できます。

1. 2048 ビットの ca.key （認証機関の鍵）を作成します。

        openssl genrsa -out ca.key 2048
2. ca.key に従って ca.cer（認証機関の証明書）を作成します（-days を使い、証明書の有効期間を設定します。）。

        openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
3. 2048 ビットの server.key （サーバ鍵）を作成します。

        openssl genrsa -out server.key 2048
4. 証明書署名リクエスト（CSR）のための設定ファイルを作成します。カギ括弧（例： `<MASTER_IP>`）記号の場所は、ファイル（例： `csr.conf` ）に保存する前に、確実に値を入力します。前のサブセクションで説明したように、 `MASTER_CLUSTER_IP` （マスタのクラスタ IP）とは API サーバ用のクラスタ IP サービスなのでご注意ください。また、以下の例ではデフォルトの DNS ドメイン名として、 `cluster.local` を使っている前提です。

        [ req ]
        default_bits = 2048
        prompt = no
        default_md = sha256
        req_extensions = req_ext
        distinguished_name = dn
        
        [ dn ]
        C = <country>
        ST = <state>
        L = <city>
        O = <organization>
        OU = <organization unit>
        CN = <MASTER_IP>
        
        [ req_ext ]
        subjectAltName = @alt_names
        
        [ alt_names ]
        DNS.1 = kubernetes
        DNS.2 = kubernetes.default
        DNS.3 = kubernetes.default.svc
        DNS.4 = kubernetes.default.svc.cluster
        DNS.5 = kubernetes.default.svc.cluster.local
        IP.1 = <MASTER_IP>
        IP.2 = <MASTER_CLUSTER_IP>
        
        [ v3_ext ]
        authorityKeyIdentifier=keyid,issuer:always
        basicConstraints=CA:FALSE
        keyUsage=keyEncipherment,dataEncipherment
        extendedKeyUsage=serverAuth,clientAuth
        subjectAltName=@alt_names
5. 設定ファイルをもとに、証明書署名要求（CSR）を作成します。

        openssl req -new -key server.key -out server.csr -config csr.conf
6. ca.key、ca.crt、server.csr を使ってサーバ証明書を生成します。

        openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
        -CAcreateserial -out server.crt -days 10000 \
        -extensions v3_ext -extfile csr.conf
7. 証明書を表示します。

        openssl x509  -noout -text -in ./server.crt

最後に、API サーバ起動パラメータに対し、同じパラメータを追加します。

### cfssl

**cfssl**  は証明書を生成するための他のツールです。

1. 以下にあるようにコマンドライン・ツールをダウンロードし、展開し、準備します。以下の例で使用しているハードウェアのアーキテクチャと利用する cfssl バージョンは、皆さんの環境と違う場合がありますのでご注意ください。

        curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o cfssl
        chmod +x cfssl
        curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o cfssljson
        chmod +x cfssljson
        curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o cfssl-certinfo
        chmod +x cfssl-certinfo
2. 成果物（アーティファクト）を置き、cfssl を初期化するディレクトリを作成します。

        mkdir cert
        cd cert
        ../cfssl print-defaults config > config.json
        ../cfssl print-defaults csr > csr.json
3. CA（認証機関）ファイルを生成用の JSON 設定ファイルを作成します。例えば `ca-config.json` です。

        {
          "signing": {
            "default": {
              "expiry": "8760h"
            },
            "profiles": {
              "kubernetes": {
                "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
                ],
                "expiry": "8760h"
              }
            }
          }
        }
4. CA（認証機関）証明書署名要求（CSR）用の JSON 設定ファイル、例えば `ca-csr` を作成します。カギ括弧で印がある値は、使用したい実際の値に置き換える必要がありますのでご注意ください。

        {
          "CN": "kubernetes",
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names":[{
            "C": "<country>",
            "ST": "<state>",
            "L": "<city>",
            "O": "<organization>",
            "OU": "<organization unit>"
          }]
        }
5. CA 鍵（ `ca-key.pem` ）と証明書（ `ca.pem` ）を作成します。

        ../cfssl gencert -initca ca-csr.json | ../cfssljson -bare ca
6. 以下に出てくる API サーバ用の鍵と証明書を生成するための、 JSON 設定ファイルを作成します。カギ括弧（例： `<MASTER_IP>`）記号の場所は、ファイル（例： `csr.conf` ）に保存する前に、確実に値を入力します。前のサブセクションで説明したように、 `MASTER_CLUSTER_IP` （マスタのクラスタ IP）とは API サーバ用のクラスタ IP サービスなのでご注意ください。また、以下のサンプルではデフォルトの DNS ドメイン名として `cluster.local` を使っているものと想定しています。

        {
          "CN": "kubernetes",
          "hosts": [
            "127.0.0.1",
            "<MASTER_IP>",
            "<MASTER_CLUSTER_IP>",
            "kubernetes",
            "kubernetes.default",
            "kubernetes.default.svc",
            "kubernetes.default.svc.cluster",
            "kubernetes.default.svc.cluster.local"
          ],
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names": [{
            "C": "<country>",
            "ST": "<state>",
            "L": "<city>",
            "O": "<organization>",
            "OU": "<organization unit>"
          }]
        } 
7. API サーバ用の鍵と証明書を生成します。デフォルトではファイル `server-key.pem` と `server.pem` がそれぞれ相当します。

        ../cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
        --config=ca-config.json -profile=kubernetes \
        server-csr.json | ../cfssljson -bare server

## 自己署名 CA （証明機関）証明書の配布 {#distributing-self-signed-ca-certificate}

クライアント・ノードによっては、自己署名 CA 証明書は無効と認識し、拒否する場合があります。
本番環境以外への展開（デプロイ）や、会社のファイアウォールの後ろで動くように展開（デプロイ）するには、自己署名 CA 証明書を全てのクライアントに配布し、ローカルで有効な証明書の一覧を更新できます。

クライアントごとに以下の操作を処理します：

```bash
$ sudo cp ca.crt /usr/local/share/ca-certificates/kubernetes.crt
$ sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d....
done.
```
## 証明書 API（Certificates API） {#certificates-api}

x509 証明書をプロビジョン（自動配置）するための認証で `certificates.k8s.io` API を使う場合は、[こちら](/jp/docs/tasks/tls/managing-tls-in-a-cluster) にドキュメントがあります。

{{% /capture %}}
