---
title: "オブジェクトモデル実践演習"
---

# オブジェクトモデル実践演習

この章では、Kubernetesオブジェクトモデルの基本構造を実際に操作して理解を深めます。Deployment、Service、ConfigMapなどのオブジェクトを作成・操作し、制御ループの動作を体験していきましょう。

## 目標

- Kubernetesオブジェクトの基本構造（apiVersion、kind、metadata、spec、status）を理解する
- ラベル・セレクターによるオブジェクト間の関係を実践的に学ぶ
- 制御ループ（Control Loop）の動作を実際に観察する
- 宣言的設定管理の概念を体験する
- 実際の運用で必要となるオブジェクト操作スキルを身につける

## 前提条件

- kindクラスタが正常に動作している
- 基本的なkubectlコマンドが理解できている
- YAMLファイルの基本的な構造を理解している

## 演習の準備

```bash
# kindクラスタの動作確認
kubectl get nodes --context kind-k8s-training

# 作業用名前空間を作成
kubectl create namespace object-practice --context kind-k8s-training

# 以降の演習で使用するデフォルト名前空間を設定
kubectl config set-context kind-k8s-training --namespace=object-practice
```

## 演習1: 基本的なオブジェクト構造の理解

### 1.1 Pod オブジェクトの作成と確認

最も基本的なKubernetesオブジェクトであるPodを作成し、構造を確認します。

```yaml
# simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    version: v1.0
    environment: practice
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

```bash
# Podを作成
kubectl apply -f simple-pod.yaml --context kind-k8s-training

# オブジェクトの完全な構造を確認
kubectl get pod nginx-pod -o yaml --context kind-k8s-training

# 特定フィールドの抽出
kubectl get pod nginx-pod -o jsonpath='{.spec.containers[0].image}' --context kind-k8s-training
kubectl get pod nginx-pod -o jsonpath='{.status.phase}' --context kind-k8s-training
```

**確認ポイント**:
- `spec`フィールド: ユーザーが定義した「期待する状態」
- `status`フィールド: システムが管理する「現在の状態」
- `metadata`フィールド: 名前、ラベル、作成時間などのメタデータ
- APIバージョンとKindの指定方法

### 1.2 ラベルとセレクターの基本操作

```bash
# ラベルによるPod検索
kubectl get pods -l app=nginx --context kind-k8s-training
kubectl get pods -l environment=practice --context kind-k8s-training
kubectl get pods -l 'app in (nginx,apache)' --context kind-k8s-training

# ラベルの表示
kubectl get pods --show-labels --context kind-k8s-training

# ラベルの動的追加・更新
kubectl label pod nginx-pod tier=frontend --context kind-k8s-training
kubectl label pod nginx-pod version=v1.1 --overwrite --context kind-k8s-training

# ラベルの削除
kubectl label pod nginx-pod version- --context kind-k8s-training
```

**確認ポイント**:
- ラベルによるフィルタリングの柔軟性
- セレクター表記法（等号、in演算子など）
- ラベルの動的変更が可能であること

### 1.3 アノテーションの活用

```bash
# アノテーションの追加
kubectl annotate pod nginx-pod description="Learning Kubernetes objects" --context kind-k8s-training
kubectl annotate pod nginx-pod deployment.kubernetes.io/revision="1" --context kind-k8s-training

# アノテーションの確認
kubectl describe pod nginx-pod --context kind-k8s-training | grep -A10 "Annotations:"

# アノテーションの削除
kubectl annotate pod nginx-pod deployment.kubernetes.io/revision- --context kind-k8s-training
```

**確認ポイント**:
- ラベル vs アノテーション: セレクション用途 vs メタデータ保存
- システムコンポーネントによるアノテーション利用

## 演習2: Deploymentと制御ループの観察

### 2.1 Deployment オブジェクトの作成

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: v1.0
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

```bash
# Deploymentを作成
kubectl apply -f nginx-deployment.yaml --context kind-k8s-training

# リアルタイムでPod作成プロセスを監視
kubectl get pods -l app=nginx --watch --context kind-k8s-training
```

**別ターミナルで実行**:
```bash
# Deployment → ReplicaSet → Pod の関係を確認
kubectl get deployments,replicasets,pods -l app=nginx --context kind-k8s-training

# イベントを監視
kubectl get events --watch --field-selector involvedObject.name=nginx-deployment --context kind-k8s-training
```

### 2.2 制御ループの実験

**実験1: Pod削除による自動復旧**

```bash
# 現在のPod一覧を確認
kubectl get pods -l app=nginx --context kind-k8s-training

# 1つのPodを削除
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}' --context kind-k8s-training)
kubectl delete pod $POD_NAME --context kind-k8s-training

# 自動復旧を監視
kubectl get pods -l app=nginx --watch --context kind-k8s-training
```

**実験2: レプリカ数変更**

```bash
# レプリカ数を5に変更
kubectl scale deployment nginx-deployment --replicas=5 --context kind-k8s-training

# スケールアップの過程を観察
kubectl get deployment nginx-deployment --watch --context kind-k8s-training

# Pod数の変化を確認
kubectl get pods -l app=nginx --context kind-k8s-training
```

**実験3: イメージ更新（ローリングアップデート）**

```bash
# イメージバージョンを更新
kubectl set image deployment/nginx-deployment nginx=nginx:1.22 --context kind-k8s-training

# ローリングアップデートの過程を監視
kubectl rollout status deployment/nginx-deployment --context kind-k8s-training

# ReplicaSetの変化を確認
kubectl get replicasets -l app=nginx --context kind-k8s-training
```

### 2.3 制御ループの内部動作確認

```bash
# Deploymentの詳細状態確認
kubectl describe deployment nginx-deployment --context kind-k8s-training

# Deployment Controller のイベント確認
kubectl get events --field-selector involvedObject.kind=Deployment --context kind-k8s-training

# ReplicaSet Controller のイベント確認
kubectl get events --field-selector involvedObject.kind=ReplicaSet --context kind-k8s-training

# spec と status の比較
kubectl get deployment nginx-deployment -o yaml --context kind-k8s-training | grep -A10 -B5 -E "(replicas|readyReplicas)"
```

**確認ポイント**:
- Deployment Controller → ReplicaSet → Pod の階層的な制御
- 期待する状態（spec）と現在の状態（status）の差分検出
- 制御ループによる継続的な状態調整

## 演習3: Service オブジェクトによるネットワーク抽象化

### 3.1 Service オブジェクトの作成

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
```

```bash
# Serviceを作成
kubectl apply -f nginx-service.yaml --context kind-k8s-training

# Serviceの詳細確認
kubectl describe service nginx-service --context kind-k8s-training

# Endpointsの自動作成確認
kubectl get endpoints nginx-service --context kind-k8s-training
```

### 3.2 ラベルセレクターによる動的な関連付け

```bash
# 現在のEndpoints確認
kubectl get endpoints nginx-service -o yaml --context kind-k8s-training

# Podのラベルを変更してServiceから切り離し
kubectl label pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}' --context kind-k8s-training) app=nginx-excluded --overwrite --context kind-k8s-training

# Endpointsの変化を確認
kubectl get endpoints nginx-service --context kind-k8s-training

# Deployment Controllerが新しいPodを作成することを確認
kubectl get pods -l app=nginx --context kind-k8s-training
kubectl get pods -l app=nginx-excluded --context kind-k8s-training
```

### 3.3 Service Discovery の確認

```bash
# テスト用Podでサービス接続確認
kubectl run curl-test --image=curlimages/curl -it --rm --context kind-k8s-training -- sh

# Pod内で実行:
nslookup nginx-service.object-practice.svc.cluster.local
curl nginx-service.object-practice.svc.cluster.local
exit
```

**確認ポイント**:
- DNS による Service Discovery の仕組み
- Service名からClusterIPへの解決
- 複数のPodへの負荷分散

## 演習4: ConfigMap と Secret による設定管理

### 4.1 ConfigMap の作成と利用

```yaml
# app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://localhost:5432/myapp"
  debug_level: "INFO"
  max_connections: "100"
  app.properties: |
    # Application configuration
    server.port=8080
    server.host=0.0.0.0
    logging.level=INFO
```

```bash
# ConfigMapを作成
kubectl apply -f app-config.yaml --context kind-k8s-training

# ConfigMapの内容確認
kubectl describe configmap app-config --context kind-k8s-training
```

### 4.2 ConfigMap を使用するPod

```yaml
# pod-with-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: app
    image: nginx:1.21
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    - name: DEBUG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: debug_level
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

```bash
# ConfigMapを使用するPodを作成
kubectl apply -f pod-with-config.yaml --context kind-k8s-training

# 環境変数の確認
kubectl exec app-with-config -- env | grep -E "(DATABASE_URL|DEBUG_LEVEL)" --context kind-k8s-training

# ボリュームマウントされた設定ファイル確認
kubectl exec app-with-config -- ls -la /etc/config/ --context kind-k8s-training
kubectl exec app-with-config -- cat /etc/config/app.properties --context kind-k8s-training
```

### 4.3 Secret による機密情報の管理

```bash
# コマンドラインでSecretを作成
kubectl create secret generic app-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123 \
  --context kind-k8s-training

# Secretの内容確認（Base64エンコードされている）
kubectl get secret app-secret -o yaml --context kind-k8s-training

# Secretをデコード
kubectl get secret app-secret -o jsonpath='{.data.username}' --context kind-k8s-training | base64 --decode
echo  # 改行
kubectl get secret app-secret -o jsonpath='{.data.password}' --context kind-k8s-training | base64 --decode
echo  # 改行
```

## 演習5: 高度なオブジェクト操作

### 5.1 オブジェクト間の関係確認

```bash
# ownerReferences による親子関係確認
kubectl get replicaset -l app=nginx -o yaml --context kind-k8s-training | grep -A10 ownerReferences

kubectl get pods -l app=nginx -o yaml --context kind-k8s-training | grep -A10 ownerReferences

# ラベルによるオブジェクト間の関連確認
kubectl get deployment,replicasets,pods,service,endpoints -l app=nginx --context kind-k8s-training
```

### 5.2 オブジェクトの更新戦略

**宣言的更新（推奨）**:
```bash
# YAMLファイルを編集してレプリカ数を変更
sed -i 's/replicas: 3/replicas: 4/' nginx-deployment.yaml

# 宣言的更新を適用
kubectl apply -f nginx-deployment.yaml --context kind-k8s-training

# 変更の確認
kubectl get deployment nginx-deployment --context kind-k8s-training
```

**命令的更新**:
```bash
# 直接的なパッチ適用
kubectl patch deployment nginx-deployment -p '{"spec":{"replicas":6}}' --context kind-k8s-training

# JSONパッチの利用
kubectl patch deployment nginx-deployment --type='json' -p='[{"op": "replace", "path": "/spec/replicas", "value": 2}]' --context kind-k8s-training
```

### 5.3 オブジェクトのトラブルシューティング

**意図的な設定ミス例**:
```yaml
# broken-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: different-label  # セレクターと不一致
    spec:
      containers:
      - name: nginx
        image: nginx:nonexistent-tag  # 存在しないイメージ
        resources:
          requests:
            memory: "10Gi"  # 過大なリソース要求
```

```bash
# 問題のあるDeploymentを作成
kubectl apply -f broken-deployment.yaml --context kind-k8s-training

# 問題の調査
kubectl get deployment broken-deployment --context kind-k8s-training
kubectl describe deployment broken-deployment --context kind-k8s-training
kubectl get events --field-selector involvedObject.name=broken-deployment --context kind-k8s-training
```

**確認ポイント**:
- セレクターとラベルの不整合によるReplicaSet作成失敗
- 存在しないイメージによるPod作成失敗
- 過大なリソース要求によるスケジューリング失敗

## 演習問題

### 基礎レベル

1. **オブジェクト構造確認**  
   nginx-deploymentの現在のレプリカ数（spec）と実際のレプリカ数（status）を表示してください。

2. **ラベルセレクション**  
   環境ラベル（environment）が"practice"のすべてのPodを表示してください。

3. **リソース階層**  
   nginx-deploymentが管理しているReplicaSetの名前を特定してください。

### 中級レベル

4. **動的設定変更**  
   nginx-deploymentのコンテナイメージをnginx:1.23に更新し、ローリングアップデートの進行を監視してください。

5. **Service とEndpoints**  
   nginx-serviceのEndpointsにリストされるIPアドレスと、実際のPodのIPアドレスが一致することを確認してください。

6. **ConfigMap 統合**  
   新しいConfigMapを作成し、既存のPodで環境変数として利用してください。

### 上級レベル

7. **カスタム制御ループ**  
   bashスクリプトを作成し、特定ラベル（managed=custom）を持つPodを常に2個維持する簡単なコントローラーを実装してください。

8. **リソース監視**  
   kubectlを使用して、nginx-deploymentのPod追加・削除を自動で検知し、変化を記録するスクリプトを作成してください。

## 解答例とヒント

<details>
<summary>解答例を表示</summary>

### 基礎レベル解答

```bash
# 1. レプリカ数確認
kubectl get deployment nginx-deployment -o jsonpath='{.spec.replicas}' --context kind-k8s-training
kubectl get deployment nginx-deployment -o jsonpath='{.status.replicas}' --context kind-k8s-training

# 2. ラベルセレクション
kubectl get pods -l environment=practice --context kind-k8s-training

# 3. ReplicaSet名特定
kubectl get replicasets -l app=nginx --context kind-k8s-training
```

### 中級レベル解答

```bash
# 4. イメージ更新と監視
kubectl set image deployment/nginx-deployment nginx=nginx:1.23 --context kind-k8s-training
kubectl rollout status deployment/nginx-deployment --context kind-k8s-training

# 5. Service とEndpoints確認
kubectl get endpoints nginx-service -o jsonpath='{.subsets[*].addresses[*].ip}' --context kind-k8s-training
kubectl get pods -l app=nginx -o jsonpath='{.items[*].status.podIP}' --context kind-k8s-training

# 6. ConfigMap作成と利用
kubectl create configmap my-config --from-literal=MY_VAR=hello --context kind-k8s-training
kubectl run test-pod --image=nginx --env="MY_VAR_FROM_CM"="$(kubectl get configmap my-config -o jsonpath='{.data.MY_VAR}')" --context kind-k8s-training
```

### 上級レベル解答

```bash
# 7. カスタム制御ループスクリプト
#!/bin/bash
while true; do
  CURRENT=$(kubectl get pods -l managed=custom --no-headers --context kind-k8s-training 2>/dev/null | wc -l)
  if [ $CURRENT -lt 2 ]; then
    kubectl run custom-pod-$RANDOM --image=nginx --labels=managed=custom --context kind-k8s-training
    echo "Created pod, current count: $((CURRENT+1))"
  elif [ $CURRENT -gt 2 ]; then
    POD=$(kubectl get pods -l managed=custom --no-headers -o name --context kind-k8s-training | head -1)
    kubectl delete $POD --context kind-k8s-training
    echo "Deleted pod, current count: $((CURRENT-1))"
  fi
  sleep 10
done

# 8. Pod変化監視スクリプト
#!/bin/bash
kubectl get pods -l app=nginx --watch --no-headers --context kind-k8s-training | while read line; do
  echo "$(date): Pod change detected - $line" | tee -a pod-changes.log
done
```

</details>

## 実運用への応用

### GitOps ワークフロー

```bash
# 1. YAMLファイルをGitリポジトリで管理
git add nginx-deployment.yaml nginx-service.yaml
git commit -m "Add nginx deployment and service"

# 2. 本番環境への適用
kubectl apply -f . --context production-cluster

# 3. 変更の追跡
kubectl diff -f nginx-deployment.yaml --context production-cluster
```

### リソース管理のベストプラクティス

- **名前空間による分離**: 環境やアプリケーション別の分離
- **ラベル戦略**: 一貫したラベル付けルールの確立
- **リソース制限**: CPU/メモリ制限の適切な設定
- **設定外部化**: ConfigMapとSecretによる設定とコードの分離

## まとめ

この章では、Kubernetesオブジェクトモデルを実践的に操作しました。

### 習得したスキル
- **オブジェクト操作**: YAML作成、kubectl apply/patch/deleteの使い分け
- **制御ループ理解**: Deployment Controllerの動作原理と実際の観察
- **関係管理**: ラベル・セレクターによるオブジェクト間の動的な関連付け
- **設定管理**: ConfigMapとSecretによる設定の外部化

### 実用的な知識
- **トラブルシューティング**: describe、logs、eventsによる問題調査
- **運用操作**: ローリングアップデート、スケーリング、設定変更
- **ベストプラクティス**: 宣言的設定、リソース制限、名前空間分離

### 次のステップ

この基礎編を完了したあなたは、以下の中級トピックに進む準備ができています：
- **Service Discovery & Networking**: ClusterIP、NodePort、LoadBalancer、Ingress
- **Storage & Persistent Volumes**: StatefulSet、PV/PVC、StorageClass
- **Configuration & Secrets Management**: より高度な設定管理パターン

継続的な学習により、Kubernetes運用のエキスパートを目指しましょう。