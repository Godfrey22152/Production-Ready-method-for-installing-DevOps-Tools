# 🚀 **Production-Ready Installation of Argo Rollouts with Helm & GitOps**

This guide ensures:
- ✅ Version Control – Store configurations in Git
- ✅ Declarative Deployment – Helm values managed in code
- ✅ Lifecycle Management – Helm for easy upgrades & rollbacks
- ✅ GitOps-Ready – Works with ArgoCD, Flux, or other GitOps tools

You can find the **Argo Rollout** documentation **[here](https://argo-rollouts.readthedocs.io/en/stable/)**

## 1️⃣ Check Compatibility Matrix
Before installing, verify the compatibility matrix in the **[Argo Rollout Release Repository](https://github.com/argoproj/argo-rollouts/releases)** to ensure you’re using a compatible version for your Kubernetes cluster.

---

## 2️⃣ Pre-requisites
Ensure you have the following installed:

- **A running Kubernetes cluster** (we are using Kubeadm Bare-Metal Cluster✅)
- **Nginx Ingress**
- **Helm 3 installed**
- **kubectl configured**
- **Git repo for storing Helm values** (values-argo-rollouts.${APP_VERSION}.yaml)

---

## 3️⃣ Add the Helm Repository & Update

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

---

## 4️⃣ Define Version Variables for Stability
The controller ships as a `helm` chart, you can chose the `CHART VERSION` and `APP VERSION`of your choice that are of compatible version with your Kubernetes cluster.

- **Choose `CHART VERSION` and `APP VERSION` compatible with your cluster:**

   ```bash
   helm search repo argo/argo-rollouts --versions
   ```

   From the `APP VERSION` we select the `CHART VERSION` that matches the compatibility matrix. For Example:

   ```bash
   NAME                        CHART VERSION   APP VERSION     DESCRIPTION
   argo/argo-rollouts          2.38.0          v1.7.2          A Helm chart for Argo Rollouts
   ```
   
- **Define Version Variables:**

  ```bash
  NAMESPACE="argo-rollouts"
  APP_VERSION="1.7.2"
  CHART_VERSION="2.38.0"  
  HELM_RELEASE_NAME="argo-rollouts"
  HELM_REPO="argo"
  HELM_REPO_URL="https://argoproj.github.io/argo-helm"
  MANIFEST_DIR="./argo-rollouts-manifests"
  MANIFEST_FILE="${MANIFEST_DIR}/installation_argo_rollouts.${APP_VERSION}.yaml"
  ```

---

## 5️⃣ Create Namespace (if not exists)
```bash
kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
```

---

## 6️⃣ Pull Helm Chart and default Helm values for GitOps Version Control
```bash
mkdir -p ${MANIFEST_DIR}
helm show values ${HELM_REPO}/${HELM_RELEASE_NAME} --version ${CHART_VERSION} > ${MANIFEST_DIR}/values-argo-rollouts.${APP_VERSION}.yaml
```

---

## 7️⃣ Configure Helm Values for Production
Modify `values-argo-rollouts.${APP_VERSION}.yaml` to suit your production needs.

- ✅ **Enable Metrics for Prometheus Analysis**
```bash
metrics:
    # -- Deploy metrics service
    enabled: true
    service:
      # -- Metrics service port name
      portName: metrics
      # -- Metrics service port
      port: 8090
      # -- Service annotations
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8090"
    serviceMonitor:
      # -- Enable a prometheus ServiceMonitor
      enabled: true
      # -- Namespace to be used for the ServiceMonitor
      namespace: "argo-rollouts"
      # -- Labels to be added to the ServiceMonitor
      additionalLabels:
        release: "prometheus"
      # -- Annotations to be added to the ServiceMonitor
      additionalAnnotations: {}
      # -- RelabelConfigs to apply to samples before scraping
      relabelings: []
      # -- MetricRelabelConfigs to apply to samples before ingestion
      metricRelabelings: []
```

- ✅ **Enable Webhook Notifications**
```bash
notifications:
  enabled: true
  controller:
    enabled: true
```

- ✅ **Enable Rollout Dashbaord**
```bash
dashboard:
  # -- Deploy dashboard server
  enabled: true
  # -- Set cluster role to readonly
  readonly: false
  # -- Value of label `app.kubernetes.io/component`
  component: rollouts-dashboard
  # -- Annotations to be added to the dashboard deployment
  deploymentAnnotations: {}
  # -- Labels to be added to the dashboard deployment
  deploymentLabels: {}
  # -- Annotations to be added to application dashboard pods
  podAnnotations: {}
  # -- Labels to be added to the application dashboard pods
  podLabels: {}
  # -- [Node selector]
  nodeSelector: {}
  # -- [Tolerations] for use with node taints
  tolerations: []
  # -- Assign custom [affinity] rules to the deployment
  affinity: {}
  logging:
    # -- Set the logging level (one of: `debug`, `info`, `warn`, `error`)
    level: info
    # -- Set the klog logging level
    kloglevel: "0"
```

- ✅ **Set up an Ingress for the Dashboard UI Access (If using NGINX Ingress)**
```bash
## Ingress configuration.
  ## ref: https://kubernetes.io/docs/user-guide/ingress/
  ##
  ingress:
    # -- Enable dashboard ingress support
    enabled: true
    # -- Dashboard ingress annotations
    annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/rewrite-target: /
    # -- Dashboard ingress labels
    labels: {}
    # -- Dashboard ingress class name
    ingressClassName: "nginx"

    # -- Dashboard ingress hosts
    ## Argo Rollouts Dashboard Ingress.
    ## Hostnames must be provided if Ingress is enabled.
    ## Secrets must be manually created in the namespace
    hosts:
      - argorollouts.dashboard.com

    # -- Dashboard ingress paths
    paths:
      - /
    # -- Dashboard ingress path type
    pathType: Prefix
    # -- Dashboard ingress extra paths
    extraPaths: []
      # - path: /*
      #   backend:
      #     serviceName: ssl-redirect
      #     servicePort: use-annotation
      ## for Kubernetes >=1.19 (when "networking.k8s.io/v1" is used)
      # - path: /*
      #   pathType: Prefix
      #   backend:
      #     service
      #       name: ssl-redirect
      #       port:
      #         name: use-annotation
```
- **You can find the complete and configured `values-argo-rollouts.${APP_VERSION}.yaml` file in the installation folder **[Here](https://github.com/Godfrey22152/Canary-Deployment-with-Argo-Rollout/blob/main/Infra-Setup/argo-rollout-setup/values-argo-rollouts.1.7.2.yaml)**

---

## 8️⃣ **Generate and Store Helm Manifest for GitOps (Optional)**

- **If you want to store the manifest for GitOps workflows:**

```bash
helm template ${HELM_RELEASE_NAME} ${HELM_REPO}/${HELM_RELEASE_NAME} \
  --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --set controller.metrics.enabled=true \
  --set controller.serviceMonitor.enabled=true \
  --set dashboard.enabled=true \
  --set notifications.enabled=true \
  --set notifications.controller.enabled=true \
  > ${MANIFEST_FILE}
```
✅ The generated manifest file is Stored in `argo-rollouts-manifests` for version control & GitOps.

- **Then proceed to apply the manifest file to deploy the Argo Rollout:**
```bash
kubectl apply -f ${MANIFEST_FILE}
```

---

## 9️⃣ Deploy Using Helm (Recommended)
Instead of applying the manifest directly, use Helm for better management:

```bash
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${HELM_RELEASE_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --values ${MANIFEST_DIR}/values-argo-rollouts.${APP_VERSION}.yaml
```
✅ **This keeps Helm's lifecycle management features intact.**

---

## 1️⃣0️⃣ Verify Installation

- **Check if the pods are running:** 
```bash
kubectl get pods -n ${NAMESPACE}
```
**Expected output:**
```bash
NAME                                           READY   STATUS             RESTARTS       AGE
argo-rollouts-69566b6478-j7kn2                 1/1     Running            0              1h
argo-rollouts-69566b6478-stf8j                 1/1     Running            0              1h
argo-rollouts-dashboard-f6bf5fc46-5zcdz        1/1     Running            0              1h
```

- **Check the service to ensure that controller and Dashboard services are not exposed externally:**
```bash
kubectl get svc -n ${NAMESPACE}
```

**Expected output:**
```bash
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
argo-rollouts-dashboard   ClusterIP   10.97.27.36      <none>        3100/TCP   1h
argo-rollouts-metrics     ClusterIP   10.103.227.123   <none>        8090/TCP   1h
```

- **Verify if NGINX Ingress Assigned an External IP to the Argo Rollouts Dashboard**
Check whether the Argo Rollouts Dashboard host (`argorollouts.dashboard.com`) has been assigned an external IP by NGINX Ingress:
  
```bash
kubectl get ingress -n ${NAMESPACE}
```

**Expected output:**
```bash
NAME                          CLASS   HOSTS                        ADDRESS          PORTS   AGE
argo-rollouts-dashboard       nginx   argorollouts.dashboard.com   <EXTERNAL-IP>    80      1h
```

If the `ADDRESS` column is empty, get your **NGINX Ingress Controller's external IP:**
```bash
kubectl get svc -n ingress-nginx
```

---

## 1️⃣1️⃣ **Update Your DNS (or Edit `/etc/hosts` for Local Testing)**
If you don't have a public domain (`argorollouts.dashboard.com`), you can test locally by adding an entry to your **local machine's** `/etc/hosts file`:

```bash
echo "<EXTERNAL-IP> argorollouts.dashboard.com" | sudo tee -a /etc/hosts
```
Replace `<EXTERNAL-IP>` with the external IP of your Ingress Controller.

---

## 1️⃣2️⃣ Access the Dashboard
Now, open your browser and go to:
```bash
http://argorollouts.dashboard.com
```

---

## 1️⃣3️⃣ To manage Argo Rollouts use the command for help:
```bash
kubectl-argo-rollouts --help
```
**The command consists of multiple subcommands which can be used to manage Argo Rollouts.**

---

## Why This is Better for Production
- ✅ **Helm Benefits** – Enables easy upgrades, rollbacks, and tracking (helm list, helm rollback).
- ✅ **GitOps Friendly** – Manifests can still be stored in Git, but Helm manages deployments.
- ✅ **Resource Limits** – Prevents resource starvation by defining CPU & memory limits.
- ✅ **Observability** – Metrics and logs can be scraped for monitoring and debugging.
- ✅ **Scalability** – Ensures a structured deployment that scales well.
