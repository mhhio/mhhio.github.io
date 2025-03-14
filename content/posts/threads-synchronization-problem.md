+++
title = 'هماهنگ‌سازی thread ها: یک سوال به ظاهر ساده!'
date = 2024-11-13T16:22:00+03:30
draft = false
+++

[//]: # (![ساخته شده با DALL-E]&#40;/images/threads-synchronization-problem-cover.webp&#41;)

{{< figure src="/images/threads-synchronization-problem-cover.webp"
alt="My image"
caption="ساخته شده با DALL-E">}}

## یک مسئله‌ی ساده 
فرض کنید می خوایم برنامه ای بنویسیم که به این صورت کار کنه: یک کلاس داریم که دو تا متد داره متد odd و متد even و هر کدوم از این متد ها دارن توی یک [thread](https://en.wikipedia.org/wiki/Thread_(computing)) مجزا اجرا میشن و می خواهیم طوری متد ها کار کنند که نتیجه ی اجرا شدنشون چاپ شدن یک رشته باشه از 0 تا n به صورت زیر:
```bash
// n= 2
012
// n=5
012345
```

به نظر ساده میاد اما مثال خوبی هست که نشون بدیم هماهنگ سازی thread ها در [برنامه نویسی همروند](https://en.wikipedia.org/wiki/Concurrent_computing) چطوری کار می کنه.
خب برای پیاده سازی اولیه من این کد رو زدم:
```java
public class EvenOdd {
    private int n;
    private int currentNumber = 0;

    public EvenOdd(int n) {
        this.n = n;
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        while (currentNumber <= n) {
            while (currentNumber % 2 != 0 && currentNumber <= n) {
            }
            printNumber.accept(currentNumber);
            currentNumber++;
        }
    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        while (currentNumber <= n) {
            while (currentNumber % 2 != 1 && currentNumber <= n) {
            }
            printNumber.accept(currentNumber);
            currentNumber++;

        }
    }
}
```

ولی وقتی این کد رو اجرا کردم هیچ وقت تموم نشد!(خودتون تست کنید!)
این شد که از هوش مصنوعی کمک گرفتم و این راه حل رو بهم پیشنهاد داد:

```java
public class EvenOdd {
    private final int n;
    private final Object lock = new Object();
    private int currentNumber = 0;
    private int turn = 0;

    public EvenOdd(int n) {
        this.n = n;
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        while (this.currentNumber <= this.n) {
            synchronized (lock) {
                if (this.turn != 0 && this.currentNumber <= this.n) {
                    this.lock.wait();
                }
                if (this.currentNumber <= this.n && this.currentNumber % 2 == 0) {
                    printNumber.accept(this.currentNumber);
                    this.currentNumber++;
                    this.turn = 1;
                }
                this.lock.notifyAll();
            }
        }
    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        while (this.currentNumber <= this.n) {
            synchronized (this.lock) {
                if (this.turn != 1 && this.currentNumber <= this.n) {
                    this.lock.wait();
                }
                if (this.currentNumber <= this.n && this.currentNumber % 2 == 1) {
                    printNumber.accept(this.currentNumber);
                    this.currentNumber++;
                    this.turn = 0;
                }
                this.lock.notifyAll();
            }
        }
    }
}

```

و خوش بختانه این کد به خوبی اجرا شد (خودتون تست کنید!)

## خب مشکل کد قبلی چی بود؟

تو کد قبلی من از منطقی به اسم [busy waiting](https://en.wikipedia.org/wiki/Busy_waiting) استفاده کرده بودم مثلا این تیکه از کد

```java
while (currentNumber % 2 != 0 && currentNumber <= n) 
{             }
```

و به دلیل اینکه به صورت غیر thread-safe مقدار currentNumber رو افزایش می دادم اصلا معلوم نبود که چه متغیری وارد شرط busy waiting میشه و ممکنه اتفاقی هیچ وقت از این حلقه خارج نشه و این اتفاق دقیقا می افتاد.

## چه درسهایی از کد جدید یاد گرفتم؟

۱-الگوهای همگام سازی thread ها: برای اطمینان از هماهنگی صحیح بین threadها، از الگوهای همگام‌سازی استفاده می‌شود. این الگوها به ما کمک می‌کنند تا دسترسی به منابع اشتراکی را کنترل کرده و ترتیب اجرای عملیات را مدیریت کنیم.مهم‌ترین الگوهای همگام‌سازی عبارتند از:
* قفل‌ها (Locks)
* شرایط متقابل (Condition Variables)
* سمافورها (Semaphores)

۲-مدیریت وضعیت: برای حفظ وضعیت اشتراکی باید از قفل‌ها یا سایر مکانیزم‌های همگام‌سازی برای محافظت از دسترسی به وضعیت اشتراکی استفاده کنیم.همچنین باید مطمئن شویم که رفتن از یک حالت به حالت دیگر  به صورت اتمی انجام می‌شوند.برای این کار می شود از قفل‌ها و شرایط متقابل برای هماهنگی انتقالات استفاده کرد.
۳- کارایی و عملکرد: استفاده نکردن از busy waiting و شرایطی که منجر به race condition بشود یا deadlock اتفاق بیافتد.
۴- درنظر گرفتن حالت های لبه: باید شرایط غیر معمول یا زمانی که ممکن است خطایی رخ بدهد رو قبل از وقوع پیشبینی کنیم مثل این شرط هایی که در کد وجود داره

```bash
this.currentNumber <= this.n 
```

## کاربرد برنامه نویسی همروند در دنیای واقعی چیست؟

موارد زیر از جمله سناریو هایی هستند که برنامه های همروند توشون کاربرد دارند:

سامانه‌‌های تولید کننده مصرف کننده(pub-sub): سامانه های پیام رسان، سامانه‌های پردازش رویداد و ...

سامانه‌های پردازش کلان داده(big data): پردازش رویداد به صورت دسته ای یا جریانی، پردازش تصویری یا صوت و ...

سامانه‌های نوبتی(turn-based):موتورهای بازی، برنامه ریزی منابع، ارکستراسیون وظایف و ...

---

مشکل EvenOdd، در حالی که به ظاهر ساده است، بسیاری از مفاهیم اساسی در برنامه نویسی همروند را در بر می گیرد و بهمون موارد زیر رو یاد می دهد
* هماهنگ سازی threadها
* مدیریت وضعیت
* جلوگیری از وقوع race condition
* استفاده ی بهینه‌ی منابع

و از همه مهمتر بهمون نشون میده که چطوری هماهنگ سازی مناسب threadها می تونه منجر به توسعه‌ی سیستم های همروند قابل اتکا بشه.

---

در پایان ممنونم که تا انتهای این مطلب همراهم بودین خوشحال میشم هرگونه سوال یا بهبودی راجع به این پست داشتین از طریق [ایمیل](mailto:mhhajivandy@gmail.com) با من در تماس باشین.

---

پ.ن.۱: فکر می کنم به راحتی بتونید [این مسئله](https://leetcode.com/problems/print-zero-even-odd/) رو از سایت leetcode حل کنید.

پ.ن.۲: به نظرم دونستن این مفاهیم و موارد مشابه برای موفقیت در مصاحبه‌های مرتبط با مهندسی نرم افزار ضروریه!