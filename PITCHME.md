# Compose on Kubernetesとは

masaki nakayama

---

## Compose on Kubernetesとは

---

- docker/compose-on-kubernetes
https://github.com/docker/compose-on-kubernetes
- SIMPLIFYING KUBERNETES WITH DOCKER COMPOSE AND FRIENDS
https://blog.docker.com/2018/12/simplifying-kubernetes-with-docker-compose-and-friends/
---
> Compose on Kubernetesを使用すると、Docker ComposeファイルをKubernetesクラスタに展開できます。
> 任意のKubernetesクラスタでこの機能を使用することができます。
> 素のKubernetesを利用する場合、かなり多くのリソースを管理しなければならず、開発者にとって負担となります。そこで、開発者が簡単に扱えることを重視したcomposeを組み合わせ、Kubernetesの設定を簡素化する抽象化を提供します。

---

## 経緯

---

1. 今年の初めにdockerとKubernetesの統合を進める話があり、docker-for-desktopのedge版ではkube-composeという名前でCRDとして実装されていた。
2. その後、stable版でも実装されていたが、バイナリ化されていた。
3. dockercon EU 2018 で発表されてから、gitでも20日程前から公開された。

---

## アーキテクチャ

---

<img src="https://github.com/docker/compose-on-kubernetes/blob/master/docs/images/architecture.jpg?raw=true">
<https://github.com/docker/compose-on-kubernetes/blob/master/docs/architecture.md>より引用

---

### サーバーサイド
- API server
- Compose controller

### クライアントサイド
- Docker CLIの実装
- v1beta1/v2beta2があるが、前者は廃止、後者がデフォルトになる予定

---

## 現状

---

- Docker Desktop と Docker Enterpriseにインストール済
- AKSには自力でetcdとか色々作成すればインストールできる<https://github.com/docker/compose-on-kubernetes/blob/master/docs/install-on-aks.md>
- 同じ要領でGKEへのインストールを行ってもうまく動かない(コンポーネントのインストール自体はできるが、`docker stack deploy`時に失敗する)
<https://github.com/docker/compose-on-kubernetes/issues/21>
---
> dokcer-cli自体がgcp認証プロバイダをインポートしないため、cli自体がgkeに対して認証できないことが問題の原因です。cli修正のPRを作成するつもりです。
とのこと。順次、対応クラスターが増えていくと思われる。

---

## docker-for-desktopで試す

---

compose-on-kuberntesがインストールされていることの確認
```sh
$ kubectl api-versions | grep compose
compose.docker.com/v1beta1
compose.docker.com/v1beta2
```

---

compose on kubernetesの各コンポーネントは下記の通りデプロイ済である
```sh
$ kubectl get all -n docker
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/compose       1         1         1            1           24d
deploy/compose-api   1         1         1            1           24d

NAME                        DESIRED   CURRENT   READY     AGE
rs/compose-74649b4db6       1         1         1         24d
rs/compose-api-84464cb5c9   1         1         1         24d

NAME                              READY     STATUS    RESTARTS   AGE
po/compose-74649b4db6-7dqzm       1/1       Running   0          5d
po/compose-api-84464cb5c9-b6755   1/1       Running   0          5d

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
svc/compose-api   ClusterIP   10.106.134.97   <none>        443/TCP   24d
```
---
下記のdocker-compose.ymlを使用（dockerconのサンプル画面が映るだけのデモアプリ）

```sh
$ cat docker-compose.yml
version: '3.3'

services:

  hello-db:
    build: db
    image: dockersamples/k8s-wordsmith-db

  words:
    build: words
    image: dockersamples/k8s-wordsmith-api
    deploy:
      replicas: 5

  hello-web:
    build: web
    image: dockersamples/k8s-wordsmith-web
    ports:
     - "33000:80"
```
---
デプロイする
```sh
$ docker stack deploy --orchestrator=kubernetes -c docker-compose.yml hellokube
Ignoring unsupported options: build

service "hello-db": build is ignored
service "words": build is ignored
service "hello-web": build is ignored
Waiting for the stack to be stable and running...
hello-db: Ready		[pod status: 1/1 ready, 0/1 pending, 0/1 failed]
hello-web: Ready		[pod status: 1/1 ready, 0/1 pending, 0/1 failed]
words: Ready		[pod status: 5/5 ready, 0/5 pending, 0/5 failed]

Stack hellokube is stable and running
```

---


各種Kubernetesリソースが自動作成されている

```sh
$ kubectl get all
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/hello-db     1         1         1            1           50m
deploy/hello-web    1         1         1            1           50m
deploy/words        5         5         5            5           50m

NAME                       DESIRED   CURRENT   READY     AGE
rs/hello-db-799dbdfbb9     1         1         1         50m
rs/hello-web-9fdb4688d     1         1         1         50m
rs/words-6d654698d5        5         5         5         50m

NAME                             READY     STATUS    RESTARTS   AGE
po/hello-db-799dbdfbb9-rxz2d     1/1       Running   0          50m
po/hello-web-9fdb4688d-wz84s     1/1       Running   0          50m
po/words-6d654698d5-8b8q7        1/1       Running   0          50m
po/words-6d654698d5-j26mh        1/1       Running   0          50m
po/words-6d654698d5-j4bkl        1/1       Running   0          50m
po/words-6d654698d5-vr9cf        1/1       Running   0          50m
po/words-6d654698d5-xjb7c        1/1       Running   0          50m

NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
svc/hello-db               ClusterIP      None             <none>        55555/TCP         50m
svc/hello-web              ClusterIP      None             <none>        55555/TCP         50m
svc/hello-web-published    LoadBalancer   10.107.199.168   localhost     33000:30119/TCP   50m
svc/words                  ClusterIP      None             <none>        55555/TCP         50m
```

---


stackというKubernetesリソースとして作成されている

```sh
$ kubectl get stack
NAME         AGE
hellokube    53m
```

---

## compose on KubernetesをAKSにインストール
---
前提条件
- AKSクラスター（RBAC有効）を作成済
- クライアントにhelmインストール済
- Compose on Kubernetesインストーラーをダウンロードしていること
---
### 「compose」という名前空間を作成

```sh
$ kubectl create namespace compose
namespace/compose created
```
---
### etcdをデプロイ

- tillerサービスアカウントを作成
  ```sh
  $ kubectl -n kube-system create serviceaccount tiller
  serviceaccount/tiller created
  ```
---
- 作成したtillerサービスアカウントにクラスターへのadminto your cluster (note: you might want to reduce the scope of this): kubectl -n kube-system create ---- clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount kube-system:tiller
```sh
$ kubectl -n kube-system create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount kube-system:tiller
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```
---
- Run helm init --service-account tiller to initialize the helm component.

```sh
$ helm init --service-account tiller
Creating /home/planning-g/.helm
Creating /home/planning-g/.helm/repository
Creating /home/planning-g/.helm/repository/cache
Creating /home/planning-g/.helm/repository/local
Creating /home/planning-g/.helm/plugins
Creating /home/planning-g/.helm/starters
Creating /home/planning-g/.helm/cache/archive
Creating /home/planning-g/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /home/planning-g/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
```
---

## Deploy etcd operator and create an etcd cluster

- install the etcd-operator chart
```sh
$ helm install --name etcd-operator stable/etcd-operator --namespace compose
NAME:   etcd-operator
LAST DEPLOYED: Tue Dec 25 09:31:20 2018
NAMESPACE: compose
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                                             READY  STATUS             RESTARTS  AGE
etcd-operator-etcd-operator-etcd-backup-operator-56fd448cdrwmv9  0/1    ContainerCreating  0         1s
etcd-operator-etcd-operator-etcd-operator-c5b8b8f74-chnhx        0/1    ContainerCreating  0         1s
etcd-operator-etcd-operator-etcd-restore-operator-58587cdc9k52c  0/1    ContainerCreating  0         1s

==> v1/ServiceAccount
NAME                                               SECRETS  AGE
etcd-operator-etcd-operator-etcd-backup-operator   1        2s
etcd-operator-etcd-operator-etcd-operator          1        2s
etcd-operator-etcd-operator-etcd-restore-operator  1        2s

==> v1beta1/ClusterRole
NAME                                       AGE
etcd-operator-etcd-operator-etcd-operator  2s

==> v1beta1/ClusterRoleBinding
NAME                                               AGE
etcd-operator-etcd-operator-etcd-backup-operator   2s
etcd-operator-etcd-operator-etcd-operator          2s
etcd-operator-etcd-operator-etcd-restore-operator  2s

==> v1/Service
NAME                   TYPE       CLUSTER-IP   EXTERNAL-IP  PORT(S)    AGE
etcd-restore-operator  ClusterIP  10.0.132.45  <none>       19999/TCP  1s

==> v1beta2/Deployment
NAME                                               DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
etcd-operator-etcd-operator-etcd-backup-operator   1        1        1           0          1s
etcd-operator-etcd-operator-etcd-operator          1        1        1           0          1s
etcd-operator-etcd-operator-etcd-restore-operator  1        1        1           0          1s


NOTES:
1. etcd-operator deployed.
  If you would like to deploy an etcd-cluster set cluster.enabled to true in values.yaml
  Check the etcd-operator logs
    export POD=$(kubectl get pods -l app=etcd-operator-etcd-operator-etcd-operator --namespace compose --output name)
    kubectl logs $POD --namespace=compose
```
---

```yaml
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "compose-etcd"
  namespace: "compose"
spec:
  size: 3
  version: "3.2.13"
```


```sh
$ kubectl apply -f compose-etcd.yaml
etcdcluster.etcd.database.coreos.com/compose-etcd created
```

名前空間「compose」にetcdクラスターのリソースができている
```sh
$ kubectl get all -n compose
NAME                                                                  READY     STATUS    RESTARTS   AGE
pod/compose-etcd-dkk6hzqm4b                                           1/1       Running   0          1m
pod/compose-etcd-nmpd895rjl                                           1/1       Running   0          2m
pod/compose-etcd-xqfql85tg5                                           1/1       Running   0          1m
pod/etcd-operator-etcd-operator-etcd-backup-operator-56fd448cdrwmv9   1/1       Running   0          4m
pod/etcd-operator-etcd-operator-etcd-operator-c5b8b8f74-chnhx         1/1       Running   0          4m
pod/etcd-operator-etcd-operator-etcd-restore-operator-58587cdc9k52c   1/1       Running   0          4m

NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
service/compose-etcd            ClusterIP   None          <none>        2379/TCP,2380/TCP   2m
service/compose-etcd-client     ClusterIP   10.0.197.57   <none>        2379/TCP            2m
service/etcd-restore-operator   ClusterIP   10.0.132.45   <none>        19999/TCP           4m

NAME                                                                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/etcd-operator-etcd-operator-etcd-backup-operator    1         1         1            1           4m
deployment.apps/etcd-operator-etcd-operator-etcd-operator           1         1         1            1           4m
deployment.apps/etcd-operator-etcd-operator-etcd-restore-operator   1         1         1            1           4m

NAME                                                                           DESIRED   CURRENT   READY     AGE
replicaset.apps/etcd-operator-etcd-operator-etcd-backup-operator-56fd448cd6    1         1         1         4m
replicaset.apps/etcd-operator-etcd-operator-etcd-operator-c5b8b8f74            1         1         1         4m
replicaset.apps/etcd-operator-etcd-operator-etcd-restore-operator-58587cdccf   1         1         1         4m
```

---

compose-on-kubernetesをインストール（※Cloud Shellでバイナリ認識できなかったのでmacからaksクラスター指定してインストールさせてます）
```sh
$ ./installer-darwin -namespace=compose -etcd-servers=http://compose-etcd-client:2379 -tag=v0.4.16 -skip-liveness-probes=true --kubeconfig=/Users/nakayama/.kube/aksconfig
INFO[0000] Checking installation state
INFO[0001] Install image with tag "v0.4.16" in namespace "compose"
INFO[0003] Api server: image: "docker/kube-compose-api-server:v0.4.16", pullPolicy: "Always"
INFO[0003] Controller: image: "docker/kube-compose-controller:v0.4.16", pullPolicy: "Always"
```

---
インストールを確認
```sh
$ kubectl api-versions | grep compose
compose.docker.com/v1beta1
compose.docker.com/v1beta2
```
---
```sh
$ kubectl get all -n compose
NAME                                                                  READY     STATUS    RESTARTS   AGE
pod/compose-5c84c46dd5-4ldpn                                          1/1       Running   0          4m
pod/compose-api-8694d77dd-5drfk                                       1/1       Running   0          4m
pod/compose-etcd-dkk6hzqm4b                                           1/1       Running   0          40m
pod/compose-etcd-nmpd895rjl                                           1/1       Running   0          40m
pod/compose-etcd-xqfql85tg5                                           1/1       Running   0          40m
pod/etcd-operator-etcd-operator-etcd-backup-operator-56fd448cdrwmv9   1/1       Running   0          43m
pod/etcd-operator-etcd-operator-etcd-operator-c5b8b8f74-chnhx         1/1       Running   0          43m
pod/etcd-operator-etcd-operator-etcd-restore-operator-58587cdc9k52c   1/1       Running   0          43m

NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
service/compose-api             ClusterIP   10.0.90.8     <none>        443/TCP             4m
service/compose-etcd            ClusterIP   None          <none>        2379/TCP,2380/TCP   40m
service/compose-etcd-client     ClusterIP   10.0.197.57   <none>        2379/TCP            40m
service/etcd-restore-operator   ClusterIP   10.0.132.45   <none>        19999/TCP           43m

NAME                                                                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/compose                                             1         1         1            1           4m
deployment.apps/compose-api                                         1         1         1            1           4m
deployment.apps/etcd-operator-etcd-operator-etcd-backup-operator    1         1         1            1           43m
deployment.apps/etcd-operator-etcd-operator-etcd-operator           1         1         1            1           43m
deployment.apps/etcd-operator-etcd-operator-etcd-restore-operator   1         1         1            1           43m

NAME                                                                           DESIRED   CURRENT   READY     AGE
replicaset.apps/compose-5c84c46dd5                                             1         1         1         4m
replicaset.apps/compose-api-8694d77dd                                          1         1         1         4m
replicaset.apps/etcd-operator-etcd-operator-etcd-backup-operator-56fd448cd6    1         1         1         43m
replicaset.apps/etcd-operator-etcd-operator-etcd-operator-c5b8b8f74            1         1         1         43m
replicaset.apps/etcd-operator-etcd-operator-etcd-restore-operator-58587cdccf   1         1         1         43m

```
---

```sh
$ docker stack deploy --orchestrator=kubernetes -c docker-compose.yml hellokube
Ignoring unsupported options: build

service "web": build is ignored
service "db": build is ignored
service "words": build is ignored
Waiting for the stack to be stable and running...
words: Ready            [pod status: 1/5 ready, 4/5 pending, 0/5 failed]
db: Ready               [pod status: 1/1 ready, 0/1 pending, 0/1 failed]
web: Ready              [pod status: 1/1 ready, 0/1 pending, 0/1 failed]

Stack hellokube is stable and running
```

---

## 余談：GKEへのインストール

```sh
$ kubectl create namespace compose
```

```
$ kubectl -n kube-system create serviceaccount tiller
```

```
$ helm init --service-account tiller
```

```
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

```
$ helm install --name etcd-operator stable/etcd-operator --namespace compose
NAME:   etcd-operator
LAST DEPLOYED: Thu Dec 20 23:38:08 2018
NAMESPACE: compose
STATUS: DEPLOYED

RESOURCES:
==> v1beta1/ClusterRole
NAME                                       AGE
etcd-operator-etcd-operator-etcd-operator  1s

==> v1beta1/ClusterRoleBinding
NAME                                               AGE
etcd-operator-etcd-operator-etcd-backup-operator   1s
etcd-operator-etcd-operator-etcd-operator          1s
etcd-operator-etcd-operator-etcd-restore-operator  1s

==> v1/Service
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)    AGE
etcd-restore-operator  ClusterIP  10.55.250.232  <none>       19999/TCP  1s

==> v1beta2/Deployment
NAME                                               DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
etcd-operator-etcd-operator-etcd-backup-operator   1        1        1           0          1s
etcd-operator-etcd-operator-etcd-operator          1        1        1           0          1s
etcd-operator-etcd-operator-etcd-restore-operator  1        1        1           0          1s

==> v1/Pod(related)
NAME                                                             READY  STATUS             RESTARTS  AGE
etcd-operator-etcd-operator-etcd-backup-operator-67f596795jn6t8  0/1    ContainerCreating  0         1s
etcd-operator-etcd-operator-etcd-operator-698ddd8ff9-6fkvn       0/1    ContainerCreating  0         1s
etcd-operator-etcd-operator-etcd-restore-operator-586d5c5b7b5ww  0/1    ContainerCreating  0         1s

==> v1/ServiceAccount
NAME                                               SECRETS  AGE
etcd-operator-etcd-operator-etcd-backup-operator   1        1s
etcd-operator-etcd-operator-etcd-operator          1        1s
etcd-operator-etcd-operator-etcd-restore-operator  1        1s


NOTES:
1. etcd-operator deployed.
  If you would like to deploy an etcd-cluster set cluster.enabled to true in values.yaml
  Check the etcd-operator logs
    export POD=$(kubectl get pods -l app=etcd-operator-etcd-operator-etcd-operator --namespace compose --output name)
    kubectl logs $POD --namespace=compose
```

```
kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=m.nakayama1023@gmail.com
```

```
$ ./installer-darwin -namespace=compose -etcd-servers=http://compose-etcd-client:2379 -tag=v0.4.16
INFO[0000] Checking installation state
INFO[0000] Install image with tag "v0.4.16" in namespace "compose"
INFO[0001] Api server: image: "docker/kube-compose-api-server:v0.4.16", pullPolicy: "Always"
INFO[0001] Controller: image: "docker/kube-compose-controller:v0.4.16", pullPolicy: "Always"
```


```
$ kubectl get pods --namespace compose
NAME                                                              READY     STATUS    RESTARTS   AGE
compose-6975f48fff-mv8mj                                          1/1       Running   0          2m
compose-api-6bdbb8f9d5-nhqld                                      1/1       Running   0          2m
compose-etcd-r7n5ppm2kq                                           1/1       Running   0          8m
compose-etcd-rz9krlhccv                                           1/1       Running   0          9m
compose-etcd-tzcmtw9q4j                                           1/1       Running   0          8m
etcd-operator-etcd-operator-etcd-backup-operator-67f596795jn6t8   1/1       Running   0          22m
etcd-operator-etcd-operator-etcd-operator-698ddd8ff9-6fkvn        1/1       Running   0          22m
etcd-operator-etcd-operator-etcd-restore-operator-586d5c5b7b5ww   1/1       Running   0          22m
```

```
$ kubectl api-versions | grep compose
compose.docker.com/v1beta1
compose.docker.com/v1beta2
```

```
$ docker stack deploy --orchestrator=kubernetes -c docker-compose.yml hellokube

unable to deploy to Kubernetes: No Auth Provider found for name "gcp"
```

- deployment
- replicaset
- service
- pod
  
は作成してくれている

あれ、volumeは・・・？

https://github.com/docker/compose-on-kubernetes/issues/10
- 実は、データボリュームのマウントを永続的なボリュームクレームとしてすでに変換している。すべてのpvcオプションを公開していないが、今後追加していくつもり
- ホストバインドは対応している
- 今後のPRで出てくるが、今作ってる内部資料の抜粋では

こんな感じにpv指定できるという

```
version: "3.6"

services:
  mysql:
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```

pvcはこんな感じ

```
version: "3.6"

services:
  web:
    image: nginx:alpine
    volumes:
      - type: bind
        source: /srv/data/static
        target: /opt/app/static
```

