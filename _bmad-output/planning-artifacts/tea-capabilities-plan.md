---
title: План возможностей TEA (Test Architecture)
description: Справочник возможностей TEA и рекомендации когда подключать
author: Чапаев
date: 2026-02-28
status: reference
depends_on: documentation-plan.md
---

# План возможностей TEA (Test Architecture)

## Что такое TEA

TEA (Test Architecture Enterprise) — модуль для проектирования тестовой стратегии, автоматизации тестов и оценки качества. Агент — Murat 🧪 Master Test Architect.

## Когда подключать TEA

**Не на этапе документации.** TEA подключается после реализации фичи провайдеров.

Исключение: Test Design (TD) может работать до кода — из документации проектирует тестовую стратегию. Но в текущем плане это не приоритет.

---

## Все workflow TEA по фазам

### До написания кода (работают с документами)

| Workflow | Код | Команда | Что делает |
|----------|-----|---------|-----------|
| **Teach Me Testing** | TMT | `/bmad-tea-teach-me-testing` | Обучение команды тестированию: 7 сессий от основ до продвинутых паттернов. Самостоятельный курс, не требует проекта |
| **Test Design** | TD | `/bmad-tea-testarch-test-design` | Проектирование тестовой стратегии из PRD и архитектурных документов. Выявляет риски и пробелы в тестируемости. Создаёт два документа: для архитекторов и для QA |
| **Test Framework** | TF | `/bmad-tea-testarch-framework` | Инициализация Playwright или Cypress: scaffold, фикстуры, конфигурация, хелперы |
| **CI Setup** | CI | `/bmad-tea-testarch-ci` | Настройка CI/CD pipeline с quality gates: GitHub Actions, GitLab CI, Jenkins, Azure DevOps |

### После написания кода (требуют репозиторий)

| Workflow | Код | Команда | Что делает |
|----------|-----|---------|-----------|
| **ATDD** | AT | `/bmad-tea-testarch-atdd` | Генерация падающих acceptance-тестов (TDD red phase) из acceptance criteria истории |
| **Test Automation** | TA | `/bmad-tea-testarch-automate` | Расширение покрытия: генерация E2E, API, компонентных, unit-тестов по приоритету |
| **Test Review** | RV | `/bmad-tea-testarch-test-review` | Аудит качества тестов: оценка 0-100 по детерминизму, изоляции, поддерживаемости, производительности |
| **NFR Assessment** | NR | `/bmad-tea-testarch-nfr` | Оценка нефункциональных требований: безопасность, производительность, надёжность, масштабируемость |
| **Traceability** | TR | `/bmad-tea-testarch-trace` | Матрица трассируемости: маппинг требований на тесты + решение по quality gate (PASS/FAIL) |

---

## Рекомендованная последовательность для проекта

После реализации фичи провайдеров через Quick Flow:

```
1. Test Design (TD)
   Вход: docs/* + project-context.md + код фичи
   Выход: тестовая стратегия (два документа)
        |
2. Test Framework (TF)
   Выход: scaffold Playwright/Cypress
        |
3. ATDD (AT)
   Вход: acceptance criteria фичи
   Выход: падающие тесты
        |
4. Test Automation (TA)
   Выход: расширенное покрытие
        |
5. Test Review (RV)
   Выход: оценка качества тестов
        |
6. NFR Assessment (NR)  [опционально]
   Выход: оценка безопасности и производительности контейнера
        |
7. Traceability (TR)  [опционально]
   Выход: матрица покрытия + решение по релизу
```

---

## Применимость TEA к текущему проекту

### Что можно протестировать

- **auth-gateway.py** — API-тесты аутентификации, проксирования, cookie-логики
- **Docker-контейнер** — NFR: время сборки, размер образа, потребление ресурсов, безопасность
- **entrypoint.sh** — интеграционные тесты запуска сервисов
- **Фича провайдеров** — E2E тесты переключения, API-тесты конфигурации

### Что не покрывается TEA

- Учебные материалы (course/) — это контент, не код
- Skills — это шаблоны Claude Code, не тестируемый код
- BMAD-конфигурация — валидируется через BMB, не TEA

---

## Teach Me Testing — для онбординга команды

Отдельная возможность, не привязанная к проекту. Полезна, если в команде есть люди без опыта тестирования. 7 самостоятельных сессий:

1. Quick Start — основы
2. Core Concepts — виды тестов, пирамида
3. Architecture and Patterns — паттерны тестирования
4. Test Design — проектирование тестов
5. ATDD and Automate — acceptance-driven подход
6. Quality and Trace — метрики и трассируемость
7. Advanced Patterns — продвинутые техники

Запуск: `/bmad-tea-teach-me-testing` в любой момент, независимо от остальных задач.
