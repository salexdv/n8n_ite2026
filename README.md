# Магия n8n: от простого к сложному

Материалы доклада для ITE 2026 — практические кейсы связки **1С ↔ n8n ↔ AI**: от простого прокси для Telegram до генерации карточек товаров нейросетью.

## Состав репозитория

```
repo/
├── Workflows/                                  # n8n-workflow'ы доклада (JSON, для импорта)
│   ├── ITE2026_TelegramProxy.json
│   ├── ITE2026_Notification.json
│   ├── ITE2026_MockSimple.json
│   ├── ITE2026_MockAdvanced.json
│   ├── ITE2026_Feedback.json
│   ├── ITE2026_WebsocketSync.json
│   ├── ITE2026_MoneySync.json
│   ├── ITE2026_WebsocketChat.json
│   ├── ITE2026_KaitenBitrix24.json
│   └── ITE2026_AIGeneration.json
├── 1CBase/
│   └── 1Cv8.dt                                 # демо-база 1С для кейса синхронизации через Websocket
├── Чат.epf                                     # обработка 1С для чата через Websocket
└── ГенерацияКартинокИОписанияТовара.epf        # обработка 1С для AI-генерации карточек товаров
```

## Требования

- **n8n** (self-hosted) — для websocket-кейсов n8n должен быть доступен с машин с 1С по сети
- **MongoDB** — для workflow `Feedback`, `MoneySync`, `WebsocketChat`, `KaitenBitrix24`
- **Community-ноды** — см. таблицу ниже
- **API-ключи / credentials** — Telegram Bot, Max, Gotify, MongoDB, Kaiten и Битрикс24 (webhooks/API), OpenRouter (для AI-кейса)

## Быстрый старт

### Шаг 1. Установите community-ноды (до импорта workflow!)

В n8n откройте **Settings → Community nodes → Install** и установите нужные пакеты:

| Community node | Нужна для workflow |
|---|---|
| `n8n-nodes-globals` | `ITE2026_TelegramProxy` |
| `n8n-nodes-max` | `ITE2026_Notification` |
| `n8n-nodes-websocket-standalone` | `ITE2026_WebsocketSync`, `ITE2026_WebsocketChat` |

Если установить ноды после импорта, соответствующие узлы не распознаются и workflow придётся перезагружать.

### Шаг 2. Импортируйте workflow

**Workflows → Import from File** — выберите JSON-файл из каталога `Workflows/`.

### Шаг 3. Настройте credentials

Для каждого workflow создайте и подставьте свои credentials (Telegram, MongoDB, Max, Gotify, Kaiten, Битрикс24).

Для `ITE2026_AIGeneration`:

1. В узле **OpenRouter Chat Model** выберите/создайте credential `OpenRouter API`.
2. В узле **OpenRouter — Generate Image** создайте credential `Authentication` → `Generic Credential Type`, `Generic Auth Type` → `Header Auth`. В самом credential: `Name: Authorization`, `Value: Bearer ВАШ_КЛЮЧ`.
3. При необходимости смените `agent_model` и `image_model` в узле **Prompt Router**.
4. Категории и отдельные промты также настраиваются в **Prompt Router**.

## Кейсы доклада

| # | Кейс | Workflow | Обработка 1С |
|---|---|---|---|
| 1 | Прячем Telegram | `ITE2026_TelegramProxy` | — |
| 2 | Одно событие – 3 мессенджера | `ITE2026_Notification` | — |
| 3 | Mock REST API | `ITE2026_MockSimple`, `ITE2026_MockAdvanced` | — |
| 4 | Обратная связь по задачам (Google/Yandex Forms) | `ITE2026_Feedback` | — |
| 5 | Синхронизация баз в реальном времени через Websocket | `ITE2026_WebsocketSync` | `1CBase/1Cv8.dt` |
| 6 | 1С без границ: Desktop 1C ↔ Android 1C | `ITE2026_MoneySync` | — |
| 7 | «Система взаимодействия»: чат для сотрудников | `ITE2026_WebsocketChat` | `Чат.epf` |
| 8 | Дружим таск-трекеры: Kaiten ↔ Битрикс24 | `ITE2026_KaitenBitrix24` | — |
| 9 | 1С + n8n + AI: карточки товаров | `ITE2026_AIGeneration` | `ГенерацияКартинокИОписанияТовара.epf` |

### 1. Прячем Telegram

Reverse proxy: весь трафик к Telegram идёт через ваш домен, а n8n проксирует запросы к `api.telegram.org`. Простой вариант — чистый прокси, усложнённый — с хранением токенов в глобальных константах (`n8n-nodes-globals`).

### 2. Одно событие – 3 мессенджера

Webhook получает одно событие, а оповещение уходит сразу в три мессенджера: Telegram, Max и Gotify. Нода `n8n-nodes-max` добавляет поддержку Max.

### 3. Mock REST API

Заглушка REST API для разработки и тестирования интеграций без реального бэкенда. Простой вариант — один webhook с готовым ответом; усложнённый — набор endpoint'ов (`/orders`, `/order`, `/order-status`, `/pay`) с сохранением состояния в Workflow Static Data.

### 4. Обратная связь по задачам (Google/Yandex Forms)

Google/Yandex форма через Apps Script отправляет ответы в n8n. Реализован паттерн **set / get / ack**: ответы сохраняются в MongoDB, 1С забирает их по своим задачам и подтверждает получение.

### 5. Синхронизация баз в реальном времени через Websocket

Две базы 1С обмениваются изменениями напрямую, в реальном времени, через websocket-сервер n8n. Для демонстрации используйте демо-базу из `1CBase/` (см. ниже). Требуется `n8n-nodes-websocket-standalone`.

### 6. 1С без границ: Desktop 1C ↔ Android 1C

Синхронизация данных между десктопной и мобильной базами 1С: устройство получает новые документы для своего id и подтверждает загрузку (ack). Очередь документов и идентификаторы устройств хранятся в MongoDB.

### 7. «Система взаимодействия»: чат для сотрудников

Собственный аналог «Системы взаимодействия» — чат сотрудников прямо в 1С через Websocket. Поддерживаются сообщения, файлы, история и список онлайн-клиентов (ping/broadcast). Клиентская часть — обработка `Чат.epf`, хранение — MongoDB.

### 8. Дружим таск-трекеры: Kaiten ↔ Битрикс24

Двусторонняя синхронизация задач между Kaiten и Битрикс24: webhooks обеих систем, согласование сущностей и статусов, связки задач хранятся в MongoDB.

### 9. 1С + n8n + AI: карточки товаров

1С отправляет фото и данные номенклатуры; AI Agent (OpenRouter) анализирует исходное фото, пишет продающее описание и готовит ровно N промтов, а генерация изображений остаётся детерминированной частью workflow. Результат — готовая карточка товара. Клиентская часть — обработка `ГенерацияКартинокИОписанияТовара.epf`.

## Демо-база 1С

`1CBase/1Cv8.dt` — простая демо-база 1С:Предприятие 8 для кейса **«Синхронизация баз в реальном времени через Websocket»**.

1. В Конфигураторе: **Администрирование → Загрузить информационную базу…** и выберите `1Cv8.dt`.
2. Для демонстрации обмена загрузите базу **дважды** (в два разных каталога) и запустите две копии — изменения в одной базе будут появляться в другой через workflow `ITE2026_WebsocketSync`.

## Лицензия

Материалы распространяются по лицензии [MIT](LICENSE).
