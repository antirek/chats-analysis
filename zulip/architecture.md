# Анализ архитектуры Zulip

## Обзор проекта

Zulip - это система командного обмена сообщениями с уникальной организацией по топикам (topic-based threading), которая сочетает в себе лучшие черты email и чата. Проект построен на Django (Python) с использованием Tornado для real-time доставки сообщений.

## Ключевые понятия

### Основные сущности

**Realm (Организация)**
- Изолированная организация пользователей в Zulip
- Каждый realm имеет свой поддомен и настройки
- Пользователи принадлежат одному realm

**UserProfile (Профиль пользователя)**
- Представляет пользователя в системе
- Может быть обычным пользователем или ботом
- Содержит настройки уведомлений, предпочтения и т.д.

**Stream (Канал/Поток)**
- Публичный или приватный канал для групповых обсуждений
- Пользователи подписываются на streams
- Может быть публичным (доступен всем) или приватным (invite-only)

**Recipient (Получатель)**
- Универсальная модель для определения адресата сообщения
- Типы: PERSONAL (1:1 DM), STREAM (канал), DIRECT_MESSAGE_GROUP (групповой DM)
- Использует type_id для ссылки на конкретный объект (UserProfile, Stream, DirectMessageGroup)

**Message (Сообщение)**
- Основная модель сообщения
- Содержит: отправителя, получателя (Recipient), тему, контент (markdown), отрендеренный HTML
- Хранится в PostgreSQL

**UserMessage (Связь пользователя с сообщением)**
- Связывает пользователя с сообщением
- Хранит флаги: read, starred, mentioned, has_alert_word и т.д.
- Создается для каждого получателя сообщения
- Критична для производительности - позволяет быстро находить непрочитанные сообщения пользователя

**DirectMessageGroup (Групповой прямой диалог)**
- Группа пользователей для групповых DM
- Членство хранится через Subscription к Recipient типа DIRECT_MESSAGE_GROUP

**Subscription (Подписка)**
- Связывает пользователя с Recipient (Stream или DirectMessageGroup)
- Определяет, кто получает сообщения в канале или группе

**Event Queue (Очередь событий)**
- Система для real-time доставки событий клиентам
- Каждый клиент имеет свою очередь событий
- Управляется Tornado сервером

**Client (Клиент)**
- Тип клиента, отправившего сообщение (web, mobile, API и т.д.)

### Архитектурные компоненты

**Django Backend**
- Обрабатывает HTTP запросы
- Выполняет бизнес-логику
- Работает с базой данных PostgreSQL
- Отправляет события в Tornado через RabbitMQ

**Tornado Server**
- Асинхронный сервер для real-time доставки
- Управляет event queues для клиентов
- Получает события от Django через RabbitMQ
- Отправляет события клиентам через long-polling или WebSocket

**RabbitMQ**
- Очередь сообщений между Django и Tornado
- Обеспечивает надежную доставку событий
- Поддерживает шардинг по портам для масштабирования

**PostgreSQL**
- Основная база данных
- Хранит все данные: сообщения, пользователей, настройки и т.д.

## Общая архитектура системы

```mermaid
graph TB
    subgraph "Клиенты"
        Web[Web Browser]
        Mobile[Mobile App]
        API[API Client]
    end
    
    subgraph "Django Backend"
        Django[Django WSGI Server]
        Views[Views/API Endpoints]
        Actions[Actions Layer]
        Models[(Django ORM)]
    end
    
    subgraph "Tornado Server"
        Tornado[Tornado Async Server]
        EventQueue[Event Queue Manager]
        Handlers[Request Handlers]
    end
    
    subgraph "Очереди"
        RabbitMQ[RabbitMQ Message Queue]
    end
    
    subgraph "Хранилище"
        PostgreSQL[(PostgreSQL Database)]
        Cache[(Redis Cache)]
    end
    
    subgraph "Workers"
        EmailWorker[Email Worker]
        WebhookWorker[Webhook Worker]
        EmbedWorker[Embed Links Worker]
    end
    
    Web --> Django
    Mobile --> Django
    API --> Django
    
    Django --> Views
    Views --> Actions
    Actions --> Models
    Actions --> RabbitMQ
    Models --> PostgreSQL
    
    RabbitMQ --> Tornado
    Tornado --> EventQueue
    EventQueue --> Handlers
    Handlers --> Web
    Handlers --> Mobile
    
    RabbitMQ --> EmailWorker
    RabbitMQ --> WebhookWorker
    RabbitMQ --> EmbedWorker
    
    Actions --> Cache
    Models --> Cache
```

## Поток сообщения от отправителя к получателю

### Детальная схема прохождения сообщения

```mermaid
sequenceDiagram
    participant Sender as Отправитель<br/>(Клиент)
    participant Django as Django Backend
    participant DB as PostgreSQL
    participant RabbitMQ as RabbitMQ
    participant Tornado as Tornado Server
    participant Recipient as Получатель<br/>(Клиент)
    
    Note over Sender,Recipient: 1. Отправка сообщения
    
    Sender->>Django: POST /api/v1/messages<br/>{type, to, content, topic}
    Django->>Django: send_message_backend()
    Django->>Django: check_send_message()
    
    Note over Django: 2. Валидация и подготовка
    
    Django->>DB: Проверка прав доступа
    Django->>DB: Получение Stream/User данных
    Django->>Django: build_message_send_dict()
    Django->>Django: render_message_markdown()
    
    Note over Django,DB: 3. Сохранение сообщения
    
    Django->>DB: Создание Message объекта
    Django->>DB: Сохранение в zerver_message
    
    Note over Django,DB: 4. Создание UserMessage для получателей
    
    Django->>DB: Определение списка получателей
    Django->>DB: Создание UserMessage для каждого<br/>(флаги: unread, mentioned и т.д.)
    Django->>DB: bulk_insert_ums()
    
    Note over Django,RabbitMQ: 5. Отправка события в очередь
    
    Django->>Django: do_send_messages()
    Django->>Django: send_event_on_commit()
    Django->>RabbitMQ: Публикация события "message"<br/>{message_id, message_dict, users}
    
    Note over RabbitMQ,Tornado: 6. Доставка в Tornado
    
    RabbitMQ->>Tornado: Получение события из очереди
    Tornado->>Tornado: process_notification()
    Tornado->>Tornado: Распределение по event queues<br/>получателей
    
    Note over Tornado,Recipient: 7. Доставка получателю
    
    Tornado->>Recipient: Отправка события через<br/>long-polling/WebSocket
    Recipient->>Recipient: Отображение сообщения
```

### Схема обработки сообщения в Django

```mermaid
flowchart TD
    Start([HTTP POST /api/v1/messages]) --> Auth{Аутентификация}
    Auth -->|OK| Validate[Валидация параметров]
    Auth -->|Fail| Error1[Ошибка 401]
    
    Validate --> CheckAccess{Проверка доступа}
    CheckAccess -->|Stream| CheckStream[Проверка подписки на канал]
    CheckAccess -->|DM| CheckDM[Проверка прав на DM]
    CheckAccess -->|Fail| Error2[Ошибка доступа]
    
    CheckStream --> BuildMsg[build_message_send_dict]
    CheckDM --> BuildMsg
    
    BuildMsg --> Render[Рендеринг Markdown]
    Render --> CreateMsg[Создание Message объекта]
    CreateMsg --> SaveMsg[(Сохранение в БД)]
    
    SaveMsg --> GetRecipients[Определение получателей]
    GetRecipients -->|Stream| StreamSubs[Подписчики канала]
    GetRecipients -->|DM| DMUsers[Участники DM]
    
    StreamSubs --> FilterRecipients[Фильтрация: muted, inactive]
    DMUsers --> FilterRecipients
    
    FilterRecipients --> CreateUM[Создание UserMessage<br/>для каждого получателя]
    CreateUM --> SetFlags[Установка флагов:<br/>mentioned, has_alert_word]
    
    SetFlags --> SendEvent[send_event_on_commit]
    SendEvent --> QueueEvent[Публикация в RabbitMQ]
    
    QueueEvent --> QueueWorkers{Фоновые задачи}
    QueueWorkers -->|Links| EmbedQueue[Очередь embed_links]
    QueueWorkers -->|Webhooks| WebhookQueue[Очередь webhooks]
    QueueWorkers -->|Email| EmailQueue[Очередь email]
    
    QueueEvent --> Response[HTTP 200 + message_id]
    Response --> End([Конец])
```

### Схема доставки события получателю

```mermaid
flowchart LR
    subgraph "Django"
        Event[Создание события]
        Commit[Transaction commit]
        RabbitPub[Публикация в RabbitMQ]
    end
    
    subgraph "RabbitMQ"
        Queue[Очередь notify_tornado]
    end
    
    subgraph "Tornado"
        Receive[Получение события]
        Process[process_notification]
        FindQueues[Поиск event queues<br/>получателей]
        Filter[Фильтрация по narrow]
        Send[Отправка события]
    end
    
    subgraph "Клиенты"
        Client1[Клиент 1]
        Client2[Клиент 2]
        ClientN[Клиент N]
    end
    
    Event --> Commit
    Commit --> RabbitPub
    RabbitPub --> Queue
    Queue --> Receive
    Receive --> Process
    Process --> FindQueues
    FindQueues --> Filter
    Filter --> Send
    Send --> Client1
    Send --> Client2
    Send --> ClientN
```

## Структура данных

### Схема основных моделей

```mermaid
erDiagram
    Realm ||--o{ UserProfile : contains
    Realm ||--o{ Stream : contains
    Realm ||--o{ Message : contains
    
    UserProfile ||--o{ Message : sends
    UserProfile ||--o{ Subscription : has
    UserProfile ||--o{ UserMessage : receives
    
    Stream ||--o{ Subscription : has
    Stream ||--o{ Recipient : "type=STREAM"
    
    Recipient ||--o{ Message : receives
    Recipient ||--o{ Subscription : "for DM groups"
    
    Message ||--o{ UserMessage : "delivered to"
    Message }o--|| Recipient : sent_to
    Message }o--|| UserProfile : sender
    
    DirectMessageGroup ||--o{ Subscription : "members"
    DirectMessageGroup }o--|| Recipient : "type=DIRECT_MESSAGE_GROUP"
    
    Subscription }o--|| UserProfile : user
    Subscription }o--|| Recipient : recipient
    
    UserMessage }o--|| UserProfile : user
    UserMessage }o--|| Message : message
```

### Детальная структура Message и UserMessage

```mermaid
classDiagram
    class Message {
        +int id
        +UserProfile sender
        +Recipient recipient
        +Realm realm
        +str subject (topic)
        +str content (markdown)
        +str rendered_content (HTML)
        +DateTime date_sent
        +bool is_channel_message
        +bool has_attachment
        +bool has_image
        +bool has_link
    }
    
    class Recipient {
        +int id
        +int type_id
        +int type
        +PERSONAL = 1
        +STREAM = 2
        +DIRECT_MESSAGE_GROUP = 3
    }
    
    class UserMessage {
        +int id
        +UserProfile user_profile
        +Message message
        +BitField flags
        +read: bool
        +starred: bool
        +mentioned: bool
        +has_alert_word: bool
        +is_private: bool
    }
    
    class Stream {
        +int id
        +str name
        +Realm realm
        +bool invite_only
        +bool is_public
    }
    
    class DirectMessageGroup {
        +int id
        +str huddle_hash
        +Recipient recipient
        +int group_size
    }
    
    Message --> Recipient
    UserMessage --> Message
    UserMessage --> UserProfile
    Recipient --> Stream : "type=STREAM"
    Recipient --> DirectMessageGroup : "type=DIRECT_MESSAGE_GROUP"
```

## Компоненты системы

### Слои приложения

```mermaid
graph TB
    subgraph "Presentation Layer"
        API[API Endpoints<br/>zerver/views/]
        WebUI[Web UI<br/>templates/]
    end
    
    subgraph "Business Logic Layer"
        Actions[Actions<br/>zerver/actions/]
        Lib[Library Functions<br/>zerver/lib/]
    end
    
    subgraph "Data Access Layer"
        Models[Models<br/>zerver/models/]
        ORM[Django ORM]
    end
    
    subgraph "Real-time Layer"
        TornadoViews[Tornado Views<br/>zerver/tornado/]
        EventQueue[Event Queue<br/>zerver/tornado/event_queue.py]
    end
    
    subgraph "Background Workers"
        Workers[Worker Processes<br/>zerver/worker/]
    end
    
    API --> Actions
    Actions --> Lib
    Lib --> Models
    Models --> ORM
    
    Actions --> TornadoViews
    TornadoViews --> EventQueue
    
    Actions --> Workers
```

### Ключевые модули

**zerver/views/message_send.py**
- `send_message_backend()` - основной endpoint для отправки сообщений
- Обрабатывает HTTP запросы POST /api/v1/messages
- Валидирует параметры и вызывает actions

**zerver/actions/message_send.py**
- `check_send_message()` - основная функция отправки
- `build_message_send_dict()` - подготовка данных сообщения
- `do_send_messages()` - сохранение и доставка
- `create_user_messages()` - создание UserMessage записей
- `send_event_on_commit()` - отправка события в очередь

**zerver/tornado/event_queue.py**
- `EventQueue` - управление очередями событий
- `ClientDescriptor` - дескриптор клиентского соединения
- `process_notification()` - обработка уведомлений от Django

**zerver/tornado/django_api.py**
- `send_event_on_commit()` - отправка события после commit транзакции
- `send_event_rollback_unsafe()` - отправка события напрямую
- Коммуникация с Tornado через HTTP или напрямую

**zerver/models/messages.py**
- `Message` - модель сообщения
- `UserMessage` - связь пользователя с сообщением
- `AbstractMessage` - базовая модель

**zerver/models/recipients.py**
- `Recipient` - модель получателя
- `DirectMessageGroup` - модель группового DM

## Особенности реализации

### 1. Система доставки сообщений

**UserMessage для каждого получателя**
- Для каждого сообщения создается отдельная запись UserMessage для каждого получателя
- Это позволяет эффективно отслеживать статус прочтения для каждого пользователя
- Флаги хранятся в битовом поле для экономии места

**Event Queue система**
- Каждый клиент имеет свою очередь событий
- События фильтруются по narrow (фильтрам просмотра)
- Long-polling для получения событий в реальном времени

### 2. Обработка получателей

**Для Stream сообщений:**
1. Получение всех подписчиков канала (Subscription)
2. Фильтрация по правам доступа
3. Исключение muted пользователей
4. Учет настроек уведомлений

**Для Direct Messages:**
1. Определение участников DM (для групповых - через DirectMessageGroup)
2. Проверка прав доступа к пользователям
3. Создание UserMessage для каждого участника

### 3. Рендеринг и обработка контента

**Markdown рендеринг:**
- Контент хранится в markdown формате
- HTML версия генерируется при сохранении
- Поддержка упоминаний (@user, @group), ссылок, эмодзи

**Обработка упоминаний:**
- Парсинг упоминаний при рендеринге
- Установка флага `mentioned` в UserMessage
- Триггеры уведомлений для упомянутых пользователей

### 4. Фоновые задачи

**Embed Links Worker:**
- Обработка ссылок в сообщениях
- Получение Open Graph данных
- Генерация превью ссылок

**Email Worker:**
- Отправка email уведомлений
- Digest emails
- Missed message emails

**Webhook Worker:**
- Обработка outgoing webhooks
- Интеграции с внешними системами

## Производительность и масштабирование

### Индексы базы данных

**Message:**
- `realm_id, recipient_id, id` - для быстрого поиска по каналу
- `realm_id, sender_id, recipient_id` - для поиска сообщений пользователя
- `search_tsvector` - полнотекстовый поиск

**UserMessage:**
- Частичные индексы для разных флагов (starred, mentioned, unread)
- `user_profile, message` - уникальный индекс
- `user_profile, message` с условием на флаги

### Шардинг Tornado

- Поддержка нескольких процессов Tornado
- Распределение пользователей по портам
- RabbitMQ очереди для каждого порта

### Кэширование

- Redis для кэширования
- Кэш профилей пользователей
- Кэш настроек realm

## Безопасность

1. **Проверка доступа:**
   - Валидация прав на отправку в канал
   - Проверка подписки на канал
   - Проверка прав на отправку DM

2. **Фильтрация контента:**
   - Санитизация HTML при рендеринге
   - Защита от XSS
   - Валидация входных данных

3. **Изоляция данных:**
   - Realm-based изоляция
   - Проверка прав на доступ к сообщениям
   - Фильтрация по narrow для клиентов

## Заключение

Архитектура Zulip построена на разделении ответственности:
- **Django** обрабатывает бизнес-логику и работу с БД
- **Tornado** обеспечивает real-time доставку
- **RabbitMQ** связывает компоненты
- **PostgreSQL** хранит данные

Ключевая особенность - система UserMessage, которая позволяет эффективно отслеживать статус сообщений для каждого пользователя, и event queue система для real-time доставки.

