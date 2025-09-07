---
title: "イントロダクション"
---

# Kuberntes実践入門

:::message
Docker やコンテナそのものに関して詳しい解説は行いません。
:::

Kubernetesを深く理解するためのおすすめ学習テーマをレベル別に紹介します！

## **Kubernetesマスタリー・ロードマップ**

### **基礎編（入門）**

**1. Kubernetes アーキテクチャと基本概念**
- Control Plane コンポーネント（API Server, etcd, Controller Manager, Scheduler）
- Worker Node コンポーネント（Kubelet, Container Runtime, kube-proxy）
- Kubernetes オブジェクトモデル
- 基本的な kubectl コマンドとトラブルシューティング

**2. コアワークロードリソース**
- Pod のライフサイクルと設計パターン
- Deployment と ReplicaSet の関係
- DaemonSet, StatefulSet の使い分け
- Job, CronJob によるバッチ処理
- 基本的なトラブルシューティング手法

### **初級～中級編**

**3. Service Discovery & Networking**
- Service types (ClusterIP, NodePort, LoadBalancer, ExternalName)
- kube-proxy の動作メカニズム（iptables/IPVS）
- DNS解決の仕組み (CoreDNS)
- EndpointSlices と Service の関係
- iptables/IPVS rules の変化観察

**4. Ingress と外部トラフィック管理**
- Ingress Controller の選択と設定（NGINX, Traefik, HAProxy）
- TLS/SSL 証明書管理（cert-manager）
- パスベース/ホストベースルーティング
- Rate limiting と WAF 統合

**5. Storage & Persistent Volumes**
- PV, PVC, StorageClass の関係
- Dynamic Provisioning の内部動作
- Container Storage Interface (CSI) ドライバー
- Volume のアクセスモード（RWO, ROX, RWX）
- データ永続化のベストプラクティス

**6. Configuration & Secrets Management**
- ConfigMap, Secret のマウント方法
- 環境変数 vs ボリュームマウント
- External Secrets Operator
- Sealed Secrets による GitOps セキュリティ
- 設定変更時の Pod 動作とローリングアップデート

**7. Package Management with Helm**
- Chart の構造と作成
- Values ファイルとテンプレート化
- Helm hooks とライフサイクル
- Chart リポジトリ管理
- Helmfile による環境管理

### **中級～上級編**

**8. Scheduler Deep Dive**
- スケジューリングアルゴリズムとフィルタリング/スコアリング
- Node Affinity, Pod Affinity/Anti-affinity
- Taints & Tolerations の実用例
- Pod Topology Spread Constraints
- Custom Scheduler の作成と拡張

**9. Controller Pattern & Operators**
- Built-in Controllers の動作原理
- Custom Resource Definitions (CRD) と API 拡張
- Operator Pattern の実装（Kubebuilder, Operator SDK）
- Controller の reconciliation loop
- Finalizers と削除処理

**10. Autoscaling & Resource Management**
- Horizontal Pod Autoscaler (HPA) と custom metrics
- Vertical Pod Autoscaler (VPA) の動作
- Cluster Autoscaler との連携
- Resource Quotas & LimitRanges
- Quality of Service (QoS) classes と Pod Priority

**11. Admission Control**
- Validating/Mutating Webhooks
- Pod Security Standards/Admission
- Open Policy Agent (OPA) integration
- Dynamic Admission Control の実装

### **上級編**

**12. Multi-tenancy & Security**
- Namespace isolation 戦略
- RBAC (Role-Based Access Control) の設計
- Network Policies の実装と検証
- Pod Security Standards (Restricted, Baseline, Privileged)
- Security scanning と compliance

**13. Service Mesh Integration**
- Istio/Linkerd のアーキテクチャ
- Traffic management（カナリア、A/Bテスト）
- mTLS と zero-trust networking
- Observability 統合
- Service Mesh のパフォーマンス影響

**14. Observability & Monitoring**
- Prometheus + Grafana スタック構築
- Custom metrics と ServiceMonitor
- Distributed tracing (Jaeger, Zipkin)
- Log aggregation (EFK/Loki スタック)
- Health checks & Probes の最適化
- AlertManager によるアラート設計

**15. Disaster Recovery & High Availability**
- etcd backup & restore 戦略
- Multi-master 構成
- Cross-cluster/Cross-region migration
- StatefulSet data backup 自動化
- Disaster Recovery テストの実施

### **実践的統合編**

**16. CI/CD Pipeline Integration**
- GitOps with ArgoCD/Flux v2
- Progressive Delivery (Flagger, Argo Rollouts)
- Image scanning と supply chain security
- Policy as Code (OPA, Kyverno)
- Rollback strategies と自動復旧

**17. Performance Tuning & Cost Optimization**
- Resource optimization と right-sizing
- Node utilization 分析
- Network performance チューニング
- Storage I/O optimization
- Spot instances と Karpenter
- FinOps practices と cost allocation

**18. Multi-cluster Management**
- Cluster Federation
- Service Mesh による multi-cluster
- Submariner によるクラスター間接続
- Centralized monitoring/logging
- GitOps による multi-cluster 管理

# kind クラスタセットアップガイド

## 前提条件

### 必要なソフトウェア
- **Docker**: コンテナランタイム
- **kubectl**: Kubernetes CLI ツール
- **kind**: Kubernetes IN Docker

### インストール手順

#### 1. Docker のインストール
```bash
# macOS (Homebrew)
brew install --cask docker

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker.io

# 起動確認
docker --version
```

#### 2. kubectl のインストール
```bash
# macOS (Homebrew)
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 確認
kubectl version --client
```

#### 3. kind のインストール
```bash
# macOS (Homebrew)
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# 確認
kind --version
```

## kindクラスタの作成

### 基本的なクラスタ作成

```bash
# シンプルなシングルノードクラスタ
kind create cluster --name my-cluster

# クラスタ確認
kubectl cluster-info --context kind-my-cluster
```

### カスタム設定でのクラスタ作成

```bash
# 設定ファイルを使用したクラスタ作成
kind create cluster --name k8s-training --config kind-cluster-config.yaml

# クラスタ一覧表示
kind get clusters

# ノード確認
kubectl get nodes --context kind-k8s-training
```

### 複数ノード構成の例

```yaml
# multi-node-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

```bash
# 複数ワーカーノードでクラスタ作成
kind create cluster --name multi-node --config multi-node-config.yaml
```

## よく使用するkindコマンド

### クラスタ管理

```bash
# クラスタ作成
kind create cluster --name <CLUSTER_NAME>

# クラスタ削除
kind delete cluster --name <CLUSTER_NAME>

# 全クラスタ一覧
kind get clusters

# クラスタの詳細情報
kind get kubeconfig --name <CLUSTER_NAME>
```

### ノード管理

```bash
# ノード一覧（Dockerコンテナとして）
docker ps --filter "label=io.x-k8s.kind.cluster=<CLUSTER_NAME>"

# ノード内でのコマンド実行
docker exec -it <CLUSTER_NAME>-control-plane bash
docker exec -it <CLUSTER_NAME>-worker bash
```

### イメージ管理

```bash
# ローカルのDockerイメージをkindクラスタに読み込み
docker build -t my-app:latest .
kind load docker-image my-app:latest --name <CLUSTER_NAME>

# 読み込まれたイメージの確認
docker exec -it <CLUSTER_NAME>-control-plane crictl images
```

## トラブルシューティング

### よくある問題と解決策

#### 1. クラスタ作成時のポート競合
```bash
# エラー例: port 6443 is already allocated
# 解決策: 既存のクラスタを確認・削除
kind get clusters
kind delete cluster --name <EXISTING_CLUSTER>
```

#### 2. kubectl コンテキストの問題
```bash
# 現在のコンテキスト確認
kubectl config current-context

# kindクラスタのコンテキストに切り替え
kubectl config use-context kind-<CLUSTER_NAME>
```

#### 3. イメージプルの問題
```bash
# kindクラスタ内でのイメージ確認
docker exec -it <CLUSTER_NAME>-control-plane crictl images

# 必要に応じてイメージを手動で読み込み
kind load docker-image <IMAGE_NAME> --name <CLUSTER_NAME>
```

#### 4. ネットワーク接続の問題
```bash
# kindクラスタのネットワーク確認
docker network ls | grep kind

# コンテナ間の通信確認
docker exec <CLUSTER_NAME>-control-plane ping <CLUSTER_NAME>-worker
```

### ログ確認

```bash
# kindクラスタのログ
kind export logs --name <CLUSTER_NAME> ./kind-logs

# 特定ノードのログ
docker logs <CLUSTER_NAME>-control-plane
docker logs <CLUSTER_NAME>-worker

# Kubernetesコンポーネントのログ
kubectl logs -n kube-system <POD_NAME> --context kind-<CLUSTER_NAME>
```

## パフォーマンス設定

### リソース制限の調整

```yaml
# kind-config.yaml でのリソース制限
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  # カスタムイメージまたは追加設定
  extraMounts:
  - hostPath: /tmp
    containerPath: /tmp
- role: worker
  # ワーカーノード固有の設定
```

### Docker設定の調整

```bash
# Docker Desktop でのリソース割り当て確認
# Docker Desktop > Preferences > Resources > Advanced

# コンテナのリソース使用状況確認
docker stats <CLUSTER_NAME>-control-plane <CLUSTER_NAME>-worker
```

## kindネットワーキングの理解

### なぜAPI ServerのIPを確認するのか？

この確認の目的は、**kindクラスター特有のネットワーク構成を理解する**ためです。

#### 学習目的

**1. 本番クラスターとの違いを理解**
- **本番**: API ServerはクラスターIP（例：10.0.1.100:6443）で直接アクセス
- **kind**: Dockerコンテナなので、外部アクセスにはポートフォワーディングが必要

**2. Kubernetesネットワークの基本概念**
```
外部クライアント → 127.0.0.1:6443 → Docker port mapping → 172.18.0.2:6443 (API Server)
```
この流れを理解することで：
- ポートフォワーディングの仕組み
- コンテナネットワークの概念
- Load BalancerやIngress の必要性

**3. トラブルシューティングスキル**
演習中によくある問題：
- 「kubectlが接続できない」→ ポートフォワーディング確認
- 「Pod同士が通信できない」→ 内部IP vs 外部IP の理解
- 「Serviceがアクセスできない」→ ネットワーク層の理解

**4. 実際の運用への準備**
本番環境では：
- API ServerはLoad Balancerの背後
- 複数のAPI Serverインスタンス
- 証明書とTLSの考慮

### API Server エンドポイントの確認手順

#### 基本確認コマンド

```bash
# 1. kubectlで見えるエンドポイント（外部アクセス用）
kubectl cluster-info --context kind-k8s-training | grep "control plane"
# 出力例: https://127.0.0.1:6443

# 2. Docker内部IP（コンテナ間通信用）
docker inspect k8s-training-control-plane --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# 出力例: 172.18.0.2

# 3. 内部からのアクセステスト
docker exec k8s-training-worker curl -k -s https://k8s-training-control-plane:6443/version

# 4. kubeconfigの実際のserver設定
kubectl config view --context kind-k8s-training --minify --output jsonpath='{.clusters[0].cluster.server}'
```

#### 詳細な確認スクリプト

```bash
#!/bin/bash
echo "=== kind Network Analysis ==="

echo "1. External endpoint (kubectl):"
kubectl cluster-info --context kind-k8s-training | grep "control plane"

echo -e "\n2. Docker internal IP:"
docker inspect k8s-training-control-plane --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

echo -e "\n3. Docker network details:"
docker inspect k8s-training-control-plane | jq '.[0].NetworkSettings.Networks'

echo -e "\n4. Internal access test:"
docker exec k8s-training-worker curl -k -s https://k8s-training-control-plane:6443/version | jq .gitVersion

echo -e "\n5. kubeconfig server setting:"
kubectl config view --context kind-k8s-training --minify --output jsonpath='{.clusters[0].cluster.server}'
```

#### 実演での学習ポイント

**よくある間違い例:**
```bash
# ❌ 内部IPで外部からアクセスしようとする
curl -k https://172.18.0.2:6443/version  # 失敗

# ✅ 正しい：適切なエンドポイントを使用
curl -k https://127.0.0.1:6443/version   # 成功
```

**ネットワーク層の理解:**
```bash
# 外部 → kind（ポートフォワード経由）
kubectl get nodes --context kind-k8s-training

# Docker内部ネットワーク（直接通信）
docker exec k8s-training-control-plane kubectl get nodes
```

### 本番環境との比較

| 項目                | kind環境                      | 本番環境                   |
| ------------------- | ----------------------------- | -------------------------- |
| API Server アクセス | 127.0.0.1:6443 (port forward) | Load Balancer IP:443       |
| 内部通信            | Docker bridge network         | Pod/Service network        |
| 高可用性            | 単一コンテナ                  | 複数API Serverインスタンス |
| 証明書              | 自己署名（学習用）            | 正規CA発行                 |
| ネットワーク分離    | Docker network                | VLAN/Subnet分離            |
| 外部アクセス        | localhost port mapping        | Ingress/LoadBalancer       |

## 本番との違いと注意点

### kindクラスタの制限
- シングルマシン上でのシミュレーション
- 永続化されないデータ（クラスタ削除で消失）
- 限定的なネットワーク機能
- ロードバランサーの動作が異なる

### 学習用途での活用
- ローカル開発・テスト環境として最適
- CI/CDパイプラインでの使用
- Kubernetesの概念学習に適している
- アップグレードやダウングレードが簡単

## 次のステップ

1. **基本操作の習得**: `kubectl` コマンドでクラスタ操作
2. **演習の実施**: 各演習ファイルでの実践学習
3. **高度な設定**: カスタムCNI、Ingress Controller等
4. **アプリケーションデプロイ**: 実際のワークロード実行
