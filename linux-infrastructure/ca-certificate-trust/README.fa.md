<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">🔐 نصب <span dir="ltr">CA Certificate</span> سفارشی در <span dir="ltr">Ubuntu و Debian</span></h1>

<p dir="rtl" align="right">در این راهنما یک <span dir="ltr">CA Certificate</span> داخلی یا <span dir="ltr">Private</span> را به <span dir="ltr">Trust Store</span> سیستم‌عامل‌های <span dir="ltr">Ubuntu و Debian</span> اضافه می‌کنیم.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">اسم <span dir="ltr">Certificate</span>ها، <span dir="ltr">Domain</span>ها و <span dir="ltr">Endpoint</span>های این <span dir="ltr">README</span> نمونه هستند. قبل از اجرا، آن‌ها را با مقادیر واقعی و تأییدشده محیط خودتان جایگزین کنید.</p>

> [!CAUTION]
> <p dir="rtl" align="right">هیچ‌وقت <span dir="ltr">Private Key</span>، <span dir="ltr">Password</span>، <span dir="ltr">Token</span>، فایل <span dir="ltr">PKCS#12</span> یا <span dir="ltr">Certificate Bundle</span> محرمانه را داخل ریپو قرار ندهید. برای این کار فقط <span dir="ltr">Public CA Certificate</span> لازم است.</p>

<a id="table-of-contents"></a>
<h2 dir="rtl" align="right">فهرست مطالب</h2>

<ul dir="rtl" align="right">
  <li><a href="#trust-store-locations">مسیرهای Trust Store</a></li>
  <li><a href="#install-required-packages">۱. نصب پکیج‌های موردنیاز</a></li>
  <li><a href="#inspect-ca-certificate">۲. بررسی CA Certificate</a></li>
  <li><a href="#install-ca-certificate">۳. نصب CA Certificate</a></li>
  <li><a href="#validate-installation">۴. بررسی نهایی نصب</a></li>
  <li><a href="#test-tls-endpoint">۵. تست TLS Endpoint</a></li>
  <li><a href="#remove-replace-ca">۶. حذف یا جایگزینی CA Certificate</a></li>
  <li><a href="#troubleshooting">رفع خطاهای رایج</a></li>
  <li><a href="#acceptance-checklist">چک‌لیست نهایی</a></li>
</ul>

<a id="trust-store-locations"></a>
<h2 dir="rtl" align="right">مسیرهای <span dir="ltr">Trust Store</span></h2>

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
      <td align="right">محل قرار دادن CA Certificateهایی که خودمان نصب می‌کنیم</td>
    </tr>
    <tr>
      <td dir="ltr"><code>/etc/ssl/certs/</code></td>
      <td align="right">محل Symlinkهایی که سیستم به‌صورت خودکار می‌سازد</td>
    </tr>
    <tr>
      <td dir="ltr"><code>/etc/ssl/certs/ca-certificates.crt</code></td>
      <td align="right">فایل اصلی CA Bundle سیستم‌عامل</td>
    </tr>
  </tbody>
</table>

> [!NOTE]
> <p dir="rtl" align="right">فایل <span dir="ltr">Certificate</span> باید پسوند <code dir="ltr">.crt</code> داشته باشد. داخل هر فایل هم فقط یک <span dir="ltr">CA Certificate</span> قرار دهید.</p>

<a id="install-required-packages"></a>
<h2 dir="rtl" align="right">۱. نصب پکیج‌های موردنیاز</h2>

<p dir="rtl" align="right">پکیج‌های <span dir="ltr">ca-certificates</span> و <span dir="ltr">openssl</span> را نصب کنید:</p>

~~~bash
sudo apt update
sudo apt install -y ca-certificates openssl
~~~

<a id="inspect-ca-certificate"></a>
<h2 dir="rtl" align="right">۲. بررسی <span dir="ltr">CA Certificate</span></h2>

<p dir="rtl" align="right">فایل <span dir="ltr">Certificate</span> باید با فرمت <span dir="ltr">PEM</span> باشد و با این دو خط شروع و تمام شود:</p>

~~~text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
~~~

<p dir="rtl" align="right">برای اینکه کل فایل روی صفحه چاپ نشود، فقط خط اول و آخر را ببینید:</p>

~~~bash
head -n 1 example-internal-ca.crt
tail -n 1 example-internal-ca.crt
~~~

<p dir="rtl" align="right">جزئیات <span dir="ltr">Certificate</span> را هم با دستور زیر بررسی کنید:</p>

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
> <p dir="rtl" align="right">قبل از نصب، <span dir="ltr">SHA-256 Fingerprint</span> را از تیم <span dir="ltr">PKI</span> یا یک کانال مطمئن بگیرید و با خروجی بالا مقایسه کنید. درست بودن اسم فایل به‌تنهایی کافی نیست.</p>

<a id="convert-der-to-pem"></a>
<h3 dir="rtl" align="right">تبدیل <span dir="ltr">DER</span> به <span dir="ltr">PEM</span></h3>

<p dir="rtl" align="right">اگر فایل شما با فرمت <span dir="ltr">DER</span> است، قبل از نصب آن را به <span dir="ltr">PEM</span> تبدیل کنید:</p>

~~~bash
openssl x509 \
  -inform DER \
  -in example-internal-ca.der \
  -out example-internal-ca.crt
~~~

<p dir="rtl" align="right">بعد از تبدیل، مطمئن شوید فایل بدون خطا خوانده می‌شود:</p>

~~~bash
openssl x509 -in example-internal-ca.crt -noout -subject -issuer
~~~

<a id="install-ca-certificate"></a>
<h2 dir="rtl" align="right">۳. نصب <span dir="ltr">CA Certificate</span></h2>

<p dir="rtl" align="right">فایل <span dir="ltr">Public CA Certificate</span> را با مالک <span dir="ltr">root</span> و <span dir="ltr">Permission 0644</span> در مسیر محلی <span dir="ltr">Trust Store</span> قرار دهید:</p>

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

<p dir="rtl" align="right">حالا <span dir="ltr">Trust Store</span> سیستم را به‌روزرسانی کنید:</p>

~~~bash
sudo update-ca-certificates
~~~

<p dir="rtl" align="right">اگر همه‌چیز درست باشد، خروجی تقریباً شبیه این است:</p>

~~~text
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
~~~

> [!NOTE]
> <p dir="rtl" align="right">دستور <code dir="ltr">update-ca-certificates</code> فایل <span dir="ltr">CA Bundle</span> و <span dir="ltr">Symlink</span>های موردنیاز را به‌صورت خودکار می‌سازد. فایلی را دستی داخل <code dir="ltr">/etc/ssl/certs/</code> کپی نکنید.</p>

<a id="validate-installation"></a>
<h2 dir="rtl" align="right">۴. بررسی نهایی نصب</h2>

<p dir="rtl" align="right">اول اطلاعات فایل نصب‌شده را ببینید:</p>

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

<p dir="rtl" align="right">بعد بررسی کنید که <span dir="ltr">Symlink</span> مربوط به آن ساخته شده باشد:</p>

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca'
~~~

<p dir="rtl" align="right">برای دیدن جزئیات فرایند <span dir="ltr">Update</span> هم از حالت <span dir="ltr">Verbose</span> استفاده کنید:</p>

~~~bash
sudo update-ca-certificates --verbose
~~~

> [!TIP]
> <p dir="rtl" align="right"><span dir="ltr">SHA-256 Fingerprint</span> فایل نصب‌شده باید دقیقاً با مقداری که تیم <span dir="ltr">PKI</span> به شما داده یکسان باشد.</p>

<a id="test-tls-endpoint"></a>
<h2 dir="rtl" align="right">۵. تست یک <span dir="ltr">TLS Endpoint</span></h2>

<p dir="rtl" align="right">در مستندات عمومی همیشه از یک <span dir="ltr">Endpoint</span> نمونه استفاده کنید:</p>

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

<p dir="rtl" align="right">اگر <span dir="ltr">TLS Chain</span> درست و <span dir="ltr">Trusted</span> باشد، باید این خروجی را ببینید:</p>

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

<p dir="rtl" align="right">همان <span dir="ltr">Endpoint</span> را با <span dir="ltr">cURL</span> هم تست کنید:</p>

~~~bash
curl -Iv https://service.example.com/
~~~

> [!WARNING]
> <p dir="rtl" align="right">اضافه کردن <span dir="ltr">CA</span> فقط مشکل <span dir="ltr">Trust</span> را حل می‌کند. خطاهایی مثل <span dir="ltr">Hostname Mismatch</span>، <span dir="ltr">Expired Certificate</span>، ناقص بودن <span dir="ltr">Server Chain</span> یا اشتباه بودن ساعت سیستم باید جداگانه برطرف شوند.</p>

<a id="reload-applications"></a>
<h3 dir="rtl" align="right"><span dir="ltr">Restart</span> کردن سرویس مرتبط</h3>

<p dir="rtl" align="right">بعضی سرویس‌ها <span dir="ltr">Trust Store</span> را <span dir="ltr">Cache</span> می‌کنند. اگر بعد از نصب هنوز خطای <span dir="ltr">Trust</span> دارید، فقط همان سرویس را <span dir="ltr">Restart</span> کنید:</p>

~~~bash
sudo systemctl restart example-service
sudo systemctl --no-pager status example-service
~~~

> [!NOTE]
> <p dir="rtl" align="right">بعضی برنامه‌ها <span dir="ltr">Trust Store</span> جداگانه دارند؛ برای مثال <span dir="ltr">Java</span>، بعضی مرورگرها، محیط‌های <span dir="ltr">Python</span> مبتنی بر <span dir="ltr">Certifi</span>، کانتینرها و برخی <span dir="ltr">Runtime</span>ها. در این موارد باید <span dir="ltr">CA</span> را داخل همان <span dir="ltr">Trust Store</span> هم اضافه کنید.</p>

<a id="remove-replace-ca"></a>
<h2 dir="rtl" align="right">۶. حذف یا جایگزینی <span dir="ltr">CA Certificate</span></h2>

<p dir="rtl" align="right">برای حذف <span dir="ltr">CA</span>، فایل آن را از مسیر محلی پاک کنید:</p>

~~~bash
sudo rm -f /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

<p dir="rtl" align="right">بعد <span dir="ltr">Trust Store</span> را از نو بسازید تا <span dir="ltr">Symlink</span>های قدیمی هم حذف شوند:</p>

~~~bash
sudo update-ca-certificates --fresh
~~~

<p dir="rtl" align="right">در آخر بررسی کنید که اثری از فایل قبلی نمانده باشد:</p>

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca' || true
~~~

> [!CAUTION]
> <p dir="rtl" align="right">حذف یک <span dir="ltr">CA</span> می‌تواند ارتباط <span dir="ltr">TLS</span> سرویس‌های وابسته را قطع کند. قبل از حذف یا <span dir="ltr">Rotate</span> کردن <span dir="ltr">Certificate</span>، سرویس‌های وابسته را پیدا کنید و یک راه <span dir="ltr">Rollback</span> داشته باشید.</p>

<a id="replace-expiring-ca"></a>
<h3 dir="rtl" align="right">جایگزینی <span dir="ltr">CA</span> در آستانه انقضا</h3>

<p dir="rtl" align="right">قبل از هر تغییری، از <span dir="ltr">Public Certificate</span> فعلی یک <span dir="ltr">Backup</span> بگیرید:</p>

~~~bash
sudo cp \
  /usr/local/share/ca-certificates/example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt.bak
~~~

<p dir="rtl" align="right">نسخه جدید و تأییدشده را جایگزین کنید و <span dir="ltr">Trust Store</span> را دوباره بسازید:</p>

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  replacement-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt

sudo update-ca-certificates
~~~

<p dir="rtl" align="right">تا وقتی <span dir="ltr">Fingerprint</span> و اتصال <span dir="ltr">TLS</span> را دوباره تست نکرده‌اید، فایل <span dir="ltr">Backup</span> را حذف نکنید.</p>

<a id="troubleshooting"></a>
<h2 dir="rtl" align="right">رفع خطاهای رایج</h2>

<a id="certificate-ignored"></a>
<h3 dir="rtl" align="right"><span dir="ltr">Certificate</span> توسط سیستم شناسایی نمی‌شود</h3>

<p dir="rtl" align="right">پسوند فایل، مالک و <span dir="ltr">Permission</span> را بررسی کنید:</p>

~~~bash
find /usr/local/share/ca-certificates \
  -maxdepth 1 \
  -type f \
  -printf '%M %u:%g %f\n'
~~~

<p dir="rtl" align="right">فایل باید پسوند <code dir="ltr">.crt</code> داشته باشد و فقط شامل یک <span dir="ltr">PEM Certificate</span> باشد.</p>

<a id="certificate-parse-error"></a>
<h3 dir="rtl" align="right"><span dir="ltr">OpenSSL</span> نمی‌تواند <span dir="ltr">Certificate</span> را بخواند</h3>

~~~bash
openssl x509 \
  -in /usr/local/share/ca-certificates/example-internal-ca.crt \
  -noout \
  -text
~~~

<p dir="rtl" align="right">اگر دستور بالا خطا داد، فرمت فایل درست نیست یا فایل خراب است. نسخه صحیح <span dir="ltr">Public CA Certificate</span> را از منبع معتبر بگیرید.</p>

<a id="endpoint-untrusted"></a>
<h3 dir="rtl" align="right"><span dir="ltr">Endpoint</span> هنوز <span dir="ltr">Trusted</span> نیست</h3>

<p dir="rtl" align="right"><span dir="ltr">Certificate Chain</span>ای را که سرور برمی‌گرداند بررسی کنید:</p>

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -showcerts \
  </dev/null
~~~

<p dir="rtl" align="right">دلیل‌های رایج:</p>

<ul dir="rtl" align="right">
  <li><span dir="ltr">CA Certificate</span> اشتباهی نصب شده است.</li>
  <li>سرور <span dir="ltr">Intermediate Certificate</span> موردنیاز را ارسال نمی‌کند.</li>
  <li><span dir="ltr">Hostname</span> با <span dir="ltr">Certificate</span> مطابقت ندارد.</li>
  <li><span dir="ltr">Certificate</span> منقضی شده یا هنوز زمان اعتبارش شروع نشده است.</li>
  <li>برنامه از <span dir="ltr">Trust Store</span> جداگانه‌ای استفاده می‌کند.</li>
  <li>سرویس هنوز اطلاعات قبلی <span dir="ltr">Trust Store</span> را در <span dir="ltr">Cache</span> نگه داشته است.</li>
</ul>

<a id="acceptance-checklist"></a>
<h2 dir="rtl" align="right">چک‌لیست نهایی</h2>

<ul dir="rtl" align="right">
  <li>☐ فایل فقط شامل <span dir="ltr">Public CA Certificate</span> است و هیچ <span dir="ltr">Private Key</span> داخل آن نیست.</li>
  <li>☐ <span dir="ltr">SHA-256 Fingerprint</span> از یک کانال مطمئن گرفته و بررسی شده است.</li>
  <li>☐ فایل با فرمت <span dir="ltr">PEM</span> و پسوند <code dir="ltr">.crt</code> است.</li>
  <li>☐ فایل در مسیر <code dir="ltr">/usr/local/share/ca-certificates/</code> قرار دارد.</li>
  <li>☐ دستور <code dir="ltr">update-ca-certificates</code> بدون خطا اجرا می‌شود.</li>
  <li>☐ <span dir="ltr">Symlink</span> مربوط به <span dir="ltr">Certificate</span> داخل <code dir="ltr">/etc/ssl/certs/</code> ساخته شده است.</li>
  <li>☐ <span dir="ltr">OpenSSL</span> برای <span dir="ltr">Endpoint</span> مقدار <code dir="ltr">Verify return code: 0 (ok)</code> را نشان می‌دهد.</li>
  <li>☐ سرویس بدون خاموش کردن <span dir="ltr">TLS Verification</span> به مقصد وصل می‌شود.</li>
  <li>☐ برای حذف یا <span dir="ltr">Rotate</span> کردن <span dir="ltr">Certificate</span>، روش <span dir="ltr">Rollback</span> مشخص است.</li>
</ul>

> [!IMPORTANT]
> <p dir="rtl" align="right">برای دور زدن خطای <span dir="ltr">Trust</span> هیچ‌وقت <span dir="ltr">TLS Verification</span> را در محیط <span dir="ltr">Production</span> خاموش نکنید. <span dir="ltr">CA Chain</span> را درست نصب کنید و علت اصلی خطا را برطرف کنید.</p>
