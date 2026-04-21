# Gitea on Kubernetes (Raspberry Pi)

Self-hosted Git + container registry deployed on a Raspberry Pi using k3s.
Storage backed by a USB drive mounted at `/mnt/usb`.

---

## Repository Structure

```
.
├── application.yaml        # ArgoCD Application manifest
├── README.md
└── k8s/
    ├── namespace.yaml      # Gitea namespace
    ├── pv.yaml             # PersistentVolume (USB drive)
    ├── pvc.yaml            # PersistentVolumeClaim
    ├── deployment.yaml     # Gitea Deployment
    └── service.yaml        # NodePort Service
```

---

## Deployment via ArgoCD (Recommended)

ArgoCD watches the `k8s/` folder in the `main` branch and auto-deploys on every git push.

### Step 1: Prepare the USB directory on the Pi
```bash
mkdir -p /mnt/usb/gitea
```

### Step 2: Push manifests to GitHub
```bash
git add k8s/ application.yaml
git commit -m "Add Gitea manifests"
git push origin main
```

### Step 3: Register the app in ArgoCD
```bash
kubectl apply -f application.yaml
```

### Step 4: Watch ArgoCD sync
```bash
kubectl get application gitea -n argocd
# Or open the ArgoCD UI at https://<pi-ip>:30443
```

ArgoCD will automatically create the namespace, deploy all resources, and keep them in sync with git.

---

## Manual Deployment (without ArgoCD)

```bash
# Create the Gitea data directory on the USB drive
mkdir -p /mnt/usb/gitea

# Deploy all resources
kubectl apply -f k8s/

# Watch the pod start
kubectl get pods -n gitea -w
```

Access the web UI at: `http://<pi-ip>:30300`

---

## File Structure: `gitea.yaml`

The manifest is split into 5 sections separated by `---`.

---

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitea
```

| Parameter | Description |
|---|---|
| `name: gitea` | Creates an isolated Kubernetes namespace. All other resources below are deployed into this namespace. |

---

### 2. PersistentVolume (PV)

```yaml
kind: PersistentVolume
```

Represents the actual physical storage — in this case, the USB drive.

| Parameter | Value | Description |
|---|---|---|
| `capacity.storage` | `100Gi` | Total storage size allocated (USB is 114.6GB, leaving headroom) |
| `accessModes` | `ReadWriteOnce` | Only one pod can mount this volume at a time (suitable for single-node Pi) |
| `persistentVolumeReclaimPolicy` | `Retain` | When the PVC is deleted, the data is **kept** on disk (not wiped) |
| `hostPath.path` | `/mnt/usb/gitea` | The actual directory on the Raspberry Pi (USB mount point) |

---

### 3. PersistentVolumeClaim (PVC)

```yaml
kind: PersistentVolumeClaim
```

A request for storage by the application. It binds to the PV above.

| Parameter | Value | Description |
|---|---|---|
| `namespace` | `gitea` | Must match the namespace of the Deployment that uses it |
| `accessModes` | `ReadWriteOnce` | Must match the PV's access mode to bind correctly |
| `resources.requests.storage` | `100Gi` | Must be ≤ PV capacity to bind |
| `volumeName` | `gitea-pv` | Explicitly binds to the PV named `gitea-pv` (avoids random binding) |

---

### 4. Deployment

```yaml
kind: Deployment
```

Defines the Gitea application pod and its configuration.

#### Replicas & Selector

| Parameter | Value | Description |
|---|---|---|
| `replicas` | `1` | Run one pod instance (single Pi node) |
| `selector.matchLabels` | `app: gitea` | Deployment manages pods with this label |

#### Container

| Parameter | Value | Description |
|---|---|---|
| `image` | `gitea/gitea:latest` | Official Gitea image (multi-arch, supports ARM64) |
| `containerPort: 3000` | web | Gitea web UI listens on port 3000 inside the container |
| `containerPort: 22` | ssh | Git over SSH listens on port 22 inside the container |

#### Environment Variables

| Variable | Value | Description |
|---|---|---|
| `GITEA__database__DB_TYPE` | `sqlite3` | Use SQLite — no separate database pod needed, lightweight for Pi |
| `GITEA__database__PATH` | `/data/gitea/gitea.db` | Where the SQLite database file is stored (inside the container, maps to USB) |
| `GITEA__server__ROOT_URL` | `http://EdiRaspberryPI1:30300/` | The public URL of your Gitea instance. Used for generating links in the UI |
| `GITEA__server__SSH_DOMAIN` | `EdiRaspberryPI1` | Hostname shown in git clone SSH URLs |
| `GITEA__server__SSH_PORT` | `30022` | External SSH port shown in git clone SSH URLs (matches the NodePort) |

#### Volume Mount

| Parameter | Value | Description |
|---|---|---|
| `mountPath` | `/data` | Mounts the USB storage into the container at `/data` — all Gitea data (repos, DB, config) lives here |

#### Resource Limits

| Parameter | Value | Description |
|---|---|---|
| `requests.memory` | `128Mi` | Minimum memory reserved for the pod |
| `requests.cpu` | `100m` | Minimum CPU reserved (100m = 0.1 cores) |
| `limits.memory` | `512Mi` | Maximum memory the pod can use before being OOM-killed |
| `limits.cpu` | `500m` | Maximum CPU the pod can use (500m = 0.5 cores) |

---

### 5. Service

```yaml
kind: Service
spec:
  type: NodePort
```

Exposes Gitea to the outside world via the Raspberry Pi's IP address.

| Parameter | Value | Description |
|---|---|---|
| `type` | `NodePort` | Exposes the service on a port of the Pi's IP (accessible from your PC) |
| `selector.app` | `gitea` | Routes traffic to pods with label `app: gitea` |

#### Ports

| Name | port | targetPort | nodePort | Description |
|---|---|---|---|---|
| `web` | 3000 | 3000 | **30300** | Web UI — access at `http://<pi-ip>:30300` |
| `ssh` | 22 | 22 | **30022** | SSH git — use in `git clone ssh://git@<pi-ip>:30022/...` |

---

## USB Storage

The USB drive is formatted as **ext4** and mounted at `/mnt/usb`.

| Detail | Value |
|---|---|
| Device | `/dev/sda1` |
| UUID | `3880ed7c-9f5d-4698-b74c-f856d22c449f` |
| Mount point | `/mnt/usb` |
| Filesystem | ext4 |
| Size | 114.6GB |
| fstab entry | `UUID=3880ed7c-9f5d-4698-b74c-f856d22c449f  /mnt/usb  ext4  defaults,nofail  0  2` |

---

## First-Time Setup

1. Open `http://<pi-ip>:30300` in your browser
2. Complete the setup wizard (settings are pre-filled)
3. Create your admin account
4. Go to **Admin Panel → Site Administration → Packages** to enable the container registry

## Using as a Container Registry

```bash
# Login from your PC
docker login <pi-ip>:30300

# Tag an image
docker tag myimage:latest <pi-ip>:30300/<username>/myimage:latest

# Push
docker push <pi-ip>:30300/<username>/myimage:latest

# Pull
docker pull <pi-ip>:30300/<username>/myimage:latest
```
