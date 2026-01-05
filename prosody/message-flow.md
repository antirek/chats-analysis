# Детальный поток обработки сообщения

## Обзор

Этот документ описывает детальный путь прохождения сообщения от одного участника к другому в Prosody.

## Сценарий 1: Локальная доставка (один сервер)

### Исходные данные
- Отправитель: `alice@example.com/phone`
- Получатель: `bob@example.com/laptop`
- Оба пользователя на одном сервере `example.com`

### Последовательность обработки

```mermaid
sequenceDiagram
    participant A as Alice Client<br/>(phone)
    participant C2S as C2S Module
    participant SR as Stanza Router
    participant MSG as mod_message
    participant SM as Session Manager
    participant B as Bob Client<br/>(laptop)
    
    Note over A: Пользователь отправляет<br/>сообщение
    
    A->>C2S: TCP/XMPP: message stanza<br/>&lt;message to="bob@example.com"&gt;
    
    Note over C2S: Получение stanza<br/>из XMPP потока
    
    C2S->>SR: core_process_stanza(origin, stanza)
    
    Note over SR: 1. Валидация stanza<br/>2. Добавление 'from' атрибута<br/>3. Нормализация JID
    
    SR->>SR: stanza.attr.from = "alice@example.com/phone"
    SR->>SR: jid_prepped_split("bob@example.com")
    
    SR->>SR: core_post_stanza(origin, stanza, true)
    
    Note over SR: Определение типа адресата:<br/>node="bob", host="example.com"<br/>resource отсутствует → /bare
    
    SR->>MSG: fire_event("message/bare", {<br/>  origin=origin,<br/>  stanza=stanza,<br/>  to_self=false<br/>})
    
    MSG->>MSG: process_to_bare("bob@example.com", origin, stanza)
    
    MSG->>SM: Проверка: bare_sessions["bob@example.com"]
    
    alt Ресурсы онлайн
        SM->>SM: Получение top_resources
        Note over SM: Выбор ресурсов с<br/>наивысшим приоритетом
        
        SM->>B: session.send(stanza)
        B->>A: Подтверждение доставки
    else Ресурсы офлайн
        MSG->>MSG: fire_event("message/offline/handle")
        Note over MSG: Сохранение в<br/>offline storage
        MSG->>A: Подтверждение сохранения
    end
```

### Детальный код обработки

#### 1. Получение stanza (core/stanza_router.lua)

```lua
function core_process_stanza(origin, stanza)
    -- Для c2s соединений добавляем 'from' атрибут
    if origin.type == "c2s" and not stanza.attr.xmlns then
        stanza.attr.from = origin.full_jid;
    end
    
    -- Нормализация JID получателя
    local to = stanza.attr.to;
    if to then
        local node, host, resource = jid_prepped_split(to);
        -- ...
        stanza.attr.to = to;
    end
    
    -- Маршрутизация
    core_post_stanza(origin, stanza, origin.full_jid);
end
```

#### 2. Определение типа адресата (core/stanza_router.lua)

```lua
function core_post_stanza(origin, stanza, preevents)
    local to = stanza.attr.to;
    local node, host, resource = jid_split(to);
    local to_bare = node and (node.."@"..host) or host;
    
    local to_type;
    if node then
        if resource then
            to_type = '/full';  -- user@host/resource
        else
            to_type = '/bare';  -- user@host
        end
    else
        if host then
            to_type = '/host';  -- host
        else
            to_type = '/bare';  -- self
        end
    end
    
    -- Генерация события
    local event = stanza.name .. to_type;  -- "message/bare"
    hosts[host].events.fire_event(event, {
        origin = origin,
        stanza = stanza
    });
end
```

#### 3. Обработка сообщения (plugins/mod_message.lua)

```lua
module:hook("message/bare", function(data)
    local origin, stanza = data.origin, data.stanza;
    local bare = stanza.attr.to;
    
    return process_to_bare(bare, origin, stanza);
end, -1);

function process_to_bare(bare, origin, stanza)
    local user = bare_sessions[bare];
    
    if user then
        -- Пользователь онлайн
        local recipients = user.top_resources;
        if recipients then
            for i=1,#recipients do
                recipients[i].send(stanza);
            end
            return true;
        end
    end
    
    -- Пользователь офлайн
    module:fire_event('message/offline/handle', {
        username = node,
        origin = origin,
        stanza = stanza,
    });
end
```

## Сценарий 2: Удаленная доставка (разные серверы)

### Исходные данные
- Отправитель: `alice@server1.com/phone`
- Получатель: `bob@server2.com/laptop`
- Разные серверы

### Последовательность обработки

```mermaid
sequenceDiagram
    participant A as Alice Client
    participant S1 as Server 1<br/>(Prosody)
    participant S2 as Server 2<br/>(Remote XMPP)
    participant B as Bob Client
    
    A->>S1: message stanza<br/>(to: bob@server2.com)
    
    S1->>S1: core_process_stanza()
    S1->>S1: core_post_stanza()
    S1->>S1: Определение: удаленный хост
    
    S1->>S1: core_route_stanza()
    
    alt S2S соединение существует
        S1->>S2: Отправка через существующее s2s
    else S2S соединение отсутствует
        S1->>S1: Создание s2sout сессии
        S1->>S1: DNS резолвинг server2.com
        S1->>S2: TCP соединение на порт 5269
        S2->>S1: Открытие XMPP потока
        S1->>S2: STARTTLS
        S2->>S1: TLS handshake
        S1->>S2: SASL или Dialback аутентификация
        S2->>S1: Подтверждение аутентификации
        Note over S1,S2: S2S соединение установлено
        S1->>S2: Отправка сообщения
    end
    
    S2->>S2: core_process_stanza()
    S2->>S2: Проверка авторизации from_host
    S2->>S2: core_post_stanza()
    S2->>B: Доставка сообщения
    B->>S2: Подтверждение
```

### Детальный код маршрутизации

#### 1. Определение удаленного хоста (core/stanza_router.lua)

```lua
function core_route_stanza(origin, stanza)
    local to_host = jid_host(stanza.attr.to);
    local from_host = jid_host(stanza.attr.from);
    
    if hosts[to_host] then
        -- Локальный хост
        core_post_stanza(origin, stanza);
    else
        -- Удаленный хост
        local host_session = hosts[from_host];
        host_session.events.fire_event("route/remote", {
            origin = origin,
            stanza = stanza,
            from_host = from_host,
            to_host = to_host
        });
    end
end
```

#### 2. Обработка удаленной маршрутизации (plugins/mod_s2s.lua)

```lua
module:hook("route/remote", function(event)
    local from_host = event.from_host;
    local to_host = event.to_host;
    local stanza = event.stanza;
    
    -- Проверка существующего соединения
    local session = hosts[from_host].s2sout[to_host];
    
    if session and session.type == "s2sout" then
        -- Использование существующего соединения
        session.send(stanza);
        return true;
    end
    
    -- Создание нового соединения
    if not session or session.type == "s2sout_unauthed" then
        -- Инициирование соединения
        initiate_s2s_connection(from_host, to_host);
        -- Очередь stanza для отправки после установления соединения
        queue_stanza(session, stanza);
    end
end);
```

## Сценарий 3: Сообщение на full JID

### Исходные данные
- Отправитель: `alice@example.com/phone`
- Получатель: `bob@example.com/laptop` (конкретный ресурс)

### Последовательность обработки

```mermaid
sequenceDiagram
    participant A as Alice Client
    participant SR as Stanza Router
    participant MSG as mod_message
    participant SM as Session Manager
    participant B as Bob Client<br/>(laptop)
    
    A->>SR: message stanza<br/>(to: bob@example.com/laptop)
    
    SR->>SR: core_process_stanza()
    SR->>SR: Определение типа: /full
    
    SR->>MSG: fire_event("message/full")
    
    MSG->>SM: Проверка: full_sessions["bob@example.com/laptop"]
    
    alt Ресурс онлайн
        SM->>B: Прямая доставка на ресурс
        B->>A: Подтверждение
    else Ресурс офлайн
        MSG->>MSG: process_to_bare("bob@example.com")
        Note over MSG: Обработка как bare JID:<br/>доставка на другие ресурсы<br/>или сохранение офлайн
    end
```

### Код обработки full JID

```lua
module:hook("message/full", function(data)
    local origin, stanza = data.origin, data.stanza;
    
    -- Попытка доставки на конкретный ресурс
    local session = full_sessions[stanza.attr.to];
    if session and session.send(stanza) then
        return true;  -- Доставлено
    end
    
    -- Ресурс не найден, обработка как bare JID
    return process_to_bare(jid_bare(stanza.attr.to), origin, stanza);
end, -1);
```

## Сценарий 4: Сообщение без 'to' атрибута

### Обработка сообщений без указания получателя

```mermaid
graph TD
    START[Получение stanza без 'to'] --> CHECK_TYPE{Тип stanza}
    
    CHECK_TYPE -->|IQ| IQ_HANDLER[Передача IQ обработчику]
    CHECK_TYPE -->|Presence| PRES_BROADCAST[Broadcast контактам]
    CHECK_TYPE -->|Message| MSG_SELF[Маршрутизация на bare JID<br/>отправителя]
    
    PRES_BROADCAST --> INIT_PRES{Initial presence?}
    INIT_PRES -->|Да| PROBE[Отправка presence probes]
    INIT_PRES -->|Нет| BROADCAST[Broadcast всем контактам]
    
    MSG_SELF --> BARE_ROUTE[Обработка как bare JID]
```

### Код обработки (doc/stanza_routing.txt)

```
No 'to' attribute:
	IQ:			Pass to appropriate handler
	Presence:		Broadcast to contacts
				- if initial presence, also send out presence probes
					- if probe would be to local user, generate presence stanza for them
	Message:		Route as if it is addressed to the bare JID of the sender
```

## Сценарий 5: Обработка ошибок

### Типы ошибок при доставке

```mermaid
graph TD
    ERROR[Ошибка доставки] --> CHECK_TYPE{Тип ошибки}
    
    CHECK_TYPE -->|JID malformed| JID_ERROR[Отправка error:<br/>jid-malformed]
    CHECK_TYPE -->|Resource not found| RES_ERROR[Отправка error:<br/>service-unavailable]
    CHECK_TYPE -->|Remote server not found| REMOTE_ERROR[Отправка error:<br/>remote-server-not-found]
    CHECK_TYPE -->|Not authorized| AUTH_ERROR[Отправка error:<br/>not-authorized]
    
    JID_ERROR --> REPLY[Формирование error reply]
    RES_ERROR --> REPLY
    REMOTE_ERROR --> REPLY
    AUTH_ERROR --> REPLY
    
    REPLY --> SEND[Отправка отправителю]
```

### Формат error reply

```xml
<message type="error" to="alice@example.com/phone" from="bob@example.com">
    <error type="cancel" condition="service-unavailable">
        <text>Resource is not available</text>
    </error>
</message>
```

## Оптимизации

### Кэширование сессий

- `full_sessions` - быстрый поиск по full JID
- `bare_sessions` - доступ ко всем ресурсам пользователя
- `top_resources` - предварительно отсортированные ресурсы по приоритету

### Очередь отправки (sendq)

Для s2s соединений используется очередь отправки:
- Станзы помещаются в очередь до установления соединения
- После установления соединения все станзы отправляются
- При ошибке соединения станзы возвращаются с error reply

### Приоритет ресурсов

Алгоритм выбора ресурса:
1. Фильтрация по presence (только онлайн)
2. Сортировка по priority (выше = лучше)
3. При равном priority - первый подключенный

## Логирование

### Точки логирования

1. **Получение stanza**: `core_process_stanza` логирует входящие станзы
2. **Маршрутизация**: `core_route_stanza` логирует решения маршрутизации
3. **Доставка**: модули логируют успешную доставку
4. **Ошибки**: все ошибки логируются с деталями

### Уровни логирования

- `debug` - детальная информация о каждом шаге
- `info` - важные события (доставка, ошибки)
- `warn` - предупреждения (неавторизованные запросы)
- `error` - критические ошибки
