
# 🌍 Laravel Background Translation System  
### (Queue + Cache + DB | Bangla + English)

এই ডকুমেন্টে দেখানো হবে কিভাবে Laravel প্রজেক্টে  
**Google Translate (background queue সহ)** ব্যবহার করে  
**Facebook-style scalable translation system** তৈরি করা যায়।

---

## 🎯 Purpose / লক্ষ্য

আমরা যা achieve করতে চাই:

- ✅ `Accept-Language` header থেকে language detect
- ✅ Request block না করে background এ translate
- ✅ একবার translate → DB তে save
- ✅ Cache + DB fallback
- ✅ Key-based translation (best practice)

---

## 🧱 System Architecture (Overview)

Client Request  
└─ Accept-Language Header  
└─ Helper::translateCached()  
├─ Cache hit → instant response  
├─ DB hit → cache → response  
└─ Miss → Queue job → fallback original text  

👉 User কখনো delay feel করবে না  
👉 Translation async/background এ হবে

---

## 📦 Step 1: Install Package

```bash
composer require stichoza/google-translate-php
```

এই package Google Translate API unofficial ভাবে ব্যবহার করে।

---

## 🗄️ Step 2: Database Table (Key-based)

```bash
php artisan make:migration create_translations_table
```

```php
Schema::create('translations', function (Blueprint $table) {
    $table->id();
    $table->string('key');        // example: product.title.5
    $table->string('lang', 10);   // en, bn, ar
    $table->text('value');
    $table->timestamps();

    $table->unique(['key', 'lang']);
});
```

```bash
php artisan migrate
```

### 🔑 Why key-based?

❌ Direct text translate করলে duplicate হয়  
✅ Key-based হলে scalable & editable

---

## 🧠 Step 3: Translation Helper

```php
<?php

namespace App\Helpers;

use App\Jobs\TranslateTextJob;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Stichoza\GoogleTranslate\Exceptions\LargeTextException;
use Stichoza\GoogleTranslate\Exceptions\RateLimitException;
use Stichoza\GoogleTranslate\Exceptions\TranslationRequestException;
use Stichoza\GoogleTranslate\GoogleTranslate;

class translateHelper
{


    /**
     * @throws LargeTextException
     * @throws RateLimitException
     * @throws TranslationRequestException
     */
    public static function translateText(string $text, string $lang): ?string
    {
        if ($lang === 'en') {
            return $text;
        }
        $tr = new GoogleTranslate();
        $tr->setTarget($lang);

        return $tr->translate($text);
    }
    
    /**
     * @throws LargeTextException
     * @throws RateLimitException
     * @throws TranslationRequestException
     */
    public static function translateCached(string $key, string $text, string $lang)
    {
        if ($lang === 'en') {
            return $text;
        }

        $cacheKey = "translation.{$key}.{$lang}";

        // Return cached value if exists
        if (Cache::has($cacheKey)) {
            return Cache::get($cacheKey);
        }

        // Check DB
        $row = DB::table('translations')
            ->where('key', $key)
            ->where('lang', $lang)
            ->first();

        if ($row) {
            Cache::put($cacheKey, $row->value, 86400);
            return $row->value;
        }
        $translated = self::translateText($text, $lang);
        TranslateTextJob::dispatch($key, $text, $lang);  // Dispatch job ONCE

        return $translated;  //  Fallback response
    }

}

```

👉 Cache miss হলেও user original English text পাবে  
👉 Translation background এ হবে

---

## 🔁 Step 4: Queue Job (Background Translation)

```bash
php artisan make:job TranslateTextJob
```

```php
class TranslateTextJob implements ShouldQueue
{
    public function __construct(
        public string $key,
        public string $text,
        public string $lang
    ) {}

    public function handle()
    {
        if (DB::table('translations')
            ->where('key', $this->key)
            ->where('lang', $this->lang)
            ->exists()) {
            return;
        }

        try {
            $tr = new GoogleTranslate();
            $tr->setTarget($this->lang);

            $translated = $tr->translate($this->text);

            DB::table('translations')->insert([
                'key'   => $this->key,
                'lang'  => $this->lang,
                'value' => $translated,
                'created_at' => now(),
                'updated_at' => now(),
            ]);

        } catch (\Throwable $e) {
            Log::error('Translation failed', [
                'key' => $this->key,
                'lang' => $this->lang,
            ]);
        }
    }
}
```

---

## ⚡ Step 5: Queue & Cache Setup

### `.env`

```env
QUEUE_CONNECTION=redis
CACHE_DRIVER=redis
```

### Run Queue Worker

```bash
php artisan queue:work
```

👉 Production এ supervisor ব্যবহার করা recommended

---

## 🌐 Step 6: API / Resource Usage

```php
$lang = $request->header('Accept-Language', 'en');

return [
    'title' => Helper::translateCached(
        key: "product.title.{$this->id}",
        text: $this->title,
        lang: $lang
    ),
];
```

Client header example:

```http
Accept-Language: bn
```

---

## 🧩 Best Practices

✅ Always keep **English as master language**  
✅ Never translate on request thread  
✅ Use Redis for cache + queue  
✅ Use meaningful translation keys  

---

## 🚀 Optional Advanced Features

- Admin panel for manual translation edit
- Disable auto-translate, only manual
- Cron job for bulk translation
- Language fallback chain (bn → en)
- API middleware based translation

---

## 📌 Summary

✔ Facebook-style translation  
✔ Non-blocking system  
✔ Laravel native  
✔ Highly scalable  

---

### ✨ Maintained by

Backend Stack Team  
Laravel | Redis | Queue | API
