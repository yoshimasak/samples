# GKE Inference Gateway を利用した推論基盤の構築

本チュートリアルでは、Google Kubernetes Engine (GKE) 上で **本番運用を見据えた AI 推論基盤** を構築します。
単にモデルを動かすだけでなく、**GKE Inference Gateway** を活用し、高度なルーティングを実装します。

**本チュートリアルのゴール:**
- **GKE Autopilot クラスタの構築:** モデルをデプロイするための GKE Autopilot クラスタの構築 
- **高度なルーティング:** Inference Gateway による負荷分散とボディーベースのルーティング

---

## **1. プロジェクトとネットワークの準備**

GKE で利用する VPC 及びサブネットを作成していきます。また、今回は Inference Gateway (リージョン外部ロードバランサ) を利用するため、Envoy プロキシが配置されるプロキシ専用サブネットの作成が含まれます。

### **1.1 環境変数の設定**

```
export PROJECT_ID=<YOUR PROJECT ID>
gcloud config set project $PROJECT_ID
export REGION=<YOUR REGION>>
export CLUSTER_NAME="inference-gateway-lab"
export NETWORK_NAME="inference-vpc"
export SUBNET_NAME="inference-subnet"
echo "Project: $PROJECT_ID / Region: $REGION"
```

### **1.2 API の有効化**

```bash
gcloud services enable \
  compute.googleapis.com \
  container.googleapis.com \
  cloudresourcemanager.googleapis.com \
  artifactregistry.googleapis.com \
  networkservices.googleapis.com \
  servicenetworking.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com
```

### **1.3 VPC とサブネットの作成**

GKE ノード用のサブネット及び、プロキシ専用サブネット (Proxy-only Subnet) を作成します。既存の VPC を利用する場合は VPC とサブネットの作成は不要ですがプロキシ専用サブネットがない場合はそれだけ既存の VPC 内に作成ください。(既存の VPC を利用する場合はクラスタ作成時のコマンドに含まれる VPC 名を利用する VPC、サブネット名に置き換えてください)

```bash
gcloud compute networks create ${NETWORK_NAME} \
  --project=${PROJECT_ID} \
  --subnet-mode=custom \
  --bgp-routing-mode=regional
```
```bash
gcloud compute networks subnets create ${SUBNET_NAME} \
  --project=${PROJECT_ID} \
  --network=${NETWORK_NAME} \
  --region=${REGION} \
  --range=10.0.0.0/20 \
  --secondary-range=pods-range=10.4.0.0/14,services-range=10.8.0.0/20
```
```bash
gcloud compute networks subnets create proxy-only-subnet \
  --project=${PROJECT_ID} \
  --purpose=REGIONAL_MANAGED_PROXY \
  --role=ACTIVE \
  --region=${REGION} \
  --network=${NETWORK_NAME} \
  --range=10.129.0.0/23
```

## **2. GKE クラスタと機能拡張のインストール**

### **2.1 GKE Autopilot クラスタの作成**

GKE Autopilot クラスタを作成します。Autopilot モードでは Pod のリクエストに応じたノードが用意される仕組みのため、現時点で GPU などのリソース定義は不要です。

```bash
gcloud container clusters create-auto ${CLUSTER_NAME} \
  --project=${PROJECT_ID} \
  --region=${REGION} \
  --network=${NETWORK_NAME} \
  --subnetwork=${SUBNET_NAME} \
  --cluster-secondary-range-name=pods-range \
  --services-secondary-range-name=services-range \
  --release-channel=rapid \
  --async
```

クラスタが `RUNNING` になるまで待機します。

```bash
echo "Provisioning..."; sleep 60; until gcloud container clusters describe ${CLUSTER_NAME} --region=${REGION} --format="value(status)" 2>/dev/null | grep -q "RUNNING"; do echo -n "."; sleep 20; done; echo "Completed"
```

作成したクラスタを操作できるようにクレデンシャルの取得をします。
```bash
gcloud container clusters get-credentials ${CLUSTER_NAME} --region=${REGION}
```

### **2.2 Gateway API Inference Extension (CRD) のインストール**

AI 推論専用の機能 (`InferencePool` 等) を有効化します。GKE のバージョンに応じてインストールする CRD が異なります。

- GKE バージョン `1.34.0-gke.1626000` 以降
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/raw/v1.0.0/config/crd/bases/inference.networking.x-k8s.io_inferenceobjectives.yaml
```

- GKE バージョン `1.34.0-gke.1626000` 以前
```bash
kubectl apply -f  https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.0.0/manifests.yaml
```

---

## **3. 推論モデル (vLLM / Llama3.1-8B-Instruct) のデプロイ**

今回は `meta-llama/Llama-3.1-8B-Instruct` を使用します。

### **3.1 カスタム コンピューティング クラスの定義**

GKE のデフォルトではひとつのノードプールにひとつの Consumption モデル (オンデマンド VM, Spot VM など) のみが指定できます。カスタム コンピューティング クラスを利用することで複数の Consumption モデルの優先順位を指定し、優先順位が高い Consumption モデルのリソースがない場合などに次のモデルにフォールバックできます。

今回のチュートリアルでは予約済みリソース、DWS Flex VM、Spot VM の順に優先度を高く設定したマニフェストとして model-inference-class.yaml を用意しています。必要に応じて優先順位の変更などを実施した上で以下のコマンドを実行します。

```bash
kubectl apply -f model-inference-class.yaml
```

### **3.2 モデルサーバーのデプロイ**

事前に用意されているマニフェストを表示して確認します。
```bash
cat vllm-llama3-1-8b-instruct.yaml
```
* **cloud.google.com/compute-class** に作成したカスタム コンピューティング クラスを指定

本チュートリアルではモデルを Hugging Face からダウンロードするため、Hugging Face のトークンを Kubernetes の Secret に格納します。

```bash
kubectl create secret generic hf-token --from-literal=token=<YOUR HUGGING FACE TOKEN>
```

作成したマニフェストを適用します
```bash
kubectl apply -f vllm-llama3-1-8b-instruct.yaml
```

モデルのダウンロードとコンパイルには数分かかります。その間にゲートウェイの設定を進めます。

### **3.3 InferencePool の作成**

`Service` の代わりに `InferencePool` を作成します。これにより、サイドカー (Endpoint Picker) が注入され、各 Pod の負荷状況（KV キャッシュ等）に基づいたルーティングなどが可能になります。

```bash
helm install vllm-llama-pool \
  --set inferencePool.modelServers.matchLabels.app=vllm-llama3-1-8b-instruct \
  --set inferencePool.selector.matchLabels.app=vllm-llama3-1-8b-instruct \
  --set inferencePool.targetPort=8000 \
  --set provider.name=gke \
  --set provider.gke.autopilot=true \
  --version v1.0.1 \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool
```

### **3.4 Gateway と HTTPRoute の作成**

推論サーバーへのルーティングを提供するロードバランサー＝ゲートウェイを作成します。まずはマニフェストを確認します。
```bash
cat gateway.yaml
```
マニフェストを適用します。
```bash
kubectl apply -f gateway.yaml
```

---

## **4. 動作確認と推論テスト**

### **4.1 起動確認**

Gateway の IP アドレスが払い出され、Pod が `Running` になるまで待ちます。こちらのコマンドをシェル内で実行ください。
```
echo "Provisioning..."; GATEWAY_IP=""; while [ -z "$GATEWAY_IP" ]; do GATEWAY_IP=$(kubectl get gateway inference-gateway -o jsonpath='{.status.addresses[0].value}'); if [ -z "$GATEWAY_IP" ]; then echo -n "."; sleep 5; fi; done; echo -e "\nGateway IP: $GATEWAY_IP"
```
以下のコマンドにより Pod が Ready になるまで待ちます。

```bash
kubectl wait --for=condition=Ready pod -l app=vllm-llama3-1-8b-instruct --timeout=900s
```

### **4.2 推論テスト**

以下のコマンドでリクエストで疎通を確認します。

```
curl -i -X POST http://${GATEWAY_IP}/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "meta-llama/Llama-3.1-8B-Instruct", "messages": [{"role": "user", "content": "GKE Inference Gatewayについて一言で"}], "max_tokens": 100}'
```

---

## **5. 高度な機能: ボディベースルーティング**

### **5.1 本文ベースルーティング (Body-Based Routing)**

通常のロードバランサは URL パスしか見ませんが、Inference Gateway は **JSON ボディ内のモデル名** (`"model": "..."`) を見てルーティングできます。

**1. 拡張機能のインストール:**
```bash
helm install body-based-router oci://registry.k8s.io/gateway-api-inference-extension/charts/body-based-routing \
    --set provider.name=gke \
    --set inferenceGateway.name=inference-gateway \
    --version v1.0.0
```

**2. ルート定義の更新:**
「モデル名が `meta-llama/Llama-3.1-8B-Instruct` の時だけ通す」設定に変更します。マニフェストを確認します。
```bash
cat body-routing.yaml
```
マニフェストを適用します。
```bash
kubectl apply -f body-routing.yaml
```

**3. 検証:**
正しいモデル名で検証します。
```bash
curl -i -X POST http://${GATEWAY_IP}/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "meta-llama/Llama-3.1-8B-Instruct", "messages": [{"role": "user", "content": "Hello"}]}'
```
続いて誤ったモデル名で検証します。
```bash
curl -i -X POST http://${GATEWAY_IP}/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "meta-llama/Llama-3.1-10B-Instruct", "messages": [{"role": "user", "content": "Hello"}]}'
```
Inference Gateway によりリクエストが拒否されることを確認してください。
これにより、Gateway がリクエストの中身（L7）を理解して制御していることが確認できます。
