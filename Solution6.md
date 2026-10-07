**جست‌وجو هنوز اصلاح نشده است.** درخواست `O'Reilly` باعث خطای SQL شده و خروجی نشان می‌دهد برنامه همچنان از `$db->query(...)` استفاده می‌کند. درست بودن ساختار PHP با `php -l`، به معنی امن بودن کد نیست.

همچنین:

- پاسخ خالی `scheda.php` ثابت نمی‌کند انتخاب فایل امن شده؛ باید کد فعلی را ببینیم.
- عبارت‌های `deploy ... NOPASSWD ...` و `warehouse ...` **دستور نیستند**؛ توضیح مجوزها بودند. آن‌ها را در ترمینال اجرا نکنید.

برای رفع جست‌وجو، این مراحل را انجام دهید.

**۱. فایل فعال را باز کنید:**

```bash
sudo nano /var/www/allyoucansnow/cerca.php
```

سه خط زیر را پیدا کنید:

```php
$sql = "SELECT id, nome, prezzo, descrizione FROM prodotti WHERE nome LIKE '%" . $q . "%'";
$db = ays_db();
$res = $db->query($sql);
```

آن‌ها را **کامل حذف کنید** و این بلوک را جایگزین کنید:

```php
$db = ays_db();
$res = false;

try {
    $stmt = $db->prepare(
        "SELECT id, nome, prezzo, descrizione FROM prodotti WHERE nome LIKE ?"
    );
    $like = '%' . $q . '%';
    $stmt->bind_param('s', $like);
    $stmt->execute();
    $res = $stmt->get_result();
    $stmt->close();
} catch (Throwable $e) {
    error_log('Search query failed');
}
```

**۲. خط تعریف `$q` را با این کد جایگزین کنید:**

```php
$q = $_GET['q'] ?? '';
if (!is_string($q)) {
    $q = '';
}
```

**۳. خط نمایش خطای دیتابیس را جایگزین کنید:**

```php
echo "<p class='err'>Ricerca temporaneamente non disponibile.</p>";
```

خط پیام بدون نتیجه را هم جایگزین کنید:

```php
echo "<p>Nessun risultato per: " .
    htmlspecialchars($q, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') .
    "</p>";
```

با **Ctrl+O، سپس Enter** ذخیره کنید و با **Ctrl+X** خارج شوید. کدهای PHP بالا فقط برای ویرایشگر هستند؛ آن‌ها را در Bash وارد نکنید.

**۴. این دستورهای ترمینال را یکی‌یکی اجرا کنید:**

```bash
sudo docker exec allyoucansnow php -l /var/www/allyoucansnow/cerca.php
```

```bash
sudo sed -n '1,120p' /var/www/allyoucansnow/cerca.php
```

```bash
curl -sS --get --data-urlencode "q=O'Reilly" http://127.0.0.1/cerca.php
```

```bash
curl -sS --get --data-urlencode 'q=pupazzo' http://127.0.0.1/cerca.php
```

انتظار: `O'Reilly` پیام بدون نتیجه بدهد و جست‌وجوی `pupazzo` همچنان محصولات را نمایش دهد. اگر هر دو درخواست پیام «سرویس موقتاً در دسترس نیست» بدهند، اصلاح کامل نشده و باید پشتیبانی `get_result` و خطای سمت سرور بررسی شود.

**۵. برای بررسی صفحه اسناد، این خروجی را هم بفرستید:**

```bash
sudo cat /var/www/allyoucansnow/scheda.php
```

فعلاً بازسازی کانتینر یا تغییر sudo لازم نیست؛ ابتدا همین دو صفحه را اصلاح و تأیید کنیم.
