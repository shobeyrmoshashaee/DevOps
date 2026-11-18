<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">🔒 گرفتن <span dir="ltr">SSL Certificate</span> با <span dir="ltr">Certbot</span> برای <span dir="ltr">NGINX</span></h1>

<p dir="rtl" align="right">در این راهنما سه روش را انجام می‌دهیم: گرفتن <span dir="ltr">Wildcard Certificate</span> با <span dir="ltr">DNS Verification</span> دستی، گرفتن Certificate با پلاگین <span dir="ltr">NGINX</span> و در آخر انجام خودکار <span dir="ltr">DNS-01</span> با <span dir="ltr">Cloudflare</span>.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">همه <span dir="ltr">Domain</span>ها، مسیرها و مقادیر این راهنما نمونه هستند. موقع اجرا از مقادیر محیط خودتان استفاده کنید.</p>

> [!CAUTION]
> <p dir="rtl" align="right"><span dir="ltr">API Token</span>، <span dir="ltr">Private Key</span>، فایل <span dir="ltr">Credential</span> یا محتوای <code dir="ltr">/etc/letsencrypt/</code> را داخل ریپو قرار ندهید.</p>

<h2 dir="rtl" align="right">روش‌ها در یک نگاه</h2>

<table dir="rtl">
  <thead><tr><th align="right">روش</th><th align="right">Validation</th><th align="right">Wildcard</th><th align="right">Renew خودکار</th></tr></thead>
  <tbody>
    <tr><td align="right">Manual Certbot</td><td align="right">DNS-01 دستی</td><td align="right">بله</td><td align="right">خیر</td></tr>
    <tr><td align="right">NGINX Plugin</td><td align="right">HTTP-01 روی Port 80</td><td align="right">خیر</td><td align="right">بله</td></tr>
    <tr><td align="right">Cloudflare Plugin</td><td align="right">DNS-01 خودکار</td><td align="right">بله</td><td align="right">بله</td></tr>
  </tbody>
</table>

> [!NOTE]
> <p dir="rtl" align="right">اشاره کردن <span dir="ltr">DNS Record</span> به سرور با <span dir="ltr">DNS-01 Challenge</span> فرق دارد. پلاگین <span dir="ltr">NGINX</span> معمولاً از <span dir="ltr">HTTP-01</span> روی <span dir="ltr">Port 80</span> استفاده می‌کند. برای <span dir="ltr">Wildcard</span> باید از <span dir="ltr">DNS-01</span> استفاده کنیم.</p>

<h2 dir="rtl" align="right">پیش‌نیازها</h2>

<ul dir="rtl" align="right">
  <li>یک Domain عمومی که به DNS آن دسترسی دارید</li>
  <li>ساعت درست سیستم</li>
  <li>دسترسی root یا sudo</li>
  <li>نصب بودن NGINX برای روش دوم</li>
  <li>استفاده از Cloudflare DNS برای روش سوم</li>
</ul>

<p dir="rtl" align="right">اول ساعت سیستم را چک کنید:</p>

~~~bash
timedatectl status
~~~

<h2 dir="rtl" align="right">نصب <span dir="ltr">Certbot</span> و پلاگین <span dir="ltr">NGINX</span></h2>

~~~bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
~~~

<p dir="rtl" align="right">بعد نصب را بررسی کنید:</p>

~~~bash
certbot --version
sudo certbot plugins
~~~

<p dir="rtl" align="right">در لیست پلاگین‌ها باید <code dir="ltr">nginx</code> را ببینید.</p>

> [!WARNING]
> <p dir="rtl" align="right">با گزینه <code dir="ltr">--register-unsafely-without-email</code> هیچ ایمیلی برای انقضا یا مشکل <span dir="ltr">Renew</span> نمی‌گیرید. پس در محیط واقعی باید خطای Renew را با روش دیگری مانیتور کنید.</p>

<h2 dir="rtl" align="right">مرحله ۱: گرفتن <span dir="ltr">Wildcard Certificate</span> با <span dir="ltr">DNS</span> دستی</h2>

<p dir="rtl" align="right">این دستور برای <code dir="ltr">example.com</code> و <code dir="ltr">*.example.com</code> یک Certificate می‌گیرد:</p>

~~~bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --register-unsafely-without-email \
  --agree-tos \
  -d "example.com" \
  -d "*.example.com"
~~~

<p dir="rtl" align="right"><span dir="ltr">Certbot</span> یک <span dir="ltr">TXT Record</span> شبیه این نشان می‌دهد:</p>

~~~text
Name:  _acme-challenge.example.com
Type:  TXT
Value: <ACME_VALIDATION_VALUE>
~~~

<p dir="rtl" align="right">این <span dir="ltr">TXT Record</span> را در DNS بسازید. قبل از زدن <span dir="ltr">Enter</span>، با دستورهای زیر چک کنید که Record از اینترنت دیده شود:</p>

~~~bash
dig +short TXT _acme-challenge.example.com @1.1.1.1
dig +short TXT _acme-challenge.example.com @8.8.8.8
~~~

<p dir="rtl" align="right">وقتی مقدار درست را در خروجی دیدید، به <span dir="ltr">Certbot</span> برگردید و ادامه دهید.</p>

> [!WARNING]
> <p dir="rtl" align="right">ممکن است <span dir="ltr">Certbot</span> بیشتر از یک <span dir="ltr">TXT Value</span> بدهد. تا پایان همه Challengeها هیچ‌کدام را پاک نکنید.</p>

<p dir="rtl" align="right">بعد از صدور، فایل‌ها در این مسیرها قرار می‌گیرند:</p>

~~~text
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
/etc/letsencrypt/live/example.com/cert.pem
/etc/letsencrypt/live/example.com/chain.pem
~~~

<p dir="rtl" align="right">Certificate را بررسی کنید:</p>

~~~bash
sudo certbot certificates

sudo openssl x509 \
  -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
~~~

> [!IMPORTANT]
> <p dir="rtl" align="right">روش <span dir="ltr">Manual DNS</span> به‌صورت خودکار Renew نمی‌شود. موقع تمدید باید دوباره دستور را اجرا کنید و <span dir="ltr">TXT Record</span> جدید بسازید.</p>

<h2 dir="rtl" align="right">مرحله ۲: گرفتن Certificate با پلاگین <span dir="ltr">NGINX</span></h2>

<p dir="rtl" align="right">این روش از <span dir="ltr">HTTP-01</span> استفاده می‌کند و تنظیمات <span dir="ltr">NGINX</span> را هم خودش تغییر می‌دهد.</p>

<h3 dir="rtl" align="right">آماده کردن <span dir="ltr">DNS</span></h3>

<p dir="rtl" align="right">یک <span dir="ltr">A Record</span> یا <span dir="ltr">AAAA Record</span> بسازید که به سرور <span dir="ltr">NGINX</span> اشاره کند:</p>

~~~text
app.example.com  A  192.0.2.20
~~~

<p dir="rtl" align="right">بعد نتیجه را چک کنید:</p>

~~~bash
dig +short app.example.com A
dig +short app.example.com AAAA
~~~

<h3 dir="rtl" align="right">آماده کردن <span dir="ltr">NGINX Server Block</span></h3>

~~~nginx
server {
    listen 80;
    listen [::]:80;

    server_name app.example.com;

    root /var/www/app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
~~~

<p dir="rtl" align="right">تنظیمات را تست و <span dir="ltr">Reload</span> کنید:</p>

~~~bash
sudo nginx -t
sudo systemctl reload nginx
~~~

<p dir="rtl" align="right">قبل از ادامه، مطمئن شوید <span dir="ltr">TCP Port 80</span> از اینترنت در دسترس است.</p>

<h3 dir="rtl" align="right">صدور و نصب Certificate</h3>

~~~bash
sudo certbot --nginx \
  --register-unsafely-without-email \
  --agree-tos \
  --redirect \
  -d "app.example.com"
~~~

<p dir="rtl" align="right">این دستور <span dir="ltr">HTTP-01 Validation</span> را انجام می‌دهد، Certificate را می‌گیرد، <span dir="ltr">Server Block</span> را تغییر می‌دهد و <span dir="ltr">HTTP</span> را به <span dir="ltr">HTTPS</span> Redirect می‌کند.</p>

<p dir="rtl" align="right">نتیجه را تست کنید:</p>

~~~bash
sudo nginx -t
sudo systemctl --no-pager status nginx
sudo certbot certificates
curl -I https://app.example.com/
~~~

> [!NOTE]
> <p dir="rtl" align="right">پلاگین <span dir="ltr">NGINX</span> برای <span dir="ltr">Wildcard</span> مناسب نیست، چون از <span dir="ltr">HTTP-01</span> استفاده می‌کند.</p>

<h2 dir="rtl" align="right">مرحله ۳: انجام خودکار <span dir="ltr">DNS Verification</span> با <span dir="ltr">Cloudflare</span></h2>

<p dir="rtl" align="right">این پلاگین <span dir="ltr">TXT Record</span> را از طریق <span dir="ltr">Cloudflare API</span> می‌سازد و بعد از Validation پاک می‌کند. برای Wildcard و Renew خودکار گزینه مناسبی است.</p>

<h3 dir="rtl" align="right">نصب پلاگین</h3>

~~~bash
sudo apt update
sudo apt install -y python3-certbot-dns-cloudflare
~~~

~~~bash
sudo certbot plugins | grep -A3 dns-cloudflare
~~~

<h3 dir="rtl" align="right">ساخت <span dir="ltr">API Token</span></h3>

<p dir="rtl" align="right"><span dir="ltr">Token</span> را فقط با این دسترسی بسازید:</p>

~~~text
Permission: Zone / DNS / Edit
Resource:   Include / Specific zone / example.com
~~~

> [!CAUTION]
> <p dir="rtl" align="right">از <span dir="ltr">Global API Key</span> استفاده نکنید. Token را فقط به همان Zone موردنیاز محدود کنید.</p>

<p dir="rtl" align="right">مسیر امن فایل Credential را بسازید:</p>

~~~bash
sudo install -d -o root -g root -m 0700 /root/.secrets/certbot
sudo nano /root/.secrets/certbot/cloudflare.ini
~~~

<p dir="rtl" align="right">داخل فایل فقط این خط را قرار دهید:</p>

~~~ini
dns_cloudflare_api_token = <CLOUDFLARE_API_TOKEN>
~~~

<p dir="rtl" align="right">دسترسی فایل را محدود کنید:</p>

~~~bash
sudo chown root:root /root/.secrets/certbot/cloudflare.ini
sudo chmod 600 /root/.secrets/certbot/cloudflare.ini
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
~~~

<p dir="rtl" align="right">خروجی Permission باید شبیه این باشد:</p>

~~~text
-rw------- root:root /root/.secrets/certbot/cloudflare.ini
~~~

<h3 dir="rtl" align="right">گرفتن <span dir="ltr">Wildcard Certificate</span></h3>

~~~bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  --register-unsafely-without-email \
  --agree-tos \
  -d "example.com" \
  -d "*.example.com"
~~~

<p dir="rtl" align="right">عدد <code dir="ltr">60</code> به DNS فرصت می‌دهد تا <span dir="ltr">TXT Record</span> را قبل از Validation منتشر کند.</p>

> [!NOTE]
> <p dir="rtl" align="right">فایل <code dir="ltr">cloudflare.ini</code> را حذف نکنید؛ <span dir="ltr">Certbot</span> برای Renewهای بعدی به آن نیاز دارد.</p>

<h3 dir="rtl" align="right">تنظیم Certificate روی <span dir="ltr">NGINX</span></h3>

~~~nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/example;
    index index.html;
}
~~~

~~~bash
sudo nginx -t
sudo systemctl reload nginx
~~~

<h2 dir="rtl" align="right">بررسی Certificate</h2>

~~~bash
sudo certbot certificates
~~~

~~~bash
sudo openssl x509 \
  -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -noout \
  -subject \
  -issuer \
  -serial \
  -dates \
  -fingerprint \
  -sha256 \
  -ext subjectAltName
~~~

<p dir="rtl" align="right">اتصال واقعی را هم تست کنید:</p>

~~~bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -showcerts \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

<p dir="rtl" align="right">خروجی سالم:</p>

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

<h2 dir="rtl" align="right"><span dir="ltr">Renew</span> و نگهداری</h2>

<p dir="rtl" align="right">اول Renew را به‌صورت آزمایشی اجرا کنید:</p>

~~~bash
sudo certbot renew --dry-run
~~~

<p dir="rtl" align="right"><span dir="ltr">Timer</span> را بررسی کنید:</p>

~~~bash
sudo systemctl status certbot.timer --no-pager
sudo systemctl list-timers --all | grep certbot
~~~

~~~bash
sudo ls -l /etc/letsencrypt/renewal/
~~~

> [!IMPORTANT]
> <p dir="rtl" align="right">روش‌های <span dir="ltr">NGINX Plugin</span> و <span dir="ltr">Cloudflare Plugin</span> می‌توانند خودکار Renew شوند. روش <span dir="ltr">Manual DNS</span> بدون Hook خودکار Renew نمی‌شود.</p>

<p dir="rtl" align="right">برای تست <span dir="ltr">Reload</span> شدن <span dir="ltr">NGINX</span> بعد از Renew:</p>

~~~bash
sudo certbot renew \
  --dry-run \
  --deploy-hook "systemctl reload nginx"
~~~

<h2 dir="rtl" align="right">خطاهای رایج</h2>

<h3 dir="rtl" align="right"><span dir="ltr">TXT Record</span> دیده نمی‌شود</h3>

~~~bash
dig +trace _acme-challenge.example.com TXT
dig +short TXT _acme-challenge.example.com @1.1.1.1
~~~

<p dir="rtl" align="right">کمی صبر کنید و مطمئن شوید Record را در Zone درست ساخته‌اید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">NGINX Validation</span> خطا می‌دهد</h3>

~~~bash
sudo nginx -t
sudo ss -lntp | grep ':80'
curl -I http://app.example.com/
sudo journalctl -u nginx -n 100 --no-pager
~~~

<p dir="rtl" align="right"><span dir="ltr">DNS</span> و دسترسی <span dir="ltr">TCP/80</span> را بررسی کنید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">Cloudflare Credential</span> خطا می‌دهد</h3>

~~~bash
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
sudo certbot plugins
~~~

<p dir="rtl" align="right">دسترسی Token، Zone، فرمت فایل و نصب بودن پلاگین را چک کنید. Token را در Terminal یا Log چاپ نکنید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">NGINX</span> هنوز Certificate قبلی را نشان می‌دهد</h3>

~~~bash
sudo nginx -T | grep -E 'ssl_certificate|ssl_certificate_key'
sudo nginx -t
sudo systemctl reload nginx
~~~

<p dir="rtl" align="right">مسیرهای <code dir="ltr">fullchain.pem</code> و <code dir="ltr">privkey.pem</code> را در تنظیمات <span dir="ltr">NGINX</span> بررسی کنید.</p>

<h2 dir="rtl" align="right">چک‌لیست نهایی</h2>

<ul dir="rtl" align="right">
  <li>☐ Certbot و پلاگین NGINX نصب شده‌اند.</li>
  <li>☐ برای Wildcard از DNS-01 استفاده شده است.</li>
  <li>☐ قبل از ادامه، TXT Record از اینترنت دیده می‌شود.</li>
  <li>☐ روش NGINX به DNS درست و Port 80 باز دسترسی دارد.</li>
  <li>☐ Cloudflare Token فقط به یک Zone و DNS Edit دسترسی دارد.</li>
  <li>☐ Permission فایل Credential برابر 0600 است.</li>
  <li>☐ هیچ Token، Private Key یا مقدار واقعی داخل ریپو نیست.</li>
  <li>☐ تاریخ انقضا و SANهای Certificate بررسی شده‌اند.</li>
  <li>☐ خروجی TLS مقدار <code dir="ltr">Verify return code: 0 (ok)</code> دارد.</li>
  <li>☐ دستور <code dir="ltr">certbot renew --dry-run</code> بدون خطا اجرا می‌شود.</li>
</ul>

<h2 dir="rtl" align="right">منابع</h2>

<ul dir="rtl" align="right">
  <li><a href="https://eff-certbot.readthedocs.io/en/stable/using.html">Certbot User Guide</a></li>
  <li><a href="https://certbot-dns-cloudflare.readthedocs.io/en/stable/">Certbot DNS Cloudflare Plugin</a></li>
  <li><a href="https://certbot.eff.org/instructions">Certbot NGINX Instructions</a></li>
</ul>
