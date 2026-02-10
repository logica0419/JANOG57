# アウトライン

## 挨拶/イントロ

## Kubernetes、正直わからないですよね

- 効率の良いネットワークのためには、ネットワークエンジニアの皆さんが基盤と手を組む必要がある
- Kubernetesのネットワークを知っていただき、基盤チームと共に効率の良いネットワークを作っていきましょう

## Kubernetesを知る

<https://speakerdeck.com/logica0419/discovering-kubernetes> の、細かい/具体的な部分を除いてお話

- Kubernetesは怖く思われがち、ひいてはそのネットワークも敬遠されがち
  - しかし、基本はそんなに難しくない！
- 基本コンポーネント
  - Pod
  - Deployment
  - Service
  - Gateway API
- Kubernetesはコンテナだけど「コンテナだけ」じゃない
  - ワークロードの管理者、形態は関係ない
  - Reconciliation Loop
  - 宣言的API
- ネットワークの人が知っておくべきポイント
  - コンテナはネットワーク的にはただのNetwork Namespace
  - Network Namespace同士の疎通を通せば、Kubernetesのネットワークは完璧
  - ただし、宣言的/Reconciliation Loopの方法に準じて実装する必要がある

## Kubernetesとネットワーク

<https://docs.google.com/presentation/d/1Fva8ZnfZ9gEpYCJ387TUB5aAH7Ydp6UvX-UQNxDWrso/edit?usp=sharing> の90Pから

- Kubernetesを流れる通信
  - クラスタ内: Pod - Service - Pod
  - クラスタ外から: 外部 - Ingress - Service (- Pod)
  - クラスタ外へ: Pod - 外部
- Kubernetesを構成する部品
- ネットワークを担当する部品
  - L2/L3ネットワーク: CNI
  - L4ロードバランサー: kube-proxy
  - Ingress通信用 L7LB: Gateway API Controller

## 前提: Linuxのネットワークスタック

- Network Namespace: L1
  - 一つのハードウェアの上に、(ネットワーク的に)仮想的なマシンを複数置ける
  - (ネットワーク的な)コンテナの実態
  - そもそもホストのネットワークもRoot Namespaceに置かれている
- ip-link: L2/L3
  - 特にコンテナナットワーキングでは、Namespaceや他のホストとの繋がりを作る役割を担う
  - Virtual Ethernet
  - Linux Bridge
- ip-route: L3
  - ルーティングテーブルの管理
  - Linuxをルーターとして扱う
- netfilter: L4
  - L4ベースのパケットフィルタリング
  - NAT (SNAT/DNAT)
  - ユーザーからの窓口
    - 昔はiptables
    - 今はnftablesに移行が進む

## CNI: L2/L3ネットワーク

<https://docs.google.com/presentation/d/1Fva8ZnfZ9gEpYCJ387TUB5aAH7Ydp6UvX-UQNxDWrso/edit?usp=sharing> の104Pから

- CNIはKubernetesだけじゃない、コンテナのネットワークシステム
  - 余談: CNIは (実質) Kubernetesだけ
- CNIを構成する要素
  - CNI仕様
  - 基盤の要件定義
  - CNIプラグイン

### CNI仕様

- プラグインが満たすべきユーザーとのインターフェース
- 定められたCNIの操作方法

### 基盤の要件定義

- 基盤によって定められる、ネットワークが満たすべき要件
- The Kubernetes network model
- 要件さえ満たせれば何を使っても良い
  - CNIの仕組みはどんなネットワーク技術でも受け入れる広い受け皿

### CNIプラグインの実装

それぞれのプラグインのLife of a Packetを紹介

- Flannel
  - Linux Bridge
  - VXLANオーバーレイネットワーク
- Calico
  - ルーティングテーブル
  - BGPピアリング
- Cilium
  - eBPFプログラム
  - Tunnelモード / Nativeモード
- これらは OSS だから全部汎用
- CNIプラグイン自作・改造したいインセンティブ例 (色々)
- 実際に独自の CNIプラグインを使用している例
- Amazon VPC CNI
  - VPCのENIを直接割り当てる
  - ルーティングテーブル
  - AWS VPCの仕組みでノード間をルーティング
- GKE Dataplane V2
  - ノード内はCiliumベース
  - ノード間はGoogleの基盤ネットワークを利用
- Coil on Neko
  - ノード内はCiliumベース
  - ノード間は、Nekoのネットワークに対してBGPで持っているPodのIPを広報

Pod - 外部 の通信は、netfilterのSNAT (masquerade) で実装されることがほとんど

## L4ロードバランサー: kube-proxy

- Serviceの実態
  - 複数のPodに対して、L4ロードバランシングを提供
- やってることはただのDNAT
  - Serviceに属するPodの中からランダムに一つ選んで、宛先IPを書き換え
- 実装方式
  - これまで: デフォルトはiptables
  - これから: デフォルトがnftablesへ

## Ingress通信用 L7LB: Gateway API Controller

- 外部から受けるHTTPトラフィックの、L7ロードバランサー
  - Webアプリケーション用に作られた基盤なので、これがあらかじめ考慮されている
- Gateway APIとGateway API Controller
  - Gateway APIとしてKubernetesに設定を登録
  - Gateway API Controllerが設定を読み取り、実際にトラフィックをロードバランス
- Gateway API Controller
  - 一般的なHTTPリバースプロキシが、Kubernetesから設定を読み取る機能を備えた物
  - Kubernetesはデフォルトを提供していない
  - 代表例: NGINX Gateway Fabric, Envoy Gateway, AWS Load Balancer Controller, Google Kubernetes Engine

### 実装例の紹介

- NGINX Gateway Fabric (OSS/汎用型 代表)
  - 外から来たトラフィックをNGINXのPodに流し、そこからServiceに流す
  - Serviceから先はPod-to-Podの通信
- AWS Load Balancer Controller (ネットワーク基盤特化型 代表)
  - あらかじめ、全ての割り振り先ルート用の入り口 (NodePort Service) をノードに空けておく
  - 外から来たトラフィックはAWS ALBを通り、そこから上記割り振り先ごとの入り口に流す
  - Serviceから先はPod-to-Podの通信

## プラットフォームとの対話を

- 汎用的 ≠ それだけでいい
  - 見てきた通り、特にCNIやL7LBは、今持っている基盤に合わせるメリットが大きく、デカいプラットフォームでは既に行われている
  - 自分たちのネットワークの状態に合わせて、最適なものを選ぶ/作ることができる
- CNIがインターフェースになっているのは、クラウドベンダーが自社のネットワークに合わせて効率の良い処理基盤を組むため
  - オンプレ環境でネットワークが全部わかるなら、それに効率化するに越したことはない
- 今日取り扱った部品を「共通言語」にして、プラットフォームチームと対話を
  - 例えばCNI/L7LBの選定やカスタマイズ、果てには自作など

## まとめ

ネットワークとの「フラっと」な対話で共にスケールする基盤を！

---

話したい内容一覧

- CNI の責務と kube-proxy の責務
  - CNI
    - NAT なしの pod 間通信を成立させる
    - Pod の IPAM も担う
  - kube-proxy
    - Service (仮想のIP)→ Pod の IP への変換を実現させる
- CNI の責務
  - node 内 pod-to-pod
    - bridge型
    - routed型
    - eBPF datapath型
  - node 間 pod-to-pod
    - underlay routing 型
    - overlay 型
- 仕様の範疇ではないが追加で必要になるケースのある責務
  - network policy
    - デフォルトでは pod-to-pod に制限をかけないが、network policy リソースへの対応は CNI の責務になる
  - kube-proxy
    - 例えば cilium は CNI の実装に eBPF を用いるが、kube-proxy でも eBPF を用いる実装がある (衝突する)
    - cilium は kubernetes を取り巻くネットワークスタックを全て eBPF で高速化するために、kube-proxy の機能も eBPF で実装して取り込んだ
      - ≒ cilium は CNI ではなく Network Solution と名乗っている

- 既存の CNI (OSS) と CNI 自前用意の例
- 自前の CNI を用意したいインセンティブ
