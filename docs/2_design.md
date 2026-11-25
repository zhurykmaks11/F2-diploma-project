# РОЗДІЛ 2. ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

> Архітектура описана за ієрархічною моделлю C4 (Simon Brown) за підходом Diagrams-as-Code (Mermaid.js).
> Вхідний контракт вимог — [Специфікація вимог (SRS)](./srs.md). Архітектурні компроміси — каталог [ADR](./adr/).
>
> **Нотація.** Діаграми виконано синтаксисом `flowchart` Mermaid з нотацією C4: тип елемента вказано в квадратних дужках (`[Person]`, `[Container: …]`, `[Component]`), технологія/протокол зв'язку — у підписі стрілки. Колір: темно-синій — актори, синій — контейнери системи, сірий — зовнішні системи та контейнери поза межами діаграми.

## Зміст

- [2.1. Контекстне та контейнерне моделювання системи (C4 Model)](#21-контекстне-та-контейнерне-моделювання-системи-c4-model)
  - [2.1.1. C4 Level 1 — System Context Diagram](#211-c4-level-1--system-context-diagram)
  - [2.1.2. C4 Level 2 — Container Diagram](#212-c4-level-2--container-diagram)
- [2.2. Компонентне моделювання критичного сервісу системи](#22-компонентне-моделювання-критичного-сервісу-системи)
- [2.3. Документування архітектурних рішень (Architecture Decision Records)](#23-документування-архітектурних-рішень-architecture-decision-records)
- [2.4. Матриця трасування «Вимога → Архітектурний елемент»](#24-матриця-трасування-вимога--архітектурний-елемент)

---

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model)

<a id="diagram-level1"></a>

### 2.1.1. C4 Level 1 — System Context Diagram

```mermaid
flowchart TB
  student["<b>Студент / Покупець</b><br/>[Person]<br/>Шукає курси, оплачує замовлення,<br/>отримує доступ до матеріалів"]
  author["<b>Автор курсу</b><br/>[Person]<br/>Публікує курси та матеріали"]
  admin["<b>Адміністратор платформи</b><br/>[Person]<br/>Керує ролями, контролює Outbox/DLQ і метрики"]
  platform["<b>Платформа електронної комерції цифрових курсів</b><br/>[Software System]<br/>CQRS + Event-Driven Architecture:<br/>NestJS, Apache Kafka, PostgreSQL, Redis"]
  psp["<b>Платіжний провайдер (PSP)</b><br/>[External System]<br/>Hosted Checkout, sandbox-режим,<br/>вебхуки з HMAC-підписом"]

  student -- "Каталог, замовлення, доступ<br/>[HTTPS / REST, JSON]" --> platform
  author -- "Керування курсами, завантаження файлів<br/>[HTTPS / REST, multipart/form-data]" --> platform
  admin -- "Адміністрування, моніторинг<br/>[HTTPS / REST, JSON]" --> platform
  platform -- "Створення checkout-сесії<br/>[HTTPS / REST, JSON]" --> psp
  psp -- "Вебхук результату оплати<br/>[HTTPS POST, HMAC-SHA256]" --> platform
  student -- "Введення платіжних даних<br/>[HTTPS, redirect]" --> psp

  class student,author,admin person
  class platform system
  class psp ext
  classDef person fill:#08427b,stroke:#052e56,color:#fff
  classDef system fill:#1168bd,stroke:#0b4884,color:#fff
  classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
  classDef container fill:#438dd5,stroke:#2e6295,color:#fff
  classDef db fill:#438dd5,stroke:#2e6295,color:#fff
  classDef infra fill:#85bbf0,stroke:#5d82a8,color:#000
```

#### Опис елементів та зв'язків

| Елемент | Тип | Роль | Трасування |
| :--- | :--- | :--- | :--- |
| Студент / Покупець | Person | Пошук курсів, оформлення замовлення, отримання доступу до матеріалів. | [SRS-FUN-11](./srs.md#srs-fun-11), [SRS-FUN-13](./srs.md#srs-fun-13), [SRS-FUN-23](./srs.md#srs-fun-23) |
| Автор курсу | Person | Створення та редагування курсів, завантаження файлів матеріалів. | [SRS-FUN-07](./srs.md#srs-fun-07), [SRS-FUN-08](./srs.md#srs-fun-08) |
| Адміністратор платформи | Person | Керування ролями, контроль Outbox/DLQ, перегляд метрик. | [SRS-FUN-06](./srs.md#srs-fun-06), [SRS-FUN-25](./srs.md#srs-fun-25), [SRS-FUN-26](./srs.md#srs-fun-26) |
| Платформа | Software System | Межі системи згідно з [SRS, розділ 2.1](./srs.md#21-межі-системи-scope). | — |
| Платіжний провайдер (PSP) | External System | Hosted Checkout та асинхронне повідомлення про результат оплати. Дані платіжних карток вводяться лише на сторінці PSP. | [SRS-FUN-16](./srs.md#srs-fun-16), [SRS-FUN-17](./srs.md#srs-fun-17), [SRS-NFR-SEC-07](./srs.md#srs-nfr-sec-07) |

| Зв'язок | Протокол | Призначення |
| :--- | :--- | :--- |
| Актори → Платформа | HTTPS / REST, JSON (multipart/form-data для файлів) | Усі сценарії користувача через Web SPA; TLS 1.3 ([SRS-NFR-SEC-01](./srs.md#srs-nfr-sec-01)). |
| Платформа → PSP | HTTPS / REST, JSON | Створення checkout-сесії ([SRS-FUN-16](./srs.md#srs-fun-16)). |
| PSP → Платформа | HTTPS POST, підпис HMAC-SHA256 | Вебхук результату оплати ([SRS-NFR-SEC-05](./srs.md#srs-nfr-sec-05)). |
| Студент → PSP | HTTPS, redirect | Введення платіжних даних на стороні PSP ([SRS-NFR-SEC-07](./srs.md#srs-nfr-sec-07)). |

<a id="diagram-level2"></a>

### 2.1.2. C4 Level 2 — Container Diagram

```mermaid
flowchart TB
  user["<b>Студент / Автор / Адміністратор</b><br/>[Person, браузер]"]
  psp["<b>Платіжний провайдер (PSP)</b><br/>[External System]<br/>Hosted Checkout, вебхуки"]

  subgraph platform["Платформа цифрових курсів [Software System]"]
    direction TB
    gw["<b>API Gateway</b><br/>[Container: Nginx]<br/>TLS 1.3, rate limiting,<br/>маршрутизація, роздача SPA"]
    spa["<b>Web SPA</b><br/>[Container: React, TypeScript]<br/>UI студента, автора, адмін-панель"]

      identity["<b>Identity Service</b><br/>[Container: NestJS]<br/>JWT RS256, JWKS, RBAC,<br/>GDPR-анонімізація"]
      course["<b>Course Service</b><br/>[Container: NestJS, Command]<br/>Зміна курсів, Outbox"]
      query["<b>Catalog Query Service</b><br/>[Container: NestJS, Query]<br/>Пошук, фільтри, проєкція подій"]
      order["<b>Order Service</b><br/>[Container: NestJS, Command]<br/>Замовлення, вебхук PSP,<br/>Outbox + Outbox Relay"]
      access["<b>Access Service</b><br/>[Container: NestJS]<br/>Idempotent Consumer,<br/>Access Grant, presigned URL"]

    kafka[["<b>Event Bus</b><br/>[Container: Apache Kafka, KRaft]<br/>course.events, order.events, *.dlq"]]

      pg[("<b>Write DB</b><br/>[PostgreSQL 16]<br/>схеми: identity, course,<br/>orders, access")]
      redis[("<b>Read Model / Cache</b><br/>[Redis 7]<br/>read-модель каталогу,<br/>ключі ідемпотентності")]
      minio[("<b>Object Storage</b><br/>[MinIO, S3 API]<br/>файли матеріалів")]

    obs["<b>Observability</b><br/>[Container: Prometheus + Grafana]<br/>RED-метрики, consumer lag"]
  end

  user -- "1. SPA та API<br/>[HTTPS / REST, JSON, TLS 1.3]" --> gw
  gw -- "2. Статичні файли<br/>[HTTP]" --> spa
  gw -- "/api/auth/*<br/>[HTTP/1.1 / REST]" --> identity
  gw -- "/api/courses/*<br/>[HTTP/1.1 / REST]" --> course
  gw -- "/api/catalog/*<br/>[HTTP/1.1 / REST]" --> query
  gw -- "/api/orders/*, /api/webhooks/psp<br/>[HTTP/1.1 / REST]" --> order
  gw -- "/api/access/*<br/>[HTTP/1.1 / REST]" --> access
  psp -- "Вебхук оплати<br/>[HTTPS POST, HMAC-SHA256]" --> gw
  order -- "Checkout-сесія<br/>[HTTPS / REST, JSON]" --> psp

  course & order -- "Публікація подій (Outbox Relay)<br/>[Kafka protocol / TCP 9092]" --> kafka
  kafka -- "Споживання подій<br/>[Kafka protocol / TCP 9092]" --> query & order & access

  identity & course & order & access -- "[SQL / TCP 5432]" --> pg
  query & access -- "[RESP / TCP 6379]" --> redis
  course & access -- "Upload / presign<br/>[S3 API / HTTP 9000]" --> minio
  user -. "Завантаження матеріалу<br/>[HTTPS GET, presigned URL]" .-> minio
  obs -. "Scrape /metrics усіх сервісів<br/>[HTTP / Prometheus exposition]" .-> order

  class user person
  class psp ext
  class gw,spa,identity,course,query,order,access container
  class kafka,pg,redis,minio db
  class obs infra
  classDef person fill:#08427b,stroke:#052e56,color:#fff
  classDef system fill:#1168bd,stroke:#0b4884,color:#fff
  classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
  classDef container fill:#438dd5,stroke:#2e6295,color:#fff
  classDef db fill:#438dd5,stroke:#2e6295,color:#fff
  classDef infra fill:#85bbf0,stroke:#5d82a8,color:#000
```

#### Опис контейнерів

| Контейнер | Технологія | Відповідальність | Реалізує вимоги |
| :--- | :--- | :--- | :--- |
| Web SPA | React, TypeScript, React Query, Tailwind CSS | UI студента, автора та адмін-панель; зберігає access token у пам'яті, refresh token — у cookie `HttpOnly; Secure; SameSite=Strict`. | [SRS-FUN-11](./srs.md#srs-fun-11), [SRS-FUN-25](./srs.md#srs-fun-25) |
| API Gateway | Nginx | Єдина точка входу: TLS 1.3 termination, rate limiting (`limit_req`), маршрутизація за префіксом шляху, роздача статики SPA, генерація `X-Correlation-Id`. | [SRS-NFR-SEC-01](./srs.md#srs-nfr-sec-01), [SRS-NFR-SEC-04](./srs.md#srs-nfr-sec-04), [SRS-NFR-OBS-02](./srs.md#srs-nfr-obs-02) |
| Identity Service | NestJS | Реєстрація, видача та ротація JWT RS256, JWKS-endpoint, RBAC, анонімізація за GDPR. | [SRS-FUN-01](./srs.md#srs-fun-01)…[SRS-FUN-06](./srs.md#srs-fun-06), [SRS-NFR-SEC-02](./srs.md#srs-nfr-sec-02), [SRS-NFR-SEC-03](./srs.md#srs-nfr-sec-03) |
| Course Service | NestJS (Command side) | Зміна курсів авторами, завантаження файлів у MinIO, запис `CourseUpserted` в Outbox. | [SRS-FUN-07](./srs.md#srs-fun-07), [SRS-FUN-08](./srs.md#srs-fun-08), [SRS-FUN-09](./srs.md#srs-fun-09) |
| Catalog Query Service | NestJS (Query side) | Проєкція `course.events` у read-модель Redis; пошук і фільтрація без звернення до PostgreSQL. | [SRS-FUN-10](./srs.md#srs-fun-10), [SRS-FUN-11](./srs.md#srs-fun-11), [SRS-FUN-12](./srs.md#srs-fun-12), [SRS-NFR-PERF-02](./srs.md#srs-nfr-perf-02) |
| Order Service | NestJS (Command side) | Життєвий цикл замовлення, інтеграція з PSP, Transactional Outbox, Outbox Relay. Деталізація — [C4 Level 3](#diagram-level3). | [SRS-FUN-13](./srs.md#srs-fun-13)…[SRS-FUN-19](./srs.md#srs-fun-19), [SRS-NFR-PERF-03](./srs.md#srs-nfr-perf-03) |
| Access Service | NestJS | Idempotent Consumer `PaymentConfirmed`, Access Grant, presigned URL. | [SRS-FUN-21](./srs.md#srs-fun-21)…[SRS-FUN-24](./srs.md#srs-fun-24), [SRS-NFR-REL-02](./srs.md#srs-nfr-rel-02) |
| Event Bus | Apache Kafka 4.x (KRaft) | Топіки `course.events`, `order.events`, `*.dlq`; 3 партиції, ключ `aggregateId`. Рішення — [ADR-001](./adr/0001-choice-of-technology.md). | [SRS-FUN-19](./srs.md#srs-fun-19), [SRS-FUN-20](./srs.md#srs-fun-20), [SRS-NFR-REL-01](./srs.md#srs-nfr-rel-01), [SRS-NFR-SCAL-02](./srs.md#srs-nfr-scal-02) |
| Write DB | PostgreSQL 16 | Окрема схема та окремий користувач БД для кожного сервісу (database-per-service на рівні схем). | [SRS-NFR-REL-05](./srs.md#srs-nfr-rel-05) |
| Read Model / Cache | Redis 7 | Read-модель каталогу (Hash + Sorted Set для фільтрів), ключі ідемпотентності з TTL 7 діб. | [SRS-FUN-10](./srs.md#srs-fun-10), [SRS-NFR-REL-02](./srs.md#srs-nfr-rel-02) |
| Object Storage | MinIO (S3 API) | Приватний bucket з файлами матеріалів; доступ лише за presigned URL. | [SRS-FUN-08](./srs.md#srs-fun-08), [SRS-FUN-23](./srs.md#srs-fun-23) |
| Observability | Prometheus, Grafana | Збір метрик `/metrics` усіх сервісів і Kafka (consumer lag), дашборди для адміністратора. | [SRS-NFR-OBS-01](./srs.md#srs-nfr-obs-01), [SRS-NFR-PERF-04](./srs.md#srs-nfr-perf-04) |

#### Специфікація протоколів взаємодії

| Від → До | Протокол / порт | Примітка |
| :--- | :--- | :--- |
| Браузер → API Gateway | HTTPS (TLS 1.3) / 443, REST, JSON | Rate limiting: 20 req/s на IP; 5 req/хв на IP для `/api/auth/login`. |
| API Gateway → сервіси | HTTP/1.1 / 3000, REST, JSON | Внутрішня мережа Docker; заголовок `X-Correlation-Id`. |
| PSP → API Gateway | HTTPS POST / 443 | Тіло вебхука передається в Order Service без змін для перевірки HMAC. |
| Order Service → PSP | HTTPS / 443, REST, JSON | Timeout 3 с, 2 повтори, ключ ідемпотентності = `orderId`. |
| Сервіси → PostgreSQL | SQL / TCP 5432 (TypeORM, пул з'єднань) | — |
| Course, Order → Kafka | Kafka protocol / TCP 9092 | Лише через Outbox Relay; `acks=all`, idempotent producer. |
| Kafka → Catalog Query, Order, Access | Kafka protocol / TCP 9092 | Consumer groups `catalog-query`, `order-price-snapshot`, `access`. |
| Catalog Query, Access → Redis | RESP / TCP 6379 | — |
| Course, Access → MinIO | S3 API / HTTP 9000 | Upload (Course), генерація presigned URL (Access). |
| Браузер → MinIO | HTTPS GET, presigned URL | Строк дії 15 хв ([SRS-FUN-23](./srs.md#srs-fun-23)). |
| Сервіси → Identity Service | HTTP / JWKS (`/.well-known/jwks.json`) | Кешування публічного ключа на 10 хв; на діаграмі не показано для читабельності. |
| Prometheus → сервіси | HTTP / Prometheus exposition format | Scrape кожні 15 с; на діаграмі показано один зв'язок. |

---

<a id="diagram-level3"></a>

## 2.2. Компонентне моделювання критичного сервісу системи

**Обраний сервіс:** Order Service (NestJS, TypeScript). Обґрунтування вибору: сервіс одночасно виконує фінансову транзакцію, інтегрується із зовнішнім PSP, приймає вебхуки з неперевіреного периметра та є джерелом подій `OrderCreated` і `PaymentConfirmed`, від яких залежить видача доступу. Відмова або дефект цього сервісу безпосередньо порушує [SRS-NFR-REL-01](./srs.md#srs-nfr-rel-01) та [SRS-NFR-PERF-03](./srs.md#srs-nfr-perf-03).

```mermaid
flowchart TB
  gw["<b>API Gateway</b><br/>[Container: Nginx]"]
  kafkaIn[["<b>Event Bus</b><br/>[Apache Kafka]<br/>topic course.events"]]

  subgraph os["Order Service [Container: NestJS, TypeScript]"]
    jwt["<b>JwtAuthGuard + RolesGuard</b><br/>[Guard]<br/>перевірка RS256, RBAC: STUDENT"]
    sig["<b>PspSignatureGuard</b><br/>[Guard]<br/>HMAC-SHA256, timestamp ≤ 5 хв"]
    obsi["<b>CorrelationId & Metrics Interceptor</b><br/>[Interceptor]<br/>X-Correlation-Id, histogram"]
    val["<b>ValidationPipe</b><br/>[Pipe, class-validator]<br/>whitelist, forbidNonWhitelisted"]
    ctrl["<b>OrdersController</b><br/>[Inbound: REST Controller]<br/>POST /orders, GET /orders/:id"]
    wh["<b>PaymentWebhookController</b><br/>[Inbound: REST Controller]<br/>POST /webhooks/psp"]
    cons["<b>CourseEventsConsumer</b><br/>[Inbound: Kafka Consumer]<br/>оновлення знімка ціни"]
    err["<b>GlobalExceptionFilter</b><br/>[Exception Filter]<br/>DomainError → 400/404/409,<br/>інші → 500 без stack trace"]

    bus["<b>CommandBus</b><br/>[Application: Mediator, @nestjs/cqrs]"]
    create["<b>CreateOrderHandler</b><br/>[Application: Command Handler]"]
    confirm["<b>ConfirmPaymentHandler</b><br/>[Application: Command Handler]"]
    expire["<b>ExpireOrdersJob</b><br/>[Application: Scheduler, 1 хв]"]

    agg["<b>Order Aggregate</b><br/>[Domain: Entity]<br/>PENDING → PAID / EXPIRED,<br/>генерує доменні події"]
    ports["<b>Ports</b><br/>[Domain: Interfaces]<br/>IOrderRepository, IUnitOfWork,<br/>IPaymentGateway, ICoursePriceReader"]

    uow["<b>TypeOrmUnitOfWork</b><br/>[Outbound Adapter]<br/>order + outbox в одній транзакції"]
    pspa["<b>PspPaymentAdapter</b><br/>[Outbound Adapter: HTTP client]<br/>timeout 3 с, 2 повтори"]
    price["<b>CoursePriceReader</b><br/>[Outbound Adapter]<br/>таблиця course_snapshot"]
    relay["<b>OutboxRelay</b><br/>[Background Worker]<br/>poll 500 мс, batch 100"]
    producer["<b>KafkaProducerAdapter</b><br/>[Outbound Adapter: kafkajs]<br/>acks=all, idempotent=true"]
  end

  pg[("<b>Write DB</b><br/>[PostgreSQL 16, схема orders]")]
  psp["<b>PSP</b><br/>[External System]"]
  kafkaOut[["<b>Event Bus</b><br/>[Apache Kafka]<br/>topic order.events"]]
  identity["<b>Identity Service</b><br/>[JWKS endpoint]"]

  gw -- "POST /orders<br/>[REST / HTTP/1.1]" --> jwt
  gw -- "POST /webhooks/psp<br/>[REST / HTTP/1.1]" --> sig
  jwt -. "публічний ключ, кеш 10 хв<br/>[HTTP / JWKS]" .-> identity
  jwt --> obsi
  sig --> obsi
  obsi --> val
  val -- "CreateOrderDto" --> ctrl
  val -- "PspWebhookDto" --> wh
  kafkaIn -- "[Kafka protocol / TCP 9092]" --> cons

  ctrl -- "execute(CreateOrderCommand)" --> bus
  wh -- "execute(ConfirmPaymentCommand)" --> bus
  bus -- "dispatch" --> create & confirm
  ctrl & wh -. "виняток" .-> err

  create & confirm & expire -- "бізнес-правила" --> agg
  create & confirm & expire -- "DI через інтерфейси" --> ports
  cons -- "DI: ICoursePriceReader" --> ports

  uow & pspa & price -. "implements" .-> ports
  uow -- "BEGIN; INSERT order; INSERT outbox; COMMIT<br/>[SQL / TCP 5432]" --> pg
  price -- "[SQL / TCP 5432]" --> pg
  pspa -- "створення checkout-сесії<br/>[HTTPS / REST, JSON]" --> psp
  relay -- "SELECT … FOR UPDATE SKIP LOCKED<br/>[SQL / TCP 5432]" --> pg
  relay -- "publish(event)" --> producer
  producer -- "OrderCreated, PaymentConfirmed<br/>[Kafka protocol / TCP 9092]" --> kafkaOut

  classDef ext fill:#999,stroke:#6b6b6b,color:#fff
  classDef inb fill:#85bbf0,stroke:#5d82a8,color:#000
  classDef sec fill:#f6c26b,stroke:#b07d1e,color:#000
  classDef appl fill:#c9a6e8,stroke:#7c55a0,color:#000
  classDef dom fill:#9fd49b,stroke:#4f8a4b,color:#000
  classDef out fill:#d9d9d9,stroke:#8c8c8c,color:#000
  class gw,psp,kafkaIn,kafkaOut,pg,identity ext
  class ctrl,wh,cons inb
  class jwt,sig,val,err,obsi sec
  class bus,create,confirm,expire appl
  class agg,ports dom
  class uow,pspa,price,relay,producer out
```

### Опис компонентів

| Компонент | Шар | Роль | Взаємодія |
| :--- | :--- | :--- | :--- |
| JwtAuthGuard + RolesGuard | Cross-cutting (Guard) | Перевірка підпису RS256 і строку дії JWT; перевірка ролі `STUDENT` для `POST /orders`. | Декоратор `@UseGuards`; публічний ключ — з JWKS Identity Service ([SRS-FUN-04](./srs.md#srs-fun-04), [SRS-NFR-SEC-02](./srs.md#srs-nfr-sec-02)). |
| PspSignatureGuard | Cross-cutting (Guard) | Перевірка HMAC-SHA256 підпису та часової мітки вебхука; відхилення — HTTP 401. | `@UseGuards` на `PaymentWebhookController` ([SRS-NFR-SEC-05](./srs.md#srs-nfr-sec-05)). |
| CorrelationId & Metrics Interceptor | Cross-cutting (Interceptor) | Прокидання `X-Correlation-Id` у логи та заголовки подій; гістограма `http_request_duration_seconds`. | Глобальний `APP_INTERCEPTOR` ([SRS-NFR-OBS-01](./srs.md#srs-nfr-obs-01), [SRS-NFR-OBS-02](./srs.md#srs-nfr-obs-02)). |
| ValidationPipe | Cross-cutting (Pipe) | Валідація DTO (`class-validator`), відхилення невідомих полів — HTTP 400. | Глобальний pipe ([SRS-NFR-SEC-06](./srs.md#srs-nfr-sec-06)). |
| GlobalExceptionFilter | Cross-cutting (Filter) | Відображення доменних помилок у коди HTTP (`OrderAlreadyExists` → 409); приховування stack trace. | Глобальний `APP_FILTER` ([SRS-FUN-14](./srs.md#srs-fun-14)). |
| OrdersController | Inbound | REST-адаптер: перетворює DTO на `CreateOrderCommand`. Не містить бізнес-логіки й не звертається до БД. | `CommandBus.execute()` |
| PaymentWebhookController | Inbound | REST-адаптер вебхука: перетворює DTO на `ConfirmPaymentCommand`. | `CommandBus.execute()` |
| CourseEventsConsumer | Inbound | Kafka-консюмер `course.events`: оновлює локальний знімок ціни курсу (таблиця `course_snapshot`), що усуває синхронну залежність від Catalog Query Service ([SRS-NFR-REL-03](./srs.md#srs-nfr-rel-03)). | Порт `ICoursePriceReader` |
| CommandBus | Application | Mediator (`@nestjs/cqrs`): маршрутизація команд до обробників. | Патерн Mediator |
| CreateOrderHandler | Application | Сценарій створення замовлення: перевірка дубліката, фіксація ціни, збереження замовлення та події в одній транзакції, створення checkout-сесії PSP. | Порти `IUnitOfWork`, `ICoursePriceReader`, `IPaymentGateway` ([SRS-FUN-13](./srs.md#srs-fun-13), [SRS-FUN-15](./srs.md#srs-fun-15), [SRS-FUN-16](./srs.md#srs-fun-16)) |
| ConfirmPaymentHandler | Application | Перехід `PENDING → PAID` та запис `PaymentConfirmed` в Outbox; повторний вебхук для `PAID`-замовлення ігнорується. | Порт `IUnitOfWork` ([SRS-FUN-17](./srs.md#srs-fun-17)) |
| ExpireOrdersJob | Application | Переведення замовлень `PENDING` старших за 30 хв у `EXPIRED`. | `@nestjs/schedule` ([SRS-FUN-18](./srs.md#srs-fun-18)) |
| Order Aggregate | Domain | Інваріанти та переходи станів; генерує доменні події. Не залежить від NestJS, TypeORM і Kafka. | Чистий TypeScript |
| Ports | Domain | Інтерфейси `IOrderRepository`, `IUnitOfWork`, `IPaymentGateway`, `ICoursePriceReader`. | Dependency Inversion Principle: application-шар залежить від абстракцій, адаптери їх реалізують. |
| TypeOrmUnitOfWork | Outbound | Транзакція `order + outbox` через `QueryRunner` ([SRS-FUN-15](./srs.md#srs-fun-15)). | SQL / TCP 5432 |
| PspPaymentAdapter | Outbound | HTTP-клієнт PSP: timeout 3 с, 2 повтори, `Idempotency-Key`. Викликається після коміту транзакції, щоб мережевий виклик не утримував з'єднання з БД. | HTTPS / REST |
| CoursePriceReader | Outbound | Читання знімка ціни з `course_snapshot`. | SQL / TCP 5432 |
| OutboxRelay | Background Worker | Вибірка неопублікованих записів `SELECT … FOR UPDATE SKIP LOCKED` (безпечно для кількох реплік), публікація, позначка `published_at` ([SRS-FUN-19](./srs.md#srs-fun-19)). | SQL / TCP 5432 |
| KafkaProducerAdapter | Outbound | Продюсер `kafkajs` з `acks=all`, `idempotent=true`, ключ партиції `orderId`. | Kafka protocol / TCP 9092 ([ADR-001](./adr/0001-choice-of-technology.md)) |

### Принципи, дотримані на рівні компонентів

- **Clean Architecture:** залежності спрямовані всередину (Inbound → Application → Domain ← Outbound). Контролери не мають доступу до репозиторіїв.
- **Dependency Inversion (SOLID-D):** обробники отримують порти через DI-контейнер NestJS (`@Inject(ORDER_REPOSITORY)`); адаптери TypeORM, PSP та Kafka замінюються в тестах на in-memory реалізації.
- **Single Responsibility (SOLID-S):** публікація подій винесена з обробників у `OutboxRelay`; обробник не знає про Kafka.
- **Request pipeline NestJS:** Guard → Interceptor → Pipe → Controller → Exception Filter.

---

## 2.3. Документування архітектурних рішень (Architecture Decision Records)

| ID | Рішення | Статус | Пов'язані вимоги |
| :--- | :--- | :--- | :--- |
| [ADR-001](./adr/0001-choice-of-technology.md) | Вибір брокера доменних подій — Apache Kafka замість RabbitMQ | Approved | [SRS-FUN-10](./srs.md#srs-fun-10), [SRS-FUN-19](./srs.md#srs-fun-19), [SRS-FUN-20](./srs.md#srs-fun-20), [SRS-NFR-REL-01](./srs.md#srs-nfr-rel-01), [SRS-NFR-REL-02](./srs.md#srs-nfr-rel-02) |

---

## 2.4. Матриця трасування «Вимога → Архітектурний елемент»

| Вимога | Контейнер (Level 2) | Компонент (Level 3) | Рішення |
| :--- | :--- | :--- | :--- |
| [SRS-FUN-04](./srs.md#srs-fun-04) | Identity Service, усі сервіси | JwtAuthGuard + RolesGuard | — |
| [SRS-FUN-10](./srs.md#srs-fun-10) | Catalog Query Service, Redis, Kafka | — | [ADR-001](./adr/0001-choice-of-technology.md) |
| [SRS-FUN-13](./srs.md#srs-fun-13) | Order Service | CreateOrderHandler, Order Aggregate | — |
| [SRS-FUN-14](./srs.md#srs-fun-14) | Order Service | CreateOrderHandler, GlobalExceptionFilter | — |
| [SRS-FUN-15](./srs.md#srs-fun-15) | Order Service, PostgreSQL | TypeOrmUnitOfWork | [ADR-001](./adr/0001-choice-of-technology.md) |
| [SRS-FUN-16](./srs.md#srs-fun-16) | Order Service, PSP | PspPaymentAdapter | — |
| [SRS-FUN-17](./srs.md#srs-fun-17) | Order Service | PaymentWebhookController, PspSignatureGuard, ConfirmPaymentHandler | — |
| [SRS-FUN-18](./srs.md#srs-fun-18) | Order Service | ExpireOrdersJob | — |
| [SRS-FUN-19](./srs.md#srs-fun-19) | Order Service, Kafka | OutboxRelay, KafkaProducerAdapter | [ADR-001](./adr/0001-choice-of-technology.md) |
| [SRS-FUN-22](./srs.md#srs-fun-22) | Access Service, Redis, PostgreSQL | — | [ADR-001](./adr/0001-choice-of-technology.md) |
| [SRS-NFR-SEC-04](./srs.md#srs-nfr-sec-04) | API Gateway | — | — |
| [SRS-NFR-SEC-05](./srs.md#srs-nfr-sec-05) | Order Service | PspSignatureGuard | — |
| [SRS-NFR-SEC-06](./srs.md#srs-nfr-sec-06) | Усі бекенд-сервіси | ValidationPipe | — |
| [SRS-NFR-OBS-02](./srs.md#srs-nfr-obs-02) | API Gateway, усі сервіси | CorrelationId & Metrics Interceptor | — |
