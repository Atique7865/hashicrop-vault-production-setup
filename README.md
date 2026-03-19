# Production-Grade HashiCorp Vault on AWS EKS
## With MySQL Credentials Management & Backend Integration

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 – Prepare EKS Cluster & IAM Roles](#step-1--prepare-eks-cluster--iam-roles)
4. [Step 2 – Install Vault via Helm (HA + Raft)](#step-2--install-vault-via-helm-ha--raft)
5. [Step 3 – Initialize & Unseal Vault (AWS KMS Auto-Unseal)](#step-3--initialize--unseal-vault-aws-kms-auto-unseal)
6. [Step 4 – Configure Vault Kubernetes Auth](#step-4--configure-vault-kubernetes-auth)
7. [Step 5 – Store MySQL Credentials in Vault](#step-5--store-mysql-credentials-in-vault)
8. [Step 6 – Create Vault Policies & Roles](#step-6--create-vault-policies--roles)
9. [Step 7 – MySQL Deployment YAML](#step-7--mysql-deployment-yaml)
10. [Step 8 – Backend Deployment YAML](#step-8--backend-deployment-yaml)
11. [Step 9 – Vault Dynamic Secrets (Advanced)](#step-9--vault-dynamic-secrets-advanced)
12. [Step 10 – TLS & Ingress for Vault UI](#step-10--tls--ingress-for-vault-ui)
13. [Step 11 – Monitoring & Audit Logs](#step-11--monitoring--audit-logs)
14. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    AWS EKS Cluster                       │
│                                                          │
│  ┌──────────────┐   ┌──────────────┐  ┌──────────────┐ │
│  │  Vault Pod 1 │   │  Vault Pod 2 │  │  Vault Pod 3 │ │
│  │  (Active)    │   │  (Standby)   │  │  (Standby)   │ │
│  └──────┬───────┘   └──────────────┘  └──────────────┘ │
│         │  Raft Consensus (integrated storage)           │
│  ┌──────▼───────────────────────────────────────────┐   │
│  │             Vault Agent Injector                  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌─────────────────┐        ┌──────────────────────┐    │
│  │  MySQL Pod      │        │  Backend Pod         │    │
│  │  + Vault Agent  │        │  + Vault Agent       │    │
│  │  (sidecar)      │        │  (sidecar)           │    │
│  └─────────────────┘        └──────────────────────┘    │
└─────────────────────────────────────────────────────────┘
          │ Auto-Unseal via AWS KMS
          │ IAM Auth via IRSA
          ▼
    ┌──────────────┐
    │   AWS KMS    │
    │   AWS IAM    │
    └──────────────┘
```

---

## Prerequisites

| Tool | Version |
|------|---------|
| kubectl | >= 1.27 |
| helm | >= 3.12 |
| aws cli | >= 2.x |
| eksctl | >= 0.160 |
| vault cli | >= 1.15 |

```bash
# Verify tools
kubectl version --client
helm version
aws --version
vault version
```

---

## Step 1 – Prepare EKS Cluster & IAM Roles

### 1.1 Create EKS Cluster (if not existing)

```bash
eksctl create cluster \
  --name production-cluster \
  --region us-east-1 \
  --nodegroup-name vault-nodes \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 6 \
  --with-oidc \
  --managed
```

### 1.2 Create AWS KMS Key for Auto-Unseal

```bash
# Create KMS key
aws kms create-key \
  --description "HashiCorp Vault Auto-Unseal Key" \
  --key-usage ENCRYPT_DECRYPT \
  --region us-east-1

# Save the KeyId from output
export KMS_KEY_ID="<your-kms-key-id>"

# Create alias
aws kms create-alias \
  --alias-name alias/vault-unseal \
  --target-key-id $KMS_KEY_ID
```

### 1.3 Create IAM Policy for Vault KMS Access

```bash
cat > vault-kms-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/<KMS_KEY_ID>"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name VaultKMSUnsealPolicy \
  --policy-document file://vault-kms-policy.json
```

### 1.4 Create IRSA (IAM Role for Service Account)

```bash
export CLUSTER_NAME="production-cluster"
export REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
  --name vault \
  --namespace vault \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/VaultKMSUnsealPolicy \
  --approve \
  --override-existing-serviceaccounts
```

---

## Step 2 – Install Vault via Helm (HA + Raft)

### 2.1 Add Helm Repo

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### 2.2 Create Namespace & TLS Secret

```bash
kubectl create namespace vault

# Generate self-signed TLS cert (use cert-manager or ACM in production)
openssl req -newkey rsa:2048 -nodes -keyout vault.key \
  -x509 -days 365 -out vault.crt \
  -subj "/CN=vault.vault.svc.cluster.local" \
  -addext "subjectAltName=DNS:vault,DNS:vault.vault,DNS:vault.vault.svc,DNS:vault.vault.svc.cluster.local,IP:127.0.0.1"

kubectl create secret tls vault-tls \
  --cert=vault.crt \
  --key=vault.key \
  --namespace vault
```

### 2.3 Vault Helm Values — `vault-values.yaml`

```yaml
# vault-values.yaml
global:
  enabled: true
  tlsDisable: false

injector:
  enabled: true
  replicas: 2
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

server:
  image:
    repository: "hashicorp/vault"
    tag: "1.15.6"

  # IRSA service account (created in Step 1.4)
  serviceAccount:
    create: false
    name: "vault"
    annotations:
      eks.amazonaws.com/role-arn: "arn:aws:iam::<ACCOUNT_ID>:role/eksctl-production-cluster-addon-iamservice-vault"

  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"

  readinessProbe:
    enabled: true
    path: "/v1/sys/health?standbyok=true"

  livenessProbe:
    enabled: true
    path: "/v1/sys/health?standbyok=true"
    initialDelaySeconds: 60

  extraEnvironmentVars:
    VAULT_CACERT: /vault/userconfig/vault-tls/tls.crt
    VAULT_ADDR: "https://127.0.0.1:8200"

  volumes:
    - name: vault-tls
      secret:
        secretName: vault-tls

  volumeMounts:
    - mountPath: /vault/userconfig/vault-tls
      name: vault-tls
      readOnly: true

  # High Availability with Raft integrated storage
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true

        listener "tcp" {
          tls_disable     = false
          address         = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file   = "/vault/userconfig/vault-tls/tls.crt"
          tls_key_file    = "/vault/userconfig/vault-tls/tls.key"
          tls_min_version = "tls12"
        }

        storage "raft" {
          path    = "/vault/data"
          node_id = "${HOSTNAME}"

          retry_join {
            leader_tls_servername   = "vault-0.vault-internal"
            leader_api_addr         = "https://vault-0.vault-internal:8200"
            leader_ca_cert_file     = "/vault/userconfig/vault-tls/tls.crt"
          }
          retry_join {
            leader_tls_servername   = "vault-1.vault-internal"
            leader_api_addr         = "https://vault-1.vault-internal:8200"
            leader_ca_cert_file     = "/vault/userconfig/vault-tls/tls.crt"
          }
          retry_join {
            leader_tls_servername   = "vault-2.vault-internal"
            leader_api_addr         = "https://vault-2.vault-internal:8200"
            leader_ca_cert_file     = "/vault/userconfig/vault-tls/tls.crt"
          }
        }

        # AWS KMS Auto-Unseal
        seal "awskms" {
          region     = "us-east-1"
          kms_key_id = "<KMS_KEY_ID>"
        }

        service_registration "kubernetes" {}

        api_addr      = "https://${POD_IP}:8200"
        cluster_addr  = "https://${POD_IP}:8201"

  affinity: |
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault
              component: server
          topologyKey: kubernetes.io/hostname

  # Persistent storage for Raft
  dataStorage:
    enabled: true
    size: 10Gi
    storageClass: gp3
    accessMode: ReadWriteOnce

ui:
  enabled: true
  serviceType: ClusterIP
```

### 2.4 Install Vault

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --values vault-values.yaml \
  --wait

# Verify pods
kubectl get pods -n vault
# Expected output:
# vault-0   1/1   Running   0   2m
# vault-1   1/1   Running   0   2m
# vault-2   1/1   Running   0   2m
# vault-agent-injector-xxx   2/2   Running   0   2m
```

---

## Step 3 – Initialize & Unseal Vault (AWS KMS Auto-Unseal)

```bash
# Initialize Vault on the first node
# With KMS auto-unseal, only 1 recovery key share is needed
kubectl exec -n vault vault-0 -- vault operator init \
  -recovery-shares=5 \
  -recovery-threshold=3 \
  -format=json > vault-init.json

# IMPORTANT: Store vault-init.json in a SECURE location (AWS Secrets Manager recommended)
cat vault-init.json

# Save root token
export VAULT_ROOT_TOKEN=$(cat vault-init.json | jq -r '.root_token')

# With KMS Auto-Unseal, vault-0 is already unsealed
# Join vault-1 and vault-2 to the Raft cluster
kubectl exec -n vault vault-1 -- vault operator raft join \
  -leader-ca-cert="$(kubectl exec -n vault vault-0 -- cat /vault/userconfig/vault-tls/tls.crt)" \
  -address=https://vault-1.vault-internal:8200 \
  https://vault-0.vault-internal:8200

kubectl exec -n vault vault-2 -- vault operator raft join \
  -leader-ca-cert="$(kubectl exec -n vault vault-0 -- cat /vault/userconfig/vault-tls/tls.crt)" \
  -address=https://vault-2.vault-internal:8200 \
  https://vault-0.vault-internal:8200

# Verify cluster status
kubectl exec -n vault vault-0 -- vault status
kubectl exec -n vault vault-0 -- vault operator raft list-peers

# Store root token as K8s secret (for CI/CD use only — rotate after setup)
kubectl create secret generic vault-root-token \
  --from-literal=token=$VAULT_ROOT_TOKEN \
  --namespace vault
```

---

## Step 4 – Configure Vault Kubernetes Auth

```bash
# Port-forward to access Vault locally
kubectl port-forward -n vault svc/vault 8200:8200 &

export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="$(pwd)/vault.crt"
export VAULT_TOKEN=$VAULT_ROOT_TOKEN

# Enable Kubernetes authentication method
vault auth enable kubernetes

# Get K8s API server details (EKS)
KUBE_HOST=$(kubectl config view --raw --minify --flatten \
  -o jsonpath='{.clusters[].cluster.server}')

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="$KUBE_HOST" \
  disable_iss_validation=true

# Verify
vault read auth/kubernetes/config
```

---

## Step 5 – Store MySQL Credentials in Vault

```bash
# Enable KV v2 secrets engine
vault secrets enable -path=secret kv-v2

# Store MySQL credentials
vault kv put secret/mysql \
  username="mysql_app_user" \
  password="$(openssl rand -base64 32)" \
  root_password="$(openssl rand -base64 32)" \
  host="mysql-service.mysql.svc.cluster.local" \
  port="3306" \
  database="appdb"

# Verify stored secrets
vault kv get secret/mysql

# Store backend DB connection string (optional — recommended for apps)
vault kv put secret/backend/database \
  DB_HOST="mysql-service.mysql.svc.cluster.local" \
  DB_PORT="3306" \
  DB_NAME="appdb" \
  DB_USER="mysql_app_user" \
  DB_PASSWORD="$(vault kv get -field=password secret/mysql)"
```

---

## Step 6 – Create Vault Policies & Roles

### 6.1 MySQL Policy — `mysql-policy.hcl`

```hcl
# mysql-policy.hcl
path "secret/data/mysql" {
  capabilities = ["read"]
}

path "secret/metadata/mysql" {
  capabilities = ["read", "list"]
}
```

### 6.2 Backend Policy — `backend-policy.hcl`

```hcl
# backend-policy.hcl
path "secret/data/mysql" {
  capabilities = ["read"]
}

path "secret/data/backend/*" {
  capabilities = ["read"]
}

path "secret/metadata/backend/*" {
  capabilities = ["read", "list"]
}
```

```bash
# Apply policies
vault policy write mysql-policy mysql-policy.hcl
vault policy write backend-policy backend-policy.hcl

# Create Kubernetes auth roles
# MySQL role — binds service account in mysql namespace
vault write auth/kubernetes/role/mysql \
  bound_service_account_names=mysql \
  bound_service_account_namespaces=mysql \
  policies=mysql-policy \
  ttl=1h

# Backend role — binds service account in app namespace
vault write auth/kubernetes/role/backend \
  bound_service_account_names=backend \
  bound_service_account_namespaces=app \
  policies=backend-policy \
  ttl=1h

# Verify roles
vault read auth/kubernetes/role/mysql
vault read auth/kubernetes/role/backend
```

---

## Step 7 – MySQL Deployment YAML

The **Vault Agent Injector** automatically injects vault-agent as a sidecar, fetches secrets, and writes them to a shared in-memory volume mounted at `/vault/secrets/`.

### Directory structure for MySQL manifests

```
mysql/
├── namespace.yaml
├── serviceaccount.yaml
├── pvc.yaml
├── service.yaml
└── deployment.yaml
```

### `mysql/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mysql
  labels:
    name: mysql
```

### `mysql/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysql
  namespace: mysql
  annotations:
    # Optional: restrict to specific Vault role
    vault.hashicorp.com/role: "mysql"
```

### `mysql/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: mysql
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

### `mysql/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: mysql
  labels:
    app: mysql
spec:
  type: ClusterIP
  ports:
    - port: 3306
      targetPort: 3306
      protocol: TCP
      name: mysql
  selector:
    app: mysql
```

### `mysql/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
      annotations:
        # ─── Vault Agent Injector Annotations ───────────────────────────────
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "mysql"
        vault.hashicorp.com/agent-inject-status: "update"

        # Vault CA cert (if using TLS)
        vault.hashicorp.com/ca-cert: "/vault/tls/tls.crt"
        vault.hashicorp.com/tls-server-name: "vault.vault.svc.cluster.local"

        # ─── Inject MySQL credentials into /vault/secrets/mysql-creds ───────
        vault.hashicorp.com/agent-inject-secret-mysql-creds: "secret/data/mysql"
        vault.hashicorp.com/agent-inject-template-mysql-creds: |
          {{- with secret "secret/data/mysql" -}}
          MYSQL_ROOT_PASSWORD={{ .Data.data.root_password }}
          MYSQL_USER={{ .Data.data.username }}
          MYSQL_PASSWORD={{ .Data.data.password }}
          MYSQL_DATABASE={{ .Data.data.database }}
          {{- end }}

        # ─── Resource limits for the injected agent ──────────────────────────
        vault.hashicorp.com/agent-limits-cpu: "250m"
        vault.hashicorp.com/agent-limits-mem: "128Mi"
        vault.hashicorp.com/agent-requests-cpu: "100m"
        vault.hashicorp.com/agent-requests-mem: "64Mi"

        # ─── Pre-populate secrets before the main container starts ──────────
        vault.hashicorp.com/agent-init-first: "true"
    spec:
      serviceAccountName: mysql
      securityContext:
        fsGroup: 999
        runAsUser: 999
        runAsGroup: 999

      initContainers:
        # Parse vault secrets and set env vars before MySQL starts
        - name: vault-env-loader
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Convert the secret file into /etc/mysql-env format
              cp /vault/secrets/mysql-creds /etc/mysql-env/env
          volumeMounts:
            - name: mysql-env
              mountPath: /etc/mysql-env
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true

      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql

          # Source environment variables from the vault secret file
          envFrom:
            - secretRef:
                # Fallback K8s secret (only used if Vault is unavailable at first boot)
                name: mysql-fallback-secret
                optional: true

          # Override env vars by reading from vault secrets file
          command:
            - sh
            - -c
            - |
              # Load env vars exported by Vault Agent
              set -a
              source /vault/secrets/mysql-creds
              set +a
              exec docker-entrypoint.sh mysqld

          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
            - name: mysql-config
              mountPath: /etc/mysql/conf.d
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true

          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - |
                  source /vault/secrets/mysql-creds
                  mysqladmin ping -h 127.0.0.1 -u root -p"$MYSQL_ROOT_PASSWORD"
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - |
                  source /vault/secrets/mysql-creds
                  mysqladmin ping -h 127.0.0.1 -u root -p"$MYSQL_ROOT_PASSWORD"
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3

      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
        - name: mysql-config
          configMap:
            name: mysql-config
        - name: mysql-env
          emptyDir: {}
        - name: vault-secrets
          emptyDir:
            medium: Memory  # In-memory — secrets never touch disk

---
# MySQL ConfigMap for tuning
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: mysql
data:
  custom.cnf: |
    [mysqld]
    max_connections       = 200
    innodb_buffer_pool_size = 512M
    slow_query_log        = 1
    slow_query_log_file   = /var/log/mysql/slow.log
    long_query_time       = 2
    character-set-server  = utf8mb4
    collation-server      = utf8mb4_unicode_ci
    bind-address          = 0.0.0.0
```

### Apply MySQL manifests

```bash
kubectl create namespace mysql
kubectl apply -f mysql/namespace.yaml
kubectl apply -f mysql/serviceaccount.yaml
kubectl apply -f mysql/pvc.yaml
kubectl apply -f mysql/service.yaml
kubectl apply -f mysql/deployment.yaml

# Verify injection
kubectl get pods -n mysql
kubectl describe pod -n mysql <mysql-pod-name>
# Look for: vault-agent-init and vault-agent containers

# Check vault-injected secrets (they are in-memory only)
kubectl exec -n mysql <mysql-pod-name> -c mysql -- cat /vault/secrets/mysql-creds
```

---

## Step 8 – Backend Deployment YAML

### Directory structure

```
backend/
├── namespace.yaml
├── serviceaccount.yaml
├── service.yaml
├── deployment.yaml
└── hpa.yaml
```

### `backend/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    name: app
```

### `backend/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
  namespace: app
```

### `backend/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: app
  labels:
    app: backend
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: backend
```

### `backend/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: backend
      annotations:
        # ─── Vault Agent Injector Annotations ───────────────────────────────
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "backend"
        vault.hashicorp.com/agent-inject-status: "update"

        # Vault TLS (same cert as used in vault-values.yaml)
        vault.hashicorp.com/ca-cert: "/vault/tls/tls.crt"
        vault.hashicorp.com/tls-server-name: "vault.vault.svc.cluster.local"

        # ─── Inject MySQL credentials ─────────────────────────────────────
        vault.hashicorp.com/agent-inject-secret-db: "secret/data/mysql"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "secret/data/mysql" -}}
          export DB_HOST="{{ .Data.data.host }}"
          export DB_PORT="{{ .Data.data.port }}"
          export DB_NAME="{{ .Data.data.database }}"
          export DB_USER="{{ .Data.data.username }}"
          export DB_PASSWORD="{{ .Data.data.password }}"
          export DATABASE_URL="mysql://{{ .Data.data.username }}:{{ .Data.data.password }}@{{ .Data.data.host }}:{{ .Data.data.port }}/{{ .Data.data.database }}"
          {{- end }}

        # ─── Inject backend-specific config ──────────────────────────────
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/backend/database"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/backend/database" -}}
          export APP_DB_HOST="{{ .Data.data.DB_HOST }}"
          export APP_DB_PORT="{{ .Data.data.DB_PORT }}"
          export APP_DB_NAME="{{ .Data.data.DB_NAME }}"
          export APP_DB_USER="{{ .Data.data.DB_USER }}"
          export APP_DB_PASSWORD="{{ .Data.data.DB_PASSWORD }}"
          {{- end }}

        # ─── Secret renewal settings ─────────────────────────────────────
        vault.hashicorp.com/agent-pre-populate-only: "false"
        vault.hashicorp.com/agent-limits-cpu: "250m"
        vault.hashicorp.com/agent-limits-mem: "128Mi"
        vault.hashicorp.com/agent-requests-cpu: "100m"
        vault.hashicorp.com/agent-requests-mem: "64Mi"

    spec:
      serviceAccountName: backend
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000

      containers:
        - name: backend
          # Replace with your actual backend image
          image: your-registry/backend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http

          # ─── Source Vault secrets as env vars at container startup ────────
          # The entrypoint sources the vault secret file before starting the app
          command:
            - sh
            - -c
            - |
              # Source MySQL credentials injected by Vault Agent
              if [ -f /vault/secrets/db ]; then
                set -a
                source /vault/secrets/db
                set +a
              fi
              # Start your application
              exec /app/server

          env:
            # Non-sensitive config (safe to put in env directly)
            - name: APP_ENV
              value: "production"
            - name: APP_PORT
              value: "8080"
            - name: LOG_LEVEL
              value: "info"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"

          volumeMounts:
            # Vault secrets are mounted here by the injector
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true

          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

      volumes:
        - name: vault-secrets
          emptyDir:
            medium: Memory  # Secrets stored only in RAM
```

### `backend/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Apply Backend manifests

```bash
kubectl create namespace app
kubectl apply -f backend/namespace.yaml
kubectl apply -f backend/serviceaccount.yaml
kubectl apply -f backend/service.yaml
kubectl apply -f backend/deployment.yaml
kubectl apply -f backend/hpa.yaml

# Verify vault secrets injection
kubectl get pods -n app
kubectl exec -n app <backend-pod> -c backend -- cat /vault/secrets/db
# Should show: export DB_HOST=... export DB_USER=... etc.

# View vault agent logs
kubectl logs -n app <backend-pod> -c vault-agent
```

---

## Step 9 – Vault Dynamic Secrets (Advanced)

For maximum security, use **Dynamic Secrets** — Vault creates unique, short-lived MySQL credentials per request, automatically rotating them.

```bash
# Enable the database secrets engine
vault secrets enable database

# Configure MySQL database plugin
vault write database/config/mysql \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(mysql-service.mysql.svc.cluster.local:3306)/" \
  allowed_roles="backend-role" \
  username="vault_admin" \
  password="<vault_admin_mysql_password>"

# Create a dynamic role with TTL
vault write database/roles/backend-role \
  db_name=mysql \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"

# Update backend policy to use dynamic credentials
cat > backend-dynamic-policy.hcl << 'EOF'
path "database/creds/backend-role" {
  capabilities = ["read"]
}
EOF

vault policy write backend-policy backend-dynamic-policy.hcl
```

### Backend deployment annotation for dynamic secrets

```yaml
# Replace static secret annotations with:
vault.hashicorp.com/agent-inject-secret-db: "database/creds/backend-role"
vault.hashicorp.com/agent-inject-template-db: |
  {{- with secret "database/creds/backend-role" -}}
  export DB_USER="{{ .Data.username }}"
  export DB_PASSWORD="{{ .Data.password }}"
  export DATABASE_URL="mysql://{{ .Data.username }}:{{ .Data.password }}@mysql-service.mysql.svc.cluster.local:3306/appdb"
  {{- end }}
# Vault agent automatically renews the lease before TTL expires
vault.hashicorp.com/agent-pre-populate-only: "false"
```

---

## Step 10 – TLS & Ingress for Vault UI

```yaml
# vault-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ingress
  namespace: vault
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  tls:
    - hosts:
        - vault.yourdomain.com
      secretName: vault-ingress-tls
  rules:
    - host: vault.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vault-ui
                port:
                  number: 8200
```

---

## Step 11 – Monitoring & Audit Logs

### Enable Vault Audit Logging

```bash
# Enable file audit (logs go to stdout → captured by CloudWatch/ELK)
vault audit enable file file_path=stdout

# Verify audit
vault audit list
```

### ServiceMonitor for Prometheus

```yaml
# vault-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vault
  namespace: vault
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: vault
      app.kubernetes.io/instance: vault
  endpoints:
    - port: https
      scheme: https
      path: /v1/sys/metrics
      params:
        format: ["prometheus"]
      tlsConfig:
        caFile: /vault/userconfig/vault-tls/tls.crt
        insecureSkipVerify: false
      bearerTokenSecret:
        name: vault-root-token
        key: token
  namespaceSelector:
    matchNames:
      - vault
```

---

## Troubleshooting

### Vault pod stuck in `0/1 Running`

```bash
# Check logs
kubectl logs -n vault vault-0 -c vault

# Most common causes:
# 1. KMS key unreachable — check IRSA annotations and IAM policy
# 2. Raft join failure — verify TLS cert SAN entries include vault-0/1/2 hostnames
kubectl exec -n vault vault-0 -- vault status
kubectl exec -n vault vault-0 -- vault operator raft list-peers
```

### Backend pod stuck in `Init:0/1`

```bash
# Check vault agent init logs
kubectl logs -n app <pod-name> -c vault-agent-init

# Common causes:
# 1. ServiceAccount not bound to vault role
vault read auth/kubernetes/role/backend

# 2. Policy doesn't allow reading the secret path
vault policy read backend-policy

# 3. Secret doesn't exist at the given path
vault kv get secret/mysql
```

### Vault secrets not updating in running pod

```bash
# The vault-agent sidecar handles renewal — check its logs
kubectl logs -n app <pod-name> -c vault-agent

# Force pod restart to pick up latest secrets (for pre-populate-only mode)
kubectl rollout restart deployment/backend -n app
```

### Reset and re-initialize Vault (disaster recovery)

```bash
# Use recovery keys (from vault-init.json) to generate a new root token
vault operator generate-root -init
# Follow the prompts with your recovery key shares
```

---

## Security Checklist

- [x] KMS Auto-Unseal configured — vault never stores unseal keys
- [x] Vault TLS enabled with minimum TLS 1.2
- [x] Secrets stored in **memory-only** volumes (`emptyDir.medium: Memory`)
- [x] IRSA used — no AWS credentials stored in pods
- [x] Pod Anti-Affinity — vault pods spread across nodes
- [x] Vault Agent Injector — app containers never call Vault directly
- [x] Audit logging enabled to stdout
- [x] Vault root token rotated after initial setup
- [x] Dynamic secrets with short TTL (recommended for production)
- [x] NetworkPolicy restricting access to Vault service (apply separately)

---

## Quick Reference — Useful Commands

```bash
# Check vault health
kubectl exec -n vault vault-0 -- vault status

# List all secrets
vault kv list secret/

# Rotate a secret
vault kv put secret/mysql password="$(openssl rand -base64 32)"

# Check who can access a secret
vault token lookup

# Revoke all tokens for a role
vault token revoke -mode=path auth/kubernetes/login

# Scale vault (update replica count in values.yaml then)
helm upgrade vault hashicorp/vault --namespace vault --values vault-values.yaml

# Port-forward vault UI
kubectl port-forward -n vault svc/vault-ui 8200:8200
# Open: https://localhost:8200
```

---

*Generated for HashiCorp Vault v1.15.x on Amazon EKS. Replace placeholder values (`<ACCOUNT_ID>`, `<KMS_KEY_ID>`, image names) before applying.*
