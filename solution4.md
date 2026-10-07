**پشتیبان با موفقیت ساخته شده است:**

```text
/root/exam-backup-20261007-120650
```

اما در `scheda.php` یک خطای مشخص دیده می‌شود:

```php
doc = isset(_GET['doc']) ? basename($_GET['doc']) : 'pupazzo-classico.html';
```

قبل از `doc` و `_GET` علامت `$` حذف شده است. این خط، کد PHP را خراب می‌کند. همچنین استفاده از `basename` به‌تنهایی جایگزین مناسبی برای فهرست فایل‌های مجاز نیست.

به‌هم‌ریختگی دستورهای چسبانده‌شده و خطای `.php: command not found` هم نشان می‌دهد بعضی دستورها ناقص وارد شده‌اند؛ این خروجی به‌تنهایی نشانه نفوذ نیست.

**۱. ابتدا اسناد موجود را ببینید:**

```bash
sudo ls -la /opt/docker-apps/allyoucansnow/app/schede
```

**۲. فایل را باز کنید:**

```bash
sudo nano /opt/docker-apps/allyoucansnow/app/scheda.php
```

خط خراب `doc = ...` و خط `include(...)` را حذف کنید و این کد را جای آن‌ها بگذارید:

```php
$allowed = ['pupazzo-classico.html'];
$doc = $_GET['doc'] ?? 'pupazzo-classico.html';
$base = realpath(__DIR__ . '/schede');
$file = false;

if (
    is_string($doc) &&
    in_array($doc, $allowed, true) &&
    $base !== false
) {
    $file = realpath($base . DIRECTORY_SEPARATOR . $doc);
}

if (
    $file === false ||
    !is_file($file) ||
    dirname($file) !== $base
) {
    echo '<p>Documento non disponibile.</p>';
} else {
    readfile($file);
}
```

نام سند پیش‌فرض را با خروجی مرحله ۱ تطبیق دهید. نام سایر اسناد معتبر را هم به `$allowed` اضافه کنید تا صفحات سالم از دسترس خارج نشوند.

این روش فقط اسناد تأییدشده را نمایش می‌دهد و برخلاف `include`، محتوای فایل را به‌عنوان PHP اجرا نمی‌کند. اسناد HTML باید قابل اعتماد باشند.

با **Ctrl+O، سپس Enter** ذخیره کنید و با **Ctrl+X** خارج شوید.

**۳. این دو دستور را جداگانه اجرا کنید و خروجی را بفرستید:**

```bash
sudo docker inspect allyoucansnow --format '{{json .Mounts}}'
```

```bash
sudo cat /opt/docker-apps/allyoucansnow/rebuild.sh
```

این خروجی‌ها مشخص می‌کنند چگونه اصلاح را به برنامه فعال منتقل کنیم. فعلاً `rebuild.sh` را اجرا نکنید.

سپس ساختار فایل داخل کانتینر را بررسی کنید:

```bash
sudo docker exec allyoucansnow php -l /var/www/allyoucansnow/scheda.php
```

**این بررسی فقط نسخه داخل کانتینر را می‌سنجد؛ ممکن است هنوز نسخه ویرایش‌شده میزبان نباشد.**

اسکریپت خروجی سفارش‌ها نیز هنوز رمز دیتابیس را داخل متن خود دارد. آن را در خروجی‌های بعدی حذف کنید. این نسخه پشتیبان، وضعیت فعلی را حفظ می‌کند و ممکن است همین خطای PHP را هم داشته باشد؛ نسخه سالم اولیه محسوب نمی‌شود.
