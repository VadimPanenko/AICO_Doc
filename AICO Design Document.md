# AICO Platform

## Единый Design Document

**Платформа AI-компаньона для образовательных курсов**

---

# 1. Executive Summary

## 1.1 Миссия проекта

Создать MVP платформы AI-компаньонов для образования, которая трансформирует учебный контент в персонализированные траектории развития навыков.

## 1.2 Три сценария использования

| Сценарий | Описание | Ключевые функции |
|----------|----------|------------------|
| **Lecture-Free** | AI-тьютор для самостоятельного изучения | Персонализированные лонгриды, граф знаний, диалог с компаньоном |
| **Кейс-тренажер** | Практико-ориентированное обучение | Симуляция бизнес-ситуаций, пошаговый диалог, оценка решений |
| **AI-помощник** | Универсальный ассистент | RAG по материалам курса, ответы на вопросы, рекомендации |

## 1.3 Параметры MVP

- **Масштаб:** до 2000 активных студентов
- **Срок разработки:** 11-12 недель
- **Команда:** 5 человек (1 Frontend, 1 Backend, 2 ML/AI, 1 Дизайнер)
- **Архитектура:** микросервисная (6 сервисов)
- **Инфраструктура:** cloud-agnostic (Docker + Kubernetes)

---

# 2. Архитектура системы

## 2.1 Принципы проектирования

### Почему микросервисы

- **Независимое масштабирование:** AI-сервисы требуют больше ресурсов при пиковых нагрузках
- **Изоляция отказов:** сбой в генерации контента не блокирует базовую работу платформы
- **Параллельная разработка:** команды работают независимо над разными сервисами

### Почему Redis Streams + Celery

- Отслеживание прогресса студентов требует audit trail всех действий
- Асинхронная генерация контента не должна блокировать UI
- ML-пайплайны (персонализация, оценка) работают в фоне

## 2.2 Высокоуровневая архитектура

Система состоит из 6 микросервисов, взаимодействующих через REST API и асинхронную очередь задач:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                    │
│         Web App (React)         Mobile (PWA)         Author Tool        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           API GATEWAY                                   │
│              (Kong / nginx — Routing, Rate Limiting, JWT)               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
       ┌────────────────────────────┼────────────────────────────┐
       ▼                            ▼                            ▼
 ┌──────────────┐        ┌──────────────────┐        ┌────────────────────┐
 │ AUTH SERVICE │        │   CORE SERVICE   │        │    AI SERVICE      │
 │ (Supertokens/│        │                  │        │                    │
 │  Keycloak)   │        │  • Курсы/Модули  │        │  • RAG Pipeline    │
 │              │        │  • Группы        │        │  • Graph Gen       │
 │  • JWT Auth  │        │  • Компетенции   │        │  • Personalization │
 │  • RBAC      │        │  • Доступы       │        │  • Companion Chat  │
 └──────────────┘        └──────────────────┘        │  • Assessment      │
                                                     └────────────────────┘
      ┌────────────────────────────┼────────────────────────────┐
      ▼                            ▼                            ▼
┌──────────────┐        ┌──────────────────┐        ┌────────────────────┐
│   CONTENT    │        │    ANALYTICS     │        │   NOTIFICATION     │
│   SERVICE    │        │     SERVICE      │        │     SERVICE        │
│              │        │                  │        │                    │
│  • Upload    │        │  • Статус-матрица│        │  • Email           │
│  • Parsing   │        │  • Прогресс      │        │  • Push (PWA)      │
│  • Chunking  │        │  • Портфолио     │        │  • Background Jobs │
│  • Storage   │        │  • Вопросы       │        │  (Celery)          │
└──────────────┘        └──────────────────┘        └────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
│  │ PostgreSQL │  │   Redis    │  │   Qdrant   │  │   Object Storage   │ │
│  │ (основные  │  │ (кэш,      │  │ (векторы,  │  │   (S3/MinIO)       │ │
│  │  данные)   │  │  очереди)  │  │ embeddings)│  │   файлы, медиа     │ │
│  └────────────┘  └────────────┘  └────────────┘  └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2.3 Детальное описание сервисов

### 2.3.1 Auth Service

**Назначение:** Аутентификация, авторизация и управление пользователями.

**Реализация:** Готовое решение (Supertokens или Keycloak) для ускорения разработки MVP.

**Функции:**
- Регистрация и аутентификация (email + password, OAuth)
- Выпуск и валидация JWT токенов (access + refresh)
- Ролевая модель RBAC: author, student, admin
- Хранение профилей пользователей

### 2.3.2 Core Service

**Назначение:** Основная бизнес-логика платформы — курсы, материалы, группы.

**Технологии:** FastAPI, PostgreSQL, SQLAlchemy

**Функции:**
- CRUD операции для модулей/курсов
- Управление компетенциями и образовательными результатами (Таксономия Блума)
- Группы студентов и ссылки доступа
- Граф знаний: Курс → Кластер → Концепт → Раздел
- Статусы публикации (draft → published → archived)

### 2.3.3 Content Service

**Назначение:** Загрузка, обработка и хранение учебных материалов.

**Технологии:** FastAPI, MinIO/S3, PyMuPDF, python-docx, Whisper (ASR)

**Функции:**
- Загрузка файлов: PDF, DOCX, TXT, видео, изображения
- Парсинг документов с сохранением структуры
- Транскрибация видео/аудио (Whisper ASR)
- Semantic Chunking для RAG (512-1024 токенов, overlap 20%)
- Embedding Generation и сохранение в Vector DB

### 2.3.4 AI Service

**Назначение:** Все AI/ML функции платформы в едином сервисе.

**Технологии:** FastAPI, LangChain/LangGraph, Qdrant, SSE streaming

**Подсистемы:**

1. **Knowledge Graph Generation** — извлечение концептов, построение связей, сопоставление с РПД
2. **RAG Pipeline** — hybrid search (vector + BM25), reranking, context assembly
3. **Personalization Engine** — адаптация контента по Колбу, интересам, сложности
4. **AI Companion** — диалоговый тьютор с контекстом страницы
5. **Assessment Engine** — генерация диагностик, оценка кейсов (LLM Judge)

### 2.3.5 Analytics Service

**Назначение:** Аналитика для авторов и статистика для студентов.

**Технологии:** FastAPI, PostgreSQL (read replicas), Redis (кэш агрегатов)

**Функции:**
- Статус-матрица: студенты × концепты с прогрессом
- Детальная информация по студенту (граф прохождения, результаты)
- Суммаризация вопросов к компаньону
- Портфолио студента (пройденные курсы, компетенции)

### 2.3.6 Notification Service

**Назначение:** Уведомления и фоновые задачи.

**Технологии:** FastAPI, Celery + Redis, SMTP

**Функции:**
- Email-уведомления (welcome, напоминания, результаты)
- Push-уведомления (PWA)
- Обработка фоновых событий из Redis Streams
- Очередь фоновых задач (генерация, экспорт)

---

# 3. AI/ML Pipeline

## 3.1 Data Ingestion Pipeline

Этап обработки материалов при создании курса автором:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Upload  │───▶│  Parse   │───▶│  Chunk   │───▶│  Embed   │───▶│  Vector  │
│  Files   │    │  (docx,  │    │  (smart  │    │  (text-  │    │    DB    │
│          │    │   pdf)   │    │  split)  │    │  embed)  │    │ (Qdrant) │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     │
     ▼
┌──────────┐
│  Whisper │  (для видео/аудио материалов)
│   ASR    │
└──────────┘
```

**Ключевые параметры:**
- Chunk size: 512-1024 токенов с overlap 20%
- Metadata: source_file, page, concept_id, section_id
- Hybrid search: vector + BM25 для лучшего recall

## 3.2 Knowledge Graph Generation

Алгоритм генерации графа (методика ВШЭ):

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Чанки     │───▶│Кластеризация│───▶│  Концепты   │
│  документов │    │  (HDBSCAN)  │    │  (LLM)      │
└─────────────┘    └─────────────┘    └─────────────┘
                                             │
                   ┌─────────────────────────┴─────────────────────────┐
                   ▼                                                   ▼
            ┌─────────────┐                                     ┌─────────────┐
            │Сопоставление│                                     │ Определение │
            │   с РПД     │                                     │   связей    │
            └─────────────┘                                     └─────────────┘
                   │                                                   │
                   └─────────────────────────┬─────────────────────────┘
                                             ▼
                                      ┌─────────────┐
                                      │  Маршрут    │
                                      │(топ. сорт.) │
                                      └─────────────┘
```

**Graph Schema (JSON):**

```json
{
  "concept_id": "UUID",
  "title": "Название темы",
  "prerequisites": ["list_of_concept_ids"],
  "content_chunks": ["chunk_id_1", "chunk_id_2"],
  "bloom_level": "Analysis",
  "diagnostics": ["test_id_1", "case_id_1"]
}
```

## 3.3 RAG Pipeline

Детальная схема обработки запроса:

```
User Query: "Что такое полиморфизм?"
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     1. QUERY ANALYSIS                           │
│  • Определение типа (факт / объяснение / связь)                 │
│  • Извлечение концептов: ["полиморфизм"]                        │
│  • Intent: объяснение + применение                              │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     2. HYBRID RETRIEVAL                         │
│  ┌───────────────────┐         ┌───────────────────┐            │
│  │   Vector Search   │         │   Graph Lookup    │            │
│  │   (Qdrant)        │         │   (PostgreSQL)    │            │
│  │   → Top-10 chunks │         │   → Prerequisites │            │
│  └───────────────────┘         └───────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     3. CONTEXT ASSEMBLY                         │
│  System: Ты AI-тьютор курса "ООП на Java"                       │
│  Student Profile: Converger, интересы: [игры, веб]              │
│  Relevant Content: [Chunk 1] [Chunk 2] [Chunk 3]                │
│  Knowledge Graph: Prerequisites ✓, Related topics               │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     4. LLM GENERATION                           │
│  Streaming Response (SSE) с обязательной ссылкой на источник    │
└─────────────────────────────────────────────────────────────────┘
```

## 3.4 Personalization Engine

**Параметры персонализации:**

| Параметр | Значения | Влияние на контент |
|----------|----------|-------------------|
| Стиль Колба | Diverger, Assimilator, Converger, Accommodator | Структура изложения, тип примеров |
| Интересы | Минимум 3 (до 500 знаков каждый) | Контекст примеров |
| Сложность | basic, intermediate, advanced | Глубина терминологии |
| Стиль компаньона | informal, academic, custom | Тон диалога |

**Адаптация контента по стилям Колба:**

| Стиль | Характеристики | Адаптация |
|-------|----------------|-----------|
| Diverger | Наблюдение, генерация идей | Примеры из жизни, визуализация |
| Assimilator | Логика, теория | Структурированная информация, схемы |
| Converger | Практика, решение задач | Алгоритмы, пошаговые инструкции |
| Accommodator | Эксперименты, действия | Интерактивные элементы, вопросы |

## 3.5 Assessment Engine

### Диагностики понимания (RAT-модель)

**Типы:** единичный выбор, множественный выбор, соотнесение, ввод значения

**Генерация:** LLM с self-reflection для проверки корректности дистракторов

### Кейсовые диагностики

Диалоговая симуляция с LLM Judge:

```
┌───────────────┐
│ Описание      │  Контекст бизнес-ситуации + Роль студента
│ ситуации      │
└───────┬───────┘
        ▼
┌───────────────┐                              
│   Вопрос N    │◀─────────────────────────────┐
└───────┬───────┘                              │
        ▼                                      │
┌───────────────┐      ┌───────────────┐       │
│ Ответ студента│─────▶│ LLM Judge     │       │
└───────────────┘      │ (score 0-1)   │       │
                       └───────┬───────┘       │
                               │               │
             ┌─────────────────┴─────────────┐ │
             ▼                               ▼ │
      score >= 0.5                  score < 0.5│
      ┌───────────┐                ┌───────────┤
      │ Следующий │                │ Подсказка │
      │  вопрос   │                │ (max 2)   │
      └───────────┘                └───────────┘
             │                                  
             ▼                                  
      ┌───────────────────────────────────────┐
      │         ФИНАЛЬНЫЙ FEEDBACK            │
      │  • Оценка 1-5                         │
      │  • Пояснения по каждому вопросу       │
      │  • Рекомендации по изучению           │
      └───────────────────────────────────────┘
```

---

# 4. Модель данных

## 4.1 PostgreSQL — основные сущности

Доменная модель платформы:

```
Course (Курс)
├── id, name, description, status (draft/published)
├── scenario_type: lecture_free | case_trainer | ai_assistant
├── author_id, created_at, updated_at
├── competencies: [Competency]
├── learning_outcomes: [LearningOutcome]
└── knowledge_graph: KnowledgeGraph

KnowledgeGraph (Граф знаний)
├── id, course_id, version
├── concepts: [Concept]
└── edges: [ConceptEdge]  // prerequisite, relates_to, next

Concept (Концепт/Тема)
├── id, graph_id, name, description, order
├── sections: [Section]
├── case_diagnostics: [CaseDiagnostic]
└── prerequisites: [Concept]

Section (Раздел/Подтема)
├── id, concept_id, name, order
├── content_essence (базовый контент для персонализации)
├── additional_content: [ContentBlock]  // видео, код, таблицы
├── diagnostics: [Diagnostic]
└── source_chunks: [SourceChunk]

StudentProfile
├── user_id, kolb_style, kolb_answers
├── interests: [str]  // минимум 3
├── complexity_level: basic | intermediate | advanced
└── companion_style, companion_custom_prompt

StudentProgress
├── user_id, course_id
├── concept_statuses: {concept_id: status}
├── diagnostic_results: [DiagnosticResult]
├── case_results: [CaseResult]
└── companion_interactions: [Interaction]
```

## 4.2 Qdrant — векторные коллекции

```python
# Коллекция чанков документов
course_chunks = {
    "vectors": { "size": 1024, "distance": "Cosine" },
    "payload_schema": {
        "module_id": "keyword",
        "concept_id": "keyword",
        "section_id": "keyword",
        "source_file": "text",
        "page": "integer",
        "text": "text"
    }
}
```

## 4.3 Redis — структуры данных

```
# Сессии пользователей
session:{session_id} → {user_id, role, expires_at}  TTL: 7 days

# Кэш персонализированного контента
personalized:{section_id}:{profile_hash} → {content_json}  TTL: 24h

# Семантический кэш LLM ответов
llm_cache:{query_hash}:{module_id} → {response_json}  TTL: 1h

# Rate limiting
rate_limit:{user_id}:ai_requests → counter  TTL: 1 min

# Прогресс генерации
generation:{module_id}:graph → {status, progress_percent}  TTL: 1h
```

---

# 5. User Flows

## 5.1 Author Flow: Создание курса

1. **Онбординг и авторизация** — вход, просмотр слайдов о платформе
2. **Создание модуля** — название, описание, УГСН, сценарий использования
3. **Компетенции и результаты** — выбор ПК/ОПК/УК из справочника, формулировка по Блуму
4. **Загрузка контента** — файлы (DOCX, PDF, TXT), ссылки на видео, РПД
5. **Генерация графа** — автоматическое извлечение концептов, фоновый процесс с прогресс-баром
6. **Работа с графом** — редактирование структуры, связей, визуальное представление
7. **Работа с содержанием** — просмотр контент-эссенции, добавление доп. контента
8. **Создание диагностик** — тесты (4 типа), кейсовые диагностики
9. **Публикация** — генерация ссылки, выдача доступов группам
10. **Мониторинг** — статус-матрица, аналитика по студентам, вопросы к преподавателю

## 5.2 Student Flow: Изучение курса

1. **Онбординг** — авторизация, слайды о компаньоне и персонализации
2. **Настройка компаньона** — тест Колба (20-30 вопросов), ввод интересов (мин. 3), выбор стиля
3. **Выбор сложности** — basic / intermediate / advanced
4. **Генерация персонализированной версии** — адаптация текстов и примеров
5. **Навигация по курсу** — граф концептов, рекомендованный маршрут
6. **Изучение контента** — Лонгрид (текст) или Лектор (слайды + голос)
7. **Дополнительные опции** — объяснить проще, привести пример, задать вопрос
8. **Диагностики понимания** — тесты после разделов
9. **Кейсовые диагностики** — диалоговая симуляция после концепта
10. **Портфолио** — результаты, достижения, компетенции

## 5.3 Структура лонгрида

1. Заголовок
2. Суммаризация (3-4 тезиса)
3. Ключевые понятия [Понятие — описание]
4. Основной текст (персонализация по Колбу и сложности)
5. Примеры (персонализация по интересам)
6. Выжимка содержания
7. Дополнительный контент автора (видео, код, таблицы)
8. Связанные концепты
9. Кнопка перехода к диагностике

---

# 6. Технологический стек

## 6.1 Backend

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| API Framework | FastAPI (Python) | Async, автодокументация, ML-экосистема |
| ORM | SQLAlchemy 2.0 + Alembic | Async поддержка, миграции |
| Task Queue | Celery + Redis | Фоновые задачи, retry logic |
| LLM Orchestration | LangChain + LangGraph | RAG, chains, stateful agents |
| Auth | Supertokens / Keycloak | OAuth2, RBAC, self-hosted |

## 6.2 Базы данных

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| Primary DB | PostgreSQL 15 + JSONB | ACID, гибкие схемы, масштабируемость |
| Vector DB | Qdrant (self-hosted) | ~2ms latency, metadata filtering, hybrid search |
| Cache / Queue | Redis | Сессии, кэш LLM, очереди задач |
| Object Storage | S3 / MinIO | Учебные материалы, медиа |

## 6.3 Frontend

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| Framework | React 18 + TypeScript | Экосистема, типизация |
| Build Tool | Vite | HMR, оптимизированные билды |
| State | TanStack Query + Zustand | Кэш API, UI-состояние |
| UI Kit | shadcn/ui + Tailwind | Accessibility, быстрая кастомизация |
| Graph Viz | React Flow | Drag-n-drop, customizable nodes |
| Rich Text | TipTap | Markdown, кастомные блоки |
| Real-time | SSE / WebSocket | Streaming LLM, чат |

## 6.4 AI/ML

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| LLM | Любой через LangChain | Абстракция от провайдера |
| Embeddings | text-embedding-3 / e5-large | 1024-мерные векторы |
| ASR | Whisper | Транскрибация видео |
| Semantic Cache | GPTCache + Redis | Снижение затрат на LLM |

## 6.5 Инфраструктура

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| Container | Docker | Изоляция, воспроизводимость |
| Orchestration | Kubernetes | Автомасштабирование, HA |
| API Gateway | Kong / nginx | Routing, rate limiting, JWT |
| CI/CD | GitHub Actions / GitLab CI | Автоматизация деплоя |
| Monitoring | Prometheus + Grafana | Метрики, алерты |

---

# 7. API Contracts

## 7.1 Core Service

```
# Модули
POST   /modules                     # Создание модуля
GET    /modules/{id}                # Получение модуля
PUT    /modules/{id}                # Редактирование
POST   /modules/{id}/publish        # Публикация
POST   /modules/{id}/access-link    # Генерация ссылки доступа

# Компетенции
POST   /modules/{id}/competencies   # Добавление компетенций
GET    /modules/{id}/competencies   # Список компетенций
POST   /modules/{id}/outcomes       # Образовательные результаты

# Группы
POST   /groups                      # Создание группы
GET    /modules/{id}/students       # Студенты модуля
POST   /groups/{id}/join            # Присоединение по ссылке
```

## 7.2 Content Service

```
POST   /modules/{id}/files              # Загрузка файлов
POST   /modules/{id}/files/url          # Загрузка по URL
GET    /modules/{id}/files              # Список файлов
POST   /modules/{id}/syllabus           # Загрузка РПД
GET    /sections/{id}/source            # Исходные чанки раздела
POST   /sections/{id}/additional        # Добавление доп. контента
```

## 7.3 AI Service

```
# Knowledge Graph
POST   /modules/{id}/graph/generate     # Запуск генерации
GET    /modules/{id}/graph/status       # Статус генерации
GET    /modules/{id}/graph              # Получение графа
PUT    /modules/{id}/graph/concepts     # Редактирование концептов
PUT    /modules/{id}/graph/relations    # Редактирование связей
POST   /modules/{id}/graph/confirm      # Подтверждение автором
GET    /modules/{id}/graph/route        # Рекомендованный маршрут

# AI Companion
POST   /chat/message                    # Отправка сообщения
WS     /chat/stream                     # WebSocket для streaming
POST   /chat/context                    # Установка контекста страницы
GET    /chat/history/{module_id}        # История диалогов
POST   /content/simplify                # "Объясни проще"
POST   /content/example                 # "Приведи пример"

# Personalization
GET    /kolb/questions                  # Вопросы теста Колба
POST   /kolb/submit                     # Отправка ответов
PUT    /profile/interests               # Обновление интересов
PUT    /profile/companion-style         # Выбор стиля компаньона
PUT    /profile/complexity              # Уровень сложности
POST   /modules/{id}/personalize        # Генерация персонализированного контента
GET    /sections/{id}/personalized      # Получение персонализированного раздела

# Assessment
POST   /sections/{id}/diagnostics       # Создание диагностики
POST   /diagnostics/generate            # Автогенерация вопросов
POST   /diagnostics/{id}/submit         # Отправка ответов
POST   /concepts/{id}/case              # Создание кейса
POST   /case/{id}/start                 # Начало прохождения
POST   /case/{id}/answer                # Ответ на вопрос кейса
GET    /case/{id}/result                # Результат кейса
```

## 7.4 Analytics Service

```
GET    /modules/{id}/status-matrix              # Статус-матрица
GET    /modules/{id}/students/{sid}/details     # Детали студента
GET    /modules/{id}/students/{sid}/questions   # Суммаризация вопросов
GET    /students/{id}/portfolio                 # Портфолио студента
GET    /modules/{id}/analytics                  # Агрегированные метрики
```

---

# 8. План разработки MVP

## 8.1 Фазы и сроки

| Фаза | Недели | Ключевые результаты |
|------|--------|---------------------|
| Phase 0: Foundation | 1–2 | Инфраструктура, CI/CD, API skeleton, Design system |
| Phase 1: Author Core | 3–5 | Auth, Course CRUD, File Upload, Graph Generation, Graph Editing |
| Phase 2: Student Flow (MVP) | 6–9 | Test Diagnostics, Case Diagnostics, Result Storage, Onboarding, Personalization, Navigation |
| Phase 3: Stabilization & Polish | 10–12 | Progress Tracking, Analytics Dashboard, Bug fixes, UX improvements, Basic Testing Coverage |

## 8.2 Распределение ролей

- **Frontend (1 чел)**  
  Архитектура, Auth flow, Course navigation, Graph viz, Chat interface, Content editors, Diagnostic UI

- **Backend (1 чел)**  
  Архитектура, API design, Core API, File upload, WebSocket, Analytics, Groups, Diagnostic results, Background jobs

- **ML/AI (2 чел)**  
  - **ML/AI 1:** RAG pipeline, Content personalization, Graph generation, Dialog context
  - **ML/AI 2:** Document parsing, Diagnostic/case generation, Answer evaluation

- **Дизайнер (1 чел)**  
  Design system, Author/Student flows, Avatar и эмоции, Responsive

> Продакт-роль распределена между командой: приоритизация задач, user stories и принятие решений — совместно на еженедельных sync-встречах.

---

# 9. Риски и митигация

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| Качество генерации графа | Высокая | Высокое | Итеративное улучшение промптов, ручная корректировка |
| Стоимость LLM API | Средняя | Среднее | Кэширование, локальные модели для embedding |
| Сложность RAG | Средняя | Высокое | Начать с простого, итерировать |
| Персонализация не работает | Средняя | Среднее | A/B тесты, сбор обратной связи |
| Перегруз команды | Средняя | Высокое | Четкий scope MVP, отсечение nice-to-have |

---

# 10. Метрики успеха MVP

## 10.1 Технические метрики

- Time to generate graph < 3 min
- RAG response latency < 3 sec
- Uptime > 99%

## 10.2 Продуктовые метрики

- Author: создание курса < 2 часов
- Student: completion rate > 60%
- NPS > 30

---

# 11. Что НЕ входит в MVP

- ❌ Интеграция с LMS (Moodle)
- ❌ Mobile native apps
- ❌ Аудиолектор (TTS)

---
