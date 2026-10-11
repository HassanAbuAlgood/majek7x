# 1️⃣2️⃣ قالب التقرير — المهارة اللي بتحدد فلوسك

> **حقيقة مرة**: ثغرة قوية + تقرير ضعيف = `Informative`.
> ثغرة متوسطة + تقرير ممتاز = `Accepted` بمبلغ محترم.
> **كتابة التقرير نص المهنة.**

---

## 📐 البنية الذهبية

```
1. عنوان دقيق (محدد، مش عام)
2. ملخص في سطرين
3. التصنيف والسيفيريتي (مع سبب CVSS)
4. الخطوات (مرقمة، قابلة للتكرار في 5 دقايق)
5. الـ PoC (request/response خام + صور)
6. التأثير (Impact) — بالأرقام، مش بالكلام العام
7. الإصلاح المقترح
```

---

## 📝 القالب (بالإنجليزية — لغة المنصات)

```markdown
# Title: IDOR on /api/v1/invoices/{id} allows any authenticated user
# to read all customers' invoices (PII + financial data)

## Summary
The endpoint `/api/v1/invoices/{id}` does not verify that the requested
invoice belongs to the authenticated user. Any authenticated user can
enumerate the sequential `invoice_id` and retrieve invoices belonging to
any other customer.

## Severity
**High (CVSS 3.1: 7.5)** — AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N

Rationale: Requires only low-privilege authentication, no user interaction,
and results in complete disclosure of confidential data (H).

## Affected Asset
`https://api.target.com/v1/invoices/{id}` (in scope per program policy)

## Steps to Reproduce

1. Create two accounts:
   - Attacker: `attacker+a@example.com`
   - Victim:   `attacker+b@example.com`
2. Log in as the **victim** and create an invoice.
   Note the returned `invoice_id` (e.g. `5001`).
3. Log in as the **attacker** and send the following request:

```http
GET /v1/invoices/5001 HTTP/1.1
Host: api.target.com
Authorization: Bearer <ATTACKER_TOKEN>
```

4. Observe the server returns **HTTP 200** with the victim's full invoice:

```json
{
  "invoice_id": 5001,
  "client_name": "Acme Corporation",
  "client_email": "billing@acme.com",
  "amount": 12400.00,
  "bank_details": "XXXX",
  "status": "paid"
}
```

5. Incrementing/decrementing the ID (`5000`, `5002`, `4999` …) returns
   invoices belonging to other customers. No rate limiting was observed
   across 100 sequential requests.

> Note: Only invoice `5001` (my own test account) was accessed for
> verification. No additional customer data was stored or exfiltrated.

## Proof of Concept
[ attach: screenshot of the request/response in Burp ]
[ attach: raw request/response as .txt ]
[ optional: short screen recording ]

## Impact
Any authenticated user — including a free-tier trial account — can:

- Enumerate **all invoices on the platform** (~ estimated 50,000 customers)
- Access **PII**: client names, billing emails, billing addresses
- Access **financial data**: amounts, payment status, bank details

This constitutes a **complete confidentiality breach** of billing data
across all tenants and likely violates GDPR obligations.

## Remediation
Perform an ownership check on every request, server-side:

```python
# ❌ Current
invoice = Invoice.objects.get(id=request.invoice_id)

# ✅ Recommended
invoice = Invoice.objects.get(id=request.invoice_id,
                              owner_id=request.user.id,
                              tenant_id=request.user.tenant_id)
if not invoice:
    return 404  # avoid existence oracles
```

Additionally:
- Use non-sequential identifiers (UUIDv4) — *in addition to*, not instead of, ownership checks
- Apply consistent authorization middleware across all HTTP methods
- Add rate limiting on object-access endpoints
```

---

## 🎯 إزاي تكتب العنوان

| ❌ ضعيف | ✅ قوي |
|---|---|
| "IDOR vulnerability" | "IDOR on `/api/v1/invoices/{id}` allows any authenticated user to read all customers' billing data" |
| "Broken access control" | "Broken access control on `PATCH /api/v1/users/{id}` allows changing any user's email → account takeover" |
| "Security issue found" | "Unauthenticated IDOR on `/api/v1/files/{id}/download` exposes all uploaded documents" |

**العنوان الممتاز = المسار + نوع الثغرة + من يقدر يعملها + النتيجة.**

---

## 📊 حساب الـ Severity (CVSS 3.1 مبسط)

| المكون | القيمة للـ IDOR النموذجي |
|---|---|
| `AV` Attack Vector | `N` (Network) |
| `AC` Attack Complexity | `L` (Low) |
| `PR` Privileges Required | `L` (Low — حساب عادي) أو `N` (None — بلا دخول) |
| `UI` User Interaction | `N` (None) |
| `S` Scope | `U` (Unchanged) |
| `C` Confidentiality | `H` لو PII/مالي، `L` لو بيانات عادية |
| `I` Integrity | `H` لو Write IDOR، `N` لو Read |
| `A` Availability | `H` لو Delete، وإلا `N` |

### تقديرات سريعة

| السيناريو | التقييم |
|---|---|
| Read IDOR لبيانات غير حساسة | 🟡 Medium (~5.3) |
| Read IDOR لـ PII | 🟠 High (~7.5) |
| Write IDOR | 🟠 High (~8.1) |
| Delete IDOR | 🟠 High (~8.1) |
| IDOR على API keys/secrets | 🔴 Critical (~9.1) |
| IDOR → ATO | 🔴 Critical (~9.6) |

---

## ❌ ليه التقارير بتترفض؟ (وكيف تتجنبه)

| حالة الرفض | السبب | الحل |
|---|---|---|
| **Duplicate** | حد سبقك | طبيعي جدًا. كمل. راجع الـ Hacktivity قبل البعت |
| **Informative** | مفيش تأثير واضح | اربط الثغرة بنتيجة عملية (تسريب بيانات حقيقية، ATO…) |
| **N/A (Not Applicable)** | مش بيقبلوا النوع ده | اقرأ الـ program policy كويس |
| **Out of scope** | اختبرت بره النطاق | راجع الـ Scope حرفيًا قبل كل تقرير |
| **Cannot reproduce** | الـ PoC ناقص | خطوات مرقمة + كامل الـ requests + حسابات تجريبية |
| **Won't fix (accepted risk)** | قرار تجاري | مقبول، خُدها خبرة |

> 💡 **أهم سبب للرفض في الـ IDOR**: إنك تثبت **الوصول** بس وما تثبتش **التأثير**.
> دايمًا اكتب: "مين يقدر يعمل ده؟ وكم مستخدم يتأثر؟ وإيه نوع البيانات؟"

---

## ✨ 7 حاجات بتخلي التقرير يقبل

1. ✅ **قابل للتكرار في 5 دقايق** — الـ triager وقته محدود
2. ✅ **حسابان واضحان** (A و B) — مش كلام عام
3. ✅ **request/response خام** مش بس صورة
4. ✅ **تأثير بالأرقام** ("كل الـ 50,000 عميل" أقوى من "بعض المستخدمين")
5. ✅ **صورة واحدة واضحة** أحسن من 5 صور مشوشة
6. ✅ **اقتراح إصلاح** — بيوريك محترف مش مخرب
7. ✅ **احترام**: مفيش بيانات مسربة أكتر من اللازم، ومفيش استعجال

---

## 📧 التعامل مع الـ Triage

| الموقف | التصرف |
|---|---|
| سؤال من الـ triager | رد خلال 24 ساعة بوضوح |
| طلب معلومات إضافية | ابعتها فورًا |
| خفضوا السيفيريتي | ناقش بأدلة، مش بعاطفة |
| سكوت طويل (&gt; 30 يوم) | استفسر مرة واحدة باحترام |
| رفض | اسأل عن السبب بتعلم، ومتجادلش |

**متعملش**: رسايل متكررة، ضغط، تهديد بالنشر. دي بتدمر سمعتك للأبد.

---

## 🏁 بعد أول تقرير مقبول

1. اطلب إذن النشر (`Disclosure`)
2. اكتب write-up على مدونتك أو X
3. حلّل إيه اللي خلاه يتقبل وكرره
4. كرر نفس المنهجية على نفس الهدف — غالبًا فيه أكتر

---

## ✅ اختبر نفسك

1. إيه الفرق بين تقرير `Informative` و `Accepted` بنفس الثغرة؟
2. اكتب عنوان لتقرير IDOR على `/api/teams/{id}/members`.
3. ليه "عدد المستخدمين المتأثرين" مهم في التقييم؟
4. إيه أول حاجة تعملها لو الـ triager قال `Cannot reproduce`؟
