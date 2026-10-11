# 3️⃣ الـ APIs وأين يختبئ الـ Object ID

> **الدرس**: الـ IDOR محتاج حاجتين: (1) معرّف كائن، (2) غياب فحص الملكية.
> فأول مهارة: **تلاقي المعرّف في أي مكان بيختبئ فيه**.

---

## 🌐 REST vs GraphQL

### REST — النمط الكلاسيكي
```
GET    /api/v1/users/42              → اقرأ مستخدم
GET    /api/v1/users/42/orders       → اقرأ طلباته
POST   /api/v1/users/42/orders       → أنشئ طلب
PATCH  /api/v1/orders/8821           → عدّل الطلب
DELETE /api/v1/orders/8821           → احذف الطلب
```
كل `{42}` و `{8821}` = **مرشح IDOR**.

### GraphQL — endpoint واحد
```http
POST /graphql
{"query": "{ user(id: 42) { email, orders { id, total } } }"}
```
المعرّفات هنا في الـ `variables` أو جوه الـ query نفسه.

**مميزات للمهاجم**:
- `Introspection` مفعّل → يكشف الـ schema كله:
  ```graphql
  { __schema { types { name, fields { name } } } }
  ```
- **Aliases** تسمح بآلاف الطلبات في request واحد (تخطي rate limit):
  ```graphql
  { a: user(id:1){email} b: user(id:2){email} c: user(id:3){email} }
  ```
- **Batching** = نفس الفكرة لتعداد المعرّفات

> 🎯 نصيحة: لو الهدف فيه `/graphql`، ابدأ من هناك. نقطة دخول واحدة = كل الكائنات.

---

## 🔍 أين يختبئ الـ Object ID؟ (الخريطة الكاملة)

دي القايمة اللي هترجع لها طول حياتك:

| # | المكان | مثال | ملاحظات |
|---|---|---|---|
| 1 | **Path segment** | `/api/users/42/invoices` | الأوضح والأسهل |
| 2 | **Path (أخير)** | `/api/orders/8821` | أشيع مكان |
| 3 | **Query string** | `?user_id=42&account=99` | سهل الإفلات من الفحص |
| 4 | **JSON body** | `{"user_id": 42}` | 👈 **متجاهل كتير** |
| 5 | **Nested / wrapped** | `{"filter":{"user":{"id":42}}}` | التغليف بيفلت من الفلترة |
| 6 | **Array** | `ids[]=1&ids[]=2` أو `{"ids":[1,2]}` | استعلام جماعي |
| 7 | **Header** | `X-User-Id: 42` | نادر بس موجود |
| 8 | **Cookie** | `Cookie: uid=42` | 👈 غالبًا مش بيتفحص |
| 9 | **JWT claim** | `{"user_id":42}` جوه التوكن | محتاج تزوير توقيع |
| 10 | **GraphQL variable** | `{"variables":{"id":"42"}}` | |
| 11 | **Form data** | `user_id=42` بـ `Content-Type: form` | جرّب تحويل النوع |
| 12 | **XML** | `<userId>42</userId>` | جرّب `Content-Type: xml` |

> 💡 **التكتيك الذهبي**: لو الـ ID موجود في **مكانين** (مثلاً body الـ POST و الـ JWT)،
> غيّر **واحد بس** في كل مرة وشوف أيُّهما السيرفر بيصدّقه.
> لو صدّق اللي في الـ body وتجاهل التوكن → **IDOR مؤكد**.

---

## 🧬 أنواع المعرّفات — وإيه قابليتها للتخمين

| النوع | مثال | التخمين | ملاحظة |
|---|---|---|---|
| **Numeric متسلسل** | `1001, 1002, 1003` | ⭐⭐⭐⭐⭐ | أسهل حاجة. عدّد واستغل |
| **Numeric كبير** | `1712345678` | ⭐⭐⭐⭐ | timestamp أحيانًا |
| **UUID v4** | `f47ac10b-58cc-4372-a567-0e02b2c3d479` | ⭐ | مش قابل للتخمين — بس غالبًا **بيتسرب** |
| **UUID v1** | فيه timestamp + MAC | ⭐⭐⭐ | قابل للتخمين جزئيًا |
| **Base64** | `NDI=` = `42` | ⭐⭐⭐⭐ | فكّه ← غالبًا رقم أو UUID جوه |
| **Hash / Mongo ObjectId** | `507f1f77bcf86cd799439011` (24 hex) | ⭐⭐⭐ | ObjectId = timestamp + machine + pid + counter |
| **Composite** | `acme:42` أو `42-8821` | ⭐⭐⭐ | بدّل كل جزء لوحده |
| **Email / username** | `ahmed@corp.com` | ⭐⭐⭐⭐ | لو الـ API يقبله كمعرف |
| **Slug** | `acme-corp-2024` | ⭐⭐⭐ | أنماط متوقعة |

> 🔑 **الدرس الكبير**: المعرّف غير القابل للتخمين (UUID) **مش حماية**.
> لأنه هيتسرب في مكان تاني: رد API تاني، ملف JS، رابط دعوة، إيميل، webhook، سجل تدقيق،
> أو في الـ response بتاع endpoint تاني مافيهوش حماية.
>
> **وظيفة الاستطلاع = تدور على أماكن التسريب دي.** شوف [[04-hunting/02-recon-for-idor]].

---

## 🧨 Mass Assignment (Auto-binding)

مرتبط جدًا بالـ IDOR وغالبًا بيتلاقى معاه.

التطبيق بيربط كل حقل في الـ JSON بالكائن مباشرة:

```http
PATCH /api/users/42
{"name": "Ahmed", "email": "a@b.com", "role": "admin", "is_verified": true}
                                        ↑ ماكانش المفروض يكون في الفورم!
```

حقول تستهدفها:
`role`, `is_admin`, `permissions`, `is_verified`, `tenant_id`, `balance`, `credits`,
`price`, `discount`, `plan`, `owner_id`, `user_id`, `status`

> 🎯 **إزاي تعرف الحقول المخفية؟**
> - الـ `GET` على نفس الكائن غالبًا بيرجّع كل الحقول (بما فيها `role`) — خُدها من هناك
> - ملفات الـ JS فيها الـ models
> - GraphQL introspection
> - swagger/OpenAPI

---

## 🗂️ الكائنات اللي تستحق البحث عنها (قايمة ذهبية)

لما تستطلع هدف، دور على الكائنات دي — دي أكتر حاجة بتطلع IDOR:

| الفئة | أمثلة | ليه |
|---|---|---|
| 👤 الحساب | profile, settings, avatar, KYC docs | PII مباشر |
| 💳 الدفع | invoices, payment methods, transactions, refunds | مالي = High |
| 📦 الطلبات | orders, shipments, addresses | بيانات حساسة |
| 🏢 الفريق | teams, invites, members, roles | صلاحيات |
| 📄 الملفات | uploads, attachments, exports, reports | تسريب وثائق |
| 🔑 الأسرار | API keys, webhooks, tokens, secrets |Critical |
| 💬 المحتوى | tickets, messages, comments, drafts | |
| 📊 التحليلات | dashboards, logs, audit trails | بيانات تنظيمية |

**ترتيب القيمة**: API keys/secrets > مالي > PII > محتوى > تحليلات.

---

## 🧪 تمرين عملي (اعمله)

1. ثبّت [crAPI](https://github.com/OWASP/crAPI) أو استخدم نسخة مستضافة
2. سجّل حسابين (A و B)
3ـ بص في Burp على كل request وجمّع جدول:

| Endpoint | Method | مكان الـ ID | نوع الـ ID | الكائن |
|---|---|---|---|---|
| `/identity/api/v2/user/dashboard` | GET | JWT / header | UUID | profile |
| | | | | |

**الهدف**: 15 endpoint على الأقل. الجدول ده هو "خريطة الكائنات" اللي هتستخدمها في الصيد.

---

## ✅ اختبر نفسك

1. في كام مكان ممكن يختبئ فيه الـ object ID؟ اذكر 8.
2. UUID غير قابل للتخمين — يبقى آمن؟ ليه لا؟
3. إيه الـ Mass Assignment؟ وإزاي تعرف أسماء الحقول المخفية؟
4. إيه الـ MongoDB ObjectId وبيتكوّن من إيه؟
