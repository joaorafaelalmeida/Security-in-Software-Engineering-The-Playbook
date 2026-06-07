# HashiCorp Vault

> Industry-standard secrets management platform providing dynamic credentials, RBAC, and cryptographic identity for applications and infrastructure

## Overview

HashiCorp Vault is the definitive enterprise secrets management platform, far exceeding simple credential storage. It manages the complete secrets lifecycle: generation, access control, rotation, and revocation. Rather than applications retrieving a static password, Vault dynamically generates short-lived credentials on-demand, creating a paradigm shift in how modern infrastructure handles authentication. Every credential access is logged with full audit trails, enabling compliance with strict governance requirements.

Vault is cloud-agnostic and widely adopted across AWS, GCP, Azure, Kubernetes, and on-premises environments. It integrates with virtually every major platform and is the reference architecture for secrets management in cloud-native organizations.

### Architecture Overview

```mermaid
graph TB
    subgraph "Clients"
        APP["Applications<br/>(Python, Go, Node.js)"]
        K8S["Kubernetes<br/>Pods"]
        CICD["CI/CD<br/>Pipelines"]
        HUMAN["Human Users"]
    end

    subgraph "Authentication Layer"
        K8SAUTH["Kubernetes Auth<br/>(ServiceAccount tokens)"]
        APPROLE["AppRole<br/>(Machine auth)"]
        JWT["JWT/OIDC<br/>(Human IdP)"]
        AWSIAM["AWS IAM<br/>(Cloud workloads)"]
    end

    subgraph "Vault Core"
        VAULTCORE["Vault API Server<br/>(gRPC + HTTP/REST)"]
        POLICY["Policy Engine<br/>(RBAC, ACLs)"]
        LEASE["Lease Manager<br/>(TTL, Renewal, Revocation)"]
    end

    subgraph "Secret Engines"
        KV["KV v2<br/>(Key-Value Storage)"]
        DB["Database<br/>(Dynamic Credentials)"]
        PKI["PKI<br/>(Certificates)"]
        AWS["AWS Engine<br/>(IAM, STS)"]
        GCP["GCP Engine<br/>(Service Accounts)"]
    end

    subgraph "Storage Backend"
        STORAGE["Persistent Storage<br/>(Consul, S3, PostgreSQL, etcd, Raft)"]
    end

    subgraph "Audit & Logging"
        AUDIT["Audit Backends<br/>(File, Syslog, Splunk, CloudWatch)"]
    end

    APP -->|authenticate| K8SAUTH
    K8S -->|authenticate| K8SAUTH
    CICD -->|authenticate| APPROLE
    HUMAN -->|authenticate| JWT

    K8SAUTH -->|grant token| VAULTCORE
    APPROLE -->|grant token| VAULTCORE
    JWT -->|grant token| VAULTCORE
    AWSIAM -->|grant token| VAULTCORE

    VAULTCORE -->|check permissions| POLICY
    VAULTCORE -->|request secret| KV
    VAULTCORE -->|generate credential| DB
    VAULTCORE -->|issue certificate| PKI
    VAULTCORE -->|assume role| AWS
    VAULTCORE -->|create key| GCP

    VAULTCORE -->|track lease| LEASE
    LEASE -->|revoke on expiry| DB

    KV -->|read/write| STORAGE
    DB -->|persist state| STORAGE
    PKI -->|persist state| STORAGE

    VAULTCORE -->|log access| AUDIT
    POLICY -->|log decisions| AUDIT
    LEASE -->|log operations| AUDIT

    style VAULTCORE fill:#4CAF50,stroke:#2E7D32,color:#fff
    style POLICY fill:#2196F3,stroke:#1565C0,color:#fff
    style LEASE fill:#FF9800,stroke:#E65100,color:#fff
    style STORAGE fill:#9C27B0,stroke:#6A1B9A,color:#fff
    style AUDIT fill:#F44336,stroke:#C62828,color:#fff
```

The diagram above shows Vault's layered architecture: clients authenticate via various methods, Vault's policy engine enforces RBAC, secret engines generate/store secrets, and the lease manager automates credential lifecycle (renewal and revocation). All access is logged to audit backends for compliance.

## Key Features

- **Dynamic Secrets:** Database engines generate unique, time-limited credentials per request; upon lease expiry, the credential is automatically revoked from the database, eliminating long-lived static passwords.
- **Secret Engines:** Pluggable backends for different secret types — KV v2 (versioned key-value storage), PKI (TLS certificate and CA issuance), Database (Postgres, MySQL, MongoDB, etc.), AWS (STS credentials), SSH (signed certificates), GCP (service account keys), Azure (VM access credentials), Kubernetes (JWT).
- **Auth Methods:** Multiple identity verification strategies — Kubernetes ServiceAccount (pod identity without human credentials), AppRole (for machines/CI), JWT/OIDC (for humans), AWS IAM (cloud workloads prove identity via IAM), GitHub, LDAP, Okta, and others.
- **Vault Agent & Vault Secrets Operator (VSO):** Sidecar containers (Agent) or Kubernetes operators (VSO) inject secrets as environment variables or mounted files without application code changes, handling lease renewal transparently.
- **Audit Backends:** Every secret access logged to files, syslog, splunk, or CloudWatch. Logs include requester identity, timestamp, requested path, and response (redacted). Non-repudiation essential for compliance (PCI DSS, HIPAA, SOC 2).
- **Encryption as a Service (Transit):** Vault's Transit secret engine encrypts/decrypts data without exposing encryption keys to applications, enabling DEK management at scale.
- **Seal/Unseal & Shamir's Secret Sharing:** Master key protected by Shamir's threshold scheme (N shares, M required to reconstruct), or delegated to cloud KMS (Auto-Unseal). Protects against cold-start attacks.
- **Lease Management:** Every secret issued with a TTL (lease); applications request renewal before expiry; Vault Agent automates this. Expired leases are auto-revoked.
- **Multi-Tenancy & Namespaces:** Enterprise-grade isolation for different teams/projects within a single Vault cluster.

## Use Cases

- **Database Access:** Applications authenticate to Postgres/MySQL without storing the password; Vault issues ephemeral per-connection credentials.
- **Kubernetes Pod Identity:** Pods prove identity via their ServiceAccount token; Vault issues short-lived credentials for accessing AWS/GCP/secrets.
- **CI/CD Pipeline Secrets:** GitHub Actions, GitLab CI, or Jenkins fetch secrets for build time without storing them in the pipeline definition.
- **TLS Certificate Management:** Vault's PKI engine issues short-lived certs, replaces static cert files, enables automatic rotation.
- **SSH Access:** Vault signs short-lived SSH certificates; users present the cert to bastion hosts without permanent SSH keys.
- **Multi-Cloud Credential Unification:** Centralized interface for AWS IAM, GCP service accounts, Azure credentials — consistent RBAC across clouds.
- **Compliance & Audit:** Financial institutions, healthcare, and regulated industries use Vault's audit trail to demonstrate credential governance.

## Installation & Setup

### Prerequisites

- Linux, macOS, or Windows (with WSL)
- 512 MB RAM minimum (HA setups require more)
- Storage backend (Consul, S3, PostgreSQL, etcd, etc. for production; file storage for dev)
- For Kubernetes: Helm 3+, K8s 1.19+
- For cloud integrations: AWS/GCP/Azure credentials with appropriate permissions

### Installation

```bash
# macOS via Homebrew
brew install vault

# Linux via HashiCorp APT/YUM repositories
# Ubuntu
curl https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vault

# Docker
docker run -p 8200:8200 hashicorp/vault server -dev

# Kubernetes Helm Chart (production)
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --set server.ha.enabled=true \
  --set server.ha.replicas=3
```

## Usage Examples

### Basic Usage: Dev Server & KV Secrets

```bash
# Start dev server (insecure, for testing only)
vault server -dev
# Output includes unseal key and root token

# In another terminal, set environment
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='hvs.xxxxx'

# Write a secret
vault kv put secret/database password="db-password-123"

# Read the secret
vault kv get secret/database
# Output:
# ====== Metadata ======
# Key              Value
# ---              -----
# created_time     2025-12-15T10:30:00Z
# custom_metadata  <nil>
# deletion_time    n/a
# destroyed        false
# version          1
# 
# ====== Data ======
# Key         Value
# ---         -----
# password    db-password-123
```

### Advanced Usage: Dynamic Database Credentials

```bash
# Enable database secret engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="app-role,readonly" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.example.com:5432/appdb" \
  username="vault_admin" \
  password="vault_admin_password"

# Create a role that generates users with 1-hour TTL
vault write database/roles/app-role \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Application requests a credential
vault read database/creds/app-role
# Output:
# ====== Data ======
# Key                Value
# ---                -----
# expiration         2025-12-15T11:30:00Z
# last_rotation      2025-12-15T10:30:00Z
# password           A1b2C3d4E5f6G7h8I9j0
# ttl                59m56s
# username           v-token-app-role-abcd1234

# Application connects to DB with this credential; upon 1-hour expiry, Vault revokes the role
vault lease revoke database/creds/app-role/abc123
# The database user "v-token-app-role-abcd1234" is deleted from PostgreSQL
```

### Kubernetes Auth: Pod Identity

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# Configure Kubernetes auth to use pod SA tokens
vault write auth/kubernetes/config \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token \
  kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a policy for app-ns/app-pod
vault write auth/kubernetes/role/app-role \
  bound_service_account_names=app-sa \
  bound_service_account_namespaces=app-ns \
  policies=app-policy \
  ttl=1h

# Inside a pod, the ServiceAccount token authenticates to Vault
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST \
  --data "{\"jwt\": \"$TOKEN\", \"role\": \"app-role\"}" \
  http://vault.vault.svc:8200/v1/auth/kubernetes/login
# Response includes a Vault token and lease info
```

### Vault Agent: Transparent Secret Injection

```hcl
# agent-config.hcl
vault {
  address = "http://vault.vault.svc.cluster.local:8200"
}

auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = {
      role = "app-role"
    }
  }

  sink "file" {
    config = {
      path = "/tmp/vault-token"
    }
  }
}

# Automatically inject secrets as environment variables
env_template "database_password" {
  source      = "/tmp/vault-templates/database.tpl"
  destination = "/app/.env"
  command     = "systemctl reload app"
}

# Or inject as files
template {
  source      = "/tmp/vault-templates/cert.tpl"
  destination = "/app/certs/tls.crt"
  change_mode = "restart"  # Restart the app if cert changes
}
```

Template file (Vault template syntax):
```
{{- with secret "database/creds/app-role" -}}
DATABASE_USER={{ .Data.data.username }}
DATABASE_PASSWORD={{ .Data.data.password }}
DATABASE_URL=postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/appdb
{{- end }}
```

### GitHub Actions Integration

```yaml
# .github/workflows/deploy.yml
name: Deploy with Vault Secrets

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Import Secrets from Vault
        uses: hashicorp/vault-action@v2
        with:
          url: https://vault.example.com
          method: jwt
          role: github-actions-role
          path: jwt
          jwtGithubAudience: https://vault.example.com
          secrets: |
            secret/data/github-actions/docker-registry username | DOCKER_USERNAME;
            secret/data/github-actions/docker-registry password | DOCKER_PASSWORD;
            secret/data/github-actions/aws-credentials access_key | AWS_ACCESS_KEY_ID;
            secret/data/github-actions/aws-credentials secret_key | AWS_SECRET_ACCESS_KEY
      
      - name: Build and Push Docker Image
        run: |
          docker login -u ${{ env.DOCKER_USERNAME }} -p ${{ env.DOCKER_PASSWORD }}
          docker build -t myimage:latest .
          docker push myimage:latest
```

### Configuration Example: HA Vault with Raft Storage

```hcl
# vault.hcl
ui = true
disable_mlock = false

storage "raft" {
  path = "/vault/data"
  node_id = "vault-1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/vault/certs/tls.crt"
  tls_key_file  = "/vault/certs/tls.key"
}

seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
}

api_addr = "https://vault.example.com:8200"
cluster_addr = "https://vault.example.com:8201"
```

## Pros and Cons

### Advantages ✅

- **Dynamic Credentials Paradigm:** Eliminates long-lived passwords; drastically reduces blast radius of leaks.
- **Enterprise Audit Trail:** Full non-repudiation; every secret access logged with identity and timestamp.
- **Multi-Cloud Agnostic:** Single control plane for AWS IAM, GCP service accounts, Azure, Kubernetes, on-premises.
- **Zero-Trust Ready:** Cryptographic workload identity (Kubernetes auth, SPIFFE/SVID) without shared secrets.
- **Operational Transparency:** Lease system forces active credential rotation; passive expiry prevents stale credentials.
- **Ecosystem:** Widely adopted; abundant integrations, tools, and community knowledge.
- **Auto-Unseal & HA:** Cloud KMS integration and clustering eliminate manual unseal ceremonies.

### Limitations ⚠️

- **Operational Complexity:** Requires dedicated HA infrastructure, careful unsealing strategy, disaster recovery planning, and skilled operators.
- **Learning Curve:** Steep for teams unfamiliar with secrets management; many concepts (leases, engines, auth methods) are non-standard.
- **Cost:** Self-hosted HA requires infrastructure investment; HashiCorp Cloud Platform (HCP) Vault adds recurring costs.
- **Debugging Difficulty:** Auditing secret access and troubleshooting lease renewal issues requires expertise.
- **Not a Compliance Silver Bullet:** Audit logging helps but doesn't automatically satisfy compliance requirements; still requires proper policies and monitoring.
- **Adoption Inertia:** Requires application changes (reading from files vs. env vars) and infrastructure overhaul to realize benefits.

## Integration

### Kubernetes Deployment (Helm + VSO)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: app-ns

---
# Vault Secrets Operator watches for SecretProviderClass and injects secrets
apiVersion: secrets.hashicorp.com/v1beta1
kind: HCPAuth
metadata:
  name: hcp-auth
  namespace: vault
spec:
  method: servicePrincipal
  servicePrincipal:
    clientID: "your-client-id"
    clientSecretRef:
      secretKeyRef:
        name: hcp-client-secret
        key: clientSecret

---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: app-ns
spec:
  method: kubernetes
  kubernetes:
    role: app-role
    serviceAccount: app-sa

---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: app-db-secret
  namespace: app-ns
spec:
  vaultAuthRef: vault-auth
  path: secret/data/app/database
  destination:
    name: app-db-secret
    create: true
  refreshAfter: 60s  # Refresh every 60 seconds

---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: app-ns
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - secretRef:
        name: app-db-secret  # Populated by VSO from Vault
```

### CI/CD Integration Patterns

```bash
# GitLab CI example
deploy:
  image: alpine:latest
  script:
    - apk add curl jq
    - |
      VAULT_TOKEN=$(curl -s \
        -X POST \
        -H "Content-Type: application/json" \
        -d "{\"jwt\": \"$CI_JOB_JWT\", \"role\": \"gitlab-ci\"}" \
        https://vault.example.com/v1/auth/jwt/login | jq -r '.auth.client_token')
    - |
      DB_PASSWORD=$(curl -s \
        -H "X-Vault-Token: $VAULT_TOKEN" \
        https://vault.example.com/v1/secret/data/app/db | jq -r '.data.data.password')
    - ./deploy-app.sh $DB_PASSWORD
```

## Alternatives

- **AWS Secrets Manager + Parameter Store** — Cloud-native, simpler for AWS-only workloads, but vendor-locked and lacks dynamic secrets.
- **GCP Secret Manager** — Simpler GCP integration, minimal learning curve, but limited multi-cloud and no dynamic credentials.
- **Azure Key Vault** — Azure-first, RBAC integration, but weak support for non-Azure workloads.
- **Infisical** — Open-source Vault alternative, lower operational overhead, but smaller ecosystem.
- **Doppler** — SaaS secrets manager, zero operational burden, but proprietary and higher cost at scale.
- **Sealed Secrets / External Secrets Operator** — Kubernetes-native, lightweight, but require Vault or another backend for full lifecycle management.

## Resources

- [Official Documentation](https://www.vaultproject.io/docs)
- [GitHub Repository](https://github.com/hashicorp/vault)
- [Vault Tutorials](https://learn.hashicorp.com/vault)
- [Community Forum](https://discuss.hashicorp.com/c/vault)
- [Certified Associate Exam Guide](https://www.vaultproject.io/certification)

## License

Open Source (MPL 2.0) with commercial support via HashiCorp. The open-source version is feature-complete for most use cases; the Enterprise edition adds MFA, namespaces, and additional auth methods.

## Tags

`secrets-management` `vault` `dynamic-credentials` `kubernetes` `iam` `identity` `cloud-native` `compliance` `audit-logging`

---

**Contributed by:** Rodrigo Abreu
**Last Updated:** 2026-06-06
