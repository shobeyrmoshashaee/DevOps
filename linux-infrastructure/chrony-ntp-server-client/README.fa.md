<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">🕒 راه‌اندازی <span dir="ltr">Chrony NTP Server</span> و <span dir="ltr">Client</span></h1>

<p dir="rtl" align="right">اینجا دو <span dir="ltr">Chrony Server</span> راه‌اندازی می‌کنیم تا سرورهای <span dir="ltr">Linux</span> ساعتشان را از آن‌ها بگیرند. اگر سرور اصلی در دسترس نبود، سرور دوم جای آن را می‌گیرد.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">همه <span dir="ltr">IP</span>های این راهنما نمونه هستند. موقع اجرا، آن‌ها را با <span dir="ltr">IP</span>های محیط خودتان عوض کنید.</p>

<a id="table-of-contents"></a>
<h2 dir="rtl" align="right">فهرست مطالب</h2>

<ul dir="rtl" align="right">
  <li><a href="#architecture">معماری</a></li>
  <li><a href="#configure-servers">۱. تنظیم Chrony Serverها</a></li>
  <li><a href="#configure-clients">۲. تنظیم Clientها</a></li>
  <li><a href="#health-output">۳. خواندن خروجی‌های Health Check</a></li>
  <li><a href="#troubleshooting">۴. خطاهای رایج</a></li>
  <li><a href="#final-check">۵. بررسی نهایی</a></li>
</ul>

<a id="architecture"></a>
<h2 dir="rtl" align="right">معماری</h2>

~~~mermaid
flowchart TD
    U["Public NTP Sources"]
    P["Primary Chrony Server<br/>192.0.2.10"]
    S["Secondary Chrony Server<br/>192.0.2.11"]
    C["Linux Clients<br/>192.0.2.0/24"]

    U --> P
    U --> S
    P --> C
    S --> C
~~~

<table dir="rtl">
  <thead>
    <tr>
      <th align="right">بخش</th>
      <th align="right">آدرس</th>
      <th align="right">کاربرد</th>
    </tr>
  </thead>
  <tbody>
    <tr><td align="right">Upstream</td><td align="right">Public NTP Sources</td><td align="right">ساعت مرجع را به Chrony Serverها می‌دهد</td></tr>
    <tr><td align="right">Primary Server</td><td dir="ltr"><code>192.0.2.10</code></td><td align="right">سرور اصلی NTP</td></tr>
    <tr><td align="right">Secondary Server</td><td dir="ltr"><code>192.0.2.11</code></td><td align="right">سرور جایگزین NTP</td></tr>
    <tr><td align="right">Client Network</td><td dir="ltr"><code>192.0.2.0/24</code></td><td align="right">سرورهایی که ساعتشان را Sync می‌کنند</td></tr>
  </tbody>
</table>

> [!NOTE]
> <p dir="rtl" align="right">در این مثال فقط شبکه <code dir="ltr">192.0.2.0/24</code> اجازه استفاده از <span dir="ltr">NTP</span> را دارد. این شبکه را با <span dir="ltr">Subnet</span> واقعی خودتان عوض کنید.</p>

<a id="configure-servers"></a>
<h2 dir="rtl" align="right">۱. تنظیم <span dir="ltr">Chrony Server</span>ها</h2>

<p dir="rtl" align="right">این مراحل را روی هر دو سرور انجام دهید.</p>

<h3 dir="rtl" align="right">نصب <span dir="ltr">Chrony</span></h3>

~~~bash
sudo apt update
sudo apt install -y chrony
~~~

<p dir="rtl" align="right">قبل از تغییر فایل تنظیمات، از آن <span dir="ltr">Backup</span> بگیرید:</p>

~~~bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
sudo nano /etc/chrony/chrony.conf
~~~

<p dir="rtl" align="right">خطوط پیش‌فرض <code dir="ltr">server</code> و <code dir="ltr">pool</code> را حذف یا <span dir="ltr">Comment</span> کنید و این تنظیمات را قرار دهید:</p>

~~~conf
# External NTP sources
server 217.218.127.128 iburst
server ntp.time.ir iburst
pool 2.ir.pool.ntp.org iburst maxsources 4
pool 0.asia.pool.ntp.org iburst maxsources 4

# Allow NTP clients only from the authorized internal network
allow 192.0.2.0/24

# Correct large offsets during the first three updates
makestep 1.0 3

# Synchronize the hardware clock
rtcsync
~~~

<h3 dir="rtl" align="right">معنی گزینه‌های مهم</h3>

<table dir="rtl">
  <thead><tr><th align="right">گزینه</th><th align="right">کاربرد</th></tr></thead>
  <tbody>
    <tr><td dir="ltr"><code>iburst</code></td><td align="right">در شروع کار چند Request سریع می‌فرستد تا Sync زودتر انجام شود</td></tr>
    <tr><td dir="ltr"><code>maxsources 4</code></td><td align="right">حداکثر چهار Source را از یک Pool انتخاب می‌کند</td></tr>
    <tr><td dir="ltr"><code>allow 192.0.2.0/24</code></td><td align="right">فقط به این شبکه اجازه استفاده از NTP را می‌دهد</td></tr>
    <tr><td dir="ltr"><code>makestep 1.0 3</code></td><td align="right">اگر اختلاف ساعت زیاد باشد، در سه Update اول آن را سریع اصلاح می‌کند</td></tr>
    <tr><td dir="ltr"><code>rtcsync</code></td><td align="right">ساعت سخت‌افزاری را با ساعت سیستم Sync می‌کند</td></tr>
  </tbody>
</table>

> [!WARNING]
> <p dir="rtl" align="right">گزینه <code dir="ltr">allow</code> را باز نگذارید. فقط شبکه‌هایی را وارد کنید که واقعاً باید به این سرویس وصل شوند.</p>

<h3 dir="rtl" align="right">باز کردن <span dir="ltr">Firewall</span></h3>

<p dir="rtl" align="right"><span dir="ltr">NTP</span> از <span dir="ltr">UDP Port 123</span> استفاده می‌کند. اگر <span dir="ltr">UFW</span> فعال است، این Rule را اضافه کنید:</p>

~~~bash
sudo ufw allow from 192.0.2.0/24 to any port 123 proto udp
~~~

> [!TIP]
> <p dir="rtl" align="right">اگر از <span dir="ltr">Firewall</span> دیگری استفاده می‌کنید، یک Rule مشابه برای <span dir="ltr">UDP/123</span> بسازید و Source را فقط روی شبکه‌های مجاز بگذارید.</p>

<h3 dir="rtl" align="right">اجرای سرویس</h3>

~~~bash
sudo systemctl enable chrony
sudo systemctl restart chrony
sudo systemctl --no-pager status chrony
~~~

<h3 dir="rtl" align="right">تست سرورها</h3>

~~~bash
chronyc sources -v
chronyc tracking
chronyc sourcestats -v
chronyc selectdata
chronyc clients
~~~

> [!NOTE]
> <p dir="rtl" align="right">دستور <code dir="ltr">chronyc clients</code> زمانی Clientها را نشان می‌دهد که حداقل یک Request به سرور فرستاده باشند.</p>

<a id="configure-clients"></a>
<h2 dir="rtl" align="right">۲. تنظیم <span dir="ltr">Client</span>ها</h2>

<p dir="rtl" align="right">این مراحل را روی همه سرورهای <span dir="ltr">Linux</span> که قرار است ساعتشان Sync شود انجام دهید.</p>

<h3 dir="rtl" align="right">نصب <span dir="ltr">Chrony</span></h3>

~~~bash
sudo apt update
sudo apt install -y chrony
~~~

<p dir="rtl" align="right">از تنظیمات فعلی <span dir="ltr">Backup</span> بگیرید:</p>

~~~bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
sudo nano /etc/chrony/chrony.conf
~~~

<p dir="rtl" align="right">خطوط <code dir="ltr">server</code> و <code dir="ltr">pool</code> قبلی را حذف یا <span dir="ltr">Comment</span> کنید و دو سرور داخلی را قرار دهید:</p>

~~~conf
# Internal Chrony servers
server 192.0.2.10 iburst prefer
server 192.0.2.11 iburst

# Correct large offsets during the first three updates
makestep 1.0 3

# Synchronize the hardware clock
rtcsync
~~~

> [!NOTE]
> <p dir="rtl" align="right">تا وقتی سرور اصلی سالم است، <span dir="ltr">Chrony</span> آن را ترجیح می‌دهد. اگر سرور اصلی در دسترس نباشد، سرور دوم انتخاب می‌شود.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">روی <span dir="ltr">Client</span> نیازی به گزینه <code dir="ltr">allow</code> نداریم؛ چون Client فقط ساعت می‌گیرد و قرار نیست به بقیه سرویس بدهد.</p>

<h3 dir="rtl" align="right">اجرای سرویس</h3>

~~~bash
sudo systemctl enable chrony
sudo systemctl restart chrony
sudo systemctl --no-pager status chrony
~~~

<h3 dir="rtl" align="right">تست <span dir="ltr">Client</span></h3>

~~~bash
chronyc sources -v
chronyc tracking
chronyc sourcestats -v
chronyc selectdata
timedatectl status
~~~

<p dir="rtl" align="right">یک خروجی سالم می‌تواند شبیه این باشد:</p>

~~~text
^* 192.0.2.10
^+ 192.0.2.11
~~~

<p dir="rtl" align="right">ممکن است Source انتخاب‌شده با توجه به کیفیت و دسترسی تغییر کند؛ این رفتار طبیعی است.</p>

<a id="health-output"></a>
<h2 dir="rtl" align="right">۳. خواندن خروجی‌های <span dir="ltr">Health Check</span></h2>

<h3 dir="rtl" align="right"><code dir="ltr">chronyc sources -v</code></h3>

<table dir="rtl">
  <thead><tr><th align="right">علامت</th><th align="right">معنی</th></tr></thead>
  <tbody>
    <tr><td dir="ltr"><code>^*</code></td><td align="right">Source اصلی که الان برای Sync انتخاب شده است</td></tr>
    <tr><td dir="ltr"><code>^+</code></td><td align="right">Source سالمی که آماده استفاده است</td></tr>
    <tr><td dir="ltr"><code>^-</code></td><td align="right">Source معتبر است ولی فعلاً انتخاب نشده</td></tr>
    <tr><td dir="ltr"><code>^?</code></td><td align="right">Source در دسترس نیست یا هنوز اطلاعات کافی ندارد</td></tr>
    <tr><td dir="ltr"><code>Reach</code></td><td align="right">بعد از چند Poll موفق معمولاً به <code dir="ltr">377</code> می‌رسد</td></tr>
  </tbody>
</table>

<h3 dir="rtl" align="right"><code dir="ltr">chronyc tracking</code></h3>

<p dir="rtl" align="right">در حالت سالم معمولاً این موارد را می‌بینیم:</p>

<ul dir="rtl" align="right">
  <li><code dir="ltr">Leap status : Normal</code></li>
  <li><span dir="ltr">Reference ID</span> معتبر</li>
  <li><span dir="ltr">Stratum</span> بزرگ‌تر از صفر</li>
  <li>مقدار کم برای <span dir="ltr">System Time</span> و <span dir="ltr">Last Offset</span></li>
</ul>

<h3 dir="rtl" align="right"><code dir="ltr">timedatectl status</code></h3>

<p dir="rtl" align="right">این دو مقدار باید دیده شوند:</p>

~~~text
System clock synchronized: yes
NTP service: active
~~~

<a id="troubleshooting"></a>
<h2 dir="rtl" align="right">۴. خطاهای رایج</h2>

<h3 dir="rtl" align="right"><span dir="ltr">Source</span>ها با علامت <code dir="ltr">^?</code> می‌مانند</h3>

<p dir="rtl" align="right"><span dir="ltr">DNS</span>، <span dir="ltr">Route</span>، <span dir="ltr">Firewall</span> و <span dir="ltr">UDP/123</span> را بررسی کنید:</p>

~~~bash
getent hosts ntp.time.ir
getent hosts 2.ir.pool.ntp.org
sudo ss -ulpn | grep ':123'
~~~

<p dir="rtl" align="right">بعد وضعیت سرویس و Logها را ببینید:</p>

~~~bash
sudo systemctl status chrony
sudo journalctl -u chrony -n 100 --no-pager
~~~

> [!CAUTION]
> <p dir="rtl" align="right">برای تست، <span dir="ltr">Firewall</span> را کامل خاموش نکنید. فقط دسترسی <span dir="ltr">UDP/123</span> را بررسی کنید.</p>

<h3 dir="rtl" align="right">برگرداندن تنظیمات قبلی</h3>

<p dir="rtl" align="right">اگر تنظیمات جدید مشکل داشت، فایل <span dir="ltr">Backup</span> را برگردانید:</p>

~~~bash
sudo cp /etc/chrony/chrony.conf.bak /etc/chrony/chrony.conf
sudo systemctl restart chrony
~~~

<a id="final-check"></a>
<h2 dir="rtl" align="right">۵. بررسی نهایی</h2>

<h3 dir="rtl" align="right">روی <span dir="ltr">Chrony Server</span>ها</h3>

~~~bash
chronyc sources -v
chronyc tracking
chronyc clients
~~~

<h3 dir="rtl" align="right">روی <span dir="ltr">Client</span>ها</h3>

~~~bash
chronyc sources -v
chronyc tracking
timedatectl status
~~~

<h3 dir="rtl" align="right">چک‌لیست نهایی</h3>

<ul dir="rtl" align="right">
  <li>☐ هر دو Chrony Server به Sourceهای بیرونی وصل هستند.</li>
  <li>☐ سرویس Chrony روی سرورها و Clientها فعال است.</li>
  <li>☐ Clientها به UDP Port 123 سرورها دسترسی دارند.</li>
  <li>☐ روی هر Client یک Source با <code dir="ltr">^*</code> دیده می‌شود.</li>
  <li>☐ اگر سرور اصلی قطع شود، سرور دوم در دسترس است.</li>
  <li>☐ خروجی <code dir="ltr">chronyc tracking</code> مقدار <code dir="ltr">Leap status : Normal</code> دارد.</li>
  <li>☐ خروجی <code dir="ltr">timedatectl status</code> نشان می‌دهد ساعت Sync شده است.</li>
  <li>☐ دستور <code dir="ltr">chronyc clients</code> روی سرورها Clientهای موردنظر را نشان می‌دهد.</li>
</ul>

<h2 dir="rtl" align="right">منابع</h2>

<ul dir="rtl" align="right">
  <li><a href="https://chrony-project.org/doc/latest/chrony.conf.html">Chrony Configuration Reference</a></li>
  <li><a href="https://chrony-project.org/documentation.html">Chrony Documentation</a></li>
</ul>
