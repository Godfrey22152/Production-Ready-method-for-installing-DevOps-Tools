# **Production-Ready NGINX Ingress Controller Installation**

This method ensures version control, easy upgrades, and rollback capabilities while allowing GitOps-style configuration management.
You can find the Kubernetes NGINX documentation **[here](https://kubernetes.github.io/ingress-nginx/)**

## 1. Check Compatibility Matrix
Before installing, verify the compatibility matrix in the **[NGINX Ingress GitHub Repository](https://github.com/kubernetes/ingress-nginx/)** to ensure you’re using a compatible version for your Kubernetes cluster.

---

## 2. Add the Helm Repository & Update

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
---

## 3. Define Version Variables for Stability
The controller ships as a `helm` chart, you can chose the `CHART VERSION` and `APP VERSION`of your choice that are of compatible version with your Kubernetes cluster.

- **Choose `CHART VERSION` and `APP VERSION` compatible with your cluster:**

   ```bash
   helm search repo ingress-nginx --versions
   ```

   From the `APP VERSION` we select the `CHART VERSION` that matches the compatibility matrix. For Example:

   ```bash
   NAME                            CHART VERSION   APP VERSION     DESCRIPTION
   ingress-nginx/ingress-nginx     4.12.0          1.12.0           Ingress controller for Kubernetes using NGINX a...
   ```
   
- **Define Version Variables:**

   ```bash
   CHART_VERSION="4.12.0"
   APP_VERSION="1.12.0"
   NAMESPACE="ingress-nginx"
   HELM_RELEASE_NAME="ingress-nginx"
   HELM_REPO="ingress-nginx"
   MANIFEST_DIR="./nginx-ingress-manifests"
   MANIFEST_FILE="${MANIFEST_DIR}/installation_nginx_ingress.${APP_VERSION}.yaml"
   ```

---

## 4. Create Namespace (if not exists)
```bash
kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
```
---

## 5. Apply and Store the Helm-Generated Manifest for GitOps (Optional)

- **If you want to store the manifest for GitOps workflows:**

```bash
mkdir -p ${MANIFEST_DIR}

helm template ${HELM_RELEASE_NAME} ${HELM_REPO}/${HELM_RELEASE_NAME} \
  --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --set controller.service.type=LoadBalancer \
  --set controller.resources.requests.cpu="100m" \
  --set controller.resources.requests.memory="128Mi" \
  --set controller.resources.limits.cpu="500m" \
  --set controller.resources.limits.memory="512Mi" \
  > ${MANIFEST_FILE}
```
Commit this to Git if using a GitOps pipeline.

- **Then proceed to apply the manifest file to deploy the Ingress controller:**
```bash
kubectl apply -f ${MANIFEST_FILE}
```

---

## 6. Deploy Using Helm (Recommended)
Instead of applying the manifest directly, use Helm for better management:

```bash
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${HELM_RELEASE_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --set controller.service.type=LoadBalancer \
  --set controller.resources.requests.cpu="100m" \
  --set controller.resources.requests.memory="128Mi" \
  --set controller.resources.limits.cpu="500m" \
  --set controller.resources.limits.memory="512Mi"
```
This keeps Helm's lifecycle management features intact.

---

## 7. Verify the Deployment
Check if the pods are running:

```bash
kubectl get pods -n ${NAMESPACE}
```

Check the service and external IP assignment:
```bash
kubectl get svc -n ${NAMESPACE}
```

Ensure the controller logs show no errors:
```bash
kubectl logs -l app.kubernetes.io/name=ingress-nginx -n ${NAMESPACE} --tail=20
```

---

## 8. (Optional) Enable Metrics & Logging
For monitoring, enable Prometheus metrics:
```bash
helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_REPO}/${HELM_RELEASE_NAME} \
  --namespace ${NAMESPACE} \
  --set controller.metrics.enabled=true \
  --set controller.podAnnotations."prometheus\.io/scrape"="true" \
  --set controller.podAnnotations."prometheus\.io/port"="10254"
```

If using logging solutions like Fluentd, set log format:
```bash
--set controller.config.log-format-escape-json="true"
```
---

## Why This is Better for Production
- ✅ **Helm Benefits** – Enables easy upgrades, rollbacks, and tracking (helm list, helm rollback).
- ✅ **GitOps Friendly** – Manifests can still be stored in Git, but Helm manages deployments.
- ✅ **Resource Limits** – Prevents resource starvation by defining CPU & memory limits.
- ✅ **Observability** – Metrics and logs can be scraped for monitoring and debugging.
- ✅ **Scalability** – Ensures a structured deployment that scales well.
