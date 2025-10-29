Perfect ✅ — here’s a clean, **Markdown-only file** that includes **only Sections 3 & 4** (GitHub commit signing + why you shouldn’t reuse Docker keys).
Everything is complete, deeply explained, and formatted for direct copy-paste into your docs repo.

---

```markdown
# 🔐 Git Commit Signing (GPG) and Key Management Best Practices

## 🧩 1. Git Commit Signing (GPG)

When you see **“Verified”** next to a commit on GitHub, it means the commit was **digitally signed** using a **GPG or SSH key** that GitHub recognizes as belonging to that user.

Example:

```

✔️ Verified
Commit by Rahul [rahul@example.com](mailto:rahul@example.com)

```

Unsigned commits simply show:

```

Commit by Rahul [rahul@example.com](mailto:rahul@example.com)

````

---

### 🔐 What Happens When You Sign a Commit

1. Git hashes the commit metadata and contents.
2. The hash is **digitally signed** with your **private GPG key**.
3. The signature is embedded into the commit object.
4. GitHub verifies the commit using your **public GPG key**, which you’ve added to your GitHub account.

If everything matches — your key, your email, and your identity — GitHub displays the **“✔️ Verified”** badge.

---

### 🧰 How to Set Up GPG-Signed Commits

1. **Generate a GPG key:**
   ```bash
   gpg --full-generate-key
````

* Choose RSA and RSA (option 1)
* Use key size **4096**
* Use your **GitHub account email** when prompted
* Set an expiry if desired (e.g., 1y)

2. **List your GPG keys:**

   ```bash
   gpg --list-secret-keys --keyid-format=long
   ```

   Example output:

   ```
   sec   rsa4096/AB12CD34EF56GH78 2025-10-29 [SC]
   uid           [ultimate] Rahul DevOps <rahul@example.com>
   ssb   rsa4096/1122AABBCCDDEEFF 2025-10-29 [E]
   ```

   Here, `AB12CD34EF56GH78` is your **GPG key ID**.

3. **Configure Git to use this key:**

   ```bash
   git config --global user.signingkey AB12CD34EF56GH78
   git config --global commit.gpgsign true
   ```

   This ensures every commit you make will be signed automatically.

4. **Export your public key for GitHub:**

   ```bash
   gpg --armor --export AB12CD34EF56GH78
   ```

   Copy the full block that looks like this:

   ```
   -----BEGIN PGP PUBLIC KEY BLOCK-----
   ...
   -----END PGP PUBLIC KEY BLOCK-----
   ```

5. **Add the key to GitHub:**

   * Go to **GitHub → Settings → SSH and GPG keys → New GPG Key**
   * Paste the exported public key
   * Save it

6. **Make a signed commit:**

   ```bash
   git commit -S -m "feat: added GPG-signed commit"
   ```

   Output:

   ```
   gpg: Signature made Wed 29 Oct 2025
   gpg: Good signature from "Rahul DevOps <rahul@example.com>"
   ```

   Push your changes — you’ll now see a **“Verified”** badge next to your commit on GitHub.

---

### 🧠 Git Signing vs Docker Image Signing

| Concept                | Git Commit Signing      | Docker Content Trust                |
| ---------------------- | ----------------------- | ----------------------------------- |
| Signed Object          | Commit / Tag            | Docker Image                        |
| Signing Mechanism      | GPG or SSH              | Repo & Root Keys                    |
| Verification Location  | GitHub                  | Docker Notary                       |
| Default Behavior       | Manual (opt-in)         | Optional via `DOCKER_CONTENT_TRUST` |
| Identity Binding       | Email-based             | Repository-based                    |
| Verification Indicator | ✔️ “Verified” on GitHub | Successful DCT pull                 |

Both systems rely on **public key cryptography**, but the signing scopes and verification domains are completely different.

---

## 🧩 2. Why You Shouldn’t Use the Same GPG Key for Docker and GitHub

Even though both Docker and GitHub rely on **GPG keys for signing and verification**, their **trust boundaries**, **metadata**, and **intended ownership** are different.
Reusing the same key can lead to security, audit, and identity issues.

---

### ⚠️ Why Reusing Keys Is a Bad Practice

| Risk                                | Description                                                                                                                                                                             |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 🔒 **Security Boundary Violation**  | Docker signing keys are typically stored in CI/CD systems or build pipelines, while GitHub signing keys belong to individual users. Mixing them widens the blast radius if compromised. |
| 🧾 **Different Metadata & Formats** | Docker Notary uses delegation roles (root, repo, targets), while GitHub only uses OpenPGP UIDs (name/email). The same key cannot properly represent both contexts.                      |
| 🧠 **Lifecycle Confusion**          | Rotating or revoking one key may unintentionally break the other use case. It complicates key management and compliance.                                                                |
| 👤 **Identity Mismatch**            | GitHub verifies based on the key’s UID email. Docker doesn’t — it’s repository-bound. The same key won’t pass GitHub verification if email metadata doesn’t match.                      |

---

### 🧰 Best Practice — Use Separate Keys

| Purpose                  | Recommended Key  | Stored At             | Managed By             |
| ------------------------ | ---------------- | --------------------- | ---------------------- |
| 🧑‍💻 Git Commit Signing | Personal GPG key | Developer machine     | Developer              |
| 🐳 Docker Image Signing  | Root & Repo keys | CI/CD or secure vault | DevOps / Security team |

By maintaining separate keys:

* Each key serves a **single, clear purpose**.
* **Compromise isolation** is guaranteed.
* **Auditing and rotation** are simpler and cleaner.

---

### 🧠 Technical Note

It is **technically possible** to reuse the same GPG key:

```bash
gpg --import docker_signing_key.asc
git config --global user.signingkey <KEY_ID>
```

…but GitHub will verify signatures only if:

* The GPG key’s email matches your GitHub account.
* The key is uploaded to your GitHub account.

Even then, **this is strongly discouraged** in secure or enterprise environments because of key management and identity scoping conflicts.

---

### 🧩 Recommended Architecture

```
        ┌─────────────────────────────┐
        │      Developer Laptop       │
        │  GPG Key (Git commits)      │
        │  Linked to GitHub email     │
        └──────────────┬──────────────┘
                       │
                       ▼
        ┌─────────────────────────────┐
        │         GitHub              │
        │  Verifies commit signature  │
        └─────────────────────────────┘

        ┌─────────────────────────────┐
        │           CI/CD             │
        │  Docker Content Trust Keys  │
        │  (root + repo)              │
        └──────────────┬──────────────┘
                       │
                       ▼
        ┌─────────────────────────────┐
        │     Docker Notary / Hub     │
        │ Verifies image signatures   │
        └─────────────────────────────┘
```

---

### ✅ Summary

| Topic             | GitHub                                       | Docker                        |
| ----------------- | -------------------------------------------- | ----------------------------- |
| Verification Type | GPG or SSH commit signing                    | Docker Content Trust (Notary) |
| Key Ownership     | Personal                                     | Repository / Organization     |
| Default Behavior  | Manual                                       | Optional                      |
| Reusing Keys      | Technically possible but **not recommended** |                               |
| Best Practice     | Separate keys for code and image signing     |                               |
| Identity Source   | GitHub email                                 | Docker repository trust roles |

---

### 📘 TL;DR

* GitHub commit signing proves **authorship authenticity** using your personal GPG key.
* Docker image signing proves **artifact authenticity** using Content Trust keys.
* Never reuse Docker signing keys for GitHub or vice versa — keep identities and trust scopes separate.
* Store **Git keys** on developer systems, and **Docker keys** in secure CI/CD environments.

```