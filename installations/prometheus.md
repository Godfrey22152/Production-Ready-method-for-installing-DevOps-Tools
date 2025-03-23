# Production-Ready Installation of Prometheus with Helm & GitOps

## 1️⃣ Overview of Prometheus & the Prometheus Stack
Prometheus is an open-source monitoring and alerting toolkit designed for reliability and scalability. It provides robust querying, visualization, and alerting capabilities. It collects metrics from configured targets at given intervals, evaluates rule expressions, displays results, and triggers alerts when predefined conditions are met.

The **[Prometheus Stack](https://github.com/prometheus-operator/kube-prometheus?tab=readme-ov-file)** (kube-prometheus-stack) includes:
- **[Prometheus](https://prometheus.io/):** The core component responsible for scraping and storing metrics as time series data.
- **[AlertManager](https://github.com/prometheus/alertmanager):** Handles alerts and notifications based on user-defined rules.
- **[Grafana](https://grafana.com/):** Provides dashboards for visualizing collected metrics.
- **[Node Exporter](https://github.com/prometheus/node_exporter):** Gathers host-level metrics.
- **[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics):** Exposes Kubernetes resource metrics for monitoring.
- **[Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator):** manages these custom resources (**`ServiceMonitor`** and **`PodMonitor`**), ensuring Prometheus automatically discovers and scrapes the configured services and pods for metrics.

### 📌 Reference Documentation
- [Prometheus Official Documentation](https://prometheus.io/docs/introduction/overview/)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

### 🔹 Pre-requisites
Before installing Prometheus, ensure the following:
- A running Kubernetes cluster (**`v1.19+`**)
- Helm **`v3+`** Installed (**[Installation guide](https://helm.sh/docs/intro/install/)**)
- Persistent Storage Backend for Prometheus (such as OpenEBS)
- Sufficient cluster resources for monitoring workloads

## 2️⃣ Check Compatibility Matrix
Before installation, verify that the **Helm chart version** is compatible with your **Kubernetes version**.

📌 **Verify compatibility here:**
🔗 **[Helm Chart Compatibility Matrix](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)**

---

## 3️⃣ Add the Helm Repository & Update
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 🔹 Check Available Chart Versions
```sh
helm search repo prometheus-community/kube-prometheus-stack --versions | head -5
```

**Expected Output:**
```bash
NAME                                            CHART VERSION   APP VERSION     DESCRIPTION
prometheus-community/kube-prometheus-stack      70.0.0          v0.81.0         kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack      69.8.2          v0.80.1         kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack      69.8.1          v0.80.1         kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack      69.8.0          v0.80.1         kube-prometheus-stack collects Kubernetes manif...
```

---

## 4️⃣ Define Version Variables for Stability
Set environment variables to **ensure version stability** in installations.

### 🔹 Guide to Choosing `CHART_VERSION` & `APP_VERSION`
 - **Run:** `helm search repo prometheus-community/kube-prometheus-stack --versions | head -5`
 - Select the latest stable version for your Kubernetes cluster or version compatible with your **Kubernetes version**.

### 🔹 Set Environment Variables
```sh
CHART_VERSION="70.0.0"
APP_VERSION="0.81.0"
NAMESPACE="monitoring"
HELM_RELEASE_NAME="prometheus"
HELM_REPO="prometheus-community"
CHART_NAME="kube-prometheus-stack"
MANIFEST_DIR="./helm_manifests/prometheus"
MANIFEST_FILE="${MANIFEST_DIR}/installation_prometheus.${CHART_VERSION}.yaml"
```

---

## 5️⃣ Create Namespace (if not exists)
```sh
kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
```

---

## 6️⃣ Extract, Customize, and Store Helm Values and Manifest for GitOps

To ensure a **GitOps-friendly** deployment of Prometheus, we will first extract the **default values file**, modify it with the required configurations, and then use it to deploy Prometheus.

### 📌 Step 1️⃣: Pull the Default Values File
Before customizing Prometheus, extract its default Helm values file:
```sh
mkdir -p ${MANIFEST_DIR}
helm show values ${HELM_REPO}/${CHART_NAME} --version ${CHART_VERSION} > ${MANIFEST_DIR}/values-prometheus.${CHART_VERSION}.yaml
```
**This command:**
- **Creates** the `./helm_manifests/prometheus` directory for storing Helm manifests. 
- **Pulls the default values** for the `kube-prometheus-stack` Helm chart.
- **Saves it** to `${MANIFEST_DIR}/values-prometheus.${CHART_VERSION}.yaml`.
- **Ensures consistency** with the Helm chart version in use.

### 📌 Step 2️⃣: Customize the Values File
Open the `values-prometheus.${CHART_VERSION}.yaml` file and modify the parameters as you wish for your cluster, **For Instance**:
```sh
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: openebs-hostpath
          resources:
            requests:
              storage: 8Gi

grafana:
  enabled: true
  adminPassword: "admin"

alertmanager:
  enabled: true
```

### 📌 Step 3️⃣: Store Helm-Generated Manifest for GitOps
Now, generate and store the customized Helm manifest:

```sh
helm template ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --values ${MANIFEST_DIR}/values-prometheus.${CHART_VERSION}.yaml \
  > ${MANIFEST_FILE}
```
**This step:**
- **Generates a full Kubernetes manifest** from Helm.
- **Saves the generated YAML** for GitOps-style declarative deployment.

👉 **Commit the YAML files to Git** for version tracking and declarative deployments.


### 📌 Step 4️⃣ (Optional) Install Prometheus using the customized values file in GitOps
Use Helm to install Prometheus with the customized values file:
```sh
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --values ${MANIFEST_DIR}/values-prometheus.${CHART_VERSION}.yaml
```
#### ✅ Why Use a Pulled Values File Instead of Inline `--set` Flags?
- ✔ Readability → The configuration is structured and easy to manage.
- ✔ Version Control → Any changes can be tracked using Git.
- ✔ GitOps-Friendly → Ensures declarative configuration for CI/CD.
- ✔ Scalability → Easier to modify as the stack grows.

---

## 7️⃣ Deploy Prometheus Using Helm for Lifecycle Management (RECOMMENDED)

### ✅ Option 1: If Persistent Storage is Automatically Provisioned
```sh
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set grafana.enabled=true \
  --set grafana.adminPassword="admin" \
  --set alertmanager.enabled=true
```

### ✅ Option 2: If Using OpenEBS as Persistent Storage Backend
#### 🔹 Install OpenEBS
```sh
helm repo add openebs https://openebs.github.io/openebs
helm repo update
helm install openebs openebs/openebs -n openebs --set engines.replicated.mayastor.enabled=false --create-namespace
```

#### 🔹 Install Prometheus with OpenEBS Storage
```sh
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName="openebs-hostpath" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=8Gi \
  --set grafana.enabled=true \
  --set grafana.adminPassword="admin" \
  --set alertmanager.enabled=true
```

---

## 8️⃣ Verify Installation
### 🔹 Check Running Pods
```sh
kubectl get pods -n ${NAMESPACE}
```

**Expected Output:**
```bash
NAME                                                       READY   STATUS    RESTARTS       AGE
alertmanager-kube-prometheus-stack-alertmanager-0          2/2     Running   0              1h
kube-prometheus-stack-grafana-6b45695fb9-pfxdn             3/3     Running   0              1h
kube-prometheus-stack-kube-state-metrics-bb69b454c-nf9sw   1/1     Running   0              1h
kube-prometheus-stack-operator-5b6bff4c76-7fmln            1/1     Running   0              1h
kube-prometheus-stack-prometheus-node-exporter-fpfhb       1/1     Running   0              1h
kube-prometheus-stack-prometheus-node-exporter-pjdgv       1/1     Running   0              1h
prometheus-kube-prometheus-stack-prometheus-0              2/2     Running   0              1h
```

### 🔹 Check Services
```sh
kubectl get svc -n ${NAMESPACE}
```

**Expected Output:**
```bash
NAME                                             TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                         AGE
alertmanager-operated                            ClusterIP      None             <none>           9093/TCP,9094/TCP,9094/UDP      1h
kube-prometheus-stack-alertmanager               ClusterIP      10.111.155.232   <none>           9093/TCP,8080/TCP               1h
kube-prometheus-stack-grafana                    ClusterIP      10.105.193.58    <none>           80:31799/TCP                    1h
kube-prometheus-stack-kube-state-metrics         ClusterIP      10.107.6.171     <none>           8080/TCP                        1h
kube-prometheus-stack-operator                   ClusterIP      10.99.47.20      <none>           443/TCP                         1h
kube-prometheus-stack-prometheus                 ClusterIP      10.101.119.196   <none>           9090:31723/TCP,8080:30867/TCP   1h
kube-prometheus-stack-prometheus-node-exporter   ClusterIP      10.106.233.217   <none>           9100/TCP                        1h
prometheus-operated                              ClusterIP      None             <none>           9090/TCP                        1h
```

### 🔹 Check Logs for Debugging
```sh
kubectl logs -l app.kubernetes.io/name=prometheus -n ${NAMESPACE}
```

---

## 9️⃣ Accessing Prometheus, Grafana & AlertManager

### ✅ First Approach: Port Forwarding

#### 🔹 Access Prometheus Dashboard
```sh
kubectl port-forward svc/${CHART_NAME}-prometheus 9090:9090 -n ${NAMESPACE}
```
📌 Open [http://localhost:9090](http://localhost:9090)

#### 🔹 Access Grafana Dashboard
```sh
kubectl port-forward svc/${CHART_NAME}-grafana 3000:80 -n ${NAMESPACE}
```
📌 Open [http://localhost:3000](http://localhost:3000)
- **Username:** admin
- **Password:** admin (or as set in Helm values)

#### 🔹 Access AlertManager Dashboard
```sh
kubectl port-forward svc/${CHART_NAME}-alertmanager 9093:9093 -n ${NAMESPACE}
```
📌 Open [http://localhost:9093](http://localhost:9093)

---

### ✅ Second Approach: change service type to `NodePort`

#### 1️⃣ Expose Prometheus Dashboard
Run this command to change the Prometheus service type to NodePort:
```bash
kubectl patch svc kube-prometheus-stack-prometheus -n monitoring \
  -p '{"spec": {"type": "NodePort"}}'
```
🔹 **Access Prometheus:**
Find the assigned `NodePort`:
```bash
kubectl get svc kube-prometheus-stack-prometheus -n monitoring
```
Look for the `NodePort` under the **PORT(S)** column (default Prometheus port is `9090`).
Now, access Prometheus using:
```bash
http://<NODE_IP>:<NODEPORT>
```

#### 2️⃣ Expose Grafana Dashboard
Modify the Grafana service to `NodePort`:
```bash
kubectl patch svc kube-prometheus-stack-grafana -n monitoring \
  -p '{"spec": {"type": "NodePort"}}'
```

🔹 **Access Grafana:**
Get the NodePort:
```bash
kubectl get svc kube-prometheus-stack-grafana -n monitoring
```

Default Grafana port is `3000`, so open:
```bash
http://<NODE_IP>:<NODEPORT>
```
(Default credentials: admin / admin)

#### 3️⃣ Expose AlertManager Dashboard
Change the AlertManager service to `NodePort`:
```bash
kubectl patch svc kube-prometheus-stack-alertmanager -n monitoring \
  -p '{"spec": {"type": "NodePort"}}'
```

🔹 **Access AlertManager:**
Get the NodePort:
```bash
kubectl get svc kube-prometheus-stack-alertmanager -n monitoring
```

Default AlertManager port is `9093`, so access it via:
```bash
http://<NODE_IP>:<NODEPORT>
```

---

### ✅ Third Approach: Expose Service Through LoadBalancer

#### 1️⃣ Expose Prometheus via LoadBalancer
Run this command to modify the Prometheus service:
```bash
kubectl patch svc kube-prometheus-stack-prometheus -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

🔹 Find the External IP:
```bash
kubectl get svc kube-prometheus-stack-prometheus -n monitoring
```

**Once the `EXTERNAL-IP` field gets assigned, access Prometheus via:**
```bash
http://<EXTERNAL-IP>:9090
```

#### 2️⃣ Expose Grafana via LoadBalancer
Modify the Grafana service:
```bash
kubectl patch svc kube-prometheus-stack-grafana -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

🔹 Find the External IP:
```bash
kubectl get svc kube-prometheus-stack-grafana -n monitoring
```
Access Grafana via:
```bash
http://<EXTERNAL-IP>:3000
```
(Default credentials: admin / admin)

#### 3️⃣ Expose AlertManager via LoadBalancer
Modify the AlertManager service:
```bash
kubectl patch svc kube-prometheus-stack-alertmanager -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

🔹 Find the External IP:
```bash
kubectl get svc kube-prometheus-stack-alertmanager  -n monitoring
```
Access AlertManager via:
```bash
http://<EXTERNAL-IP>:9093
```

---

#### 🛠 Troubleshooting (If EXTERNAL-IP Stays Pending)
If your cluster is running in bare metal or a local environment, you may need **MetalLB** as a **LoadBalancer solution**.
📌 Follow this detailed installation guide to **[Install MetalLB](https://github.com/Godfrey22152/Production-Ready-method-for-installing-DevOps-Tools/blob/main/installations/metallb.md)** and configure an IP address pool for MetalLB to assign external IPs.🚀

---

## 📌 Why This Method is Best?
✅ Version Control – You control installed versions, preventing unexpected updates.
✅ Easy Rollback – helm rollback prometheus <REVISION> makes it easy to revert changes.
✅ GitOps Ready – Store manifests in Git for traceability and automation.
✅ Namespace Isolation – Keeps monitoring components organized.
✅ Observability – Includes Prometheus, Grafana, Alertmanager, Node Exporter, ETC.
