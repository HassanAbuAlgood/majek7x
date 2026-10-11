# 1️⃣3️⃣ المعامل والمصادر

> **القاعدة**: متدفعش فلوس في كورسات في الأول. كل اللي تحت ده **مجاني** وكافي تمامًا.

---

## 🏆 المصدر الأول: PortSwigger Web Security Academy

[portswigger.net/web-security](https://portswigger.net/web-security) — **أفضل مصدر مجاني في العالم للمبتدئين.**

### المسار المطلوب بالترتيب

| الترتيب | الـ Module | عدد اللابز | الأسبوع |
|---|---|---|---|
| 1 | `HTTP` / `Learning paths` | قليل | 1 |
| 2 | `Authentication` | ~12 | 2 |
| 3 | `JWT` | ~8 | 2 |
| 4 | `API testing` | ~6 | 3 |
| 5 | **`Access control`** 👈 | ~18 | **4** |
| 6 | `Business logic vulnerabilities` | ~10 | اختياري |
| 7 | `CORS` | ~4 | اختياري |
| 8 | `Server-side request forgery` | ~8 | أسابيع 12+ |

> 🎯 **لابز الـ Access control هي أهم حاجة في المسار كله.**
> حلّها كلها، وبعدين حلّها تاني بعد أسبوعين.

---

## 🧪 بيئات التدريب

| البيئة | الرابط | التركيز | الصعوبة |
|---|---|---|---|
| **OWASP Juice Shop** | [github.com/juice-shop/juice-shop](https://github.com/juice-shop/juice-shop) | كل حاجة، فيه 100+ تحدي | ⭐⭐ |
| **crAPI** | [github.com/OWASP/crAPI](https://github.com/OWASP/crAPI) | ⭐ **مصمم لثغرات API** — الأهم ليك | ⭐⭐ |
| **vAPI** | [github.com/roottusk/vapi](https://github.com/roottusk/vapi) | OWASP API Top 10 حرفيًا | ⭐⭐ |
| **DVWA** | [github.com/digininja/DVWA](https://github.com/digininja/DVWA) | أساسيات جدًا | ⭐ |
| **PortSwigger labs** | أونلاين | منهجي ومشرح | ⭐-⭐⭐⭐ |
| **HackTheBox Academy** | ` academy.hackthebox.com` | `Bug Bounty Hunting` + `Broken Access Control` | ⭐⭐⭐ |
| **PentesterLab** | `pentesterlab.com` | مجاني جزئيًا | ⭐⭐ |
| **BugBountyReports** | — | تقارير حقيقية | ⭐⭐⭐⭐ |

### تشغيل سريع
```bash
# Juice Shop
docker run --rm -p 3000:3000 bkimminich/juice-shop

# crAPI
git clone https://github.com/OWASP/crAPI && cd crAPI
docker-compose up -d
```

---

## 📚 مصادر التعلم المجانية

### أساسية (متخليش عنها)
| المصدر | المحتوى |
|---|---|
| **PortSwigger Academy** | المنهج الكامل |
| **OWASP API Security Top 10** | [owasp.org/API-Security](https://owasp.org/www-project-api-security/) — اقرا `API1:2023 BOLA` و `API5:2023 BFLA` |
| **HackerOne Hacktivity** | تقارير حقيقية disclosed — [hackerone.com/hacktivity](https://hackerone.com/hacktivity) |
| **BugBountyNotes** | [bugbountynotes.com](https://bugbountynotes.com) |
| **PayloadsAllTheThings** | [github.com/swisskyrepo/PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) |
| **HackerOne University** | `university.hackerone.com` — مجاني |
| **OWASP Web Security Testing Guide** | مرجع شامل |

### YouTube / قنوات عربية ودولية
- `STÖK` — Bug Bounty عملي
- `Nahamsec` — منهجية وrecon
- `InsiderPHD` — للمبتدئين جدًا
- `PwnFunction` — شرح تقني عميق
- `LiveOverflow` — تفكير أمني

### كتب (لما تتقدم)
- `Web Application Hacker's Handbook` — المرجع الكلاسيكي
- `Bug Bounty Bootcamp` (Vickie Li) — ⭐ ممتاز للمبتدئين
- `API Security in Action` — للتعمق في الـ APIs

---

## 🧭 تحدي تدريبي منظم (اعمله في أسابيع 5-7)

### التحدي 1: Juice Shop — Broken Access Control
- [ ] ثبّتها وسجّل حسابين
- [ ] حل كل تحديات `Broken Access Control`
- [ ] اكتب تقرير لكل واحدة بالقالب بتاعنا

### التحدي 2: crAPI — OWASP API
- [ ] شغّلها محليًا
- [ ] حل كل التحديات (فيه BOLA, BFLA, mass assignment, JWT)
- [ ] الهدف: 8 findings على الأقل

### التحدي 3: قراءة 20 تقرير حقيقي
- [ ] [hackerone.com/hacktivity?query=idor](https://hackerone.com/hacktivity?query=idor&filter=disclosed)
- [ ] لكل تقرير: الـ endpoint، المعرّف، التأثير، المبلغ
- [ ] المخرج: ملف `hacktivity-notes.md`

### التحدي 4: محاكاة صيد كامل
- [ ] اختبر هدف تدريبي بالـ workflow الكامل (استطلاع → خريطة كائنات → اختبار → تقرير)
- [ ] المخرج: تقرير احترافي كامل

---

## 📅 خطة المعامل أسبوعيًا

| الأسبوع | اللابز |
|---|---|
| 1 | HTTP basics (5) |
| 2 | Authentication (8) + JWT (5) |
| 3 | API testing (6) + تثبيت crAPI |
| 4 | **Access control (18)** 👈 |
| 5 | قراءة 20 تقرير + Juice Shop BAC |
| 6 | crAPI بالكامل + تطبيق الأنماط الـ 16 |
| 7 | Autorize + سيناريو الحسابين |
| 8+ | صيد حقيقي + HackTheBox Academy |

---

## ⚠️ نصايح التعلم

1. **متشوفش الحل قبل 20 دقيقة تفكير** ([[01-roadmap/02-weekly-schedule]])
2. **كرر اللاب بعد أسبوع** — التكرار بيثبّت الفهم
3. **اكتب بأسلوبك** — النسخ مابيعلمش
4. **متجمعش مصادر وتتفرج بس** — "جمع المصادر" فخ بيوهمك إنك بتتعلم
5. **طبّق فورًا** — اقرا درس → جرّبه في نفس الساعة

> 🎓 **أهم نصيحة**: شخص واحد حلّ 18 لاب Access control وطبّقها على 5 تارجتات،
> أحسن بكتير من شخص قرا 20 كتاب وماجربش.

---

## 🔗 روابط سريعة

```
PortSwigger Academy    https://portswigger.net/web-security
OWASP API Top 10       https://owasp.org/www-project-api-security/
HackerOne Hacktivity   https://hackerone.com/hacktivity
Intigriti Programs     https://app.intigriti.com/researcher/programs
Bugcrowd               https://bugcrowd.com/programs
PayloadsAllTheThings   https://github.com/swisskyrepo/PayloadsAllTheThings
crAPI                  https://github.com/OWASP/crAPI
Juice Shop             https://github.com/juice-shop/juice-shop
JWT.io                 https://jwt.io
```
