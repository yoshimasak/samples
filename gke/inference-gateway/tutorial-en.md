# Building an Inference Foundation using GKE Inference Gateway

In this tutorial, we will build an **AI inference foundation ready for production operation** on Google Kubernetes Engine (GKE).
Beyond simply running models, we will implement advanced routing using [GKE Inference Gateway](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/about-gke-inference-gateway).

**Goals of this tutorial:**
- **Building a GKE Autopilot Cluster:** Building a GKE Autopilot cluster for deploying models.
- **Advanced Routing:** Load balancing and body-based routing with Inference Gateway.

---

## **1. Project and Network Preparation**

We will create the VPC and subnets to be used with GKE. In this case, since we are using Inference Gateway (Regional External Load Balancer), it includes the creation of a proxy-only subnet where the Envoy proxy will be placed.

### **1.1 Setting Environment Variables**

```
export PROJECT_ID=<YOUR PROJECT ID>
gcloud config set project $PROJECT_ID
export REGION=<YOUR REGION>>
export CLUSTER_NAME="inference-gateway-lab"
export NETWORK_NAME="inference-vpc"
export SUBNET_NAME="inference-subnet"
echo "Project: $PROJECT_ID / Region: $REGION"
```

### **1.2 Enabling APIs**

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

### **1.3 Creating VPC and Subnets**

We will create a subnet for GKE nodes and a proxy-only subnet. If you are using an existing VPC, you do not need to create a VPC and subnets, but if you do not have a proxy-only subnet, please create it within your existing VPC. (If you are using an existing VPC, replace the VPC name in the cluster creation command with your VPC name and subnet name.)

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

## **2. GKE Cluster and Feature Extension Installation**

### **2.1 Creating a GKE Autopilot Cluster**

We will create a GKE Autopilot cluster. In Autopilot mode, nodes are provisioned according to Pod requests, so resource definitions such as GPUs are not required at this stage.

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

Wait until the cluster is `RUNNING`.

```bash
echo "Provisioning..."; sleep 60; until gcloud container clusters describe ${CLUSTER_NAME} --region=${REGION} --format="value(status)" 2>/dev/null | grep -q "RUNNING"; do echo -n "."; sleep 20; done; echo "Completed"
```

Get credentials to operate the created cluster.
```bash
gcloud container clusters get-credentials ${CLUSTER_NAME} --region=${REGION}
```

### **2.2 Installing Gateway API Inference Extension (CRD)**

Enable AI inference-specific features (e.g., `InferencePool`). The CRD to install differs depending on the GKE version.

- GKE version `1.34.0-gke.1626000` or later
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/raw/v1.0.0/config/crd/bases/inference.networking.x-k8s.io_inferenceobjectives.yaml
```

- GKE version `1.34.0-gke.1626000` or earlier
```bash
kubectl apply -f  https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.0.0/manifests.yaml
```

---

## **3. Deploying Inference Model (vLLM / Llama3.1-8B-Instruct)**

This section uses `meta-llama/Llama-3.1-8B-Instruct`.

### **3.1 Defining Custom Compute Classes**

By default, GKE allows only one consumption model (e.g., On-Demand VMs, Spot VMs) per node pool. By using custom compute classes, you can specify priorities for multiple consumption models, allowing fallback to the next model if resources for a higher-priority consumption model are unavailable.

In this tutorial, `model-inference-class.yaml` is provided as a manifest with priorities set in the order of reserved resources, [DWS Flex VM](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/dws), and Spot VMs. Execute the following command after making any necessary priority changes.

By setting reserved resources with the highest priority, followed by DWS Flex VM and Spot VM, resources can be deployed with another consumption model even if all reserved resources are utilized.

```bash
kubectl apply -f model-inference-class.yaml
```

### **3.2 Deploying the Model Server**

Display and confirm the pre-prepared manifest.
```bash
cat vllm-llama3-1-8b-instruct.yaml
```
* Specify the created custom compute class for **cloud.google.com/compute-class**.

In this tutorial, we will download the model from Hugging Face, so we will store the Hugging Face token in a Kubernetes Secret.

```bash
kubectl create secret generic hf-token --from-literal=token=<YOUR HUGGING FACE TOKEN>
```

Apply the created manifest.
```bash
kubectl apply -f vllm-llama3-1-8b-instruct.yaml
```

Model download and compilation will take several minutes. In the meantime, we will proceed with gateway configuration.

### **3.3 Creating an InferencePool**

Instead of a `Service`, an `InferencePool` is created. This injects a sidecar (Endpoint Picker) and enables routing based on the load status (e.g., KV cache) of each Pod.

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

### **3.4 Creating Gateway and HTTPRoute**

Create a load balancer = gateway that provides routing to the inference server. First, confirm the manifest.
```bash
cat gateway.yaml
```
Apply the manifest.
```bash
kubectl apply -f gateway.yaml
```

---

## **4. Operation Check and Inference Test**

### **4.1 Startup Check**

Wait until the Gateway IP address is allocated and the Pod is `Running`. Please execute this command in the shell.
```
echo "Provisioning..."; GATEWAY_IP=""; while [ -z "$GATEWAY_IP" ]; do GATEWAY_IP=$(kubectl get gateway inference-gateway -o jsonpath='{.status.addresses[0].value}'); if [ -z "$GATEWAY_IP" ]; then echo -n "."; sleep 5; fi; done; echo -e "
Gateway IP: $GATEWAY_IP"
```
Wait until the Pod is Ready using the following command.

```bash
kubectl wait --for=condition=Ready pod -l app=vllm-llama3-1-8b-instruct --timeout=900s
```

### **4.2 Inference Test**

Confirm connectivity with a request using the following command.

```
curl -i -X POST http://${GATEWAY_IP}/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "meta-llama/Llama-3.1-8B-Instruct", "messages": [{"role": "user", "content": "Tell me about GKE Inference Gateway in one sentence"}], "max_tokens": 100}'
```

---

## **5. Advanced Feature: Body-Based Routing**

### **5.1 Body-Based Routing**

While standard load balancers only inspect URL paths, Inference Gateway can route requests by examining the **model name within the JSON body** (`"model": "..."`).

**1. Installing the Extension:**
```bash
helm install body-based-router oci://registry.k8s.io/gateway-api-inference-extension/charts/body-based-routing \
    --set provider.name=gke \
    --set inferenceGateway.name=inference-gateway \
    --version v1.0.0
```

**2. Updating Route Definition:**
Change the setting to "only allow requests with `meta-llama/Llama-3.1-8B-Instruct` as the model name". Confirm the manifest.
```bash
cat body-routing.yaml
```
Apply the manifest.
```bash
kubectl apply -f body-routing.yaml
```

**3. Verification:**
Verify with the correct model name.
```bash
curl -i -X POST http://${GATEWAY_IP}/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "meta-llama/Llama-3.1-8B-Instruct", "messages": [{"role": "user", "content": "Hello"}]}'
```
Next, verify with an incorrect model name. (Changed the model name as Llama 3.1 10B which is not correct)
```bash
curl -i -X POST http://${GATEWAY_IP}/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "meta-llama/Llama-3.1-10B-Instruct", "messages": [{"role": "user", "content": "Hello"}]}'
```
Confirm that the request is rejected by the Inference Gateway.
This confirms that the Gateway understands and controls the content of the request (L7).