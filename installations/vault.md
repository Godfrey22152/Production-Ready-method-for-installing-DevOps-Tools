# Production-Ready Installation of Vault on Kubernetes deployment guide

This guide provides step-by-step instructions for deploying a production-ready HashiCorp Vault cluster on Kubernetes using the official HashiCorp Helm chart. This method is recommended for its robustness, scalability, and ease of management.

This guide is based on the official [HashiCorp Vault on Kubernetes deployment guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide).

---
## 1️⃣. Prerequisites

Before you begin, ensure you have the following:

- A running Kubernetes cluster.
- `kubectl` installed and configured to connect to your cluster.
- Helm **`v3+`** Installed (**[Installation guide](https://helm.sh/docs/intro/install/)**)
- Persistent Storage Backend for Prometheus (such as OpenEBS)

---

## 2️⃣. Add the HashiCorp Helm Repository

Add the HashiCorp Helm repository to your local Helm client and update it to fetch the latest chart information.

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### 🔹 Check Available Chart Versions
```sh
helm search repo hashicorp/vault --versions | head -5
```

**Expected Output:**
```bash
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
hashicorp/vault                         0.30.1          1.20.1          Official HashiCorp Vault Chart
hashicorp/vault                         0.30.0          1.19.0          Official HashiCorp Vault Chart
hashicorp/vault                         0.29.1          1.18.1          Official HashiCorp Vault Chart
hashicorp/vault                         0.29.0          1.18.1          Official HashiCorp Vault Chart
```
---

## 3️⃣ Define Version Variables for Stability
Set environment variables to **ensure version stability** in installations.

### 🔹 Guide to Choosing `CHART_VERSION` & `APP_VERSION`
 - **Run:** `helm search repo hashicorp/vault --versions | head -5`
 - Select the latest stable version for your Kubernetes cluster or version compatible with your **Kubernetes version**.

### 🔹 Set Environment Variables
```sh
CHART_VERSION="0.30.1"
NAMESPACE="vault"
HELM_RELEASE_NAME="vault"
HELM_REPO="hashicorp"
CHART_NAME="vault"
MANIFEST_DIR="./helm_manifests/vault"
MANIFEST_FILE="${MANIFEST_DIR}/override-values.${CHART_VERSION}.yaml"
```
---

## 4️⃣. Install Vault with Helm

This command deploys Vault in High Availability (HA) mode with the Raft integrated storage backend.

### 📌 Create a Namespace (if not yet created)
First, create a dedicated namespace `vault` for Vault:

```sh
kubectl create namespace ${NAMESPACE} 
```

### 📌 Export the default values into the versioned file
Use the following command to export default configuration so it can be modified to desired configuration:

```bash
helm show values ${HELM_REPO}/${CHART_NAME} --version ${CHART_VERSION} > "${MANIFEST_FILE}"
```
> Note: See the **[Vault Helm Configuration](https://developer.hashicorp.com/vault/docs/platform/k8s/helm/configuration)** page for a full list of available options and their descriptions.

### 📌 Customize the Values File

Now your default Vault values will be stored in:
`./helm_manifests/vault/override-values.0.30.1.yaml`

You can then edit that file and modify the parameters as you wish for your cluster, **For Instance**:

```bash
cat > ./helm_manifests/vault/override-values.0.30.1.yaml << EOF
# Vault Helm chart Value Overrides
global:
  enabled: true
  tlsDisable: false
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 256Mi
      cpu: 250m

server:
  # Use the Enterprise Image
  image:
    repository: "hashicorp/vault-enterprise"
    tag: "1.16.1-ent"

  # These Resource Limits are in line with node requirements in the
  # Vault Reference Architecture for a Small Cluster
  resources:
    requests:
      memory: 8Gi
      cpu: 2000m
    limits:
      memory: 16Gi
      cpu: 2000m

  # For HA configuration and because we need to manually init the vault,
  # we need to define custom readiness/liveness Probe settings
  readinessProbe:
    enabled: true
    path: "/v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204"
  livenessProbe:
    enabled: true
    path: "/v1/sys/health?standbyok=true"
    initialDelaySeconds: 60

  # extraEnvironmentVars is a list of extra environment variables to set with the stateful set. These could be
  # used to include variables required for auto-unseal.
  extraEnvironmentVars:
    VAULT_CACERT: /vault/userconfig/tls-ca/ca.crt

  # extraVolumes is a list of extra volumes to mount. These will be exposed
  # to Vault in the path `/vault/userconfig/<name>/`.
  extraVolumes:
    - type: secret
      name: tls-server
    - type: secret
      name: tls-ca
    - type: secret
      name: kms-creds

  # This configures the Vault Statefulset to create a PVC for audit logs.
  # See https://www.vaultproject.io/docs/audit/index.html to know more
  auditStorage:
    enabled: true

  standalone:
    enabled: false

  # Run Vault in "HA" mode.
  ha:
    enabled: true
    replicas: 5
    raft:
      enabled: true
      setNodeId: true

      config: |
        ui = true
        cluster_name = "vault-integrated-storage"
        listener "tcp" {
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file = "/vault/userconfig/tls-server/tls.crt"
          tls_key_file = "/vault/userconfig/tls-server/tls.key"
        }

        storage "raft" {
          path = "/vault/data"
          retry_join {
            leader_api_addr = "https://vault-0.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/tls-ca/ca.crt"
            leader_client_cert_file = "/vault/userconfig/tls-server/tls.crt"
            leader_client_key_file = "/vault/userconfig/tls-server/tls.key"
          }
          retry_join {
            leader_api_addr = "https://vault-1.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/tls-ca/ca.crt"
            leader_client_cert_file = "/vault/userconfig/tls-server/tls.crt"
            leader_client_key_file = "/vault/userconfig/tls-server/tls.key"
          }
          retry_join {
            leader_api_addr = "https://vault-2.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/tls-ca/ca.crt"
            leader_client_cert_file = "/vault/userconfig/tls-server/tls.crt"
            leader_client_key_file = "/vault/userconfig/tls-server/tls.key"
          }
          retry_join {
            leader_api_addr = "https://vault-3.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/tls-ca/ca.crt"
            leader_client_cert_file = "/vault/userconfig/tls-server/tls.crt"
            leader_client_key_file = "/vault/userconfig/tls-server/tls.key"
          }
          retry_join {
            leader_api_addr = "https://vault-4.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/tls-ca/ca.crt"
            leader_client_cert_file = "/vault/userconfig/tls-server/tls.crt"
            leader_client_key_file = "/vault/userconfig/tls-server/tls.key"
          }
        } 

# Vault UI
ui:
  enabled: true
  serviceType: "LoadBalancer"
  serviceNodePort: null
  externalPort: 8200
EOF
```
**This step:**
- The above generated **`./helm_manifests/vault/override-values.0.30.1.yaml`** default configuration file specifies a subset of values for parameters that are often modified when deploying Vault in production on Kubernetes. After creating this file, Helm can reference it to install Vault with a customized configuration. 
- **Saves the generated YAML** for GitOps-style declarative deployment.

👉 **Modify the generated **`./helm_manifests/vault/override-values.0.30.1.yaml`** to suit your kubernetes environment and Commit the YAML file to Git** for version tracking and declarative deployments.

### 🔹 Install following the Options below: 

#### 📌 Option 1: Install Vault with Default Configuration

```bash
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} --namespace ${NAMESPACE} --version ${CHART_VERSION}
```

#### 📌 Option 2: Install the Chart
Use the following command to install the Vault Helm chart with recommended settings for production:

##### Dry Run: Preview Only
```bash
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --set "server.ha.enabled=true" \
  --set "server.ha.raft.enabled=true" \
  --set "server.resources.requests.cpu=250m" \
  --set "server.resources.requests.memory=512Mi" \
  --set "server.resources.limits.cpu=500m" \
  --set "server.resources.limits.memory=1Gi" \
  --set "server.service.type=LoadBalancer"
  --dry-run \
  --debug 
```

##### Real Install: Deploy to the cluster
```bash
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --version ${CHART_VERSION} \
  --set "server.ha.enabled=true" \
  --set "server.ha.raft.enabled=true" \
  --set "server.resources.requests.cpu=250m" \
  --set "server.resources.requests.memory=512Mi" \
  --set "server.resources.limits.cpu=500m" \
  --set "server.resources.limits.memory=1Gi" \
  --set "server.service.type=LoadBalancer"
```
**Configuration Breakdown:**
- `server.ha.enabled=true`: Enables High Availability mode.
- `server.ha.raft.enabled=true`: Configures Vault to use the Raft integrated storage backend, which is recommended for production.
- `server.resources...`: Sets CPU and memory requests and limits to ensure Vault has adequate resources and to prevent it from consuming excessive resources.
- `server.service.type=LoadBalancer`: Exposes the Vault service via a cloud provider's load balancer.

#### 📌 Option 3: Deploy Vault Using Helm for Lifecycle Management (RECOMMENDED)
```bash
# Dry run install
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --values "${MANIFEST_FILE}" \
  --version ${CHART_VERSION} \
  --dry-run \
  --debug

# Real install
helm install ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --values "${MANIFEST_FILE}" \
  --version ${CHART_VERSION}
```

> Easily modify the configuration file and upgrade to a new version using:

```bash
# Dry run upgrade
helm upgrade ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --values "${MANIFEST_FILE}" \
  --version ${CHART_VERSION} \
  --dry-run \
  --debug

# Real upgrade
helm upgrade ${HELM_RELEASE_NAME} ${HELM_REPO}/${CHART_NAME} \
  --namespace ${NAMESPACE} \
  --values "${MANIFEST_FILE}" \
  --version ${CHART_VERSION}
```
---

## 5️⃣ Verify Installation
### 🔹 Check Running Pods
```sh
kubectl get pods -n ${NAMESPACE}
```

---
## 4. Initialize and Unseal Vault

After deploying Vault, you need to initialize and unseal it.

### Initialize Vault
1. **Exec into the `vault-0` pod:**
   ```bash
   kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
   ```

2. **Initialize Vault:**
   Inside the pod's shell, run the `vault operator init` command:
   ```bash
   vault operator init
   ```
   This command will output the unseal keys and the initial root token. **Save these securely!** You will need them to unseal Vault and for initial authentication.

### Unseal Vault
Vault starts in a sealed state. You need to unseal it using the unseal keys generated during initialization.

1. **Unseal the first pod (`vault-0`):**
   From within the `vault-0` pod's shell, run the following command three times, each time with a different unseal key:
   ```bash
   vault operator unseal <UNSEAL_KEY>
   ```

2. **Unseal the other pods:**
   The other Vault pods (`vault-1`, `vault-2`) will automatically join the Raft cluster and unseal themselves.

3. **Exit the pod's shell:**
   ```bash
   exit
   ```

---
## 5. Verify the Deployment

Check the status of the Vault pods and service to ensure they are running correctly.

### Check Pods
```bash
kubectl get pods -n vault
```
You should see three `vault` pods in the `Running` state, and one `vault-agent-injector` pod.

### Check Service
```bash
kubectl get svc -n vault
```
You should see the `vault` service with a `LoadBalancer` type and an external IP address.

### Check Vault Status
You can also exec into one of the pods and run `vault status` to verify that the Vault is unsealed and that the Raft cluster is healthy.
```bash
kubectl exec vault-0 -n vault -- vault status
```

---
## Why This is Better for Production

- ✅ **High Availability:** The `server.ha.enabled=true` setting ensures that Vault is deployed in a high-availability configuration, which is essential for production environments.
- ✅ **Integrated Storage:** The Raft storage backend (`server.ha.raft.enabled=true`) is integrated directly into Vault, simplifying the architecture and reducing operational overhead.
- ✅ **Scalability:** The Helm chart allows for easy scaling of the Vault cluster by adjusting the number of replicas.
- ✅ **Resource Management:** Defining resource requests and limits prevents Vault from consuming excessive resources and ensures stable operation.
- ✅ **Managed by Helm:** Using Helm simplifies the deployment, upgrade, and rollback processes.
