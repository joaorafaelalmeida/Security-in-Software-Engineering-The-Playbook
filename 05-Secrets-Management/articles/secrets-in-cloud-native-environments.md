# Secrets in Cloud-Native Environments

> A comprehensive guide to managing secrets across containerized, cloud-native, and microservices architectures, covering the complete lifecycle and cloud-specific pitfalls

## Metadata

- **Author(s):** OWASP Community / Cheat Sheets Series
- **Publication:** OWASP Cheat Sheet Series
- **Date:** Continuously updated (last reviewed 2025-11)
- **Reading Time:** 30–45 minutes (comprehensive reference)
- **Level:** Intermediate / Advanced
- **Link:** [Read the full OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

## Summary

The OWASP Secrets Management Cheat Sheet is the definitive practitioner reference for managing secrets in cloud-native and containerized environments. It covers the complete lifecycle — generation, storage, access control, rotation, revocation, and audit — with explicit guidance on Kubernetes-specific patterns, container image handling, and cloud platform integrations (AWS, GCP, Azure).

The cheat sheet goes far beyond the obvious ("don't commit secrets to git") to address the nuanced, often-overlooked pitfalls that plague production systems: K8s Secrets without encryption at rest, secrets baked into container images, application code that caches credentials, lack of audit trails, and the "secret zero" bootstrap problem. It provides decision frameworks for choosing between cloud-native secrets services (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) and self-hosted solutions (HashiCorp Vault, Sealed Secrets).

The guide emphasizes that **not all secrets carry equal risk and should not be managed identically**. Highly sensitive secrets (database root passwords, TLS private keys) require different handling than transient secrets (API tokens, service account tokens), which in turn differ from infrastructure secrets (SSH keys for operational access). Classification is the prerequisite for effective management.

## Key Takeaways

1. **Classify Secrets by Sensitivity & Lifetime**
   - Highly sensitive (root DB passwords, TLS private keys, signing keys): Require encrypted storage, strict RBAC, and frequent audit. Consider external vaults (Vault, AWS Secrets Manager).
   - Transient (API tokens, service account tokens, CI/CD tokens): Short-lived by design; refresh automatically.
   - Infrastructure secrets (SSH keys, bastion credentials): Require bastion/jump-box access patterns, never stored on disk in production.
   - **Implication:** One-size-fits-all approaches fail. Tailor controls to sensitivity.

   #### Secret Classification Matrix

   | **Sensitivity** | **EPHEMERAL** | **SHORT-LIVED** | **LONG-LIVED** | **PERMANENT** |
   |---|---|---|---|---|
   | **HIGH** | — | TLS Keys (1h TTL) | SSH Keys (monthly rotate) | PKI CA Keys |
   | | — | Database Root (if dynamic) | Service Account Keys | Master Keys |
   | | — | **→ Cloud KMS Auto-Unseal** | **→ Vault storage** | **→ Encrypted vault** |
   | **MEDIUM** | CI/CD Tokens (15m) | API Admin Keys (1h) | Bastion SSH Keys (weekly) | — |
   | | GitHub Deploy Keys | Database App Creds | SSH Certificates | |
   | | **→ K8s Secret** | **→ Vault Dynamic** | **→ Vault or ESO** | |
   | **LOW** | Session Tokens (1h) | Registry Tokens (6h) | API Client Keys (monthly) | — |
   | | Auth Codes | Webhook Secrets | Package Keys | |
   | | **→ App Cache** | **→ K8s Secret** | **→ K8s Secret + rotation** | |

   **Key principle:** Secrets in the top-left (high sensitivity + ephemeral) are least risky because they expire quickly and affect few systems. Secrets in the bottom-right (low sensitivity + permanent) are operationally simplest but require careful access control. The middle cells represent the most common production scenarios.

2. **Kubernetes Secrets Are NOT Encrypted by Default**
   - `kubectl create secret` stores secrets in etcd as base64-encoded strings (base64 is encoding, not encryption).
   - Attackers with etcd access or node access can extract secrets trivially.
   - **Solution:** Enable encryption at rest in etcd (`--encryption-provider=aescbc`), use external secret stores (Vault, AWS Secrets Manager, Sealed Secrets), or use the Kubernetes External Secrets Operator (ESO).
   - **Real-world gap:** Many Kubernetes clusters never enable encryption at rest. Audit your cluster now: `kubectl get secret -o json | grep -i "base64"` — if you see base64, you likely lack encryption.

   #### Kubernetes Secrets Attack Surface
   ```mermaid
   graph LR
       subgraph "Threat Actors"
           ATTACKER["External Attacker"]
           INSIDER["Insider / Node Compromise"]
       end

       subgraph "Attack Vectors"
           VEC1["1. etcd<br/>(no encryption at rest)"]
           VEC2["2. Image Layers<br/>(docker history)"]
           VEC3["3. Logs<br/>(app prints secrets)"]
           VEC4["4. Pod Memory<br/>(crash dumps)"]
           VEC5["5. Registry<br/>(pull image)"]
           VEC6["6. Backups<br/>(unencrypted)"]
       end

       subgraph "Result"
           LEAK["Secrets Exposed"]
       end

       ATTACKER -->|exploit| VEC1
       ATTACKER -->|exploit| VEC2
       ATTACKER -->|exploit| VEC3
       ATTACKER -->|exploit| VEC5

       INSIDER -->|exploit| VEC1
       INSIDER -->|exploit| VEC4
       INSIDER -->|exploit| VEC6

       VEC1 -->|lead to| LEAK
       VEC2 -->|lead to| LEAK
       VEC3 -->|lead to| LEAK
       VEC4 -->|lead to| LEAK
       VEC5 -->|lead to| LEAK
       VEC6 -->|lead to| LEAK

       style LEAK fill:#F44336,stroke:#C62828,color:#fff,stroke-width:3px
       style ATTACKER fill:#C62828,stroke:#7F0000,color:#fff
       style INSIDER fill:#C62828,stroke:#7F0000,color:#fff
       style VEC1 fill:#FF9800,stroke:#E65100,color:#fff
       style VEC2 fill:#FF9800,stroke:#E65100,color:#fff
       style VEC3 fill:#FF9800,stroke:#E65100,color:#fff
       style VEC4 fill:#FF9800,stroke:#E65100,color:#fff
       style VEC5 fill:#FF9800,stroke:#E65100,color:#fff
       style VEC6 fill:#FF9800,stroke:#E65100,color:#fff
   ```

   Kubernetes secrets leak through **six primary vectors**: unencrypted etcd, container image layers, application logs, process memory dumps, registry access, and backup files. External attackers exploit vectors 1-3 and 5; insiders additionally exploit memory and backups. A secure secrets architecture must defend against all of them.

3. **The "Secret Zero" Problem — Bootstrap Identity**
   - To authenticate to Vault (or any secrets manager), an application needs an initial credential. Where does that come from?
   - Storing that bootstrap credential defeats the purpose.
   - **Solutions:**
     - **Kubernetes ServiceAccount Auth:** Pod's ServiceAccount token (automatically mounted) proves its identity. No stored credential needed.
     - **AWS IRSA (IAM Roles for Service Accounts):** Pod's ServiceAccount is bound to an IAM role. Pod's AWS SDK automatically assumes the role using its token. No keys stored.
     - **GCP Workload Identity Federation:** Pod's ServiceAccount is bound to a GCP service account. GCP exchanges the pod's token for a GCP credential automatically.
     - **SPIFFE/SPIRE:** Pod receives a cryptographic identity (SVID). No keys; identity is certificate-based.
   - **Implication:** Zero-trust systems require cryptographic workload identity, not credential inheritance.

4. **Centralized Secret Management vs. Distributed Secrets**
   - **Centralized (preferred in production):** Single source of truth (Vault, AWS Secrets Manager). Enables:
     - Consistent rotation and revocation across all services.
     - Audit trail at the secrets manager, not scattered across applications.
     - RBAC and access control enforced at the manager.
   - **Distributed (common anti-pattern):** Each service stores its own secrets (in K8s Secrets, .env files, Dockerfiles). Results in:
     - Rotation requires coordinating across N services; often skipped.
     - Secrets leak through multiple vectors (logs, crash dumps, version control).
     - No central audit trail.
   - **Real-world gap:** Many teams use K8s Secrets as their "vault" — cheaper operationally, but a single breach exposes everything.

5. **Secrets in Container Images Persist in Layer History**
   - If a Dockerfile includes `COPY secrets.txt /app/` or `ENV DB_PASSWORD=secret`, the file/value is baked into the image layer.
   - Layers are stored in the registry. Anyone with pull access to the image (or who steals the image) can extract the layer and recover the secret.
   - `docker history <image>` reveals it. `docker image inspect <image>` shows environment variables.
   - **Solution:** Never bake secrets into images. Inject at runtime (sidecar, init container, Vault Agent).
   - Even after rebuilding, old images remain in registries; secrets persist unless images are deleted.

6. **Rotation Must Be Automated; Manual Rotation Fails at Scale**
   - Manual rotation requires:
     1. Generate new secret.
     2. Update all applications using the old secret.
     3. Wait for all to adopt the new secret.
     4. Revoke the old secret.
   - In a microservices environment (100s of services), this is operationally impossible. One service always falls behind.
   - **Solution:** Automated rotation with graceful re-authentication:
     - Vault's lease renewal: Application refreshes credential before expiry.
     - Cloud IAM's token refresh: SDK automatically requests new STS token before old one expires.
     - Sidecar pattern: Vault Agent or ESO updates files/env vars; application reloads on file change.

7. **Audit Trails Are Non-Negotiable for Compliance**
   - Every secret access must be logged: who (identity), what (secret path), when (timestamp), result (success/failure).
   - Kubernetes audit logs do not capture secret access; you need Vault's audit backend or external logging.
   - **Real-world gap:** Most organizations have no idea who accessed which secrets when.

8. **Kubernetes Admission Controllers & Pod Security Policies**
   - Use admission controllers to prevent pods from mounting secrets carelessly (e.g., preventing `secret/root-ca-key` from being mounted in untrusted pods).
   - Pod Security Standards (successor to PSP) should enforce least privilege on secret access.

9. **Secret Scanning in CI/CD Pipelines**
   - Use Gitleaks, detect-secrets, or Vault Secret Scanning to detect hardcoded secrets in commits before they reach main.
   - Scan must run pre-commit (even better) or in CI.

10. **Environment Variables Are Not a Secret Storage Mechanism**
    - Environment variables:
      - Leak in process listings: `ps aux` reveals them.
      - Leak in logs if application prints them.
      - Leak in crash dumps and core files.
      - Are inherited by child processes.
    - Use files (owned by the app user, read-only) or mounted secrets instead. Even better: let the application read directly from Vault at startup.

## Why This Matters

This cheat sheet is essential reading because:

1. **Security Engineering requires depth.** "Don't hardcode secrets" is the surface level. Real systems deal with Kubernetes, multi-cloud, audit compliance, incident response, and operational handoff. This guide addresses all of them.

2. **Cloud-native architecture amplifies secrets challenges.** Kubernetes, containers, microservices, and orchestration introduce new attack surfaces (etcd, image registries, overlay networks) that traditional on-premises IT departments never faced. Solutions must be cloud-native.

3. **Incidents happen frequently.** GitHub constantly reports credential leaks in public repos. Organizations with weak secrets management suffer breaches from leaked database passwords, API keys, and cloud credentials. This guide helps teams avoid these patterns.

4. **Compliance requires proof.** Financial, healthcare, and regulated industries must demonstrate secrets are not exposed, are rotated, and access is audited. This guide maps controls to HIPAA, PCI DSS, SOC 2, and other frameworks.

## Practical Applications

### For Architects

Use this guide to evaluate secrets management approaches when designing new systems:
- Building a new microservices platform? Reference section on "Centralized vs. Distributed" to justify a Vault investment.
- Migrating to Kubernetes? Reference section on K8s encryption to mandate `--encryption-provider=aescbc` in the cluster spec.
- Multi-cloud environment? Reference section on "Secret Engines" to design a unified interface across AWS, GCP, and Azure.

### For Code Review

Use as a checklist when reviewing pull requests:
- Does the code read secrets from environment variables or files? (Both okay, but understand the trade-offs.)
- Are secrets ever logged or printed? (Must catch this.)
- Does the image Dockerfile bake in secrets? (Must not.)
- Are credentials hardcoded? (Catch with Gitleaks, must not pass.)

### For Ops/Infrastructure

Use as a runbook for secrets system design:
- "How do I set up encryption at rest for K8s Secrets?" Reference the etcd encryption section.
- "How do I rotate database passwords across 50 microservices?" Reference the automated rotation section.
- "How do we audit secret access?" Reference the audit backends section.

### For Compliance/Security Audits

Use to assess organizational posture:
- Do we classify secrets by sensitivity? (If no, big gap.)
- Is K8s encryption at rest enabled? (If no, critical gap.)
- Do we have a centralized secrets manager? (If no, rotation and audit are broken.)
- Are secrets scanned in CI/CD? (If no, preventive controls are weak.)

## Notable Quotes

> "Secrets are for authenticating between systems, not between humans. Humans should use identity providers (Okta, Azure AD)."

> "Encryption at rest is the bare minimum. If your secrets manager is compromised, even encrypted secrets are at risk. Focus on access control and audit."

> "The best secret is no secret. Use cryptographic identity (SPIFFE/SVID) instead of shared secrets whenever possible."

> "Kubernetes Secrets are not encrypted by default. If you use K8s Secrets as your secrets manager in production without encryption at rest, you are running on a false sense of security."

## Related Topics

- HashiCorp Vault (tool for centralized secrets management)
- Kubernetes External Secrets Operator (integrates K8s with external secret managers)
- AWS Secrets Manager, GCP Secret Manager, Azure Key Vault (cloud-native solutions)
- SPIFFE/SPIRE (workload identity framework)
- Gitleaks, detect-secrets (secret scanning for version control)
- PKI & Certificate Management (TLS secrets, certificate rotation)
- Incident Response for Compromised Credentials (breach response playbook)

## Further Reading

- **OWASP Secrets Management Cheat Sheet** — [Full resource](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- **HashiCorp Learn: Vault Architecture** — [Learn Path](https://learn.hashicorp.com/vault)
- **NIST SP 800-57: Key Management Lifecycle** — [NIST Publication](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-57p1r5.pdf)
- **NIST SP 800-204B: Security Recommendations for Service-Oriented Architecture** — [NIST Publication](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-204B.pdf)
- **AWS IRSA (IAM Roles for Service Accounts)** — [AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- **GCP Workload Identity Federation** — [GCP Documentation](https://cloud.google.com/iam/docs/workload-identity-federation)
- **Kubernetes External Secrets Operator (ESO)** — [GitHub Repository](https://github.com/external-secrets/external-secrets)
- **SPIFFE/SPIRE: Secure Production Identity Framework** — [spiffe.io](https://spiffe.io/)

## Critical Analysis

### Strengths

- **Comprehensive & Practical:** The cheat sheet is neither purely academic nor overly simplistic. It covers theory, architecture, and hands-on configurations.
- **Cloud-Native First:** Unlike older security guidance written for monolithic applications, this explicitly addresses Kubernetes, containers, and multi-cloud.
- **Actionable:** The "What You Should Do" sections are concrete (e.g., "Enable `--encryption-provider=aescbc` in kube-apiserver").
- **Continuously Updated:** OWASP community maintains the cheat sheet; recent additions cover newer topics like Workload Identity.

### Limitations

- **Tool-Agnostic Nature:** The guide doesn't recommend specific tools (e.g., Vault vs. AWS Secrets Manager). This is a strength (vendor-neutral) and weakness (you still have to decide).
- **Operational Depth:** While architectural guidance is strong, operational details (Vault HA setup, disaster recovery, incident response) are lighter. Refer to HashiCorp documentation for Vault specifics.
- **Compliance Mapping:** References to PCI DSS, HIPAA, etc., are brief. Compliance teams will need deeper frameworks.

### Age & Relevance

**Very current.** Secret management is an active area (Workload Identity, SPIFFE adoption, encrypted secret scanning in registries). The cheat sheet is updated regularly. Main concepts are stable; implementation tools evolve.

## Tags

`secrets-management` `kubernetes` `cloud-native` `owasp` `vault` `rotation` `encryption-at-rest` `audit-logging` `secret-zero` `compliance` `aws` `gcp` `azure` `microservices`

---

**Contributed by:** Rodrigo Abreu (based on OWASP Secrets Management Cheat Sheet)
**Last Updated:** 2026-06-06
**Relevance:** High — Essential reference for cloud-native and Kubernetes-based organizations. Directly applicable to security architecture, code review, and compliance validation.
