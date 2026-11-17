# 📁 Kubernetes NFS Dynamic Provisioning with Deployment Manifests

This project configures an Ubuntu or Debian NFS server and installs the Kubernetes NFS Subdirectory External Provisioner with Kubernetes manifests and a Kustomize patch.

> [!IMPORTANT]
> This is the **Deployment-based installation method**. The Helm-based method is documented separately in [Kubernetes NFS Dynamic Provisioning with Helm](../kubernetes-nfs-subdir-provisioner-helm/).

> [!WARNING]
> The Helm-based and Deployment-based projects are alternative installation methods. Do not install both in the same cluster with the same StorageClass name and provisioner identity.

> [!IMPORTANT]
> All addresses, networks, paths, resource names, and outputs are sanitized examples. Replace them with approved values before deployment.

## Table of Contents

- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [1. Configure the NFS Server](#1-configure-the-nfs-server)
- [2. Prepare Kubernetes Worker Nodes](#2-prepare-kubernetes-worker-nodes)
- [3. Review the Upstream Deploy Directory](#3-review-the-upstream-deploy-directory)
- [4. Configure the Deployment with Kustomize](#4-configure-the-deployment-with-kustomize)
- [5. Install in the Required Order](#5-install-in-the-required-order)
- [6. Verify the Provisioner](#6-verify-the-provisioner)
- [7. Test with a PVC and Deployment](#7-test-with-a-pvc-and-deployment)
- [8. Cleanup](#8-cleanup)
- [Troubleshooting](#troubleshooting)
- [Operational Considerations](#operational-considerations)
- [Acceptance Checklist](#acceptance-checklist)

## Architecture

~~~mermaid
flowchart LR
    W["Kubernetes Workers<br/>192.0.2.0/24"]
    D["Provisioner Deployment"]
    SC["StorageClass<br/>nfs-client"]
    N["NFS Server<br/>192.0.2.50"]
    E["Export<br/>/mnt/nfs/kubernetes"]
    P["PVC → PV → Subdirectory"]

    W --> D
    D --> N
    N --> E
    SC --> P
    D --> P
~~~

## Project Structure

~~~text
kubernetes-nfs-subdir-provisioner-deployment/
├── README.md
├── manifests/
│   ├── namespace.yaml
│   ├── rbac.yaml
│   ├── class.yaml
│   └── deployment/
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       └── patch_nfs_details.yaml
└── examples/
    ├── pvc.yaml
    └── deployment.yaml
~~~

The partial <code>patch_nfs_details.yaml</code> file is a Kustomize patch. It is not a complete Deployment and must not be applied directly with <code>kubectl apply -f</code>.

## 1. Configure the NFS Server

The sanitized example uses:

| Setting | Value |
|---|---|
| NFS server | <code>192.0.2.50</code> |
| Kubernetes node network | <code>192.0.2.0/24</code> |
| Export path | <code>/mnt/nfs/kubernetes</code> |
| Provisioner namespace | <code>nfs-provisioner</code> |
| StorageClass | <code>nfs-client</code> |

### Install the NFS Server

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
~~~

Create the export directory:

~~~bash
sudo install \
  -d \
  -o nobody \
  -g nogroup \
  -m 0770 \
  /mnt/nfs/kubernetes
~~~

### Configure the Export

~~~bash
sudo nano /etc/exports.d/kubernetes-nfs.exports
~~~

Add:

~~~exports
/mnt/nfs/kubernetes 192.0.2.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
~~~

> [!WARNING]
> This configuration intentionally enables <code>no_root_squash</code>. Root on an authorized Kubernetes node retains root privileges on the export and can modify or delete accessible data.

> [!CAUTION]
> The <code>insecure</code> option accepts requests from client source ports above 1024. Restrict the export to trusted node addresses and enforce the same restriction with network and host firewalls.

> [!NOTE]
> <code>no_all_squash</code> preserves client UID/GID identities. Keep UID/GID policy consistent across the Kubernetes nodes and workloads.

Apply and verify:

~~~bash
sudo exportfs -rav
sudo exportfs -v
sudo systemctl enable --now nfs-kernel-server
sudo systemctl --no-pager status nfs-kernel-server
sudo showmount -e localhost
~~~

> [!NOTE]
> Ubuntu and Debian provide <code>nfs-kernel-server.service</code>. Some systems also expose <code>nfs-server.service</code> as an alias, but this guide uses the package-native service name.

### Test the Export

Run from an authorized client:

~~~bash
sudo install -d -m 0755 /mnt/nfs-test

sudo mount \
  -t nfs \
  -o nfsvers=4.1 \
  192.0.2.50:/mnt/nfs/kubernetes \
  /mnt/nfs-test

mount | grep '/mnt/nfs-test'
sudo touch /mnt/nfs-test/connectivity-test
sudo ls -la /mnt/nfs-test
sudo rm -f /mnt/nfs-test/connectivity-test
sudo umount /mnt/nfs-test
~~~

## 2. Prepare Kubernetes Worker Nodes

Install NFS client utilities on every worker that may run a Pod using this StorageClass:

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-common
~~~

Verify:

~~~bash
dpkg -l nfs-common
~~~

> [!CAUTION]
> Export access must include every eligible worker node, not only the control-plane nodes or the node running the provisioner Pod.

## 3. Review the Upstream Deploy Directory

The upstream repository provides <code>rbac.yaml</code>, <code>deployment.yaml</code>, and <code>class.yaml</code> under its <code>deploy</code> directory:

~~~bash
git clone \
  --depth 1 \
  https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git

cd nfs-subdir-external-provisioner/deploy
ls -la
~~~

Before applying the upstream files, review these fields:

- Namespace in the ServiceAccount, Role, RoleBinding, ClusterRoleBinding subject, and Deployment
- <code>PROVISIONER_NAME</code>
- Both <code>NFS_SERVER</code> occurrences
- Both <code>NFS_PATH</code> occurrences
- StorageClass <code>provisioner</code>, which must match <code>PROVISIONER_NAME</code>

> [!WARNING]
> Do not apply the unmodified upstream Deployment. Its example NFS server and path are not values for your environment.

The self-contained manifests in this project already use the dedicated <code>nfs-provisioner</code> namespace and a patch for sanitized NFS details.

## 4. Configure the Deployment with Kustomize

Edit the patch:

~~~bash
nano manifests/deployment/patch_nfs_details.yaml
~~~

The patch must update both the environment variables and NFS volume source:

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
spec:
  template:
    spec:
      containers:
        - name: nfs-client-provisioner
          env:
            - name: NFS_SERVER
              value: 192.0.2.50
            - name: NFS_PATH
              value: /mnt/nfs/kubernetes
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.0.2.50
            path: /mnt/nfs/kubernetes
~~~

Render and inspect the patched Deployment before applying it:

~~~bash
kubectl kustomize manifests/deployment/
~~~

Check the four NFS values:

~~~bash
kubectl kustomize manifests/deployment/ \
  | grep -nE 'NFS_SERVER|NFS_PATH|server:|path:'
~~~

> [!IMPORTANT]
> <code>NFS_SERVER</code> and <code>NFS_PATH</code> must match the NFS volume's <code>server</code> and <code>path</code>. A mismatch can start the provisioner with one configuration while mounting another.

## 5. Install in the Required Order

### Step 1 — Namespace

~~~bash
kubectl apply -f manifests/namespace.yaml
kubectl get namespace nfs-provisioner
~~~

### Step 2 — RBAC

~~~bash
kubectl apply -f manifests/rbac.yaml
~~~

Verify the ServiceAccount and permissions:

~~~bash
kubectl get serviceaccount -n nfs-provisioner
kubectl auth can-i create persistentvolumes \
  --as=system:serviceaccount:nfs-provisioner:nfs-client-provisioner
~~~

### Step 3 — Patched Deployment

~~~bash
kubectl apply -k manifests/deployment/
~~~

Wait for the rollout:

~~~bash
kubectl rollout status \
  deployment/nfs-client-provisioner \
  -n nfs-provisioner \
  --timeout=180s
~~~

### Step 4 — StorageClass

~~~bash
kubectl apply -f manifests/class.yaml
kubectl get storageclass nfs-client
~~~

> [!IMPORTANT]
> The StorageClass <code>provisioner</code> must exactly match the Deployment's <code>PROVISIONER_NAME</code>: <code>k8s-sigs.io/nfs-subdir-external-provisioner</code>.

## 6. Verify the Provisioner

~~~bash
kubectl get all -n nfs-provisioner
kubectl get deployment nfs-client-provisioner -n nfs-provisioner
kubectl get pods -n nfs-provisioner -o wide
kubectl describe storageclass nfs-client
~~~

Review logs:

~~~bash
kubectl logs \
  -n nfs-provisioner \
  deployment/nfs-client-provisioner \
  --tail=100
~~~

Confirm the rendered NFS configuration:

~~~bash
kubectl get deployment nfs-client-provisioner \
  -n nfs-provisioner \
  -o yaml \
  | grep -nE 'NFS_SERVER|NFS_PATH|server:|path:'
~~~

## 7. Test with a PVC and Deployment

### Create the PVC

~~~bash
kubectl apply -f examples/pvc.yaml
kubectl get pvc nfs-dynamic-claim -w
~~~

The PVC should become <code>Bound</code> and a PV should be created:

~~~bash
kubectl describe pvc nfs-dynamic-claim
kubectl get pv
~~~

### Create the Example Deployment

The example Deployment starts two Pods that write their timestamp and hostname to the same ReadWriteMany volume:

~~~bash
kubectl apply -f examples/deployment.yaml
kubectl rollout status deployment/nfs-rwx-demo --timeout=180s
kubectl get pods -l app=nfs-rwx-demo -o wide
~~~

Wait for both replicas to write, then inspect the shared file:

~~~bash
kubectl exec deployment/nfs-rwx-demo -- tail -n 20 /data/replicas.log
~~~

Inspect the subdirectory on the NFS server:

~~~bash
sudo find /mnt/nfs/kubernetes \
  -mindepth 1 \
  -maxdepth 2 \
  -printf '%M %u:%g %p\n'
~~~

> [!NOTE]
> The requested PVC capacity is not a filesystem quota. NFS Subdirectory External Provisioner creates subdirectories but does not enforce per-PVC capacity by itself.

## 8. Cleanup

~~~bash
kubectl delete -f examples/deployment.yaml
kubectl delete -f examples/pvc.yaml
~~~

Inspect retention behavior:

~~~bash
kubectl get pv
sudo find /mnt/nfs/kubernetes -mindepth 1 -maxdepth 2 -ls
~~~

Remove the provisioner in reverse order only after confirming no PVC depends on it:

~~~bash
kubectl delete -f manifests/class.yaml
kubectl delete -k manifests/deployment/
kubectl delete -f manifests/rbac.yaml
kubectl delete -f manifests/namespace.yaml
~~~

> [!CAUTION]
> Deleting the provisioner does not replace a data-retention or backup procedure. Review PVs, archived directories, and backups before cleanup.

## Troubleshooting

### Deployment Does Not Become Available

~~~bash
kubectl get pods -n nfs-provisioner -o wide
kubectl describe pods -n nfs-provisioner
kubectl get events -n nfs-provisioner --sort-by=.lastTimestamp
kubectl logs -n nfs-provisioner deployment/nfs-client-provisioner --tail=200
~~~

### NFS Mount Fails

On the worker hosting the provisioner Pod:

~~~bash
sudo apt-get install -y nfs-common
sudo ss -tn | grep ':2049'
sudo journalctl -u kubelet -n 200 --no-pager
~~~

Repeat the manual mount test from Section 1.

### PVC Remains Pending

~~~bash
kubectl describe pvc nfs-dynamic-claim
kubectl get events --sort-by=.lastTimestamp
kubectl logs -n nfs-provisioner deployment/nfs-client-provisioner --tail=200
~~~

Compare these values:

~~~bash
kubectl get storageclass nfs-client -o jsonpath='{.provisioner}{"\n"}'
kubectl get deployment nfs-client-provisioner \
  -n nfs-provisioner \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="PROVISIONER_NAME")].value}{"\n"}'
~~~

They must be identical.

### Permission Denied

On the NFS server:

~~~bash
sudo exportfs -v
sudo namei -l /mnt/nfs/kubernetes
sudo stat -c '%A %u:%g %n' /mnt/nfs/kubernetes
~~~

Inside an example Pod:

~~~bash
kubectl exec deployment/nfs-rwx-demo -- id
kubectl exec deployment/nfs-rwx-demo -- stat -c '%A %u:%g %n' /data
~~~

Review export options, directory permissions, Pod security context, and UID/GID mapping.

### Kustomize Patch Does Not Apply

~~~bash
kubectl kustomize manifests/deployment/
~~~

The Deployment name and container name in the patch must match the base manifest:

~~~text
Deployment: nfs-client-provisioner
Container:  nfs-client-provisioner
Volume:     nfs-client-root
~~~

## Operational Considerations

- The NFS server is a shared dependency and can become a single point of failure.
- Monitor storage capacity, inode consumption, NFS latency, mount failures, and provisioner errors.
- Back up application data independently of PV/PVC lifecycle.
- Keep exports restricted to trusted Kubernetes nodes.
- This configuration deliberately uses <code>no_root_squash</code> and <code>insecure</code>; document and approve the risk.
- Pin the provisioner image by an approved version.
- Keep the StorageClass provisioner identity synchronized with the Deployment.
- Test PVC deletion, archiving, and restoration behavior.
- Do not interpret requested PVC capacity as an enforced NFS quota.
- Review upstream changes before replacing the included manifests.

## Acceptance Checklist

- [ ] The NFS server is active and the export exists.
- [ ] Export access is restricted to trusted Kubernetes nodes.
- [ ] The risks of <code>no_root_squash</code> and <code>insecure</code> are approved.
- [ ] Every eligible worker has <code>nfs-common</code>.
- [ ] Every eligible worker can mount the export.
- [ ] The namespace and RBAC resources are present.
- [ ] The Kustomize patch sets both environment and volume NFS values.
- [ ] The provisioner Deployment is available.
- [ ] StorageClass <code>provisioner</code> matches <code>PROVISIONER_NAME</code>.
- [ ] The test PVC becomes <code>Bound</code>.
- [ ] Both example Deployment replicas can write to the RWX volume.
- [ ] The NFS server contains the dynamically created subdirectory.
- [ ] Retention and backup behavior is validated.
- [ ] No real address, internal hostname, credential, or sensitive workload name is committed.

## References

- [NFS Subdirectory External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [Upstream Deployment](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/deploy/deployment.yaml)
- [Upstream RBAC](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/deploy/rbac.yaml)
- [Upstream StorageClass](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/deploy/class.yaml)
