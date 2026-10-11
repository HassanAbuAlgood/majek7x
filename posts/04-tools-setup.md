# 4️⃣ إعداد بيئة العمل (دليل عملي سريع)

> الهدف من الملف: تجهيز بيئة يقدر معها أي حد يطبّق باقي المسار من غير تعقيد.

---

## 1. المتصفح والـ Proxy

1. **Burp Suite Community** — [portswigger.net/burp/communitydownload](https://portswigger.net/burp/communitydownload)
2. شغّله → `Proxy → Proxy Settings` → اتأكد الليستنر على `127.0.0.1:8080`
3. في المتصفح: ثبّت **FoxyProxy** وضيف profile للـ `127.0.0.1:8080`
4. زُر `http://burp` وحمّل **CA Certificate** وثبّته في المتصفح (لازم عشان HTTPS)

> 💡 استخدم متصفح منفصل للاختبار ومتسجلش فيه دخول على حساباتك الشخصية.

---

## 2. الإضافات الأساسية (Extensions → BApp Store)

| الإضافة | الفايدة | أولوية |
|---|---|---|
| **Autorize** | ⭐ أهم أداة في المسار — بتعيد كل request بتوكن مخفض الصلاحية وتقارن | 🔴 ضروري |
| **Logger++** | سجلّ كامل لكل الطلبات وتصفيتها | 🔴 ضروري |
| **Param Miner** | يكتشف بارامترات مخفية في الـ headers والـ body | 🟠 مهم |
| **JWT Editor** | تعديل وفحص التوكنز (alg none, key confusion) | 🟠 مهم |
| **Turbo Intruder** | تعداد سريع للـ IDs — **استخدمه بحذر شديد** | 🟠 مهم |
| **Wsdler / OpenAPI Parser** | استيراد swagger وتوليد requests | 🟡 مفيد |
| **Hackvertor** | ترميز/فك ترميز سريع | 🟡 مفيد |
| **Burp Bounty** | قواعد تمييز للـ scan | 🟡 مفيد |
| **Freddy, Deserialization Bug Finder** | اكتشاف أنواع ثغرات تانية | ⚪ اختياري |

---

## 3. إعداد Autorize (اعمله بالحرف)

Autorize هي اللي هتخليك تلاقي IDOR بشكل **منهجي** بدل الصدفة.

1. ثبّتها من BApp Store
2. سجّل دخول بحساب **المهاجم (A)** — الأقل صلاحية
3. في تبويب **Autorize** → حط الكوكيز/التوكن بتاع A في خانة `Insert injected header / cookies`
   (أسهل طريقة: كليك يمين في الـ HTTP history على request بتاع A → `Autorize → Send to Autorize`)
4. فعّل `Autorize is on`
5. **اتصفح التطبيق بحساب الضحية أو الإداري (B)** بشكل طبيعي
6. Autorize هتعيد **كل** request تلقائيًا بتوكن A وتقارن النتيجة:

| الحالة | المعنى | تصرفك |
|---|---|---|
| 🟢 **Enforced** | A اترفض زي ما كان مفروض | تمام، كمل |
| 🔴 **Bypassed!** | A وصل لبيانات B 👈 | **تحقق يدويًا فورًا** |
| ⚪ **Is enforced!!! (灰色)** | مش قادر يحدد | راجعه يدويًا |

> ⚠️ Autorize بتولّد **أضعاف** عدد الطلبات. على أهداف حقيقية:
> خليها على مسارات محددة (scope ضيق) ومتعملش Browse لكل التطبيق دفعة واحدة.

---

## 4. إدارة حسابين (A مهاجم / B ضحية)

دي الـ setup اللي هتستخدمها في كل اختبار IDOR:

```
الحساب A (المهاجم)   → توكن: eyJ...user_id=1...
الحساب B (الضحية)    → توكن: eyJ...user_id=2... + عنده كائنات حقيقية
```

**الطريقة في Burp**:
- `Project options → Sessions → Session Handling Rules`
  - Rule 1: لو الـ request فيه header `X-Test-A` → بدّل الـ Authorization بتوكن A
  - Rule 2: لو فيه `X-Test-B` → بدّل بتوكن B
- أو الأسهل: استخدم **Repeater** وبدّل التوكن يدويًا (أبطأ بس أوضح)

**ما تعمله**:
1. سجّل دخول بـ B وأنشئ كائنات حقيقية (طلب، فاتورة، ملف، تذكرة)
2. سجّل أرقام معرّفاتها
3. سجّل دخول بـ A
4. جرّب توصل لكائنات B بـ A

---

## 5. أدوات الاستطلاع (للأسبوع 9)

```bash
# Subdomains
subfinder -d target.com -o subs.txt
amass enum -passive -d target.com -o amass.txt

# URLs قديمة من الأرشيف
echo target.com | gau --threads 5 >> urls.txt
waybackurls target.com >> urls.txt

# استخراج endpoints من ملفات JS
cat subs.txt | getJS --complete > js.txt
python3 linkfinder.py -i https://target.com/app.js -o cli

# تصفية البارامترات المثيرة
cat urls.txt | grep -Ei "id=|user|account|order|doc|file|token|key" > interesting.txt
```

**مصادر للـ API docs** (مفيش أدوات، يدوي وبسيط):
```
/api-docs        /swagger.json      /openapi.json     /v1/api-docs
/graphql         /graphiql          /api/v1/docs      /docs
/.well-known/    /redoc             Postman collections على GitHub
```

---

## 6. بيئة التدريب المحلية (شغّلها من الأسبوع 3)

| الأداة | الرابط | ملاحظة |
|---|---|---|
| **OWASP Juice Shop** | [github.com/juice-shop/juice-shop](https://github.com/juice-shop/juice-shop) | Docker: `docker run -p 3000:3000 bkimminich/juice-shop` |
| **crAPI** | [github.com/OWASP/crAPI](https://github.com/OWASP/crAPI) | مصمم خصيصًا لثغرات الـ API 👈 |
| **vAPI** | [github.com/roottusk/vapi](https://github.com/roottusk/vapi) | OWASP API Top 10 |
| **DVWA** | [github.com/digininja/DVWA](https://github.com/digininja/DVWA) | كلاسيكي للمبتدئين |
| **PortSwigger Academy** | [portswigger.net/web-security](https://portswigger.net/web-security) | ⭐ أهم مصدر، مجاني ومنهجي |

---

## 7. قوايد السلامة أثناء الاستخدام

| ❌ متعملش | ✅ اعمل |
|---|---|
| Fuzzing بـ threads عالية على هدف حقيقي | threads قليلة + تأخير بين الطلبات |
| تشغّل ماسح آلي على برنامج بيمنع الـ scanners | استطلاع يدوي وهادف |
| تعداد 100 ألف ID | عيّنة صغيرة (50-200) تكفي للإثبات |
| تختبر بره الـ Scope | التزم بالـ Scope حرفيًا |

> 🚨 **Turbo Intruder و Intruder بيولّدوا نشاط عنيف.**
> على أهداف حقيقية: استخدمهم بـ `requests per connection` منخفض،
> ولو الهدف عليه حمل إنتاجي حقيقي، بلّغ الـ program أو استخدم عيّنة صغيرة جدًا.

---

## ✅ Checklist تجهيز (تأكد منها قبل الأسبوع 3)

- [ ] Burp شغال وبيعترض HTTPS صح
- [ ] Autorize مثبتة ومضبوطة على توكن A
- [ ] عندك حسابين على بيئة تدريبية
- [ ] Juice Shop أو crAPI شغال محليًا
- [ ] حساب على PortSwigger Academy
- [ ] حساب على HackerOne + Intigriti
