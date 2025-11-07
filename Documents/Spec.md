# Linker

### Author  
**Oleksandr Starshynov**

### Current Doc Version  
**v1.0 (November 2025)**  

---

## 1. Introduction  

### Project name  
**Linker**  

### Brief description  
**Linker** — интеллектуальная система для обработки, векторизации и стандартизации текстовых фрагментов.  
Приложение помогает автоматизировать рутинную работу с большими объемами текстов, объединяя классический поиск по содержанию с поиском по смысловому сходству.  

### Project goal  
Создать инструмент, который освобождает человека от повторяющихся задач обработки текста,  
позволяя сосредоточиться на анализе, выявлении связей и формировании новых правил для обучения модели.  
Linker — это не просто экономия времени, а шаг к созданию «мыслящего партнёра» для исследователя.  

### Product history  
Проект начался как пример цифрового инструмента для изучения истории Нидерландов и города Алкмар. Была создана MVP версия, получившая признание у группы гидов. Это дало основание для развития проекта и расширения функционала. Первая коммерческая версия ориентирована на ручное редактирование и анализ тематических фрагментов.  
В будущих коммерческих релизах запланировано добавление автоматического определения тем, интерфейса визуальных связей и обучающего контура для модели.

---

## 2. Overall Architecture  

### Architectural approach  
**Modular Monolithic Architecture** — приложение с чётко выделенными слоями: UI, API, Model и Storage.  
Компоненты разделены логически, но работают в одном контексте, что обеспечивает простоту сопровождения и предсказуемость поведения.  

### Core principles  
- Разделение ответственности между слоями (UI / API / Model / Storage)  
- Использование двух типов хранилищ: реляционного (Supabase) и векторного (Qdrant)  
- Полная локальная работа модели (SentenceTransformer)  
- Минимальные зависимости, простой запуск в локальной среде  
- Возможность масштабирования и выделения сервисов в будущем  

### Components  

#### Frontend (TypeScript)
Интерфейс для создания, редактирования и анализа фрагментов.  

**Функциональность:**  
- Создание фрагментов и тем  
- Редактирование и группировка по `topic`  
- Визуализация иерархий через `id` / `parentId`  
- Поиск по тексту и смыслу  
- Работа напрямую с REST API  
- Локальное хранение JSON-фрагментов  

#### ⚙️ Backend (FastAPI + Python)
Серверная часть выполняет приём текстов, их разбиение, векторизацию и координацию взаимодействия между базами.  

**Pipeline:**  
1. Принимает текстовый фрагмент  
2. Делит на подфрагменты по смыслу  
3. Присваивает `id` и `parentId`  
4. Сохраняет оригинал в Supabase  
5. Генерирует эмбеддинги через SentenceTransformer  
6. Записывает эмбеддинги в Qdrant  
7. Возвращает результаты поиска клиенту  

**Основные эндпоинты:**  
- `POST /fragment` — добавить фрагмент  
- `GET /fragment/{id}` — получить фрагмент  
- `GET /search/text` — поиск по содержимому  
- `GET /search/vector` — поиск по смысловому сходству  

#### AI Model Layer
Используется локальная модель `SentenceTransformer (all-MiniLM-L6-v2)` с дообучением на собственных данных.  

**Особенности:**  
- Модель расположена в `models/custom-embeddings-v1`  
- Дообучение на триплетах *(Anchor / Positive / Negative)*  
- Тематическая специализация под область исследования  
- Совместимость всех эмбеддингов между сеансами  
- Возможность переобучения без нарушения согласованности данных  

#### Data Layer
- **Supabase (PostgreSQL)** — хранение исходных текстов, метаданных (`id`, `parentId`, `topic`, timestamps`)  
- **Qdrant (Vector DB)** — хранение векторных представлений и поиск ближайших соседей  

**Data Flow:**  
`Frontend → FastAPI → Supabase + Qdrant → Frontend`

---

## 3. Technology Stack  

- **Frontend:** TypeScript (без фреймворков), Live Server  
- **Backend:** Python + FastAPI  
- **Databases:** Supabase (PostgreSQL), Qdrant (Vector Search)  
- **AI:** SentenceTransformer (all-MiniLM-L6-v2), local fine-tuned model  
- **Deployment:** локальный запуск, планируется Docker  
- **Version control:** GitHub

Code Structure (Server)

| File / Folder                  | Purpose |
|-------------------------------|---------|
| `main.py`                     | Точка входа FastAPI, объявления маршрутов и запуск приложения |
| `config.py`                   | Конфигурация (ключи/URL Supabase и Qdrant, пути к моделям, параметры обучения) |
| `models.py`                   | Pydantic-схемы: `Fragment`, `SearchRequest`, `SearchResult`, служебные типы |
| `text_splitter.py`            | Разбиение входного текста на подфрагменты по смысловым границам |
| `process_fragment.py`         | Нормализация и подготовка фрагментов, присвоение `id`/`parentId` |
| `embeddings.py`               | Загрузка и использование модели SentenceTransformer, генерация эмбеддингов |
| `embeddings_generator.py`     | Батч-генерация эмбеддингов для уже сохранённых текстов |
| `qdrant_service.py`           | Клиент к Qdrant: создание коллекций, upsert, vector-search, фильтры |
| `supabase_client.py`          | Клиент к Supabase/PostgreSQL: insert/select/update фрагментов и метаданных |
| `train_embeddings.py`         | Обучение/дообучение модели на триплетах (Anchor/Positive/Negative) |
| `TrainingData/`               | Тренировочные наборы данных и вспомогательные словари |
| `models/custom-embeddings-v1/`| Локально сохранённая дообученная SentenceTransformer-модель |
| `training_logs.csv`           | Логи обучения и метрики (эпохи, loss, выборки) |
| `requirements.txt`            | Зависимости сервера (FastAPI, Uvicorn, qdrant-client, supabase-py, sentence-transformers и т.д.) |
| `README.md`                   | Инструкции по запуску бэкенда |

**Запуск backend:**
```
# из папки Linker_server
pip install -r requirements.txt
python -m uvicorn main:app --reload
```

---

## 4. Users and Roles  

Пока один тип пользователя — **исследователь текста**.  

**В будущем:**  
- **Admin** — управление моделями и базами  
- **Editor** — редактирование фрагментов  
- **Analyst** — работа с графом связей и статистикой  

---

## 5. Integration and Communication  

### Data exchange  
Коммуникация по REST API между фронтендом и FastAPI.  
Асинхронное взаимодействие между Supabase и Qdrant внутри бэкенда.  

### Versioning  
- API: `/api/v1`  
- Модель: `models/custom-embeddings-v1`  
- Коллекции Qdrant имеют версионные префиксы для безопасного обновления.  

---

## 6. Data and Storage  

### Data model  
Каждый фрагмент содержит:  
- `id` — уникальный идентификатор  
- `parentId` — ссылка на родительский фрагмент  
- `text` — содержимое  
- `topic` — тема (выбирается вручную)  
- `embedding` — векторное представление  
- `timestamp` — дата добавления  

### Storage  
- **Supabase:** текст и метаданные  
- **Qdrant:** эмбеддинги, индексы, семантический поиск  
- **Model folder:** локальные модели и их версии  

### Search modes  
- **Strict text match** — точное совпадение  
- **Like search** — частичное совпадение  
- **Vector search** — поиск по смысловому сходству  

---

## 7. Security and PII  

- Вся обработка выполняется локально  
- Данные не передаются во внешние API  
- Поддержка Supabase Auth (в планах)  
- Изоляция коллекций и пользовательских сессий  
- Возможность добавления шифрования AES-256-GCM  

---

## 8. Logging and Monitoring  

- Логи FastAPI (uvicorn)  
- Ошибки и время отклика — локальные файлы  
- Планируется интеграция Prometheus для latency/throughput  

---

## 9. DevOps and CI/CD  

- Репозиторий GitHub  
- **Frontend:** открыть `index.html` через Live Server  
- **Backend:** запустить `py -m uvicorn main:app --reload`  
- Планируется Docker Compose и GitHub Actions для автосборки  

---

## 10. Testing and Quality  

- **Unit tests:** pytest (API endpoints)  
- **Manual testing:** фронтенд и интеграция  
- **Future:** тесты для сравнения эмбеддингов и консистентности поиска  

---

## 11. Monitoring and Incident Response  

- Метрики: время отклика, количество сохранённых фрагментов, скорость поиска  
- Планируется система оповещений при сбоях модели или потере индекса  

---

## 12. Roadmap and Future Development  

- Docker-контейнеризация  
- Поддержка Supabase Auth  
- Автоматическое определение темы текста  
- Визуализация графа связей фрагментов  
- Персональные коллекции пользователей  
- Поддержка offline-first режима  
- Обучающий контур для модели (feedback loop)  
