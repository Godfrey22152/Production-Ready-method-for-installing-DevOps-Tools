# **📌 Production-Ready Installation of MetalLB**

This method ensures version control, easy upgrades, and rollback capabilities while allowing GitOps-style configuration management.

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols. You can find the **MetalLB documentation **[here](https://metallb.io/)**

---

## 1️⃣ Check Compatibility & Version Matrix
Before installation, check the latest MetalLB Helm chart version from the official repository:
👉 **[MetalLB Helm Chart](https://artifacthub.io/packages/helm/metallb/metallb)** also checkout the **[Installation with Helm guide](https://metallb.universe.tf/installation/#installation-with-helm)**

---

## 2️⃣ Add the Helm Repository & Update

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
```
---

## 3️⃣ Define Version Variables for Stability
The controller ships as a `helm` chart, you can chose the `CHART VERSION` and `APP VERSION`of your choice that are of compatible version with your Kubernetes cluster.

- **Choose `CHART VERSION` and `APP VERSION` compatible with your cluster:**

   ```bash
   helm search repo metallb --versions
   ```

   From the `APP VERSION` we select the `CHART VERSION` that matches the compatibility matrix. For Example:

   ```bash
   NAME            CHART VERSION   APP VERSION     DESCRIPTION
   metallb/metallb 0.14.9          v0.14.9         A network load-balancer implementation for Kube...
   ```
   
- **Define Version Variables:**

   ```bash
   CHART_VERSION="0.14.9"
   APP_VERSION="0.14.9"
   NAMESPACE="metallb-system"
   HELM_RELEASE_NAME="metallb"
   HELM_REPO="metallb"
   MANIFEST_DIR="./metallb_manifest"
   MANIFEST_FILE="${MANIFEST_DIR}/installation_matallb.${APP_VERSION}.yaml"
   ```

---

## 4️⃣ Create Namespace (if not exists)
```bash
kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
```

---

## 5️⃣ Store Helm-Generated Manifest for GitOps (Optional)
If following GitOps, generate the MetalLB manifest and store it in your Git repository:

```bash
mkdir -p ${MANIFEST_DIR}

helm template ${HELM_RELEASE_NAME} ${HELM_REPO}/${HELM_RELEASE_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  > ${MANIFEST_FILE}

```
👉 Commit the YAML file **`$MANIFEST_FILE`** to Git for version tracking and declarative deployments.

- **Then proceed to apply the manifest file to deploy MetalLb:**
```bash
kubectl apply -f ${MANIFEST_FILE}
```

---

## 6️⃣ Deploy Using Helm for Lifecycle Management
Instead of applying the raw **`${MANIFEST_FILE}`** YAML file above, use Helm for better control:

```bash
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${HELM_RELEASE_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --set controller.resources.requests.cpu="100m" \
  --set controller.resources.requests.memory="128Mi" \
  --set controller.resources.limits.cpu="500m" \
  --set controller.resources.limits.memory="512Mi"
```
---

## 7️⃣ Verify Installation
Check if the MetalLB pods are running:

```bash
kubectl get pods -n ${NAMESPACE}
```

Example output:
```bash
NAME                                  READY   STATUS    RESTARTS         AGE
metallb-controller-587d67c997-qhtw4   1/1     Running   0                2m
metallb-speaker-f4stp                 4/4     Running   0                2m
metallb-speaker-zxgld                 4/4     Running   0                2m
```

Check the MetalLB service:
```bash
kubectl get svc -n ${NAMESPACE}
```

Example output:
```bash
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
metallb-webhook-service   ClusterIP   10.105.144.245   <none>        443/TCP   2m
```

---

## 8️⃣ Configure a Layer 2 or BGP Address Pool
Now, configure MetalLB to assign external IPs.

Layer 2 Configuration Example (for a simple setup and modifiable) to `./metallb_manifest/metallb-config.yaml`:

```bash
# MetalLB IP range pool and L2 configuration file

# IPAddressPool (CRD) Configuration
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: cluster-pool             # Replace with your favorite pool name
  namespace: metallb-system      # Replace with the namespace Metallb was installed
spec:
  addresses:
  - 192.168.56.100-192.168.56.129   # Replace with your IP address Pool range

---
# L2 Configuration
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: kubeadm-cluster         # Replace with your favorite name
  namespace: metallb-system     # Replace with the namespace Metallb was installed
spec:
  ipAddressPools:
  - cluster-pool                # Replace with your IP address Pool range name
```

Apply it:
```bash
kubectl apply -f ./metallb_manifest/metallb-config.yaml
```
👉 Store this in Git for version-controlled GitOps deployments.

---

📌 Why This Method is Best?
✅ Version Control – You control installed versions, preventing unexpected updates.
✅ Easy Rollback – helm rollback metallb <REVISION> makes it easy to revert changes.
✅ GitOps Ready – Store manifests in Git for traceability and automation.
✅ Namespace Isolation – Keeps MetalLB deployments clean and manageable.
✅ Observability – Can be integrated with Prometheus for monitoring.
