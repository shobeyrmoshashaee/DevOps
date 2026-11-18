<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">📁 راه‌اندازی <span dir="ltr">NFS Dynamic Provisioning</span> با فایل‌های <span dir="ltr">Deployment</span></h1>

<p dir="rtl" align="right">اینجا یک <span dir="ltr">NFS Server</span> می‌سازیم و <span dir="ltr">NFS Subdirectory External Provisioner</span> را با فایل‌های Kubernetes و یک <span dir="ltr">Kustomize Patch</span> نصب می‌کنیم.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">این پروژه مخصوص روش نصب با <span dir="ltr">Deployment Manifest</span> است. روش <span dir="ltr">Helm</span> در <a href="../kubernetes-nfs-subdir-provisioner-helm/README.fa.md">پروژه جدا</a> قرار دارد.</p>

> [!WARNING]
> <p dir="rtl" align="right">روش Helm و Deployment دو روش جایگزین هستند. هر دو را با یک اسم StorageClass و Provisioner داخل یک Cluster نصب نکنید.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">همه IPها، مسیرها و اسم‌ها نمونه هستند. موقع اجرا، مقادیر محیط خودتان را جایگزین کنید.</p>

<h2 dir="rtl" align="right">معماری</h2>

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

<h2 dir="rtl" align="right">ساختار پروژه</h2>

~~~text
kubernetes-nfs-subdir-provisioner-deployment/
├── README.md
├── README.fa.md
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

> [!NOTE]
> <p dir="rtl" align="right">فایل <code dir="ltr">patch_nfs_details.yaml</code> یک Patch است، نه یک Deployment کامل. آن را مستقیم با <code dir="ltr">kubectl apply -f</code> اجرا نکنید.</p>

<h2 dir="rtl" align="right">۱. تنظیم <span dir="ltr">NFS Server</span></h2>

<table dir="rtl">
  <thead><tr><th align="right">تنظیم</th><th align="right">مقدار مثال</th></tr></thead>
  <tbody>
    <tr><td align="right">NFS Server</td><td dir="ltr"><code>192.0.2.50</code></td></tr>
    <tr><td align="right">Node Network</td><td dir="ltr"><code>192.0.2.0/24</code></td></tr>
    <tr><td align="right">Export Path</td><td dir="ltr"><code>/mnt/nfs/kubernetes</code></td></tr>
    <tr><td align="right">Namespace</td><td dir="ltr"><code>nfs-provisioner</code></td></tr>
    <tr><td align="right">StorageClass</td><td dir="ltr"><code>nfs-client</code></td></tr>
  </tbody>
</table>

<h3 dir="rtl" align="right">نصب و ساخت مسیر <span dir="ltr">Export</span></h3>

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
~~~

~~~bash
sudo install \
  -d \
  -o nobody \
  -g nogroup \
  -m 0770 \
  /mnt/nfs/kubernetes
~~~

<h3 dir="rtl" align="right">تنظیم <span dir="ltr">Export</span></h3>

~~~bash
sudo nano /etc/exports.d/kubernetes-nfs.exports
~~~

~~~exports
/mnt/nfs/kubernetes 192.0.2.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
~~~

> [!WARNING]
> <p dir="rtl" align="right">با <code dir="ltr">no_root_squash</code>، کاربر root روی Node می‌تواند فایل‌های NFS را تغییر دهد یا پاک کند. Export را فقط برای Nodeهای مورد اعتماد باز کنید.</p>

> [!CAUTION]
> <p dir="rtl" align="right">گزینه <code dir="ltr">insecure</code> اتصال از Source Portهای بالاتر از 1024 را قبول می‌کند. دسترسی را با Firewall و Subnet محدود کنید.</p>

> [!NOTE]
> <p dir="rtl" align="right">گزینه <code dir="ltr">no_all_squash</code> باعث می‌شود UID و GID کاربران حفظ شود. UID/GID روی Nodeها و Podها باید هماهنگ باشد.</p>

~~~bash
sudo exportfs -rav
sudo exportfs -v
sudo systemctl enable --now nfs-kernel-server
sudo systemctl --no-pager status nfs-kernel-server
sudo showmount -e localhost
~~~

<h3 dir="rtl" align="right">تست <span dir="ltr">Mount</span></h3>

<p dir="rtl" align="right">از یکی از Clientهای مجاز این تست را انجام دهید:</p>

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

<h2 dir="rtl" align="right">۲. آماده کردن <span dir="ltr">Worker Node</span>ها</h2>

<p dir="rtl" align="right">روی همه Workerهایی که ممکن است Pod دارای NFS Volume روی آن‌ها اجرا شود، این پکیج را نصب کنید:</p>

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-common
dpkg -l nfs-common
~~~

> [!CAUTION]
> <p dir="rtl" align="right">Export باید برای همه Workerهای مجاز باز باشد، نه فقط Control Plane یا Nodeای که Provisioner Pod روی آن اجرا می‌شود.</p>

<h2 dir="rtl" align="right">۳. بررسی فایل‌های اصلی پروژه</h2>

<p dir="rtl" align="right">Repository اصلی فایل‌های <code dir="ltr">rbac.yaml</code>، <code dir="ltr">deployment.yaml</code> و <code dir="ltr">class.yaml</code> را داخل مسیر <code dir="ltr">deploy</code> دارد:</p>

~~~bash
git clone \
  --depth 1 \
  https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git

cd nfs-subdir-external-provisioner/deploy
ls -la
~~~

<p dir="rtl" align="right">قبل از اجرا این موارد را بررسی کنید:</p>

<ul dir="rtl" align="right">
  <li>Namespace در ServiceAccount، RoleBinding و Deployment</li>
  <li>مقدار <code dir="ltr">PROVISIONER_NAME</code></li>
  <li>هر دو مقدار <code dir="ltr">NFS_SERVER</code></li>
  <li>هر دو مقدار <code dir="ltr">NFS_PATH</code></li>
  <li>مقدار Provisioner در StorageClass</li>
</ul>

> [!WARNING]
> <p dir="rtl" align="right">Deployment اصلی را بدون تغییر اجرا نکنید؛ NFS Server و Path مثال آن برای محیط شما نیست.</p>

<h2 dir="rtl" align="right">۴. تنظیم <span dir="ltr">Deployment</span> با <span dir="ltr">Kustomize</span></h2>

~~~bash
nano manifests/deployment/patch_nfs_details.yaml
~~~

<p dir="rtl" align="right">در Patch باید هم Environment Variableها و هم NFS Volume تغییر کنند:</p>

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

<p dir="rtl" align="right">قبل از Apply، خروجی نهایی را ببینید:</p>

~~~bash
kubectl kustomize manifests/deployment/
~~~

~~~bash
kubectl kustomize manifests/deployment/ \
  | grep -nE 'NFS_SERVER|NFS_PATH|server:|path:'
~~~

> [!IMPORTANT]
> <p dir="rtl" align="right">مقدارهای <code dir="ltr">NFS_SERVER</code> و <code dir="ltr">NFS_PATH</code> باید با <code dir="ltr">server</code> و <code dir="ltr">path</code> داخل Volume یکی باشند.</p>

<h2 dir="rtl" align="right">۵. نصب به ترتیب درست</h2>

<h3 dir="rtl" align="right">مرحله ۱: <span dir="ltr">Namespace</span></h3>

~~~bash
kubectl apply -f manifests/namespace.yaml
kubectl get namespace nfs-provisioner
~~~

<h3 dir="rtl" align="right">مرحله ۲: <span dir="ltr">RBAC</span></h3>

~~~bash
kubectl apply -f manifests/rbac.yaml
~~~

~~~bash
kubectl get serviceaccount -n nfs-provisioner
kubectl auth can-i create persistentvolumes \
  --as=system:serviceaccount:nfs-provisioner:nfs-client-provisioner
~~~

<h3 dir="rtl" align="right">مرحله ۳: <span dir="ltr">Deployment</span></h3>

~~~bash
kubectl apply -k manifests/deployment/
~~~

~~~bash
kubectl rollout status \
  deployment/nfs-client-provisioner \
  -n nfs-provisioner \
  --timeout=180s
~~~

<h3 dir="rtl" align="right">مرحله ۴: <span dir="ltr">StorageClass</span></h3>

~~~bash
kubectl apply -f manifests/class.yaml
kubectl get storageclass nfs-client
~~~

> [!IMPORTANT]
> <p dir="rtl" align="right">مقدار <code dir="ltr">provisioner</code> در StorageClass باید دقیقاً با <code dir="ltr">PROVISIONER_NAME</code> داخل Deployment یکی باشد.</p>

<h2 dir="rtl" align="right">۶. بررسی <span dir="ltr">Provisioner</span></h2>

~~~bash
kubectl get all -n nfs-provisioner
kubectl get deployment nfs-client-provisioner -n nfs-provisioner
kubectl get pods -n nfs-provisioner -o wide
kubectl describe storageclass nfs-client
~~~

~~~bash
kubectl logs \
  -n nfs-provisioner \
  deployment/nfs-client-provisioner \
  --tail=100
~~~

<p dir="rtl" align="right">مقادیر NFS داخل Deployment را هم چک کنید:</p>

~~~bash
kubectl get deployment nfs-client-provisioner \
  -n nfs-provisioner \
  -o yaml \
  | grep -nE 'NFS_SERVER|NFS_PATH|server:|path:'
~~~

<h2 dir="rtl" align="right">۷. تست با <span dir="ltr">PVC</span> و <span dir="ltr">Deployment</span></h2>

<p dir="rtl" align="right">اول PVC را بسازید:</p>

~~~bash
kubectl apply -f examples/pvc.yaml
kubectl get pvc nfs-dynamic-claim -w
~~~

<p dir="rtl" align="right">PVC باید <code dir="ltr">Bound</code> شود:</p>

~~~bash
kubectl describe pvc nfs-dynamic-claim
kubectl get pv
~~~

<p dir="rtl" align="right">حالا Deployment تست را با دو Replica اجرا کنید:</p>

~~~bash
kubectl apply -f examples/deployment.yaml
kubectl rollout status deployment/nfs-rwx-demo --timeout=180s
kubectl get pods -l app=nfs-rwx-demo -o wide
~~~

<p dir="rtl" align="right">هر دو Pod داخل یک فایل Shared می‌نویسند. نتیجه را ببینید:</p>

~~~bash
kubectl exec deployment/nfs-rwx-demo -- tail -n 20 /data/replicas.log
~~~

<p dir="rtl" align="right">روی NFS Server هم پوشه ساخته‌شده را بررسی کنید:</p>

~~~bash
sudo find /mnt/nfs/kubernetes \
  -mindepth 1 \
  -maxdepth 2 \
  -printf '%M %u:%g %p\n'
~~~

> [!NOTE]
> <p dir="rtl" align="right">مقدار Storage در PVC فقط یک Request است. این Provisioner به‌تنهایی برای هر PVC Quota واقعی ایجاد نمی‌کند.</p>

<h2 dir="rtl" align="right">۸. پاک کردن تست</h2>

~~~bash
kubectl delete -f examples/deployment.yaml
kubectl delete -f examples/pvc.yaml
~~~

~~~bash
kubectl get pv
sudo find /mnt/nfs/kubernetes -mindepth 1 -maxdepth 2 -ls
~~~

<p dir="rtl" align="right">اگر هیچ PVCای به Provisioner وابسته نیست، منابع را برعکس ترتیب نصب پاک کنید:</p>

~~~bash
kubectl delete -f manifests/class.yaml
kubectl delete -k manifests/deployment/
kubectl delete -f manifests/rbac.yaml
kubectl delete -f manifests/namespace.yaml
~~~

> [!CAUTION]
> <p dir="rtl" align="right">قبل از Cleanup، وضعیت PVها، پوشه‌های داده و Backupها را بررسی کنید.</p>

<h2 dir="rtl" align="right">خطاهای رایج</h2>

<h3 dir="rtl" align="right"><span dir="ltr">Deployment</span> آماده نمی‌شود</h3>

~~~bash
kubectl get pods -n nfs-provisioner -o wide
kubectl describe pods -n nfs-provisioner
kubectl get events -n nfs-provisioner --sort-by=.lastTimestamp
kubectl logs -n nfs-provisioner deployment/nfs-client-provisioner --tail=200
~~~

<h3 dir="rtl" align="right"><span dir="ltr">NFS Mount</span> خطا می‌دهد</h3>

~~~bash
sudo apt-get install -y nfs-common
sudo ss -tn | grep ':2049'
sudo journalctl -u kubelet -n 200 --no-pager
~~~

<p dir="rtl" align="right">تست Mount دستی مرحله ۱ را دوباره انجام دهید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">PVC</span> در حالت <span dir="ltr">Pending</span> می‌ماند</h3>

~~~bash
kubectl describe pvc nfs-dynamic-claim
kubectl get events --sort-by=.lastTimestamp
kubectl logs -n nfs-provisioner deployment/nfs-client-provisioner --tail=200
~~~

<p dir="rtl" align="right">این دو مقدار را مقایسه کنید:</p>

~~~bash
kubectl get storageclass nfs-client -o jsonpath='{.provisioner}{"\n"}'
kubectl get deployment nfs-client-provisioner \
  -n nfs-provisioner \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="PROVISIONER_NAME")].value}{"\n"}'
~~~

<p dir="rtl" align="right">خروجی هر دو دستور باید یکی باشد.</p>

<h3 dir="rtl" align="right"><span dir="ltr">Permission Denied</span></h3>

~~~bash
sudo exportfs -v
sudo namei -l /mnt/nfs/kubernetes
sudo stat -c '%A %u:%g %n' /mnt/nfs/kubernetes
~~~

~~~bash
kubectl exec deployment/nfs-rwx-demo -- id
kubectl exec deployment/nfs-rwx-demo -- stat -c '%A %u:%g %n' /data
~~~

<p dir="rtl" align="right">Export Optionها، Permission مسیر، Security Context و UID/GID را بررسی کنید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">Kustomize Patch</span> اعمال نمی‌شود</h3>

~~~bash
kubectl kustomize manifests/deployment/
~~~

<p dir="rtl" align="right">اسم Deployment، Container و Volume در Patch باید با فایل اصلی یکی باشد:</p>

~~~text
Deployment: nfs-client-provisioner
Container:  nfs-client-provisioner
Volume:     nfs-client-root
~~~

<h2 dir="rtl" align="right">نکته‌های Production</h2>

<ul dir="rtl" align="right">
  <li>NFS Server یک سرویس مشترک و در حالت تک‌سرور Single Point of Failure است.</li>
  <li>Capacity، Inode، Latency، Mount Failure و خطاهای Provisioner را مانیتور کنید.</li>
  <li>از داده‌های برنامه‌ها جدا از PV/PVC Backup بگیرید.</li>
  <li>Export را فقط برای Workerهای مورد اعتماد باز کنید.</li>
  <li>ریسک <code dir="ltr">no_root_squash</code> و <code dir="ltr">insecure</code> را ثبت و بررسی کنید.</li>
  <li>مقدار Provisioner در StorageClass و Deployment را همیشه یکسان نگه دارید.</li>
  <li>مقدار Storage داخل PVC را به‌عنوان Quota واقعی در نظر نگیرید.</li>
</ul>

<h2 dir="rtl" align="right">چک‌لیست نهایی</h2>

<ul dir="rtl" align="right">
  <li>☐ NFS Server و Export فعال هستند.</li>
  <li>☐ Export فقط برای Workerهای مجاز باز است.</li>
  <li>☐ همه Workerها nfs-common دارند و می‌توانند Export را Mount کنند.</li>
  <li>☐ Namespace و RBAC ساخته شده‌اند.</li>
  <li>☐ Patch هم Environment Variableها و هم Volume را تغییر می‌دهد.</li>
  <li>☐ Provisioner Deployment آماده است.</li>
  <li>☐ Provisioner در StorageClass و Deployment یکی است.</li>
  <li>☐ PVC تست به حالت <code dir="ltr">Bound</code> می‌رسد.</li>
  <li>☐ هر دو Pod روی RWX Volume می‌نویسند.</li>
  <li>☐ پوشه مربوط به PVC روی NFS Server ساخته شده است.</li>
  <li>☐ رفتار حذف و Backup داده تست شده است.</li>
</ul>

<h2 dir="rtl" align="right">منابع</h2>

<ul dir="rtl" align="right">
  <li><a href="https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner">NFS Subdirectory External Provisioner</a></li>
  <li><a href="https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/deploy/deployment.yaml">Upstream Deployment</a></li>
  <li><a href="https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/deploy/rbac.yaml">Upstream RBAC</a></li>
  <li><a href="https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/deploy/class.yaml">Upstream StorageClass</a></li>
</ul>
