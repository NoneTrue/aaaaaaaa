# الخطة التنفيذية الكاملة لمنصة AI Manager

> الحالة المرجعية: تدقيق المستودع `D:\NEW` في 2026-08-12، وتحديث التدقيق في 2026-08-13. المصدر المعماري الرسمي الوحيد المستخدم هو `docs/AI_MANAGER_MASTER_ARCHITECTURE.md` (الإصدار المرجعي 1.0، 3917 سطرًا، 44 قسمًا). هذه خطة تنفيذ فقط؛ لم يُعدّل الكود أو قاعدة البيانات أو الاعتماديات أثناء إعدادها.

## 1. الملخص التنفيذي

وصل المشروع إلى `Backbone MVP v0.1` عامل محليًا: يستقبل ملفات وطلبًا عربيًا، ينشئ نية وخطة من مهمتين، يتحقق من الخطة، ينفذ أدوات `Mock` متسلسلة، يحفظ الحالة والخبرة في SQLite، ويعرض النتيجة في مساحة عمل React عربية. أكد الفحص وجود API والتخزين والأدوات الأربعة وواجهات الإلغاء والإعدادات، كما أن فحص TypeScript الحالي نجح. قاعدة البيانات التشغيلية تحتوي بالفعل على طلبات مكتملة وتجارب محفوظة.

ما زال العمود الفقري تجريبيًا: الصورة والصوت والبيانات والتقرير كلها `Mock`، و`OllamaLLMProvider` محول أولي غير مثبت باختبارات تكامل. كما أن العقود موزعة داخل ملفات التنفيذ، وتُخزّن الحالات كنصوص بلا آلة انتقالات، ولا توجد مهاجرات، أو `Plugin Manifest — ملف تعريف الإضافة`، أو `Request Gateway — بوابة الطلبات` مستقلة، أو أحداث/تقدم/استكمال.

هدف التطوير التالي ليس إعادة البناء، بل تثبيت العقود والحالة والاختبارات أولًا، ثم استبدال `image.compare` بأداة صور محلية حقيقية خلف المعرف والعقد نفسيهما. التقنية المقترحة لأول تكامل هي `OpenCV — مكتبة الرؤية الحاسوبية` مع `Pillow` لقراءة/كتابة الصور: فحص النوع والأبعاد، مواءمة اختيارية، فرق بكسلي وحواف/مكوّنات، درجة تشابه معلنة المنهج، وصورة فرق مشتقة محفوظة كأصل جديد. يبقى `Mock` للاختبارات الحتمية.

الطريق الواقعي هو: تثبيت `v0.1` → صورة حقيقية `v0.2` → نموذج محلي منظم `v0.3` → صوت وبيانات حقيقيان `v0.4` → سير متعدد البرامج `v0.5` → معيار تكامل وSDK `v0.6` → تقوية `v0.8` → إصدار محلي ثابت `v1.0` → جاهزية الخادم و`User AI` لاحقًا.

## 2. خط الأساس الحالي

| المكون | الحالة الفعلية | الدليل من الكود | النقص الحالي |
| --- | --- | --- | --- |
| Backend | يعمل بـFastAPI | `backend/app/main.py` | تطبيق أحادي الحزمة؛ لا منافذ تخزين/Workflow مستقلة |
| إصدار Python | التنفيذ الفعلي عبر `uv` استخدم Python 3.13، لكن `pyproject.toml` يسمح `>=3.12` والتقرير يذكر 3.12.10 | `backend/pyproject.toml`، `BUILD_REPORT.md`، سجل `uv sync` | المصدر الرسمي يعتمد Python 3.13 baseline مع اختبار 3.14؛ يجب توحيد المتطلب والتقرير وCI دون ترقية عمياء |
| Frontend | مساحة عمل React RTL عاملة | `frontend/src/main.tsx`، `frontend/src/styles.css` | مكوّن واحد كبير، Polling فقط، لا عرض Artifacts أو تقدم دقيق |
| دورة الطلب | شريحة رأسية مكتملة للـMock | `backend/app/services/orchestrator.py:run` | انتقالات نصية مباشرة، لا إعادة تخطيط أو استكمال أو سجل أحداث |
| نماذج البيانات | 7 جداول موجودة | `backend/app/models/entities.py`، فحص SQLite | لا `intents/providers/tools/artifacts/events` مستقلة ولا Migration |
| حالات الطلب/المهمة | حالات MVP مستخدمة | `orchestrator.py`، `executor.py` | لا Enum/قواعد انتقال مركزية؛ أسماء الكود (`VALIDATING_PLAN`) لا تطابق العقد الرسمي (`PLAN_VALIDATION`) وتغيب حالات مثل `VALIDATING/INGESTING_ASSETS/AWAITING_APPROVAL/REPLANNING` و`BLOCKED/QUEUED/WAITING_EXTERNAL/RETRYING/TIMED_OUT/SKIPPED`؛ `READY/PENDING` لا تُحدّث حسب DAG بصورة عامة |
| Asset Manager | حفظ آمن نسبيًا وبصمة SHA-256 | `backend/app/services/asset_manager.py` | يقرأ الملف كاملًا للذاكرة، يثق بـContent-Type، لا أصل مشتق/سلسلة نسب/سياسة احتفاظ |
| Intent Analyzer | غلاف رقيق لمزود النموذج | `backend/app/services/intent_analyzer.py` | يعيد `dict` غير متحقق بعقد Pydantic |
| LLM Providers | Fake وOllama محلي أولي | `backend/app/llm/` | العقد مخصص لثلاث عمليات لا `generate_structured` عام؛ لا تحقق Schema أو fallback تشغيلي فعلي |
| Planner | ينادي المزود | `backend/app/services/planner.py` | الخطة تربط `tool_id` مباشرة بدل قدرات مجردة؛ Fake ثابت من مهمتين |
| Plan Validator | يفحص العدد/المعرفات/DAG/التسجيل | `backend/app/services/plan_validator.py` | لا توافق أنواع أو ميزانية زمن/موارد أو سياسة/هدف |
| Tool Registry | سجل ذاكرة يمنع المجهول | `backend/app/tools/registry.py` | لا إصدارات Schema/Capabilities/Health أو اختيار مزود متعدد؛ جميع الأدوات المضمنة مسجلة `enabled=True` بدل قاعدة `Disabled or Restricted` للأداة الجديدة |
| Plugin Loader | يسجل أربع أدوات مضمنة | `backend/app/tools/plugin_loader.py` | ليس Loader لملفات Manifest؛ مجلدا الجذر `plugins/` و`contracts/` غير موجودين |
| Tool Router | Mapping ثابت حسب المجال | `backend/app/services/tool_router.py` | لا يستخدم النية/نوع الملف/الصحة/Offline فعليًا؛ ناتجه لا يربط الخطة |
| Policy Engine | يسمح فقط بخطر `low` | `backend/app/services/policy_engine.py` | Profile غير مؤثر؛ لا موافقات أو حدود اتصال/موارد |
| Executor | تسلسلي مع مهلة 10 ثوانٍ | `backend/app/services/executor.py` | Payload مبني حسب أسماء أدوات محددة؛ لا Topological scheduler عام أو تقدم/Retry/Artifacts |
| Verifier | يفحص عدد المخرجات و`mock: true` | `backend/app/services/verifier.py` | يمنع نتائج حقيقية عمليًا؛ لا تحقق مجال/ثقة/مخرجات مطلوبة |
| Result Composer | حزمة نتيجة ومصدر مبسط | `backend/app/services/result_composer.py` | يفترض أن كل نتيجة Mock، ولا يدعم Artifacts أو عدم اليقين أو ToolCall IDs |
| Experience Store | يسجل Episode أساسيًا | `backend/app/services/experience_store.py` | لا تقييم/تعليقات/Recovery/Provenance كامل؛ كتابة مباشرة عبر Session |
| Mock Plugins | صورة وصوت وبيانات وتقرير | `backend/app/tools/plugins/mock_*` | ليست Plugins مستقلة؛ العقود والتنفيذ في `__init__.py` |
| API | 10 عمليات مثبتة من OpenAPI | `backend/app/api/` | لا Content للأصل، Providers/Health، Plans/Tasks، Resume، Feedback أو أخطاء معيارية |
| Offline Mode | إعداد محفوظ واختيار Fake عند تفعيله | `provider_factory.py`، `api/system.py` | عند `offline_mode=True` يعيد `FakeLLMProvider` حتى إن اختير Ollama؛ هذا يخالف المصدر الرسمي الذي يتطلب استخدام مزود محلي حقيقي أو الفشل الآمن عند غيابه. كذلك Health يقرأ البيئة لا القيمة المحفوظة، ولا يوجد Network Guard شامل للأدوات/MCP |
| Windows/Linux | يستخدم `pathlib` ومسارات محلية | `config.py`، `asset_manager.py` | لم يوجد دليل CI على النظامين؛ SQLite URL وتوليد frontend يتركان نواتج غير متجاهلة |
| الاختبارات | أربعة اختبارات حرجة | `backend/tests/test_critical.py` | تستخدم قاعدة التشغيل الفعلية افتراضيًا؛ اختبار in-memory التشخيصي فشل بسبب تعدد اتصالات SQLite؛ لا اختبار إخفاق/إلغاء/DAG/عقد حقيقي |
| Git | غير موجود | `git status` أعاد “not a git repository” | لا نقطة رجوع أو تاريخ تغييرات؛ عائق P0 قبل التوسع |
| المصدر المعماري | Markdown رسمي كامل | `docs/AI_MANAGER_MASTER_ARCHITECTURE.md` | التطبيق لم يثبت بعد ADRs والعقود والمراحل الرسمية؛ الملفات PDF ليست مصدرًا لهذه الخطة |

المجلدات المطلوبة في التدقيق: `backend/` و`frontend/` موجودان؛ `contracts/` و`plugins/` و`tests/` في جذر المستودع غير موجودة. توجد الاختبارات فقط تحت `backend/tests/`، وتوجد الأدوات المضمنة تحت `backend/app/tools/plugins/`.

## 3. خريطة المكونات الحالية

التدفق الفعلي الحالي:

```text
User
→ React Workspace (`frontend/src/main.tsx`)
→ API client (`frontend/src/services/api.ts`)
→ POST /api/v1/assets (`backend/app/api/assets.py:upload_asset`)
→ AssetManager.save (`backend/app/services/asset_manager.py`)
→ POST /api/v1/requests (`backend/app/api/requests.py:create_request`)
→ RequestManager.create (`backend/app/services/request_manager.py`)
→ FastAPI BackgroundTasks
→ Orchestrator.run (`backend/app/services/orchestrator.py`)
→ select_provider → IntentAnalyzer → Planner
→ ToolRouter.route (فحص المجال فقط؛ لا يعيد تشكيل الخطة)
→ PlanValidator.validate
→ Executor.execute
→ ToolRegistry.require_enabled → PolicyEngine.check
→ Mock handler (`backend/app/tools/plugins/mock_*`)
→ Pydantic output schema
→ Verifier.verify
→ ResultComposer.compose
→ ExperienceStore.save
→ GET /api/v1/requests/{id} كل ثانية
→ Workspace result
```

ملاحظتان حاسمتان: `Planner` يضع أسماء الأدوات مباشرة، لذلك `Router` ليس صاحب الاختيار الفعلي بعد. كذلك `Executor` يحتوي فروعًا خاصة بـ`image.compare` و`audio.inspect` و`data.statistics` و`report.compose`، وهذا اقتران يجب إزالته قبل تعدد البرامج من دون تغيير واجهات المستخدم الحالية.

## 4. تحليل الفجوات

| المتطلب المعماري | الموجود حاليًا | الفجوة | الأولوية | المرحلة المقترحة |
| --- | --- | --- | --- | --- |
| نقطة رجوع قابلة للاستعادة | لا مستودع Git | لا يمكن Rollback موثوق | P0 | A |
| الأمان الافتراضي للأدوات | Built-ins مفعلة تلقائيًا | لا دورة Validate/Restricted/Enable ولا trust classification | P0 | A/F |
| عقود ذات إصدارات | Pydantic داخل Plugins و`dict` للخطة/النية | لا Schema موحد أو Compatibility | P0 | A |
| قاعدة بيانات قابلة للتطور | `create_all` فقط | لا Alembic أو Upgrade/Rollback | P0 | A |
| انتقالات حالة قانونية | تعيين String مباشر | يمكن القفز أو التناقض | P0 | A |
| اختبارات معزولة | الاختبارات على ملف DB الافتراضي | تلوث بيانات وخطر عدم التكرار؛ in-memory الحالي يفشل | P0 | A |
| أداة صور حقيقية | `mock_image` | لا تحليل/Artifact/ثقة حقيقي | P1 | B |
| أصول مشتقة وProvenance | `artifacts: []` | لا فرق مرئي أو سلسلة نسب | P1 | B |
| LLM محلي منظم | Ollama أولي | لا Schema validation/fallback/evaluation | P1 | C |
| Audio/Data حقيقيان | Mock | لا Adapters أو نتائج علمية | P1 | D |
| DAG متعدد البرامج | Executor تسلسلي خاص بالأسماء | لا قدرة مجردة أو تمرير مخرجات عام | P1 | E |
| معيار Plugin | تسجيل imports ثابت | لا Manifest/Health/Jobs/Versioning/SDK | P1 | F |
| Offline صارم | اختيار مزود فقط | لا منع اتصالات مركزي | P0 | A ثم C/F |
| سلوك LLM في Offline | Offline يجبر Fake | لا يستخدم المحلي المختار ولا يفشل آمنًا عند غيابه | P0 | A ثم C |
| آلة الحالات الرسمية | Strings جزئية وغير موحدة | أسماء وحالات ناقصة ولا State Transition Service | P0 | A |
| Python baseline | `>=3.12` مع تشغيل 3.13 | لا سياسة CI موحدة لـ3.13/3.14 | P2 | A/G |
| Policy profiles | أسماء مخزنة | لا اختلاف سلوكي أو Approval | P2 | F/G |
| الاعتمادية | Timeout وإلغاء بين المهام | لا Retry/Checkpoint/Resume/Circuit Breaker | P2 | G |
| Observability/Audit | ToolCall وExperience | لا Events/Correlation/Metrics | P2 | G |
| جاهزية الخادم | SQLite/ملفات/عملية واحدة | لا Ports أو PostgreSQL/Workers/Object Storage | P3 | H |
| User AI | غير موجود | لا Gateway/Source/Authority | P3 | I |
| SOLIDWORKS/CST | غير موجود | يحتاج Adapter مستقل وتقييم واجهة البرنامج | P3 | بعد F، لا تنفيذ الآن |

## 5. المعمارية المستهدفة

التطوير تدريجي داخل البنية الحالية؛ لا نقل شامل إلى هيكل جديد دفعة واحدة:

- `AI Manager Core — نواة المدير الذكي`: يحتفظ بـRequest/Plan/Task transitions، orchestration، validation، policy، verification. لا يعتمد على OpenCV أو Ollama أو برنامج تجاري.
- `LLM Providers — مزودات النموذج`: تطبق عقدًا منظمًا واحدًا وتعيد كائنات متحققة. لا ترى مسارات أو bytes ولا تنفذ أدوات.
- `Tool System — نظام الأدوات`: يفصل Capability عن Tool عن Provider؛ السجل والتوجيه والبوابة وحدها تسمح بالتنفيذ.
- `Plugin System — نظام الإضافات`: Manifest بإصدار، عقود، منصات، نقل، صحة، وظائف، تقدم وإلغاء. تحميله لا يمنح التنفيذ تلقائيًا.
- `Program Adapters — محولات البرامج`: `OpenCVAdapter` أولًا، ثم Audio/Data؛ ولاحقًا Adapters لـSOLIDWORKS/CST بعد إثبات معيار التكامل.
- `Workspace UI — مساحة العمل`: تبقى عميل API؛ تعرض الطلب والخطة والتقدم والArtifacts والثقة والمصدر ولا تتصل ببرنامج متخصص.
- `Storage — التخزين`: Repository ports فوق SQLite وAsset Storage؛ لاحقًا PostgreSQL وObject Storage دون تغيير العقود.
- `Experience Store — مخزن الخبرة`: يسجل Episode موثقة وتقييمًا، دون تدريب تلقائي.
- `Request Gateway — بوابة الطلبات`: نقطة دخول موحدة تضيف `source/context/authority/correlation_id` قبل Request Manager؛ يستخدمها Workspace ثم `User AI` مستقبلًا.

الحد الحاكم يبقى: `LLM Thinks → Planner Plans → Router Selects → Orchestrator Controls → Executor Executes → Tools Work → Verifier Verifies → Composer Presents`.

## 6. خارطة الطريق التنفيذية

### المرحلة A — تثبيت النسخة الحالية

إنشاء مستودع Git ونقطة رجوع، عزل الاختبارات، توحيد Python 3.13 baseline، تثبيت Pydantic contracts والإصدارات، إضافة Enums وانتقالات قانونية متوافقة مع الأسماء الرسمية، ثم Alembic بأول Baseline. تُصحح فجوة Verifier/Composer التي تفترض `mock: true` لتقبل `execution_kind: mock|real` مع إبقاء تحذير Mock، وتصبح الأدوات الجديدة Restricted افتراضيًا. كما يُفصل Fake بوصفه Test Provider عن سلوك Offline الإنتاجي. لا ميزات كبيرة ولا إعادة هيكلة سداسية شاملة.

### المرحلة B — أول تكامل حقيقي للصور

إضافة `OpenCVImageAdapter` محلي خلف `image.compare` مع Provider اختياري `mock_image` أو `opencv_image`. المدخلات تبقى `image_a/image_b` كـAsset IDs، ويحوّلها Asset Manager داخليًا إلى ملفات مقيدة. المخرجات: `similarity_score` بمنهج موثق، `differences[]` بمربعات وحدود وثقة، `difference_map_asset_id` و`annotated_image_asset_id`، تحذيرات ومحاذاة/أبعاد، و`mock: false`. أخطاء معيارية: `UNSUPPORTED_FORMAT`، `IMAGE_DECODE_FAILED`، `IMAGE_ALIGNMENT_FAILED`، `OUTPUT_VALIDATION_FAILED`. يحافظ Fake/Mock على الاختبارات السريعة. لا ادعاء دقة جنائية؛ يجب وجود صور Fixture ومعيار معروف.

### المرحلة C — النموذج المحلي الحقيقي

تقوية Ollama عبر Structured Outputs متحققة لكل Intent/Plan/Verify، timeouts واضحة، Health/Capabilities، وFallback مضبوط إلى Fake في الاختبار أو عند سياسة صريحة ومعلنة فقط. في التشغيل العادي يختار Offline مزودًا محليًا متاحًا، ويفشل بخطأ `MODEL_UNAVAILABLE` واضح إذا لم يوجد؛ لا يصمت بالتحول إلى Fake. يمنع Offline أي مزود غير محلي؛ ولا تمر سوى Asset IDs وmetadata. تضاف مجموعة صغيرة عربية/إنجليزية لقياس النية والخطة واختيار الأدوات.

### المرحلة D — الصوت والبيانات الحقيقيان

إنشاء `Audio Adapter — محول الصوت` للفحص المحلي أولًا (metadata، مدة، قنوات، sample rate، warnings) وتقنية مقترحة `soundfile`/`wave`، و`Data Adapter — محول البيانات` باستخدام `pandas` أو `polars` بعد قياس الحجم (Schema، missing values، summary statistics). كل Adapter مستقل عن النواة وله عقد وخطأ وثقة/حدود، مع بقاء Mock.

### المرحلة E — أول Workflow متعدد البرامج

تنفيذ `image.extract_table → dataset.normalize → data.statistics → data.visualize → report.compose`. يتغير Planner ليخطط بالقدرات، وRouter يربط كل قدرة بأداة/Provider. Executor ينفذ ترتيبًا طوبولوجيًا عامًا ويمرر Asset IDs/مخرجات بعقود لا بفروع أسماء. يتحقق Verifier من توافق الأنواع، كل Required Output، والنتائج الجزئية.

### المرحلة F — معيار تكامل البرامج

تثبيت Manifest وTool/Provider contracts وHealth/Jobs/Progress/Cancellation/Errors/Versioning، ثم Python SDK أولي وقالب Plugin واختبار توافق. يفتح هذا طريق SOLIDWORKS/CST عبر REST/SDK/CLI أو file exchange Adapter مستقل، لكنه لا يدمجهما الآن.

### المرحلة G — التقوية الهندسية

إضافة Event/Audit log، Correlation، Checkpoints وResume، Retry مصنف، Fallback، Circuit Breaker، Idempotency، حدود CPU/RAM/file/time، Network Guard، وتنظيف/احتفاظ بالأصول. تُضاف Observability تدريجيًا دون خدمة خارجية إجبارية.

### المرحلة H — الجاهزية للخادم

بعد استقرار `v1.0` المحلي: Repository ports وAlembic يمهّدان SQLite→PostgreSQL؛ Workflow port يمهّد in-process→Workers؛ Asset port يمهّد filesystem→Object Storage. يُضاف Docker Compose فقط هنا، وEvent Bus عند إثبات الحاجة. تشمل الخطة dual-read/backup migration وrollback إلى الإصدار المحلي وملف إعداد Windows/Linux.

### المرحلة I — توافق User AI

إضافة Request Gateway بعقد `source/identity/goal/context/authority/correlation_id`، مع بقاء User AI مصدر طلب لا منفذ أدوات. كل طلب يمر عبر Request Manager وPolicy/Tool Gateway؛ لا يصل User AI إلى Registry handlers أو الملفات مباشرة.

ترتيب A قبل B إلزامي بسبب أن Verifier الحالي يرفض النتيجة الحقيقية ضمنيًا، والاختبارات غير معزولة ولا توجد نقطة رجوع. أما C فيمكن بدء إعداد Fixtures له بالتوازي مع B بعد تثبيت العقود.

## 7. حزم العمل التنفيذية

### TASK-01 — نقطة رجوع وتدقيق قابل للتكرار

**الهدف:** إنشاء أساس Git نظيف وتثبيت طريقة تشغيل baseline دون تعديل السلوك.

**سبب التنفيذ الآن:** المستودع ليس Git؛ لا Rollback موثوق قبل تكامل حقيقي.

**النطاق:** تهيئة Git، مراجعة `.gitignore`، منع نواتج TypeScript/SQLite/assets، تشغيل baseline وتسجيل tag `v0.1-backbone` بعد تحقق المستخدم من الملفات المراد تتبعها.

**خارج النطاق:** أي إصلاح معماري أو Feature.

**الاعتماديات السابقة:** لا شيء.

**الملفات الحالية المتأثرة:**
- `[EXISTS] .gitignore`
- `[EXISTS] backend/tests/test_critical.py`
- `[EXISTS] frontend/tsconfig.json`

**الملفات الجديدة المتوقعة:**
- `[NEW] .git/` (بيانات Git، لا كود)

**الخطوات التنفيذية:**
1. تصنيف الملفات المصدرية مقابل `node_modules/dist/.venv/data/__pycache__/tsbuildinfo`.
2. تحديث ignore فقط لما هو مولد، ثم تشغيل tests/build/demo.
3. إنشاء commit baseline وtag بعد التأكد من عدم وجود أسرار أو ملفات بيانات مستخدم.

**العقود أو API أو Schema المتأثرة:** لا شيء.

**معيار القبول:**
- [ ] `git status` نظيف بعد baseline ولا يتتبع ملفات تشغيل أو بيانات.
- [ ] `pytest` وbuild وdemo تمر من commit المثبت.

**الاختبارات الضرورية فقط:** الأربعة الحالية، frontend build، demo.

**طريقة التراجع Rollback:** حذف تهيئة Git فقط إن لم يُنشأ تاريخ مطلوب؛ بعد tag استخدم revert لا reset مدمر.

**التقدير:**
- الحجم: S
- الزمن التقريبي: نصف يوم
- مستوى المخاطرة: منخفض

### TASK-02 — عزل الاختبارات وتثبيت قاعدة البيانات

**الهدف:** جعل الاختبارات مستقلة عن `backend/data/ai_manager.db` وقابلة للتكرار.

**سبب التنفيذ الآن:** الاختبارات الحالية تكتب في قاعدة التشغيل، ومحاولة SQLite `:memory:` التشخيصية فشلت لأن الاتصالات لا تشارك الحالة.

**النطاق:** App factory/engine fixture مؤقت، قاعدة ملف مؤقتة أو StaticPool مضبوط، تنظيف assets في temp، وعدم الاعتماد على DB موجودة.

**خارج النطاق:** Alembic أو تغيير Schema.

**الاعتماديات السابقة:** `TASK-01`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/db/session.py`
- `[EXISTS] backend/app/main.py`
- `[EXISTS] backend/tests/test_critical.py`
- `[EXISTS] backend/app/services/asset_manager.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/tests/conftest.py`

**الخطوات التنفيذية:**
1. حقن Database URL وAsset root في factory/fixtures.
2. نقل كل اختبار إلى DB/مجلد مؤقت.
3. إثبات أن قاعدة التشغيل وعدّاداتها لا تتغير بعد الاختبارات.

**العقود أو API أو Schema المتأثرة:** لا تغيير خارجي.

**معيار القبول:**
- [ ] تشغيل الاختبارات مرتين يعطي النتيجة نفسها ولا يضيف Requests/Assets تشغيلية.
- [ ] اختبار health ينشئ جداول في قاعدة الاختبار نفسها.

**الاختبارات الضرورية فقط:** health/database، E2E، تحقق عدم تغيير DB التشغيل.

**طريقة التراجع Rollback:** revert للحزمة؛ لا Migration ولا بيانات تشغيلية تتغير.

**التقدير:**
- الحجم: S
- الزمن التقريبي: نصف يوم إلى يوم
- مستوى المخاطرة: منخفض

### TASK-03 — تثبيت العقود والحالات والمهاجرات

**الهدف:** جعل Request/Intent/Plan/Task/ToolResult/Error عقودًا متحققة وإصدار أول Migration.

**سبب التنفيذ الآن:** إدخال `mock: false` أو DAG أوسع سيكسر افتراضات `dict`/String الحالية.

**النطاق:** Pydantic schemas، Enums وانتقالات بالأسماء الرسمية، Error envelope، Alembic baseline، تحديث Verifier/Composer لنتائج Mock وReal، وADRs موجزة للقرارات التي يثبتها التنفيذ.

**خارج النطاق:** أداة صور حقيقية أو إعادة تنظيم شاملة للمجلدات.

**الاعتماديات السابقة:** `TASK-02`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/models/entities.py`
- `[EXISTS] backend/app/schemas/api.py`
- `[EXISTS] backend/app/services/orchestrator.py`
- `[EXISTS] backend/app/services/verifier.py`
- `[EXISTS] backend/app/services/result_composer.py`
- `[EXISTS] backend/app/tools/contracts.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/schemas/contracts.py`
- `[NEW] backend/app/core/states.py`
- `[NEW] backend/alembic.ini`
- `[NEW] backend/migrations/` 
- `[NEW] contracts/` (نسخ JSON Schema منشورة فقط إذا تقرر جعلها واجهة مستقلة)
- `[NEW] docs/adr/` (ADRs الضرورية فقط: العقود، الحالات، Migration، Local-first/Offline)

**الخطوات التنفيذية:**
1. تعريف العقود والإصدارات و`execution_kind` وError codes، وحسم alias/migration من `VALIDATING_PLAN` إلى `PLAN_VALIDATION` دون كسر السجلات الحالية.
2. فرض transition service واختبار الحالات الرسمية والانتقالات غير القانونية.
3. إنشاء Alembic baseline مطابق للقاعدة الحالية واختبار upgrade من قاعدة فارغة.
4. إزالة شرط `all(mock is True)` من معنى الاكتمال مع إبقاء Mock labeling.
5. تسجيل القرارات المعمارية التي تحولت إلى كود كـADRs قصيرة؛ لا إنشاء حزمة توثيق واسعة.

**العقود أو API أو Schema المتأثرة:** Request response، Plan، Tool Result، Error؛ تغييرات additive مع إبقاء الحقول الحالية في `v1`.

**معيار القبول:**
- [ ] العقود ترفض مخرجات ناقصة أو حالات غير قانونية.
- [ ] Migration ينشئ Schema كاملًا ويرتقي من baseline.
- [ ] سيناريو Mock الحالي يظل مكتملًا دون تغيير UI.

**الاختبارات الضرورية فقط:** contract validation، state transitions، migration smoke، E2E الحالي.

**طريقة التراجع Rollback:** Alembic downgrade للـrevision السابق ثم revert؛ لا حذف بيانات دون backup.

**التقدير:**
- الحجم: M
- الزمن التقريبي: 2-4 أيام
- مستوى المخاطرة: متوسط

### TASK-04 — فصل Tool Gateway وتنفيذ Payload عام

**الهدف:** إزالة فروع أسماء الأدوات من Executor وجعل Router هو صاحب اختيار التنفيذ.

**سبب التنفيذ الآن:** `Executor.execute` مقترن بالأدوات الأربع، ما يمنع Adapter حقيقي قابل للاستبدال.

**النطاق:** Capability/Provider metadata، `ToolGateway`، input binding من الخطة/outputs، policy قبل ToolCall، ترتيب topological متسلسل، وحالة Restricted/Enabled صريحة لا تُمنح تلقائيًا لأي Plugin جديد.

**خارج النطاق:** توازي أو Workers.

**الاعتماديات السابقة:** `TASK-03`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/services/executor.py`
- `[EXISTS] backend/app/services/tool_router.py`
- `[EXISTS] backend/app/services/plan_validator.py`
- `[EXISTS] backend/app/tools/contracts.py`
- `[EXISTS] backend/app/tools/registry.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/services/tool_gateway.py`
- `[NEW] backend/app/services/input_binding.py`

**الخطوات التنفيذية:**
1. تمثيل tasks بقدرة ومدخلات references، ثم route إلى tool/provider.
2. نقل registry/schema/policy/asset checks إلى Gateway.
3. تنفيذ Topological order عام وربط outputs دون `if tool_id`.

**العقود أو API أو Schema المتأثرة:** Plan Task يضيف `capability/provider_id/input_bindings` مع توافق خلفي.

**معيار القبول:**
- [ ] إضافة أداة اختبار خامسة لا تتطلب تعديل Executor.
- [ ] Unknown/disabled/policy-denied لا ينشئ execution غير موثق.
- [ ] Plugin/Tool جديد يبقى غير قابل للتنفيذ حتى اجتياز validation والتفعيل الصريح.

**الاختبارات الضرورية فقط:** DAG order، binding، unknown tool، policy denial، E2E.

**طريقة التراجع Rollback:** إبقاء Executor القديم خلف feature flag حتى اجتياز E2E ثم حذفه في commit مستقل.

**التقدير:**
- الحجم: M
- الزمن التقريبي: 3-5 أيام
- مستوى المخاطرة: متوسط

### TASK-05 — الأصول المشتقة ومصدر النتائج

**الهدف:** تمكين الأدوات الحقيقية من إنشاء صور/تقارير مرتبطة بالأصول الأصلية.

**سبب التنفيذ الآن:** نتيجة الصور الحقيقية تحتاج Difference Map وAnnotated Image، بينما `artifacts` فارغة دائمًا.

**النطاق:** حفظ أصل مشتق atomically، role/parent/provenance/hash، content endpoint آمن، عرض Artifacts في UI.

**خارج النطاق:** Object Storage أو retention متقدم.

**الاعتماديات السابقة:** `TASK-03`، ويفضل `TASK-04`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/models/entities.py`
- `[EXISTS] backend/app/services/asset_manager.py`
- `[EXISTS] backend/app/api/assets.py`
- `[EXISTS] backend/app/services/result_composer.py`
- `[EXISTS] frontend/src/main.tsx`
- `[EXISTS] frontend/src/types/api.ts`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/services/artifact_service.py`
- `[NEW] backend/migrations/versions/[VERIFY]_add_artifacts.py`

**الخطوات التنفيذية:**
1. إضافة metadata/provenance/parent relations عبر Migration.
2. إنشاء API content بContent-Type مضبوط وبدون raw path.
3. تمرير Artifact refs إلى Result/UI.

**العقود أو API أو Schema المتأثرة:** `GET /api/v1/assets/{asset_id}/content`، Result `artifacts[]`.

**معيار القبول:**
- [ ] المخرج المشتق يعاد عبر Asset ID وتثبت بصمته وأصله الأب.
- [ ] لا يقبل endpoint traversal أو Asset غير معروف.

**الاختبارات الضرورية فقط:** derived asset، content security، provenance serialization.

**طريقة التراجع Rollback:** downgrade Migration مع الاحتفاظ بنسخة ملفات artifacts؛ revert UI/API.

**التقدير:**
- الحجم: M
- الزمن التقريبي: 2-4 أيام
- مستوى المخاطرة: متوسط

### TASK-06 — OpenCV image.compare الحقيقي

**الهدف:** تنفيذ أول تحليل حقيقي خلف عقد `image.compare`.

**سبب التنفيذ الآن:** هو إثبات أن النواة تستبدل Mock دون تغيير Planner أو Workspace flow.

**النطاق:** decode، normalization، alignment اختياري مضبوط، absdiff/threshold/components، score وثقة، annotated/difference assets، أخطاء وتحذيرات.

**خارج النطاق:** تعرف دلالي عميق، OCR، ادعاء جنائي أو طبي.

**الاعتماديات السابقة:** `TASK-04`، `TASK-05`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/tools/plugins/mock_image/__init__.py` (يبقى للاختبارات)
- `[EXISTS] backend/app/tools/plugin_loader.py`
- `[EXISTS] backend/pyproject.toml`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/adapters/image/opencv_adapter.py`
- `[NEW] backend/app/tools/plugins/real_image/` 
- `[NEW] backend/tests/fixtures/images/`
- `[NEW] backend/tests/test_image_integration.py`

**الخطوات التنفيذية:**
1. اعتماد Fixtures ومعنى similarity/confidence قبل الخوارزمية.
2. بناء Adapter لا يعرف DB أو Orchestrator.
3. تسجيل Provider محلي واختيار Mock/Real بالإعداد/health.
4. حفظ الصور المشتقة والتحقق منها.

**العقود أو API أو Schema المتأثرة:** `ImageCompareOutput v1.x` بإضافات اختيارية؛ لا تغيير `tool_id`.

**معيار القبول:**
- [ ] صورتان متطابقتان تعطيان فرقًا صفريًا/قريبًا ضمن tolerance.
- [ ] Fixture بتغيير معروف ينتج region وArtifact صحيحين و`mock: false`.
- [ ] Mock E2E يبقى حتميًا.

**الاختبارات الضرورية فقط:** identical/known difference/invalid decode/size mismatch/contract compatibility.

**طريقة التراجع Rollback:** تعطيل Provider الحقيقي في Manifest/settings والعودة إلى `mock_image` دون Migration rollback.

**التقدير:**
- الحجم: L
- الزمن التقريبي: 1-2 أسبوع
- مستوى المخاطرة: مرتفع

### TASK-07 — تحقق الصور والواجهة الواقعية

**الهدف:** منع الإعلان عن نتائج صور غير موثوقة وعرض Artifact/ثقة بوضوح.

**سبب التنفيذ الآن:** النجاح التقني للأداة لا يكفي لصحة الهدف.

**النطاق:** Image domain verifier، thresholds/warnings/uncertainty، UI comparison cards/downloads، إزالة Mock banner للنتيجة الحقيقية فقط.

**خارج النطاق:** تقييم بشري واسع أو تحسين بصري تجاري.

**الاعتماديات السابقة:** `TASK-06`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/services/verifier.py`
- `[EXISTS] backend/app/services/result_composer.py`
- `[EXISTS] frontend/src/main.tsx`
- `[EXISTS] frontend/src/styles.css`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/services/verifiers/image_verifier.py`
- `[NEW] frontend/src/components/[VERIFY]ArtifactGallery.tsx`

**الخطوات التنفيذية:**
1. تعريف complete/partial حسب required outputs وconfidence.
2. منع Composer من اختراع report/score.
3. عرض طريقة التحليل والقيود والمخرجات.

**العقود أو API أو Schema المتأثرة:** Result warnings/uncertainty/provenance/artifacts.

**معيار القبول:**
- [ ] غياب Artifact مطلوب ينتج `PARTIAL`.
- [ ] UI يفرق بوضوح بين Mock وReal ومنهج الدرجة.

**الاختبارات الضرورية فقط:** partial output، low confidence، UI build، سيناريو يدوي للصورتين.

**طريقة التراجع Rollback:** العودة إلى Result rendering السابق مع إبقاء حقول API الإضافية.

**التقدير:**
- الحجم: M
- الزمن التقريبي: 2-4 أيام
- مستوى المخاطرة: متوسط

### TASK-08 — Ollama Structured Provider وFallback

**الهدف:** تشغيل Intent/Plan/Verify بموديل محلي مع تحقق بنيوي وفشل آمن.

**سبب التنفيذ الآن:** المحول الحالي يطلب JSON لكنه يعيد `dict` غير مطابق وقد يسقط الطلب بلا fallback.

**النطاق:** prompt/version، JSON Schema، retry parse واحد، health/capabilities، اختيار provider، fallback policy، Asset metadata فقط.

**خارج النطاق:** Cloud LLM أو fine-tuning.

**الاعتماديات السابقة:** `TASK-03`؛ يمكن بالتوازي مع `TASK-06` بعد العقود.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/llm/base.py`
- `[EXISTS] backend/app/llm/ollama_provider.py`
- `[EXISTS] backend/app/llm/provider_factory.py`
- `[EXISTS] backend/app/services/orchestrator.py`
- `[EXISTS] backend/app/core/config.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/llm/prompts.py`
- `[NEW] backend/tests/fixtures/llm_cases.json`
- `[NEW] backend/tests/test_local_llm_contract.py`

**الخطوات التنفيذية:**
1. توحيد `generate_structured(schema, input)` أو مكافئ داخلي، مع `health` و`capabilities` وفق المنفذ الرسمي.
2. تحقق Pydantic لكل استجابة وتطبيع خطأ المزود.
3. تطبيق provider routing محلي أولًا؛ Fake للاختبارات، وfallback صريح ومسجل دون إخفاء انخفاض الجودة أو ادعاء تشغيل نموذج حقيقي.
4. تقييم حالات عربية/إنجليزية ثابتة.

**العقود أو API أو Schema المتأثرة:** Provider health/capabilities وProvenance provider/model/prompt version.

**معيار القبول:**
- [ ] JSON غير صالح لا يصل إلى Planner/Executor.
- [ ] Offline لا يتصل بغير localhost.
- [ ] غياب Ollama يتبع policy المحددة ويظهر Warning.
- [ ] `OFFLINE_MODE=true` لا يحوّل طلب الإنتاج إلى Fake بصمت؛ يستخدم المحلي أو يعيد `MODEL_UNAVAILABLE`.

**الاختبارات الضرورية فقط:** mocked HTTP structured/failure/timeout، Fake fallback، تقييم intent/plan صغير.

**طريقة التراجع Rollback:** ضبط `LLM_PROVIDER=fake`؛ لا تغيير Data مطلوب.

**التقدير:**
- الحجم: M
- الزمن التقريبي: 3-5 أيام، بخلاف تنزيل نموذج يدوي
- مستوى المخاطرة: متوسط

### TASK-09 — Audio inspect الحقيقي

**الهدف:** استبدال Mock audio metadata بمحول محلي حقيقي محدود.

**سبب التنفيذ الآن:** إثبات أن معيار العقود يعمل في مجال ثانٍ دون تضخيم إلى transcription.

**النطاق:** decode metadata، duration/channels/sample rate/level summary، errors، Mock fallback.

**خارج النطاق:** transcription وتصنيف دلالي طويل.

**الاعتماديات السابقة:** `TASK-04` و`TASK-03`؛ بعد استقرار صورة `v0.2`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/tools/plugins/mock_audio/__init__.py`
- `[EXISTS] backend/app/tools/plugin_loader.py`
- `[EXISTS] backend/pyproject.toml`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/adapters/audio/local_audio_adapter.py`
- `[NEW] backend/app/tools/plugins/real_audio/`
- `[NEW] backend/tests/fixtures/audio/`

**الخطوات التنفيذية:** فحص المكتبة الأقل اعتمادًا، تنفيذ العقد، تسجيل provider، contract/integration test.

**العقود أو API أو Schema المتأثرة:** `audio.inspect` output v1.x.

**معيار القبول:**
- [ ] WAV fixture يعيد metadata صحيحة ضمن tolerance.
- [ ] format غير مدعوم يعيد Error code منظمًا.

**الاختبارات الضرورية فقط:** valid fixture، corrupt/unsupported، Mock compatibility.

**طريقة التراجع Rollback:** تعطيل real audio provider والعودة للـMock.

**التقدير:**
- الحجم: M
- الزمن التقريبي: 3-5 أيام
- مستوى المخاطرة: متوسط

### TASK-10 — Data statistics الحقيقي

**الهدف:** تحليل CSV/TSV محليًا بنتيجة منظمة وآمنة الحجم.

**سبب التنفيذ الآن:** يمهد لسير image→dataset→statistics.

**النطاق:** schema inference محدود، row/column counts، missing values، numeric summaries، encoding/delimiter errors، Mock fallback.

**خارج النطاق:** SQL خارجي، notebooks، بيانات ضخمة موزعة.

**الاعتماديات السابقة:** `TASK-04` و`TASK-03`؛ يمكن بالتوازي مع `TASK-09`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/tools/plugins/mock_data/__init__.py`
- `[EXISTS] backend/app/tools/plugin_loader.py`
- `[EXISTS] backend/pyproject.toml`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/adapters/data/tabular_adapter.py`
- `[NEW] backend/app/tools/plugins/real_data/`
- `[NEW] backend/tests/fixtures/data/`

**الخطوات التنفيذية:** تثبيت limits، اختيار pandas/polars، تنفيذ streaming/size guard المناسب، عقد وإخفاقات.

**العقود أو API أو Schema المتأثرة:** `data.statistics` output v1.x.

**معيار القبول:**
- [ ] fixture معروف يعطي إحصاءات متوقعة.
- [ ] oversized/malformed input يفشل منظمًا دون استنزاف الذاكرة.

**الاختبارات الضرورية فقط:** known CSV، missing values، malformed/limit، Mock compatibility.

**طريقة التراجع Rollback:** تعطيل Provider الحقيقي والعودة للـMock.

**التقدير:**
- الحجم: M
- الزمن التقريبي: 3-5 أيام
- مستوى المخاطرة: متوسط

### TASK-11 — Planner بالقدرات وWorkflow متعدد البرامج

**الهدف:** تنفيذ السيناريو المرجعي متعدد البرامج بعقود DAG عامة.

**سبب التنفيذ الآن:** يثبت أن النظام مدير Workflow لا مجرد اختيار أداة واحدة.

**النطاق:** capabilities، typed bindings، `image.extract_table` (أولًا fixture/محرك محلي مناسب)، normalize/statistics/visualize/report، partial semantics.

**خارج النطاق:** توازي موزع أو OCR تجاري.

**الاعتماديات السابقة:** `TASK-04`، `TASK-05`، `TASK-07`، `TASK-10`؛ `TASK-08` مطلوب لمسار LLM الحقيقي لا لمسار Fake fixture.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/llm/fake_provider.py`
- `[EXISTS] backend/app/services/planner.py`
- `[EXISTS] backend/app/services/plan_validator.py`
- `[EXISTS] backend/app/services/executor.py`
- `[EXISTS] backend/app/services/verifier.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/tools/plugins/table_extraction/`
- `[NEW] backend/app/tools/plugins/data_visualization/`
- `[NEW] backend/tests/test_multi_program_workflow.py`

**الخطوات التنفيذية:** تعريف output references، بناء خطة Fake ثابتة، route providers، تنفيذ DAG، تحقق كل مرحلة وعرض artifacts.

**العقود أو API أو Schema المتأثرة:** Plan capability/bindings، dataset/visualization artifacts، Result provenance chain.

**معيار القبول:**
- [ ] طلب واحد ينفذ السلسلة كاملة دون أسماء أدوات من المستخدم.
- [ ] فشل مرحلة ينتج downstream cancelled/skipped ونتيجة `PARTIAL` موضحة.

**الاختبارات الضرورية فقط:** happy path، type mismatch، dependency failure، manual workspace flow.

**طريقة التراجع Rollback:** تعطيل capability الجديدة؛ يبقى image compare المستقل يعمل.

**التقدير:**
- الحجم: L
- الزمن التقريبي: 1-2 أسبوع
- مستوى المخاطرة: مرتفع

### TASK-12 — Plugin Manifest وPython SDK أولي

**الهدف:** تحويل تسجيل imports إلى معيار تكامل قابل للاختبار.

**سبب التنفيذ الآن:** بعد أداتين حقيقيتين يمكن تثبيت معيار من واقع الاستخدام لا من التخمين.

**النطاق:** Manifest schema/version/platform/transport/health/tools/jobs/progress/cancel/errors، loader آمن، compatibility suite، Python SDK صغير.

**خارج النطاق:** Marketplace أو تشغيل Plugin غير موثوق تلقائيًا.

**الاعتماديات السابقة:** `TASK-06`، `TASK-09` أو `TASK-10`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/tools/contracts.py`
- `[EXISTS] backend/app/tools/plugin_loader.py`
- `[EXISTS] backend/app/tools/registry.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] contracts/plugins/manifest.schema.json`
- `[NEW] plugins/` 
- `[NEW] sdk/python/`
- `[NEW] tests/contracts/`

**الخطوات التنفيذية:** استخراج المشترك من Adapters الحقيقية، تعريف Schema، validation/allowlist، SDK interfaces، مثال Plugin Mock خارجي.

**العقود أو API أو Schema المتأثرة:** Plugin/Tool/Provider v1، health/job/progress/error contracts.

**معيار القبول:**
- [ ] Manifest صالح يسجل أداة، وغير صالح/غير متوافق يُرفض قبل Registry.
- [ ] Plugin مثال يعمل على Windows وLinux CI دون تعديل Core.

**الاختبارات الضرورية فقط:** schema compatibility، duplicate/version/platform/disabled cases، example E2E.

**طريقة التراجع Rollback:** إبقاء built-in loader كمسار fallback لإصدار واحد؛ تعطيل external discovery.

**التقدير:**
- الحجم: L
- الزمن التقريبي: 1-2 أسبوع
- مستوى المخاطرة: مرتفع

### TASK-13 — Events والتقدم والإلغاء والموافقات الأساسية

**الهدف:** جعل العمليات الطويلة قابلة للرؤية والتحكم دون WebSocket إلزامي.

**سبب التنفيذ الآن:** Audio/program adapters قد تطول، والإلغاء الحالي لا يحدث إلا بين المهام.

**النطاق:** event table/sequence، progress endpoint أو توسيع polling، cancellation token، provider health، autonomy profiles وسلوك Approval بسيط لـR2+.

**خارج النطاق:** Event Bus موزع أو واجهة موافقات مؤسسية.

**الاعتماديات السابقة:** `TASK-12`، `TASK-03`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/models/entities.py`
- `[EXISTS] backend/app/services/policy_engine.py`
- `[EXISTS] backend/app/services/executor.py`
- `[EXISTS] backend/app/api/requests.py`
- `[EXISTS] frontend/src/main.tsx`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/services/event_store.py`
- `[NEW] backend/app/services/cancellation.py`
- `[NEW] backend/app/api/providers.py`
- `[NEW] backend/app/api/approvals.py`

**الخطوات التنفيذية:** event schema، emit transitions/progress، propagate cancellation، policy matrix، UI polling cursor.

**العقود أو API أو Schema المتأثرة:** Events، provider health، approvals، progress.

**معيار القبول:**
- [ ] المستخدم يرى progress مرتبًا ويوقف job يدعم cancel.
- [ ] R2 في conservative لا ينفذ دون Approval.

**الاختبارات الضرورية فقط:** ordered events، cancellation during tool، policy profiles، approval decision.

**طريقة التراجع Rollback:** إيقاف progress/approval feature flags؛ event rows additive.

**التقدير:**
- الحجم: L
- الزمن التقريبي: 1-2 أسبوع
- مستوى المخاطرة: مرتفع

### TASK-14 — Checkpoint وResume وRetry/Fallback

**الهدف:** التعافي من الأعطال دون إعادة العمل الصحيح أو مضاعفة الأثر.

**سبب التنفيذ الآن:** يصبح ضروريًا بعد workflows والأدوات الطويلة.

**النطاق:** checkpoint after task، resume validation، retryable errors/backoff محدود، provider fallback، circuit state، idempotency keys.

**خارج النطاق:** Temporal أو Queue موزعة.

**الاعتماديات السابقة:** `TASK-11`، `TASK-13`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/services/orchestrator.py`
- `[EXISTS] backend/app/services/executor.py`
- `[EXISTS] backend/app/models/entities.py`
- `[EXISTS] backend/app/api/requests.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/services/workflow_runtime.py`
- `[NEW] backend/app/services/retry_policy.py`
- `[NEW] backend/app/services/circuit_breaker.py`

**الخطوات التنفيذية:** normalize errors، persist attempts/checkpoints، resume endpoint، idempotency، failure injection.

**العقود أو API أو Schema المتأثرة:** Task attempts/checkpoint، `POST /requests/{id}/resume`، error retryability.

**معيار القبول:**
- [ ] transient failure يتعافى ضمن budget.
- [ ] restart/resume لا يعيد task ناجحة ولا يكرر Artifact.

**الاختبارات الضرورية فقط:** timeout/retry، permanent failure، resume، idempotency، circuit open/half-open.

**طريقة التراجع Rollback:** تعطيل resume/retry واستخدام التنفيذ مرة واحدة؛ migrations additive قابلة للـdowngrade.

**التقدير:**
- الحجم: L
- الزمن التقريبي: 1-2 أسبوع
- مستوى المخاطرة: مرتفع

### TASK-15 — الأمن والمراقبة وجودة الإصدار المحلي v1.0

**الهدف:** وضع حدود تشغيل وتدقيق وقياس تكفي لإصدار محلي ثابت.

**سبب التنفيذ الآن:** الاستقرار والأمان شرط قبل توزيع البرنامج أو ربط برنامج تجاري.

**النطاق:** network deny policy، file sniffing/limits/retention، structured logs/correlation، audit، health aggregate، Windows/Linux CI، backup/restore، benchmark صغير.

**خارج النطاق:** SOC/enterprise IAM أو Sandbox كامل.

**الاعتماديات السابقة:** `TASK-12` إلى `TASK-14`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/main.py`
- `[EXISTS] backend/app/core/config.py`
- `[EXISTS] backend/app/services/asset_manager.py`
- `[EXISTS] backend/app/api/system.py`
- `[EXISTS] .env.example`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/core/network_policy.py`
- `[NEW] backend/app/core/logging.py`
- `[NEW] backend/app/services/audit.py`
- `[NEW] .github/workflows/[VERIFY]_ci.yml`
- `[NEW] tests/security/`
- `[NEW] tests/evaluation/`

**الخطوات التنفيذية:** threat cases، guards، correlation/audit، dual-platform CI، backup drill، release checklist.

**العقود أو API أو Schema المتأثرة:** aggregate health/audit metadata؛ لا كسر API v1.

**معيار القبول:**
- [ ] Offline failure tests تثبت عدم الاتصال غير المصرح.
- [ ] CI Windows/Linux يمر؛ backup/restore يعيدان E2E data.
- [ ] كل Result له tool/provider/version/hash/correlation.

**الاختبارات الضرورية فقط:** path/network/prompt-data separation، resource limit، dual OS، restore smoke، benchmark regression.

**طريقة التراجع Rollback:** feature flags للقياسات، لكن لا تعطّل قيود الأمن؛ restore من backup قبل أي downgrade بيانات.

**التقدير:**
- الحجم: L
- الزمن التقريبي: 2-3 أسابيع
- مستوى المخاطرة: مرتفع

### TASK-16 — منافذ الخادم وبوابة User AI

**الهدف:** إعداد انتقال لاحق للخادم وتعدد مصادر الطلب دون تغيير Core contracts.

**سبب التنفيذ الآن:** بعد `v1.0` فقط؛ قبل ذلك يزيد التجريد المخاطر بلا قيمة مثبتة.

**النطاق:** Repository/Asset/Workflow ports، PostgreSQL migration rehearsal، Worker boundary، Object Storage interface، Request Gateway source/context/authority، Docker Compose للخادم التجريبي.

**خارج النطاق:** Multi-user production، distributed scale، تنفيذ SOLIDWORKS/CST، وصول User AI للأدوات.

**الاعتماديات السابقة:** `TASK-15` وإصدار `v1.0`.

**الملفات الحالية المتأثرة:**
- `[EXISTS] backend/app/db/session.py`
- `[EXISTS] backend/app/services/request_manager.py`
- `[EXISTS] backend/app/services/orchestrator.py`
- `[EXISTS] backend/app/services/asset_manager.py`

**الملفات الجديدة المتوقعة:**
- `[NEW] backend/app/ports/`
- `[NEW] backend/app/api/request_gateway.py`
- `[NEW] backend/app/infrastructure/postgres/`
- `[NEW] backend/app/infrastructure/workers/`
- `[NEW] compose.yaml`

**الخطوات التنفيذية:** استخراج ports من الاستخدام الفعلي، dual-backend tests، data migration/rollback rehearsal، gateway authorization envelope، deploy smoke.

**العقود أو API أو Schema المتأثرة:** Request adds optional `source/context/authority/correlation_id`; Core Tool access لا يتغير.

**معيار القبول:**
- [ ] نفس contract suite تعمل على SQLite/PostgreSQL وعلى in-process/worker test runtime.
- [ ] User AI request لا يستطيع تسمية handler أو تجاوز Policy/Gateway.
- [ ] rollback يعيد الخدمة المحلية والبيانات من backup مثبت.

**الاختبارات الضرورية فقط:** repository contract، migration rehearsal، worker idempotency، gateway authority/security.

**طريقة التراجع Rollback:** إيقاف خدمات الخادم، استعادة backup SQLite، الرجوع إلى in-process adapter؛ لا dual-write دائم قبل نجاح rehearsal.

**التقدير:**
- الحجم: L
- الزمن التقريبي: 3-6 أسابيع على دفعات
- مستوى المخاطرة: مرتفع

## 8. التبعيات والمسار الحرج

الترتيب الأساسي:

```text
TASK-01 → TASK-02 → TASK-03 → TASK-04 → TASK-05 → TASK-06 → TASK-07
                                      └────────→ TASK-08
TASK-04 → TASK-09 ─┐
TASK-04 → TASK-10 ─┴→ TASK-11 → TASK-12 → TASK-13 → TASK-14 → TASK-15 → TASK-16
```

- يمكن تنفيذ `TASK-08` بالتوازي مع `TASK-05/06` بعد `TASK-03`.
- يمكن تنفيذ `TASK-09` و`TASK-10` بالتوازي بعد ثبات العقود/Gateway، لكن يفضل بدء واحد فقط حتى يثبت Plugin pattern.
- يمكن تطوير UI في `TASK-07` بالتوازي مع جزء verifier بعد ثبات Result contract.
- **Critical Path — المسار الحرج:** `01 → 02 → 03 → 04 → 05 → 06 → 07 → 10 → 11 → 12 → 13 → 14 → 15`. `TASK-16` مسار ما بعد الإصدار المحلي.

نقاط التوقف اليدوية:

1. بعد `TASK-03`: تشغيل سيناريو Mock ومراجعة API/DB migration والانتقالات.
2. بعد `TASK-07`: مقارنة Fixtures وصور مستخدم غير حساسة في Workspace ومراجعة Artifact/ثقة.
3. بعد `TASK-08`: تشغيل Offline مع Ollama حاضر/غائب ومراجعة fallback.
4. بعد `TASK-11`: تشغيل Workflow متعدد البرامج من UI وفحص كل مرحلة ومصدر.
5. بعد `TASK-15`: اختبار إصدار Windows وLinux، backup/restore، ثم قرار `v1.0`.
6. قبل `TASK-16`: قرار معماري مستقل بالموافقة على الانتقال للخادم.

## 9. خطة الاختبارات

| المرحلة | الحد الأدنى المطلوب |
| --- | --- |
| A | Contract tests للنية/الخطة/النتيجة/الخطأ، state transitions، Migration smoke، E2E Mock مع DB مؤقتة |
| B | صور identical/known-diff/corrupt، Artifact/provenance، failure tool، سيناريو Workspace يدوي |
| C | Structured JSON صالح/فاسد/timeout، Fake fallback، Offline no-network، مجموعة Intent/Plan صغيرة |
| D | Fixture صوت معروف، CSV معروف، corrupt/unsupported/size limits، Mock compatibility |
| E | DAG/type bindings، workflow happy path، downstream failure→PARTIAL، سيناريو UI كامل |
| F | Manifest/Tool/Provider compatibility، invalid version/platform، health/job/cancel example plugin |
| G | timeout/retry/resume/idempotency/circuit، cancellation أثناء الأداة، audit/correlation، resource limits |
| H/I | repository contract على قاعدتين، migration+rollback، worker duplication، gateway authority/security |

لا يوجد هدف Coverage رقمي. معيار الاختيار هو المخاطر: كسر العقد، خروج Offline، نتيجة علمية خاطئة، فقد الحالة، تكرار الأثر، أو تجاوز السياسة.

## 10. خطة الإصدارات

| الإصدار | الوظائف | معيار الإصدار | ما لا يدخل | نقطة الرجوع |
| --- | --- | --- | --- | --- |
| `v0.1` Backbone Mock | الدورة الحالية Mock | tests/build/demo وtag baseline | تحليل حقيقي/LLM فعلي | `v0.1-backbone` |
| `v0.1.1` Stabilized Core | عقود/حالات/Migrations/Test isolation/Gateway | Mock E2E بلا كسر وMigration من صفر | Features جديدة | tag `v0.1.1` + Alembic downgrade |
| `v0.2` Real Image | OpenCV compare + Artifacts + verifier | Fixtures معروفة وUI real/mock واضح | OCR/صوت/بيانات | تعطيل real provider والعودة Mock |
| `v0.3` Local LLM | Ollama structured/fallback/evaluation | Offline/invalid JSON/intent-plan tests | Cloud LLM | `LLM_PROVIDER=fake` |
| `v0.4` Audio and Data | inspect/statistics حقيقيان | fixtures وعقود وفشل منظم | transcription/big data | تعطيل providers الحقيقية |
| `v0.5` Multi-Program Workflow | صورة→Dataset→Statistics→Visualization→Report | E2E متعدد الأدوات مع provenance وPARTIAL | Workers | تعطيل capabilities الجديدة |
| `v0.6` Integration SDK | Manifest/SDK/health/jobs/progress | مثال Plugin على Windows/Linux | Marketplace/SOLIDWORKS/CST | built-in loader fallback |
| `v0.8` Hardening | events/resume/retry/circuit/audit/security | failure/offline/restore tests | Server deployment | feature flags + DB backup |
| `v1.0` Stable Local | صورة/صوت/بيانات محلية، Workflow، UI، backup، dual OS | release checklist، لا P0/P1 مفتوحة، نتائج موثقة | Auth/multi-user/server | آخر tag مستقر + restore |
| `v1.1+` Server Deployment | PostgreSQL/workers/storage/gateway | migration/rollback وcontract parity | توزيع واسع أولًا | SQLite/in-process rollback |

## 11. المخاطر والقرارات

| الخطر | الاحتمال | التأثير | التخفيف |
| --- | --- | --- | --- |
| هلوسة النموذج | مرتفع | خطة/ادعاء خاطئ | Structured contracts، Validator، Fake baseline، domain verifier |
| كسر العقود | مرتفع | تعطيل Plugins/UI | Schema versioning، compatibility tests، تغييرات additive |
| ارتباط قوي ببرنامج واحد | متوسط | صعوبة الاستبدال | Capability/Tool/Provider/Adapter separation وعدم وضع OpenCV في Core |
| مشاكل Windows/Linux | متوسط | فشل تشغيل/مسارات | `pathlib`، dual-OS CI، fixtures وtemp dirs |
| تسرب بيانات خارج Offline | متوسط | خصوصية عالية التأثير | Network policy deny-by-default، localhost allowlist، failure tests |
| اعتبار Fake نموذجًا محليًا حقيقيًا | مرتفع حاليًا | نجاح زائف وتضليل المستخدم | فصل test provider عن runtime، `MODEL_UNAVAILABLE`، provenance وواجهة واضحة |
| تفعيل Plugin جديد تلقائيًا | مرتفع حاليًا | تجاوز مراجعة العقد والسياسة | Restricted-by-default، Manifest validation، تفعيل صريح وسجل تدقيق |
| تعطل برنامج خارجي | مرتفع | طلبات عالقة | health/timeout/cancel/retry/fallback/circuit/partial |
| تضخم النطاق | مرتفع | تأخير وفشل | بوابات إصدار وحزم متوسطة، تأجيل قائمة العناصر أدناه |
| تكامل برامج تجارية | مرتفع | Adapter هش/ترخيص | معيار F أولًا، API/SDK/CLI/file exchange؛ UI automation حل أخير |
| نتائج علمية غير موثوقة | مرتفع | قرار مستخدم خاطئ | منهج معلن، confidence/uncertainty، fixtures معيارية، provenance، لا ادعاء جنائي |
| فقد/تلوث بيانات الاختبار | مرتفع حاليًا | فساد baseline | `TASK-02` قبل أي توسع، temp DB/assets وbackup |
| Migration خاطئة | متوسط | فقد بيانات | Alembic، backup، upgrade/downgrade rehearsal |
| تكرار آثار عند Resume | متوسط لاحقًا | ملفات/عمليات مكررة | Idempotency keys وcheckpoint validation |

قرارات تنفيذية:

- لا إعادة كتابة إلى Hexagonal folders الآن؛ تُستخرج Ports عند ظهور Adapter ثانٍ أو حاجة الخادم.
- `OpenCV` هو أول برنامج حقيقي لأنه محلي ومتعدد الأنظمة وله Fixtures حتمية، وليس لأنه سيغطي تحليلًا دلاليًا كاملًا. يبدأ `v0.2` باستبدال Provider خلف `image.compare` كما تنص المرحلة 12 الرسمية؛ ثم تُفصل قدرات `inspect/align/detect/annotate` تدريجيًا للوصول إلى السيناريو المرجعي ذي المهام السبع، لا دفعة واحدة قبل إثبات الاستبدال.
- يبقى `tool_id=image.compare`؛ اختلاف Mock/Real يكون Provider، لا Tool جديد يفرض تغيير Planner/UI.
- لا يعلن similarity بوصفه حقيقة عامة؛ يرفق algorithm/version/parameters/confidence.

## 12. العناصر المؤجلة

- `Multi-User Authentication — مصادقة متعددة المستخدمين` والأدوار المؤسسية.
- `Hardware Integration — تكامل العتاد`.
- `Automatic Fine-Tuning — الضبط الدقيق التلقائي` والتعلم الذاتي في الإنتاج.
- `Plugin Marketplace — سوق الإضافات`.
- `Large-Scale Distributed Execution — التنفيذ الموزع واسع النطاق`.
- `Advanced Semantic Cache — ذاكرة دلالية متقدمة` وRAG/Vector DB.
- Cloud LLM providers قبل تثبيت Offline policy.
- SOLIDWORKS وCST قبل اكتمال `TASK-12` وتقييم واجهاتهما/تراخيصهما.
- WebSocket، Tauri، Kubernetes، Event Bus خارجي، GPU scheduler.
- OCR/Transcription/Computer Vision عميق قبل نجاح الأدوات الأساسية.
- أتمتة واجهات البرامج التجارية إلا كخيار أخير مدروس.

## 13. أول ثلاث مهام جاهزة للتنفيذ

### 1) تنفيذ TASK-01

> افحص `D:\NEW`، هيئ Git دون تتبع `.venv/node_modules/dist/data/__pycache__/tsbuildinfo` أو أسرار، حدّث `.gitignore` فقط عند الحاجة، شغّل Backend tests وFrontend build وdemo، ثم أنشئ commit/tag baseline باسم `v0.1-backbone`. لا تغيّر سلوك التطبيق. اعرض قائمة الملفات المتتبعة وأي مانع قبل commit إذا وجدت بيانات مستخدم أو أسرار.

### 2) تنفيذ TASK-02

> اعزل اختبارات `backend/tests/test_critical.py` عن `backend/data/ai_manager.db` و`backend/data/assets` باستخدام قاعدة ومجلد مؤقتين وApp/DB dependency injection بسيطة. أصلح حالة SQLite in-memory متعددة الاتصالات أو استخدم ملف SQLite مؤقتًا. أثبت أن تشغيل الاختبارات مرتين لا يغير عدادات قاعدة التشغيل. لا تغيّر API أو Schema.

### 3) تنفيذ TASK-03

> ثبّت عقود Pydantic ذات إصدار لـIntent/Plan/Task/ToolResult/Error/Result، وأنشئ Enums وخدمة انتقالات قانونية مطابقة للحالات الرسمية في `docs/AI_MANAGER_MASTER_ARCHITECTURE.md` مع migration/alias لحالة `VALIDATING_PLAN` الحالية، وأضف Alembic baseline مطابقًا لـ`backend/app/models/entities.py`. عدّل Verifier/Composer لقبول `execution_kind=mock|real` مع تحذير Mock دون اشتراط `mock:true` للاكتمال، وسجل ADRs موجزة للقرارات المنفذة. حافظ على API v1 وE2E الحالي، وأضف فقط اختبارات العقود/الحالات/Migration الضرورية.

## 14. القرار التنفيذي النهائي

1. **الخطوة التالية مباشرة:** `TASK-01` ثم `TASK-02`؛ لا يبدأ OpenCV قبل نقطة رجوع واختبارات معزولة.
2. **أول برنامج حقيقي:** Adapter محلي مبني على OpenCV/Pillow لتنفيذ `image.compare` وإنتاج Difference/Annotated Artifacts.
3. **ما لا يتغير في العمود الفقري:** فصل LLM عن التنفيذ، Asset IDs بدل paths، Registry/Policy/Gateway قبل Executor، Orchestrator صاحب السيطرة، Verifier قبل COMPLETED، Result provenance، وOffline افتراضي.
4. **متى يصبح جاهزًا للخادم:** بعد `v1.0` محلي يمر على Windows/Linux، وله Migrations وRepository/Asset/Workflow boundaries، Idempotency/Resume/Audit، backup/restore، وcontract suite؛ ثم تنجح rehearsal لـPostgreSQL/Workers مع rollback.
5. **شروط `v1.0`:** تحليل محلي حقيقي واحد على الأقل للصور والصوت والبيانات، Workflow متعدد البرامج موثق، LLM محلي منظم مع Fake fallback، Plugin contract ثابت، Offline مثبت باختبار منع اتصال، أخطاء/إلغاء/استكمال معقولة، نتائج وثقة/مصدر واضحة، UI عملية، لا فجوات P0/P1 مفتوحة، وإصدار قابل للاستعادة.
