# Навігаційний дашборд документації дипломного проєкту

**Тема:** Розподілена платформа електронної комерції цифрових товарів та курсів на основі патерна CQRS та подієво-керованої архітектури
**Автор:** Журик М. І. · **Науковий керівник:** Томюк Г. М. · **Спеціальність:** F2 (121) «Інженерія програмного забезпечення»

## Навігація

| Документ | Зміст | Стан |
| :--- | :--- | :--- |
| [`proposal.md.txt`](./proposal.md.txt) | Thesis Proposal (ЛР№1) | Done |
| [`1_analysis.md.txt`](./1_analysis.md.txt) | Розділ 1: аналіз аналогів, модель As-Is / To-Be, Core Features, NFRs | Done |
| [`srs.md`](./srs.md) | Специфікація вимог за ISO/IEC/IEEE 29148 (System Context, FR, NFR, Verification Criteria, Glossary) | Done |
| [`2_design.md`](./2_design.md) | Розділ 2: C4 Model Levels 1–3 (Mermaid), трасування вимог | In Progress |
| [`adr/`](./adr/) | Architecture Decision Records | In Progress |
| [`adr/0001-choice-of-technology.md`](./adr/0001-choice-of-technology.md) | ADR-001: Apache Kafka замість RabbitMQ | Approved |

## Календарний план виконання кваліфікаційної роботи

| № | Назва етапу роботи | Термін виконання | Статус |
| :--- | :--- | :--- | :--- |
| 1 | Аналітичний огляд предметної області та аналіз аналогів за ДСТУ | Жовтень 2026 | Done |
| 2 | Обґрунтування технологічного стеку та розробка ТЗ | Листопад 2026 | Done |
| 3 | Архітектурне проєктування мікросервісів (UML, C4 Model) | Грудень 2026 | In Progress |
| 4 | Алгоритмізація контурів CQRS та взаємодії з Kafka | Січень – Лютий 2027 | To Do |
| 5 | Програмна реалізація бекенду та клієнтського інтерфейсу | Березень 2027 | To Do |
| 6 | Інструментальний аудит коду, навантажувальне тестування | Квітень 2027 | To Do |
| 7 | Апробація результатів, підготовка наукової публікації | Квітень 2027 | To Do |

## Ключові нефункціональні вимоги (NFRs Matrix)

Повна матриця — [SRS, розділ 4](./srs.md#4-non-functional-requirements-нефункціональні-вимоги).

| ID | Метрика | Цільове значення |
| :--- | :--- | :--- |
| [SRS-NFR-PERF-01](./srs.md#srs-nfr-perf-01) | Пропускна здатність | ≥ 600 RPS, 5xx < 1 % |
| [SRS-NFR-PERF-02](./srs.md#srs-nfr-perf-02) | Query API latency | p95 ≤ 120 мс |
| [SRS-NFR-PERF-03](./srs.md#srs-nfr-perf-03) | Command API latency | p95 ≤ 250 мс |
| [SRS-NFR-REL-01](./srs.md#srs-nfr-rel-01) | Доставка подій | At-Least-Once, 0 втрат при простої Kafka ≤ 5 хв |
| [SRS-NFR-SEC-01](./srs.md#srs-nfr-sec-01) | Шифрування трафіку | TLS 1.3 |
| [SRS-NFR-SEC-02](./srs.md#srs-nfr-sec-02) | Автентифікація | JWT RS256, access TTL 15 хв |
