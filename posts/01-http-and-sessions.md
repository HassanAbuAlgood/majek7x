# 1️⃣ HTTP والجلسات — الأساس اللي كل حاجة بتتبنى عليه

> **القاعدة**: لو مش قادر تقرأ request وتفهم كل بايت فيه، مش هتلاقي IDOR. مش مبالغة.

---

## 🧱 بنية الـ HTTP Request

```http
POST /api/v2/orders/8821/cancel HTTP/1.1          ← Method + Path + Version
Host: api.target.com                              ← الهدف
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...     ← مين أنت (الهوية)
Cookie: session=abc123; tenant_id=44             ← هوية تانية + بيانات
Content-Type: application/json                    ← شكل الجسم (مهم جدًا!)
Origin: https://app.target.com
Referer: https://app.target.com/orders
X-Forwarded-For: 1.2.3.4                          ← غالبًا بيُثق فيه غلط
Content-Length: 42

{"reason": "changed my mind"}                     ← الـ body
```

**كل سطر في الطلب ده = مكان ممكن تلاعب فيه.** دي طريقة التفكير اللي لازم تتبناها.

---

## 🔤 الـ Methods — وإيه اللي يهمنا

| Method | المعنى | ليه يهم في الـ IDOR |
|---|---|---|
| `GET` | قراءة | أشيع مكان للـ read IDOR |
| `POST` | إنشاء | إنشاء كائن في tenant تاني |
| `PUT` | استبدال كامل | تعديل بيانات مستخدم تاني |
| `PATCH` | تعديل جزئي | ← **مُهمَل أمنيًا في أغلب الأحيان** 👈 |
| `DELETE` | حذف | أخطر نوع IDOR |
| `OPTIONS` | الاستعلام عن المسموح | بيكشف الـ methods المتاحة |

> 💡 **نمط ذهبي**: لو `GET /api/users/123` رجّع 403، جرّب `PATCH` أو `PUT` على نفس المسار.
> كتير من الفرق بتحمي الـ GET وتنسى الباقي. المجال ده فيه findings حقيقية كتير.

---

## 📊 الـ Status Codes — لغة السيرفر

الكود بيقولك **إيه اللي بيحصل جوه السيرفر**. اتعلم تقراه كـ oracle:

| الكود | المعنى | تفسيره في سياق IDOR |
|---|---|---|
| `200 OK` | نجح | ✅ يرجّع بيانات = **IDOR محتمل** |
| `201 Created` | أُنشئ | جرب تنشئ كائن عند مستخدم تاني |
| `204 No Content` | نجح بلا محتوى | الحذف نجح = **IDOR delete** 👈 |
| `301/302` | تحويل | لو الـ redirect اختلف = فرق سلوكي |
| `400` | طلب غلط | الـ ID مش بالشكل المتوقع |
| `401` | **مفيش هوية** | مش معرّف |
| `403` | **عرفناك بس ممنوع** | 👈 **الكنز**: الـ endpoint موجود والكائن موجود، بس الفحص شغال |
| `404` | مش موجود | إما الـ ID مش موجود، أو **بيخفي الوجود** |
| `405` | الـ method مش مسموح | جرّب method تاني |
| `409` | تعارض | الكائن موجود بالفعل |
| `422` | فشل تحقق | شكل البيانات غلط |
| `429` | تجاوزت الحد | 👈 غياب الـ 429 في تخمين IDs = ثغرة بروحها |
| `500` | خطأ سيرفر | غالبًا الـ ID مش بالصيغة اللي الكود متوقعها |

### 🎯 أهم فكرة في الملف ده: الـ `403 vs 404` oracle

```
GET /api/users/999999  →  404 Not Found    (ID مش موجود)
GET /api/users/1234    →  403 Forbidden    (موجود بس ممنوع!)

الفرق ده بيقولك: الكائن 1234 موجود فعلًا وينتمي لشخص تاني.
دلوقتي دور على طريقة تدور بيها حوالين الـ 403.
```
ده بالظبط اللي هنتكلم عنه في [[03-idor-deepdive/04-bypass-playbook]].

---

## 🍪 الـ Cookies وإدارة الجلسة

```http
Set-Cookie: session=xyz; HttpOnly; Secure; SameSite=Lax; Path=/; Domain=.target.com
```

| الخاصية | الفايدة |
|---|---|
| `HttpOnly` | JS مش يقدر يقراها (حماية من XSS تسرق الجلسة) |
| `Secure` | HTTPS بس |
| `SameSite` | `Strict` / `Lax` / `None` — حماية CSRF |
| `Domain` / `Path` | نطاق الصلاحية |

**إدارة الجلسات في Burp** (هنحتاجها جدًا في حسابين):
- `Proxy → HTTP history` ← كل الطلبات
- كليك يمين على request → `Engagement tools → Set session handling rules`
- أو الأسهل: استخدم إضافة **Autorize** ([[03-idor-deepdive/03-detection-methodology]])

---

## 🪙 الـ Tokens: Session vs JWT

### Session Cookie
```
Cookie: session=a8f3c9...        → السيرفر بيدور عليه في الـ DB
```

### JWT (المنتشر جدًا في الـ APIs)
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo0MiwidGVuYW50IjoiYWNtZSJ9.H7k...
                     └─── header ──────────────┘ └────── payload ──────────────┘ └─ signature ─┘
```

فكّ الـ payload (base64) هتلاقي حاجة زي:
```json
{"user_id": 42, "tenant": "acme", "role": "user", "exp": 1735689600}
```

> 👀 **انتبه**: الـ `user_id` ده جوه التوكن! لو السيرفر بياخد الـ `user_id` من body أو query
> وبيتجاهل اللي جوه التوكن → **IDOR مضمون**.
> والأسوأ: لو السيرفر بياخد من التوكن بس، جرّب تغيّر الـ `user_id` جوه التوكن (محتاج توقيع صحيح — شوف أسبوع 2).

---

## 🌐 CORS و Same-Origin Policy (مفهوم سريع)

- **SOP**: المتصفح يمنع `evil.com` يقرأ رد `target.com`
- **CORS**: السيرفر يقدر يستثني origins معينة

```http
Access-Control-Allow-Origin: https://app.target.com
Access-Control-Allow-Credentials: true
```

لو لقيت `Access-Control-Allow-Origin: null` أو انعكاس لأي origin مع `Allow-Credentials: true`
→ تقدر تقرأ بيانات الضحية من موقعك. وغالبًا بيترافق مع read IDOR.

---

## 🛠️ إعداد بيئة العمل (اعملها النهارده)

1. **Burp Suite Community** — [تحميل](https://portswigger.net/burp/communitydownload)
2. **FoxyProxy** (إضافة متصفح) للتنقل بين proxy on/off
3. **متصفح منفصل للاختبار** (Firefox Developer Edition ممتاز) — متستخدمهوش في حاجاتك الشخصية
4. ثبّت شهادة Burp CA في المتصفح عشان الـ HTTPS
5. إضافات Burp من `Extensions → BApp Store`:
   - `Logger++` — تسجيل شامل
   - `Turbo Intruder` — تعداد سريع (استخدمه بحذر!)
   - `Param Miner` — يكتشف بارامترات مخفية
   - `JWT Editor` — للتلاعب بالتوكنز
   - `Wsdler` / `OpenAPI Parser` — لو فيه swagger

---

## ✅ اختبر نفسك

جاوب من غير ما تشوف فوق:
1. إيه الفرق بين 401 و 403؟ وإيه اللي بيهمك أكتر وليه؟
2. لو `GET /api/orders/55` رجّع 403 و `GET /api/orders/99999` رجّع 404، إيه الاستنتاج؟
3. إيه الـ `PATCH` وليه غالبًا أقل حماية من الـ `GET`؟
4. إزاي تقرأ محتوى JWT؟

**الإجابات في** [[06-assessment/quiz]] — بس جرب الأول.
