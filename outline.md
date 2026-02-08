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
  - Ingress
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
  - Pod - Service - Pod
  - 外部 - Ingress - Service - Pod
  - Pod - 外部
- Kubernetesを構成する部品
- ネットワークを担当する部品
