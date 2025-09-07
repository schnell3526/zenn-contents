---
title: "Kubertes のアーキテクチャと基本概念"
---

# aaa

## aaa

## Kubernetes クラスタの全体構造

Kubernetes のクラスタは大きく分けて、Controle Plane と Worker Node に分かれる。

```mermaid
graph TB
    subgraph CP ["Control Plane"]
        direction TB
        API[API Server<br/>kube-apiserver]
        ETCD[etcd<br/>分散KVS]
        CM[Controller Manager<br/>kube-controller-manager]
        SCHED[Scheduler<br/>kube-scheduler]

        ETCD -.->|Read/Write| API
        CM -.->|Watch/Update| API
        SCHED -.->|Watch/Bind| API
    end

    subgraph WN ["Worker Nodes"]
        direction TB
        subgraph N1 ["Node 1"]
            KUBELET1[kubelet]
            PROXY1[kube-proxy]
            CRI1[Container Runtime<br/>containerd/Docker]
            POD1[Pod1]
            POD2[Pod2]
            POD3[Pod3]

            KUBELET1 -.->|Manage| POD1
            KUBELET1 -.->|Manage| POD2
            KUBELET1 -.->|Manage| POD3
            CRI1 -.->|Run| POD1
            CRI1 -.->|Run| POD2
            CRI1 -.->|Run| POD3
        end

        subgraph N2 ["Node 2"]
            KUBELET2[kubelet]
            PROXY2[kube-proxy]
            CRI2[Container Runtime]
            POD4[Pod4]
            POD5[Pod5]

            KUBELET2 -.->|Manage| POD4
            KUBELET2 -.->|Manage| POD5
            CRI2 -.->|Run| POD4
            CRI2 -.->|Run| POD5
        end
    end

    API -->|gRPC/HTTPS| KUBELET1
    API -->|gRPC/HTTPS| KUBELET2
    API -->|REST API| PROXY1
    API -->|REST API| PROXY2
```

### Controle Plane コンポーネント詳解

1. API Server ([kube-apiserver](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-apiserver))

- 役割: Kubernetesの中枢となるRESTful APIサーバー
- 主な機能
  - すべてのコンポーネント間の通信ハブ
  - 認証・認可・Admission Controlの実行
  - etcdへの唯一のアクセスポイント
  - リソースの検証と永続化

例えば、`kubectl get pods` を実行して pod の一覧を取得する場合、コンポーネント間では以下のようなやり取りが行われている。

```
kubectl get pods
→ HTTPS Request → API Server → etcd (データ取得)
→ Response → kubectl
```

2. etcd

- 役割: 分散型のKey-Valueストア
- 保存データ
  - クラスターの全状態（desired state）
  - ConfigMap、Secret、Service定義
  - RBAC設定、Network Policy
- 特徴
  - Raft合意アルゴリズムによる高可用性
  - デフォルトで3台以上の奇数構成を推奨

3. Controller Manager ([kube-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-controller-manager))

- 役割: 各種コントローラーを実行する管理プロセス
- 主要コントローラー
  - Deployment Controller: ReplicaSet の管理
  - ReplicaSet Controller: Pod 数の維持
  - Node Controller: ノードの監視
  - Service Account Controller: SA 作成
  - Endpoint Controller: Service/Pod 関連付け
- 動作原理: Control Loop (制御ループ)
```
現在の状態(Current State) → 期待する状態(Desired State)との差分検出
→ 差分を解消するアクション実行 → 繰り返し
```

4. Scheduler ([kube-scheduler](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-scheduler))

- 役割: Pod を適切な Worker Node に配置する
- 主な機能
  - 新しく作成された Pod を監視
  - リソース要求・制約条件を考慮したNode選択
  - Affinity/Anti-affinity ルールの適用
  - Taints & Tolerations の考慮
- スケジューリングプロセス
  1. **Filtering（フィルタリング）**: 条件を満たさないNodeを除外
  2. **Scoring（スコアリング）**: 残ったNodeに優先度をつけて最適なNodeを選択

```
Pod作成リクエスト → Scheduler → Node選択 → API Server → Kubelet(選択されたNode)
```

### Worker Node コンポーネント詳解

1. **kubelet**

- 役割: Node上で実行される主要なKubernetesエージェント
- 主な機能
  - API Serverと通信してPod仕様を取得
  - Container Runtimeに指示してコンテナを起動/停止
  - Podとコンテナの健全性監視
  - Node状態の報告 (CPU, メモリ, ディスク使用量等)
  - Volume マウント処理
- 動作例

```mermaid
sequenceDiagram
    participant API as API Server
    participant KB as kubelet
    participant CRI as Container Runtime

    API->>KB: Pod作成リクエスト
    KB->>CRI: コンテナ起動指示
    CRI->>KB: 起動完了通知
    KB->>API: Pod状態更新(Running)

    loop Health Check
        KB->>CRI: コンテナ状態確認
        KB->>API: 状態報告
    end
```

2. **Container Runtime**

- 役割: 実際にコンテナを実行するランタイム
- 主な実装
  - **containerd**: Docker社が開発、軽量で高性能
  - **CRI-O**: Red Hat主導、OCI準拠
  - **Docker Engine**: 従来のDockerデーモン (非推奨)
- 機能
  - イメージのプル（registry からのダウンロード）
  - コンテナの作成・起動・停止・削除
  - ネットワーク設定
  - ストレージマウント

3. **kube-proxy**

- 役割: Kubernetesネットワークの実装を担当
- 主な機能
  - Serviceの仮想IP（ClusterIP）を実現
  - 負荷分散（ServiceのEndpointsへのトラフィック分散）
  - ネットワークルールの管理（iptables / IPVS）
- 動作モード
  - **iptablesモード**: iptablesルールでトラフィック転送（デフォルト）
  - **IPVSモード**: IPVS（IP Virtual Server）を使用、高性能
  - **userspaceモード**: ユーザー空間でProxy実行（非推奨）

```mermaid
graph LR
    CLIENT[Client] -->|request| SVC[Service:80]
    SVC -->|iptables rule| EP1[Pod1:8080]
    SVC -->|iptables rule| EP2[Pod2:8080]
    SVC -->|iptables rule| EP3[Pod3:8080]

    subgraph "kube-proxy manages"
        SVC
    end
```

## Kubernetes オブジェクトモデル

Kubernetesは、すべてのリソースを「オブジェクト」として扱う宣言的システム。

### オブジェクトの基本構造

すべてのKubernetesオブジェクトは以下の構造を持つ：

```yaml
apiVersion: apps/v1        # APIのバージョン
kind: Deployment          # リソースの種類
metadata:                 # メタデータ（名前、ラベルなど）
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
spec:                    # 期待する状態（Desired State）
  replicas: 3
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
        image: nginx:1.21
status:                  # 現在の状態（Current State）※システム管理
  replicas: 3
  readyReplicas: 3
  updatedReplicas: 3
```

### 主要なフィールド説明

1. **apiVersion**
   - リソースが属するAPIグループとバージョン
   - 例: `v1`, `apps/v1`, `extensions/v1beta1`

2. **kind**
   - リソースタイプを指定
   - 例: `Pod`, `Service`, `Deployment`, `ConfigMap`

3. **metadata**
   - `name`: リソース名（namespace内で一意）
   - `namespace`: 所属するnamespace（省略時は`default`）
   - `labels`: キー/バリューペアのメタデータ（Selectorで使用）
   - `annotations`: 任意のメタデータ（ツールが使用）

4. **spec**
   - ユーザーが定義する「期待する状態」
   - リソースタイプごとに異なる構造

5. **status**
   - システムが管理する「現在の状態」
   - ユーザーは直接編集不可（Controller によって更新）

### Control Loop（制御ループ）

Kubernetesの核となる概念：

```mermaid
graph TB
    WATCH[Watch API]
    CURRENT[Current State<br/>status]
    DESIRED[Desired State<br/>spec]
    COMPARE{Compare}
    ACTION[Take Action]
    WAIT[Wait]

    WATCH --> CURRENT
    WATCH --> DESIRED
    CURRENT --> COMPARE
    DESIRED --> COMPARE
    COMPARE -->|Different| ACTION
    COMPARE -->|Same| WAIT
    ACTION --> WATCH
    WAIT --> WATCH
```

**制御ループの例（Deployment Controller）**:

1. Deployment の `spec.replicas: 3` を監視
2. 現在のPod数を確認 (status)
3. 期待値と現実値を比較
4. 差分があればReplicaSetを作成/更新してPod数を調整
5. 繰り返し

### よく使用するオブジェクトタイプ

| オブジェクト   | 役割                                        | 例                           |
| -------------- | ------------------------------------------- | ---------------------------- |
| **Pod**        | 最小実行単位（1つ以上のコンテナ）           | アプリケーションプロセス     |
| **Service**    | Pod群への安定したネットワークエンドポイント | ロードバランサー             |
| **Deployment** | Pod のレプリカ管理とローリングアップデート  | Webアプリケーション          |
| **ConfigMap**  | アプリケーション設定データ                  | 環境変数、設定ファイル       |
| **Secret**     | 機密データ（パスワード、証明書など）        | DB接続情報                   |
| **Namespace**  | リソースの論理的分離                        | 環境分離（dev/staging/prod） |

### ラベルとセレクター

オブジェクト間の関連付けに使用：

```yaml
# ラベル付きPod
metadata:
  labels:
    app: nginx
    version: v1.0
    tier: frontend

# Serviceでラベルセレクター使用
spec:
  selector:
    app: nginx      # app=nginx ラベルを持つPodを対象
    tier: frontend
```

## 学習リソース

### 演習・実践
- **[クラスタ状態確認演習](./exercises/01_cluster_inspection.md)**: kubectlを使ったクラスタ探索
- **[コンポーネント障害演習](./exercises/02_component_troubleshoot.md)**: トラブルシューティング実践
- **[オブジェクトモデル演習](./exercises/03_object_model_practice.md)**: 制御ループの体験

### 図解・アーキテクチャ
- **[詳細アーキテクチャ図](./diagrams/cluster_architecture.md)**: 完全なクラスタ構成図
- **[制御ループ図解](./diagrams/control_loop.md)**: Controller動作の詳細解説

### リファレンス
- **[kubectl基本コマンド集](./cheatsheet/kubectl_basics.md)**: 必須コマンドとTips集

### kind環境セットアップ
- **[kindクラスタセットアップガイド](./kind-setup.md)**: kindのインストールと設定
- **[kindクラスタ設定ファイル](./kind-cluster-config.yaml)**: カスタマイズされたクラスタ設定例

## 学習の進め方

1. **理解度確認**: まずREADME.mdの内容を読んで各コンポーネントの役割を理解
2. **実践演習**: `exercises/` 内の演習を順番に実施
3. **深堀り**: `diagrams/` の図解で動作原理を詳しく学習
4. **日常利用**: `cheatsheet/` を参照して実際のクラスタ操作に活用

## 学習目標の達成確認

**基礎編完了の目安**:
- [ ] Control Planeの4つのコンポーネント(etcd, API Server, Controller Manager, Scheduler)の役割を説明できる
- [ ] Worker Nodeの3つのコンポーネント(kubelet, Container Runtime, kube-proxy)の役割を説明できる
- [ ] Kubernetesオブジェクトの基本構造(apiVersion, kind, metadata, spec, status)を理解している
- [ ] kubectl基本コマンドでクラスタ状態を確認できる
- [ ] 制御ループ(Control Loop)の概念と動作を理解している

**確認方法**: 各演習の基礎レベル問題をすべて解答できること
