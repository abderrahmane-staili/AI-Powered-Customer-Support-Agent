[English](README.md) | **العربية**

# وكيل دعم العملاء بالذكاء الاصطناعي

وكيل دعم عملاء متعدد الوسائط على **Telegram**، مبني بـ **n8n** و**OpenAI** و**PostgreSQL مع pgvector**. يفهم الرسائل النصية والملاحظات الصوتية والصور، ويجيب انطلاقاً من مستنداتك الخاصة بتقنية RAG (التوليد المعزَّز بالاسترجاع)، ويتذكر سياق المحادثة.

## المميزات

- **بوت Telegram** يستقبل رسائل العملاء ويرد عليها تلقائياً
- **مدخلات متعددة الوسائط**: نص، وملاحظات صوتية (تُحوَّل إلى نص)، وصور (يتم تحليلها)
- **وكيل ذكي بذاكرة محادثة**، لتعمل الأسئلة المتتابعة بشكل طبيعي
- **RAG على مستنداتك** باستعمال PostgreSQL وpgvector للبحث الدلالي
- **إدخال تلقائي للمستندات**: ضع ملف PDF في Google Drive فيصبح جزءاً من قاعدة المعرفة
- **نموذج دردشة احتياطي** يضمن استمرار الإجابة إذا تعطل النموذج الرئيسي
- **قاعدة بيانات ذاتية الاستضافة** على Docker، تشتغل بأمر واحد

## البنية

يتكون المشروع من workflowين في n8n يتشاركان قاعدة بيانات متجهات واحدة.

1. **الإدخال (Ingestion)**: عند إضافة ملف PDF أو تعديله في Google Drive، يتم تنزيله واستخراج نصه وتقسيمه إلى أجزاء، ثم يُحوَّل كل جزء إلى متجه (embedding) بواسطة OpenAI ويُخزَّن في PostgreSQL.
2. **المحادثة (Chat)**: يكتب العميل للبوت على Telegram. النص يذهب مباشرة إلى الوكيل، والملاحظات الصوتية تُحوَّل إلى نص، والصور يتم وصفها. يبحث الوكيل في قاعدة المعرفة، ويستعمل الذاكرة ونموذج الدردشة، ثم يرسل الجواب على Telegram.

للمخطط التفصيلي راجع [النسخة الإنجليزية](README.md#architecture).

## التقنيات المستعملة

| الطبقة | التقنية |
|---|---|
| الأتمتة | n8n |
| المراسلة | Telegram Bot API |
| نماذج الذكاء الاصطناعي | OpenAI (دردشة، تحويل الصوت لنص، تحليل الصور، embeddings) |
| قاعدة البيانات | PostgreSQL 16 مع pgvector |
| مصدر المستندات | Google Drive |
| البنية التحتية | Docker وDocker Compose |

## هيكل المشروع

```
.
├── README.md
├── README.ar.md
├── docker-compose.yml        # حاوية PostgreSQL + pgvector
├── database/
│   └── setup.sql             # الإضافات وجدول documents
├── workflows/                # workflows مُصدَّرة من n8n (JSON)
│   └── README.md
├── docs/                     # لقطات شاشة ومخططات
├── .env.example              # انسخه إلى .env وضع كلمة السر
├── .gitignore
└── LICENSE
```

## المتطلبات

- Docker Desktop (أو Docker Engine) وDocker Compose
- n8n يشتغل محلياً على Docker
- توكن بوت Telegram من [@BotFather](https://t.me/BotFather)
- مفتاح OpenAI API
- حساب Google لمحفز Google Drive (اعتماد OAuth2 في n8n)

> n8n السحابي لا يستطيع الوصول إلى قاعدة بيانات تعمل على جهازك. استعمل n8n محلياً، أو PostgreSQL مستضافة تدعم pgvector، أو نفقاً آمناً. لا تفتح منفذ قاعدة البيانات للإنترنت مباشرة.

## البدء السريع

### 1. تشغيل قاعدة البيانات

```bash
git clone https://github.com/YOUR-USERNAME/ai-powered-customer-support-agent.git
cd ai-powered-customer-support-agent
cp .env.example .env        # في PowerShell: copy .env.example .env
```

افتح `.env` وضع كلمة سر قوية، ثم شغّل الحاوية:

```bash
docker compose up -d
```

الملف `database/setup.sql` يُنفَّذ تلقائياً عند أول تشغيل وينشئ إضافة `vector` وجدول `documents`. للتحقق:

```bash
docker exec -it DB.rag psql -U rag_user -d rag_db -c "\d documents"
```

يجب أن تظهر الأعمدة `id` و`content` و`metadata` و`embedding`.

### 2. ربط n8n بقاعدة البيانات

ضع حاوية n8n على نفس شبكة Docker التي فيها قاعدة البيانات (بدّل `n8n` باسم حاويتك من `docker ps`):

```bash
docker network connect rag-network n8n
```

في n8n أنشئ اعتماد **Postgres**:

| الحقل | القيمة |
|---|---|
| Host | `db-rag` |
| Port | `5432` |
| Database | `rag_db` |
| User | `rag_user` |
| Password | القيمة الموجودة في `.env` |
| SSL | Disable (للاستعمال المحلي) |

إذا لم يكن n8n يعمل على Docker، استعمل Host: `localhost` وPort: `5433`.

### 3. استيراد الـ workflows

في n8n اذهب إلى **Workflows ← Import from File** واستورد ملفات JSON الموجودة في مجلد `workflows/` (انظر [workflows/README.md](workflows/README.md)).

### 4. إنشاء الاعتمادات (Credentials)

| الاعتماد | يستعمله |
|---|---|
| Telegram API (توكن البوت) | محفز Telegram وعقد تنزيل الملفات والرد |
| OpenAI API | نماذج الدردشة، وتحويل الصوت، وتحليل الصور، والـ embeddings |
| Google Drive OAuth2 | محفز Drive وتنزيل الملفات |
| Postgres | عقدتا مخزن المتجهات |

### 5. إعداد عقد مخزن المتجهات

في عقدتي PGVector كلتيهما (الإدخال والبحث):

- **Table Name**: `documents`
- **Options ← Column Names**: ضع **Content Column** على `content` (القيمة الافتراضية في n8n هي `text`)
- يجب أن يكون **نموذج الـ embeddings نفسه** في الإدخال وفي البحث

### 6. الاختبار

1. أضف ملف PDF إلى مجلد Google Drive المراقَب وشغّل workflow الإدخال.
2. تأكد من تخزين الصفوف:
   ```bash
   docker exec -it DB.rag psql -U rag_user -d rag_db -c "SELECT id, LEFT(content, 80) AS preview FROM documents LIMIT 5;"
   ```
3. فعّل workflow المحادثة وأرسل رسالة إلى بوت Telegram.
4. اسأل عن شيء موجود في الـ PDF، ثم عن شيء غير موجود. يجب أن يجيب الوكيل عن الأول من المستندات، ويعترف بأنه لا يعرف الثاني.

## أوامر يومية

```bash
docker compose up -d          # تشغيل قاعدة البيانات
docker compose stop           # إيقافها (البيانات تبقى)
docker compose logs -f        # متابعة السجلات
docker exec -it DB.rag psql -U rag_user -d rag_db   # فتح psql
```

نسخ احتياطي واستعادة:

```bash
docker exec DB.rag pg_dump -U rag_user -d rag_db > rag_backup.sql
cat rag_backup.sql | docker exec -i DB.rag psql -U rag_user -d rag_db
```

## حل المشاكل

| المشكلة | الحل |
|---|---|
| `extension "vector" is not available` | تستعمل صورة `postgres` عادية. استعمل `pgvector/pgvector:pg16` كما في `docker-compose.yml` |
| `database "db-rag" does not exist` في n8n | كتبت `db-rag` في خانة **Database**. مكانها خانة **Host**، وقاعدة البيانات هي `rag_db` |
| n8n لا يتصل | تأكد أن n8n على الشبكة `rag-network`، أو استعمل `host.docker.internal` / `localhost` مع المنفذ `5433` |
| `column "text" does not exist` | ضع Content Column على `content` في عقدة PGVector |
| خطأ عدم تطابق الأبعاد | نموذج الـ embeddings لا يُخرج 1536 بعداً. غيّر `VECTOR(1536)` في `database/setup.sql` أو غيّر النموذج |
| الوكيل لا يستعمل المستندات | نموذجا الـ embeddings مختلفان بين الإدخال والبحث، أو وصف الأداة ناقص |

## الأمان

- لا ترفع أبداً `.env` أو مفاتيح API أو توكنات Telegram أو كلمات السر أو الاعتمادات.
- ملفات تصدير الـ workflows لا تحتوي أسرار الاعتمادات، لكن راجع كل ملف JSON قبل الرفع (انظر `workflows/README.md`).
- لا تعرّض منفذ PostgreSQL للإنترنت.
- غيّر أي مفتاح أو كلمة سر ظهرت في لقطة شاشة أو محادثة.

## الرخصة

المشروع مرخص بموجب [رخصة MIT](LICENSE).
