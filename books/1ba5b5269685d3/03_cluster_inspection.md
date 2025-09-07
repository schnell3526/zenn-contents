---
title: "クラスタ状態確認演習"
---

# クラスタ状態確認演習

この章では、前章で学んだKubernetesアーキテクチャの理論を実際のkindクラスタで確認します。kubectlコマンドとDockerコマンドを組み合わせて、各コンポーネントの動作状況を詳しく観察していきます。

## 目標

- kindクラスタの構成要素を実際に確認する
- Control PlaneとWorker Nodeの各コンポーネントを識別する
- Kubernetesアーキテクチャの理解を実践を通じて深める
- トラブルシューティングに必要な基本的な調査スキルを身につける

## 前提条件

- 第1章でkindクラスタが作成済み
- kubectl、kind、dockerコマンドが使用可能
- kindクラスタが動作中

:::message
この章を始める前に、kindクラスタが正常に動作していることを確認してください。
:::

## kindクラスタの準備

まずはkindクラスタが正常に動作していることを確認しましょう。

### クラスタの存在確認

```bash
# kindクラスタ一覧表示
kind get clusters

# クラスタがない場合は作成
kind create cluster --name k8s-training
```

### 複数ノードクラスタの推奨設定

学習効果を高めるため、複数ノード構成での実行を推奨します：

```yaml
# kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

```bash
# 既存クラスタがあれば削除
kind delete cluster --name k8s-training

# 複数ノードクラスタを作成
kind create cluster --name k8s-training --config kind-multi-node.yaml
```

## 演習1: クラスタ全体の概要確認

### 1.1 基本的なクラスタ情報

```bash
# クラスタ情報の表示
kubectl cluster-info --context kind-k8s-training

# APIサーバーのバージョン情報
kubectl version --context kind-k8s-training

# クラスタの健全性確認
kubectl get componentstatuses --context kind-k8s-training
```

**確認ポイント**:
- API ServerのエンドポイントURL（kindでは`https://127.0.0.1:ポート番号`）
- クライアント（kubectl）とサーバー（API Server）のKubernetesバージョン
- Control Planeコンポーネントの状態（Healthy/Unhealthy）

### 1.2 kindクラスタのDockerコンテナ確認

```bash
# kindクラスタのDockerコンテナ一覧
docker ps --filter "label=io.x-k8s.kind.cluster=k8s-training"

# より詳細な情報表示
docker ps --filter "label=io.x-k8s.kind.cluster=k8s-training" --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

# 各コンテナのリソース使用状況
docker stats --no-stream $(docker ps --filter "label=io.x-k8s.kind.cluster=k8s-training" -q)
```

**確認ポイント**:
- Control Planeノード（k8s-training-control-plane）の存在
- Worker Nodeコンテナ（k8s-training-worker、k8s-training-worker2）の存在
- 各コンテナが使用しているKubernetes Nodeイメージ
- ポートフォワーディングの設定（6443ポートなど）

### 1.3 API Serverへのアクセス確認

```bash
# kubectlでのAPI Server接続確認
kubectl get --raw /healthz --context kind-k8s-training

# API Serverが公開するAPIグループ確認
kubectl api-resources --context kind-k8s-training | head -20

# 利用可能なAPIバージョン確認
kubectl api-versions --context kind-k8s-training | head -10
```

**確認ポイント**:
- API Serverの健全性（`ok`が返される）
- 利用可能なKubernetesリソース種別
- 各リソースが属するAPIグループとバージョン

## 演習2: ノード情報の詳細確認

### 2.1 ノードの基本情報

```bash
# ノード一覧表示
kubectl get nodes --context kind-k8s-training

# より詳細な情報表示
kubectl get nodes -o wide --context kind-k8s-training

# 特定ノードの詳細情報
kubectl describe node k8s-training-control-plane --context kind-k8s-training
```

**確認ポイント**:
- ノードの役割（Control Plane/Worker）の識別
- 各ノードのInternal-IP（Dockerコンテナ内のIP）
- Container Runtimeの種類とバージョン（containerd）
- 各ノードのCapacity（CPU、メモリ、Pod数の上限）

### 2.2 ノードラベルとTaintの確認

```bash
# ノードのラベル確認
kubectl get nodes --show-labels --context kind-k8s-training

# Control Planeノードのtaint確認
kubectl describe node k8s-training-control-plane --context kind-k8s-training | grep -A5 "Taints:"

# 特定ラベルでのノード選択
kubectl get nodes -l node-role.kubernetes.io/control-plane --context kind-k8s-training
```

**確認ポイント**:
- `node-role.kubernetes.io/control-plane`ラベルの存在
- Control Planeノードのtaint設定
- ノードの属性（OS、アーキテクチャ、Kubernetesバージョンなど）

### 2.3 kindコンテナ内部のプロセス確認

```bash
# Control Planeコンテナ内のプロセス確認
docker exec k8s-training-control-plane ps aux | grep -E "(kube-|etcd)"

# Worker Nodeコンテナ内のプロセス確認
docker exec k8s-training-worker ps aux | grep -E "(kubelet|containerd)"

# Control Planeコンテナ内のKubernetes構成ファイル確認
docker exec k8s-training-control-plane ls -la /etc/kubernetes/
```

**確認ポイント**:
- Control Planeでの4つの主要プロセス（etcd、kube-apiserver、kube-controller-manager、kube-scheduler）
- Worker Nodeでのkubelet、containerdプロセス
- `/etc/kubernetes/`ディレクトリ内の設定ファイル構造

## 演習3: システムPodの確認

### 3.1 kube-system名前空間のPod確認

```bash
# kube-system名前空間のPod一覧
kubectl get pods -n kube-system --context kind-k8s-training

# より詳細な情報（どのノードで動作しているか）
kubectl get pods -n kube-system -o wide --context kind-k8s-training

# Control Planeコンポーネントの確認
kubectl get pods -n kube-system -o wide --context kind-k8s-training | grep -E "(etcd|apiserver|controller|scheduler)"
```

**確認ポイント**:
- Control Planeコンポーネントが静的Podとして動作
- すべてのControl PlaneコンポーネントがControl Planeノードで実行
- Pod名にノード名が含まれている（例：`kube-apiserver-k8s-training-control-plane`）

### 3.2 DaemonSetの確認

```bash
# DaemonSet一覧確認
kubectl get daemonsets -n kube-system --context kind-k8s-training

# kube-proxyの配置確認
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide --context kind-k8s-training

# kindnetの確認（kindのCNI）
kubectl get pods -n kube-system -l app=kindnet -o wide --context kind-k8s-training
```

**確認ポイント**:
- kube-proxyがすべてのノードで動作
- kindnet（CNIプラグイン）がすべてのノードで動作
- DaemonSetによる各ノードへの自動配置

### 3.3 静的Podマニフェストの確認

```bash
# Control Planeコンテナ内の静的Podマニフェスト確認
docker exec k8s-training-control-plane ls -la /etc/kubernetes/manifests/

# API Serverの静的Podマニフェスト内容確認
docker exec k8s-training-control-plane cat /etc/kubernetes/manifests/kube-apiserver.yaml | head -20

# etcdの静的Podマニフェスト確認
docker exec k8s-training-control-plane cat /etc/kubernetes/manifests/etcd.yaml | grep -A5 -B5 "image:"
```

**確認ポイント**:
- 4つの静的Podマニフェストファイルの存在
- 各マニフェストファイルの基本構造
- 使用されているコンテナイメージ

## 演習4: コンポーネントログの確認

### 4.1 Control Planeコンポーネントのログ

```bash
# API Serverログ確認
kubectl logs -n kube-system kube-apiserver-k8s-training-control-plane --tail=20 --context kind-k8s-training

# etcdログ確認
kubectl logs -n kube-system etcd-k8s-training-control-plane --tail=10 --context kind-k8s-training

# Controller Managerログ確認
kubectl logs -n kube-system kube-controller-manager-k8s-training-control-plane --tail=10 --context kind-k8s-training

# Schedulerログ確認
kubectl logs -n kube-system kube-scheduler-k8s-training-control-plane --tail=10 --context kind-k8s-training
```

**確認ポイント**:
- 各コンポーネントが正常に動作している証拠
- エラーメッセージや警告の有無
- コンポーネント間の通信状況

### 4.2 Worker Nodeコンポーネントのログ

```bash
# kube-proxyログ確認
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=10 --context kind-k8s-training

# CNI（kindnet）ログ確認
kubectl logs -n kube-system -l app=kindnet --tail=10 --context kind-k8s-training

# kindコンテナのDocker ログも確認可能
docker logs k8s-training-worker --tail=20
```

**確認ポイント**:
- kube-proxyのiptables/IPVSルール管理状況
- CNIプラグインのネットワーク設定状況
- kubeletとContainer Runtimeの連携状況

### 4.3 イベントログの確認

```bash
# クラスタ全体のイベント確認
kubectl get events --sort-by='.metadata.creationTimestamp' --context kind-k8s-training

# 特定名前空間のイベント
kubectl get events -n kube-system --sort-by='.metadata.creationTimestamp' --context kind-k8s-training

# 最近のイベントのみ表示
kubectl get events --sort-by='.metadata.creationTimestamp' --context kind-k8s-training | tail -10
```

**確認ポイント**:
- クラスタ起動時のコンポーネント初期化順序
- Pod作成・削除に関するイベント
- エラーや警告イベントの有無

## 演習5: ネットワークとストレージの確認

### 5.1 ネットワーク設定の確認

```bash
# kindのDockerネットワーク確認
docker network ls | grep kind

# kindネットワークの詳細
docker network inspect kind

# Control Planeコンテナのネットワーク設定確認
docker exec k8s-training-control-plane ip addr show

# Worker Nodeのネットワーク設定確認
docker exec k8s-training-worker ip addr show
```

**確認ポイント**:
- kindクラスタ専用のDockerネットワーク
- 各ノードコンテナのIPアドレス
- クラスタ内通信用のネットワーク設定

### 5.2 Podネットワークの確認

```bash
# テスト用Podを作成
kubectl run test-pod --image=nginx --context kind-k8s-training

# PodのIP確認
kubectl get pods test-pod -o wide --context kind-k8s-training

# Pod内からのネットワーク確認
kubectl exec test-pod -- ip addr show --context kind-k8s-training

# テストPod削除
kubectl delete pod test-pod --context kind-k8s-training
```

**確認ポイント**:
- PodにアサインされるクラスタIPアドレス
- CNIによるPodネットワーク設定
- Podから外部ネットワークへのアクセス

### 5.3 ストレージの確認

```bash
# kindノードのストレージマウント確認
docker exec k8s-training-control-plane df -h

# Kubernetesデータディレクトリ
docker exec k8s-training-control-plane ls -la /var/lib/kubelet/

# etcdデータディレクトリ
docker exec k8s-training-control-plane ls -la /var/lib/etcd/

# containerdデータディレクトリ
docker exec k8s-training-worker ls -la /var/lib/containerd/
```

**確認ポイント**:
- 各コンポーネントのデータ保存場所
- ディスク使用量の確認
- kindにおけるストレージの制約

## 演習問題

### 基礎レベル

1. **ノード数の確認**  
   現在のkindクラスタにはControl PlaneノードとWorker Nodeがそれぞれ何台ありますか？

2. **CNIプラグインの特定**  
   このkindクラスタで使用されているCNIプラグインは何ですか？

3. **システムPod数**  
   kube-system名前空間で動作しているPod数を数えてください。

4. **Container Runtime**  
   Worker Nodeで使用されているContainer Runtimeは何ですか？

### 中級レベル

5. **API Server エンドポイント**  
   kindクラスタのAPI ServerのエンドポイントURL（外部からアクセス用）を特定してください。

6. **静的Pod確認**  
   Control Planeで動作している静的Podの名前をすべて挙げてください。

7. **taint設定**  
   Control Planeノードに設定されているtaintの内容を確認してください。

8. **プロセス確認**  
   Worker Nodeコンテナ内でkubeletプロセスが動作していることを確認してください。

### 上級レベル

9. **ネットワーク構成**  
   kindクラスタが使用しているDockerネットワークの詳細（サブネット、ゲートウェイなど）を調べてください。

10. **証明書の場所**  
    Control PlaneコンテナのKubernetes証明書がどこに保存されているか確認してください。

11. **Leader Election**  
    Controller ManagerとSchedulerのLeader Election状況を確認してください。

12. **etcdデータ**  
    etcdに保存されているデータの一部を確認してください（セキュリティに注意）。

## 解答例とヒント

<details>
<summary>解答例を表示</summary>

### 基礎レベル解答

```bash
# 1. ノード数確認
kubectl get nodes --context kind-k8s-training
kubectl get nodes -l node-role.kubernetes.io/control-plane --context kind-k8s-training
kubectl get nodes -l '!node-role.kubernetes.io/control-plane' --context kind-k8s-training

# 2. CNIプラグイン確認
kubectl get pods -n kube-system -l app=kindnet --context kind-k8s-training

# 3. システムPod数
kubectl get pods -n kube-system --no-headers --context kind-k8s-training | wc -l

# 4. Container Runtime確認
kubectl get nodes -o wide --context kind-k8s-training
# またはWorker Nodeコンテナ内で
docker exec k8s-training-worker crictl version
```

### 中級レベル解答

```bash
# 5. API Server エンドポイント
kubectl cluster-info --context kind-k8s-training

# 6. 静的Pod確認
kubectl get pods -n kube-system --context kind-k8s-training | grep k8s-training-control-plane

# 7. taint設定確認
kubectl describe node k8s-training-control-plane --context kind-k8s-training | grep -A5 "Taints:"

# 8. kubeletプロセス確認
docker exec k8s-training-worker ps aux | grep kubelet
```

### 上級レベル解答

```bash
# 9. Dockerネットワーク詳細
docker network inspect kind

# 10. 証明書の場所
docker exec k8s-training-control-plane ls -la /etc/kubernetes/pki/

# 11. Leader Election確認
kubectl get endpoints -n kube-system kube-controller-manager --context kind-k8s-training
kubectl get endpoints -n kube-system kube-scheduler --context kind-k8s-training

# 12. etcdデータ確認（注意して実行）
docker exec k8s-training-control-plane etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get --prefix --keys-only / | head -10
```

</details>

## トラブルシューティングのコツ

### 情報収集の手順

1. **全体像の把握**: `kubectl cluster-info`, `kubectl get nodes`
2. **システムPodの状態**: `kubectl get pods -n kube-system`
3. **イベントの確認**: `kubectl get events`
4. **ログの確認**: `kubectl logs -n kube-system <pod-name>`
5. **Dockerコンテナ状態**: `docker ps`, `docker logs`

### よくある問題と確認ポイント

- **API Serverにアクセスできない**: ポートフォワーディング、証明書の問題
- **Podが起動しない**: イメージプル、リソース不足、スケジューリング問題
- **ネットワーク接続できない**: CNI、kube-proxy、iptablesの問題
- **パフォーマンスが悪い**: リソース不足、ディスクI/O、ネットワーク遅延

## まとめ

この章では、kindクラスタを使用してKubernetesアーキテクチャの実際の動作を確認しました。

### 学習したポイント
- **クラスタ構成**: Control PlaneとWorker Nodeの役割分担
- **コンポーネント確認**: 各コンポーネントの動作状況と設定
- **ログとイベント**: トラブルシューティングに必要な情報収集
- **kindの特徴**: 本番環境との違いと学習での活用方法

### 次の章へ

次の章では、意図的にコンポーネント障害を発生させ、各コンポーネントの役割と依存関係をより深く理解していきます。実際のトラブルシューティングスキルも身につけましょう。