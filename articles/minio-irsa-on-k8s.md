---
title: "MinIOをKubernetesからアクセスキーなしで使いたい！" # 記事のタイトル
emoji: "🔑" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kubernetes", "minio"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# TL;DR
* 同一クラスタ内 && minio-operatorを使う場合
  * →[公式のexample](https://github.com/minio/operator/tree/v5.0.11/examples/kustomization/sts-example)に従おう
  * (こっちだとかなり楽に設定できそう)
* 外部のMinIO || minio-operator以外の手順でMinIOを構築している場合
  * → MinIOのOpenID Connect設定に追加して頑張る **←今回はこっちメインの話**

# はじめに
本当はAdvent Calendarのネタとして用意してたんですが、色々躓いてminioにPR送ったりしていたら遅くなりました。

自称Secret撲滅エンジニア、[Tsuzu](https://twitter.com/_tsuzu_) です。
今年NASを自作しTrueNAS Scaleをインストールするなど自宅サーバの拡張を進めています。
また、R86Sを購入したりフレッツ光クロスを引いて10G化を進めた1年でした。

@[tweet](https://x.com/_tsuzu_/status/1736774832455659790)

そんな中で、S3互換オブジェクトストレージが欲しくなったので、NAS上で[MinIO](https://min.io/) を立ててみました。MinIOはGoで書かれたオープンソースのS3互換のオブジェクトストレージです。

Kuberenetes上にインストールする場合は[minio-operatorを利用したインストール](https://min.io/docs/minio/kubernetes/upstream/operations/installation.html) が公式では推奨されていますが、沢山のテナントを立てるわけでもないのでoperatorは使わず1台だけ建てました。

NASの外部にあるKubernetesクラスタを自前で運用しており、ここから接続してMinIOを利用したくなりました。MinIOはS3と互換性を持っているため、接続にはAccess Key IDとSecret Access Keyを使うのが一般的です。

しかし、S3のためのAccess Keyをたくさん発行してSecret Storeに保存したりするのは、発行自体にコストがかかるだけでなく、流出時のリスクも高まり、更新時のコストも同様に発生します。そんなSecretは撲滅しましょう。

## IAM Roles for Service Accounts(IRSA)について
AWSでEKS(on EC2)からAWSのAPIを叩く際には大きく分けて3つの方法があります。「愚直にAWS IAM Userを発行する方法」、「EC2 Instanceに紐づいたIAM Roleを使う方法」、そして「IAM Roles for Service Accounts(IRSA)」を使う方法です。

追記
...と思っていたら [EKS Pod Identity](https://www.kaitoy.xyz/2023/12/25/eks-pod-identity/)というのが増えていたらしい...知らなかった。これから追います

「愚直にAWS IAM Userを発行する方法」については管理コストがかかり面倒だったり、「EC2 Instanceに紐づいたIAM Roleを使う方法」についてはノードごとに権限が付与されてしまうためPod単位の制御ができずセキュアでない、などの問題があります。それを解決するのが [IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) を使う方法です。

この後につながるので簡単に仕組みについて説明します。すでに知っている方は読み飛ばしてください。

PodでService Accountを利用する際、kube-apiserverが信頼するOpenID Connect ProviderのID Tokenが付与されます。これはPod作成時に自動的にPodにProjected Volumeとしてmountされるようになっています。これはTokenRequestProjectionという仕組みを通じて実現されています。

```console
$ k run --image busybox test -- sleep infinity 
pod/test created
$ k get po test -o yaml | yq .spec.volumes
- name: kube-api-access-97x8t
  projected:
    defaultMode: 420
    sources:
      - serviceAccountToken: # ←これ
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
            - key: ca.crt
              path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
            - fieldRef:
                apiVersion: v1
                fieldPath: metadata.namespace
              path: namespace
```

この機能はService Account以外の用途で使うこともできます。こんな感じでaudience(Client ID)を明示的に指定すると、そのaudience向けのID Tokenを発行することができます。

```yaml
- name: aws-iam-token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 86400
        audience: sts.amazonaws.com
```

これによって発行されたID TokenのJWTを見てみるとaudに適切に指定したaudienceが入っています。

```json
{
  "aud": [
    "sts.amazonaws.com"
  ],
  "exp": 1704041245,
  "iat": 1703954845,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "my-pod",
      "uid": "c0e21bfe-fd34-485b-bcc2-7d58c7d0fc8f"
    },
    "serviceaccount": {
      "name": "default",
      "uid": "c3dcebd8-d895-40f4-89d4-d8b1da80abb0"
    }
  },
  "nbf": 1703954845,
  "sub": "system:serviceaccount:default:default"
}
```

AWSのIAMにはAssume Role with Web IdentityというAPIがあり、外部のOpenID Connect Providerを信頼することができます。ID Tokenを付与して[STSのAssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html)を叩くと、一時的なcredentialが付与され、通常通りAWSのAPIが叩けるようになります。

詳細な動作については AWS公式の [Diving into IAM Roles for Service Accounts](https://aws.amazon.com/jp/blogs/containers/diving-into-iam-roles-for-service-accounts/) が詳細に書いているのでおすすめです。

# MinIOでもIAM Roles for Service Accounts
## 同一クラスタかつminio-operatorの場合
同じことがMinIOでもできるようになっています。同一クラスタ内に [minio-operator](https://github.com/minio/operator)を利用している場合は公式ドキュメントの [MinIO Operator STS: Native IAM Authentication for Kubernetes](https://github.com/minio/operator/tree/v5.0.11/examples/kustomization/sts-example)を参考に構築すれば動く(はず)です。ただ、私は試していないのでわかりません。


## 別クラスタ、もしくはminio-opratorを使わない場合
### openid-configurationの公開
別クラスタを利用している、もしくはminio-operatorを利用せずに構築している場合は上記の手順が使えないので自力で頑張る必要があります。ここからはその話を書きます。
ドキュメント自体は [Configure MinIO for Authentication using OpenID](https://min.io/docs/minio/windows/operations/external-iam/configure-openid-external-identity-management.html) のページなどにあります。

まず、MinIOがID Tokenを検証するための情報を取得するエンドポイントを指定する必要があります。 kube-apiserverの `/.well-known/openid-configuration` をMinIOからアクセスできるようにすれば良いのですが、通常は認証済みユーザしかアクセスできません。本来は直接kube-apiserverを許可するよりはIngress等を介してあげた方がセキュリティ的には良いのですが、どうせインターネットからはアクセスできないのでkube-apiserverにanonymousでもアクセスできるようにしました。以下の `ClusterRoleBinding` をapplyして上げれば許可できます。`ClusterRoleは` 元々あるので作成不要です。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openid-configuration
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:service-account-issuer-discovery
subjects:
- kind: User
  name: system:anonymous
  apiGroup: rbac.authorization.k8s.io
```

kube-apiserverはデフォルトでanonymous-authが有効ですが、k3sでは無効化されているので有効化するのをお忘れなきよう...私はハマりました。

### kube-apiserverのCA証明書をMinIOで信頼する
kube-apiserverが使用する証明書はクラスタ内の独自CA certificateであることが多いのでMinIOから接続する際に信頼できるようにする必要があります。適当なConfigMapを利用して `/root/.minio/certs/CAs` に配置するようにします。コンテナ以外で動かす場合は `/root` 以外になりうるのでいい感じにしてください。

### MinIOでOpenID Connectの設定をする

Web UIからも出来ますがIaCがお好きなみなさんはYAMLで設定投入したいと思うのでそのやり方で書いておきます。以下を環境変数に突っ込みましょう。

```
MINIO_IDENTITY_OPENID_CONFIG_URL=https://{{ kube-apiserverのエンドポイント }}:6443/.well-known/openid-configuration
MINIO_IDENTITY_OPENID_CLIENT_ID=sts.amazonaws.com
MINIO_IDENTITY_OPENID_CLIENT_SECRET=dummy
MINIO_IDENTITY_OPENID_ROLE_POLICY=readwrite
MINIO_IDENTITY_OPENID_DISPLAY_NAME=k8s
```

OpenID Config URLは先ほどのURLです。
Client IDはAWSでの値に合わせます。合わせなくてもMinIO自体は問題ありませんが、AWS CLIを使うことを考えると合わせておいた方が楽です。Client SecretはWeb UIでのログインフローを使わないので適当で良いです。Role Policyは

```console
$ mc admin policy ls truenas-scale
consoleAdmin
diagnostics
readonly
readwrite
writeonly
```

やUIで取得してきた好きなPolicyを指定しておきます。ただ、後々を考えると別途用意した方が良いとは思います。

この設定を適用してminioを起動すると起動時に以下のようなIAM Rolesが出力されているのでこれをメモっておきます。MinIOで利用する場合はAssumeするRoleがこの起動時に表示される1つのみになるようです。Roleを自由に作れないのはちょっと残念。

```
IAM Roles: arn:minio:iam:::role/oxnUw3DAuQF1uggjXGxI2AmW8IA
```

じゃあ権限をService Accountごとに制御できないのかというと一応PolicyのConditionで制御はできるらしいです。Policyは自前で別途用意した方が良い、というのはこれが理由です。
https://min.io/docs/minio/windows/administration/identity-access-management/oidc-access-management.html#minio-external-identity-management-openid-access-control

### PodからMinIOに接続する

以上で設定はできたのでPodに設定を投入していきます。AWSでIAM Roles for Service Accountsを使う場合はMutation Webhookが勝手にこのあたりの設定を投入してくれますがMinIOの場合は自分で投入します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: amazon/aws-cli:latest
    command:
    - bash
    - -c
    - "sleep infinity"
    ports:
    - containerPort: 80
    env:
    - name: AWS_ROLE_ARN
      value: {{ 先ほどログから取得したARN }}
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token # この辺りもAWSの仕様に合わせています
    - name: AWS_ENDPOINT_URL_STS
      value: http://{{ MinIOのendpoint(consoleではなくAPIのため注意) }}
    - name: AWS_ENDPOINT_URL_S3
      value: http://{{ MinIOのendpoint(consoleではなくAPIのため注意) }}
    volumeMounts:
    - name: aws-iam-token
      readOnly: true
      mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
  volumes:
  - name: aws-iam-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 86400
          audience: sts.amazonaws.com
```

以下のようにexecして接続できていればOKです。
```console
$ pbpaste | k apply -f -
$ k exec -n default -ti my-pod -- bash
bash-4.2# aws s3api create-bucket --bucket test
{
    "Location": "/test"
}
bash-4.2# aws s3 ls
2023-12-31 12:41:09 test
```

### 注意点
2023/12/31現在、kube-apiserverのOpenID Connectの設定をMinIOに追加しているとMinIOのconsoleが見られないという問題があります。

![minio consoleでAn error has occurredにより開けない様子](/images/minio-error.png)

https://github.com/minio/console/pull/3168 のPRで修正されているのですが、まだリリースタグが打たれていないのでがリリースされるのを待った方が良いでしょう。OpenIDでログインできないだけでなく、ID/Passwordでもログインできません。

## litestreamでMinIOのIRSAを使う
自宅クラスタでMySQLやPostgreSQLを用意するのは地味に面倒で、PV等を使わずサクッと作りたい時があります。そんな時はSQLiteを使って[litestream](https://litestream.io/) でS3にreplicateし続けて、起動時にS3から復旧することでPVを利用せずDBを使うことができます。

そのlitestreamで今回のIRSA for MinIOを使う方法を紹介しておきます。こんな感じでlitestreamのconfigを用意します。

```yaml
dbs:
  - path: /db/app.db
    replicas:
      - type: s3
        endpoint: {{ MinIOのS3 API endpoint (http://example.com:9000) }}
        bucket: app
        path: app.db
```

上記コンフィグをConfigMapとして読み込み、emptyDirのvolumeをinitContainerとsidecarでlitestreamを動かします。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  serviceName: app  
  template:
    metadata:
      labels:
        app: app
    spec:
      initContainers:
      - name: litestream-restore
        image: litestream/litestream:0.3
        args: ['restore', '-if-db-not-exists', '-if-replica-exists', '-config', '/config/litestream.yml', '/db/app.db']
        env:
        - name: AWS_ROLE_ARN
          value: {{ 先ほどログから取得したARN }}
        - name: AWS_WEB_IDENTITY_TOKEN_FILE
          value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
        volumeMounts:
        - name: aws-iam-token
          readOnly: true
          mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
        - name: db
          mountPath: /db
        - name: litestream-config
          mountPath: /config
          readOnly: true
      containers:
      - name: litestream-replicate
        image: litestream/litestream:0.3
        args: ['replicate', '-config', '/config/litestream.yml']
        env:
        - name: AWS_ROLE_ARN
          value: {{ 先ほどログから取得したARN }}
        - name: AWS_WEB_IDENTITY_TOKEN_FILE
          value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
        volumeMounts:
        - name: aws-iam-token
          readOnly: true
          mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
        - name: db
          mountPath: /db
        - name: litestream-config
          mountPath: /config
          readOnly: true
      - name: app
        image: ... # 省略
      volumes:
      - name: aws-iam-token
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 86400
              audience: sts.amazonaws.com
      - name: db
        emptyDir: {}
      - name: litestream-config
        configMap:
          name: litestream-config
```

参考: https://zenn.dev/mattn/articles/fef682a8b204ac

# 終わりに
以上が、AssumeRoleWithWebIdentityを使ってMinIOでSecretを使わずにIAM Roles for Service Accountsでアクセスする方法でした。ちなみにaws-sdk-goとかもLoadDefaultConfig()するだけで勝手にIRSAを使ってくれるので便利です。余裕があったらMutating Webhookも用意してもっと楽にしたい気持ちがあります。

私はSecretを無くしたかっただけなのに意外とやることが多くて大変でしたが皆さんも是非リスクの原因となるSecretはどんどん撲滅していきましょう。

それでは良いお年を！
