# Специфікація вимог до програмного забезпечення (SRS)

> **Стандарт:** ISO/IEC/IEEE 29148:2018 (Systems and software engineering — Life cycle processes — Requirements engineering)
> **Система:** Розподілена платформа електронної комерції цифрових товарів та курсів на основі патерна CQRS та подієво-керованої архітектури
> **Версія документа:** 1.0 (після ШІ-аудиту) · **Статус:** Baseline · **Автор:** Журик М. І.
> **Пов'язані документи:** [Розділ 1 (вимоги To-Be, ЛР№4)](./1_analysis.md.txt) · [Розділ 2 (архітектура C4)](./2_design.md) · [ADR-001](./adr/0001-choice-of-technology.md)

---

## Зміст

1. [Вступ](#1-вступ)
2. [System Context (Контекст системи)](#2-system-context-контекст-системи)
3. [Functional Requirements (Функціональні вимоги)](#3-functional-requirements-функціональні-вимоги)
4. [Non-Functional Requirements (Нефункціональні вимоги)](#4-non-functional-requirements-нефункціональні-вимоги)
5. [Verification Criteria (Критерії верифікації)](#5-verification-criteria-критерії-верифікації)
6. [Матриця трасування вимог](#6-матриця-трасування-вимог)
7. [Glossary (Глосарій)](#7-glossary-глосарій)

---

## 1. Вступ

### 1.1. Призначення документа

Документ фіксує вимоги до програмної системи у формі технічного контракту між замовником (науковим керівником, екзаменаційною комісією) та розробником. Кожна вимога має унікальний ідентифікатор, модальне дієслово «повинна» / «має», один спосіб верифікації та трасується в архітектурні елементи [Розділу 2](./2_design.md).

### 1.2. Правила формулювання вимог

- Одна вимога описує одну системну дію або одне вимірюване обмеження (атомарність).
- Ідентифікатори: `SRS-FUN-NN` — функціональні; `SRS-NFR-<КАТЕГОРІЯ>-NN` — нефункціональні (`PERF`, `SCAL`, `REL`, `SEC`, `OBS`, `MAINT`, `ENV`).
- Числові значення NFR вимірюються в еталонному середовищі, визначеному у [SRS-NFR-ENV-01](#srs-nfr-env-01).

### 1.3. Межі версії (Out of Scope)

Не входять до обсягу кваліфікаційної роботи: мобільні клієнти, відеостримінг з адаптивним бітрейтом, мультивалютність, розгортання у Kubernetes, багатоброкерний кластер Kafka, реплікація PostgreSQL. Обмеження зумовлені локальним середовищем розгортання (Docker Compose, один хост) та обсягом бакалаврського проєкту.

---

## 2. System Context (Контекст системи)

### 2.1. Межі системи (Scope)

Система реалізує повний цикл продажу цифрових курсів: реєстрацію та автентифікацію користувачів, публікацію курсів авторами, пошук у каталозі, оформлення замовлення, приймання підтвердження оплати від зовнішнього платіжного провайдера та автоматичну видачу доступу до навчальних матеріалів.

Внутрішній периметр системи охоплює клієнтський SPA-застосунок, API Gateway, бекенд-мікросервіси (Identity, Course, Catalog Query, Order, Access), брокер подій Apache Kafka, сховища PostgreSQL, Redis та об'єктне сховище MinIO, а також контур спостережуваності (Prometheus, Grafana).

Поза периметром системи перебувають: платіжний провайдер (PSP) та браузер кінцевого користувача.

### 2.2. Актори

| Актор | Тип | Взаємодія з системою |
| :--- | :--- | :--- |
| Студент / Покупець | Person | Пошук курсів, оформлення замовлення, отримання доступу до матеріалів через Web SPA (HTTPS). |
| Автор курсу / Інструктор | Person | Створення та редагування курсів, завантаження матеріалів через Web SPA (HTTPS). |
| Платформний адміністратор | Person | Керування ролями, перегляд стану Outbox/DLQ, моніторинг метрик через Web SPA та Grafana. |
| Платіжний провайдер (PSP) | Зовнішня система | Hosted Checkout-сторінка оплати; асинхронне повідомлення про результат платежу через підписаний вебхук (HTTPS POST). Для розробки використовується sandbox-режим провайдера. |

### 2.3. Зовнішні інтерфейси

| ID | Інтерфейс | Протокол / формат |
| :--- | :--- | :--- |
| IF-01 | Web SPA ↔ API Gateway | REST, JSON, HTTPS (TLS 1.3) |
| IF-02 | Order Service → PSP (створення checkout-сесії) | REST, JSON, HTTPS |
| IF-03 | PSP → API Gateway (вебхук результату оплати) | HTTPS POST, JSON, підпис HMAC-SHA256 |
| IF-04 | Браузер → MinIO (завантаження матеріалів) | HTTPS GET за presigned URL (S3 API) |

### 2.4. Припущення та залежності

- PSP підтримує hosted checkout і вебхуки з HMAC-підписом; дані платіжних карток не проходять через периметр системи (див. [SRS-NFR-SEC-07](#srs-nfr-sec-07)).
- Час на хостах синхронізується через NTP (необхідно для перевірки строку дії JWT і presigned URL).

---

## 3. Functional Requirements (Функціональні вимоги)

### 3.1. Модуль автентифікації та керування доступом (Identity Service)

| ID | Вимога | Пріоритет |
| :--- | :--- | :--- |
| <a id="srs-fun-01"></a>SRS-FUN-01 | Система повинна реєструвати обліковий запис за e-mail та паролем і призначати йому роль `STUDENT`. | Must |
| <a id="srs-fun-02"></a>SRS-FUN-02 | Система повинна видавати пару токенів (access JWT RS256, refresh token) після успішної перевірки облікових даних. | Must |
| <a id="srs-fun-03"></a>SRS-FUN-03 | Система повинна видавати нову пару токенів за дійсним refresh token та анулювати використаний refresh token (ротація). | Must |
| <a id="srs-fun-04"></a>SRS-FUN-04 | Система повинна відхиляти запит із кодом HTTP 403, якщо роль у JWT не входить до переліку ролей, дозволених для кінцевої точки (RBAC: `STUDENT`, `AUTHOR`, `ADMIN`). | Must |
| <a id="srs-fun-05"></a>SRS-FUN-05 | Система повинна анонімізувати персональні дані користувача (e-mail, ім'я) за запитом на видалення облікового запису зі збереженням знеособлених фінансових записів. | Must |
| <a id="srs-fun-06"></a>SRS-FUN-06 | Система повинна надавати адміністратору можливість змінювати роль облікового запису. | Should |

### 3.2. Модуль каталогу (Course Service — Command Side, Catalog Query Service — Query Side)

| ID | Вимога | Пріоритет |
| :--- | :--- | :--- |
| <a id="srs-fun-07"></a>SRS-FUN-07 | Система повинна створювати та оновлювати курс (назва, опис, категорія, теги, ціна, перелік уроків) за запитом користувача з роллю `AUTHOR`. | Must |
| <a id="srs-fun-08"></a>SRS-FUN-08 | Система повинна завантажувати файли матеріалів курсу в об'єктне сховище та зберігати їхні ключі в моделі курсу. | Must |
| <a id="srs-fun-09"></a>SRS-FUN-09 | Система повинна публікувати подію `CourseUpserted` у Kafka після кожної успішної зміни курсу. | Must |
| <a id="srs-fun-10"></a>SRS-FUN-10 | Система повинна оновлювати read-модель каталогу в Redis на основі подій `CourseUpserted`. | Must |
| <a id="srs-fun-11"></a>SRS-FUN-11 | Система повинна повертати сторінку списку курсів (курсорна пагінація, до 50 елементів) з фільтрацією за категорією, тегом, автором і діапазоном ціни. | Must |
| <a id="srs-fun-12"></a>SRS-FUN-12 | Система повинна повертати картку курсу за ідентифікатором з read-моделі без звернення до PostgreSQL. | Must |

### 3.3. Модуль замовлень (Order Service — Command Side)

| ID | Вимога | Пріоритет |
| :--- | :--- | :--- |
| <a id="srs-fun-13"></a>SRS-FUN-13 | Система повинна створювати замовлення у статусі `PENDING` з фіксацією ціни курсу на момент створення. | Must |
| <a id="srs-fun-14"></a>SRS-FUN-14 | Система повинна відхиляти створення замовлення з кодом HTTP 409, якщо користувач уже має доступ до курсу або активне замовлення `PENDING` на цей курс. | Must |
| <a id="srs-fun-15"></a>SRS-FUN-15 | Система повинна записувати подію `OrderCreated` у таблицю Outbox у межах тієї самої транзакції PostgreSQL, що й замовлення (Transactional Outbox). | Must |
| <a id="srs-fun-16"></a>SRS-FUN-16 | Система повинна створювати checkout-сесію у PSP і повертати клієнту URL переадресації на сторінку оплати. | Must |
| <a id="srs-fun-17"></a>SRS-FUN-17 | Система повинна переводити замовлення у статус `PAID` і записувати подію `PaymentConfirmed` в Outbox після отримання вебхука з дійсним підписом. | Must |
| <a id="srs-fun-18"></a>SRS-FUN-18 | Система повинна переводити замовлення у статус `EXPIRED`, якщо підтвердження оплати не надійшло протягом 30 хвилин. | Should |

### 3.4. Асинхронний конвеєр подій (Kafka Pipeline)

| ID | Вимога | Пріоритет |
| :--- | :--- | :--- |
| <a id="srs-fun-19"></a>SRS-FUN-19 | Система повинна публікувати неопубліковані записи Outbox у відповідні топіки Kafka з ключем партиції `aggregateId`. | Must |
| <a id="srs-fun-20"></a>SRS-FUN-20 | Система повинна повторювати обробку події у консюмері до 5 разів з експоненційною затримкою (1, 2, 4, 8, 16 с), після чого переміщувати подію в DLQ-топік `<topic>.dlq`. | Must |

### 3.5. Модуль активації доступу (Access Service)

| ID | Вимога | Пріоритет |
| :--- | :--- | :--- |
| <a id="srs-fun-21"></a>SRS-FUN-21 | Система повинна створювати запис доступу (Access Grant) до курсу після отримання події `PaymentConfirmed`. | Must |
| <a id="srs-fun-22"></a>SRS-FUN-22 | Система повинна ігнорувати повторну доставку події з тим самим `eventId` без створення дубліката Access Grant (Idempotent Consumer). | Must |
| <a id="srs-fun-23"></a>SRS-FUN-23 | Система повинна видавати presigned URL до файлу матеріалу зі строком дії 15 хвилин лише користувачу з активним Access Grant на цей курс. | Must |
| <a id="srs-fun-24"></a>SRS-FUN-24 | Система повинна повертати перелік курсів, до яких користувач має доступ. | Must |

### 3.6. Адміністративний модуль

| ID | Вимога | Пріоритет |
| :--- | :--- | :--- |
| <a id="srs-fun-25"></a>SRS-FUN-25 | Система повинна відображати адміністратору кількість неопублікованих записів Outbox та подій у DLQ-топіках. | Should |
| <a id="srs-fun-26"></a>SRS-FUN-26 | Система повинна повторно публікувати подію з DLQ у вихідний топік за командою адміністратора. | Should |

---

## 4. Non-Functional Requirements (Нефункціональні вимоги)

### 4.1. Продуктивність (PERF)

| ID | Метрика | Цільове значення | Умови вимірювання |
| :--- | :--- | :--- | :--- |
| <a id="srs-nfr-perf-01"></a>SRS-NFR-PERF-01 | Пропускна здатність API Gateway | ≥ 600 RPS протягом 10 хв, частка помилок HTTP 5xx < 1 % | Профіль: 80 % `GET /catalog/*`, 20 % `POST /orders`; ENV-01 |
| <a id="srs-nfr-perf-02"></a>SRS-NFR-PERF-02 | Час відгуку Query API (`GET /catalog/*`) | p95 ≤ 120 мс | Навантаження згідно з PERF-01 |
| <a id="srs-nfr-perf-03"></a>SRS-NFR-PERF-03 | Час відгуку Command API (`POST /orders`, без часу відповіді PSP) | p95 ≤ 250 мс | Навантаження згідно з PERF-01; PSP замінено mock-сервером |
| <a id="srs-nfr-perf-04"></a>SRS-NFR-PERF-04 | Наскрізна затримка `PaymentConfirmed` → Access Grant | p95 ≤ 2 с | 100 подій/с |
| <a id="srs-nfr-perf-05"></a>SRS-NFR-PERF-05 | Затримка оновлення read-моделі (staleness) після `CourseUpserted` | p95 ≤ 5 с | Без навантаження та під навантаженням PERF-01 |

### 4.2. Масштабованість (SCAL)

| ID | Вимога | Цільове значення |
| :--- | :--- | :--- |
| <a id="srs-nfr-scal-01"></a>SRS-NFR-SCAL-01 | Сервіси Catalog Query та Access повинні бути stateless і запускатися у кількох репліках без змін коду. | ≥ 2 репліки (`docker compose up --scale`) |
| <a id="srs-nfr-scal-02"></a>SRS-NFR-SCAL-02 | Топіки доменних подій повинні мати кількість партицій, достатню для паралельної обробки консюмерами. | ≥ 3 партиції на топік |

### 4.3. Надійність та відмовостійкість (REL)

| ID | Вимога | Цільове значення |
| :--- | :--- | :--- |
| <a id="srs-nfr-rel-01"></a>SRS-NFR-REL-01 | Гарантія доставки доменних подій — At-Least-Once. Недоступність Kafka не повинна призводити до втрати подій. | 0 втрачених подій при простої брокера до 5 хв |
| <a id="srs-nfr-rel-02"></a>SRS-NFR-REL-02 | Повторна доставка події не повинна створювати дублікатів бізнес-записів. Ідемпотентність забезпечується ключем у Redis (TTL 7 діб) та обмеженням `UNIQUE(event_id)` у PostgreSQL. | 0 дублікатів при 1000 повторних доставках |
| <a id="srs-nfr-rel-03"></a>SRS-NFR-REL-03 | Відмова Catalog Query Service не повинна блокувати прийом вебхуків та видачу доступу. | Ізоляція відмов між контурами Query та Command |
| <a id="srs-nfr-rel-04"></a>SRS-NFR-REL-04 | Кожен контейнер повинен надавати кінцеві точки `/health/live` та `/health/ready` і мати політику перезапуску `restart: unless-stopped`. | 100 % сервісів |
| <a id="srs-nfr-rel-05"></a>SRS-NFR-REL-05 | Резервне копіювання PostgreSQL (`pg_dump`) щодоби. | RPO ≤ 24 год; RTO ≤ 1 год |

### 4.4. Безпека та комплаєнс (SEC)

| ID | Вимога | Цільове значення |
| :--- | :--- | :--- |
| <a id="srs-nfr-sec-01"></a>SRS-NFR-SEC-01 | Зовнішній трафік повинен шифруватися на API Gateway. | Лише TLS 1.3 |
| <a id="srs-nfr-sec-02"></a>SRS-NFR-SEC-02 | Access token має бути JWT з підписом RS256; публічний ключ публікується через JWKS-endpoint Identity Service. | TTL access — 15 хв; TTL refresh — 7 діб |
| <a id="srs-nfr-sec-03"></a>SRS-NFR-SEC-03 | Паролі повинні зберігатися лише у вигляді хешу Argon2id. | 0 паролів у відкритому вигляді |
| <a id="srs-nfr-sec-04"></a>SRS-NFR-SEC-04 | API Gateway повинен обмежувати частоту запитів (Rate Limiting) та повертати HTTP 429 при перевищенні. | 20 req/s на IP (загальний ліміт); 5 req/хв на IP для `POST /auth/login` |
| <a id="srs-nfr-sec-05"></a>SRS-NFR-SEC-05 | Вебхук PSP повинен прийматися лише з дійсним підписом HMAC-SHA256 та часовою міткою не старшою за 5 хв. | 100 % запитів без дійсного підпису відхиляються (HTTP 401) |
| <a id="srs-nfr-sec-06"></a>SRS-NFR-SEC-06 | Вхідні DTO повинні валідуватися за схемою; невідомі поля відхиляються. | HTTP 400 для 100 % невалідних запитів |
| <a id="srs-nfr-sec-07"></a>SRS-NFR-SEC-07 | Система не повинна приймати, обробляти чи зберігати дані платіжних карток (PCI DSS: використання hosted checkout PSP). | 0 полів PAN/CVV у БД, логах та API |
| <a id="srs-nfr-sec-08"></a>SRS-NFR-SEC-08 | Збірка не повинна містити відомих вразливостей високого рівня у залежностях та образах контейнерів. | 0 Critical/High (Trivy); SonarQube Quality Gate — Passed |
| <a id="srs-nfr-sec-09"></a>SRS-NFR-SEC-09 | GDPR: запит на видалення облікового запису повинен виконуватися без невиправданої затримки; журнали не повинні містити персональних даних. | Анонімізація ≤ 30 діб (у реалізації — синхронно); 0 e-mail у логах |

### 4.5. Спостережуваність (OBS)

| ID | Вимога | Цільове значення |
| :--- | :--- | :--- |
| <a id="srs-nfr-obs-01"></a>SRS-NFR-OBS-01 | Кожен бекенд-сервіс повинен експортувати метрики у форматі Prometheus (`/metrics`): кількість запитів, помилки, гістограма тривалості, consumer lag. | Інтервал scrape — 15 с |
| <a id="srs-nfr-obs-02"></a>SRS-NFR-OBS-02 | Журнали повинні мати структурований формат JSON із полем `correlationId`, що передається через HTTP-заголовок `X-Correlation-Id` та заголовки повідомлень Kafka. | 100 % запитів і подій |

### 4.6. Супровідність (MAINT) та середовище (ENV)

| ID | Вимога | Цільове значення |
| :--- | :--- | :--- |
| <a id="srs-nfr-maint-01"></a>SRS-NFR-MAINT-01 | Покриття коду модульними тестами для шарів domain та application. | ≥ 70 % рядків |
| <a id="srs-nfr-maint-02"></a>SRS-NFR-MAINT-02 | Статичний аналіз коду без помилок. | 0 errors ESLint; 0 Blocker/Critical issues SonarQube |
| <a id="srs-nfr-env-01"></a>SRS-NFR-ENV-01 | Еталонне середовище вимірювання NFR: Docker Compose на одному хості, 8 vCPU, 16 ГБ RAM, SSD; Kafka — 1 брокер у режимі KRaft. | Фіксується у звіті тестування |

---

## 5. Verification Criteria (Критерії верифікації)

| Вимога | Метод (ISO 29148) | Інструмент | Критерій проходження |
| :--- | :--- | :--- | :--- |
| SRS-FUN-01…03 | Test | Jest + Supertest (e2e) | Коректні коди HTTP 201/200/401; ротація refresh token підтверджена тестом повторного використання |
| SRS-FUN-04 | Test | Jest + Supertest, Postman/Newman | Матриця «роль × endpoint»: усі заборонені комбінації повертають 403 |
| SRS-FUN-05, SRS-NFR-SEC-09 | Test, Inspection | Jest (integration, Testcontainers PostgreSQL); SQL-запит перевірки | Після видалення в БД відсутні e-mail та ім'я; записи замовлень збережено |
| SRS-FUN-07…12 | Test | Jest + Supertest; Testcontainers (Kafka, Redis) | Зміна курсу відображається у `GET /catalog` після обробки події |
| SRS-FUN-13…18 | Test | Jest (unit domain), Supertest (e2e), mock-сервер PSP (WireMock) | Переходи станів `PENDING → PAID / EXPIRED` відповідають діаграмі станів |
| SRS-FUN-15, SRS-NFR-REL-01 | Test (fault injection) | Testcontainers; `docker stop kafka` на 5 хв під час потоку замовлень | Кількість подій у Kafka після відновлення = кількість замовлень |
| SRS-FUN-20 | Test | Jest + Testcontainers (Kafka) | Після 5 невдалих спроб подія з'являється в `<topic>.dlq` |
| SRS-FUN-22, SRS-NFR-REL-02 | Test | Скрипт повторної публікації 1000 подій з однаковим `eventId` | Кількість Access Grant = 1 |
| SRS-FUN-23 | Test | Supertest | URL недійсний після 15 хв; користувач без доступу отримує 403 |
| SRS-NFR-PERF-01…03 | Test (load) | k6 (сценарії `constant-arrival-rate` та Spike Testing) | Thresholds k6: `http_req_duration{type:query} p(95)<120`, `{type:command} p(95)<250`, `http_req_failed<0.01` |
| SRS-NFR-PERF-04, PERF-05 | Analysis | Prometheus histogram `event_processing_latency_seconds`, Grafana | p95 у межах значень за 10 хв вимірювання |
| SRS-NFR-SCAL-01, SCAL-02 | Demonstration, Inspection | `docker compose up --scale`; `kafka-topics.sh --describe` | Сервіси працюють у 2 репліках; кількість партицій ≥ 3 |
| SRS-NFR-REL-04 | Inspection, Test | `docker compose ps`; health-check у CI | Усі контейнери у стані `healthy` |
| SRS-NFR-REL-05 | Demonstration | Скрипт `pg_dump` / `pg_restore` | Відновлення бази з копії ≤ 1 год |
| SRS-NFR-SEC-01 | Test | `testssl.sh`, OWASP ZAP baseline scan | TLS 1.2 та нижче відхиляються; 0 High alerts ZAP |
| SRS-NFR-SEC-02, SEC-03 | Test, Inspection | Jest; перевірка схеми БД | Алгоритм у заголовку JWT — RS256; хеш паролів має префікс `$argon2id$` |
| SRS-NFR-SEC-04 | Test | k6 (сплеск 50 req/s з одного IP) | Відповіді понад ліміт — HTTP 429 |
| SRS-NFR-SEC-05, SEC-06 | Test | Jest + Supertest (негативні сценарії) | Невалідний підпис → 401; невалідний DTO → 400 |
| SRS-NFR-SEC-07 | Inspection | Code review, пошук за шаблонами (grep/Semgrep) | Відсутні поля PAN/CVV |
| SRS-NFR-SEC-08, MAINT-02 | Analysis | Trivy, SonarQube, ESLint у GitHub Actions | Pipeline зелений, Quality Gate — Passed |
| SRS-NFR-OBS-01, OBS-02 | Inspection | Prometheus targets, Grafana dashboard, вибірка логів | Усі targets у стані UP; `correlationId` присутній у логах трьох сервісів одного запиту |
| SRS-NFR-MAINT-01 | Analysis | Jest `--coverage` у CI | Coverage ≥ 70 % для domain та application |

---

## 6. Матриця трасування вимог

| Вимога | Контейнер (C4 Level 2) | Архітектурне рішення |
| :--- | :--- | :--- |
| SRS-FUN-01…06 | Identity Service, PostgreSQL | — |
| SRS-FUN-07…09 | Course Service, PostgreSQL, MinIO, Kafka | [ADR-001](./adr/0001-choice-of-technology.md) |
| SRS-FUN-10…12 | Catalog Query Service, Redis, Kafka | [ADR-001](./adr/0001-choice-of-technology.md) (replay для перебудови read-моделі) |
| SRS-FUN-13…18 | Order Service, PostgreSQL, PSP | — |
| SRS-FUN-19…20 | Order Service (Outbox Relay), Kafka | [ADR-001](./adr/0001-choice-of-technology.md) |
| SRS-FUN-21…24 | Access Service, PostgreSQL, Redis, MinIO | — |
| SRS-FUN-25…26 | Web SPA (Admin), Order Service, Kafka | [ADR-001](./adr/0001-choice-of-technology.md) |
| SRS-NFR-SEC-01, SEC-04 | API Gateway (Nginx) | — |
| SRS-NFR-OBS-01 | Prometheus, Grafana | — |

Детальне трасування на рівні контейнерів і компонентів наведено в [Розділі 2](./2_design.md).

---

## 7. Glossary (Глосарій)

| Термін | Визначення |
| :--- | :--- |
| **CQRS** (Command Query Responsibility Segregation) | Архітектурний патерн, за яким операції зміни стану (Command) та операції читання (Query) обслуговуються окремими моделями даних і сервісами, що масштабуються незалежно. |
| **Eventual Consistency** | Модель узгодженості, за якої read-модель досягає стану write-моделі через обмежений проміжок часу після обробки подій (див. [SRS-NFR-PERF-05](#srs-nfr-perf-05)). |
| **Transactional Outbox** | Патерн атомарного збереження бізнес-даних і доменної події в одній локальній транзакції БД з подальшою асинхронною публікацією події в брокер. |
| **Idempotent Consumer** | Консюмер, повторна обробка повідомлення з тим самим `eventId` яким не змінює стан системи. |
| **JWT (RS256)** | JSON Web Token, підписаний асиметричним алгоритмом RSA-SHA256: підпис створює лише Identity Service (приватний ключ), перевіряють усі сервіси (публічний ключ). |
| **p95 Latency** | Значення часу відгуку, яке не перевищують 95 % запитів у вибірці вимірювання. |
| **DLQ** (Dead Letter Queue) | Окремий топік для повідомлень, які не вдалося обробити після вичерпання ліміту повторних спроб. |
