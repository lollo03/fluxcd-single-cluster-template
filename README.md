
# K0s GitOps Cluster Template (FluxCD)

This repository is a template for managing a bare-metal Kubernetes cluster via **GitOps** using **[FluxCD](https://fluxcd.io/)**.

It is explicitly designed to work seamlessly with the **[bootc-homelab-k0s](https://github.com/lollo03/bootc-homelab-k0s)** OS image. Using that image ensures your nodes come pre-baked with the necessary dependencies for this configuration.

**Included Components:**

  * **SOPS:** Secret encryption (Hybrid GPG/Age).
  * **MetalLB:** Bare-metal LoadBalancing.
  * **Nginx Gateway Fabric:** Gateway API implementation.
  * **Local Path Provisioner:** Automatic local storage management.

## 📋 Prerequisites

Before bootstrapping, ensure you have the following:

1.  **The Infrastructure:**
      * Nodes running the **[bootc-homelab-k0s](https://github.com/lollo03/bootc-homelab-k0s)** image.
      * A running k0s cluster accessible via your local `kubeconfig`.
2.  **The CLI Tools:**
      * **[flux](https://fluxcd.io/flux/installation/)**
      * **[kubectl](https://kubernetes.io/docs/tasks/tools/)**
      * **[age](https://github.com/FiloSottile/age)** (for cluster encryption)
      * **[sops](https://github.com/getsops/sops)**
      * **[gpg](https://gnupg.org/)** (for local encryption)


## 📂 Repository Structure

This template follows a modular GitOps structure to separate core infrastructure from user applications.

```text
.
├── apps/                  # User-facing workloads (e.g., echo-server)
├── cluster/               # FluxCD entry point (bootstrapped directory)
│   ├── apps.yaml          # Activates the 'apps' folder
│   ├── infrastructure.yaml# Activates the 'infrastructure' folder
│   └── helmrepos.yaml     # Activates the 'helmrepos' folder
├── helmrepos/             # Source definitions for Helm Charts (Bitnami, etc.)
├── infrastructure/        # System components (MetalLB, Storage, Gateway)
└── .sops.yaml             # Encryption rules for SOPS
```

  * **`cluster/`**: The "Control Center." Flux monitors this directory. It contains `Kustomization` objects that tell Flux to go look at the *infrastructure* and *apps* directories.
  * **`infrastructure/`**: The "Plumbing." Contains system-level software that must be running *before* your apps (e.g., `metallb` for networking, `local-path-provisioner` for storage).
  * **`apps/`**: Your actual applications. This is where you add new services (like Plex, Home Assistant, etc.).
  * **`helmrepos/`**: A central list of external Helm repositories used by your charts.

-----

## 🚀 Installation Guide

### 1\. Bootstrap Flux

Initialize Flux on your cluster and link it to this GitHub repository.

> **Note:** Replace variables with your actual GitHub username and repository name.

```bash
flux bootstrap github \
  --token-auth \
  --owner=<githubUsername> \
  --repository=<githubRepo> \
  --branch=main \
  --path=cluster \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

### 2\. Configure SOPS (Hybrid GPG + Age)

We configure SOPS so **you** can decrypt secrets locally (GPG) and **Flux** can decrypt them inside the cluster (Age).

1.  **Generate Keys:**

      * **Admin (GPG):** Run `gpg --list-secret-keys --keyid-format=long` and copy your key ID.
      * **Cluster (Age):** Run `age-keygen -o age.agekey` and copy the public key (starts with `age1...`).

2.  **Update Config:**

      * Open `.sops.yaml` in the root of this repo.
      * Paste your **GPG Key ID** into the `YOUR_FULL_GPG_FINGERPRINT_HERE` placeholder.
      * Paste the **Age Public Key** into the `age1_YOUR_PUBLIC_KEY_HERE` placeholder.

3.  **Upload Age Key to Cluster:**
    Flux needs the *private* Age key to decrypt secrets.

    ```bash
    cat age.agekey | kubectl create secret generic sops-age \
      --namespace=flux-system \
      --from-file=age.agekey=/dev/stdin
    ```

4.  **Cleanup:**

    > ⚠️ **IMPORTANT:** Delete `age.agekey` from your local machine. **DO NOT** push it to GitHub.

### 3\. Configure Networking

#### MetalLB (Load Balancer)

Edit `infrastructure/metallb-config/resources/pools.yaml`. Define the IPs MetalLB is allowed to use:

  * **Local IP Pool:** For services accessible only inside your home network.
  * **Public IP Pool:** For services you intend to expose (requires port forwarding).

#### Nginx Gateway Fabric

Review `infrastructure/nginx-gateway-fabric/resources/gateways.yaml` to ensure the Gateway classes match your network needs.

*(Optional)* Install [external-dns](https://kubernetes-sigs.github.io/external-dns/latest/) and [cert-manager](https://cert-manager.io/) for automated DNS and SSL.

### 4\. Configure Storage

This template uses `local-path-provisioner` to utilize node disk space.

  * Check `infrastructure/local-path-provisioner/resources/values.yaml` if you need to customize the storage path (defaults to `/opt/local-path-provisioner`).

-----

## 📦 Usage

### 1\. Push to GitHub

Once configured, commit and push your changes:

```bash
git add .
git commit -m "chore: configure cluster infrastructure"
git push origin main
```

### 2\. Verification

1.  **Check Sync:** `flux get kustomizations --watch`
2.  **Test App:** After a few minutes, the demo `echo-server` should be running.
      * Get the IP: `kubectl get svc -n echo-server-namespace`
      * Access the IP in your browser.

-----

## 🔒 Security & Notes

  * **Repo Privacy:** Keep this repository **Private**.
  * **Secret Management:**
      * Never commit raw secrets.
      * Encrypt before pushing: `sops --encrypt --in-place path/to/secret.yaml`
      * To edit an encrypted file: `sops path/to/secret.yaml` (it handles decryption/encryption automatically).

## 🤖 Disclaimer

> **Note**: The code, templates, and configuration logic in this repository were manually crafted and tested. **Only this README file was generated/enhanced with the assistance of AI.**
