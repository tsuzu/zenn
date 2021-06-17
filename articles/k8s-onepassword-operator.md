---
title: "KubernetesのSecretを1Passwordで管理する" # 記事のタイトル
emoji: "🔐" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kubernetes", "1password"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# 1Password Connect Kubernetes Operator
GitHub: https://github.com/1Password/onepassword-operator

1Password上に保存されているアイテムをKubernetes上のSecretリソースに同期してくれるオペレーターです。Secretリソースはその特性上GitHubに直接あげることが難しく、IaCを実現するために様々なアプローチが取られていますが1Password Connect Kubernetes Operatorもその1つです。感覚としては[external-secrets](https://github.com/external-secrets/kubernetes-external-secrets)などに近く、外部のCredential Storeの情報を取得しKubernetes上にSecretリソースを作成してくれるものです。

1Password公式のOrganization下で開発されています。比較的最近開発が始まったプロジェクトのようで、First Commitは2020年12月でした。

# 余談
@[tweet](https://twitter.com/_tsuzu_/status/1404134241059831810)

先日自宅のKubernetesクラスタを構築し直し、その際にSecretリソースの管理をどうしようかと悩んでいました。ふと最近LastPassから移行した1Passwordの存在を思い出し、これが使えないかと調べていたところ発見しました。試してみたところ良さそうだったので記事を書いています。

クラスタ構築話についてもいつか書こうと思っています。(思っているだけ)

# デプロイ方法
私が使っているのはTeamsやBusinessではない個人向けプランですが利用することができました。自宅クラスタ勢の強い味方ですね。

## Secrets Automation workflowのセットアップ
簡単に言えばAPIトークンの発行です。詳しくは[Get started with a 1Password Secrets Automation workflow](https://support.1password.com/secrets-automation/)を参考にしてください。ドキュメント上部に `TEAMS AND BUSINESSES` と書いてあるので個人プランでは作れないかと思いましたが普通に作成することができました。

一点注意することがあるとすれば、個人用保管庫は使えません。Kubernetesで利用しない情報と混ざるのもよくないため予め1Password上で専用の保管庫を作成しておくことをオススメします。本来はdev、qa、prodなど環境ごとに分けるべきですが、自宅クラスタなので1つで。

![](https://storage.googleapis.com/zenn-user-upload/f8559f30ad62fb49c615ecc3.png)

セットアップが終わると資格情報ファイルとアクセストークンが得られます。どちらも保存しておく必要があります。ボタン1つで1Passwordに保存できるのは体験が良いですね。

![](https://storage.googleapis.com/zenn-user-upload/dd7691c67fdac3123f0b343d.png)

## 1Password Connectのデプロイ
1Passwordでは1Password Connectというソフトウェアが提供されています。これは1PasswordをREST API経由で利用するためのものです。1Passwordはサーバ上では暗号化されており、手元で復号化する必要があります。1Password Connectはそのためのプロキシのような動作をします。

1Password Connectのインストールは[Helm Chart](https://github.com/1Password/connect-helm-charts)が公式に提供されているためこれを用いてもインストールすることができます。

ですが、特に1Password Connectをカスタマイズする必要がない場合は1Password Connect Kubernetes Operatorに1Password Connectの管理も任せられるようなので今回はそれを使いました。

## 1Password Connect Kubernetes Operatorのデプロイ
やり方はREADMEにある通りであるため詳細は省略します。ただ、2021年6月18日現在注意すべき点がいくつかあったのでメモしておきます。

### $WATCH_NAMESPACEを空にする
deploy/operator.ymlの `$WATCH_NAMESPACE` がデフォルトでは制限されているためこの制限を外す必要があります。ただし、環境変数自体を消すとエラーになるため空白で残しておく必要があります。(パッとコードを見た感じv1.0.2以降では解決しそうな予感がします。)

### (default以外のnamespaceにデプロイする場合) permissions.ymlのClusterRoleBindingの修正
default namespace以外にデプロイする場合はpermissions.yml内のClusterRoleBindingのServiceAccountのnamespaceの指定がdefaultになっているためそれを書き換える必要があります。

### (default以外のnamespaceにデプロイする場合) 1Password Connectがdefault namespaceにインストールされてしまう
v1.0.0では、Operatorによって自動でインストールされる1Password Connectは強制でdefault namespaceにインストールされます。これを解決するためには自分で1Password Connectをインストールするか、最新のmainブランチを自身でビルドする必要があります。私は今回後者を取りました。

# 使用方法
## 1Passwordに追加する
1Passwordの初めに追加した保管庫に以下のような情報を追加してみました。

![](https://storage.googleapis.com/zenn-user-upload/818ca926c64af6bdd0fcd92b.png)

## OnePasswordItemリソースを作成する
`vaults/<保管庫名>/items/<アイテム名>` をitemPathに指定したOnePasswordItemを作成します。

```yaml
apiVersion: onepassword.com/v1
kind: OnePasswordItem
metadata:
  name: my-secret-token
spec:
  itemPath: "vaults/Kubernetes/items/my-secret-token" 
```

名称ではなくIDでも指定できるようです。共有ボタンからリンクを取得すると以下のようなリンクが取得でき、vが保管庫ID、iがアイテムのIDに相当するようです。

```
https://start.1password.com/open/i?a=...4&v=...&i=...&h=my.1password.com
```


applyして10秒ほどすると `my-secret-token` Secretが作成されました。

```console
tsuzu@MacBook-Pro onepassword-operator % k get secret
NAME                  TYPE                                  DATA   AGE
default-token-85pns   kubernetes.io/service-account-token   3      9m18s
my-secret-token       Opaque                                1      0s
```

```
tsuzu@MacBook-Pro onepassword-operator % k get secret my-secret-token -o yaml
apiVersion: v1
data:
  password: cXdlcnR5
kind: Secret
metadata:
  annotations:
    operator.1password.io/item-path: vaults/.../items/...
    operator.1password.io/item-version: "2"
...
  name: my-secret-token
  namespace: nginx
...
type: Opaque
```

1Password上でアイテムを更新したところ10秒かからずSecretリソースも更新されました。これは `operator.yaml` 内部の `$POLLING_INTERVAL` にも依存します。デフォルトで10に設定されておりかなり快適に使えるように思います。


## Auto Restartを試す
KubernetesのSecretリソースは更新されたとしてもすでに起動しているPodは更新されたデータを認識しないという問題があります。これを解決するために[Reloader](https://github.com/stakater/Reloader) というコントローラを使ったり、ConfigMapに関してはkustomizeのconfigMapGeneratorを使ったりします。

1Password Connect Kubernetes Operatorに関してはDeploymentの自動ローリングアップデートに対応しています。デフォルトでは無効化されていますが、オペレータ全体、namespace単位、Deployment単位、OnePasswordItem単位で有効化できるので必要に応じて有効化するのが良いと思います。

適当なDeploymentを用意し、`annotations`で `operator.1password.io/auto-restart: "true"` を追加しています。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
  annotations:
    operator.1password.io/auto-restart: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        volumeMounts:
        - name: my-secret-token
          mountPath: "/usr/share/nginx/html"
          readOnly: true
      volumes:
      - name: my-secret-token
        secret:
          secretName: my-secret-token
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: NodePort
```

予めフィールド名はindex.htmlというファイルに変更しておきます。
```
tsuzu@MacBook-Pro Downloads % op get item my-secret-token --vault Kubernetes | jq ".details.sections[1].fields[0].v"   
"previous secret"
tsuzu@MacBook-Pro Downloads % curl http://10.20.40.24:31217/
previous secret%   
tsuzu@MacBook-Pro Downloads % op edit item --vault Kubernetes my-secret-token ".index.html=new secret" # itemの更新
tsuzu@MacBook-Pro Downloads % curl http://10.20.40.24:31217/
new secret% 
```

15秒ほどで更新された情報が取得できました。

# おわりに
Kubernetesのシークレット管理の新しいOperatorとして[1Password Connect Kubernetes Operator](https://github.com/1Password/onepassword-operator) を試しました。 まだ登場したてではありますがローリングアップデート機能などもついていて便利に感じました。 あとはnamespace単位で利用できるシークレット名のprefixやtagの制限などができるとセキュリティ面でも強化できて良さそうな気がしています。

また、1Password Connectをデプロイしなければいけないというのがセキュリティ的な懸念になりうるのでは?という感じもあります。実際どのようなリスクがあるかはわかっていませんが調査をする必要があると思います。

少なくとも自宅Kubernetesクラスタの用途であれば完璧だったので今後はこれを利用していこうと思っています。 お読みいただきありがとうございました。
