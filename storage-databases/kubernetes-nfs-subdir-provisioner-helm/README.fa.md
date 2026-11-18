<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">📦 راه‌اندازی <span dir="ltr">NFS Dynamic Provisioning</span> در <span dir="ltr">Kubernetes</span> با <span dir="ltr">Helm</span></h1>

<p dir="rtl" align="right">اینجا یک <span dir="ltr">NFS Server</span> می‌سازیم و <span dir="ltr">NFS Subdirectory External Provisioner</span> را با <span dir="ltr">Helm</span> روی Kubernetes نصب می‌کنیم. بعد از آن، هر <span dir="ltr">PVC</span> به‌صورت خودکار یک <span dir="ltr">PV</span> و یک پوشه روی NFS می‌گیرد.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">این پروژه مخصوص روش نصب با <span dir="ltr">Helm</span> است. روش نصب با فایل‌های <span dir="ltr">Deployment</span> در پروژه جدا قرار دارد.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">همه IPها، مسیرها و اسم‌ها نمونه هستند. موقع اجرا، مقادیر محیط خودتان را جایگزین کنید.</p>

<h2 dir="rtl" align="right">معماری</h2>

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

<h2 dir="rtl" align="right">اجزای اصلی</h2>

<table dir="rtl">
  <thead><tr><th align="right">بخش</th><th align="right">کاربرد</th></tr></thead>
  <tbody>
    <tr><td align="right">NFS Server</td><td align="right">فضای Shared را در اختیار Nodeها می‌گذارد</td></tr>
    <tr><td align="right">nfs-common</td><td align="right">امکان Mount کردن NFS روی Nodeها را می‌دهد</td></tr>
    <tr><td align="right">NFS Provisioner</td><td align="right">برای هر PVC یک پوشه و PV می‌سازد</td></tr>
    <tr><td align="right">StorageClass</td><td align="right">Dynamic Provisioning را در اختیار Workloadها می‌گذارد</td></tr>
    <tr><td align="right">Helm</td><td align="right">Provisioner را نصب و مدیریت می‌کند</td></tr>
  </tbody>
</table>

<h2 dir="rtl" align="right">فایل‌های پروژه</h2>

~~~text
kubernetes-nfs-subdir-provisioner-helm/
├── README.md
├── README.fa.md
├── values.yaml
└── examples/
    ├── pvc.yaml
    └── pod.yaml
~~~

<h2 dir="rtl" align="right">۱. نصب و تنظیم <span dir="ltr">NFS Server</span></h2>

<table dir="rtl">
  <thead><tr><th align="right">تنظیم</th><th align="right">مقدار مثال</th></tr></thead>
  <tbody>
    <tr><td align="right">NFS Server</td><td dir="ltr"><code>192.0.2.50</code></td></tr>
    <tr><td align="right">Node Network</td><td dir="ltr"><code>192.0.2.0/24</code></td></tr>
    <tr><td align="right">Export Path</td><td dir="ltr"><code>/var/nfs/kubernetes</code></td></tr>
    <tr><td align="right">NFS Version</td><td dir="ltr"><code>4.1</code></td></tr>
    <tr><td align="right">StorageClass</td><td dir="ltr"><code>nfs-client</code></td></tr>
    <tr><td align="right">Namespace</td><td dir="ltr"><code>nfs-storage</code></td></tr>
  </tbody>
</table>

<h3 dir="rtl" align="right">نصب <span dir="ltr">NFS Server</span></h3>

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
sudo systemctl enable --now nfs-kernel-server
sudo systemctl --no-pager status nfs-kernel-server
~~~

<h3 dir="rtl" align="right">ساخت مسیر <span dir="ltr">Export</span></h3>

~~~bash
sudo install \
  -d \
  -o nobody \
  -g nogroup \
  -m 0770 \
  /var/nfs/kubernetes
~~~

~~~bash
sudo stat -c '%A %U:%G %n' /var/nfs/kubernetes
~~~

~~~text
drwxrwx--- nobody:nogroup /var/nfs/kubernetes
~~~

> [!NOTE]
> <p dir="rtl" align="right">در این پروژه <span dir="ltr">UID/GID 65534</span> استفاده شده که معمولاً روی Ubuntu و Debian همان <code dir="ltr">nobody:nogroup</code> است. قبل از اجرا این مقدار را روی سیستم خودتان چک کنید.</p>

<h3 dir="rtl" align="right">تنظیم <span dir="ltr">NFS Export</span></h3>

~~~bash
sudo nano /etc/exports.d/kubernetes-nfs.exports
~~~

~~~exports
/var/nfs/kubernetes 192.0.2.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
~~~

~~~bash
sudo exportfs -rav
sudo exportfs -v
~~~

> [!WARNING]
> <p dir="rtl" align="right">با <code dir="ltr">no_root_squash</code>، کاربر root روی Node می‌تواند فایل‌های NFS را با دسترسی root تغییر دهد یا پاک کند. Export را فقط برای Nodeهای مورد اعتماد باز کنید.</p>

> [!CAUTION]
> <p dir="rtl" align="right">گزینه <code dir="ltr">insecure</code> اتصال از Source Portهای بالاتر از 1024 را قبول می‌کند. فقط زمانی از آن استفاده کنید که واقعاً لازم است و دسترسی را با Firewall محدود کنید.</p>

> [!NOTE]
> <p dir="rtl" align="right">گزینه <code dir="ltr">no_all_squash</code> باعث می‌شود UID و GID کاربران حفظ شود. پس UID/GID روی Nodeها و Podها باید هماهنگ باشد.</p>

<h3 dir="rtl" align="right">باز کردن <span dir="ltr">Firewall</span></h3>

<p dir="rtl" align="right">برای <span dir="ltr">NFSv4.1</span> فقط <span dir="ltr">TCP Port 2049</span> را برای شبکه Nodeها باز کنید:</p>

~~~bash
sudo ufw allow \
  from 192.0.2.0/24 \
  to any port 2049 \
  proto tcp
~~~

<h2 dir="rtl" align="right">۲. آماده کردن <span dir="ltr">Kubernetes Node</span>ها</h2>

<p dir="rtl" align="right">روی همه Nodeهایی که ممکن است Pod دارای NFS Volume روی آن‌ها اجرا شود، این پکیج را نصب کنید:</p>

~~~bash
sudo apt-get update
sudo apt-get install -y nfs-common
dpkg -l nfs-common
~~~

<p dir="rtl" align="right">قبل از نصب Provisioner، Mount را روی هر Node تست کنید:</p>

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
> <p dir="rtl" align="right">تا وقتی Mount روی همه Nodeهای موردنظر درست نشده، سراغ نصب Provisioner نروید.</p>

<h2 dir="rtl" align="right">۳. نصب <span dir="ltr">Provisioner</span> با <span dir="ltr">Helm</span></h2>

<h3 dir="rtl" align="right">اضافه کردن <span dir="ltr">Helm Repository</span></h3>

~~~bash
helm repo add \
  nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm repo update
helm search repo nfs-subdir-external-provisioner
~~~

<h3 dir="rtl" align="right">بررسی فایل <code dir="ltr">values.yaml</code></h3>

<p dir="rtl" align="right">فایل <code dir="ltr">values.yaml</code> شامل NFS Server، مسیر Export، نسخه NFS، اسم StorageClass، Resourceها و Permission پوشه‌هاست. قبل از نصب همه مقادیر را با محیط خودتان تطبیق دهید.</p>

> [!WARNING]
> <p dir="rtl" align="right">قبل از Production، مقدارهای <code dir="ltr">reclaimPolicy</code> و <code dir="ltr">archiveOnDelete</code> را دقیق بررسی کنید؛ چون مشخص می‌کنند بعد از حذف PVC چه اتفاقی برای داده می‌افتد.</p>

<h3 dir="rtl" align="right">نصب یا <span dir="ltr">Upgrade</span></h3>

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
> <p dir="rtl" align="right">در Production بهتر است با گزینه <code dir="ltr">--version</code> نسخه Chart را ثابت نگه دارید.</p>

~~~bash
helm list -n nfs-storage
helm status nfs-subdir-external-provisioner -n nfs-storage
helm get values nfs-subdir-external-provisioner -n nfs-storage
~~~

<h2 dir="rtl" align="right">۴. بررسی نصب</h2>

~~~bash
kubectl get namespace nfs-storage
kubectl get all -n nfs-storage
kubectl rollout status \
  deployment/nfs-subdir-external-provisioner \
  -n nfs-storage \
  --timeout=180s
~~~

~~~bash
kubectl logs \
  -n nfs-storage \
  deployment/nfs-subdir-external-provisioner \
  --tail=100
~~~

<p dir="rtl" align="right"><span dir="ltr">StorageClass</span> را هم چک کنید:</p>

~~~bash
kubectl get storageclass nfs-client
kubectl describe storageclass nfs-client
~~~

> [!NOTE]
> <p dir="rtl" align="right">مقدار <span dir="ltr">Provisioner</span> را از Cluster بخوانید و از روی خروجی قدیمی کپی نکنید؛ این مقدار ممکن است با تنظیمات Chart فرق کند.</p>

<h2 dir="rtl" align="right">۵. تست <span dir="ltr">Dynamic Provisioning</span></h2>

<p dir="rtl" align="right">اول PVC تست را بسازید:</p>

~~~bash
kubectl apply -f examples/pvc.yaml
kubectl get pvc nfs-dynamic-test -w
~~~

<p dir="rtl" align="right">وضعیت PVC باید <code dir="ltr">Bound</code> شود:</p>

~~~bash
kubectl describe pvc nfs-dynamic-test
kubectl get pv
~~~

<p dir="rtl" align="right">حالا PVC را داخل Pod Mount کنید:</p>

~~~bash
kubectl apply -f examples/pod.yaml
kubectl wait \
  --for=condition=Ready \
  pod/nfs-rwx-test \
  --timeout=120s
~~~

<p dir="rtl" align="right">خواندن و نوشتن را تست کنید:</p>

~~~bash
kubectl exec nfs-rwx-test -- cat /data/healthcheck.txt
kubectl exec nfs-rwx-test -- sh -c 'echo manual-check >> /data/healthcheck.txt'
kubectl exec nfs-rwx-test -- tail /data/healthcheck.txt
~~~

<p dir="rtl" align="right">روی NFS Server هم باید یک پوشه جدید ساخته شده باشد:</p>

~~~bash
sudo find /var/nfs/kubernetes \
  -mindepth 1 \
  -maxdepth 2 \
  -printf '%M %u:%g %p\n'
~~~

<h2 dir="rtl" align="right">۶. پاک کردن تست</h2>

~~~bash
kubectl delete -f examples/pod.yaml
kubectl delete -f examples/pvc.yaml
~~~

~~~bash
kubectl get pv
sudo find /var/nfs/kubernetes -mindepth 1 -maxdepth 2 -ls
~~~

> [!IMPORTANT]
> <p dir="rtl" align="right">با <code dir="ltr">reclaimPolicy: Delete</code> و <code dir="ltr">archiveOnDelete: true</code>، PV حذف می‌شود و پوشه داده معمولاً Rename می‌شود. قبل از Production این رفتار را خودتان تست کنید.</p>

<p dir="rtl" align="right">برای حذف Release:</p>

~~~bash
helm uninstall \
  nfs-subdir-external-provisioner \
  -n nfs-storage
~~~

<p dir="rtl" align="right">تا وقتی تکلیف نگهداری داده مشخص نشده، Namespace، Export یا پوشه‌های Archiveشده را پاک نکنید.</p>

<h2 dir="rtl" align="right">خطاهای رایج</h2>

<h3 dir="rtl" align="right"><span dir="ltr">PVC</span> در حالت <span dir="ltr">Pending</span> می‌ماند</h3>

~~~bash
kubectl describe pvc nfs-dynamic-test
kubectl get events --sort-by=.lastTimestamp
kubectl logs \
  -n nfs-storage \
  deployment/nfs-subdir-external-provisioner \
  --tail=200
~~~

<p dir="rtl" align="right">بررسی کنید PVC از <code dir="ltr">storageClassName: nfs-client</code> استفاده کند.</p>

<h3 dir="rtl" align="right"><span dir="ltr">Provisioner Pod</span> آماده نمی‌شود</h3>

~~~bash
kubectl get pods -n nfs-storage -o wide
kubectl describe pods -n nfs-storage
kubectl get events -n nfs-storage --sort-by=.lastTimestamp
~~~

<p dir="rtl" align="right">از Nodeای که Pod روی آن اجرا شده، دسترسی به NFS Server را تست کنید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">Mount</span> روی Node خطا می‌دهد</h3>

~~~bash
sudo journalctl -u kubelet -n 200 --no-pager
sudo dmesg | tail -n 100
sudo ss -tn | grep ':2049'
dpkg -l nfs-common
~~~

<p dir="rtl" align="right">تست Mount دستی مرحله ۲ را دوباره انجام دهید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">Permission Denied</span></h3>

~~~bash
sudo exportfs -v
sudo namei -l /var/nfs/kubernetes
sudo stat -c '%A %u:%g %n' /var/nfs/kubernetes
~~~

~~~bash
kubectl exec nfs-rwx-test -- id
kubectl exec nfs-rwx-test -- stat -c '%A %u:%g %n' /data
~~~

<p dir="rtl" align="right">مقدارهای <code dir="ltr">runAsUser</code>، <code dir="ltr">runAsGroup</code> و <code dir="ltr">fsGroup</code> را با UID/GID مسیر NFS هماهنگ کنید.</p>

<h3 dir="rtl" align="right">نصب <span dir="ltr">Helm</span> خطا می‌دهد</h3>

~~~bash
helm status nfs-subdir-external-provisioner -n nfs-storage
helm get manifest nfs-subdir-external-provisioner -n nfs-storage
kubectl get events -n nfs-storage --sort-by=.lastTimestamp
~~~

<p dir="rtl" align="right">دسترسی به Helm Repository و امکان Pull کردن Image را بررسی کنید.</p>

<h2 dir="rtl" align="right">نکته‌های Production</h2>

<ul dir="rtl" align="right">
  <li>NFS یک سرویس مشترک است؛ Availability، Latency، Capacity، Inode و خطاهای Mount را مانیتور کنید.</li>
  <li>همه Worker Nodeهای مجاز باید به Export دسترسی داشته باشند.</li>
  <li>از داده‌های Production جدا از Provisioner بکاپ بگیرید.</li>
  <li>رفتار حذف و Restore کردن PVC را قبل از Production تست کنید.</li>
  <li>دسترسی <code dir="ltr">no_root_squash</code> و <code dir="ltr">insecure</code> را فقط به Nodeهای مورد اعتماد بدهید.</li>
  <li>نسخه Chart و Image را ثابت نگه دارید.</li>
  <li>یک NFS Server تکی، Single Point of Failure است.</li>
</ul>

<h2 dir="rtl" align="right">چک‌لیست نهایی</h2>

<ul dir="rtl" align="right">
  <li>☐ سرویس NFS فعال است.</li>
  <li>☐ مسیر Export با Owner و Permission درست ساخته شده است.</li>
  <li>☐ Export فقط برای شبکه Nodeهای مجاز باز است.</li>
  <li>☐ ریسک <code dir="ltr">no_root_squash</code> و <code dir="ltr">insecure</code> بررسی شده است.</li>
  <li>☐ همه Nodeها پکیج <code dir="ltr">nfs-common</code> دارند و می‌توانند Export را Mount کنند.</li>
  <li>☐ Helm Release و Provisioner Pod سالم هستند.</li>
  <li>☐ StorageClass با نام <code dir="ltr">nfs-client</code> وجود دارد.</li>
  <li>☐ PVC تست به حالت <code dir="ltr">Bound</code> می‌رسد.</li>
  <li>☐ Pod می‌تواند روی Volume بخواند و بنویسد.</li>
  <li>☐ پوشه مربوط به PVC روی NFS Server ساخته می‌شود.</li>
  <li>☐ رفتار حذف و نگهداری داده تست شده است.</li>
</ul>

<h2 dir="rtl" align="right">منابع</h2>

<ul dir="rtl" align="right">
  <li><a href="https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner">NFS Subdirectory External Provisioner</a></li>
  <li><a href="https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/charts/nfs-subdir-external-provisioner">Official Helm Chart</a></li>
  <li><a href="https://manpages.ubuntu.com/manpages/focal/man5/exports.5.html">Ubuntu NFS Exports Manual</a></li>
</ul>
