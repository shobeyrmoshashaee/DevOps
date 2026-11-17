<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<div dir="rtl">

# 🔐 نصب گواهی CA سفارشی در Ubuntu و Debian

این راهنما روش افزودن گواهی یک مرجع صدور گواهی داخلی یا خصوصی را به Trust Store سیستم‌عامل در Ubuntu و Debian توضیح می‌دهد.

> [!IMPORTANT]
> تمام نام گواهی‌ها، دامنه‌ها و Endpointهای موجود در این مستند عمومی نمونه هستند. قبل از اجرا، آن‌ها را با مقادیر تأییدشده محیط خود جایگزین کنید.

> [!CAUTION]
> هرگز Private Key، Password، Token، فایل PKCS#12 یا مجموعه گواهی محرمانه را کپی یا منتشر نکنید. این فرایند فقط به گواهی عمومی CA نیاز دارد.

## فهرست مطالب

- [مسیرهای Trust Store](#مسیرهای-trust-store)
- [۱. نصب بسته‌های موردنیاز](#۱-نصب-بستههای-موردنیاز)
- [۲. بررسی گواهی CA](#۲-بررسی-گواهی-ca)
- [۳. نصب گواهی CA](#۳-نصب-گواهی-ca)
- [۴. اعتبارسنجی نصب](#۴-اعتبارسنجی-نصب)
- [۵. آزمایش یک Endpoint مبتنی بر TLS](#۵-آزمایش-یک-endpoint-مبتنی-بر-tls)
- [۶. حذف یا جایگزینی گواهی CA](#۶-حذف-یا-جایگزینی-گواهی-ca)
- [عیب‌یابی](#عیبیابی)
- [چک‌لیست پذیرش](#چکلیست-پذیرش)

## مسیرهای Trust Store

| مسیر | کاربرد |
|---|---|
| <code>/usr/local/share/ca-certificates/</code> | مسیر مبدأ گواهی‌های CA که به‌صورت محلی مدیریت می‌شوند |
| <code>/etc/ssl/certs/</code> | لینک‌های تولیدشده سازگار با OpenSSL |
| <code>/etc/ssl/certs/ca-certificates.crt</code> | مجموعه یکپارچه گواهی‌های CA سیستم‌عامل |

> [!NOTE]
> گواهی‌های محلی باید پسوند <code>.crt</code> داشته باشند. در هر فایل فقط یک گواهی CA قرار دهید.

## ۱. نصب بسته‌های موردنیاز

مطمئن شوید بسته مدیریت گواهی‌های CA سیستم‌عامل و OpenSSL نصب شده‌اند:

~~~bash
sudo apt update
sudo apt install -y ca-certificates openssl
~~~

## ۲. بررسی گواهی CA

گواهی باید با فرمت PEM باشد و با نشانگرهای زیر شروع و پایان یابد:

~~~text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
~~~

بدون نمایش کامل گواهی، خط اول و آخر آن را بررسی کنید:

~~~bash
head -n 1 example-internal-ca.crt
tail -n 1 example-internal-ca.crt
~~~

اطلاعات گواهی را مشاهده کنید:

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
> قبل از نصب گواهی، Fingerprint مبتنی بر SHA-256 را از طریق یک کانال قابل‌اعتماد تأیید کنید. صرفاً به‌دلیل صحیح به‌نظر رسیدن نام فایل، به یک گواهی CA اعتماد نکنید.

### تبدیل گواهی DER به PEM

اگر گواهی با فرمت DER است، قبل از نصب آن را تبدیل کنید:

~~~bash
openssl x509 \
  -inform DER \
  -in example-internal-ca.der \
  -out example-internal-ca.crt
~~~

گواهی تبدیل‌شده را اعتبارسنجی کنید:

~~~bash
openssl x509 -in example-internal-ca.crt -noout -subject -issuer
~~~

## ۳. نصب گواهی CA

گواهی عمومی CA را با مالکیت و سطح دسترسی مناسب در مسیر گواهی‌های محلی نصب کنید:

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

Trust Store سیستم‌عامل را به‌روزرسانی کنید:

~~~bash
sudo update-ca-certificates
~~~

خروجی موفق معمولاً مشابه نمونه زیر است:

~~~text
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
~~~

> [!NOTE]
> این دستور مجموعه گواهی‌های CA سیستم را بازسازی کرده و لینک‌های لازم را در مسیر <code>/etc/ssl/certs/</code> ایجاد می‌کند. فایل‌ها را به‌صورت دستی داخل مسیر تولیدشده Trust Store کپی نکنید.

## ۴. اعتبارسنجی نصب

گواهی نصب‌شده در مسیر مبدأ را بررسی کنید:

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

وجود لینک تولیدشده برای گواهی را بررسی کنید:

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca'
~~~

اطمینان حاصل کنید فرایند به‌روزرسانی گواهی بدون خطا تکمیل می‌شود:

~~~bash
sudo update-ca-certificates --verbose
~~~

> [!TIP]
> Fingerprint مبتنی بر SHA-256 فایل نصب‌شده را با Fingerprint ارائه‌شده توسط مالک گواهی یا مدیر PKI مقایسه کنید.

## ۵. آزمایش یک Endpoint مبتنی بر TLS

در مستندات عمومی از یک Endpoint نمونه و غیرحساس استفاده کنید:

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

اگر زنجیره TLS معتبر و مورد اعتماد باشد، باید خروجی مشابه زیر مشاهده شود:

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

همان Endpoint را با cURL آزمایش کنید:

~~~bash
curl -Iv https://service.example.com/
~~~

> [!WARNING]
> نصب گواهی CA مشکلاتی مانند عدم تطابق Hostname، منقضی شدن گواهی، ناقص بودن زنجیره گواهی سرور یا اشتباه بودن ساعت سیستم را برطرف نمی‌کند.

### بارگذاری مجدد برنامه‌ها

برنامه‌هایی که برای مدت طولانی اجرا می‌شوند ممکن است Trust Store را Cache کنند. پس از اعتبارسنجی نصب CA، فقط سرویس تحت‌تأثیر را Restart کنید:

~~~bash
sudo systemctl restart example-service
sudo systemctl --no-pager status example-service
~~~

> [!NOTE]
> بعضی برنامه‌ها Trust Store مستقل دارند. برنامه‌های Java، مرورگرها، محیط‌های Python مبتنی بر Certifi، کانتینرها و برخی Runtimeها ممکن است به تنظیم جداگانه گواهی نیاز داشته باشند.

## ۶. حذف یا جایگزینی گواهی CA

گواهی CA نصب‌شده به‌صورت محلی را حذف کنید:

~~~bash
sudo rm -f /usr/local/share/ca-certificates/example-internal-ca.crt
~~~

Trust Store را بازسازی کرده و لینک‌های تولیدشده قدیمی را حذف کنید:

~~~bash
sudo update-ca-certificates --fresh
~~~

اطمینان حاصل کنید گواهی دیگر وجود ندارد:

~~~bash
ls -l /etc/ssl/certs/ | grep 'example-internal-ca' || true
~~~

> [!CAUTION]
> حذف یک CA مورد اعتماد می‌تواند بلافاصله ارتباط TLS سرویس‌های وابسته را مختل کند. قبل از حذف یا چرخش گواهی، برنامه‌های تحت‌تأثیر را شناسایی کرده و برای Rollback آماده باشید.

### جایگزینی CA در آستانه انقضا

از گواهی عمومی موجود نسخه پشتیبان تهیه کنید:

~~~bash
sudo cp \
  /usr/local/share/ca-certificates/example-internal-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt.bak
~~~

گواهی جایگزین تأییدشده را نصب کرده و Trust Store را به‌روزرسانی کنید:

~~~bash
sudo install \
  -o root \
  -g root \
  -m 0644 \
  replacement-ca.crt \
  /usr/local/share/ca-certificates/example-internal-ca.crt

sudo update-ca-certificates
~~~

قبل از حذف نسخه پشتیبان، بررسی Fingerprint و آزمایش Endpoint را دوباره انجام دهید.

## عیب‌یابی

### گواهی نادیده گرفته می‌شود

پسوند فایل و سطح دسترسی آن را بررسی کنید:

~~~bash
find /usr/local/share/ca-certificates \
  -maxdepth 1 \
  -type f \
  -printf '%M %u:%g %f\n'
~~~

فایل باید پسوند <code>.crt</code> داشته و فقط شامل یک گواهی PEM باشد.

### گواهی قابل خواندن نیست

~~~bash
openssl x509 \
  -in /usr/local/share/ca-certificates/example-internal-ca.crt \
  -noout \
  -text
~~~

اگر این دستور با خطا مواجه شد، فرمت ورودی را بررسی کرده و یک گواهی عمومی CA معتبر را از منبع مجاز دریافت کنید.

### Endpoint همچنان مورد اعتماد نیست

زنجیره گواهی ارائه‌شده توسط سرور را بررسی کنید:

~~~bash
openssl s_client \
  -connect service.example.com:443 \
  -servername service.example.com \
  -showcerts \
  </dev/null
~~~

دلایل رایج عبارت‌اند از:

- گواهی CA اشتباهی نصب شده است.
- سرور گواهی Intermediate موردنیاز را ارائه نمی‌کند.
- Hostname مربوط به Endpoint با گواهی مطابقت ندارد.
- گواهی منقضی شده یا هنوز معتبر نشده است.
- برنامه از Trust Store مستقلی استفاده می‌کند.
- سرویس اطلاعات Cacheشده Trust Store را مجدداً بارگذاری نکرده است.

## چک‌لیست پذیرش

- [ ] گواهی فقط شامل اطلاعات عمومی CA است و هیچ Private Key ندارد.
- [ ] Fingerprint مبتنی بر SHA-256 از طریق یک کانال قابل‌اعتماد تأیید شده است.
- [ ] گواهی با فرمت PEM است و پسوند <code>.crt</code> دارد.
- [ ] فایل در مسیر <code>/usr/local/share/ca-certificates/</code> نصب شده است.
- [ ] دستور <code>update-ca-certificates</code> بدون خطا تکمیل می‌شود.
- [ ] لینک تولیدشده Trust Store در مسیر <code>/etc/ssl/certs/</code> وجود دارد.
- [ ] OpenSSL برای Endpoint مقصد مقدار <code>Verify return code: 0 (ok)</code> را گزارش می‌کند.
- [ ] برنامه تحت‌تأثیر بدون غیرفعال کردن TLS Verification متصل می‌شود.
- [ ] روش‌های Rollback و چرخش گواهی مشخص و قابل اجرا هستند.

> [!IMPORTANT]
> در محیط Production هرگز برای رفع خطای Trust، بررسی TLS را غیرفعال نکنید. زنجیره صحیح CA را نصب کرده و مشکل اصلی گواهی یا تنظیمات سرور را برطرف کنید.

</div>
