# 🔟 الاستطلاع الموجه للـ IDOR

> **الهدف من الاستطلاع مش "أجمع أكبر عدد URLs".**
> الهدف: **ألاقي الكائنات والمعرّفات بتاعتها** — دي مادة الـ IDOR الخام.

---

## 🎯 قاعدة ذهبية

> **الاستطلاع العشوائي = صفر نتايج.**
> الاستطلاع الموجه = إنت بتدور على **3 حاجات بس**:
> 1. **أسماء الكائنات** (orders, invoices, teams, files…)
> 2. **أشكال المعرّفات** (numeric, UUID, base64…)
> 3. **أماكن بتسرب المعرّفات** (ملفات JS، روابط، أرشيف، موبايل)

---

## 🔍 المرحلة 1: توسيع السطح

### Subdomains
```bash
subfinder -d target.com -silent -o subs.txt
amass enum -passive -d target.com -o amass.txt
assetfinder --subs-only target.com >> subs.txt
sort -u subs.txt -o subs.txt

# فلترة الحية
httpx -l subs.txt -silent -o live.txt
```

### أولوية النطاقات الفرعية للـ IDOR
```
api.target.com          👈 الكنز الأول
app.target.com
internal.target.com     (لو في الـ scope)
admin.target.com
staging.target.com      👈 حماية أضعف غالبًا
dev.target.com
mobile-api.target.com   👈 غالبًا منسي
*.target.com (wildcard) 👈 لو في الـ Scope = مساحة واسعة
```

---

## 🔍 المرحلة 2: جمع الـ URLs

```bash
# الأرشيف (URLs قديمة = endpoints منسية = حماية أقدم)
echo target.com | gau --threads 5 > urls.txt
waybackurls target.com >> urls.txt

# تصفية للبارامترات المثيرة
cat urls.txt | grep -Ei "id=|user|account|order|doc|file|invoice|team|ticket|key|token" > hot.txt

# أنواع الملفات
cat urls.txt | grep -Ei "\.(json|pdf|xlsx|csv|zip)$" > files.txt
```

> 💡 الـ URLs القديمة من Wayback = **endpoints اتنسوا تحديث حمايتهم**. قيمة عالية جدًا.

---

## 🔍 المرحلة 3: استخراج الـ Endpoints من ملفات JS 👈 أهم مرحلة

الكائنات والمعرّفات غالبًا **مكتوبة صلبة** في كود الـ frontend.

```bash
# 1. اجمع ملفات JS
cat live.txt | getJS --complete > js.txt
cat live.txt | waybackurls | grep "\.js$" >> js.txt
sort -u js.txt -o js.txt

# 2. حمّلها وحللها
mkdir js && cd js
cat ../js.txt | xargs -P 10 -I{} curl -s -O {}

# 3. استخرج الـ endpoints
python3 linkfinder.py -i ../js.txt -o cli
cat *.js | grep -oEi '"/(api|v1|v2|v3|internal)/[a-zA-Z0-9/_{}-]+"' | sort -u
```

### إيه اللي تدور عليه جوه الـ JS؟

| النمط | مثال |
|---|---|
| مسارات API | `"/api/v2/users/" + userId + "/orders"` |
| أسماء الكائنات | `order`, `invoice`, `workspace`, `member`, `invite` |
| أسماء الحقول | `role`, `tenantId`, `ownerId`, `isAdmin` |
| قيم UUID صلبة | `f47ac10b-58cc-4372-a567-0e02b2c3d479` |
| مفاتيح/أسرار | `apiKey`, `client_secret`, tokens |
| تعليقات | `// TODO: add ownership check` 👈 بيحصل فعلًا |

---

## 🔍 المرحلة 4: وثائق الـ API (الطريق المختصر)

جرّب المسارات دي يدويًا:

```
/swagger.json        /api-docs          /openapi.json      /v1/api-docs
/swagger-ui.html     /docs              /redoc             /api/v1/docs
/graphql             /graphiql          /v1/graphql        /api/graphql
/.well-known/openapi.json
```

ودور على GitHub:
```
site:github.com "target.com" postman_collection
site:github.com "target.com" swagger
"api.target.com" in:url
```

لو لقيت swagger → استورده في Burp بـ `OpenAPI Parser` أو حوّله لـ Postman
→ **كل الـ endpoints جاهزة للاختبار في دقايق**. 👈 أكبر اختصار في الاستطلاع كله.

---

## 🔍 المرحلة 5: تطبيق الموبايل (منجم ذهب)

لو الهدف عنده تطبيق iOS/Android:

1. حمّل الـ APK (`apkpure`, `apkmirror`, أو من الجهاز بـ `adb`)
2. فكّه: `apktool d app.apk` أو `jadx`
3. دور على:
   - URLs الـ API (غالبًا hardcoded)
   - أسماء البارامترات
   - معرّفات وأنماط
4. أو اعترض الترافيك بـ Burp + Frida/objection لتخطي pinning

> 🎯 الـ APIs بتاعة الموبايل غالبًا **أقدم** من الويب ومبنية على نفس الـ backend
> بس بطبقة تحقق أضعف. نسبة نجاح IDOR فيها أعلى بكتير.

---

## 🗺️ المخرج: خريطة الكائنات

كل الاستطلاع يصبّ في الجدول ده:

| الكائن | endpoint | Method | مكان الـ ID | نوعه | الأولوية | ملاحظة |
|---|---|---|---|---|---|---|
| user profile | `/api/v1/users/{id}` | GET, PATCH | path | UUID | 🔴 | PII كامل |
| invoice | `/api/v1/invoices/{id}` | GET | path | numeric | 🔴 | مالي، متسلسل! |
| team members | `/api/v1/teams/{id}/members` | GET | path | UUID | 🔴 | صلاحيات |
| file upload | `/api/v1/files/{id}/download` | GET | path | base64 | 🔴 | تسريب وثائق |
| api keys | `/api/v1/keys/{id}` | GET | path | UUID | 🔴 | Critical لو اتقرا |
| order | `/api/v1/orders/{id}` | GET, PATCH | path | numeric | 🟠 | |
| ticket | `/api/v1/tickets/{id}` | GET | query `?tid=` | hash | 🟡 | |

**الهدف**: 20-30 كائن. الجدول ده هو **قائمة العمل** بتاعتك في الأسبوع 10.

---

## 🔎 أماكن تسريب المعرّفات (UUID مش موجود؟ دور هنا)

لو المعرّفات UUID ومش قابلة للتخمين، لازم تلاقيها **متسربة**:

| المصدر | إزاي |
|---|---|
| ردود API تانية | `/api/teams/1/members` بيرجّع IDs الأعضاء |
| الروابط العامة | profile عام، روابط مشاركة |
| الـ JS files | UUIDs صلبة في الكود |
| الأرشيف (Wayback) | URLs قديمة فيها IDs |
| رسائل الإيميل | روابط التفعيل والدعوة |
| الـ Webhooks | الـ payload بيرجّع IDs |
| سجلات التدقيق / audit | في الـ UI بتاع الإداري (لو تقدر توصله) |
| الـ error messages | `User f47ac10b-... already exists` |
| الـ Search endpoints | `/api/search?q=ahmed` بيرجّع IDs |
| التصدير | `/api/export` بيرجّع كل المعرّفات 👈 |

> 💡 **أقوى تكتيك**: دور على endpoint بيرجّع **قائمة** (`GET /api/users`, `/api/teams/{id}/members`).
> القوايم دي غالبًا مش محمية زي التفاصيل، وبتديك كل المعرّفات دفعة واحدة.

---

## ⏱️ توزيع وقت الاستطلاع

| المرحلة | الوقت |
|---|---|
| Subdomains + live hosts | 15% |
| URLs + أرشيف | 15% |
| ملفات JS | **35%** 👈 |
| وثائق API | 15% |
| خريطة الكائنات | 20% |

---

## ⚠️ قواعد السلامة في الاستطلاع

- ❌ متعملش port scan أو brute force subdomains عنيف
- ❌ متعملش crawlers سريعة على الهدف
- ✅ استخدم مصادر سلبية (`subfinder`, `gau`, Wayback) قدر الإمكان
- ✅ `User-Agent` معقول + وتيرة هادية
- ❌ متعدّيش على الـ Scope أبدًا — النطاقات الفرعية المكتشفة بره الـ Scope = **ممنوعة**

---

## ✅ اختبر نفسك

1. ليه ملفات الـ JS أهم مصدر في الاستطلاع؟
2. UUID غير قابل للتخمين — منين تجيب IDs صالحة؟
3. إيه أكبر اختصار في الاستطلاع كله؟
4. إيه الـ 3 حاجات اللي الاستطلاع الموجه بيدور عليها؟
