# Notes: أسئلة وأجوبة عن المشروع

ملف بيتجمع فيه كل الأسئلة اللي اتسألت مع إجاباتها بالتفصيل.

---

## س1: لخص لي كل خطوات git اللي اتعملت على المشروع وفايدة كل أمر

الـ reflog بيسجل اللي اتعمل على الجهاز بالظبط. الريبو فيه 296 commit، واللي اتعمل من الجهاز هو الـ 18 commit الأخيرة (من `028e211` لـ `367e61c`).

### 1) إعداد الريبو (أوامر git على الجهاز)

| الأمر | اللي حصل | فايدته |
|---|---|---|
| `git clone git@github.com:Eagle-Vision-Company/DeepstreamService.git` | نزّلت المشروع من GitHub على نسخة عند commit `d019c44` | بيجيب نسخة كاملة من الكود مع كل التاريخ، وبيضبط `origin` تلقائيًا |
| `git checkout main` ثم `git checkout release/fit-fix` | روحت على branch الـ release | بيبدّل الـ branch اللي شغال عليه |
| `git checkout -b R&D/DeepStream_Tracking` | عملت branch جديد للـ R&D (بيظهر في الـ reflog كـ checkout من `release/fit-fix`) | بيعزل شغل البحث عن الـ main والـ release، فمفيش حاجة تتأثر |
| `git reset` (moving to HEAD) | تنضيف الـ staging area | بيشيل أي ملفات اتضافت بالغلط من غير ما يمس الكود |
| `git submodule` | المشروع فيه submodules: `people_count_checker` و`carwash_checker` و`door_checker` | كل usecase ليه ريبو مستقل جوه `src/core/processors/` |

الـ branch الحالي هو `R&D/DeepStream_Tracking`، والـ remote واحد هو `origin`.

### 2) الـ commits اللي اتعملت (بالترتيب)

كل commit بيتعمل بـ `git add` ثم `git commit -m "..."`. الفايدة من `add` إنه بيحدد إيه يدخل الـ commit، والفايدة من `commit` إنه بيحفظ لقطة بتقدر ترجعلها.

**مرحلة تحسين الـ reception counter والـ tracking (22 سبتمبر إلى 4 أكتوبر):**
- `028e211`: إصلاح detection الموظفين وثبات الـ identity.
- `30c1614`: ضبط ReID matching ومدة الاحتفاظ بالـ track.
- `857a7de`: تحقيق في مشكلة انقسام الـ tracker لما حد ورا المكتب (occlusion).
- `941b194`: تحديث إعدادات الـ reception counter والـ tracking.

**مرحلة Kafka والـ shutdown (5 أكتوبر):**
- `1329b09`: إضافة `check_kafka` (عارض topics واختبار end-to-end).
- `05f73a3`: إضافة Kafka broker لنود واحدة للاختبار المحلي.
- `6d16ce4`: أوامر `just` للـ Kafka: `kafka-test-up`, `kafka-test-down`, `kafka-check`, `kafka-e2e`.
- `36cbfa1` و`434d70e` و`7656ccc`: جعل الـ `shutdown()` آمنة، وإضافة `ProcessorRegistry.shutdown_all()`. دلوقتي الـ processors بتتفلش وKafka بيتوقف قبل ما الـ pipeline يتقفل، فمفيش داتا بتضيع.

**مرحلة الـ production (5 أكتوبر):**
- `f45ace5`: الـ Dockerfile بيشغل `nvidia_entrypoint.sh` مباشرة في مرحلة الـ production.
- `0bee8a9`: `.dockerignore` بيمنع الـ recordings واللوجات والموديلات من دخول الـ image.
- `1341d3a` و`59322cc`: production compose override، وتوثيق المتغيرات في `.env.example`.
- `340287b`: أوامر `just`: `prod-build` و`prod-up` و`prod-down` و`prod-logs`.
- `2d9563e`: نشر آخر صف للعميل على Kafka لما الـ track ينتهي.
- `3c49a3b`: تحديث `.gitignore`.
- `367e61c`: تحديث (bump) الـ submodule `people_count_checker` بعد حذف `legacy_code`.

### 3) أوامر مفيدة للمتابعة

```bash
git log --oneline          # تاريخ الـ commits
git reflog                 # كل حركاتك المحلية
git status                 # حالة الملفات
git diff                   # التغييرات اللي لسه ما اتعملهاش commit
git submodule status       # حالة الـ submodules
git push -u origin R&D/DeepStream_Tracking   # رفع الشغل على GitHub
```

### ملاحظات
- مفيش `push` ظاهر في الـ reflog، لأنه مبيسجلش عمليات الـ push. اتأكد بـ `git status -sb` إن الـ branch ما بقاش "ahead" عن `origin`.
- في الـ branch تاريخ قديم فيه 33 merge، ومعظمه شغل الفريق عبر الـ Pull Requests (الأرقام زي `#96` و`#90`).
- أمر `git submodule status` رجّع خطأ عند `third_party/DeepStream-Yolo`. الـ gitdir بتاعه مش موجود، فلو احتجت الـ submodule ده نفّذ `git submodule update --init`.

---

## س2: لو عايزة أشغل المشروع as development، إيه الخطوات اللي أقدر أعملها بنفسي؟ ولو عايزة أشغله as production؟

الشرح مبني على `justfile` و`deploy/docker-compose.yml` و`deploy/docker-compose.prod.yml`. لازم تشتغل جوه Docker، لأن DeepStream SDK 8.0 مش موجود على الجهاز.

### أولًا: قبل أي حاجة (مرة واحدة)

```bash
git submodule update --init --recursive     # يجيب الـ processors (people_count_checker وغيره)
docker network create eagle_vision_network  # الـ compose بيستخدمها كـ external
cp deploy/.env.example deploy/.env          # وعدّل القيم (Kafka credentials وCAMERA_API وغيرهم)
```

لازم يكون عندك:
- Docker و`nvidia-container-toolkit` (للـ GPU)
- `just`

### ثانيًا: Development

الـ dev container بيشتغل idle (`tail -f /dev/null`)، وكل الكود متركب جواه من الريبو (`..:/app`). يعني أي تعديل في الكود يظهر فورًا من غير rebuild.

```bash
just up                  # يبني الصورة ويشغل الـ container في الخلفية
just attach-dev          # تدخل جواه (docker exec -it ... bash)
```

جوه الـ container، أول مرة بس:

```bash
just setup-container     # ينسخ ملفات .so + ينزّل الموديلات + يبني triton backends
```

بعدين شغّل الـ service بنفسك. الأمر الفعلي مش في الـ justfile، فاتأكد من نقطة الدخول في الـ Dockerfile. السكريبتات في `people_count_checker` بتشغّله كده:

```bash
cd /app && python3 -m src.main
```

الـ API بيتفتح على `http://localhost:8509/docs`. وعشان تضيف كاميرات من ملف JSON:

```bash
just setup-pipeline <config.json>      # من برا الـ container
```

أوامر مساعدة للتطوير:

| الأمر | الفايدة |
|---|---|
| `just kafka-test-up` | يشغل Kafka تجريبي معزول |
| `just kafka-e2e` | يختبر إن البيانات توصل Kafka صح |
| `just kafka-check` | يعرض الـ topics والرسايل |
| `just kafka-test-down` | يوقف الـ Kafka التجريبي ويمسح داتاه |
| `just lint-fmt` | `ruff format` و`ruff check --fix` (لازم يعدّي قبل كل commit) |
| `just down` | يوقف الـ dev container |

### ثالثًا: Production

الـ production بيشغّل الـ service لوحدها من الصورة (obfuscated). مفيش mount للكود، وكل الـ state بيتخزن على الـ host.

**1. جهّز فولدرات الـ host** (المسارات الافتراضية، وتتغير من `deploy/.env`):

```
/srv/reception/logs     # الـ gallery وvisits.db
/srv/reception/zones    # لازم فيه zones_last_update.csv
/srv/reception/assets   # models/peoplenet_transformerv2 و models/ReID_models
/srv/reception/app_dir  # التسجيلات
```

**2. عدّل `deploy/.env`:** بيانات Kafka الحقيقية، و`LOAD_CAMERAS_ON_STARTUP`، و`CAMERA_API_BASE_URL`، ووقت الـ inference.

**3. شغّل:**

```bash
just prod-build    # يبني الصورة من الشجرة الحالية
just prod-up       # يشغلها ويستنى لحد ما تبقى healthy (ممكن 10 دقايق)
just prod-logs     # متابعة اللوجات
just prod-down     # إيقاف بشكل نضيف (30 ثانية grace)
```

أول تشغيل بياخد وقت لأن TensorRT بيبني engines للـ GPU وبيحفظها في فولدر `assets`. التشغيلات بعد كده بتبقى أسرع.

الفحص الصحي (healthcheck) بيعمل `curl` على `:8501/docs` جوه الـ container. اتأكد من الحالة بـ `docker ps` أو `docker compose ... ps`.

### الفرق باختصار

| | Development | Production |
|---|---|---|
| الكود | متركب من الريبو (تعديل فوري) | مدمج في الصورة (obfuscated) |
| التشغيل | بتشغله يدويًا جوه الـ container | تلقائي لما الـ container يبدأ |
| الـ state | داخل الريبو | فولدرات على الـ host |
| الأوامر | `just up` ثم `attach-dev` | `just prod-build` ثم `prod-up` |

**تنبيه:** الشرح ده من قراءة الملفات بس، ما اتشغّلش فعليًا. ولو الـ `eagle_vision_network` مش موجودة، الـ compose هيفشل.

---

## س3: إيه الفايلات دي؟ `run_reception_persistent.sh` و`run_reception_on_video.sh`

(المسار: `src/core/processors/people_count_checker/`)

### الخلاصة: سكريبتات اختبار على فيديو (مش للـ production)

الاتنين سكريبتات bash للتجربة والتقييم. بتشغّل الـ `reception_counter` على **فيديو مسجّل** بدل كاميرا RTSP حقيقية، وبتسجّل الـ OSD output (الفيديو بعد رسم الـ boxes والـ IDs) في ملف `.mkv`. مفيش علاقة بينهم وبين `just prod-*`.

### الخطوات المشتركة

كل سكريبت بيشتغل على container شغّال (عن طريق `docker exec`) وبيعمل 6 خطوات:

1. يوقف أي service شغالة (`pkill python3` وتحرير بورتات 8501 و10554).
2. يجهّز الـ state (الفرق بين السكريبتين هنا).
3. يشغل الـ service: `python3 -m src.main` مع `LOAD_CAMERAS_ON_STARTUP=False`.
4. يضيف الفيديو كـ source عن طريق الـ API: `POST /sources` بـ `file://...mp4` و`file_loop:true`.
5. يشغّل الـ source ويوصّل عليه branch اسمه `reception_counter`.
6. يسجّل ستريم الـ RTSP الناتج بـ `ffmpeg`، يوقف الـ service، وينسخ الفيديو للجهاز بـ `docker cp`.

### الفرق بينهم

| | `run_reception_on_video.sh` | `run_reception_persistent.sh` |
|---|---|---|
| الـ gallery و`visits.db` | **بيمسحهم** قبل كل تشغيل (يبدأ من صفر) | **ما بيمسحهمش** (الذاكرة بتتكمل) |
| الـ container الافتراضي | `deepstream-hadeer-DS-Tracking` | `deepstream-hadeer` |
| الـ arguments | `<video> [seconds]` | `<video> [seconds] [branch-output-name] [branch_id]` |
| مكان الحفظ | `logs/` | `logs/<branch>/videos/` لو حددت branch |
| ترتيب الـ recorder | يستنى 15 ثانية بعد التشغيل ثم يسجّل | **يشغّل ffmpeg قبل الـ source** عشان ما يفوتش أول فريمات |
| التقرير النهائي | قرارات الـ identity بس | تقرير أكمل (تحت) |

التقرير الأكمل في `persistent` بيشمل: حالة `visits.db` قبل التشغيل، رسالة "restored panel" (دليل إن الذاكرة اتحمّلت)، قرارات الـ ReID، الـ visits اللي اتقفلت (`new=True` يعني زود العدد، و`new=False` يعني حدّث صف موجود)، الحالة النهائية، والـ demographics المنشورة على Kafka topic اسمه `reception-summary`.

**ليه الاتنين؟** الأول لاختبار نضيف من الصفر. التاني لاختبار إن الـ ReID بيتعرف على نفس الشخص لما يرجع (من غير ما يتعدّ مرتين).

### طريقة الاستخدام

```bash
cd src/core/processors/people_count_checker

# من الصفر، الفيديو كله
./run_reception_on_video.sh 2026-08-09_23-00

# أول 120 ثانية بس
./run_reception_on_video.sh 2026-08-09_15-00 120

# مع الاحتفاظ بالذاكرة، ويحفظ في logs/<branch>/videos/
./run_reception_persistent.sh 2026-08-09_15-00 120 my_branch default_branch
```

### قبل التشغيل

1. **اسم الـ container hardcoded:** `deepstream-hadeer-DS-Tracking` و`deepstream-hadeer`. الـ justfile بيسمّي الـ container `deepstream-<USER>` (يعني `deepstream-hadeer`). الأول هيفشل لو الـ container اسمه غير كده. شغّل `docker ps` وعدّل متغير `CONTAINER` في أول السكريبت.
2. **الفيديوهات لازم تكون في** `data/videos/` جوه الـ container (مسار `/app/src/core/processors/people_count_checker/data/videos`)، وبدون `.mp4` في الاسم.
3. **الـ dev container لازم يكون شغّال** وجاهز (`just up` ثم `just setup-container`).
4. **مسح الداتا:** الأول بيمسح `/app/logs/gallery` و`/app/logs/visits` بدون تأكيد. لو في داتا مهمة هناك، خد نسخة قبل التشغيل.
5. **الأمر `pkill -9 python3`** بيقتل كل بروسيس python3 جوه الـ container، مش الـ service بس.

الكلام ده من قراءة الكود، والسكريبتات ما اتشغّلتش. وفي `src/core/processors/people_count_checker/run_and_record_reception_counter.md` شرح إضافي لنفس الموضوع.

---

## س4: ممكن تحط كل الأسئلة اللي هسألها هنا في ملف notes.md مع الإجابات بالتفصيل؟

تمام. الملف ده هو `notes.md` في root المشروع. أي سؤال جديد هيتضاف في آخره بنفس الشكل (السؤال، ثم الإجابة بالتفصيل).
