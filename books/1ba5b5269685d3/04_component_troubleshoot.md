---
title: "コンポーネント障害シナリオ演習"
---

# コンポーネント障害シナリオ演習

この章では、kindクラスタでKubernetesコンポーネントの障害を意図的に発生させ、各コンポーネントの役割と依存関係を深く理解します。実際のトラブルシューティングスキルも身につけましょう。

## 目標

- 各コンポーネントの障害時の影響範囲を理解する
- 障害の発生から復旧までの流れを観察する
- 実践的なトラブルシューティング手法を習得する
- kindクラスタでの安全な障害シミュレーション方法を学ぶ

## 重要な注意事項

:::warning
**この演習はkind学習用クラスタでのみ実施してください**

- 本番環境では絶対に実行しないでください
- kindクラスタは削除・再作成が簡単なため、学習に最適です
- 障害シミュレーションは学習目的でのみ行ってください
:::

## 前提条件

- 前章でkindクラスタの構成を確認済み
- 複数ノード構成のkindクラスタが動作中
- 基本的なkubectlとdockerコマンドを理解している

## kindクラスタの準備

演習用のマルチノードクラスタを準備します：

```bash
# 既存クラスタがあれば削除
kind delete cluster --name k8s-training

# 複数ノード構成で作成
kind create cluster --name k8s-training --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# クラスタ動作確認
kubectl get nodes --context kind-k8s-training
```

## 障害シナリオ1: Worker NodeのKubelet障害

### 背景知識

kubeletは各Worker Node上で動作し、以下の役割を担います：
- API ServerとのPod仕様の同期
- Container Runtimeへのコンテナ管理指示
- Podとコンテナの健全性監視
- Node状態のAPI Serverへの報告

### 事前準備

```bash
# テスト用Podを作成
kubectl run test-nginx --image=nginx --context kind-k8s-training
kubectl run test-busybox --image=busybox:latest --command --context kind-k8s-training -- sleep 3600

# Podが正常に配置されることを確認
kubectl get pods -o wide --context kind-k8s-training

# どのworkerノードにPodがあるか確認
POD_NODE=$(kubectl get pod test-nginx -o jsonpath='{.spec.nodeName}' --context kind-k8s-training)
echo "test-nginx is running on: $POD_NODE"
```

### 障害の発生

```bash
# Worker nodeコンテナ内でkubeletサービスを停止
docker exec k8s-training-worker systemctl stop kubelet

# 即座にノード状態を確認
kubectl get nodes --context kind-k8s-training
```

### 観察される現象

**段階1: 即座の変化**
```bash
# kubeletプロセスが停止していることを確認
docker exec k8s-training-worker systemctl status kubelet

# しかし、ノードはまだReady状態
kubectl get nodes --context kind-k8s-training
```

**段階2: 約40秒後（node-monitor-grace-period経過後）**
```bash
# ノードがNotReady状態に変化
kubectl get nodes --context kind-k8s-training

# NotReadyの詳細確認
kubectl describe node k8s-training-worker --context kind-k8s-training | grep -A10 "Conditions:"
```

**段階3: 約5分後（tolerationSeconds経過後）**
```bash
# 該当ノード上のPodがTerminating状態に
kubectl get pods -o wide --context kind-k8s-training

# DaemonSet PodはRunningのまま残る
kubectl get pods -n kube-system -o wide --context kind-k8s-training | grep k8s-training-worker
```

### 技術的解説

kubelet障害時の影響プロセス：

1. **ハートビート停止**: kubeletがAPI Serverへの定期報告（デフォルト10秒間隔）を停止
2. **Node Controller判定**: 40秒間応答がない場合、NodeをNotReady状態に変更
3. **Taint付与**: Node ControllerがTaintを自動付与
   ```
   node.kubernetes.io/unreachable:NoSchedule
   node.kubernetes.io/unreachable:NoExecute
   ```
4. **Taint-based Eviction**: Pod上のTolerationに基づき、300秒後にPodをTerminating状態に変更

### 調査手順

```bash
# 1. Node状態の詳細確認
kubectl describe node k8s-training-worker --context kind-k8s-training

# 2. Taintの確認
kubectl get node k8s-training-worker -o yaml --context kind-k8s-training | grep -A10 "taints:"

# 3. kubeletプロセス状態確認
docker exec k8s-training-worker ps aux | grep kubelet
docker exec k8s-training-worker systemctl status kubelet

# 4. イベント確認
kubectl get events --sort-by='.metadata.creationTimestamp' --context kind-k8s-training | grep worker
```

### 復旧手順

```bash
# kubeletサービスを再起動
docker exec k8s-training-worker systemctl restart kubelet

# 復旧確認（数十秒かかる場合があります）
kubectl get nodes --context kind-k8s-training

# Terminating状態だったPodの状況確認
kubectl get pods --context kind-k8s-training
```

**復旧後の重要な観察点**:
- Terminating状態のPodは削除される
- 新しいPodが別のノードまたは復旧したノードで起動
- DaemonSet PodはRunning状態を維持

## 障害シナリオ2: etcd障害

### 背景知識

etcdは、Kubernetesクラスタの全状態を保存する分散Key-Valueストアです：
- すべてのKubernetesオブジェクトの保存
- API Serverからの唯一のデータ源
- Raft合意アルゴリズムによる高可用性

### 事前準備

```bash
# システム状態の確認
kubectl get pods -A --context kind-k8s-training
kubectl get nodes --context kind-k8s-training

# テスト用リソースの作成
kubectl create deployment test-app --image=nginx --replicas=2 --context kind-k8s-training
```

### 障害の発生

```bash
# etcdの静的Podマニフェストを移動して停止
docker exec k8s-training-control-plane mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak

# etcdプロセスが停止することを確認
sleep 10
docker exec k8s-training-control-plane ps aux | grep etcd
```

### 観察される現象

**段階1: 即座の影響**
```bash
# すべてのkubectlコマンドがタイムアウト
kubectl get pods --context kind-k8s-training --request-timeout=10s
# Error: server was unable to return a response in the time allotted

kubectl get nodes --context kind-k8s-training --request-timeout=10s
# 同様にエラー
```

**段階2: API Serverの初期状態**
```bash
# API Serverプロセスは動作中
docker exec k8s-training-control-plane ps aux | grep kube-apiserver

# しかし健全性チェックでetcd障害を報告
docker exec k8s-training-control-plane curl -k https://localhost:6443/healthz
# healthz check failed (etcd failed)
```

**段階3: kubeletによる自動対応**
```bash
# kubeletがAPI Serverの健全性チェック失敗を検知し、CrashLoopBackOff状態に
sleep 60
docker exec k8s-training-control-plane curl -k https://localhost:6443/healthz
# Connection refused (API Server停止)
```

### 技術的解説

etcd障害時の影響：

1. **即座の影響**: Kubernetes API操作が全て不可能
2. **API Server動作継続**: 初期は動作するが、etcd接続失敗を `/healthz` で報告
3. **kubeletによる再起動**: 健全性チェック失敗により、kubeletがAPI Serverを再起動試行
4. **CrashLoopBackOff**: etcdなしではAPI Serverが起動できず、バックオフで再試行継続

**既存Podへの影響**:
- 既に実行中のPodは継続実行（kubeletがローカルで管理）
- 新しいPodの作成・削除は不可能
- ネットワーク通信やストレージアクセスは継続

### 調査手順

```bash
# 1. etcdプロセス確認
docker exec k8s-training-control-plane ps aux | grep etcd

# 2. API Serverプロセス確認
docker exec k8s-training-control-plane ps aux | grep kube-apiserver

# 3. 静的Podマニフェスト確認
docker exec k8s-training-control-plane ls -la /etc/kubernetes/manifests/
docker exec k8s-training-control-plane ls -la /tmp/etcd.yaml.bak

# 4. etcdデータディレクトリ確認（データは保持される）
docker exec k8s-training-control-plane ls -la /var/lib/etcd/
```

### 復旧手順

```bash
# 静的Podマニフェストを元に戻す
docker exec k8s-training-control-plane mv /tmp/etcd.yaml.bak /etc/kubernetes/manifests/etcd.yaml

# etcdプロセス復旧確認（数十秒かかる）
sleep 30
docker exec k8s-training-control-plane ps aux | grep etcd
```

**CrashLoopBackOffからの回復**:
```bash
# API ServerはCrashLoopBackOff状態のため、回復まで時間がかかる
# 選択肢1: 自然回復を待機（最大5分）
sleep 300

# 選択肢2: kubelet再起動で即座に回復
docker exec k8s-training-control-plane systemctl restart kubelet

# 復旧確認
kubectl get nodes --context kind-k8s-training
```

## 障害シナリオ3: API Server高負荷テスト

### 背景知識

API Serverは、すべてのコンポーネントからのリクエストを処理する中央ハブです。高負荷時の動作を理解することで、スケーリングやチューニングの必要性が理解できます。

### 事前準備

```bash
# 負荷テスト用のRBACを設定
kubectl apply --context kind-k8s-training -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-load-test
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: api-load-test
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes", "configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: api-load-test
subjects:
- kind: ServiceAccount
  name: api-load-test
  namespace: default
roleRef:
  kind: ClusterRole
  name: api-load-test
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 負荷発生

```bash
# 高負荷テスト用Podをデプロイ
kubectl apply --context kind-k8s-training -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-load-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api-load-test
  template:
    metadata:
      labels:
        app: api-load-test
    spec:
      serviceAccountName: api-load-test
      containers:
      - name: load-generator
        image: curlimages/curl:latest
        command: ["/bin/sh"]
        args:
        - -c
        - |
          TOKEN=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
          count=0
          while true; do
            curl -k -s -H "Authorization: Bearer \$TOKEN" https://kubernetes.default.svc.cluster.local/api/v1/pods >/dev/null &
            curl -k -s -H "Authorization: Bearer \$TOKEN" https://kubernetes.default.svc.cluster.local/api/v1/nodes >/dev/null &
            curl -k -s -H "Authorization: Bearer \$TOKEN" https://kubernetes.default.svc.cluster.local/api/v1/services >/dev/null &
            curl -k -s -H "Authorization: Bearer \$TOKEN" https://kubernetes.default.svc.cluster.local/api/v1/configmaps >/dev/null &
            count=\$((count + 4))
            if [ \$((count % 100)) -eq 0 ]; then
              echo "\$(date): Sent \$count API requests"
            fi
            sleep 0.05
          done
EOF
```

### 観察される現象

```bash
# kubectlコマンドの応答時間測定
time kubectl get nodes --context kind-k8s-training
# 通常: 1秒以下 → 負荷時: 数秒〜数十秒

# 負荷テストの動作確認
kubectl get pods -l app=api-load-test --context kind-k8s-training
kubectl logs -l app=api-load-test --context kind-k8s-training --tail=5

# API Serverのリソース使用状況
docker exec k8s-training-control-plane top -b -n 1 -w 100 | grep -E "PID|kube"

# 一部のリクエストがタイムアウトする場合
kubectl get pods --request-timeout=5s --context kind-k8s-training
# Error: context deadline exceeded
```

### 調査手順

```bash
# 1. API Serverの負荷確認
docker stats k8s-training-control-plane --no-stream

# 2. 負荷テストPodの稼働状況
kubectl logs -l app=api-load-test --context kind-k8s-training --tail=3

# 3. API Serverログの確認
kubectl logs -n kube-system kube-apiserver-k8s-training-control-plane --tail=20 --context kind-k8s-training | grep -E "(error|limit|throttl)"
```

### 負荷テスト終了

```bash
# 負荷テスト終了
kubectl delete deployment api-load-test --context kind-k8s-training
kubectl delete clusterrolebinding api-load-test --context kind-k8s-training
kubectl delete clusterrole api-load-test --context kind-k8s-training
kubectl delete serviceaccount api-load-test --context kind-k8s-training

# 応答速度の回復確認
time kubectl get nodes --context kind-k8s-training
```

## 障害シナリオ4: Controller Manager障害

### 背景知識

Controller Managerは、Deployment、ReplicaSet、Serviceなどの各種コントローラーを実行し、期待する状態の維持を担当します。

### 障害発生と観察

```bash
# テスト用Deploymentを作成
kubectl create deployment nginx --image=nginx --replicas=1 --context kind-k8s-training

# Controller Managerを停止
docker exec k8s-training-control-plane pkill kube-controller

# 新しいレプリカが作成されないことを確認
kubectl scale deployment nginx --replicas=5 --context kind-k8s-training
kubectl get pods -l app=nginx --context kind-k8s-training

# 既存Podを削除してもレプリカが復旧しない
kubectl delete pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}' --context kind-k8s-training) --context kind-k8s-training
kubectl get pods -l app=nginx --context kind-k8s-training
```

### 復旧確認

```bash
# kubeletが静的Podを自動再起動（数十秒〜数分）
sleep 60
kubectl get deployment nginx --context kind-k8s-training

# スケールアウトが正常に動作することを確認
kubectl get pods -l app=nginx --context kind-k8s-training
```

## 障害シナリオ5: Scheduler障害

### 背景知識

Schedulerは、新しく作成されたPodを適切なWorker Nodeに配置する役割を担います。

### 障害発生と観察

```bash
# Schedulerを停止
docker exec k8s-training-control-plane pkill kube-scheduler

# 新しいPodがPending状態のまま
kubectl run test-pending --image=nginx --context kind-k8s-training
kubectl get pods test-pending --context kind-k8s-training

# イベントでスケジューリング失敗を確認
kubectl describe pod test-pending --context kind-k8s-training | grep -A5 "Events:"
```

### 復旧確認

```bash
# kubeletが静的Podを自動再起動後、Podがスケジュールされる
kubectl get pod test-pending -w --context kind-k8s-training

# クリーンアップ
kubectl delete pod test-pending --context kind-k8s-training
```

## 実践演習

### 演習A: 障害再現練習

以下の手順で、自分で障害を再現し、調査・復旧してください：

1. **kubelet障害**: Worker NodeのkubeletをTerminatingのまま停止し、Pod退避プロセスを観察
2. **etcd障害**: etcdを完全停止し、API Server経由でのクラスタ操作が不可能になることを確認
3. **複合障害**: kubelet停止中にetcdも停止し、影響の違いを比較

### 演習B: 障害切り分け練習

以下の症状が発生した場合、どのコンポーネントに問題があるか特定してください：

**症状1**:
- `kubectl get nodes`は正常に動作
- `kubectl get pods`はタイムアウト
- 既存Podは正常に動作
- 新しいPodが作成できない

**症状2**:
- すべてのkubectlコマンドがタイムアウト
- 既存Podは正常に動作
- Worker Nodeコンテナは正常

**症状3**:
- `kubectl get nodes`でWorker NodeがNotReady
- 該当ノード上のPodがTerminating
- DaemonSet PodはRunning状態

## トラブルシューティング手法

### 障害調査の基本手順

1. **現象の確認**: どのコマンドが成功/失敗するか
2. **影響範囲の特定**: 全体/特定ノード/特定コンポーネント
3. **ログ調査**: 関連コンポーネントのログ確認
4. **プロセス状態確認**: systemctl、ps、docker logsでプロセス状態確認
5. **設定ファイル確認**: マニフェスト、設定ファイルの変更確認

### kindクラスタでの調査コマンド集

```bash
# ノードレベル調査
kubectl get nodes
kubectl describe node <NODE_NAME>
docker exec <NODE_CONTAINER> systemctl status kubelet

# Podレベル調査  
kubectl get pods -A
kubectl describe pod <POD_NAME> -n <NAMESPACE>
kubectl logs <POD_NAME> -n <NAMESPACE>

# コンテナレベル調査
docker ps --filter "label=io.x-k8s.kind.cluster=k8s-training"
docker logs <CONTAINER_NAME>
docker exec <CONTAINER_NAME> ps aux

# イベント調査
kubectl get events --sort-by='.metadata.creationTimestamp'
```

## 障害回復のベストプラクティス

### kindクラスタでの回復手順

1. **プロセス再起動**: `systemctl restart kubelet`
2. **静的Pod復旧**: マニフェストファイルの復元
3. **kubelet再起動**: CrashLoopBackOffからの即座回復
4. **クラスタ再作成**: 最終手段（学習用なので簡単）

### 本番環境への応用

- **監視**: 各コンポーネントの死活監視とアラート設定
- **自動回復**: systemdやKubernetesの自動再起動機能活用
- **バックアップ**: etcdの定期バックアップ
- **高可用性**: 複数ノード構成でSingle Point of Failureの排除

## まとめ

この章では、kindクラスタを使用して各種コンポーネント障害を安全に体験しました。

### 学習したポイント
- **kubelet障害**: Node Controller、Taint-based Evictionのメカニズム
- **etcd障害**: API Serverへの影響と既存Podの継続実行
- **API Server負荷**: 高負荷時の応答遅延とタイムアウト
- **Controller/Scheduler障害**: 各コンポーネントの具体的な役割

### 実践で身についたスキル
- **障害調査**: ログ、イベント、プロセス状態の系統的確認
- **影響範囲特定**: 現象から原因コンポーネントの特定
- **復旧操作**: 安全で効果的な回復手順

### 次の章へ

次の章では、Kubernetesオブジェクトモデルを実践的に操作し、制御ループの動作を詳しく観察します。Deployment、Service、ConfigMapなどの実際の運用を体験しましょう。