Perfect ✅ — here’s the **complete single Markdown file**, containing everything:

* Full explanations of **Docker Content Trust (DCT)**
* Key structure & roles
* Backup & rotation steps
* Full **CI/CD integration for Jenkins and Azure DevOps**
* And a **visual diagram (ASCII + text-art)** showing how DCT and CI/CD signing flow works.

You can copy this entire block directly into your `README.md` or internal documentation — it’s self-contained, complete, and presentation-ready.

---

````markdown
# 🐳 Docker Content Trust (DCT) & GPG Key Signing — Deep Dive + CI/CD Integration

---

## 📘 Overview

**Docker Content Trust (DCT)** ensures Docker images are **cryptographically signed and verified** before they are used.  
It provides both **integrity** (the image hasn’t been altered) and **authenticity** (it came from the trusted source).

Under the hood, DCT uses the **Notary service**, which implements [The Update Framework (TUF)](https://theupdateframework.io/), to manage signing keys, signatures, and verification metadata.

When DCT is enabled, Docker signs image tags using local private keys and uploads **signed metadata** to a **Notary registry** (e.g., Docker Hub’s built-in Notary).

---

## ⚙️ Enabling Docker Content Trust

```bash
export DOCKER_CONTENT_TRUST=1
````

With this variable set, all subsequent `docker push` and `docker pull` commands will:

* ✅ **Sign** images on push.
* ✅ **Verify** signatures on pull.

If an image is unsigned or has invalid trust metadata — Docker refuses the operation.

---

## 🧩 Key Components of Docker Content Trust

When you push a signed image, Docker automatically creates and manages several key files.
Let’s understand each:

### 🔑 1. Root Key

* **Location:** `~/.docker/trust/private/<root-key-id>.key`
* **Purpose:** The **root of trust** for all your repositories. It identifies you (or your organization) as the owner of the signing trust.
* **Notes:**

  * Created once and reused across all repos.
  * Must be backed up securely (offline or encrypted).
  * If lost, your entire trust chain must be re-established.

---

### 🔑 2. Repository Key

* **Location:** `~/.docker/trust/private/<repo-key-id>.key`
* **Purpose:** Used to **sign specific image tags** within a repository (e.g., `myrepo/myapp`).
* **Notes:**

  * Each repository has its own key.
  * Can be rotated independently if compromised.
  * Only affects its respective repo.

---

### 🧾 3. Signed Metadata Files

Stored in:

```
~/.docker/trust/tuf/<registry>/<namespace>/<repo>/metadata/
```

| File             | Purpose                                              |
| ---------------- | ---------------------------------------------------- |
| `root.json`      | Contains your root public key (trust root info).     |
| `targets.json`   | Maps image tags to signed image digests.             |
| `snapshot.json`  | Tracks metadata version consistency.                 |
| `timestamp.json` | Adds expiration/freshness to prevent replay attacks. |

---

### 🧾 4. Signatures

Each image tag you push gets signed:

```
sign(tag_digest, repo_private_key) → signature
```

The **signature** and metadata are uploaded to Docker Hub’s Notary server.
Your **private keys never leave your local machine**.

---

### 📂 5. Trust Directory Structure

Example layout:

```
~/.docker/trust/
├── private/
│   ├── <root-key-id>.key
│   └── <repo-key-id>.key
└── tuf/
    └── docker.io/
        └── myrepo/
            └── myapp/
                ├── metadata/
                │   ├── root.json
                │   ├── targets.json
                │   ├── snapshot.json
                │   └── timestamp.json
                └── trusted_certificates/
```

---

## ☁️ What Gets Uploaded to Docker Hub

| Item                                     | Local | Uploaded to Hub |
| ---------------------------------------- | ----- | --------------- |
| Root Private Key                         | ✅     | ❌               |
| Repo Private Key                         | ✅     | ❌               |
| Public Keys                              | ✅     | ✅               |
| Metadata (root.json, targets.json, etc.) | ✅     | ✅               |
| Signatures                               | ✅     | ✅               |

Only **public keys and signed metadata** are uploaded.
Private keys always stay local.

---

## 🧠 Verification Process (During Image Pull)

When a user runs:

```bash
export DOCKER_CONTENT_TRUST=1
docker pull myrepo/myapp:1.0
```

Docker automatically:

1. Contacts Docker Hub’s Notary service.
2. Downloads trusted metadata (`root.json`, `targets.json`, etc.).
3. Verifies:

   * The metadata chain (root → targets → snapshot → timestamp).
   * The image digest matches the signed tag.
4. ✅ Pulls the image if valid, ❌ fails if invalid.

---

## 🔍 Analogy: GPG vs Docker Content Trust

| Concept              | GPG                   | Docker Content Trust         |
| -------------------- | --------------------- | ---------------------------- |
| Signing              | Manual (`gpg --sign`) | Automatic (on `docker push`) |
| Key Distribution     | Manual                | Notary handles it            |
| Verification         | `gpg --verify`        | `docker pull` (auto)         |
| Private Key Location | `~/.gnupg/`           | `~/.docker/trust/private/`   |
| Public Key Location  | Shared manually       | Stored on Docker Hub         |

---

## 🔐 Security Summary

| Component    | Description             | Best Practice                  |
| ------------ | ----------------------- | ------------------------------ |
| Root Key     | Global signing identity | Back up offline                |
| Repo Key     | Signs image tags        | Rotate regularly               |
| Private Keys | Stored locally          | Encrypt with strong passphrase |
| Metadata     | Public data             | Safe to upload                 |
| Public Keys  | Used for verification   | Auto-managed by Notary         |

---

## 🧠 TL;DR Summary

| Component | Stored Where | Type       | Purpose               |
| --------- | ------------ | ---------- | --------------------- |
| Root Key  | Local        | 🔒 Private | Root of all trust     |
| Repo Key  | Local        | 🔒 Private | Signs tags            |
| Metadata  | Local & Hub  | 🌐 Public  | Maps tags ↔ digests   |
| Signature | Docker Hub   | 🌐 Public  | Verifies authenticity |

---

## 🔒 Best Practices

1. **Always back up your root key.**
2. **Use strong passphrases.**
3. **Never share your trust directory.**
4. **Rotate repository keys quarterly.**
5. **Use separate repos per environment.**
6. **Automate signing safely in CI/CD pipelines.**

---

## 🔁 Key Rotation & Recovery Guide

### Step 1: Back Up Existing Keys

```bash
tar czf docker-trust-backup.tar.gz ~/.docker/trust/private/
gpg --symmetric --cipher-algo AES256 docker-trust-backup.tar.gz
```

> 🔐 Store the `.gpg` file securely (Vault, S3 with KMS, etc.)

---

### Step 2: Rotate Repository Key

```bash
notary key rotate docker.io/myrepo/myapp snapshot -r
docker push myrepo/myapp:1.1
```

---

### Step 3: Rotate Root Key (Rare)

```bash
notary key rotate docker.io/myrepo/myapp root -r
```

> ⚠️ Clients must re-verify with the new trust root.

---

### Step 4: Restore Keys

```bash
gpg --decrypt docker-trust-backup.tar.gz.gpg > docker-trust-backup.tar.gz
tar xzf docker-trust-backup.tar.gz -C ~/
```

---

### Step 5: Verify Trust Status

```bash
docker trust inspect --pretty myrepo/myapp
```

Example output:

```
Signatures for myrepo/myapp
SIGNED TAG   DIGEST                                     SIGNERS
1.0          81b28c3d267d2ac88cc0f89348e22bba...   (Repo Admin)
```

---

## ⚙️ CI/CD Signing Setup — Jenkins & Azure DevOps Pipelines

When using DCT in CI/CD:

* Never hardcode keys.
* Store encrypted trust archives in credentials.
* Decrypt only during the signing stage.
* Clean up afterward.

---

### 🧩 Jenkins Pipeline Integration

#### 1️⃣ Prerequisites

* Jenkins agent with Docker installed.
* Jenkins credentials:

  * `DOCKER_TRUST_KEY_ARCHIVE` → `.tar.gz.gpg` file.
  * `DOCKER_TRUST_KEY_PASSPHRASE` → key password.
  * `DOCKER_HUB_CREDS` → Docker Hub credentials.

#### Jenkinsfile Example

```groovy
pipeline {
  agent any
  environment {
    DOCKER_CONTENT_TRUST = "1"
    TRUST_KEY_DIR = "${WORKSPACE}/.docker/trust"
  }
  stages {
    stage('Prepare Trust Keys') {
      steps {
        withCredentials([
          file(credentialsId: 'DOCKER_TRUST_KEY_ARCHIVE', variable: 'TRUST_ARCHIVE'),
          string(credentialsId: 'DOCKER_TRUST_KEY_PASSPHRASE', variable: 'KEY_PASS')
        ]) {
          sh '''
            mkdir -p ~/.docker
            gpg --batch --yes --passphrase "$KEY_PASS" --decrypt "$TRUST_ARCHIVE" > docker-trust.tar.gz
            tar xzf docker-trust.tar.gz -C ~/.docker/
            echo "✅ DCT keys restored."
          '''
        }
      }
    }

    stage('Build & Sign Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_CREDS', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker build -t myrepo/myapp:${BUILD_NUMBER} .
            docker push myrepo/myapp:${BUILD_NUMBER}
            echo "✅ Image signed and pushed."
          '''
        }
      }
    }

    stage('Cleanup') {
      steps {
        sh '''
          rm -rf ~/.docker/trust
          echo "🧹 Cleaned up trust keys."
        '''
      }
    }
  }
}
```

---

### 🧩 Azure DevOps Integration

#### 1️⃣ Prerequisites

* Docker installed on agent.
* Secrets:

  * `docker-trust-backup.tar.gz.gpg` → secure file.
  * `DOCKER_TRUST_PASSPHRASE`, `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` → secret variables.

#### azure-pipelines.yml Example

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  DOCKER_CONTENT_TRUST: 1

steps:
  - task: DownloadSecureFile@1
    name: trustKeys
    inputs:
      secureFile: 'docker-trust-backup.tar.gz.gpg'

  - script: |
      mkdir -p ~/.docker
      gpg --batch --yes --passphrase "$(DOCKER_TRUST_PASSPHRASE)" \
          --decrypt $(trustKeys.secureFilePath) > docker-trust.tar.gz
      tar xzf docker-trust.tar.gz -C ~/.docker/
      echo "✅ DCT keys restored."
    displayName: 'Restore Trust Keys'

  - script: |
      echo "$(DOCKERHUB_TOKEN)" | docker login -u "$(DOCKERHUB_USERNAME)" --password-stdin
      docker build -t myrepo/myapp:$(Build.BuildId) .
      docker push myrepo/myapp:$(Build.BuildId)
      echo "✅ Image signed and pushed."
    displayName: 'Build & Sign Image'

  - script: |
      rm -rf ~/.docker/trust
      echo "🧹 Cleaned up trust keys."
    displayName: 'Cleanup'
```

---

## 🧩 CI/CD Flow Diagram

```
                          ┌─────────────────────────────────────────┐
                          │           Jenkins / Azure CI/CD         │
                          │─────────────────────────────────────────│
                          │ 1️⃣ Download encrypted trust archive (.gpg) │
                          │ 2️⃣ Decrypt → ~/.docker/trust                │
                          │ 3️⃣ export DOCKER_CONTENT_TRUST=1           │
                          │ 4️⃣ docker build & push → signed image      │
                          │ 5️⃣ Upload signature → Docker Hub Notary    │
                          │ 6️⃣ Delete trust keys (cleanup)             │
                          └──────────────┬───────────────────────────┘
                                         │
                                         ▼
                         ┌────────────────────────────────────┐
                         │         Docker Hub / Notary         │
                         │────────────────────────────────────│
                         │ • Stores public keys & metadata     │
                         │ • Verifies image signatures         │
                         │ • Provides trust data to clients    │
                         └──────────────┬─────────────────────┘
                                         │
                                         ▼
                         ┌────────────────────────────────────┐
                         │     User Pulls Image (with DCT)     │
                         │────────────────────────────────────│
                         │ • Downloads trust metadata          │
                         │ • Verifies image signature          │
                         │ • Pull succeeds only if valid ✅     │
                         └────────────────────────────────────┘
```

---

## ✅ Final Security Recommendations

| Category           | Recommendation                                                        |
| ------------------ | --------------------------------------------------------------------- |
| **Key Management** | Encrypt `.tar.gz` with GPG; store in Jenkins or Azure secure storage. |
| **Isolation**      | Use dedicated build agents for signing.                               |
| **Logging**        | Log signed tag + digest for audit.                                    |
| **Key Rotation**   | Rotate repo keys quarterly or after team changes.                     |
| **Backup Policy**  | Maintain one offline encrypted copy of root key.                      |
| **Enforcement**    | Set `DOCKER_CONTENT_TRUST=1` in all deploy pipelines.                 |

---

> **In summary:**
> Docker Content Trust provides **built-in cryptographic assurance** that your images are signed by an authorized identity and haven’t been tampered with.
> Using secure CI/CD integration ensures every image deployed in your environments is **authentic, trusted, and verifiable**.

```

---

Would you like me to generate an **actual visual diagram image (PNG/SVG)** from that ASCII diagram (with Docker + Jenkins + Azure + Notary icons) for your README? It’ll look professional for documentation or audit submissions.
```
