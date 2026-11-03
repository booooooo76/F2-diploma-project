# РОЗДІЛ 2. ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

Архітектуру описано за моделлю C4 (Simon Brown) на трьох рівнях: контекст системи, контейнери та компоненти критичного контейнера. Діаграми подано у форматі Mermaid.js (Diagrams-as-Code) у нотації C4: тип кожного елемента вказано в квадратних дужках ([Person], [Software System], [Container: технологія], [Component: тип]), кольори відповідають стандартній палітрі C4, зв’язки підписано дією та протоколом. Вимоги, на які посилаються описи, визначено в [специфікації вимог](srs.md); ключові архітектурні рішення зафіксовано в [ADR](adr/).

## 2.1 Контекстне та контейнерне моделювання системи (C4 Model)

<a id="diagram-level1"></a>

### 2.1.1 Рівень 1. System Context

```mermaid
flowchart TB
  reader(["<b>Читач</b><br/>[Person]<br/><br/>Веде особисту бібліотеку, фіксує<br/>читання, сканує полиці, створює нотатки"])
  admin(["<b>Адміністратор</b><br/>[Person]<br/><br/>Налаштовує правила гейміфікації,<br/>запускає перебудову проєкцій"])

  bookoff["<b>BookOff</b><br/>[Software System]<br/><br/>Трекінг читання з розпізнаванням<br/>книжкових полиць і подієвою гейміфікацією"]

  gbooks["<b>Google Books API</b><br/>[Software System]<br/><br/>Основне джерело<br/>бібліографічних метаданих"]
  olib["<b>Open Library API</b><br/>[Software System]<br/><br/>Резервне джерело<br/>бібліографічних метаданих"]

  reader -. "Керує бібліотекою,<br/>завантажує фото полиць<br/>[HTTPS]" .-> bookoff
  admin -. "Налаштовує правила,<br/>запускає перебудову<br/>[HTTPS]" .-> bookoff
  bookoff -. "Пошук видань за назвою,<br/>автором, ISBN<br/>[REST / HTTPS, JSON]" .-> gbooks
  bookoff -. "Резервний пошук<br/>[REST / HTTPS, JSON]" .-> olib

  classDef person fill:#08427B,stroke:#073B6F,color:#fff
  classDef system fill:#1168BD,stroke:#0B4884,color:#fff
  classDef external fill:#999999,stroke:#8A8A8A,color:#fff
  class reader,admin person
  class bookoff system
  class gbooks,olib external
```

| Елемент | Тип | Опис | Трасування |
|---|---|---|---|
| Читач | Person | Основний користувач: бібліотека, сесії читання, сканер, нотатки, показники гейміфікації | [SRS-FUN-01…SRS-FUN-23](srs.md#21-модуль-ідентифікація-та-доступ), [SRS-FUN-25](srs.md#srs-fun-25) |
| Адміністратор | Person | Користувач із роллю admin: конфігурація правил нарахування та перебудова проєкцій | [SRS-FUN-24](srs.md#srs-fun-24), [SRS-NFR-SEC-06](srs.md#srs-nfr-sec-06) |
| BookOff | System | Система, що проєктується | [SRS, розділ 1](srs.md#1-system-context-контекст-системи) |
| Google Books API | System_Ext | Основне джерело метаданих; взаємодія REST / HTTPS, відповіді JSON, обмеження добовою квотою | [SRS-FUN-07](srs.md#srs-fun-07), [SRS-NFR-REL-04](srs.md#srs-nfr-rel-04) |
| Open Library API | System_Ext | Резервне джерело метаданих; використовується після розмикання Circuit Breaker | [SRS-NFR-REL-04](srs.md#srs-nfr-rel-04) |

Усі взаємодії користувачів із системою виконуються через HTTPS (TLS 1.2/1.3) відповідно до [SRS-NFR-SEC-07](srs.md#srs-nfr-sec-07).

<a id="diagram-level2"></a>

### 2.1.2 Рівень 2. Container

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 30, "rankSpacing": 110}}}%%
flowchart LR
  reader(["<b>Читач</b><br/>[Person]<br/><br/>Веде бібліотеку, фіксує<br/>читання, сканує полиці"])
  admin(["<b>Адміністратор</b><br/>[Person]<br/><br/>Налаштовує правила<br/>гейміфікації"])

  subgraph bookoff["BookOff [Software System]"]
    direction LR
    spa["<b>Вебклієнт</b><br/>[Container: Vue 3, TypeScript]<br/><br/>SPA: бібліотека, статистика,<br/>сканер, нотатки, адмін-розділ"]
    mobile["<b>Android-клієнт</b><br/>[Container: Capacitor]<br/><br/>Та сама SPA у нативній<br/>оболонці; камера, штрихкод"]
    proxy["<b>Reverse Proxy</b><br/>[Container: Caddy 2]<br/><br/>TLS, HSTS, CORS,<br/>маршрутизація /api/v1"]
    api["<b>API-застосунок</b><br/>[Container: Python, FastAPI]<br/><br/>Модульний моноліт: identity, catalog,<br/>scanner, reading, notes, gamification"]
    worker["<b>Scanner Worker</b><br/>[Container: Celery, PaddleOCR]<br/><br/>Розпізнавання корінців<br/>і пошук кандидатів"]
    relay["<b>Outbox Relay</b><br/>[Container: Python]<br/><br/>Публікує події<br/>з outbox у Kafka"]
    consumer["<b>Gamification Consumer</b><br/>[Container: Python, aiokafka]<br/><br/>Ідемпотентне нарахування,<br/>CQRS-проєкції"]
    s3[("<b>Сховище зображень</b><br/>[Container: MinIO]<br/><br/>Фото полиць,<br/>видалення через 24 год")]
    redis[("<b>Redis</b><br/>[Container: Redis 7]<br/><br/>Черга Celery, кеш,<br/>ліміти, ключі ідемпотентності")]
    pg[("<b>PostgreSQL</b><br/>[Container: PostgreSQL 16]<br/><br/>Схема на модуль,<br/>outbox, проєкції")]
    kafka[("<b>Apache Kafka</b><br/>[Container: KRaft, 1 брокер]<br/><br/>Журнал доменних<br/>подій, DLQ")]
    mon["<b>Моніторинг</b><br/>[Container: OTel Collector,<br/>Prometheus, Grafana]<br/><br/>Метрики, траси, сповіщення"]
  end

  bib["<b>Google Books API,<br/>Open Library API</b><br/>[Software System]<br/><br/>Бібліографічні метадані"]

  reader -. "Використовує<br/>[HTTPS]" .-> spa
  reader -. "Використовує<br/>[Android]" .-> mobile
  admin -. "Адмін-розділ<br/>[HTTPS]" .-> spa
  spa -. "Виклики API<br/>[REST / HTTPS]" .-> proxy
  mobile -. "Виклики API<br/>[REST / HTTPS]" .-> proxy
  proxy -. "Проксіювання<br/>[HTTP / TCP 8000]" .-> api
  api -. "Дані модулів + outbox<br/>[SQL / TCP 5432]" .-> pg
  api -. "Задачі, кеш, ліміти<br/>[RESP / TCP 6379]" .-> redis
  api -. "Запис фото<br/>[S3 API / HTTP]" .-> s3
  api -. "Пошук видань<br/>[REST / HTTPS]" .-> bib
  api -. "Траси, метрики<br/>[OTLP / gRPC]" .-> mon
  worker -. "Отримання задач<br/>[RESP / TCP 6379]" .-> redis
  worker -. "Читання фото<br/>[S3 API / HTTP]" .-> s3
  worker -. "Результат + outbox<br/>[SQL / TCP 5432]" .-> pg
  worker -. "Пошук кандидатів<br/>[REST / HTTPS]" .-> bib
  relay -. "Вибірка outbox<br/>[SQL / TCP 5432]" .-> pg
  relay -. "Публікація подій<br/>[Kafka / TCP 9092]" .-> kafka
  consumer -. "Споживання подій<br/>[Kafka / TCP 9092]" .-> kafka
  consumer -. "Оновлення проєкцій<br/>[SQL / TCP 5432]" .-> pg

  classDef person fill:#08427B,stroke:#073B6F,color:#fff
  classDef container fill:#438DD5,stroke:#3C7FC0,color:#fff
  classDef external fill:#999999,stroke:#8A8A8A,color:#fff
  class reader,admin person
  class spa,mobile,proxy,api,worker,relay,consumer,pg,redis,kafka,s3,mon container
  class bib external
  style bookoff fill:#ffffff,stroke:#444,stroke-dasharray:5 5,color:#000
```

**Опис контейнерів**

| Контейнер | Технологія | Відповідальність | Трасування |
|---|---|---|---|
| Вебклієнт | Vue 3, TypeScript, Vite | Інтерфейс користувача й адміністратора; виклики REST API | [SRS-FUN-01…SRS-FUN-25](srs.md#2-functional-requirements-функціональні-вимоги) |
| Android-клієнт | Capacitor | Нативна оболонка тієї самої SPA; доступ до камери, декодування EAN-13 на пристрої | [SRS-FUN-10](srs.md#srs-fun-10), [SRS-FUN-14](srs.md#srs-fun-14) |
| Reverse Proxy | Caddy 2 | TLS-термінація з автоматичними сертифікатами, HSTS, CORS, роздавання статичних файлів SPA | [SRS-NFR-SEC-07](srs.md#srs-nfr-sec-07) |
| API-застосунок | Python, FastAPI | Модулі identity, catalog, scanner (прийом зображень і статус задач), reading, notes, gamification (читання проєкцій). Автентифікація JWT, RBAC, Token Bucket, Idempotency-Key. Запис подій у outbox у транзакції модуля | [SRS-FUN-01…SRS-FUN-10](srs.md#srs-fun-01), [SRS-FUN-12…SRS-FUN-25](srs.md#srs-fun-12), [SRS-NFR-SEC-01…SRS-NFR-SEC-06](srs.md#srs-nfr-sec-01), [SRS-NFR-REL-03](srs.md#srs-nfr-rel-03) |
| Outbox Relay | Python | Окремий процес; публікує події з outbox у Kafka з ключем партиціювання `user_id` | [SRS-NFR-REL-02](srs.md#srs-nfr-rel-02), [ADR-001](adr/0001-choice-of-technology.md) |
| Gamification Consumer | Python, aiokafka | Ідемпотентні споживачі: нарахування досвіду, серії, рівні навичок, досягнення, сповіщення; перебудова проєкцій | [SRS-FUN-20…SRS-FUN-25](srs.md#srs-fun-20), [SRS-NFR-PERF-05](srs.md#srs-nfr-perf-05), [SRS-NFR-REL-05](srs.md#srs-nfr-rel-05) |
| Scanner Worker | Celery, PaddleOCR, RapidFuzz | Розпізнавання корінців і формування кандидатів; деталізацію подано в підрозділі [2.2](#diagram-level3) | [SRS-FUN-11](srs.md#srs-fun-11), [SRS-NFR-PERF-04](srs.md#srs-nfr-perf-04), [SRS-NFR-QUAL-01](srs.md#srs-nfr-qual-01) |
| PostgreSQL | PostgreSQL 16 | Окрема схема на модуль; таблиці outbox і оброблених подій; CQRS-проєкції | [SRS-NFR-OPS-02](srs.md#srs-nfr-ops-02) |
| Redis | Redis 7 з ACL | Брокер задач Celery, кеш метаданих (Cache-Aside), лічильники Token Bucket, ключі ідемпотентності з TTL 24 год | [SRS-NFR-PERF-06](srs.md#srs-nfr-perf-06), [SRS-NFR-SEC-05](srs.md#srs-nfr-sec-05) |
| Apache Kafka | Kafka, KRaft, 1 брокер | Журнал доменних подій без обмеження строку зберігання; DLQ-топік | [SRS-NFR-REL-06](srs.md#srs-nfr-rel-06), [ADR-001](adr/0001-choice-of-technology.md) |
| Сховище зображень | MinIO (S3 API) | Тимчасове зберігання фото полиць; автоматичне видалення через 24 год | [SRS-NFR-SEC-08](srs.md#srs-nfr-sec-08) |
| OTel Collector, Prometheus і Grafana | OpenTelemetry, Prometheus, Grafana | Збір трас і метрик RED, панелі, сповіщення, синтетичні перевірки доступності | [SRS-NFR-OPS-03](srs.md#srs-nfr-ops-03), [SRS-NFR-REL-01](srs.md#srs-nfr-rel-01) |

**Протоколи взаємодії**

| Зв’язок | Протокол | Примітка |
|---|---|---|
| Клієнти → Reverse Proxy | REST / HTTPS (TLS 1.2/1.3) | JSON; `multipart/form-data` для зображень |
| Reverse Proxy → API | HTTP/1.1 / TCP 8000 | Внутрішня мережа Docker, порт не публікується назовні |
| API, Worker, Relay, Consumer → PostgreSQL | SQL / TCP 5432 | Драйвер asyncpg; окремі ролі СУБД на модуль |
| API, Worker → Redis | RESP / TCP 6379 | ACL-користувачі з мінімальними правами; Redis не публікується назовні |
| Relay, Consumer → Kafka | Kafka protocol / TCP 9092 | Ключ партиціювання `user_id` зберігає порядок подій користувача |
| API, Worker → MinIO | S3 API / HTTP | Внутрішня мережа Docker |
| API, Worker → Google Books, Open Library | REST / HTTPS | Таймаут, Retry, Circuit Breaker відповідно до [SRS-NFR-REL-04](srs.md#srs-nfr-rel-04) |
| API, Worker, Consumer → Моніторинг (OTel Collector) | OTLP / gRPC 4317 | Траси та метрики; на діаграмі показано зв’язок від API, інші процеси експортують дані аналогічно |

Окремий API Gateway і зовнішній провайдер ідентичності не використовуються: функції TLS-термінації та маршрутизації виконує Caddy, а автентифікацію, ліміти та перевірку прав — API-застосунок. Обґрунтування наведено в журналі архітектурного рев’ю (AI Insights, ЛР № 5).

## 2.2 Компонентне моделювання критичного сервісу системи

<a id="diagram-level3"></a>

Для деталізації обрано контейнер Scanner Worker: він виконує основне обчислювальне навантаження системи ([SRS-NFR-PERF-04](srs.md#srs-nfr-perf-04)) і реалізує ключову функцію ідентифікації видань ([SRS-FUN-11](srs.md#srs-fun-11)). Компоненти розподілено за шарами Clean Architecture: вхідний шар, шар застосунку, доменне ядро та вихідні адаптери. Доменне ядро залежить лише від інтерфейсів (портів); реалізації портів підставляються через композиційний корінь (Dependency Injection). Порти позначено в підписах зв’язків (наприклад, `[TextRecognizerPort]`), адаптери є їхніми реалізаціями.

```mermaid
flowchart TB
  redis[("<b>Redis</b><br/>[Container: Redis 7]<br/><br/>Черга задач Celery")]

  subgraph worker["Scanner Worker [Container: Python, Celery]"]
    direction TB
    consumer["<b>ScanTaskConsumer</b><br/>[Component: Celery task]<br/><br/>Приймає задачу ScanRequested,<br/>валідує payload (Pydantic)"]
    guard["<b>ScanJobGuard</b><br/>[Component: Guard]<br/><br/>Перевіряє власника і статус<br/>задачі, блокує повторну обробку"]
    errors["<b>TaskErrorHandler</b><br/>[Component: Celery signals]<br/><br/>До 2 повторів, потім<br/>статус «помилка»"]
    usecase["<b>RecognizeShelfUseCase</b><br/>[Component: Application Service]<br/><br/>Оркеструє конвеєр<br/>розпізнавання полиці"]
    grouper["<b>SpineGrouper</b><br/>[Component: Domain Service]<br/><br/>Групує фрагменти тексту<br/>за корінцями"]
    matcher["<b>CandidateMatcher</b><br/>[Component: Domain Service]<br/><br/>Оцінка схожості RapidFuzz,<br/>відсіювання за порогом"]
    ocr["<b>PaddleOcrAdapter</b><br/>[Component: Adapter]<br/><br/>Детекція, поворот ±90°,<br/>розпізнавання тексту"]
    catalog["<b>ResilientCatalogClient</b><br/>[Component: Adapter]<br/><br/>Кеш Redis, Retry, Circuit Breaker,<br/>резервне джерело"]
    storage["<b>S3ImageStorage</b><br/>[Component: Adapter]<br/><br/>Читання зображення<br/>за ключем"]
    telemetry["<b>TelemetryInstrumentation</b><br/>[Component: OpenTelemetry SDK]<br/><br/>Спани етапів конвеєра,<br/>гістограми тривалості"]
    uow["<b>PgScanUnitOfWork</b><br/>[Component: Repository]<br/><br/>Результат задачі + подія<br/>outbox в одній транзакції"]
  end

  s3[("<b>Сховище зображень</b><br/>[Container: MinIO]")]
  pg[("<b>PostgreSQL</b><br/>[Container: схема scanner]")]
  otel["<b>OTel Collector</b><br/>[Container: OpenTelemetry]"]
  bib["<b>Google Books API,<br/>Open Library API</b><br/>[Software System]"]

  redis -. "Доставка задачі<br/>[RESP / Celery]" .-> consumer
  consumer -. "Перевіряє задачу<br/>[виклик через DI]" .-> guard
  consumer -. "Запускає сценарій<br/>[виклик через DI]" .-> usecase
  errors -. "Обробляє винятки<br/>[Celery signals]" .-> consumer
  guard -. "Читає статус задачі<br/>[ScanUnitOfWorkPort]" .-> uow
  usecase -. "Групує фрагменти" .-> grouper
  usecase -. "Оцінює кандидатів" .-> matcher
  usecase -. "Розпізнає текст<br/>[TextRecognizerPort]" .-> ocr
  usecase -. "Шукає кандидатів<br/>[CandidateSourcePort]" .-> catalog
  usecase -. "Отримує зображення<br/>[ImageStoragePort]" .-> storage
  usecase -. "Фіксує результат<br/>[ScanUnitOfWorkPort]" .-> uow
  storage -. "GetObject<br/>[S3 API / HTTP]" .-> s3
  uow -. "Транзакція<br/>[SQL / TCP 5432]" .-> pg
  catalog -. "Пошук видань<br/>[REST / HTTPS]" .-> bib
  telemetry -. "Інструментує етапи<br/>[декоратор, контекст спану]" .-> usecase
  telemetry -. "Експорт спанів<br/>[OTLP / gRPC]" .-> otel

  classDef component fill:#85BBF0,stroke:#5D82A8,color:#000
  classDef container fill:#438DD5,stroke:#3C7FC0,color:#fff
  classDef external fill:#999999,stroke:#8A8A8A,color:#fff
  class consumer,guard,errors,usecase,grouper,matcher,ocr,catalog,storage,uow,telemetry component
  class redis,s3,pg,otel container
  class bib external
  style worker fill:#ffffff,stroke:#444,stroke-dasharray:5 5,color:#000
```

**Опис компонентів**

| Шар | Компонент | Роль | Взаємодія | Трасування |
|---|---|---|---|---|
| Вхідний | ScanTaskConsumer | Точка входу Celery-задачі; не містить бізнес-логіки | Отримує повідомлення з Redis; викликає валідатор, guard і сценарій через DI | [SRS-FUN-11](srs.md#srs-fun-11) |
| Вхідний | TaskPayloadValidator | Перевіряє структуру повідомлення (`job_id`, `user_id`, `image_key`, `schema_version`) моделлю Pydantic | Некоректне повідомлення відхиляється без повторів | [SRS-NFR-SEC-06](srs.md#srs-nfr-sec-06) |
| Вхідний | ScanJobGuard | Аналог guard для черги: повідомлення в Redis не є довіреним, тому власник і статус задачі звіряються із записом у PostgreSQL; повторна доставка задачі не запускає повторної обробки | Читає задачу через `ScanUnitOfWorkPort` | [SRS-NFR-SEC-06](srs.md#srs-nfr-sec-06), [SRS-NFR-REL-05](srs.md#srs-nfr-rel-05) |
| Вхідний | TaskErrorHandler | Глобальний обробник помилок задачі: тимчасові помилки — до 2 повторів, постійні — статус «помилка» | Сигнали Celery `task_failure`, `task_retry` | [SRS-NFR-REL-05](srs.md#srs-nfr-rel-05) |
| Наскрізний | TelemetryInstrumentation | Спан на кожен етап конвеєра, гістограма тривалості задачі | Експорт OTLP у OTel Collector | [SRS-NFR-PERF-04](srs.md#srs-nfr-perf-04), [SRS-NFR-OPS-03](srs.md#srs-nfr-ops-03) |
| Застосунок | RecognizeShelfUseCase | Оркестрація: зображення → OCR → групування → кандидати → оцінка → фіксація | Залежить лише від портів і доменних сервісів | [SRS-FUN-11](srs.md#srs-fun-11) |
| Домен | SpineGrouper | Групує фрагменти тексту за геометрією рамок у корінці | Чиста функція без залежностей від інфраструктури | [SRS-FUN-11](srs.md#srs-fun-11) |
| Домен | CandidateMatcher | Обчислює оцінку схожості (RapidFuzz, token set ratio) і відсіює кандидатів за порогом | Чиста функція; поріг задається конфігурацією | [SRS-NFR-QUAL-01](srs.md#srs-nfr-qual-01) |
| Порти | TextRecognizerPort, CandidateSourcePort, ImageStoragePort, ScanUnitOfWorkPort | Інтерфейси (Python Protocol), від яких залежить ядро | Реалізуються адаптерами; підміняються тестовими дублерами | — |
| Вихідний | PaddleOcrAdapter | Реалізація розпізнавання на PaddleOCR; модель завантажується один раз на процес | Реалізує `TextRecognizerPort` | [SRS-NFR-PERF-04](srs.md#srs-nfr-perf-04) |
| Вихідний | ResilientCatalogClient | Кеш Redis, таймаут 3 с, Retry, Circuit Breaker, резервне джерело Open Library | Реалізує `CandidateSourcePort` | [SRS-NFR-REL-04](srs.md#srs-nfr-rel-04), [SRS-NFR-PERF-06](srs.md#srs-nfr-perf-06) |
| Вихідний | S3ImageStorage | Читання зображення з MinIO | Реалізує `ImageStoragePort` | [SRS-NFR-SEC-08](srs.md#srs-nfr-sec-08) |
| Вихідний | PgScanUnitOfWork | Запис результату задачі та події `ScanCompleted` в outbox однією транзакцією | Реалізує `ScanUnitOfWorkPort` | [SRS-NFR-REL-02](srs.md#srs-nfr-rel-02) |

Автентифікація JWT у цьому контейнері не виконується: він не має вхідних HTTP-інтерфейсів. Перевірка токена й прав доступу виконується в API-застосунку під час прийому зображення ([SRS-FUN-10](srs.md#srs-fun-10)), а Scanner Worker повторно звіряє власника задачі з даними PostgreSQL. Redis доступний лише у внутрішній мережі Docker з ACL-автентифікацією.

## 2.3 Документування архітектурних рішень (Architecture Decision Records)

| ID | Рішення | Статус | Пов’язані вимоги |
|---|---|---|---|
| [ADR-001](adr/0001-choice-of-technology.md) | Вибір брокера доменних подій: Apache Kafka замість RabbitMQ і Redis Streams | Approved | [SRS-NFR-REL-02](srs.md#srs-nfr-rel-02), [SRS-NFR-REL-06](srs.md#srs-nfr-rel-06), [SRS-NFR-PERF-05](srs.md#srs-nfr-perf-05), [SRS-FUN-24](srs.md#srs-fun-24) |
