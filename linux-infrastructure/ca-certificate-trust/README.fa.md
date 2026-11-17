<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">🔐 نصب گواهی <span dir="ltr">CA</span> سفارشی در <span dir="ltr">Ubuntu</span> و <span dir="ltr">Debian</span></h1>

<p dir="rtl" align="right">این راهنما روش افزودن گواهی یک مرجع صدور گواهی داخلی یا خصوصی را به <span dir="ltr">Trust Store</span> سیستم‌عامل در <span dir="ltr">Ubuntu</span> و <span dir="ltr">Debian</span> توضیح می‌دهد.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">تمام نام گواهی‌ها، دامنه‌ها و <span dir="ltr">Endpoint</span>های موجود در این مستند عمومی نمونه هستند. قبل از اجرا، آن‌ها را با مقادیر تأییدشده محیط خود جایگزین کنید.</p>

> [!CAUTION]
> <p dir="rtl" align="right">هرگز <span dir="ltr">Private Key</span>، <span dir="ltr">Password</span>، <span dir="ltr">Token</span>، فایل <span dir="ltr">PKCS#12</span> یا مجموعه گواهی محرمانه را کپی یا منتشر نکنید. این فرایند فقط به گواهی عمومی <span dir="ltr">CA</span> نیاز دارد.</p>

<a id="table-of-contents"></a>
<h2 dir="rtl" align="right">فهرست مطالب</h2>

<ul dir="rtl" align="right">
  <li><a href="#trust-store-locations">مسیرهای Trust Store</a></li>
  <li><a href="#install-required-packages">۱. نصب بسته‌های موردنیاز</a></li>
  <li><a href="#inspect-ca-certificate">۲. بررسی گواهی CA</a></li>
  <li><a href="#install-ca-certificate">۳. نصب گواهی CA</a></li>
  <li><a href="#validate-installation">۴. اعتبارسنجی نصب</a></li>
  <li><a href="#test-tls-endpoint">۵. آزمایش یک Endpoint مبتنی بر TLS</a></li>
  <li><a href="#remove-replace-ca">۶. حذف یا جایگزینی گواهی CA</a></li>
  <li><a href="#troubleshooting">عیب‌یابی</a></li>
  <li><a href="#acceptance-checklist">چک‌لیست پذیرش</a></li>
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
      <td align="right">مسیر مبدأ گواهی‌های CA که به‌صورت محلی مدیریت می‌شوند</td>
    </tr>
    <tr>
      <td dir="ltr"><code>/etc/ssl/certs/</code></td>
      <td align="right">لینک‌های تولیدشده سازگار با OpenSSL</td>
    </tr>
    <tr>
      <td dir="ltr"><code>/etc/ssl/certs/ca-certificates.crt</code></td>
      <td align="right">مجموعه یکپارچه گواهی‌های CA سیستم‌عامل</td>
    </tr>
  </tbody>
</table>

> [!NOTE]
> <p dir="rtl" align="right">گواهی‌های محلی باید پسوند <code dir="ltr">.crt</code> داشته باشند. در هر فایل فقط یک گواهی <span dir="ltr">CA</span> قرار دهید.</p>

<a id="install-required-packages"></a>
<h2 dir="rtl" align="right">۱. نصب بسته‌های موردنیاز</h2>

<p dir="rtl" align="right">مطمئن شوید بسته مدیریت گواهی‌های <span dir="ltr">CA</span> سیستم‌عامل و <span dir="ltr">OpenSSL</span> نصب شده‌اند:</p>

~~~bash
sudo apt update
sudo apt install -y ca-certificates openssl
~~~

<a id="inspect-ca-certificate"></a>
<h2 dir="rtl" align="right">۲. بررسی گواهی <span dir="ltr">CA</span></h2>

<p dir="rtl" align="right">گواهی باید با فرمت <span dir="ltr">PEM</span> باشد و با نشانگرهای زیر شروع و پایان یابد:</p>

~~~text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
~~~

<p dir="rtl" align="right">بدون نمایش کامل گواهی، خط اول و آخر آن را بررسی کنید:</p>

~~~bash
head -n 1 example-internal-ca.crt
tail -n 1 example-internal-ca.crt
~~~

<p dir="rtl" align="right">اطلاعات گواهی را مشاهده کنید:</p>

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
> <p dir="rtl" align="right">قبل از نصب گواهی، <span dir="ltr">Fingerprint</span> مبتنی بر <span dir="ltr">SHA-256</span> را از طریق یک کانال قابل‌اعتماد تأیید کنید. صرفاً به‌دلیل صحیح به‌نظر رسیدن نام فایل، به یک گواهی <span dir="ltr">CA</span> اعتماد نکنید.</p>

<a id="convert-der-to-pem"></a>
<h3 dir="rtl" align="right">تبدیل گواهی <span dir="ltr">DER</span> به <span dir="ltr">PEM</span></h3>

<p dir="rtl" align="right">اگر گواهی با فرمت <span dir="ltr">DER</span> است، قبل از نصب آن را تبدیل کنید:</p>

~~~bash
openssl x509 \
  -inform DER \
  -in example-internal-ca.der \
  -out example-internal-ca.crt
~~~

<p dir="rtl" align="right">گواهی تبدیل‌شده را اعتبارسنجی کنید:</p>

~~~bash
openssl x509 -in example-internal-ca.crt -noout -subject -issuer
~~~

<a id="install-ca-certificate"></a>
<h2 dir="rtl" align="right">۳. نصب گواهی <span dir="ltr">CA</span></h2>

<p dir="rtl" align="right">گواهی عمومی <span dir="ltr">CA</span> را با مالکیت و سطح دسترسی مناسب در مسیر گواهی‌های محلی نصب کنید:</p>

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

<p dir="rtl" align="right"><span dir="ltr">Trust Store</span> سیستم‌عامل را به‌روزرسانی کنید:</p>

~~~bash
sudo update-ca-certificates
~~~

<p dir="rtl" align="right">خروجی موفق معمولاً مشابه نمونه زیر است:</p>

~~~text
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
~~~

> [!NOTE]
> <p dir="rtl" align="right">این دستور مجموعه گواهی‌های <span dir="ltr">CA</span> سیستم را بازسازی کرده و لینک‌های لازم را در مسیر <code dir="ltr">/etc/ssl/certs/</code> ایجاد می‌کند. فایل‌ها را به‌صورت دستی داخل مسیر تولیدشده <span dir="ltr">Trust Store</span> کپی نکنید.</p>

<a id="validate-installation"></a>
<h2 dir="rtl" align="right">۴. اعتبارسنجی نصب</h2>

<p dir="rtl" align="right">گواهی نصب‌شده در مسیر مبدأ را بررسی کنید:</p>

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

<p dir="rtl" align="right">وجود لینک تولیدشده برای گواهی را بررسی کنید:</p>

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca'
~~~

<p dir="rtl" align="right">اطمینان حاصل کنید فرایند به‌روزرسانی گواهی بدون خطا تکمیل می‌شود:</p>

~~~bash
sudo update-ca-certificates --verbose
~~~

> [!TIP]
> <p dir="rtl" align="right"><span dir="ltr">Fingerprint</span> مبتنی بر <span dir="ltr">SHA-256</span> فایل نصب‌شده را با <span dir="ltr">Fingerprint</span> ارائه‌شده توسط مالک گواهی یا مدیر <span dir="ltr">PKI</span> مقایسه کنید.</p>

<a id="test-tls-endpoint"></a>
<h2 dir="rtl" align="right">۵. آزمایش یک <span dir="ltr">Endpoint</span> مبتنی بر <span dir="ltr">TLS</span></h2>

<p dir="rtl" align="right">در مستندات عمومی از یک <span dir="ltr">Endpoint</span> نمونه و غیرحساس استفاده کنید:</p>

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

<p dir="rtl" align="right">اگر زنجیره <span dir="ltr">TLS</span> معتبر و مورد اعتماد باشد، باید خروجی مشابه زیر مشاهده شود:</p>

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

<p dir="rtl" align="right">همان <span dir="ltr">Endpoint</span> را با <span dir="ltr">cURL</span> آزمایش کنید:</p>

~~~bash
curl -Iv https://service.example.com/
~~~

> [!WARNING]
> <p dir="rtl" align="right">نصب گواهی <span dir="ltr">CA</span> مشکلاتی مانند عدم تطابق <span dir="ltr">Hostname</span>، منقضی شدن گواهی، ناقص بودن زنجیره گواهی سرور یا اشتباه بودن ساعت سیستم را برطرف نمی‌کند.</p>

<a id="reload-applications"></a>
<h3 dir="rtl" align="right">بارگذاری مجدد برنامه‌ها</h3>

<p dir="rtl" align="right">برنامه‌هایی که برای مدت طولانی اجرا می‌شوند ممکن است <span dir="ltr">Trust Store</span> را <span dir="ltr">Cache</span> کنند. پس از اعتبارسنجی نصب <span dir="ltr">CA</span>، فقط سرویس تحت‌تأثیر را <span dir="ltr">Restart</span> کنید:</p>

~~~bash
sudo systemctl restart example-service
sudo systemctl --no-pager status example-service
~~~

> [!NOTE]
> <p dir="rtl" align="right">بعضی برنامه‌ها <span dir="ltr">Trust Store</span> مستقل دارند. برنامه‌های <span dir="ltr">Java</span>، مرورگرها، محیط‌های <span dir="ltr">Python</span> مبتنی بر <span dir="ltr">Certifi</span>، کانتینرها و برخی <span dir="ltr">Runtime</span>ها ممکن است به تنظیم جداگانه گواهی نیاز داشته باشند.</p>

<a id="remove-replace-ca"></a>
<h2 dir="rtl" align="right">۶. حذف یا جایگزینی گواهی <span dir="ltr">CA</span></h2>

<p dir="rtl" align="right">گواهی <span dir="ltr">CA</span> نصب‌شده به‌صورت محلی را حذف کنید:</p>

~~~bash
sudo rm -f /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

<p dir="rtl" align="right"><span dir="ltr">Trust Store</span> را بازسازی کرده و لینک‌های تولیدشده قدیمی را حذف کنید:</p>

~~~bash
sudo update-ca-certificates --fresh
~~~

<p dir="rtl" align="right">اطمینان حاصل کنید گواهی دیگر وجود ندارد:</p>

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca' || true
~~~

> [!CAUTION]
> <p dir="rtl" align="right">حذف یک <span dir="ltr">CA</span> مورد اعتماد می‌تواند بلافاصله ارتباط <span dir="ltr">TLS</span> سرویس‌های وابسته را مختل کند. قبل از حذف یا چرخش گواهی، برنامه‌های تحت‌تأثیر را شناسایی کرده و برای <span dir="ltr">Rollback</span> آماده باشید.</p>

<a id="replace-expiring-ca"></a>
<h3 dir="rtl" align="right">جایگزینی <span dir="ltr">CA</span> در آستانه انقضا</h3>

<p dir="rtl" align="right">از گواهی عمومی موجود نسخه پشتیبان تهیه کنید:</p>

~~~bash
sudo cp \
  /usr/local/share/ca-certificates/example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt.bak
~~~

<p dir="rtl" align="right">گواهی جایگزین تأییدشده را نصب کرده و <span dir="ltr">Trust Store</span> را به‌روزرسانی کنید:</p>

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  replacement-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt

sudo update-ca-certificates
~~~

<p dir="rtl" align="right">قبل از حذف نسخه پشتیبان، بررسی <span dir="ltr">Fingerprint</span> و آزمایش <span dir="ltr">Endpoint</span> را دوباره انجام دهید.</p>

<a id="troubleshooting"></a>
<h2 dir="rtl" align="right">عیب‌یابی</h2>

<a id="certificate-ignored"></a>
<h3 dir="rtl" align="right">گواهی نادیده گرفته می‌شود</h3>

<p dir="rtl" align="right">پسوند فایل و سطح دسترسی آن را بررسی کنید:</p>

~~~bash
find /usr/local/share/ca-certificates \
  -maxdepth 1 \
  -type f \
  -printf '%M %u:%g %f\n'
~~~

<p dir="rtl" align="right">فایل باید پسوند <code dir="ltr">.crt</code> داشته و فقط شامل یک گواهی <span dir="ltr">PEM</span> باشد.</p>

<a id="certificate-parse-error"></a>
<h3 dir="rtl" align="right">گواهی قابل خواندن نیست</h3>

~~~bash
openssl x509 \
  -in /usr/local/share/ca-certificates/example-internal-ca.crt \
  -noout \
  -text
~~~

<p dir="rtl" align="right">اگر این دستور با خطا مواجه شد، فرمت ورودی را بررسی کرده و یک گواهی عمومی <span dir="ltr">CA</span> معتبر را از منبع مجاز دریافت کنید.</p>

<a id="endpoint-untrusted"></a>
<h3 dir="rtl" align="right"><span dir="ltr">Endpoint</span> همچنان مورد اعتماد نیست</h3>

<p dir="rtl" align="right">زنجیره گواهی ارائه‌شده توسط سرور را بررسی کنید:</p>

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -showcerts \
  </dev/null
~~~

<p dir="rtl" align="right">دلایل رایج عبارت‌اند از:</p>

<ul dir="rtl" align="right">
  <li>گواهی <span dir="ltr">CA</span> اشتباهی نصب شده است.</li>
  <li>سرور گواهی <span dir="ltr">Intermediate</span> موردنیاز را ارائه نمی‌کند.</li>
  <li><span dir="ltr">Hostname</span> مربوط به <span dir="ltr">Endpoint</span> با گواهی مطابقت ندارد.</li>
  <li>گواهی منقضی شده یا هنوز معتبر نشده است.</li>
  <li>برنامه از <span dir="ltr">Trust Store</span> مستقلی استفاده می‌کند.</li>
  <li>سرویس اطلاعات <span dir="ltr">Cache</span>شده <span dir="ltr">Trust Store</span> را مجدداً بارگذاری نکرده است.</li>
</ul>

<a id="acceptance-checklist"></a>
<h2 dir="rtl" align="right">چک‌لیست پذیرش</h2>

<ul dir="rtl" align="right">
  <li>☐ گواهی فقط شامل اطلاعات عمومی <span dir="ltr">CA</span> است و هیچ <span dir="ltr">Private Key</span> ندارد.</li>
  <li>☐ <span dir="ltr">Fingerprint</span> مبتنی بر <span dir="ltr">SHA-256</span> از طریق یک کانال قابل‌اعتماد تأیید شده است.</li>
  <li>☐ گواهی با فرمت <span dir="ltr">PEM</span> است و پسوند <code dir="ltr">.crt</code> دارد.</li>
  <li>☐ فایل در مسیر <code dir="ltr">/usr/local/share/ca-certificates/</code> نصب شده است.</li>
  <li>☐ دستور <code dir="ltr">update-ca-certificates</code> بدون خطا تکمیل می‌شود.</li>
  <li>☐ لینک تولیدشده <span dir="ltr">Trust Store</span> در مسیر <code dir="ltr">/etc/ssl/certs/</code> وجود دارد.</li>
  <li>☐ <span dir="ltr">OpenSSL</span> برای <span dir="ltr">Endpoint</span> مقصد مقدار <code dir="ltr">Verify return code: 0 (ok)</code> را گزارش می‌کند.</li>
  <li>☐ برنامه تحت‌تأثیر بدون غیرفعال کردن <span dir="ltr">TLS Verification</span> متصل می‌شود.</li>
  <li>☐ روش‌های <span dir="ltr">Rollback</span> و چرخش گواهی مشخص و قابل اجرا هستند.</li>
</ul>

> [!IMPORTANT]
> <p dir="rtl" align="right">در محیط <span dir="ltr">Production</span> هرگز برای رفع خطای <span dir="ltr">Trust</span>، بررسی <span dir="ltr">TLS</span> را غیرفعال نکنید. زنجیره صحیح <span dir="ltr">CA</span> را نصب کرده و مشکل اصلی گواهی یا تنظیمات سرور را برطرف کنید.</p>
