# 2️⃣ Authentication مقابل Authorization

> **ده أهم ملف نظري في المسار كله.**
> الـ IDOR كله بيقع بالظبط في الخط الفاصل بين الكلمتين دول.

---

## 🔑 الفرق في سطرين

| | السؤال | الإجابة بتتحدد إمتى | لو اتكسرت |
|---|---|---|---|
| **Authentication** (AuthN) | إنت مين؟ | مرة واحدة وقت الدخول | حساب مخترق / ATO |
| **Authorization** (AuthZ) | تقدر تعمل إيه؟ | **مع كل طلب** | ← **IDOR / Privilege Escalation** |

### التشبيه اللي هيفضل في دماغك

```
Authentication = بطاقة الهوية بتاعتك عند البواب. بيشوفها مرة ويدخلك المبنى.

Authorization  = هل مسموح لك تفتح باب أوضة 305؟
                 البواب مش هيسألك تاني وأنت داخل... 
                 لو أوضة 305 مش بتاعتك ومحدش سألك → IDOR.
```

**السر**: معظم التطبيقات بتعمل AuthN صح (بوابة دخول قوية)،
وبتنسى تعمل AuthZ على **كل** endpoint. ← هنا بتلاقي الثغرة.

---

## 🏛️ نماذج الصلاحيات (الصلاحيات لغة لازم تعرفها)

| النموذج | الاسم | بيشتغل إزاي |
|---|---|---|
| **ACL** | Access Control List | كل كائن معاه قائمة "مين مسموح له" |
| **RBAC** | Role-Based | الأدوار: `user`, `moderator`, `admin` |
| **ABAC** | Attribute-Based | شروط: `if user.tenant == doc.tenant AND time < 17:00` |
| **ReBAC** | Relationship-Based | "صاحب المستند" / "عضو في الفريق" |

> 💡 أغلب ثغرات IDOR الحقيقية = كود بيفحص الـ **role** (RBAC) وبينسى يفحص
> الـ **ownership** (ReBAC/ABAC).
>
> يعني: `if user.role == 'user' → اسمح` ❌
> بدل: `if doc.owner_id == user.id → اسمح` ✅

---

## 🏢 الـ Multi-tenancy — أرض خصبة جدًا

في تطبيقات B2B: الشركة = tenant، والموظفين جوه الـ tenant.

```
Tenant A (شركة "ألفا")  → users: 1, 2, 3  → documents: 101, 102
Tenant B (شركة "بيتا")  → users: 4, 5    → documents: 201, 202
```

السؤال الذهبي: **هل `/api/documents/101` بيفحص إن اللي بيطلبه من Tenant A؟**

أغلب التطبيقات بتفحص `user.role` وبتنسى `user.tenant`.
الـ cross-tenant IDOR ده غالبًا بيُقيّم **Critical** لأنه بيسرّب بيانات شركة كاملة.

### 🔺 Horizontal vs Vertical

| النوع | المعنى | مثال | السيفيريتي |
|---|---|---|---|
| **Horizontal** | أوصل لنفس مستوايا بس مستخدم تاني | أقرأ فاتورة زميلي | Medium-High |
| **Vertical** | أصعد لمستوى أعلى | أعمل نفسي admin | High-Critical |

> 🎯 **أغلى finding ممكن تلاقيه**: Vertical + Write.
> مثال: `PATCH /api/users/{{victim_id}}` بـ `{"role":"admin"}` = **Critical** فوري.

---

## 🪪 الـ JWT بالتفصيل (مهم للـ IDOR)

### البنية
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9  .  eyJ1c2VyX2lkIjo0Mn0  .  dQw4w9WgXcQ
        header (alg, typ)                    payload (claims)        signature
```

### الـ Claims المهمة

| Claim | المعنى |
|---|---|
| `sub` | subject (معرّف المستخدم) |
| `iss` | المُصدر |
| `aud` | الجمهور المستهدف |
| `exp` / `iat` / `nbf` | صلاحية / إصدار / مش قبل |
| `jti` | معرّف التوكن الفريد |

### ثغرات JWT الكلاسيكية (مهمة كخطوة تسبق الـ IDOR)

1. **`alg: none`** — السيرفر يقبل توكن بدون توقيع
   ```json
   {"alg":"none","typ":"JWT"}
   ```
2. **عدم التحقق من التوقيع** — السيرفر مش بيتحقق خالص
3. **Key confusion (RS256 → HS256)** — يوقّع بالمفتاح العام كـ HMAC
4. **`kid` injection** — يتحكم في مسار المفتاح (`/dev/null`, Path traversal, SQLi)
5. **`jku` / `jwk`** — يحدد مفتاحه الخاص في التوكن
6. **`exp` مش بيتفحص** — توكن منتهي لسه شغال

> **الخلاصة للـ IDOR**: الهدف من تزوير JWT غالبًا مش "أبقى admin" —
> الهدف إنك تبقى **مستخدم تاني** وبعدين تستغل الـ IDOR كضحية حقيقية.
> JWT ضعيف + IDOR = سلسلة ATO.

---

## 🔐 OAuth / OIDC بسرعة

- **OAuth 2.0** = تفويض (إني أسمح لتطبيق يشوف بياناتي)
- **OIDC** = طبقة هوية فوق OAuth (بيعطيك `id_token`)

نقاط الضعف اللي بتهمنا في سياق الصلاحيات:
- `redirect_uri` مش متحقق منه → تسريب الـ code
- `state` مفقود → CSRF على ربط الحسابات
- الـ `scope` المطلوب أوسع من اللازم
- الـ `access_token` بيُخزّن في `localStorage` (XSS يسرقه)

---

## 🧪 الكشف عن غياب فحص الصلاحية — 3 أسئلة

لما تشوف أي endpoint فيه معرّف، اسأل:

1. **هل بيفحص الهوية أصلًا؟** → شيل الـ `Authorization` header خالص وشوف
2. **هل بيفحص الدور؟** → جرّب توكن لمستخدم عادي على endpoint إداري
3. **هل بيفحص الملكية؟** → بدّل الـ ID بكائن مستخدم تاني 👈 **ده الـ IDOR**

```http
# الاختبار الثلاثي على /api/documents/{id}
1. بلا Authorization header      → 200؟ = broken AuthN
2. بتوكن مستخدم عادي (role=user) → 200 على /admin/*؟ = broken RBAC
3. بتوكن A على doc بتاع B        → 200؟ = IDOR ✅
```

---

## ✅ اختبر نفسك

1. اشرح الفرق بين AuthN و AuthZ في جملتين.
2. إيه الفرق بين Horizontal و Vertical escalation؟ وأيهما أخطر؟
3. ليه الـ RBAC لوحده مش كافي لمنع IDOR؟
4. إزاي تستغل JWT ضعيف عشان توصل لـ IDOR؟

---

## 📚 واجب الأسبوع 2

- [ ] PortSwigger → `Authentication` module (8 لابز)
- [ ] PortSwigger → `JWT` module (5 لابز)
- [ ] اقرأ JWT بتاعك على [jwt.io](https://jwt.io) لحساب حقيقي بتستخدمه (متعملش حاجة، بص بس)
- [ ] اكتب في ملاحظاتك: إيه الفرق بين `role` و `ownership` بمثال من تطبيق بتستخدمه
