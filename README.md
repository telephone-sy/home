# telephone-sy

سوق عربي مجاني لبيع وشراء الهواتف. التطبيق يخزن الإعلانات والمستخدمين في قاعدة بيانات الخادم؛ العملة لكل إعلان هي `SYP` أو `USD` وتُعرض كما أدخلها البائع، دون أسعار صرف أو تحويل.

## التشغيل المحلي

يتطلب Python 3.11 أو أحدث.

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
uvicorn main:app --reload
```

الجلسات المحلية تستخدم رموزاً عشوائية قابلة للإبطال محفوظة ببصمتها في قاعدة البيانات، ولا تحتاج مفتاح توقيع داخل `.env`.

افتح `http://127.0.0.1:8000`. توثيق Swagger متاح محلياً في `/docs`، وفحص التشغيل في `/healthz` والجاهزية وقاعدة البيانات في `/readyz`. يُنشئ التطبيق SQLite تلقائياً في التطوير؛ لتغيير القاعدة عيّن `DATABASE_URL`.

لإنشاء قاعدة تطوير جديدة قبل تشغيل الخادم، نفّذ `alembic upgrade head`، ثم أنشئ المشرف عبر `python -m services.create_admin`. إذا كانت قاعدة SQLite موجودة وأنشأها التطبيق سابقاً، شغّل التطبيق مرة ليتأكد من جداول التطوير ثم أنشئ المشرف؛ لا تشغّل الترحيل الأول عليها مباشرة.

## الاختبارات

```powershell
pytest -q
```

تستخدم الاختبارات قاعدة بيانات SQLite مؤقتة منفصلة.

## GitHub

يعمل GitHub Actions تلقائياً عند كل push وPull Request: يثبت الاعتمادات، يشغّل `pytest`، يتحقق من JavaScript، ويختبر Alembic على قاعدة مؤقتة. يحدّث Dependabot اعتمادات Python وGitHub Actions أسبوعياً.

قبل الرفع، تأكد أن `git status` لا يعرض `.env` أو `telephone_sy.db` أو صور المستخدمين. لا تضفها بـ`git add -f`. لربط هذا المجلد بمستودع GitHub جديد:

```powershell
git init -b main
git add .
git status --short
git commit -m "Prepare telephone-sy for GitHub"
git remote add origin https://github.com/USERNAME/REPOSITORY.git
git push -u origin main
```

استبدل `USERNAME/REPOSITORY` بعنوان المستودع الذي أنشأته. لا يوجد ملف ترخيص حالياً؛ اختر الترخيص قبل جعل المستودع عاماً لأن اختيار الترخيص قرار قانوني يخص مالك المشروع.

## النشر

### تقرير الانتقال إلى الإنتاج

1. **قاعدة البيانات:** أنشئ PostgreSQL مُداراً وخذ نسخة احتياطية، ثم اضبط `DATABASE_URL` بصيغة `postgresql+psycopg://...?...sslmode=verify-full`. يتطلب الإنتاج التحقق من شهادة TLS واسم المضيف. لقاعدة جديدة شغّل `alembic upgrade head` قبل التطبيق. إذا كنت تنقل قاعدة قديمة أُنشئت بـ`create_all`، خذ نسخة احتياطية وتحقق من كل الجداول والأعمدة أولاً؛ استخدم `alembic stamp head` فقط إذا كان المخطط مطابقاً تماماً، وإلا أنشئ ترحيلاً محافظاً للبيانات قبل التشغيل. لا تطبق الترحيل الأول عميانياً على قاعدة تحوي الجداول بالفعل.
2. **الإعدادات:** اضبط `ENVIRONMENT=production`، وعيّن أسماء النطاقات الدقيقة في `ALLOWED_HOSTS`، وعناوين PostgreSQL وRedis عبر أسرار المنصة. يتطلب الإنتاج `rediss://` لـ Redis، ويُرفض SQLite وSwagger العام؛ الكوكي `Secure` يعمل فقط مع HTTPS.
3. **الصور:** اربط `UPLOAD_DIR` بقرص دائم أو انقل `services/images.py` إلى تخزين كائنات دائم (مثل S3-compatible). القرص المؤقت قد يمحو الصور عند إعادة النشر؛ احتفظ بنسخ احتياطية وتأكد من حدود الحجم والتكلفة لدى المزود.
4. **التشغيل:** انشر تطبيق ASGI على خدمة تدعم Python وUvicorn، واجعل المنصة توفر HTTPS وتوجّه المنفذ إلى التطبيق. استخدم `/healthz` للفحص الحي و`/readyz` لفحص قاعدة البيانات وRedis؛ Swagger معطّل في الإنتاج. خطط المجانية تختلف وقد توقف التطبيق غير النشط أو لا توفر قرصاً دائماً.
5. **الإدارة:** أنشئ المشرف مرة واحدة عبر `alembic upgrade head` ثم `python -m services.create_admin` متصلاً بقاعدة الإنتاج، واستخدم كلمة مرور فريدة قوية. لا تمنح دور المشرف من التسجيل العام. لا يحتوي التطبيق حالياً على MFA للمشرف؛ فعّل طبقة MFA أمام لوحة الإدارة أو أضف MFA للتطبيق قبل تخزين بيانات حساسة.
6. **الحماية والتشغيل:** يستخدم الإنتاج Redis مشتركاً لتحديد المعدل؛ تحقق من TLS ومصادقة Redis وجدار شبكته، ولا تعرضه للإنترنت. اضبط Uvicorn ليثق بترويسات الوكيل العكسي من عناوينه فقط، ولا تثق بـ `X-Forwarded-For` عشوائياً. أضف CDN/WAF أمام الخدمة لتخفيف DDoS، ومراقبة وتنبيهات ونسخاً احتياطية مجرّبة، وحدّث الاعتمادات دورياً.
7. **البيانات القديمة:** لا يوجد استيراد تلقائي من `localStorage`. قبل استقبال مستخدمين حقيقيين، جهّز مستورداً لمرة واحدة يتحقق من كل سجل وملكيته وصوره، ولا تستورد بيانات العروض التجريبية.

هذه الخطوات لا تتطلب شراء نطاق أو استضافة أثناء التطوير المحلي، ولا توجد مدفوعات أو عمولات داخل التطبيق.

## واجهات أساسية

- `GET/POST /api/listings`, `GET/PATCH/DELETE /api/listings/{id}`
- `PATCH /api/listings/{id}/sold`, `POST /api/listings/{id}/images`
- `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me`
- `GET /api/users/me/listings`, `GET /api/favorites`, `POST/DELETE /api/favorites/{id}`
- `POST /api/reports`; واجهات الإدارة تحت `/api/admin` وتتطلب دور مشرف خادمياً.