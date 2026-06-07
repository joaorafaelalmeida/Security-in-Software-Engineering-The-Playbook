# Dynamic Secrets Over Static Credentials

> Replace long-lived passwords with ephemeral, auto-rotating credentials that expire after minutes to hours, drastically reducing the blast radius of credential leaks

## The Problem

Static credentials — database passwords, API keys, cloud access tokens, TLS private keys, service account keys — never expire. Once issued, they remain valid indefinitely until someone manually rotates them. This creates a critical vulnerability window:

1. **Detection Lag:** The average time to detect a leaked credential is 200+ days (Verizon 2024 DBIR). During this window, attackers have unrestricted access.

2. **Manual Rotation Friction:** Rotation requires coordinating changes across applications, configuration files, deployment pipelines, and monitoring systems. This coordination burden causes teams to skip or delay rotation, leaving compromised credentials in production for months.

### Exposure Window Comparison

The following comparison illustrates the critical difference in exposure windows:

```mermaid
graph LR
    A["STATIC CREDENTIAL<br/>Issued & Never Expires"]
    B["Day 200<br/>Leaked"]
    C["Day 200<br/>Detected"]
    D["Day 210+<br/>Manual Rotation"]
    E["EXPOSURE WINDOW<br/>200+ Days"]

    D1["DYNAMIC CREDENTIAL<br/>Issued, TTL 1 Hour"]
    D2["Hour 0.5<br/>Leaked"]
    D3["Hour 1<br/>Auto-Revoked"]
    D4["EXPOSURE WINDOW<br/>~30 Minutes"]

    A --> B --> C --> D
    D --> E
    
    D1 --> D2 --> D3
    D3 --> D4

    style E fill:#F44336,stroke:#C62828,color:#fff,stroke-width:2px
    style D4 fill:#4CAF50,stroke:#2E7D32,color:#fff,stroke-width:2px
```

**Key insight:** A leaked dynamic credential becomes useless within its TTL (typically 1 hour), automatically limiting the blast radius of a breach. A leaked static credential grants permanent access until manual rotation — a process that takes days or weeks.

3. **Blast Radius:** A single leaked database password or cloud API key grants attackers permanent access to resources until manual remediation. For systems with many users (e.g., microservices with dozens of replicas), a single compromised credential can be scattered across logs, crash dumps, memory, and backups, making comprehensive revocation nearly impossible.

4. **Audit Trail Opacity:** When a static credential is leaked, determining *which* application instance used it, *when*, and against *what* resources is difficult. Auditors cannot prove who accessed what, violating compliance requirements (PCI DSS, HIPAA, SOC 2).

5. **Credential Sprawl:** Teams create shared static credentials, reuse them across environments, and commit them to version control "temporarily" while rushing to launch. The "temporary" becomes permanent, creating a landscape of unmanaged credentials.

**Real-world incident:** In 2019, a Tesla AWS S3 bucket credentials were leaked in GitHub. An attacker gained access and exfiltrated confidential data. The static credential had no expiration, remained valid for weeks post-leak, and affected multiple S3 buckets. A short-lived (1-hour TTL) credential would have rendered the stolen access useless within minutes.

## The Solution

### Overview

Dynamic secrets are credentials *generated on-demand* for a specific purpose, with a finite lifetime (TTL ranging from minutes to hours). The secret is automatically revoked upon expiry, eliminating the need for manual rotation and drastically bounding the window of exposure.

Key principles:

1. **Each request gets a unique credential** — Not shared, not reused. Each application instance or user requesting database access receives a credential unique to that instance and that session.

2. **Automatic expiry via lease TTL** — The credential expires after a fixed duration (e.g., 1 hour). After expiry, the underlying resource (database user, AWS IAM role, SSH key) is automatically revoked.

3. **Lease renewal before expiry** — Applications transparently request credential renewal before expiry. Vault Agent or cloud IAM SDKs handle this automatically; applications never notice.

4. **Full audit trail per-credential** — Each dynamically-issued credential can be tied to the requesting service, timestamp, and purpose, enabling non-repudiation.

### Implementation Approaches

#### Approach 1: Vault Dynamic Secrets (Database Engine)

**Overview:** Vault's database secret engine stores a single privileged "admin" credential, then uses it to create ephemeral "application" credentials on-demand.

**Setup:**

```hcl
# Enable the database secret engine
vault secrets enable database

# Configure PostgreSQL connection with admin credentials
vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="app-readonly,app-write,job-runner" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.example.com:5432/appdb" \
  username="vault_admin" \
  password="vault_admin_secret_key_stored_in_vault"

# Create a role that defines what database user gets created
# When Vault issues a credential for this role, it runs the creation_statements
vault write database/roles/app-readonly \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' IN ROLE readonly; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

vault write database/roles/app-write \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' IN ROLE app_writes; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

**Application Usage:**

```python
# Python application using hvac (HashiCorp Vault SDK)
import hvac
import os
from datetime import datetime, timedelta

vault_client = hvac.Client(url='http://vault:8200', token=os.environ['VAULT_TOKEN'])

# Request a credential with 1-hour TTL
response = vault_client.secrets.database.generate_credentials(
    name='app-readonly',
    mount_point='database'
)

username = response['data']['username']
password = response['data']['password']
ttl = response['data']['ttl']  # e.g., 3599 seconds
lease_id = response['lease_id']

# Connect to database with this ephemeral credential
db_connection = psycopg2.connect(
    host='postgres.example.com',
    user=username,
    password=password,
    database='appdb'
)

# Before credential expires, request renewal
renewal_response = vault_client.sys.renew_lease(lease_id=lease_id)

# Upon application shutdown or after max_ttl, the credential is revoked
# The database user is automatically dropped; the password is rendered useless
vault_client.sys.revoke_lease(lease_id=lease_id)
```

**Key advantages:**
- Admin credential never exposed to application; only ephemeral per-request credentials issued.
- Database-level enforcement: expired credential literally cannot authenticate (PostgreSQL VALID UNTIL clause enforces this).
- Blast radius bounded by TTL: if credential leaks, attacker has 1 hour maximum.

**Key pitfalls:**
- Database must support dynamic user creation (not all do).
- Poorly-tuned TTL (too short: credentials expire mid-operation; too long: defeats the purpose).
- Not monitoring lease renewal failures leads to silent credential expiry and application outages.

#### Approach 2: Cloud IAM Short-Lived Credentials (AWS STS, GCP Workload Identity, Azure Managed Identity)

**AWS Example:**

Instead of storing a long-lived IAM access key, use AWS Security Token Service (STS) to issue short-lived credentials:

```python
import boto3
from botocore.credentials import RefreshableCredentials

# Workload Identity: Pod's ServiceAccount is bound to an IAM role via IRSA (IAM Roles for Service Accounts)
# AWS SDK automatically uses the pod's identity to assume the role and get short-lived credentials

sts = boto3.client('sts')

# Manual AssumeRole (if not using IRSA)
assumed_role = sts.assume_role(
    RoleArn='arn:aws:iam::123456789012:role/app-role',
    RoleSessionName='app-instance-001',
    DurationSeconds=3600  # 1 hour
)

access_key = assumed_role['Credentials']['AccessKeyId']
secret_key = assumed_role['Credentials']['SecretAccessKey']
session_token = assumed_role['Credentials']['SessionToken']
expiration = assumed_role['Credentials']['Expiration']

# Credentials automatically expire after DurationSeconds
# AWS SDK (boto3) handles automatic renewal transparently
s3 = boto3.client('s3', 
    aws_access_key_id=access_key,
    aws_secret_access_key=secret_key,
    aws_session_token=session_token
)

# Even if these credentials leak, AWS rejects them after expiration
s3.list_buckets()
```

**GCP Example (Workload Identity Federation):**

```python
from google.auth import _helpers
from google.auth.transport.requests import Request
import google.auth.credentials

# Pod's ServiceAccount token proves identity to GCP
# GCP exchanges the token for a short-lived access token (1 hour default)
credentials = google.auth.default()

# Credentials automatically refresh before expiry; SDK handles this
client = google.cloud.storage.Client(credentials=credentials)
buckets = client.list_buckets()

# The underlying access_token has a finite TTL; expired tokens are rejected by GCP
print(f"Token expires at: {credentials.expiry}")
```

**Key advantages:**
- No long-lived keys stored anywhere; identity is cryptographic (pod/workload proves itself via token).
- Cloud IAM handles credential generation and revocation transparently.
- Automatic renewal built into all major SDKs.

**Key pitfalls:**
- Requires infrastructure setup (IRSA, Workload Identity, Managed Identity).
- Tokens still need careful handling (never logged, never cached longer than necessary).
- Multi-cloud scenarios require duplicating configuration per cloud.

#### Approach 3: SPIFFE/SVID (Secure Production Identity Framework for Everyone)

**Overview:** Instead of credentials, services receive cryptographic identity certificates (SVIDs — SPIFFE Verifiable Identity Documents). A service presents its SVID to prove identity; the receiving service validates the certificate. No shared secret, no key storage — just cryptographic proof.

**Setup with SPIRE (reference implementation):**

```bash
# SPIRE Server generates SVIDs for workloads
# Workloads registered with SPIRE receive X.509 certificates

# Register a pod as a workload
kubectl exec spire-server -- /opt/spire/bin/spire-server \
  entry create \
  -spiffeID spiffe://example.com/app \
  -parentID spiffe://example.com/ns/default/sa/app-sa \
  -selector k8s:ns:default \
  -selector k8s:sa:app-sa
```

**Pod receives SVID (certificate + private key):**

```
Certificate: SPIFFE://example.com/app
  - Issued by: SPIRE CA
  - Valid for: 1 hour
  - Public key embedded

Private key: Never exposed to application; only used by SPIRE Agent to sign requests
```

**Application authenticates to another service using its SVID:**

```python
# Instead of: api_key = os.environ['API_KEY']
# Do this: cert = /spire/agent.sock (SPIRE Agent socket)

import socket
import ssl

# mTLS connection using SVID
context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.load_verify_locations('/spire/certs/ca.pem')

conn = socket.create_connection(('other-service:8443',))
conn_ssl = context.wrap_socket(conn, server_hostname='other-service')

# Mutual TLS: both client and server present certificates (SVIDs)
# Receiving service validates the client's SVID to authorize the request
conn_ssl.send(b'request data')
```

**Key advantages:**
- No keys in memory; identity is cryptographic.
- Automatic rotation at OS level.
- Works across clouds (SPIRE-to-SPIRE federation).
- Zero-trust networking: every connection authenticated and encrypted.

**Key pitfalls:**
- Significant infrastructure investment (SPIRE cluster, certificate rotation, mTLS everywhere).
- Requires applications to support mTLS (not all legacy systems do).
- Organizational complexity: teams must adopt a new security model.

## Code Examples

### Bad Practice (Vulnerable)

```python
# Database credentials hardcoded in environment variables
import os
import psycopg2

# Credentials persist in K8s Secret (stored unencrypted by default)
db_user = os.environ['DB_USER']  # 'app_user'
db_password = os.environ['DB_PASSWORD']  # 'super_secret_password_123'
db_host = os.environ['DB_HOST']  # 'postgres.example.com'

connection = psycopg2.connect(
    host=db_host,
    user=db_user,
    password=db_password,
    database='production'
)

# This credential:
# - Never changes (static)
# - Is reused across all application replicas
# - Appears in logs if connection fails
# - Is visible in: K8s Secret, pod environment, crash dumps, version control (if ever checked in)
# - Cannot be revoked without redeploying the application
# - Leaking it gives attacker permanent access until manual rotation
```

**Problems:**
- Credential visible in multiple places (K8s API, logs, crashes).
- Single credential shared by all application instances; one leak compromises the whole service.
- No audit trail per-connection.
- Manual rotation is operator-intensive and error-prone.

### Good Practice (Secure)

The recommended pattern is **Vault Agent sidecar injection**, where a sidecar container manages credential lifecycle and injects secrets into the application without requiring application code changes to handle Vault APIs.

#### Vault Agent Sidecar Architecture

```mermaid
graph TB
    subgraph "Pod Startup"
        AGENT["Vault Agent Sidecar"]
        AUTH["K8s ServiceAccount Token"]
    end

    subgraph "Authentication & Credential Issuance"
        VA["Agent authenticates to Vault"]
        VAULT["Vault creates ephemeral DB user"]
        RENDER["Agent renders /app/secrets/db.json"]
    end

    subgraph "Application Runtime"
        APP["Application<br/>(reads db.json)"]
        DB["Execute queries using<br/>ephemeral credentials"]
    end

    subgraph "Lifecycle Management"
        RENEW["Agent: renew before TTL expires"]
        REVOKE["Vault: auto-revoke on expiry"]
    end

    AGENT -->|read| AUTH
    AUTH -->|authenticate| VA
    VA -->|request cred| VAULT
    VAULT -->|return username, password, TTL| RENDER

    RENDER -->|inject into shared volume| APP
    APP -->|connect| DB

    DB -->|30 min later| RENEW
    RENEW -->|request new credential| VAULT
    VAULT -->|issue new cred| APP

    APP -->|60 min after first issue| REVOKE
    REVOKE -->|DROP database user| DB

    style AGENT fill:#4CAF50,stroke:#2E7D32,color:#fff
    style VAULT fill:#2196F3,stroke:#1565C0,color:#fff
    style APP fill:#4CAF50,stroke:#2E7D32,color:#fff
    style REVOKE fill:#FF9800,stroke:#E65100,color:#fff
```

**Flow explanation:**
1. Vault Agent reads the pod's ServiceAccount token (no stored credential needed).
2. Agent authenticates to Vault using the K8s auth method.
3. Agent requests a database credential (triggers Vault to create an ephemeral PostgreSQL user).
4. Agent renders a JSON file with the credential using a template.
5. Application reads the file and connects to the database.
6. Before the credential expires, Agent transparently renews it and updates the file.
7. Upon TTL expiry, Vault automatically revokes the database user.

This pattern enables **zero-downtime credential rotation** without application changes.

```python
# Use Vault Agent to inject a dynamic credential
import psycopg2
import json
import time
import threading

class VaultDynamicCredential:
    def __init__(self, credential_path='/app/secrets/database.json'):
        self.credential_path = credential_path
        self.credential = None
        self.lock = threading.Lock()
        self._load_credential()
        # Monitor credential file for changes (Vault Agent updates it before expiry)
        self._watch_credential()
    
    def _load_credential(self):
        with self.lock:
            try:
                with open(self.credential_path, 'r') as f:
                    self.credential = json.load(f)
                    self.credential['loaded_at'] = time.time()
            except Exception as e:
                raise RuntimeError(f"Failed to load credential from {self.credential_path}: {e}")
    
    def _watch_credential(self):
        """Monitor file for changes (Vault Agent updates it before expiry)"""
        import os
        last_mtime = os.path.getmtime(self.credential_path)
        
        def watch():
            nonlocal last_mtime
            while True:
                try:
                    current_mtime = os.path.getmtime(self.credential_path)
                    if current_mtime > last_mtime:
                        self._load_credential()
                        last_mtime = current_mtime
                except:
                    pass
                time.sleep(5)
        
        thread = threading.Thread(target=watch, daemon=True)
        thread.start()
    
    def get_connection(self):
        """Return a fresh database connection with current credential"""
        with self.lock:
            if not self.credential:
                raise RuntimeError("No credential loaded")
            
            return psycopg2.connect(
                host=self.credential['host'],
                user=self.credential['username'],
                password=self.credential['password'],  # Auto-refreshed by Vault Agent
                database=self.credential['database'],
                connect_timeout=5
            )

# Vault Agent injects the credential as a JSON file before the pod starts
# Agent continuously updates the file before credential expiry
# Application always reads the current credential; no hardcoding needed

vault_cred = VaultDynamicCredential()
connection = vault_cred.get_connection()
cursor = connection.cursor()
cursor.execute("SELECT NOW()")
```

**Vault Agent Configuration (sidecar container):**

```hcl
# /etc/vault/agent-config.hcl
vault {
  address = "http://vault.vault.svc:8200"
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

template {
  source = "/etc/vault/templates/database.json.tpl"
  destination = "/app/secrets/database.json"
  command = "kill -HUP `cat /app/app.pid`"  # Signal app to reload
  change_mode = "signal"
  perms = "0600"
}
```

**Template file:**

```
{{- with secret "database/creds/app-role" -}}
{
  "host": "postgres.example.com",
  "database": "production",
  "username": "{{ .Data.data.username }}",
  "password": "{{ .Data.data.password }}",
  "expiration": "{{ .Data.data.expiration }}"
}
{{- end }}
```

**Key improvements:**
- Credential never hardcoded; generated and injected by Vault Agent.
- Unique per instance: each pod gets a unique, per-request credential.
- Automatic renewal: Vault Agent refreshes the file before expiry.
- Auditable: every credential generation logged in Vault.
- Blast radius bounded: leaked credential valid for at most 1 hour (the TTL).
- Zero-downtime rotation: application signals reload, new credential in place.

### AWS Example with IRSA

```python
# Instead of storing AWS keys, pod uses its ServiceAccount token
import boto3

# boto3 automatically uses IRSA; no keys to manage
s3 = boto3.client('s3')

# boto3 internally:
# 1. Reads the pod's ServiceAccount token from /var/run/secrets/kubernetes.io/serviceaccount/token
# 2. Calls AWS STS AssumeRoleWithWebIdentity with the token
# 3. Receives a short-lived access key + session token
# 4. Automatically renews before expiry

buckets = s3.list_buckets()

# The underlying credentials are:
# - Generated by AWS STS
# - Valid for 1 hour by default
# - Automatically renewed by boto3 SDKs
# - Revoked by AWS upon expiry
```

**Kubernetes setup:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/app-eks-role

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      serviceAccountName: app-sa
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: AWS_ROLE_ARN
          value: arn:aws:iam::123456789012:role/app-eks-role
        - name: AWS_WEB_IDENTITY_TOKEN_FILE
          value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

## Benefits

- **Bounded Blast Radius:** If a credential is leaked, attackers have minutes-to-hours of access, not indefinite. A 1-hour TTL means automatic remediation; no human intervention required.
- **Operational Simplicity:** Automatic rotation eliminates the coordination burden. Teams no longer postpone rotation; it happens automatically.
- **Audit Non-Repudiation:** Each credential generation and use is logged, enabling compliance teams to prove who accessed what, when.
- **Zero-Trust Enabler:** Cryptographic workload identity (Kubernetes auth, SPIFFE/SVID) replaces shared secrets, making zero-trust architectures feasible.
- **Incident Response Speed:** A leaked credential can be revoked instantly without redeploying applications. Incident response goes from hours to seconds.
- **Multi-Tenancy & Isolation:** Unique per-instance credentials enable fine-grained access control. Database admins can see which app replica accessed which table.

## Common Pitfalls

### Pitfall 1: TTL Too Short

**Problem:** If TTL is 5 minutes and the application makes a 10-minute-long database query, the credential expires mid-query.

```python
# Query takes 15 minutes
cursor.execute("SELECT * FROM large_table WHERE ... ")  # 15 minutes
# At 5-minute mark, Vault revokes the credential
# Query fails with "authentication failed"
```

**Solution:** Set TTL to the longest expected operation time, with a grace period. Use `max_ttl` for upper bounds:

```hcl
vault write database/roles/app \
  ...
  default_ttl="30m"    # 30 minutes for normal operations
  max_ttl="2h"         # Upper bound to prevent indefinite access
```

### Pitfall 2: Credential Expiry During Load Spike

**Problem:** Under heavy load, credential renewal requests queue up and fail, leaving the application with expired credentials.

**Solution:** Implement jittered renewal, start refreshing before TTL approaches:

```python
# Refresh credential when it has 30% of TTL remaining
# This avoids thundering herd on exact expiry
remaining_ttl = credential['expiration'] - time.time()
refresh_threshold = remaining_ttl * 0.3

if remaining_ttl < refresh_threshold:
    vault_cred._load_credential()
```

### Pitfall 3: Confusing Token TTL with Secret TTL

**Problem:** Vault issues two TTLs:
1. **Token TTL:** How long the application's Vault auth token is valid.
2. **Secret TTL:** How long the issued database credential is valid.

```hcl
# This auth token (from K8s auth) is valid for 1 hour
vault write auth/kubernetes/role/app ...
  ttl="1h"

# This database credential (issued by the auth token) is valid for 1 hour
# They can be different!
vault write database/roles/app ...
  default_ttl="5m"    # Database credential expires after 5 minutes
```

If the token expires before the secret, the application cannot renew the secret. **Ensure token TTL > secret TTL + grace period.**

### Pitfall 4: Secrets in Logs, Crash Dumps, Memory

**Problem:** Even dynamic credentials should never appear in logs.

```python
# WRONG: credential leaks into logs
try:
    db.connect(username, password)
except Exception as e:
    logger.error(f"Failed to connect with {username}:{password}")  # NEVER DO THIS

# RIGHT: credential is never logged
try:
    db.connect(username, password)
except Exception as e:
    logger.error(f"Failed to connect to database")  # Don't log secrets
```

**Solution:** Scrub logs, use structured logging, ensure crash dumps are redacted.

### Pitfall 5: No Monitoring of Lease Renewal Failures

**Problem:** Silent failures. Vault Agent fails to renew; application continues until credential expires, then crashes.

**Solution:** Monitor the credential file's modification time; alert if it hasn't been updated in 2x the TTL:

```python
import os
import time

credential_path = '/app/secrets/database.json'
last_modified = os.path.getmtime(credential_path)
age_seconds = time.time() - last_modified

if age_seconds > 120:  # TTL is 60 seconds, alert if no refresh in 120 seconds
    raise RuntimeError("Credential not refreshed; renewal may have failed")
```

## When to Apply

- **Always:** Database credentials, cloud IAM, inter-service authentication in production. No exceptions.
- **Recommended:** CI/CD pipeline secrets (GitHub Actions, GitLab CI), third-party API integrations.
- **Consider:** Local development environments (can use static for speed, but teach developers the production pattern).

## Framework/Language-Specific Guidance

### Python

```python
# Using hvac (Vault SDK)
import hvac

client = hvac.Client(url='http://vault:8200', token=auth_token)
secret = client.secrets.database.generate_credentials(
    name='app-role',
    mount_point='database'
)
```

### JavaScript/Node.js

```javascript
// Using node-vault
const vault = require('node-vault')({
  endpoint: 'http://vault:8200',
  token: authToken
});

const secret = await vault.read('database/creds/app-role');
```

### Java

```java
// Spring Vault integration
@Configuration
@EnableVaultRepositories
public class VaultConfig extends AbstractVaultConfiguration {
    @Bean
    public VaultTemplate vaultTemplate() {
        return new VaultTemplate(vaultEndpoint(), vaultAuthentication());
    }
}

// In controller
@Autowired
private VaultTemplate vaultTemplate;

public void connectDB() {
    VaultResponseSupport<Map<String, Object>> response = 
        vaultTemplate.read("database/creds/app-role", Map.class);
    String username = response.getData().get("username");
    String password = response.getData().get("password");
}
```

## Verification & Testing

### Manual Checks

1. Verify no static credentials in environment variables:
   ```bash
   kubectl exec app-pod -- env | grep -i password  # Should be empty
   ```

2. Check Vault Agent is running and updating the credential file:
   ```bash
   kubectl exec app-pod -- ls -la /app/secrets/database.json
   # Verify mtime is recent (within last 30 minutes for 1-hour TTL)
   ```

3. Trace a request to confirm dynamic credential usage:
   ```bash
   vault audit list
   vault read auth/token/lookup/<token>  # Check TTL
   ```

### Automated Testing

```python
# Test that credential renewal happens before expiry
def test_credential_refresh():
    cred = VaultDynamicCredential()
    initial_password = cred.credential['password']
    
    # Wait for Vault Agent to refresh
    time.sleep(61)  # TTL is 60 seconds
    
    cred._load_credential()
    new_password = cred.credential['password']
    
    # Password should have changed (new credential issued)
    assert new_password != initial_password
```

### Security Scanning

- **Gitleaks:** Scan for hardcoded AWS keys, database passwords, API keys.
  ```bash
  gitleaks detect --source .
  ```
- **Vault Audit Review:** Ensure all secret access is logged and reviewed.
  ```bash
  vault audit list
  ```
- **Pod Security Policy:** Enforce no hardcoded credentials in pod templates.
  ```bash
  kubectl get pods -o jsonpath='{.items[*].spec.containers[*].env}' | grep -i password
  ```

## Related Best Practices

- [Never Hardcode Credentials](#) — The foundational security practice.
- [Secrets Encryption at Rest](#) — Protecting secrets in storage (K8s encryption, Vault transit engine).
- [Least Privilege Access Control](#) — Database roles with minimal permissions (example: `readonly` role for queries only).
- [Audit and Logging](#) — Comprehensive logging of credential access and usage.

## Standards & Compliance

- **OWASP A07:2021 — Identification and Authentication Failures:** Dynamic secrets directly mitigate this risk.
- **CWE-798 — Use of Hard-Coded Credentials:** Dynamic credentials eliminate this weakness entirely.
- **NIST SP 800-57 — Recommendation for Key Management:** Aligns with lifecycle principles (generation, distribution, storage, use, destruction).
- **NIST SP 800-204B — Security Recommendations for Federal Information System and Organizations:** Endorses dynamic, short-lived credentials.
- **PCI DSS Requirement 8.2.3:** Passwords must be protected from unauthorized access and changed regularly. Dynamic secrets exceed this requirement.
- **HIPAA Security Rule 45 CFR 164.312(a)(2):** Unique user identification and user activity logging — dynamic secrets enable both.
- **ISO 27001 A.10.1.1:** Cryptographic controls — short-lived credentials + automatic rotation.

## Further Reading

- [HashiCorp Vault Secrets Engines](https://www.vaultproject.io/docs/secrets)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [NIST SP 800-57: Recommendation for Key Management](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-57p1r5.pdf)
- [AWS IRSA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [SPIFFE/SPIRE Project](https://spiffe.io/)

## Case Studies

### Incident Example: Capital One Data Breach (2019)

Capital One suffered a breach exposing 100M+ credit card records. One vector was a compromised AWS IAM access key. If the company had used AWS STS short-lived credentials instead of long-lived IAM keys, the attacker's window of access would have been bounded to the credential's TTL (typically 1 hour), and the breach would have been self-contained.

**Lesson:** Long-lived credentials in the cloud are particularly dangerous. Always use cloud IAM's short-lived credential mechanisms (STS, Workload Identity, Managed Identity).

### Success Story: Square's Secret Management Overhaul

Square migrated from static, shared credentials to Vault-powered dynamic secrets. The transition reduced mean-time-to-remediation (MTTR) for credential leaks from 72 hours to <5 minutes (automatic revocation). Incident severity decreased because leaked credentials expired within the incident response window, making compromise recovery much faster.

**Result:** Significant reduction in credential-related incidents and faster incident resolution.

## Tags

`secrets-management` `dynamic-credentials` `vault` `aws-sts` `kubernetes` `spiffe` `zero-trust` `rotation` `compliance` `incident-response` `architecture`

---

**Contributed by:** Rodrigo Abreu
**Last Updated:** 2026-06-06
**Difficulty Level:** Advanced
**Impact:** High
