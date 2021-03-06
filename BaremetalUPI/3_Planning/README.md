# 3. クラスターインストールの計画
本章では、`OpenShift`のBaremetal UPIでインストールするクラスターの設計について説明します。
OpenShiftクラスターは様々な要素によって構成されていますが、代表的なコンポーネントとして、

- ノード
- ネットワーク
- ストレージ

が挙げられます。  
本章ではこれらと、他に必要となるサービスの設計について説明します。

---

## 3-1 設計する構成の全体像
![システム構成図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/9bc34987-cd99-c9a7-9c61-4b9bca0c547a.png)

本章では上図のような構成の`OpenShift`クラスターを設計しています。

手元の実験環境を作るものというよりは、本番環境で最低限必要なコンポーネントを何かを考えて、冗長化が必要なコンポーネントは分割しています。

### 3-1-2. 本構成の制限

`DNS(BIND)`、`LoadBalancer(HA Proxy)`は、本番環境であれば冗長化するべき所ですが、手順が冗長になってしまい複雑になってしまうため、この手順の中では、触れていません。

また ID管理のコンポーネントである`IdM` は、`OpenShift`の動作確認後、後から追加構築できる部分でもあるので、時間的都合もありこの手順書では触れていません(後から追加できれば追加します)。

### 3-1-3. 本構成のメリット・デメリット

この構成のメリットは、実際のフルの運用環境と比べると**比較的**構成が小さい事です。

一方で、`OpenShift`ではクラスタの中に監視やログ収集のコンポーネントやストレージまでが含まれています。そのため、頻繁に発生する`Kuberentes`のアップデート時において、監視やログ収集がまともに機能しません。

`OpenShift`は、クリック一つで全体をバージョンアップできる`OTA(Over The Air)`アップデートができますが、監視システムもまとめてアップデートされるためアップデート中にいろいろな`Warning`やエラーが上がっても、状況がつかめませんでした。

もともと物理サーバーの時代より**監視サーバーは、監視対象と同じプラットフォームに同居させてはいけない**というのは先人の知恵から言われていた事ですが、実際には仮想化が普及しはじめてから、同じ仮想化プラットフォーム上に監視サーバーが載っている例も多かったと思います。

これは`vMontion`や`Live Migration`と言った`ハイパーバイザー`のアップデート時にコンポーネントを待避させる仕組みや、監視サーバーの分離度が仮想マシンという形で他と大きく分離しているという観点で、現実的に監視が停止する確率がそれほど高くなかったためだと思います。

一方で現状、3ヶ月に一回アップデートが走る`Kubernetes`の世界では、監視は監視対象が乗るプラットフォームと分割するという原則は本番環境では特に守った方が良いと思いました。

が、いずれにしてもこの手順では、リソースの都合で、そこまでの環境を整備する事までは行っておらず、`OpenShift`と`OpenShift`に含まれている`OSS`製品の構築方法、分散ストレージである`OCS(OpenShift Container Storage)` の構築の作業手順やその雰囲気を届ける事を目的としたいと思います。

## 3-2. ノードの設計

ここでは以下のような `CPU`/`Memory`/`Disk`サイズのノードを使用しました。あくまでこのくらいのスペックがあれば、インストール、`OpenShift`の基本操作に問題がないように見えるというレベルであり、どんな負荷でも耐用できると保証するものではありません。使用状況によって必要なスペックが違う事については留意して下さい。

### 3-2-1. OpenShiftノードのサイジング

| サーバー        | vCPU(HT-on) | Memory      |  Disk   |    OS   |         ホスト名            | IP Address  |  note   | 
|:---------------|:--------------|:------------|:--------|:--------|:---------------------------|:------------|:------|
|BootStrapノード  | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |bs.ocp45.example.localdomain|172.16.0.11 |一時的|
|Masterノード     | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |m1.ocp45.example.localdomain| 172.16.0.21 |     | 
|                | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |m2.ocp45.example.localdomain| 172.16.0.22 |     | 
|                | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |m3.ocp45.example.localdomain| 172.16.0.23 |     | 
|Workerノード     | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |w1.ocp45.example.localdomain |172.16.0.31 |     | 
|                | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |w2.ocp45.example.localdomain| 172.16.0.32 |     | 
|                | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |w3.ocp45.example.localdomain| 172.16.0.33 |     | 
|Infraノード      | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |i1.ocp45.example.localdomain|172.16.0.41 |     | 
|                | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |i2.ocp45.example.localdomain|172.16.0.42 |     | 
|                | 4vCPU         | 16GByte     | 120G    |  RHEL CoreOS |i3.ocp45.example.localdomain|172.16.0.43 |     | 
|OCSノード        | 16vCPU        | 32GByte     | 120G    |  RHEL CoreOS |s1.ocp45.example.localdomain |172.16.0.51 | 1 TiB SSD x3 別途搭載 | 
|                | 16vCPU        | 32GByte     | 120G    |  RHEL CoreOS |s2.ocp45.example.localdomain| 172.16.0.52 | 1 TiB SSD x3 別途搭載 | 
|                | 16vCPU        | 32GByte     | 120G    |  RHEL CoreOS |s3.ocp45.example.localdomain| 172.16.0.53 | 1 TiB SSD x3 別途搭載 | 


※各`ノード`のスペックは、OpenShift 4.5 のマニュアルを基準にしています。<a href="https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#minimum-resource-requirements_installing-restricted-networks-bare-metal">Minimum resource requirements</a>

#### BootStrapノード

`BootStrapノード`は、`Masterノード`をセットアップするために必要なサーバーで、`Masterノード`のセットアップが完了した後は必要の無い、一時的なノードです。

`BootStrapノード`と`Masterノード`は、`RHEL CoreOS`が必須です。`Workerノード`と`Infraノード`は、`RHEL`もしくは`RHEL CoreOS`が選択できるのですが、この手順ではせっかくなのでコンテナ専用のOSである`RHEL CoreOS`を使用します。

#### Masterノード
`Masterノード`は、3ノード必要です。

`Masterノード`に必要となるスペックは、[Masterノードの最小リソース要件](https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.5/html/installing_on_bare_metal/installing-on-bare-metal#installation-requirements-user-infra_installing-bare-metal)のリンク先が最小要件ですが、管理対象の`Workerノード`の数で変わります。

Master Nodeはスケールアウトしたり、後からCPUやRAMのサイズを変更することができないため、あらかじめクラスターに配備する`Workerノード`の最大数を想定してスペックを決めるようにしてください。[Masterノードの推奨スペック](https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.5/html/scalability_and_performance/master-node-sizing_)のリンク先が参考となります。


#### Workerノード
`Workerノード`は、ユーザーのアプリケーション・コンテナが載る`ノード`です。最低2`ノード`からですが、この手順では3`ノード`作ります。
`OpenShift`としての最小構成では`Masterノード`と`Workerノード`を統合して合計で3`ノード`にする方法もありますが、この手法は`Edge`ロケーションでHWリソースが潤沢に取れない場合などのユースケースが想定されているため、この設計では採用しませんでした。

`Workerノード`に必要となるスペックは、[Workerノードの最小リソース要件](https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.5/html/installing_on_bare_metal/installing-on-bare-metal#installation-requirements-user-infra_installing-bare-metal)のリンク先が最小要件です。

実際には`Workerノード`で稼働するPodが求めるリソースに依存するので、一概に推奨スペックを言うことをは難しいです。あらかじめ稼働するアプリケーションが全て分かっている場合は必要なリソースを計算できますが、分からない場合は暫定的にスペックを決めてスケールアウトする方針がよいでしょう。

#### Infraノード
`Infraノード`と言う言葉は、`OpenShift` 独自の呼び方で、実体は`Workerノード` です。`Kubernetes`の**運用管理コンポーネント**を、ユーザーアプリと分離した`ノード`導入するために用意する`ノード`です。

> :information_source: ちなみに、`Kubernetes` では、`Master`のみが正式に定義されている`ノード`の`Role`で`Workerノード`という言葉はしばしば用いられますが、正式な`Role`としては定義されていません。(`Kubernetes`の`Workerノード`の`kubectl get node` の `Role`欄は `<none>`と表示されます) 

`OpenShift 4.x`では、`Master`と`Worker`がインストール時に`ノード`の`Role`として事前に定義されており、さらにこの手順では追加で`Infra`という`Role`を定義します。


`Infraノード`には、`Elastic Search`や、`Kibana`、`Prometheus`等のコンポーネントが導入されます。これらのコンポーネントは大量にリソースを消費しやすいため、ユーザーアプリが使用する`ノード`とは分けて導入する事がベストプラクティスです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/8382dad0-ecad-adae-8027-2ef996dc3ad6.png)

`OpenShift`では、ログ収集用の `ElasticSearch`や、コンテナを保管するための`Container Registry`、監視用の `Prometheus`さらに `OpenShift`の`Ingress`実装である`Router`と呼ばれる`Kubernetes`を管理するための**運用コンポーネント**があります。

こう言った**運用コンポーネント**は、ユーザーのアプリケーションに影響を与えないように`Infraノード`に分離する事にします。
つまり以下のようなイメージです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/80bb1c74-34ef-0586-5ff2-92b16dd2c92d.png)

#### OCSノード

『3-4. ストレージの設計』の節で説明します。

### 3-2-2. 周辺インフラ・サーバーのサイジング

| サーバー        | vCPU(HT-on) | Memory      |  Disk   |  OS      |        ホスト名            | IP Address | note|
|:---------------|:--------------|:------------|:--------|:---------|:---------------------------|:------|:------|
|踏み台(bastion)  | 2vCPU         | 4GByte      | 60G    | RHEL 8.2 |bastion.example.localdomain|172.16.0.101|       |
|DNS / DHCP      | 2vCPU         | 4GByte      | 60G    | RHEL 8.2 |ns1.example.localdomain     |172.16.0.102|        |
|                | 2vCPU         | 4GByte      | 60G    | RHEL 8.2 |ns2.example.localdomain     |172.16.0.103| 冗長用 |
|LoadBalancer    | 2vCPU         | 4GByte      | 60G    | RHEL 8.2 |lb1.example.localdomain     |172.16.0.110/111|       |
|                | 2vCPU         | 4GByte      | 60G    | RHEL 8.2 |lb2.example.localdomain     |172.16.0.120/121|  冗長用 |


`Kubernetes` には含まれていませんが、`Kubernetes`を動かすために必要なサーバー群です。

#### 踏み台(bastion)
`OpenShift`インストール時に作業の起点となるサーバーです。`kubectl`、`oc`コマンドなどの実行場所です。
この例では、`ノード`に対するインストール・イメージなどの配布用に`TFTPサーバー`や、`HTTPサーバー(nginx)` も導入します。 
可用性は必要無いので、冗長化は行っていません。

#### Load Balancer
この例では、`ノード`間の負荷分散装置のめに、`Load Balancer`として、`HA Proxy`を使用しています。

クラスター内部の`ノード`間通信の負荷分散用と、クラスター外部から来るアクセスの負荷分散の2種類の`Load Balancer`が必要になります。
ここでは、`内部`用の `Load Balancer`と`外部`用の `Load Balancer`で一つのサーバーに別々のIPをアサインする設計にしています。

設計として冗長構成にするために、2本の`Load Balancer`を用意していますが、構築手順としてはこの手順の中では触れていません。

#### DNS/DHCP
この手順では`Inter-Node`ネットワークに所属するサーバーの名前解決のための`DNSサーバー`として`BIND`を使用しています。

`DHCPサーバー`は、`Masterノード`、`Workerノード`、`Infraノード`に対して、`DNSサーバー`や、`gateway`のIPアドレスを教えるため等に使われます。
`DHCPサーバー`は使用しますが、`UPI`インストールでは、これらの`ノード`間の負荷分散を行うために、各`ノード`の`IPアドレス`、もしくは名前を事前に`Load Balancer`に登録して置く必要があります。

そのため、`DHCP`で自由に`IPアドレス`を割り当てるのではなく、各`ノード`の `MACアドレス`を使用して、固定IPアドレスを割り振っています。

設計として冗長構成にするために、2本の`DNS/HDCPサーバー`を用意していますが、構築手順としてはこの手順の中では触れていません。
また`DHCPサーバー`は、`ノード`のセットアップ時にのみ使用されるため、大半の要件であれば、冗長化の必要は無いでしょう。

## 3-3. ネットワークの設計

ここでは4つのネットワークを定義しています。

1) `Intra Network(社内ネットワーク)`: 192.168.124.0/24
2) `Inter-Node Network(ノード間)ネットワーク`:172.16.0.0/24
3) `Cluster Network`:10.128.0.0/14
4) `Service Network`:172.30.0.0/16

1)は社内のネットワークの想定です。

2)は `Kubernetes` のノード間を結ぶためのネットワーク + 外部からのリクエストを受け付ける LBもこのネットワークに所属させました。ネットワーク構成の取り方はいろいろな構成がありうるのですが、ここでは「外向け」ロードバランサーと「内向け」ロードバランサーを一体化して構成サーバーを減らす事に重点を置いて、このような構成を取りました。

3)と4)は `OpenShift` の内部で使用されている、仮想ネットワークです。3)と4)に関しては、Red Hat社が提供するマニュアルのサンプルで用いられているレンジをそのまま使用しています。マニュアルとの比較のしやすさも考えると、他と衝突していない限りこのまま使用するのをお勧めします。

### 3-3-1. Firewallの設計

この環境では、`Inter-Node Network`と名付けたネットワークに全ての関連コンポーネントをつなぐ事にしたので、`OpenShift`関連の通信に`Firewall`が挟まるような構成になっておらず、特にコンポーネント間の`Firewall`に関する実験はしていません。

一方で、`OpenShift`のコンポーネント間の通信で使用するポートについては以下に詳細があります。

参考：<a href="https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-bare-metal.html#installation-network-user-infra_installing-bare-metal">Installing a cluster on bare metal - Installing on bare metal | Installing | OpenShift Container Platform 4.5</a>

### 3-3-2. IPアドレスのレンジについて

1)～4)は、どれもプライベートアドレスであるはずなので、グローバルIPと被っていると予期せぬ動作を引き起こす可能性があります。

プライベートIPアドレスとして使用できる範囲はRFC 1918で規定されているので、以下から取得されていて、社内の他のネットワークとも1)～4)で使用するレンジが重複してない事を確認します。

- クラスA: `10.0.0.0`～`10.255.255.255/8` 
- クラスB: `172.16.0.0`～`172.31.255.255/12` 
- クラスC: `192.168.0.0`～`192.168.255.255/16` 

**参考** :<a href="https://ja.wikipedia.org/wiki/%E3%83%97%E3%83%A9%E3%82%A4%E3%83%99%E3%83%BC%E3%83%88%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF">プライベートネットワーク</a>

### 3-3-3. ルーティングテーブルの設定

この手順では、クラスターの`ノード間ネットワーク`(`Kubernetes`の各ノード間をつなぐネットワークをそう名付けます)は、プライベートアドレスレンジである`172.16.0.0/24`としています。

`172.16.0.0/24`から、インターネットにアクセスできる必要がありますが、プライベートのアドレスであるため、そのままでは、このネットワークからパケットが外に出る事はできても、社内ネットワークまでは戻ってくる事ができますが、その後の帰り道がわかりません。

そのため、社内ネットワークに戻ってきた際に、`172.16.0.0/24`のネットワークへのルーティングを教えてあげる必要があります。

ここでは、社内と社外をつなぐ `FW Router`で以下のコマンドを実行して、ルーティングを追加しています。

```
route add -net 172.16.0.0 netmask 255.255.255.0 gw 192.168.124.254  eno1.342  
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/9bc34987-cd99-c9a7-9c61-4b9bca0c547a.png)


### 3-3-4. OpenShiftのDNS要件

以下の6つのドメイン名が`OpenShift`ととして必要になります。

| ドメイン名                               | この手順の名前                       |   用途         |   
|:----------------------------------------|:------------------------------------|:--------------|
| api.\<cluster_name\>.\<base_domain\>.        | api.ocp45.example.localdomain.     |  Kubernetes API |
| api-int.\<cluster_name\>.\<base_domain\>.    | api-int.ocp45.example.localdomain. |Kubernetes API |
| *.app.\<cluster_name\>.\<base_domain\>.   | *.ocp45.example.localdomain.       | Routes |

この手順書では、
- \<cluster_name\> = `ocp45`
- \<base_domain\> = `example.localdomain` 

としました。これらの値は、後で`DNS`に登録していきます。

また、この値は、インストール用の `yaml`のマニフェストファイルを作成する時にもう一度使うので覚えておきます。

ここまでで、環境の基本設計は出そろったので、図にまとめると以下のようになります(再掲)。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/0c801714-ac38-c720-f85a-49bc7c607e06.png)
薄い灰色になっている`HA Proxy`と`DNS BIND`の冗長化構成の部分は、この手順では取り扱っていません。

## 3-4. ストレージの設計

ユーザーアプリケーションだけでなく、`Elasticsearch`や、`Container Registry` を構成するには、`PV(Persistent Volume)`を必要とします。`Prometheus`については、`PV`がなくてもインストールは可能(`OpenShift`をインストールすると自動で一緒にインストールされています)ですが、実際の運用を考えた場合、永続データ保管のための`PV`は必要だと思います。

ストレージの選択は、ユーザー環境によって様々だと思います。
この設計では、分散ストレージである`OCS(OpenShift Container Storage)`を使用する事にします。

ストレージ部分は、アーキテクチャーとしては疎結合なので、`OCS`の構築以外の部分の手順については、他のストレージを使用した場合もほぼ共通になります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/1bbb566e-e936-c7c5-fe85-f1381c71540b.png)

### 3-4-1. OCSノードの設計
`OCSノード`は3ノード以上が必要なため、本構成では3台で構成します。

`OCSノード`に必要となるリソースは、ドライブが増えるごとに追加でリソースが必要になることに注意が必要です。[OCSノードの最小リソース要件](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.5/html-single/planning_your_deployment/index#resource-requirements_rhocs) のリンクに詳細が書かれていますが、必要最低限のリソースだと`OCS`が完全に稼働しないことが見られたため、今回は余裕を持たせた構成としています。

また、`OCS`では、3ノードにまたがって三重でレプリケーションして冗長化するため、3ノード全てに`PV`として使用する容量分のドライブを搭載して下さい。

---

## → Next: [4. クラスターインストールの実行](../4_Installation/README.md)