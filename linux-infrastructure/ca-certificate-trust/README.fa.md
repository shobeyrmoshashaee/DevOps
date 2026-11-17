<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">🔐 نصب <span dir="ltr">CA Certificate</span> روی <span dir="ltr">Ubuntu و Debian</span></h1>

<p dir="rtl" align="right">اینجا می‌خواهیم یک <span dir="ltr">CA Certificate</span> داخلی را به <span dir="ltr">Ubuntu و Debian</span> معرفی کنیم. بعد از این کار، سیستم به <span dir="ltr">Certificate</span>هایی که با این <span dir="ltr">CA</span> صادر شده‌اند اعتماد می‌کند.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">اسم فایل‌ها، <span dir="ltr">Domain</span>ها و <span dir="ltr">Endpoint</span>های این راهنما فقط مثال هستند. موقع اجرا، مقادیر محیط خودتان را جایگزین کنید.</p>

> [!CAUTION]
> <p dir="rtl" align="right">برای این کار فقط فایل <span dir="ltr">Public CA</span> لازم است. <span dir="ltr">Private Key</span>، <span dir="ltr">Password</span>، <span dir="ltr">Token</span> یا فایل <span dir="ltr">PKCS#12</span> را داخل ریپو قرار ندهید.</p>

<a id="table-of-contents"></a>
<h2 dir="rtl" align="right">فهرست مطالب</h2>

<ul dir="rtl" align="right">
  <li><a href="#trust-store-locations">مسیرهای مهم</a></li>
  <li><a href="#install-required-packages">۱. نصب ابزارهای لازم</a></li>
  <li><a href="#inspect-ca-certificate">۲. بررسی فایل CA</a></li>
  <li><a href="#install-ca-certificate">۳. اضافه کردن CA به سیستم</a></li>
  <li><a href="#validate-installation">۴. بررسی نتیجه</a></li>
  <li><a href="#test-tls-endpoint">۵. تست اتصال HTTPS</a></li>
  <li><a href="#remove-replace-ca">۶. حذف یا جایگزینی CA</a></li>
  <li><a href="#troubleshooting">خطاهای رایج</a></li>
  <li><a href="#acceptance-checklist">چک‌لیست نهایی</a></li>
</ul>

<a id="trust-store-locations"></a>
<h2 dir="rtl" align="right">مسیرهای مهم</h2>

<table dir="rtl">
  <thead>
    <tr>
      <th align="right">مسیر</th>
      <th align="right">کاربرد</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td dir="ltr"><code>/usr/local/share/ca-certificates/</code></td>
      <td align="right">فایل CA را در این مسیر قرار می‌دهیم</td>
    </tr>
    <tr>
      <td dir="ltr"><code>/etc/ssl/certs/</code></td>
      <td align="right">سیستم لینک‌های مربوط به Certificateها را اینجا می‌سازد</td>
    </tr>
    <tr>
      <td dir="ltr"><code>/etc/ssl/certs/ca-certificates.crt</code></td>
      <td align="right">فایل اصلی CAهای مورد اعتماد سیستم است</td>
    </tr>
  </tbody>
</table>

> [!NOTE]
> <p dir="rtl" align="right">فایل <span dir="ltr">CA</span> باید پسوند <code dir="ltr">.crt</code> داشته باشد. داخل هر فایل هم فقط یک <span dir="ltr">Certificate</span> قرار دهید.</p>

<a id="install-required-packages"></a>
<h2 dir="rtl" align="right">۱. نصب ابزارهای لازم</h2>

<p dir="rtl" align="right">اول این دو پکیج را نصب می‌کنیم:</p>

~~~bash
sudo apt update
sudo apt install -y ca-certificates openssl
~~~

<a id="inspect-ca-certificate"></a>
<h2 dir="rtl" align="right">۲. بررسی فایل <span dir="ltr">CA</span></h2>

<p dir="rtl" align="right">قبل از نصب، فرمت فایل را چک می‌کنیم. یک فایل <span dir="ltr">PEM</span> باید با این دو خط شروع و تمام شود:</p>

~~~text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
~~~

<p dir="rtl" align="right">برای دیدن خط اول و آخر فایل، این دستورها را بزنید:</p>

~~~bash
head -n 1 example-internal-ca.crt
tail -n 1 example-internal-ca.crt
~~~

<p dir="rtl" align="right">حالا مشخصات <span dir="ltr">Certificate</span> را می‌بینیم:</p>

~~~bash
openssl x509 \
  -in example-internal-ca.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -fingerprint \
  -sha256
~~~

> [!WARNING]
> <p dir="rtl" align="right"><span dir="ltr">Fingerprint</span> فایل را با مقدار اصلی مقایسه کنید. اگر یکسان نبودند، فایل را نصب نکنید.</p>

<a id="convert-der-to-pem"></a>
<h3 dir="rtl" align="right">تبدیل <span dir="ltr">DER</span> به <span dir="ltr">PEM</span></h3>

<p dir="rtl" align="right">اگر فایل شما <span dir="ltr">DER</span> است، با دستور زیر آن را به <span dir="ltr">PEM</span> تبدیل کنید:</p>

~~~bash
openssl x509 \
  -inform DER \
  -in example-internal-ca.der \
  -out example-internal-ca.crt
~~~

<p dir="rtl" align="right">بعد از تبدیل، فایل را یک‌بار تست کنید:</p>

~~~bash
openssl x509 -in example-internal-ca.crt -noout -subject -issuer
~~~

<a id="install-ca-certificate"></a>
<h2 dir="rtl" align="right">۳. اضافه کردن <span dir="ltr">CA</span> به سیستم</h2>

<p dir="rtl" align="right">حالا فایل <span dir="ltr">CA</span> را با مالک <span dir="ltr">root</span> و دسترسی <span dir="ltr">0644</span> در مسیر زیر قرار می‌دهیم:</p>

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

<p dir="rtl" align="right">بعد لیست <span dir="ltr">CA</span>های سیستم را به‌روزرسانی می‌کنیم:</p>

~~~bash
sudo update-ca-certificates
~~~

<p dir="rtl" align="right">اگر فایل درست اضافه شده باشد، خروجی شبیه این است:</p>

~~~text
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
~~~

> [!NOTE]
> <p dir="rtl" align="right">فایل را مستقیم داخل <code dir="ltr">/etc/ssl/certs/</code> کپی نکنید. این مسیر را خود سیستم مدیریت می‌کند.</p>

<a id="validate-installation"></a>
<h2 dir="rtl" align="right">۴. بررسی نتیجه</h2>

<p dir="rtl" align="right">اول مشخصات فایلی را که نصب کرده‌ایم می‌بینیم:</p>

~~~bash
openssl x509 \
  -in /usr/local/share/ca-certificates/example-internal-ca.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -fingerprint \
  -sha256
~~~

<p dir="rtl" align="right">بعد چک می‌کنیم که لینک آن در مسیر <span dir="ltr">Certificate</span>های سیستم ساخته شده باشد:</p>

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca'
~~~

<p dir="rtl" align="right">اگر خواستید جزئیات بیشتری ببینید، دستور را با گزینه <span dir="ltr">Verbose</span> اجرا کنید:</p>

~~~bash
sudo update-ca-certificates --verbose
~~~

> [!TIP]
> <p dir="rtl" align="right"><span dir="ltr">Fingerprint</span> فایل نصب‌شده باید با مقدار اصلی یکی باشد.</p>

<a id="test-tls-endpoint"></a>
<h2 dir="rtl" align="right">۵. تست اتصال <span dir="ltr">HTTPS</span></h2>

<p dir="rtl" align="right">حالا یک سرویس <span dir="ltr">HTTPS</span> را با <span dir="ltr">OpenSSL</span> تست می‌کنیم:</p>

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

<p dir="rtl" align="right">اگر همه‌چیز درست باشد، این دو خط را می‌بینید:</p>

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

<p dir="rtl" align="right">همان آدرس را با <span dir="ltr">cURL</span> هم تست کنید:</p>

~~~bash
curl -Iv https://service.example.com/
~~~

> [!WARNING]
> <p dir="rtl" align="right">نصب <span dir="ltr">CA</span> فقط مشکل <span dir="ltr">Trust</span> را حل می‌کند. خطاهایی مثل اشتباه بودن <span dir="ltr">Hostname</span>، منقضی شدن <span dir="ltr">Certificate</span>، ناقص بودن <span dir="ltr">Certificate Chain</span> یا اشتباه بودن ساعت سیستم باید جداگانه بررسی شوند.</p>

<a id="reload-applications"></a>
<h3 dir="rtl" align="right"><span dir="ltr">Restart</span> کردن سرویس</h3>

<p dir="rtl" align="right">بعضی سرویس‌ها لیست <span dir="ltr">CA</span>ها را <span dir="ltr">Cache</span> می‌کنند. اگر هنوز خطای <span dir="ltr">Trust</span> می‌بینید، همان سرویس را <span dir="ltr">Restart</span> کنید:</p>

~~~bash
sudo systemctl restart example-service
sudo systemctl --no-pager status example-service
~~~

> [!NOTE]
> <p dir="rtl" align="right">بعضی برنامه‌ها <span dir="ltr">Trust Store</span> جداگانه دارند؛ مثل <span dir="ltr">Java</span>، بعضی مرورگرها، محیط‌های <span dir="ltr">Python</span>، کانتینرها و برخی <span dir="ltr">Runtime</span>ها. در این حالت باید <span dir="ltr">CA</span> را به همان برنامه هم معرفی کنید.</p>

<a id="remove-replace-ca"></a>
<h2 dir="rtl" align="right">۶. حذف یا جایگزینی <span dir="ltr">CA</span></h2>

<p dir="rtl" align="right">اگر دیگر به این <span dir="ltr">CA</span> نیاز ندارید، فایل آن را پاک کنید:</p>

~~~bash
sudo rm -f /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

<p dir="rtl" align="right">بعد لیست <span dir="ltr">CA</span>ها را دوباره بسازید:</p>

~~~bash
sudo update-ca-certificates --fresh
~~~

<p dir="rtl" align="right">در آخر چک کنید که لینک قبلی حذف شده باشد:</p>

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca' || true
~~~

> [!CAUTION]
> <p dir="rtl" align="right">قبل از حذف <span dir="ltr">CA</span>، ببینید چه سرویس‌هایی از آن استفاده می‌کنند. در غیر این صورت ممکن است اتصال <span dir="ltr">TLS</span> آن‌ها قطع شود.</p>

<a id="replace-expiring-ca"></a>
<h3 dir="rtl" align="right">جایگزینی <span dir="ltr">CA</span> قدیمی</h3>

<p dir="rtl" align="right">قبل از جایگزینی، از فایل فعلی یک <span dir="ltr">Backup</span> بگیرید:</p>

~~~bash
sudo cp \
  /usr/local/share/ca-certificates/example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt.bak
~~~

<p dir="rtl" align="right">حالا فایل جدید را نصب کنید و لیست <span dir="ltr">CA</span>ها را دوباره بسازید:</p>

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  replacement-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt

sudo update-ca-certificates
~~~

<p dir="rtl" align="right">بعد از نصب، <span dir="ltr">Fingerprint</span> و اتصال <span dir="ltr">HTTPS</span> را دوباره تست کنید. تا قبل از تمام شدن تست‌ها، فایل <span dir="ltr">Backup</span> را پاک نکنید.</p>

<a id="troubleshooting"></a>
<h2 dir="rtl" align="right">خطاهای رایج</h2>

<a id="certificate-ignored"></a>
<h3 dir="rtl" align="right"><span dir="ltr">CA</span> به سیستم اضافه نمی‌شود</h3>

<p dir="rtl" align="right">پسوند فایل، مالک و سطح دسترسی آن را چک کنید:</p>

~~~bash
find /usr/local/share/ca-certificates \
  -maxdepth 1 \
  -type f \
  -printf '%M %u:%g %f\n'
~~~

<p dir="rtl" align="right">فایل باید پسوند <code dir="ltr">.crt</code> داشته باشد و داخل آن فقط یک <span dir="ltr">PEM Certificate</span> باشد.</p>

<a id="certificate-parse-error"></a>
<h3 dir="rtl" align="right"><span dir="ltr">OpenSSL</span> فایل را نمی‌خواند</h3>

~~~bash
openssl x509 \
  -in /usr/local/share/ca-certificates/example-internal-ca.crt \
  -noout \
  -text
~~~

<p dir="rtl" align="right">اگر این دستور خطا داد، فرمت فایل درست نیست یا خود فایل خراب است. فایل صحیح را دوباره از منبع اصلی بگیرید.</p>

<a id="endpoint-untrusted"></a>
<h3 dir="rtl" align="right">سرویس هنوز <span dir="ltr">Certificate</span> را قبول نمی‌کند</h3>

<p dir="rtl" align="right">اول <span dir="ltr">Certificate Chain</span>ای را که سرور برمی‌گرداند ببینید:</p>

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -showcerts \
  </dev/null
~~~

<p dir="rtl" align="right">معمولاً مشکل یکی از این موارد است:</p>

<ul dir="rtl" align="right">
  <li><span dir="ltr">CA</span> اشتباهی را نصب کرده‌اید.</li>
  <li>سرور <span dir="ltr">Intermediate Certificate</span> را ارسال نمی‌کند.</li>
  <li><span dir="ltr">Hostname</span> با <span dir="ltr">Certificate</span> یکی نیست.</li>
  <li><span dir="ltr">Certificate</span> منقضی شده یا هنوز معتبر نشده است.</li>
  <li>برنامه از <span dir="ltr">Trust Store</span> جداگانه استفاده می‌کند.</li>
  <li>سرویس هنوز اطلاعات قبلی را در <span dir="ltr">Cache</span> نگه داشته است.</li>
</ul>

<a id="acceptance-checklist"></a>
<h2 dir="rtl" align="right">چک‌لیست نهایی</h2>

<ul dir="rtl" align="right">
  <li>☐ فایل فقط شامل <span dir="ltr">Public CA</span> است و <span dir="ltr">Private Key</span> ندارد.</li>
  <li>☐ <span dir="ltr">Fingerprint</span> فایل با مقدار اصلی یکی است.</li>
  <li>☐ فایل با فرمت <span dir="ltr">PEM</span> و پسوند <code dir="ltr">.crt</code> است.</li>
  <li>☐ فایل داخل <code dir="ltr">/usr/local/share/ca-certificates/</code> قرار دارد.</li>
  <li>☐ دستور <code dir="ltr">update-ca-certificates</code> بدون خطا اجرا می‌شود.</li>
  <li>☐ لینک <span dir="ltr">Certificate</span> داخل <code dir="ltr">/etc/ssl/certs/</code> ساخته شده است.</li>
  <li>☐ خروجی <span dir="ltr">OpenSSL</span> مقدار <code dir="ltr">Verify return code: 0 (ok)</code> را نشان می‌دهد.</li>
  <li>☐ سرویس بدون خاموش کردن <span dir="ltr">TLS Verification</span> وصل می‌شود.</li>
  <li>☐ قبل از حذف یا جایگزینی <span dir="ltr">CA</span>، یک راه برگشت دارید.</li>
</ul>

> [!IMPORTANT]
> <p dir="rtl" align="right">برای رفع خطای <span dir="ltr">Trust</span>، <span dir="ltr">TLS Verification</span> را در محیط <span dir="ltr">Production</span> خاموش نکنید. مشکل اصلی <span dir="ltr">Certificate</span> یا <span dir="ltr">CA Chain</span> را پیدا و برطرف کنید.</p>
