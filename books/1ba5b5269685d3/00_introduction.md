---
title: "イントロダクション"
---

# Kubernetes実践入門

:::message
この本では、Docker やコンテナの基礎知識があることを前提としています。コンテナそのものの詳しい解説は行いません。
:::

この本では、Kubernetesを実践的に学ぶための演習と解説を提供します。理論だけでなく、実際に手を動かしながらKubernetesの深い理解を目指します。

## この本の目的

- **実践重視**: 理論だけでなく、実際の操作を通じてKubernetesを習得
- **体系的学習**: 基礎から応用まで段階的に学習できる構成
- **運用視点**: 本番環境での運用を見据えた実用的な知識の習得

## 学習の進め方

### 前提知識
- Docker の基本的な操作（コンテナの作成・実行・削除）
- Linux コマンドラインの基礎知識
- YAML ファイルの読み書き

### 学習環境
本書では **kind（Kubernetes in Docker）** を使用したローカル学習環境を構築します。kindは軽量で、学習目的に最適なKubernetes環境を提供します。

### 章構成

各章は以下の構成で進みます：

1. **概念説明**: 該当トピックの理論的背景
2. **実践演習**: 実際に手を動かす演習問題
3. **トラブルシューティング**: よくある問題とその解決方法
4. **ベストプラクティス**: 実運用での推奨事項

## Kubernetesマスタリー・ロードマップ

以下は、この本で扱う学習テーマをレベル別に整理したロードマップです。各章はこのロードマップに基づいて構成されています。

### 基礎編（入門レベル）

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

### 初級～中級編

**3. Service Discovery & Networking**
- Service types (ClusterIP, NodePort, LoadBalancer, ExternalName)
- kube-proxy の動作メカニズム（iptables/IPVS）
- DNS解決の仕組み (CoreDNS)
- EndpointSlices と Service の関係

**4. Ingress と外部トラフィック管理**
- Ingress Controller の選択と設定（NGINX, Traefik, HAProxy）
- TLS/SSL 証明書管理（cert-manager）
- パスベース/ホストベースルーティング

**5. Storage & Persistent Volumes**
- PV, PVC, StorageClass の関係
- Dynamic Provisioning の内部動作
- Container Storage Interface (CSI) ドライバー
- データ永続化のベストプラクティス

**6. Configuration & Secrets Management**
- ConfigMap, Secret のマウント方法
- 環境変数 vs ボリュームマウント
- External Secrets Operator
- GitOps セキュリティのベストプラクティス

**7. Package Management with Helm**
- Chart の構造と作成
- Values ファイルとテンプレート化
- Helm hooks とライフサイクル
- Chart リポジトリ管理

### 中級～上級編

**8. Scheduler Deep Dive**
- スケジューリングアルゴリズム
- Node Affinity, Pod Affinity/Anti-affinity
- Taints & Tolerations の実用例
- Pod Topology Spread Constraints

**9. Controller Pattern & Operators**
- Built-in Controllers の動作原理
- Custom Resource Definitions (CRD) と API 拡張
- Operator Pattern の実装
- Controller の reconciliation loop

**10. Autoscaling & Resource Management**
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
- Cluster Autoscaler との連携
- Resource Quotas & LimitRanges

**11. Admission Control**
- Validating/Mutating Webhooks
- Pod Security Standards
- Open Policy Agent (OPA) integration

### 上級編

**12. Multi-tenancy & Security**
- Namespace isolation 戦略
- RBAC (Role-Based Access Control) の設計
- Network Policies の実装と検証
- Pod Security Standards

**13. Service Mesh Integration**
- Istio/Linkerd のアーキテクチャ
- Traffic management（カナリア、A/Bテスト）
- mTLS と zero-trust networking
- Observability 統合

**14. Observability & Monitoring**
- Prometheus + Grafana スタック構築
- Custom metrics と ServiceMonitor
- Distributed tracing
- Log aggregation

**15. Disaster Recovery & High Availability**
- etcd backup & restore 戦略
- Multi-master 構成
- Cross-cluster migration
- StatefulSet data backup 自動化

## 次の章へ

次の章では、実際にkindクラスタのセットアップを行い、学習環境を構築します。その後、基本的なKubernetesアーキテクチャから学習を開始します。

各章の演習を通じて、段階的にKubernetesの理解を深めていきましょう。
