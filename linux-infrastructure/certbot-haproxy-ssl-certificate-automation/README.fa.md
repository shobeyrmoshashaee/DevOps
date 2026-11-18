<p align="center">
  <a href="./README.md">English</a> |
  <strong>فارسی</strong>
</p>

<h1 dir="rtl" align="right">🔒 گرفتن <span dir="ltr">SSL Certificate</span> با <span dir="ltr">Certbot</span> برای <span dir="ltr">HAProxy</span></h1>

<p dir="rtl" align="right">در این راهنما یک <span dir="ltr">Wildcard Certificate</span> می‌گیریم، فایل <span dir="ltr">PEM</span> مناسب <span dir="ltr">HAProxy</span> را می‌سازیم و در آخر Renew و Reload را خودکار می‌کنیم.</p>

> [!IMPORTANT]
> <p dir="rtl" align="right">همه Domainها، IPها و مسیرهای این راهنما نمونه هستند. موقع اجرا، مقادیر محیط خودتان را جایگزین کنید.</p>

> [!CAUTION]
> <p dir="rtl" align="right"><span dir="ltr">API Token</span>، <span dir="ltr">Private Key</span>، فایل Credential، فایل PEM نهایی و محتوای <code dir="ltr">/etc/letsencrypt/</code> را داخل ریپو قرار ندهید.</p>

<h2 dir="rtl" align="right">تفاوت روش <span dir="ltr">NGINX</span> و <span dir="ltr">HAProxy</span></h2>

<p dir="rtl" align="right">پلاگین رسمی <span dir="ltr">NGINX</span> می‌تواند هم Certificate را بگیرد و هم تنظیمات NGINX را تغییر دهد. برای <span dir="ltr">HAProxy</span> چنین پلاگین رسمی داخلی وجود ندارد؛ پس Certificate را با <code dir="ltr">certonly</code> می‌گیریم، فایل PEM را خودمان می‌سازیم و بعد از تست تنظیمات، HAProxy را Reload می‌کنیم.</p>

<h2 dir="rtl" align="right">روش‌ها در یک نگاه</h2>

<table dir="rtl">
  <thead><tr><th align="right">روش</th><th align="right">Validation</th><th align="right">Wildcard</th><th align="right">Renew خودکار</th></tr></thead>
  <tbody>
    <tr><td align="right">Manual Certbot</td><td align="right">DNS-01 دستی</td><td align="right">بله</td><td align="right">خیر</td></tr>
    <tr><td align="right">Cloudflare Plugin</td><td align="right">DNS-01 خودکار</td><td align="right">بله</td><td align="right">بله</td></tr>
    <tr><td align="right">Standalone Certbot</td><td align="right">HTTP-01 روی Port 80</td><td align="right">خیر</td><td align="right">ممکن است با Port 80 تداخل داشته باشد</td></tr>
  </tbody>
</table>

> [!TIP]
> <p dir="rtl" align="right">برای <span dir="ltr">HAProxy</span> معمولاً <span dir="ltr">DNS-01</span> انتخاب بهتری است؛ هم Wildcard می‌دهد و هم لازم نیست HAProxy را Stop کنیم.</p>

<h2 dir="rtl" align="right">نصب <span dir="ltr">Certbot</span> و <span dir="ltr">HAProxy</span></h2>

~~~bash
sudo apt update
sudo apt install -y certbot haproxy
~~~

~~~bash
certbot --version
haproxy -v
sudo systemctl --no-pager status haproxy
~~~

> [!WARNING]
> <p dir="rtl" align="right">با گزینه <code dir="ltr">--register-unsafely-without-email</code> هیچ ایمیلی برای انقضا یا خطای Renew نمی‌گیرید. پس خطای Renew را با روش دیگری مانیتور کنید.</p>

<h2 dir="rtl" align="right">مرحله ۱: گرفتن <span dir="ltr">Wildcard Certificate</span> با <span dir="ltr">DNS</span> دستی</h2>

~~~bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --register-unsafely-without-email \
  --agree-tos \
  -d "example.com" \
  -d "*.example.com"
~~~

<p dir="rtl" align="right"><span dir="ltr">Certbot</span> یک یا چند <span dir="ltr">TXT Record</span> می‌دهد:</p>

~~~text
Name:  _acme-challenge.example.com
Type:  TXT
Value: <ACME_VALIDATION_VALUE>
~~~

<p dir="rtl" align="right">Recordها را در DNS بسازید و قبل از زدن Enter بررسی کنید که از اینترنت دیده شوند:</p>

~~~bash
dig +short TXT _acme-challenge.example.com @1.1.1.1
dig +short TXT _acme-challenge.example.com @8.8.8.8
~~~

> [!WARNING]
> <p dir="rtl" align="right">اگر چند <span dir="ltr">TXT Value</span> دارید، تا پایان همه Challengeها هیچ‌کدام را پاک نکنید.</p>

<p dir="rtl" align="right">Certificate صادرشده را بررسی کنید:</p>

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
> <p dir="rtl" align="right">روش <span dir="ltr">Manual DNS</span> بدون Hook خودکار Renew نمی‌شود. برای محیط واقعی، روش Cloudflare در مرحله ۳ مناسب‌تر است.</p>

<h2 dir="rtl" align="right">مرحله ۲: نصب Certificate روی <span dir="ltr">HAProxy</span></h2>

<p dir="rtl" align="right"><span dir="ltr">HAProxy</span> معمولاً <span dir="ltr">Private Key</span> و <span dir="ltr">Full Chain</span> را از یک فایل PEM می‌خواند.</p>

<h3 dir="rtl" align="right">ساخت مسیر Certificateها</h3>

~~~bash
sudo install \
  -d \
  -o root \
  -g haproxy \
  -m 0750 \
  /etc/haproxy/certs
~~~

<h3 dir="rtl" align="right">ساخت فایل <span dir="ltr">PEM</span></h3>

<p dir="rtl" align="right">اول Private Key و بعد Full Chain را داخل یک فایل قرار می‌دهیم:</p>

~~~bash
sudo sh -c '
  umask 027
  cat \
    /etc/letsencrypt/live/example.com/privkey.pem \
    /etc/letsencrypt/live/example.com/fullchain.pem \
    > /etc/haproxy/certs/example.com.pem
'
~~~

~~~bash
sudo chown root:haproxy /etc/haproxy/certs/example.com.pem
sudo chmod 0640 /etc/haproxy/certs/example.com.pem
~~~

> [!CAUTION]
> <p dir="rtl" align="right">فایل PEM شامل <span dir="ltr">Private Key</span> است. آن را داخل Git قرار ندهید و دسترسی اضافه به آن ندهید.</p>

<h3 dir="rtl" align="right">تنظیم <span dir="ltr">HAProxy</span></h3>

~~~haproxy
frontend public_http
    bind :80
    mode http
    http-request redirect scheme https code 301

frontend public_https
    bind :443 ssl crt /etc/haproxy/certs/example.com.pem alpn h2,http/1.1
    mode http
    option forwardfor
    default_backend application_backend

backend application_backend
    mode http
    balance roundrobin
    server app01 192.0.2.31:8080 check
    server app02 192.0.2.32:8080 check
~~~

<p dir="rtl" align="right">قبل از Reload حتماً تنظیمات را تست کنید:</p>

~~~bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
sudo systemctl --no-pager status haproxy
~~~

~~~bash
sudo ss -lntp | grep -E ':(80|443)\b'
~~~

<h2 dir="rtl" align="right">مرحله ۳: انجام خودکار <span dir="ltr">DNS-01</span> با <span dir="ltr">Cloudflare</span></h2>

~~~bash
sudo apt update
sudo apt install -y python3-certbot-dns-cloudflare
sudo certbot plugins | grep -A3 dns-cloudflare
~~~

<p dir="rtl" align="right"><span dir="ltr">API Token</span> را فقط با این دسترسی بسازید:</p>

~~~text
Permission: Zone / DNS / Edit
Resource:   Include / Specific zone / example.com
~~~

> [!CAUTION]
> <p dir="rtl" align="right">از <span dir="ltr">Global API Key</span> استفاده نکنید. Token را فقط به همان Zone موردنیاز محدود کنید.</p>

~~~bash
sudo install -d -o root -g root -m 0700 /root/.secrets/certbot
sudo nano /root/.secrets/certbot/cloudflare.ini
~~~

~~~ini
dns_cloudflare_api_token = <CLOUDFLARE_API_TOKEN>
~~~

~~~bash
sudo chown root:root /root/.secrets/certbot/cloudflare.ini
sudo chmod 0600 /root/.secrets/certbot/cloudflare.ini
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
~~~

~~~text
-rw------- root:root /root/.secrets/certbot/cloudflare.ini
~~~

<p dir="rtl" align="right">حالا Wildcard Certificate را بگیرید:</p>

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

> [!NOTE]
> <p dir="rtl" align="right">فایل <code dir="ltr">cloudflare.ini</code> را حذف نکنید؛ Certbot برای Renewهای بعدی به آن نیاز دارد.</p>

<h2 dir="rtl" align="right">ساخت خودکار فایل <span dir="ltr">PEM</span> بعد از Renew</h2>

<p dir="rtl" align="right">یک <span dir="ltr">Deploy Hook</span> می‌سازیم تا بعد از Renew، فایل PEM دوباره ساخته و HAProxy با خیال راحت Reload شود.</p>

~~~bash
sudo nano /usr/local/sbin/deploy-certbot-haproxy
~~~

~~~bash
#!/usr/bin/env bash
set -Eeuo pipefail

: "${RENEWED_LINEAGE:?RENEWED_LINEAGE is required}"

cert_name="$(basename "$RENEWED_LINEAGE")"
cert_dir="/etc/haproxy/certs"
target="${cert_dir}/${cert_name}.pem"
backup="${target}.previous"
temporary="$(mktemp "${cert_dir}/.${cert_name}.XXXXXX")"

cleanup() {
  rm -f "$temporary"
}
trap cleanup EXIT

install -d -o root -g haproxy -m 0750 "$cert_dir"

cat \
  "${RENEWED_LINEAGE}/privkey.pem" \
  "${RENEWED_LINEAGE}/fullchain.pem" \
  > "$temporary"

chown root:haproxy "$temporary"
chmod 0640 "$temporary"

openssl pkey -in "$temporary" -noout
openssl x509 -in "$temporary" -noout -subject -issuer -dates

if [[ -f "$target" ]]; then
  cp -a "$target" "$backup"
fi

install -o root -g haproxy -m 0640 "$temporary" "$target"

if ! haproxy -c -f /etc/haproxy/haproxy.cfg; then
  if [[ -f "$backup" ]]; then
    mv -f "$backup" "$target"
  else
    rm -f "$target"
  fi
  echo "HAProxy configuration validation failed; previous certificate restored." >&2
  exit 1
fi

systemctl reload haproxy
rm -f "$backup"

echo "HAProxy certificate deployed: $target"
~~~

<p dir="rtl" align="right">دسترسی Script را تنظیم کنید:</p>

~~~bash
sudo chown root:root /usr/local/sbin/deploy-certbot-haproxy
sudo chmod 0750 /usr/local/sbin/deploy-certbot-haproxy
~~~

<p dir="rtl" align="right">اول خود Hook و بعد Renew را تست کنید:</p>

~~~bash
sudo env \
  RENEWED_LINEAGE=/etc/letsencrypt/live/example.com \
  /usr/local/sbin/deploy-certbot-haproxy
~~~

~~~bash
sudo certbot renew \
  --dry-run \
  --deploy-hook /usr/local/sbin/deploy-certbot-haproxy
~~~

> [!IMPORTANT]
> <p dir="rtl" align="right">قبل از Reload، تنظیمات HAProxy تست می‌شود. اگر تست خطا بدهد، فایل PEM قبلی برمی‌گردد.</p>

<p dir="rtl" align="right">برای اجرای خودکار Hook بعد از Renew، لینک زیر را بسازید:</p>

~~~bash
sudo install \
  -d \
  -o root \
  -g root \
  -m 0755 \
  /etc/letsencrypt/renewal-hooks/deploy

sudo ln -sfn \
  /usr/local/sbin/deploy-certbot-haproxy \
  /etc/letsencrypt/renewal-hooks/deploy/20-haproxy
~~~

~~~bash
sudo systemctl status certbot.timer --no-pager
sudo systemctl list-timers --all | grep certbot
~~~

<h2 dir="rtl" align="right">بررسی Certificate و <span dir="ltr">HAProxy</span></h2>

~~~bash
sudo stat -c '%A %U:%G %n' /etc/haproxy/certs/example.com.pem
sudo openssl pkey -in /etc/haproxy/certs/example.com.pem -noout
sudo openssl x509 -in /etc/haproxy/certs/example.com.pem -noout -subject -issuer -dates
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
~~~

~~~bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -showcerts \
  </dev/null 2>/dev/null \
  | grep -E 'Verification|Verify return code'
~~~

~~~text
Verification: OK
Verify return code: 0 (ok)
~~~

<h2 dir="rtl" align="right">خطاهای رایج</h2>

<h3 dir="rtl" align="right"><span dir="ltr">HAProxy</span> فایل PEM را نمی‌خواند</h3>

~~~bash
sudo openssl pkey -in /etc/haproxy/certs/example.com.pem -noout
sudo openssl x509 -in /etc/haproxy/certs/example.com.pem -noout -subject -issuer
~~~

<p dir="rtl" align="right">ترتیب فایل باید Private Key و بعد Full Chain باشد.</p>

<h3 dir="rtl" align="right">تست تنظیمات <span dir="ltr">HAProxy</span> خطا می‌دهد</h3>

~~~bash
sudo haproxy -c -V -f /etc/haproxy/haproxy.cfg
sudo journalctl -u haproxy -n 100 --no-pager
~~~

<p dir="rtl" align="right">تا وقتی تست درست نشده، HAProxy را Restart یا Reload نکنید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">Cloudflare Validation</span> خطا می‌دهد</h3>

~~~bash
sudo stat -c '%A %U:%G %n' /root/.secrets/certbot/cloudflare.ini
sudo certbot plugins
dig +trace _acme-challenge.example.com TXT
~~~

<p dir="rtl" align="right">Token، Zone، فرمت فایل Credential و زمان DNS Propagation را بررسی کنید.</p>

<h3 dir="rtl" align="right"><span dir="ltr">HAProxy</span> هنوز Certificate قبلی را نشان می‌دهد</h3>

~~~bash
sudo stat /etc/haproxy/certs/example.com.pem
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
~~~

<p dir="rtl" align="right">Fingerprint فایل جدید را با Certificateای که Endpoint نشان می‌دهد مقایسه کنید.</p>

<h2 dir="rtl" align="right">چک‌لیست نهایی</h2>

<ul dir="rtl" align="right">
  <li>☐ Certbot و HAProxy نصب شده‌اند.</li>
  <li>☐ برای Wildcard از DNS-01 استفاده شده است.</li>
  <li>☐ TXT Recordها قبل از ادامه از اینترنت دیده می‌شوند.</li>
  <li>☐ فایل PEM شامل Private Key و Full Chain است.</li>
  <li>☐ مالک فایل <code dir="ltr">root:haproxy</code> و Permission آن <code dir="ltr">0640</code> است.</li>
  <li>☐ قبل از هر Reload، تنظیمات HAProxy تست می‌شود.</li>
  <li>☐ Cloudflare Token فقط به یک Zone و DNS Edit دسترسی دارد.</li>
  <li>☐ هیچ Token، Private Key یا فایل PEM واقعی داخل ریپو نیست.</li>
  <li>☐ خروجی TLS مقدار <code dir="ltr">Verify return code: 0 (ok)</code> دارد.</li>
  <li>☐ Renew آزمایشی همراه Deploy Hook بدون خطا اجرا می‌شود.</li>
</ul>

<h2 dir="rtl" align="right">منابع</h2>

<ul dir="rtl" align="right">
  <li><a href="https://eff-certbot.readthedocs.io/en/stable/using.html">Certbot User Guide</a></li>
  <li><a href="https://certbot-dns-cloudflare.readthedocs.io/en/stable/">Certbot DNS Cloudflare Plugin</a></li>
  <li><a href="https://certbot.eff.org/instructions">Certbot Instructions for HAProxy</a></li>
  <li><a href="https://docs.haproxy.org/">HAProxy Configuration Manual</a></li>
</ul>
