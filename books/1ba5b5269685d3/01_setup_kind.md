---
title: "kindクラスタセットアップガイド"
---

この章では、Kubernetes の学習環境として **kind（Kubernetes IN Docker）** クラスタの構築方法を学びます。kind は軽量で学習目的に最適なローカル Kubernetes 環境を提供します。

# kindとは

**kind** は、Docker コンテナ内で Kubernetes クラスタを実行するツールです。Kubernetes の学習、開発、CI/CD テストに適した環境を簡単に構築できます。

## kindの特徴

- **軽量**: Dockerコンテナベースで高速起動
- **シンプル**: 複雑な設定不要でクラスタ構築可能
- **マルチノード**: 複数の Worker Node をシミュレーション
- **本格的**: 実際の Kubernetes API を使用
- **クリーンアップ**: 完全な削除が簡単

## なぜ kind を選ぶのか

| 比較項目       | kind | minikube   | kubeadm |
| -------------- | ---- | ---------- | ------- |
| セットアップ   | 簡単 | 簡単       | 複雑    |
| リソース使用量 | 軽量 | 中程度     | 重い    |
| マルチノード   | 対応 | 限定的     | 対応    |
| 学習用途       | 最適 | 適している | 本格的  |
| クリーンアップ | 簡単 | 簡単       | 手動    |

# 前提条件

## 必要なソフトウェア

以下のソフトウェアをインストールする必要があります：

1. **Docker**: コンテナランタイム
2. **kubectl**: Kubernetes CLI ツール
3. **kind**: Kubernetes IN Docker

## システム要件

- **メモリ**: 最低4GB、推奨8GB以上
- **ディスク容量**: 10GB以上の空き容量
- **OS**: macOS、Linux、Windows（WSL2）

# インストール手順

## 1. Dockerのインストール

### macOS（Homebrew使用）
```bash
# Docker Desktop をインストール
brew install --cask docker

# または公式サイトからダウンロード
# https://docs.docker.com/docker-for-mac/install/
```

### Ubuntu/Debian
```bash
# パッケージを更新
sudo apt-get update

# Dockerをインストール
sudo apt-get install docker.io

# ユーザーを docker グループに追加（要ログアウト・ログイン）
sudo usermod -aG docker $USER
```

### インストール確認
```bash
# Dockerバージョン確認
docker --version

# Dockerデーモン起動確認
docker info
```

## 2. kubectlのインストール

### macOS（Homebrew使用）
```bash
# kubectlをインストール
brew install kubectl
```

### Linux
```bash
# 最新バージョンを取得
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# インストール
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### インストール確認
```bash
# kubectlバージョン確認
kubectl version --client
```

## 3. kindのインストール

### macOS（Homebrew使用）
```bash
# kindをインストール
brew install kind
```

### Linux
```bash
# 最新バージョンの kind を取得（v0.20.0の例）
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

# 実行可能にする
chmod +x ./kind

# パスに配置
sudo mv ./kind /usr/local/bin/kind
```

### インストール確認
```bash
# kindバージョン確認
kind --version
```

# 基本的なクラスタ作成

## シンプルなシングルノードクラスタ

最も簡単なクラスタを作成します：

```bash
# 基本的なクラスタ作成
kind create cluster --name my-first-cluster

# 作成結果例:
# Creating cluster "my-first-cluster" ...
# ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
# ✓ Preparing nodes 📦
# ✓ Writing configuration 📜
# ✓ Starting control-plane 🕹️
# ✓ Installing CNI 🔌
# ✓ Installing StorageClass 💾
# Set kubectl context to "kind-my-first-cluster"
```

## クラスタ接続の確認

```bash
# クラスタ情報表示
kubectl cluster-info --context kind-my-first-cluster

# ノード一覧表示
kubectl get nodes

# 出力例:
# NAME                           STATUS   ROLES           AGE   VERSION
# my-first-cluster-control-plane Ready    control-plane   2m    v1.27.3
```

## 基本的な動作確認

```bash
# システム Pod 一覧
kubectl get pods -A

# 名前空間一覧
kubectl get namespaces

# クラスタイベント確認
kubectl get events
```

# カスタム設定でのクラスタ作成

## 複数ノード構成

複数の Worker Node を含むクラスタを作成します：

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
# 設定ファイルを使用したクラスタ作成
kind create cluster --name multi-node --config multi-node-config.yaml

# ノード確認
kubectl get nodes
# 4つのノード（1 control-plane + 3 worker）が表示される
```

## ポートマッピング設定

外部からクラスタにアクセスするための設定：

```yaml
# port-mapping-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
```

```bash
# ポートマッピング付きクラスタ作成
kind create cluster --name port-mapping --config port-mapping-config.yaml
```

## カスタムイメージとマウント

```yaml
# advanced-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.27.3
  extraMounts:
  - hostPath: /tmp
    containerPath: /tmp
- role: worker
  extraMounts:
  - hostPath: /var/log
    containerPath: /var/log
    readOnly: true
```

# 実践演習

## 演習1: 最初のクラスタ作成

```bash
# ステップ1: クラスタ作成
kind create cluster --name k8s-training

# ステップ2: 接続確認
kubectl cluster-info --context kind-k8s-training

# ステップ3: ノード詳細確認
kubectl describe node k8s-training-control-plane
```

## 演習2: Podのデプロイテスト

```bash
# テスト用 Pod デプロイ
kubectl run test-pod --image=nginx:latest

# Pod状態確認
kubectl get pods

# Pod詳細確認
kubectl describe pod test-pod

# Podログ確認
kubectl logs test-pod

# Pod削除
kubectl delete pod test-pod
```

## 演習3: マルチノードクラスタでの分散確認

```bash
# マルチノードクラスタの作成
kind create cluster --name multi-node --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

# 複数 Pod デプロイ
kubectl create deployment nginx-deployment --image=nginx --replicas=3 --context kind-multi-node

# Pod配置確認（ノード分散を確認）
kubectl get pods -o wide --context kind-multi-node

# デプロイメント削除
kubectl delete deployment nginx-deployment --context kind-multi-node
```

終わったらリソースをクリーンナップしてください

```bash
kind delete cluster --name k8s-training
kind delete cluster --name multi-node
```

# よく使用する kind コマンド

## クラスタ管理

```bash
# クラスタ作成
kind create cluster --name <CLUSTER_NAME>

# クラスタ作成（設定ファイル使用）
kind create cluster --name <CLUSTER_NAME> --config <CONFIG_FILE>

# クラスタ削除
kind delete cluster --name <CLUSTER_NAME>

# 全クラスタ削除
kind delete clusters --all

# クラスタ一覧表示
kind get clusters

# クラスタ設定出力
kind get kubeconfig --name <CLUSTER_NAME>
```

## ノード操作

```bash
# ノード一覧（Dockerコンテナとして）
docker ps --filter "label=io.x-k8s.kind.cluster=<CLUSTER_NAME>"

# ノード内でのコマンド実行
docker exec -it <CLUSTER_NAME>-control-plane bash
docker exec -it <CLUSTER_NAME>-worker bash

# ノードリソース確認
docker stats <CLUSTER_NAME>-control-plane
```

## イメージ管理

```bash
# ローカル Docker イメージを kind クラスタに読み込み
docker build -t my-app:latest .
kind load docker-image my-app:latest --name <CLUSTER_NAME>

# 読み込まれたイメージ確認
docker exec -it <CLUSTER_NAME>-control-plane crictl images | grep my-app
```

# トラブルシューティング

## よくある問題と解決策

### 1. クラスタ作成時のポート競合

**問題**:
```
ERROR: failed to create cluster: node(s) already exist for a cluster with the name "kind"
```

**解決策**:

同じ名前のクラスタがすでに作成されているので、既存のクラスタの名前を確認して削除するか異なる名前で作成します。

```bash
# 既存クラスタの確認
kind get clusters

# 競合するクラスタの削除
kind delete cluster --name <EXISTING_CLUSTER>

# または異なる名前で作成
kind create cluster --name my-new-cluster
```

### 2. kubectl コンテキストの問題

**問題**: `kubectl` コマンドが正しいクラスタに接続できない

**解決策**:

`kubectl` コマンドのコンテキストが間違っている可能性があるので正しいものを指定します。

```bash
# 現在のコンテキスト確認
kubectl config current-context

# kindクラスタのコンテキスト一覧
kubectl config get-contexts | grep kind

# 正しいコンテキストに切り替え
kubectl config use-context kind-<CLUSTER_NAME>
```

### 3. イメージプルの問題

**問題**: Pod のイメージが Pull できない

**解決策**:
```bash
# kindクラスタ内のイメージ確認
docker exec -it <CLUSTER_NAME>-control-plane crictl images

# 必要なイメージを手動で読み込み
docker pull <IMAGE_NAME>
kind load docker-image <IMAGE_NAME> --name <CLUSTER_NAME>
```

### 4. DNS 解決の問題

**問題**: Pod内からの名前解決が失敗する

**解決策**:
```bash
# CoreDNS Pod状態確認
kubectl get pods -n kube-system | grep coredns

# CoreDNS ログ確認
kubectl logs -n kube-system deployment/coredns

# DNS設定確認
kubectl get configmap coredns -n kube-system -o yaml
```

## ログとデバッグ

### クラスタログの確認

```bash
# kindクラスタ全体のログをエクスポート
kind export logs --name <CLUSTER_NAME> ./kind-logs

# 特定ノードのログ
docker logs <CLUSTER_NAME>-control-plane
docker logs <CLUSTER_NAME>-worker

# Kubernetesコンポーネントのログ
kubectl logs -n kube-system <POD_NAME>
```

### リソース使用状況の確認

```bash
# コンテナリソース使用状況
docker stats

# kindクラスタの詳細情報
kubectl top nodes
kubectl top pods --all-namespaces
```

## パフォーマンス最適化

### Docker 設定の調整

```bash
# Docker Desktop リソース設定を確認
# Docker Desktop > Preferences > Resources > Advanced

# 推奨設定:
# - Memory: 6GB以上
# - CPU: 4コア以上
# - Disk image size: 64GB以上
```

### kind クラスタのリソース制限

```yaml
# resource-limited-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: KubeletConfiguration
    maxPods: 50
    kubeReserved:
      cpu: 200m
      memory: 500Mi
```

# 本番環境との違い

## kind クラスタの制限事項

1. **シングルマシン**: 実際の分散環境ではない
2. **永続化**: クラスタ削除でデータも消失
3. **ネットワーク**: 限定的な LoadBalancer 機能
4. **スケール**: 大規模クラスタのシミュレーションには限界
5. **セキュリティ**: 本番レベルのセキュリティ設定なし

## 学習用途での活用方法

### 利点
- **軽量で高速**: 短時間でクラスタ構築・削除が可能
- **実験環境**: 設定を変更して何度でも試行可能
- **バージョンテスト**: 異なる Kubernetes バージョンでのテストが容易
- **CI/CD**: 自動テストパイプラインでの利用

### 推奨される使用方法
- Kubernetes の基本概念学習
- YAML マニフェストの動作テスト
- アプリケーションの開発・デバッグ
- CRD や Operator の開発

### 本番移行時の考慮事項
- 永続化ストレージの設計
- 高可用性構成の検討
- セキュリティポリシーの適用
- モニタリング・ログ管理の設定

# まとめ

この章では、kind を使用した Kubernetes 学習環境の構築方法を学習しました。

## 重要なポイント
- kind は学習に最適なローカル Kubernetes 環境
- Docker、kubectl、kind の正しいインストールが必要
- 設定ファイルでマルチノード構成も可能
- トラブルシューティングのコマンドを理解する
- 本番環境とは違う制限があることを理解する

## 次のステップ

次の章では、kind クラスタを使用して Kubernetes アーキテクチャの実際の動作を確認し、各コンポーネントの役割を詳しく学習します。

準備した kind クラスタで実際にコンポーネントの動作を観察することで、理論と実践を組み合わせた深い理解を獲得しましょう。
