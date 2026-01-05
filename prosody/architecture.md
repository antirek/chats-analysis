# Архитектура Prosody IM Server

## Обзор

Prosody - это XMPP (Jabber) сервер, написанный на Lua. Он обеспечивает обмен сообщениями в реальном времени между пользователями через протокол XMPP.

## Ключевые понятия

### JID (Jabber ID)
JID - это уникальный идентификатор пользователя в XMPP сети. Формат: `node@domain/resource`
- **node** (username) - имя пользователя
- **domain** (host) - домен сервера
- **resource** - идентификатор клиента/устройства пользователя

Примеры:
- `user@example.com` - bare JID (без resource)
- `user@example.com/phone` - full JID (с resource)

### Stanza (Станза)
Stanza - это базовый XML элемент в XMPP протоколе. Существует три типа:
- **message** - сообщения между пользователями
- **presence** - информация о статусе пользователя (онлайн/офлайн)
- **iq** (Info/Query) - запросы и ответы для получения информации

### Session (Сессия)
Сессия представляет активное соединение клиента с сервером. Может быть:
- **c2s** (client-to-server) - соединение клиента с сервером
- **s2s** (server-to-server) - соединение между серверами
- **s2sin** - входящее s2s соединение
- **s2sout** - исходящее s2s соединение

### Host (Хост)
Host - это виртуальный домен, который обслуживает Prosody. Может быть:
- **VirtualHost** - домен для пользователей
- **Component** - специальный сервис (например, MUC для групповых чатов)

### Resource (Ресурс)
Ресурс идентифицирует конкретное устройство/клиент пользователя. Один пользователь может иметь несколько ресурсов одновременно (например, телефон и компьютер).

### Roster (Ростер)
Ростер - это список контактов пользователя, аналог списка друзей в мессенджерах.

## Архитектура системы

### Основные компоненты

```mermaid
graph TB
    subgraph "Prosody Core"
        HR[Host Manager]
        SM[Session Manager]
        SR[Stanza Router]
        MM[Module Manager]
        RM[Roster Manager]
        S2SM[S2S Manager]
    end
    
    subgraph "Network Layer"
        C2S[C2S Connections]
        S2S[S2S Connections]
        HTTP[HTTP/BOSH/WebSocket]
    end
    
    subgraph "Storage"
        FS[File Storage]
        SQL[SQL Storage]
        MEM[Memory Cache]
    end
    
    subgraph "Modules"
        MSG[mod_message]
        PRES[mod_presence]
        IQ[mod_iq]
        MUC[mod_muc]
        OTHER[Other Modules...]
    end
    
    C2S --> SM
    S2S --> S2SM
    HTTP --> SM
    
    SM --> SR
    S2SM --> SR
    SR --> HR
    
    HR --> MM
    MM --> MSG
    MM --> PRES
    MM --> IQ
    MM --> MUC
    MM --> OTHER
    
    SR --> RM
    RM --> FS
    RM --> SQL
    RM --> MEM
    
    MSG --> SR
    PRES --> SR
    IQ --> SR
```

## Поток обработки сообщения

### Сценарий 1: Сообщение между пользователями одного сервера

```mermaid
sequenceDiagram
    participant C1 as Client 1<br/>(user1@example.com/phone)
    participant SR as Stanza Router
    participant SM as Session Manager
    participant MSG as mod_message
    participant C2 as Client 2<br/>(user2@example.com/laptop)
    
    C1->>SR: message stanza<br/>(to: user2@example.com)
    SR->>SR: core_process_stanza()
    Note over SR: Добавление 'from' атрибута<br/>Валидация JID
    SR->>SR: core_post_stanza()
    Note over SR: Определение типа адресата<br/>(bare/full/host)
    SR->>MSG: fire_event("message/bare")
    MSG->>SM: Проверка online ресурсов
    SM->>SM: Поиск top_resources
    SM->>C2: Отправка сообщения
    C2->>C1: Доставка подтверждена
```

### Сценарий 2: Сообщение между пользователями разных серверов

```mermaid
sequenceDiagram
    participant C1 as Client 1<br/>(user1@server1.com)
    participant S1 as Server 1<br/>(Prosody)
    participant S2 as Server 2<br/>(Remote XMPP)
    participant C2 as Client 2<br/>(user2@server2.com)
    
    C1->>S1: message stanza<br/>(to: user2@server2.com)
    S1->>S1: core_process_stanza()
    S1->>S1: core_route_stanza()
    Note over S1: Определение удаленного хоста<br/>Проверка существующего s2s соединения
    alt Соединение существует
        S1->>S2: Отправка через s2s
    else Соединение отсутствует
        S1->>S1: Создание нового s2sout соединения
        S1->>S2: Установка соединения<br/>(TLS + Dialback)
        S1->>S2: Отправка сообщения
    end
    S2->>C2: Доставка сообщения
    C2->>S2: Подтверждение
    S2->>S1: Подтверждение (опционально)
```

### Сценарий 3: Сообщение на full JID

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant SR as Stanza Router
    participant MSG as mod_message
    participant SM as Session Manager
    participant C2 as Client 2<br/>(specific resource)
    
    C1->>SR: message stanza<br/>(to: user2@example.com/laptop)
    SR->>SR: core_process_stanza()
    SR->>SR: core_post_stanza()
    Note over SR: Определение типа: /full
    SR->>MSG: fire_event("message/full")
    MSG->>SM: Проверка существования ресурса
    alt Ресурс онлайн
        SM->>C2: Прямая доставка
    else Ресурс офлайн
        MSG->>MSG: process_to_bare()
        Note over MSG: Обработка как bare JID<br/>(доставка на другие ресурсы<br/>или сохранение офлайн)
    end
```

## Детальная архитектура маршрутизации

### Функции маршрутизации

```mermaid
graph TD
    START[Получение stanza] --> PROCESS[core_process_stanza]
    
    PROCESS --> VALIDATE{Валидация<br/>и нормализация JID}
    VALIDATE -->|Ошибка| ERROR[Отправка error reply]
    VALIDATE -->|OK| CHECK_TYPE{Тип origin}
    
    CHECK_TYPE -->|c2s| ADD_FROM[Добавление 'from' атрибута]
    CHECK_TYPE -->|s2s| CHECK_AUTH{Проверка<br/>авторизации}
    
    ADD_FROM --> POST[core_post_stanza]
    CHECK_AUTH -->|OK| POST
    CHECK_AUTH -->|FAIL| CLOSE[Закрытие соединения]
    
    POST --> DETECT_TYPE{Определение<br/>типа адресата}
    
    DETECT_TYPE -->|/full| FULL[Обработка full JID]
    DETECT_TYPE -->|/bare| BARE[Обработка bare JID]
    DETECT_TYPE -->|/host| HOST[Обработка host JID]
    DETECT_TYPE -->|/self| SELF[Обработка self]
    
    FULL --> EVENT_FULL[fire_event: message/full]
    BARE --> EVENT_BARE[fire_event: message/bare]
    HOST --> EVENT_HOST[fire_event: message/host]
    SELF --> EVENT_SELF[fire_event: message/self]
    
    EVENT_FULL --> HANDLER[Обработчики модулей]
    EVENT_BARE --> HANDLER
    EVENT_HOST --> HANDLER
    EVENT_SELF --> HANDLER
    
    HANDLER -->|Обработано| DONE[Готово]
    HANDLER -->|Не обработано| ROUTE[core_route_stanza]
    
    ROUTE --> CHECK_LOCAL{Локальный<br/>или удаленный?}
    
    CHECK_LOCAL -->|Локальный| POST
    CHECK_LOCAL -->|Удаленный| S2S_ROUTE[Маршрутизация через s2s]
    
    S2S_ROUTE --> CHECK_CONN{Существует<br/>s2s соединение?}
    
    CHECK_CONN -->|Да| SEND[Отправка через s2s]
    CHECK_CONN -->|Нет| CREATE[Создание s2sout]
    CREATE --> CONNECT[Установка соединения]
    CONNECT --> SEND
    
    SEND --> DONE
    ERROR --> DONE
    CLOSE --> DONE
```

## Управление сессиями

### Жизненный цикл c2s сессии

```mermaid
stateDiagram-v2
    [*] --> Unauthenticated: TCP соединение
    Unauthenticated --> Authenticated: SASL аутентификация
    Authenticated --> Unbound: Успешная аутентификация
    Unbound --> Bound: Привязка ресурса
    Bound --> Active: Готов к работе
    
    Active --> Bound: Потеря соединения
    Bound --> [*]: Закрытие соединения
    
    Active --> Hibernating: Неактивность<br/>(SMACKS)
    Hibernating --> Active: Возобновление
    
    note right of Unauthenticated
        c2s_unauthed
        Только SASL
    end note
    
    note right of Authenticated
        c2s_unbound
        Аутентифицирован,
        но без ресурса
    end note
    
    note right of Bound
        c2s
        Полностью активная сессия
    end note
```

### Структура данных сессий

```mermaid
classDiagram
    class Session {
        +string type
        +string id
        +string host
        +string username
        +string resource
        +string full_jid
        +connection conn
        +function send()
        +function close()
    }
    
    class C2SSession {
        +presence presence
        +number priority
        +roster roster
        +boolean interested
    }
    
    class S2SSession {
        +string from_host
        +string to_host
        +string direction
        +queue sendq
    }
    
    class HostSession {
        +string host
        +string type
        +table sessions
        +table s2sout
        +events events
        +modules modules
    }
    
    Session <|-- C2SSession
    Session <|-- S2SSession
    HostSession "1" *-- "*" C2SSession : содержит
    HostSession "1" *-- "*" S2SSession : содержит
```

## Модульная система

### Загрузка модулей

```mermaid
graph LR
    START[Старт сервера] --> LOAD_CONFIG[Загрузка конфигурации]
    LOAD_CONFIG --> ACTIVATE_HOST[Активация хостов]
    ACTIVATE_HOST --> LOAD_MODULES[Загрузка модулей]
    
    LOAD_MODULES --> CHECK_GLOBAL{Глобальный<br/>модуль?}
    CHECK_GLOBAL -->|Да| GLOBAL_ENV[Создание глобального окружения]
    CHECK_GLOBAL -->|Нет| HOST_ENV[Создание host окружения]
    
    GLOBAL_ENV --> LOAD_CODE[Загрузка кода модуля]
    HOST_ENV --> LOAD_CODE
    
    LOAD_CODE --> CALL_LOAD[Вызов module.load]
    CALL_LOAD --> HOOK_EVENTS[Регистрация обработчиков событий]
    HOOK_EVENTS --> READY[Модуль готов]
    
    READY --> CALL_READY{Есть<br/>module.ready?}
    CALL_READY -->|Да| READY_HOOK[Вызов module.ready]
    CALL_READY -->|Нет| DONE[Готово]
    READY_HOOK --> DONE
```

### Обработка событий

```mermaid
graph TD
    EVENT[Событие срабатывает] --> FIRE[fire_event]
    FIRE --> HANDLERS[Получение списка обработчиков]
    HANDLERS --> SORT[Сортировка по приоритету]
    SORT --> ITERATE[Итерация по обработчикам]
    
    ITERATE --> CALL[Вызов обработчика]
    CALL --> CHECK_RETURN{Возвращаемое<br/>значение}
    
    CHECK_RETURN -->|true| STOP[Остановка обработки]
    CHECK_RETURN -->|false/nil| NEXT[Следующий обработчик]
    CHECK_RETURN -->|другое| CONTINUE[Продолжение]
    
    NEXT --> ITERATE
    CONTINUE --> ITERATE
    
    STOP --> DONE[Событие обработано]
    ITERATE -->|Все обработаны| DONE
```

## Хранение данных

### Структура хранилища

```mermaid
graph TB
    subgraph "Storage Manager"
        SM[Storage Manager]
    end
    
    subgraph "Backends"
        FS[File Storage<br/>internal]
        SQL[SQL Storage<br/>PostgreSQL/MySQL/SQLite]
        MEM[Memory Cache]
    end
    
    subgraph "Data Types"
        ROSTER[Roster<br/>keyval/map]
        PRIVATE[Private Data<br/>keyval]
        VCARD[vCard<br/>keyval]
        MAM[Message Archive<br/>keyval]
    end
    
    SM --> FS
    SM --> SQL
    SM --> MEM
    
    FS --> ROSTER
    FS --> PRIVATE
    FS --> VCARD
    FS --> MAM
    
    SQL --> ROSTER
    SQL --> PRIVATE
    SQL --> VCARD
    SQL --> MAM
```

## Безопасность

### Аутентификация и авторизация

```mermaid
sequenceDiagram
    participant C as Client
    participant C2S as C2S Module
    participant SASL as SASL Auth
    participant UM as User Manager
    participant SM as Session Manager
    
    C->>C2S: Открытие XMPP потока
    C2S->>C: Отправка механизмов SASL
    C->>C2S: Выбор механизма (PLAIN/SCRAM)
    C2S->>SASL: SASL запрос
    SASL->>UM: Проверка учетных данных
    UM->>UM: Валидация пароля
    UM-->>SASL: Результат аутентификации
    SASL-->>C2S: Успех/Неудача
    C2S-->>C: Результат SASL
    
    alt Успешная аутентификация
        C2S->>SM: make_authenticated()
        SM->>SM: Создание сессии c2s_unbound
        C->>C2S: Привязка ресурса
        C2S->>SM: bind_resource()
        SM->>SM: Создание full JID
        SM-->>C: Подтверждение привязки
    end
```

## Производительность и оптимизация

### Кэширование

```mermaid
graph LR
    REQ[Запрос данных] --> CHECK_CACHE{В кэше?}
    CHECK_CACHE -->|Да| RETURN[Возврат из кэша]
    CHECK_CACHE -->|Нет| LOAD[Загрузка из хранилища]
    LOAD --> STORE[Сохранение в кэш]
    STORE --> RETURN
```

### Приоритет ресурсов

При доставке сообщения на bare JID, Prosody выбирает "лучший" ресурс на основе:
1. Приоритета (priority) - выше = лучше
2. Наличия presence - только онлайн ресурсы
3. Порядка подключения - при равном приоритете

## Расширяемость

### Создание модуля

Модули могут:
- Перехватывать события (hook)
- Обрабатывать stanza определенных типов
- Добавлять новые функции в API
- Расширять протокол XMPP

### Типы событий

- `stanza/message/bare` - сообщение на bare JID
- `stanza/message/full` - сообщение на full JID
- `stanza/presence/bare` - presence на bare JID
- `stanza/iq/...` - IQ запросы
- `pre-stanza` - до обработки stanza
- `route/remote` - маршрутизация на удаленный сервер

## Заключение

Prosody использует модульную архитектуру с четким разделением ответственности:
- **Core** - базовые функции (маршрутизация, сессии, хосты)
- **Modules** - расширяемая функциональность
- **Storage** - абстракция над хранилищами данных
- **Network** - обработка сетевых соединений

Такая архитектура обеспечивает гибкость, расширяемость и простоту поддержки.
