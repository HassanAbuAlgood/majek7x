# 🗺️ الخطة الكاملة — 12 أسبوع

**الهدف بنهاية الأسبوع 12**: تقرير IDOR واحد على الأقل مقبول (أو `Triaged`) على برنامج حقيقي.

---

## المرحلة 1️⃣: الأساس (أسابيع 1-4)
> الهدف: تبقى قادر تقرأ أي request/response وتفهم "مين المفروض يوصل لإيه".

### الأسبوع 1 — HTTP بعمق + إعداد بيئة العمل
**تذاكر**: [[02-foundations/01-http-and-sessions]]
- [ ] HTTP methods: `GET / POST / PUT / PATCH / DELETE / OPTIONS / HEAD` — إمتى بيستخدم كل واحد
- [ ] Status codes بحفظ: `200 201 204 301 302 400 401 403 404 405 409 422 429 500 502`
- [ ] Headers المهمة: `Authorization, Cookie, Set-Cookie, Origin, Referer, Content-Type, X-Forwarded-For, X-Original-URL`
- [ ] Cookies: `HttpOnly, Secure, SameSite, Domain, Path` + إيه الـ Session fixation
- [ ] الـ Same-Origin Policy و CORS (مفهوم مش تفاصيل)
- [ ] **ثبّت Burp Suite Community** + `FoxyProxy` + متصفح منفصل للاختبار

**عملي (4 ساعات)**:
- [ ] PortSwigger Academy → `HTTP` module (مفيش لابز كتير)
- [ ] حل 10 لابز من `Web Security Academy → Learning paths` السهلة
- [ ] شغّل Burp واعترض طلبات من موقع عادي (موقعك الشخصي أو `example.com`) واقرأ كل request بالتفصيل

**مخرج الأسبوع**: ملف ملاحظات فيه 20 request مختلفين فاهم كل حقل فيهم.

---

### الأسبوع 2 — المصادقة مقابل الصلاحيات (أهم أسبوع نظري)
**تذاكر**: [[02-foundations/02-authentication-vs-authorization]]
- [ ] **Authentication** = "إنت مين؟" vs **Authorization** = "إنت تقدر تعمل إيه؟" ← الفرق ده هو أساس الـ IDOR كله
- [ ] Session tokens vs JWT vs OAuth — إيه الفرق وكل واحد بيتبعت إزاي
- [ ] تشريح JWT: `header.payload.signature` واقرأ واحد على [jwt.io](https://jwt.io)
- [ ] RBAC / ABAC / ACL / Multi-tenancy — مصطلحات لازم تكون في دماغك
- [ ] **Horizontal** vs **Vertical** privilege escalation

**عملي (5 ساعات)**:
- [ ] PortSwigger → `Authentication` module — حل 8 لابز
- [ ] PortSwigger → `JWT` module — حل 5 لابز
- [ ] افتح تطبيق ويب بتاعك (أي SaaS بتستخدمه) وحلل الـ token بتاعه: نوعه إيه؟ فين بيتبعت؟

**مخرج الأسبوع**: تقدر تشرح الفرق بين authn و authz في 3 جمل، وتعرف تقرأ JWT.

---

### الأسبوع 3 — الـ APIs ومكان الـ Object IDs
**تذاكر**: [[02-foundations/03-apis-and-object-ids]]
- [ ] REST: الـ resource والـ endpoint وإزاي الـ URL بيتبنى
- [ ] GraphQL: `query / mutation / variables / introspection` — المفهوم بس
- [ ] **أين يختبئ الـ object ID؟** (الدرس الأهم في الأسبوع):
  - path: `/api/users/1337/orders`
  - query: `?user_id=1337`
  - body: `{"user_id": 1337}`
  - header: `X-User-Id: 1337`
  - cookie, JWT claim, GraphQL argument
- [ ] أنواع المعرّفات: numeric, UUID, base64, hash, composite (`tenant:user`)
- [ ] إيه الـ **Mass Assignment** (auto-binding) وليه بيحصل

**عملي (4 ساعات)**:
- [ ] PortSwigger → `API testing` module — حل كل اللابز
- [ ] ثبّت واجهة [crAPI](https://github.com/OWASP/crAPI) محليًا أو على نسخة أونلاين واستكشفها
- [ ] جمّع 10 API endpoints حقيقية من مواقع عامة ولاحظ شكل الـ IDs

**مخرج الأسبوع**: جدول بـ 10 endpoints + مكان الـ ID + نوعه.

---

### الأسبوع 4 — التحكم في الوصول + أول لابز IDOR فعلية
**تذاكر**: [[02-foundations/02-authentication-vs-authorization]] + [[03-idor-deepdive/01-idor-theory]]
- [ ] PortSwigger → `Access control` module بالكامل
- [ ] **حل كل لابز الـ IDOR** (فيه ~10) — دول أهم لابز في المسار كله
- [ ] افهم مصطلح `Unprotected functionality` و `Platform misconfiguration`

**عملي (6 ساعات)** — أكبر أسبوع عملي:
- [ ] كل الابز في `Access control` و `Access control vulnerabilities`
- [ ] ابدأ [OWASP Juice Shop](https://github.com/juice-shop/juice-shop) — حل تحديات الـ Broken Access Control

**مخرج الأسبوع**: حلّيت 15+ لاب access control من غير ما تشوف الحل.

---

## المرحلة 2️⃣: التعمق في IDOR (أسابيع 5-7)
> الهدف: تبقى "الشخص اللي بيلاقي IDOR في أي حاجة".

### الأسبوع 5 — تشريح الـ IDOR
**تذاكر**: [[03-idor-deepdive/01-idor-theory]]
- [ ] تعريف دقيق: `reference لكائن + غياب فحص صلاحية على مستوى الكائن`
- [ ] أنواع الـ IDOR: **Read** vs **Write** vs **Delete**
- [ ] **متسلسل**: `IDOR → تعديل إيميل → ATO` — إزاي توصلها لـ Critical
- [ ] الاقتصاديات: ليه الـ Write IDOR أغلى بكتير من الـ Read
- [ ] OWASP API Top 10: `API1:2023 BOLA` — المركز الأول

**عملي (5 ساعات)**:
- [ ] اقرأ **20 تقرير IDOR حقيقي** على [HackerOne Hacktivity](https://hackerone.com/hacktivity?query=idor) (مفلتر `disclosed`)
- [ ] لكل تقرير: اكتب في ملف `03-idor-deepdive/hacktivity-notes.md` → (الـ endpoint، نوع الـ ID، التأثير، المبلغ)

**مخرج الأسبوع**: 20 تقرير متحللين + نمط متكرر لاحظته بنفسك.

---

### الأسبوع 6 — الأنماط الـ 18 (الدرس اللي بيفرق)
**تذاكر**: [[03-idor-deepdive/02-patterns-and-variants]]

لازم تجرب **كل** دول على كل endpoint تشك فيه:
- [ ] تبديل الـ ID في path / query / body / header
- [ ] `HTTP method` switching: `GET → POST → PUT → PATCH → DELETE`
- [ ] `Content-Type` switching: `JSON → XML → form-data`
- [ ] Parameter wrapping: `{"id": 1}` ← `{"user": {"id": 1}}`
- [ ] Array pollution: `?id[]=1&id[]=2`
- [ ] Path traversal في الـ ID: `/api/users/../admin/config`
- [ ] Mass assignment: `{"role":"admin"}` في تحديث الملف الشخصي
- [ ] Encoding: `%2e%2e%2f`, double encoding, unicode
- [ ] Trailing slash / case sensitivity / `;`
- [ ] GraphQL: node IDs + aliases + batching
- [ ] **Second-order IDOR**: تنشئ كائن وبعدين توصله
- [ ] Recycled/deleted object references

**عملي (6 ساعات)**: حل لابز متقدمة + طبّق الـ 18 نمط على Juice Shop و crAPI.

---

### الأسبوع 7 — منهجية الكشف + التخطي
**تذاكر**: [[03-idor-deepdive/03-detection-methodology]] + [[03-idor-deepdive/04-bypass-playbook]]
- [ ] إعداد **حسابين** (A مهاجم / B ضحية) وإدارة الكوكيز في Burp
- [ ] تركيب واستخدام إضافة **Autorize** ← أهم أداة في المسار
- [ ] قراءة الفروق: `401 vs 403 vs 404` = oracle بيقولك فين تدور
- [ ] تخطي الـ 403: `X-Original-URL`, `X-Rewrite-URL`, `X-Forwarded-For`, `..;/`
- [ ] تخطي الـ rate limit على تخمين الـ IDs

**عملي (5 ساعات)**:
- [ ] جهّز بيئة حسابين على تطبيق تدريبي وشغّل Autorize واستخرج 3 findings
- [ ] تدرب على `Burp Comparer` لمقارنة الردود

---

## المرحلة 3️⃣: الصيد الحقيقي (أسابيع 8-12)
> الهدف: تقارير فعلية، مش تدريب.

### الأسبوع 8 — اختيار التارجت
**تذاكر**: [[04-hunting/01-target-selection]]
- [ ] نظام التقييم الرقمي (8 معايير)
- [ ] اختار **5 تارجتات** وسجلّهم في `03-progress-tracker.md`
- [ ] اقرأ `Scope` و `Rules of Engagement` لكل واحد بالتفصيل
- [ ] الفرق بين VDP و paid program، و HackerOne/Intigriti/Bugcrowd/YesWeHack

**مخرج الأسبوع**: 5 تارجتات مقيّمين، 2 منهم "ذهبي".

---

### الأسبوع 9 — الاستطلاع الموجه
**تذاكر**: [[04-hunting/02-recon-for-idor]]
- [ ] تجميع الـ subdomains + الـ JS files
- [ ] استخراج الـ endpoints والبارامترات من الـ JS (`linkfinder`, `getJS`)
- [ ] Wayback + `gau` للـ URLs القديمة
- [ ] البحث عن `swagger.json`, `/api-docs`, `/graphql`, Postman collections على GitHub
- [ ] لو فيه تطبيق موبايل: الـ APK = كنز (endpoints + IDs صلبة في الكود)
- [ ] ابنِ **خريطة الكائنات** للهدف: إيه الكائنات الموجودة؟ (order, invoice, ticket, file, team, API key, webhook…)

**عملي (6 ساعات)**: طبّق على التارجت الأول وطلّع خريطة كائنات كاملة.

---

### الأسبوع 10 — الصيد المنهجي
**تذاكر**: [[04-hunting/03-hunting-workflow]]
- [ ] الـ workflow الكامل بالترتيب (10 خطوات)
- [ ] اتبعها حرفيًا على التارجت الأول، وبعدين التاني
- [ ] سجّل **كل** محاولة حتى الفاشلة (الـ log ده أهم من أي حاجة)
- [ ] اكتب أول تقرير حقيقي حتى لو صغير

**مخرج الأسبوع**: أول تقرير مبعوت على برنامج حقيقي. 🎉

---

### الأسبوع 11 — كتابة التقارير (المهارة اللي بتحدد فلوسك)
**تذاكر**: [[04-hunting/04-report-template]]
- [ ] ليه التقارير بتترفض: `Duplicate, N/A, Informative, Out of scope`
- [ ] إزاي توصف الـ **Impact** مش بس الـ Technical
- [ ] الـ CVSS والـ Severity
- [ ] الـ PoC المثالي: خطوات مرقمة + صور + فيديو قصير اختياري
- [ ] اكتب 3 تقارير

**عملي**: راجع تقاريرك الأولى وحسّنها، وابعث 5 تقارير جديدة.

---

### الأسبوع 12 — التوسّع والدورة
- [ ] استهدف **20 تقرير مبعوت** تراكميًا (متوسط الصناعة: 1 مقبول من كل 10)
- [ ] حلّل إيه اللي اترفض وليه ← عدّل منهجيتك
- [ ] ابدأ تضيف ثغرة تانية (أنصحك بـ **Business Logic** أو **SSRF**) بنفس المنهجية
- [ ] اكتب write-up عن أول ثغرة مقبولة وانشره على مدونتك/X
- [ ] راجع [[06-assessment/quiz]] — لو جاوبت 80%+ يبقى الأساس تمام

---

## 📊 معايير النجاح (قيس نفسك بيها)

| بعد | المفروض تكون |
|---|---|
| أسبوع 4 | حلّيت 25+ لاب، تفهم أي request بالتفصيل |
| أسبوع 7 | تعرف تطبّق 18 نمط IDOR، عندك بيئة حسابين شغالة |
| أسبوع 8 | 5 تارجتات مقيّمين بنظام رقمي |
| أسبوع 10 | أول تقرير مبعوت |
| أسبوع 12 | 20 تقرير مبعوت، هدفك أول `Triaged` أو `Accepted` |

> ⚠️ **توقع واقعي**: أغلب الناس أول bounty ليهم بييجي بعد **3-6 شهور** من العمل المنهجي.
> اللي بيستسلم في شهر 2 هو اللي مابيوصلش. إحنا بنبني مهارة، مش بنجرب حظ.
