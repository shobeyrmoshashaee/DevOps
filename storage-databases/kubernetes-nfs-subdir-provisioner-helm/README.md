<p align="center">
  <strong>English</strong> |
  <a href="./README.fa.md">فارسی</a>
</p>

# 📦 Kubernetes NFS Dynamic Provisioning with Helm

This project configures an Ubuntu or Debian NFS server and installs the Kubernetes NFS Subdirectory External Provisioner through its official Helm chart.

> [!IMPORTANT]
> This is the **Helm-based installation method**. A separate project will document the manifest/Deployment-based installation method.

> [!IMPORTANT]
> All IP addresses, networks, paths, workload names, and outputs in this public document are sanitized examples. Replace them with approved values for your environment.

## Table of Contents

- [Architecture](#architecture)
- [Components](#components)
- [Important Corrections](#important-corrections)
- [1. Install and Configure the NFS Server](#1-install-and-configure-the-nfs-server)
- [2. Prepare Kubernetes Nodes](#2-prepare-kubernetes-nodes)
- [3. Install the Provisioner with Helm](#3-install-the-provisioner-with-helm)
- [4. Verify the Installation](#4-verify-the-installation)
- [5. Test Dynamic Provisioning](#5-test-dynamic-provisioning)
- [6. Cleanup](#6-cleanup)
- [Troubleshooting](#troubleshooting)
- [Operational Considerations](#operational-considerations)
- [Acceptance Checklist](#acceptance-checklist)

## Architecture

~~~mermaid
flowchart LR
    N["Kubernetes Nodes<br/>192.0.2.0/24"]
    P["NFS Subdirectory<br/>External Provisioner"]
    S["NFS Server<br/>192.0.2.50"]
    E["Export<br/>/var/nfs/kubernetes"]
    V["PVC → PV → Subdirectory"]

    N --> P
    P --> S
    S --> E
    P --> V
~~~

## Components

| Component | Purpose |
|---|---|
| NFS server | Exposes shared storage to authorized Kubernetes nodes |
| NFS client utilities | Allow nodes to mount NFS volumes |
| NFS Subdirectory External Provisioner | Creates one subdirectory and PV for each PVC |
| StorageClass | Provides dynamic NFS-backed provisioning to workloads |
| Helm | Installs and manages the provisioner release |

## Important Corrections

The original notes were normalized in this version:

- Ubuntu and Debian use the <code>nfs-kernel-server</code> package and service.
- One sanitized NFS server, client network, and export path are used consistently.
- The multi-line Helm command is valid and uses a versionable <code>values.yaml</code>.
- The requested <code>no_root_squash</code>, <code>no_all_squash</code>, and <code>insecure</code> options are included explicitly, with their security impact documented.
- The exported directory is created before the export is activated.
- The namespace is created idempotently by Helm.
- Provisioner output is verified from the live cluster instead of hard-coding an old example.
- The test PVC requests a realistic test size and is mounted by a test Pod.

## Project Files

~~~text
kubernetes-nfs-subdir-provisioner-helm/
├── README.md
├── values.yaml
└── examples/
    ├── pvc.yaml
    └── pod.yaml
~~~

## 1. Install and Configure the NFS Server

The example uses:

| Setting | Example |
|---|---|
| NFS server | <code>192.0.2.50</code> |
| Authorized node network | <code>192.0.2.0/24</code> |
| Exported path | <code>/var/nfs/kubernetes</code> |
| NFS version | <code>4.1</code> |
| StorageClass | <code>nfs-client</code> |
| Namespace | <code>nfs-storage</code> |

All addresses use the RFC 5737 documentation range.

### Install the NFS Server

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
~~~

Enable and start the service:

~~~bash
sudo systemctl enable --now nfs-kernel-server
sudo systemctl --no-pager status nfs-kernel-server
~~~

### Create the Export Directory

~~~bash
sudo install \
  -d \
  -o nobody \
  -g nogroup \
  -m 0770 \
  /var/nfs/kubernetes
~~~

Verify the directory:

~~~bash
sudo stat -c '%A %U:%G %n' /var/nfs/kubernetes
~~~

Expected result:

~~~text
drwxrwx--- nobody:nogroup /var/nfs/kubernetes
~~~

> [!NOTE]
> The provisioner values in this project use UID/GID <code>65534</code>, which normally maps to <code>nobody:nogroup</code> on Ubuntu and Debian. Validate this mapping on the target operating system.

### Configure the NFS Export

Use a dedicated export file:

~~~bash
sudo nano /etc/exports.d/kubernetes-nfs.exports
~~~

Add:

~~~exports
/var/nfs/kubernetes 192.0.2.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
~~~

Apply and verify the export:

~~~bash
sudo exportfs -rav
sudo exportfs -v
~~~

> [!WARNING]
> This export intentionally enables <code>no_root_squash</code>. Root processes on an authorized Kubernetes node retain root privileges on the NFS export and can modify or delete any accessible data. Restrict the export to explicitly trusted node addresses, protect the node network, and never expose this export to an untrusted subnet.

> [!CAUTION]
> The <code>insecure</code> option accepts NFS requests originating from client source ports above 1024, weakening the traditional privileged-port check. Use it only when required by the client environment and compensate with strict firewall rules and a narrowly scoped client subnet.

> [!NOTE]
> <code>no_all_squash</code> preserves client UID/GID identities instead of mapping every user to the anonymous account. It is normally the default behavior, but is included explicitly here as requested. Consistent UID/GID management across nodes and workloads is therefore important.

### Firewall

For NFSv4.1, allow TCP/2049 only from the authorized Kubernetes node network:

~~~bash
sudo ufw allow \
  from 192.0.2.0/24 \
  to any port 2049 \
  proto tcp
~~~

> [!NOTE]
> NFSv3 can require additional RPC and mountd ports. This project explicitly requests NFSv4.1 to keep the network policy predictable.

## 2. Prepare Kubernetes Nodes

Install the NFS client package on every node that may run a Pod using an NFS-backed volume:

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-common
~~~

Confirm the package:

~~~bash
dpkg -l nfs-common
~~~

Test NFS connectivity from a Kubernetes node:

~~~bash
sudo install -d -m 0755 /mnt/nfs-test

sudo mount \
  -t nfs \
  -o nfsvers=4.1 \
  192.0.2.50:/var/nfs/kubernetes \
  /mnt/nfs-test

mount | grep '/mnt/nfs-test'
sudo touch /mnt/nfs-test/node-connectivity-test
sudo ls -la /mnt/nfs-test
sudo rm -f /mnt/nfs-test/node-connectivity-test
sudo umount /mnt/nfs-test
~~~

> [!CAUTION]
> Do not continue with the Helm installation until every eligible Kubernetes node can reach and mount the NFS export.

## 3. Install the Provisioner with Helm

### Add the Official Helm Repository

~~~bash
helm repo add \
  nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm repo update
helm search repo nfs-subdir-external-provisioner
~~~

### Review the Values File

The included [values.yaml](./values.yaml) configures:

- The sanitized NFS server and export path
- NFSv4.1 mount options
- StorageClass name <code>nfs-client</code>
- Dynamic expansion support
- <code>Delete</code> PV reclaim policy with directory archiving enabled
- Directory UID/GID <code>65534</code> and mode <code>0770</code>
- Conservative resource requests and limits

> [!WARNING]
> Review <code>storageClass.reclaimPolicy</code> and <code>archiveOnDelete</code> before production use. Data-retention requirements must be decided explicitly.

### Install or Upgrade the Release

~~~bash
helm upgrade --install \
  nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-storage \
  --create-namespace \
  --values values.yaml \
  --wait \
  --timeout 5m
~~~

> [!TIP]
> Pin a tested chart version with <code>--version</code> in production change procedures rather than always consuming the newest chart.

Inspect the release:

~~~bash
helm list -n nfs-storage
helm status nfs-subdir-external-provisioner -n nfs-storage
helm get values nfs-subdir-external-provisioner -n nfs-storage
~~~

## 4. Verify the Installation

Check the namespace and workload:

~~~bash
kubectl get namespace nfs-storage
kubectl get all -n nfs-storage
kubectl rollout status \
  deployment/nfs-subdir-external-provisioner \
  -n nfs-storage \
  --timeout=180s
~~~

Check the provisioner logs:

~~~bash
kubectl logs \
  -n nfs-storage \
  deployment/nfs-subdir-external-provisioner \
  --tail=100
~~~

Inspect the StorageClass:

~~~bash
kubectl get storageclass nfs-client
kubectl describe storageclass nfs-client
~~~

Important fields:

| Field | Expected Purpose |
|---|---|
| <code>metadata.name</code> | StorageClass referenced by PVCs |
| <code>provisioner</code> | Must match the provisioner deployed by the chart |
| <code>reclaimPolicy</code> | Controls PV lifecycle after PVC deletion |
| <code>volumeBindingMode</code> | Controls when binding occurs |
| <code>allowVolumeExpansion</code> | Enables requested-capacity increases |

> [!NOTE]
> Do not hard-code a historical <code>PROVISIONER</code> value in documentation. Verify the value created by the installed chart because it can vary with chart configuration.

## 5. Test Dynamic Provisioning

### Create the PVC

~~~bash
kubectl apply -f examples/pvc.yaml
kubectl get pvc nfs-dynamic-test -w
~~~

The PVC should become <code>Bound</code>. Inspect it if needed:

~~~bash
kubectl describe pvc nfs-dynamic-test
kubectl get pv
~~~

### Mount the PVC in a Pod

~~~bash
kubectl apply -f examples/pod.yaml
kubectl wait \
  --for=condition=Ready \
  pod/nfs-rwx-test \
  --timeout=120s
~~~

Verify read/write access:

~~~bash
kubectl exec nfs-rwx-test -- cat /data/healthcheck.txt
kubectl exec nfs-rwx-test -- sh -c 'echo manual-check >> /data/healthcheck.txt'
kubectl exec nfs-rwx-test -- tail /data/healthcheck.txt
~~~

Verify that the provisioner created a subdirectory on the NFS server:

~~~bash
sudo find /var/nfs/kubernetes \
  -mindepth 1 \
  -maxdepth 2 \
  -printf '%M %u:%g %p\n'
~~~

## 6. Cleanup

Delete the test Pod and PVC:

~~~bash
kubectl delete -f examples/pod.yaml
kubectl delete -f examples/pvc.yaml
~~~

Inspect the PV and NFS export after cleanup:

~~~bash
kubectl get pv
sudo find /var/nfs/kubernetes -mindepth 1 -maxdepth 2 -ls
~~~

> [!IMPORTANT]
> With <code>reclaimPolicy: Delete</code> and <code>archiveOnDelete: true</code>, the PV object is deleted and the backing directory is normally renamed for archival. Verify actual behavior before relying on it for production retention.

To uninstall the Helm release:

~~~bash
helm uninstall \
  nfs-subdir-external-provisioner \
  -n nfs-storage
~~~

Do not delete the namespace, export, or archived directories until retention requirements have been reviewed.

## Troubleshooting

### PVC Remains Pending

~~~bash
kubectl describe pvc nfs-dynamic-test
kubectl get events --sort-by=.lastTimestamp
kubectl logs \
  -n nfs-storage \
  deployment/nfs-subdir-external-provisioner \
  --tail=200
~~~

Verify that the PVC uses <code>storageClassName: nfs-client</code>.

### Provisioner Pod Is Not Ready

~~~bash
kubectl get pods -n nfs-storage -o wide
kubectl describe pods -n nfs-storage
kubectl get events -n nfs-storage --sort-by=.lastTimestamp
~~~

Check NFS reachability from the node hosting the provisioner Pod.

### Mount Fails on a Kubernetes Node

~~~bash
sudo journalctl -u kubelet -n 200 --no-pager
sudo dmesg | tail -n 100
sudo ss -tn | grep ':2049'
~~~

Verify the client package:

~~~bash
dpkg -l nfs-common
~~~

Perform the manual mount test from Section 2.

### Permission Denied

On the NFS server:

~~~bash
sudo exportfs -v
sudo namei -l /var/nfs/kubernetes
sudo stat -c '%A %u:%g %n' /var/nfs/kubernetes
~~~

Inside the workload:

~~~bash
kubectl exec nfs-rwx-test -- id
kubectl exec nfs-rwx-test -- stat -c '%A %u:%g %n' /data
~~~

Align Pod <code>runAsUser</code>, <code>runAsGroup</code>, and <code>fsGroup</code> with the NFS directory ownership and access policy.

### Helm Installation Fails

~~~bash
helm status nfs-subdir-external-provisioner -n nfs-storage
helm get manifest nfs-subdir-external-provisioner -n nfs-storage
kubectl get events -n nfs-storage --sort-by=.lastTimestamp
~~~

Confirm that the chart repository is reachable and that the cluster can pull the provisioner image.

## Operational Considerations

- NFS is a shared dependency; monitor availability, latency, capacity, inode usage, and mount failures.
- Export access must cover every eligible worker node, not only control-plane nodes.
- Back up production data independently of the provisioner lifecycle.
- Test PVC deletion and restoration behavior before selecting a reclaim policy.
- Use Pod security contexts that match the approved NFS UID/GID strategy.
- This configuration deliberately uses <code>no_root_squash</code> and <code>insecure</code>; limit access to trusted Kubernetes nodes and enforce the restriction with host and network firewalls.
- Pin chart and image versions in controlled production environments.
- A single NFS server is a storage single point of failure unless protected by an HA design.
- Monitor the provisioner Deployment and alert on unavailable replicas or repeated provisioning errors.

## Acceptance Checklist

- [ ] The NFS server package and service are active.
- [ ] The export directory exists with approved ownership and permissions.
- [ ] The export is restricted to the trusted Kubernetes node network.
- [ ] The security implications of <code>no_root_squash</code> and <code>insecure</code> have been explicitly approved.
- [ ] <code>exportfs -v</code> shows the expected path and requested options.
- [ ] Every eligible Kubernetes node has <code>nfs-common</code>.
- [ ] Every eligible node can mount the NFS export.
- [ ] The Helm release is deployed and healthy.
- [ ] The provisioner Deployment is available.
- [ ] The <code>nfs-client</code> StorageClass exists.
- [ ] The test PVC becomes <code>Bound</code>.
- [ ] The test Pod can read and write the mounted volume.
- [ ] The expected subdirectory appears on the NFS server.
- [ ] Deletion and retention behavior has been validated.
- [ ] No real IP address, internal hostname, credential, or sensitive workload name is committed.

## References

- [NFS Subdirectory External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [Official Helm Chart](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/charts/nfs-subdir-external-provisioner)
- [Ubuntu NFS Exports Manual](https://manpages.ubuntu.com/manpages/focal/man5/exports.5.html)
