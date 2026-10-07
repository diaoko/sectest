**مهم‌ترین یافته این گزارش: فایل‌های فعال سایت در `/var/www/allyoucansnow` هستند، نه پوشه `app` داخل `/opt`.** بنابراین اصلاحات قبلی در `/opt/docker-apps/allyoucansnow/app` روی سایت فعال اعمال نشده‌اند.

در خروجی `Mounts` این اتصال مشخص است:

```text
Source:      /var/www/allyoucansnow
Destination: /var/www/allyoucansnow
```

یعنی همین پوشه روی میزبان، مستقیماً داخل کانتینر استفاده می‌شود.

### نتیجه بررسی

| یافته | برداشت |
|---|---|
| صفحه اصلی `200` و سرویس انبار `ok` | این دو مسیر پاسخ می‌دهند؛ سلامت همه قابلیت‌ها تأیید نشده |
| `cerca.php` فعال هنوز SQL را با ورودی می‌سازد | آسیب‌پذیری جست‌وجو هنوز اصلاح نشده |
| `scheda.php` فعال هنوز `include` مستقیم دارد | محدودسازی انتخاب فایل هنوز اعمال نشده |
| فایل `error` شامل یک پیام HTML خراب است | احتمالاً حاصل اجرای اشتباه کد PHP در Bash است؛ به‌تنهایی نشانه بدافزار نیست |
| اتصال از `192.168.14.8` تقریباً هر دو دقیقه | احتمالاً سامانه بررسی مسابقه است؛ این یک استنباط است و باید تأیید شود |
| ورود root با کلید از `192.168.14.7` | ممکن است برگزارکننده باشد؛ هویت آن هنوز معلوم نیست |
| مجوزهای sudo برای `deploy` و `warehouse` | دو مسیر دیگر برای بررسی افزایش دسترسی وجود دارد |

**فعلاً SSH، رمزها و دسترسی نشانی‌های `.7` و `.8` را تغییر ندهید.** الگوی ورود نشان می‌دهد ممکن است زیرساخت مسابقه به این دسترسی‌ها وابسته باشد.

### ۱. از پوشه واقعی سایت پشتیبان بگیرید

پشتیبان قبلی شما این پوشه را شامل نمی‌شود. کل این بلوک را یک‌جا اجرا کنید:

```bash
sudo bash <<'SH'
set -eu
task_backup="/root/live-site-backup-$(date +%Y%m%d-%H%M%S)"
install -d -m 700 "$task_backup"
cp -a /var/www/allyoucansnow "$task_backup/"
printf 'Backup saved at: %s\n' "$task_backup"
SH
```

### ۲. اصلاحات را روی فایل‌های فعال انجام دهید

برای جست‌وجو:

```bash
sudo nano /var/www/allyoucansnow/cerca.php
```

اصلاح پارامتری SQL و `htmlspecialchars` که قبلاً توضیح دادم، باید **در این فایل** اعمال شود.

برای صفحه اسناد:

```bash
sudo ls -la /var/www/allyoucansnow/schede
```

```bash
sudo nano /var/www/allyoucansnow/scheda.php
```

کد فهرست نام‌های مجاز و `readfile` پاسخ قبلی را اینجا جایگزین تعریف `$doc` و خط `include` کنید. تمام اسناد معتبر لازم را در فهرست نگه دارید.

با این اتصال مستقیم پوشه، **برای این تغییرهای PHP معمولاً بازسازی کانتینر لازم نیست**. بعد از ذخیره، نسخه داخل کانتینر و پاسخ واقعی سایت را بررسی کنید.

### ۳. بررسی ساختار و عملکرد

دستورها را جداگانه اجرا کنید:

```bash
sudo docker exec allyoucansnow php -l /var/www/allyoucansnow/cerca.php
```

```bash
sudo docker exec allyoucansnow php -l /var/www/allyoucansnow/scheda.php
```

```bash
curl -sS --get --data-urlencode 'q=pupazzo' http://127.0.0.1/cerca.php
```

```bash
curl -sS --get --data-urlencode "q=O'Reilly" http://127.0.0.1/cerca.php
```

```bash
curl -sS --get --data-urlencode 'doc=../index.php' http://127.0.0.1/scheda.php
```

انتظار داریم جست‌وجوی عادی کار کند، تک‌نقل‌قول باعث نمایش خطای دیتابیس نشود و سند خارج از فهرست رد شود. موفقیت `php -l` فقط یعنی ساختار کد درست است، نه اینکه آسیب‌پذیری بسته شده است.

### ۴. دو مجوز sudo جدید را بررسی کنید

این مجوز نگران‌کننده است:

```text
deploy ... NOPASSWD: /usr/bin/vi /etc/nginx/*
```

ویرایشگر `vi` قابلیت اجرای فرمان دارد؛ محدود کردن آرگومان فایل به‌تنهایی آن را به ابزار امن مدیریتی تبدیل نمی‌کند.

این مورد هم به مالکیت اسکریپت و وابستگی‌هایش وابسته است:

```text
warehouse ... NOPASSWD: /usr/bin/python3 /opt/warehouse/sync_stock.py
```

اگر حساب `warehouse` بتواند اسکریپت یا فایل‌های واردشده توسط آن را تغییر دهد، اجرای آن با root خطرناک است. ابتدا این خروجی‌ها را بگیرید:

```bash
sudo namei -l /opt/warehouse/sync_stock.py
```

```bash
sudo ls -la /opt/warehouse
```

```bash
sudo sed -n '1,180p' /opt/warehouse/sync_stock.py
```

```bash
sudo cat /opt/docker-apps/allyoucansnow/rebuild.sh
```

رمزها و کلیدها را قبل از ارسال حذف کنید. **اولویت فعلی، اصلاح دو فایل در مسیر واقعی سایت است؛ سپس بررسی دسترسی `deploy` و `warehouse`.**
