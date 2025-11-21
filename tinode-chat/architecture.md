# Анализ архитектуры Tinode Chat

## Обзор проекта

Tinode Chat - это сервер мгновенных сообщений с архитектурой publish-subscribe, написанный на Go. Система поддерживает одноранговые (P2P) чаты, групповые чаты, каналы с подписчиками, а также поиск пользователей и топиков.

## Основные компоненты системы

### 1. Hub (Центральный роутер)
**Файл:** `server/hub.go`

Hub - это центральный компонент, который управляет всеми топиками и маршрутизирует сообщения между сессиями и топиками.

**Основные функции:**
- Управление жизненным циклом топиков
- Маршрутизация сообщений между топиками
- Обработка подписок/отписок
- Координация кластерной работы

**Ключевые каналы:**
- `routeCli` - маршрутизация клиентских сообщений (буфер: 4096)
- `routeSrv` - маршрутизация серверных сообщений (буфер: 4096)
- `join` - запросы на подписку к топикам (буфер: 256)
- `unreg` - удаление топиков (буфер: 256)

### 2. Topic (Каналы коммуникации)
**Файл:** `server/topic.go`

Topic представляет **изолированный канал коммуникации** между пользователями. Это основная единица маршрутизации сообщений в Tinode. Каждый топик работает в отдельной горутине и обрабатывает все запросы асинхронно через каналы (channels).

#### Что такое топик?

Топик - это абстракция чата или канала коммуникации. В терминах Tinode:
- **Один топик = один чат/беседа**
- Топики полностью изолированы друг от друга (нет прямого общения между топиками)
- Каждый топик управляет своими подписчиками и сообщениями
- Топик хранит состояние всех подписчиков и их права доступа

#### Структура данных топика

```go
type Topic struct {
    // Имя топика (уникальный идентификатор)
    name string      // Например: "p2pABC123XYZ", "grpDEF456", "usr789"
    xoriginal string // Оригинальное имя для клиента (например "me")
    
    // Категория топика
    cat types.TopicCat // Me, Fnd, P2P, Grp, Sys, Slf
    
    // Временные метки
    created time.Time // Когда создан
    updated time.Time // Когда обновлен
    touched time.Time // Когда последний раз отправлено сообщение
    
    // Состояние сообщений
    lastID int  // ID последнего сообщения (последовательный номер)
    delID  int  // ID последней операции удаления
    
    // Владелец топика (только для групповых)
    owner types.Uid
    
    // Права доступа по умолчанию
    accessAuth types.AccessMode // Для аутентифицированных
    accessAnon types.AccessMode // Для анонимных
    
    // Данные топика
    public  any  // Публичные данные (видны всем)
    trusted any  // Доверенные данные (видны только серверу)
    tags    []string // Теги для поиска
    
    // Данные подписчиков (ключ - UID пользователя)
    perUser map[types.Uid]perUserData
    
    // Прикрепленные сессии (ключ - указатель на Session)
    sessions map[*Session]perSessionData
    
    // Контакты пользователя (только для 'me' топика)
    perSubs map[string]perSubsData
    
    // Каналы для коммуникации
    clientMsg chan *ClientComMessage  // Сообщения от клиентов
    serverMsg chan *ServerComMessage  // Сообщения от сервера
    reg       chan *ClientComMessage  // Запросы на подписку
    unreg     chan *ClientComMessage  // Запросы на отписку
    meta      chan *ClientComMessage  // Метаданные (get/set/del)
    supd      chan *sessionUpdate     // Обновления сессий
    exit      chan *shutDown          // Завершение работы
    
    // Кластерная поддержка
    isProxy    bool      // Если true - это прокси-топик на удаленном узле
    masterNode string    // Имя узла-мастера (для прокси)
    proxy      chan *ClusterResp      // Ответы от мастера
    master     chan *ClusterSessUpdate // Запросы от прокси
    
    // Флаги состояния
    status int32 // new, ready, paused, deleted
    isChan bool  // Канал (read-only группа)
}
```

#### Типы топиков

**1. `me` (Personal Topic)**
- Личный топик каждого пользователя
- Используется для управления профилем и получения уведомлений
- Имя: `usr<user_id>` (например `usr2il9suCbuko`)
- Клиент видит его как `"me"`
- Особенности:
  - Автоматически создается для каждого пользователя
  - Хранит список контактов (`perSubs`)
  - Получает presence-уведомления о других топиках
  - Отслеживает User Agent устройств

**2. `fnd` (Find Topic)**
- Топик для поиска пользователей и топиков
- Имя: `usr<user_id>` (тот же что у `me`)
- Клиент видит его как `"fnd"`
- Особенности:
  - Используется для поиска по тегам
  - Не поддерживает публикацию сообщений

**3. `p2p` (Peer-to-Peer Topic)**
- Чат между двумя пользователями
- Имя: `p2p<uid1><uid2>` (отсортированные UID)
- Пример: `p2pABC123XYZ789`
- Особенности:
  - Автоматически создается при первом сообщении
  - Каждый участник видит имя топика как UID другого участника
  - Поддерживает видео-звонки (`currentCall`)
  - В P2P каждый участник имеет свой `public` и `trusted` (обмен данными)

**4. `grp` (Group Topic)**
- Групповой чат с множеством участников
- Имя: `grp<random_11_chars>` (например `grpYiqEXb4QY6s`)
- Особенности:
  - Создается явно пользователем
  - Имеет владельца (`owner`)
  - Максимум 256 подписчиков (по умолчанию)
  - Поддерживает каналы (`isChan = true`) - read-only группы

**5. `sys` (System Topic)**
- Системный топик для служебных сообщений
- Имя: `"sys"`
- Особенности:
  - Один на весь сервер
  - Все могут писать без подписки
  - Используется для системных уведомлений

**6. `slf` (Self Topic)**
- Топик для заметок себе и сохраненных сообщений
- Имя: `usr<user_id>`
- Клиент видит его как `"slf"`

#### Жизненный цикл топика

```mermaid
stateDiagram-v2
    [*] --> NotLoaded: Запрос подписки {sub}
    
    NotLoaded --> Creating: Hub создает структуру Topic
    Creating --> Initializing: topicInit() вызывается в goroutine
    
    Initializing --> Loading: Загрузка из БД или создание
    Loading --> Loaded: Загрузка успешна
    Loading --> Failed: Ошибка загрузки
    
    Loaded --> Ready: markPaused(false)
    Ready --> Active: go t.run(hub) запускает горутину
    Ready --> Loading: Если требуется донастройка
    
    Active --> Active: Обработка сообщений через каналы
    Active --> Paused: markPaused(true)
    Active --> Deleted: Получен сигнал exit
    
    Paused --> Active: markPaused(false)
    Paused --> Deleted: Сигнал удаления
    
    Deleted --> ShuttingDown: Завершение работы
    ShuttingDown --> [*]: Очистка ресурсов
    
    Failed --> [*]: Ошибка инициализации
```

#### Создание и инициализация топика

**Процесс создания:**

1. **Клиент отправляет `{sub}`** запрос на подписку к топику
2. **Session** получает запрос и отправляет в Hub через канал `hub.join`
3. **Hub** проверяет, загружен ли топик:
   - Если нет - создает структуру `Topic` с каналами
   - Устанавливает статус `paused` (приостановлен)
   - Сохраняет топик в кеш: `hub.topics.Store(name, t)`
   - Запускает `topicInit()` в отдельной горутине

4. **topicInit()** определяет тип топика и вызывает соответствующую функцию:
   ```go
   switch {
   case t.xoriginal == "me":
       initTopicMe(t, join)
   case strings.HasPrefix(t.xoriginal, "p2p"):
       initTopicP2P(t, join)
   case strings.HasPrefix(t.xoriginal, "new"):
       initTopicNewGrp(t, join, false)
   // ... и т.д.
   }
   ```

5. **Инициализация** загружает данные из БД:
   - Подписчиков (`loadSubscribers()`)
   - Метаданные топика
   - Права доступа каждого подписчика
   - Состояние сообщений (`lastID`, `delID`)

6. **После успешной инициализации:**
   - `markPaused(false)` - топик активируется
   - `go t.run(hub)` - запускается главный цикл обработки

#### Главный цикл обработки топика

Каждый топик работает в своей горутине и обрабатывает события через select:

```go
func (t *Topic) runLocal(hub *Hub) {
    for {
        select {
        case msg := <-t.clientMsg:
            // Сообщение от клиента (pub, note, etc)
            t.handleClientMessage(msg)
            
        case msg := <-t.serverMsg:
            // Сообщение от сервера (из кластера)
            t.handleServerMessage(msg)
            
        case join := <-t.reg:
            // Запрос на подписку
            t.handleSubscription(join)
            
        case leave := <-t.unreg:
            // Запрос на отписку
            t.handleUnsubscription(leave)
            
        case meta := <-t.meta:
            // Запрос метаданных (get/set/del)
            t.handleMetaRequest(meta)
            
        case update := <-t.supd:
            // Обновление сессии (UA, background -> foreground)
            t.handleSessionUpdate(update)
            
        case shutdown := <-t.exit:
            // Завершение работы
            t.cleanup()
            return
        }
    }
}
```

#### Как топик обрабатывает сообщение

**Шаг 1: Получение сообщения**
```
Клиент → Session.publish() → sub.broadcast (канал) → Topic.clientMsg
```

**Шаг 2: Проверка прав доступа**
```go
pud := t.perUser[asUid]
if !(pud.modeWant & pud.modeGiven).IsWriter() {
    return ErrPermissionDenied
}
```

**Шаг 3: Сохранение в БД**
```go
store.Messages.Save(&types.Message{
    SeqId: t.lastID + 1,
    Topic: t.name,
    From:  asUid.String(),
    Content: content,
})
t.lastID++
```

**Шаг 4: Рассылка подписчикам**
```go
t.broadcastToSessions(&ServerComMessage{
    Data: &MsgServerData{
        Topic: msg.Original,
        From:  msg.AsUser,
        SeqId: t.lastID,
        Content: content,
    },
})
```

**Шаг 5: Отправка каждому подписчику**
```go
for sess, pssd := range t.sessions {
    if t.userIsReader(pssd.uid) {
        sess.queueOut(msgCopy) // Отправка в сессию
    }
}
```

#### Права доступа (Access Control)

Каждый подписчик имеет два набора прав:
- **`modeWant`** - желаемые права
- **`modeGiven`** - предоставленные права

**Эффективные права = `modeWant & modeGiven`**

**Базовые права:**
- **Read (R)** - чтение сообщений
- **Write (W)** - отправка сообщений
- **Join (J)** - возможность подписки
- **Owner (O)** - владелец топика
- **Delete (D)** - удаление сообщений
- **Approve (A)** - одобрение запросов на подписку

**Пример:**
```go
// Пользователь хочет читать и писать
modeWant = "JRWS"

// Ему предоставлено только чтение
modeGiven = "JR"

// Эффективные права = "JR" (только чтение)
effective = modeWant & modeGiven = "JR"
```

#### Уничтожение топика

Топик уничтожается когда:
1. Нет активных сессий + таймер истек (`killTimer`)
2. Получен сигнал удаления (`exit` с reason `StopDeleted`)
3. Система завершает работу (`StopShutdown`)
4. Кластер ребалансировка (`StopRehashing`)

**Процесс уничтожения:**
1. `markDeleted()` - устанавливает флаг
2. Отключение всех сессий
3. Очистка каналов
4. Удаление из кеша Hub
5. Горутина завершается

### 3. Session (Сессии клиентов)
**Файл:** `server/session.go`

Session представляет соединение между клиентом и сервером.

**Типы протоколов:**
- `WEBSOCK` - WebSocket соединение
- `LPOLL` - Long Polling
- `GRPC` - gRPC соединение
- `PROXY` - прокси-сессия в кластерном режиме
- `MULTIPLEX` - мультиплексирующая сессия для кластера

**Основные функции:**
- Прием и отправка сообщений клиенту
- Управление подписками на топики
- Обработка аутентификации
- Сериализация/десериализация сообщений

**Ключевые каналы:**
- `send` - исходящие сообщения к клиенту (буферизованный)
- `stop` - канал для остановки сессии
- `detach` - отключение от топиков

### 4. Store (Хранилище данных)
**Файл:** `server/store/store.go`

Абстракция для работы с базами данных. Поддерживает несколько бэкендов:
- MySQL/MariaDB
- PostgreSQL
- MongoDB
- RethinkDB (deprecated)

**Основные функции:**
- Сохранение и загрузка сообщений
- Управление пользователями и топиками
- Управление подписками
- Управление файлами и вложениями

### 5. Cluster (Кластерная поддержка)
**Файл:** `server/cluster.go`

Обеспечивает горизонтальное масштабирование через распределение топиков между узлами кластера.

**Основные функции:**
- Распределение топиков по узлам (consistent hashing)
- Маршрутизация сообщений между узлами
- Репликация данных между узлами
- Обработка отказов узлов

### 6. Push Notifications
**Файл:** `server/push.go`

Система push-уведомлений для офлайн пользователей. Поддерживает:
- FCM (Firebase Cloud Messaging)
- TnPG (Tinode Push Gateway)
- Stdout (для отладки)

## Архитектурная диаграмма системы

```mermaid
graph TB
    subgraph "Клиенты"
        WS[WebSocket Клиент]
        LP[Long Polling Клиент]
        GRPC[gRPC Клиент]
    end
    
    subgraph "HTTP/gRPC Layer"
        HTTP[HTTP Server]
        WSHandler[WebSocket Handler]
        LPHandler[Long Poll Handler]
        GRPCHandler[gRPC Handler]
    end
    
    subgraph "Session Layer"
        SessStore[Session Store]
        Sess1[Session 1]
        Sess2[Session 2]
        SessN[Session N]
    end
    
    subgraph "Hub Layer"
        Hub[Hub<br/>Центральный роутер]
    end
    
    subgraph "Topic Layer"
        Topic1[Topic: p2pXXX]
        Topic2[Topic: grpYYY]
        TopicMe[Topic: me]
        TopicSys[Topic: sys]
    end
    
    subgraph "Storage Layer"
        Store[(Database<br/>MySQL/PostgreSQL/MongoDB)]
        Media[Media Handler<br/>FS/S3]
    end
    
    subgraph "External Services"
        Push[Push Service<br/>FCM/TnPG]
        Auth[Auth Backend]
    end
    
    subgraph "Cluster"
        Node1[Node 1]
        Node2[Node 2]
        NodeN[Node N]
    end
    
    WS --> WSHandler
    LP --> LPHandler
    GRPC --> GRPCHandler
    
    WSHandler --> SessStore
    LPHandler --> SessStore
    GRPCHandler --> SessStore
    
    SessStore --> Sess1
    SessStore --> Sess2
    SessStore --> SessN
    
    Sess1 --> Hub
    Sess2 --> Hub
    SessN --> Hub
    
    Hub --> Topic1
    Hub --> Topic2
    Hub --> TopicMe
    Hub --> TopicSys
    
    Topic1 --> Store
    Topic2 --> Store
    TopicMe --> Store
    
    Topic1 --> Media
    Topic2 --> Media
    
    Topic1 --> Push
    Topic2 --> Push
    
    Hub --> Node1
    Hub --> Node2
    Node1 --> Node2
    Node2 --> NodeN
    
    SessStore --> Auth
```

## Детальный поток сообщения от одного участника другому

```mermaid
sequenceDiagram
    participant C1 as Клиент 1<br/>(Отправитель)
    participant S1 as Session 1
    participant H as Hub
    participant T as Topic<br/>(p2pXXX или grpYYY)
    participant DB as Database
    participant T2 as Topic.me<br/>(Получатель)
    participant S2 as Session 2
    participant C2 as Клиент 2<br/>(Получатель)
    participant P as Push Service
    
    Note over C1,C2: 1. Отправка сообщения
    
    C1->>S1: {pub} message<br/>(WebSocket/LongPoll/gRPC)
    Note over S1: Валидация и обработка
    S1->>S1: expandTopicName()<br/>Расширение имени топика
    
    alt Подписка существует
        S1->>T: sub.broadcast <- msg<br/>(канал Subscription)
    else Публикация в sys
        S1->>H: hub.routeCli <- msg
        H->>T: topic.clientMsg <- msg
    end
    
    Note over T: 2. Обработка в Topic
    
    T->>T: Проверка прав доступа<br/>(modeWant & modeGiven).IsWriter()
    
    T->>DB: store.Messages.Save()<br/>Сохранение сообщения
    DB-->>T: Сообщение сохранено<br/>SeqId присвоен
    
    T->>T: t.lastID++<br/>Увеличение счетчика
    T->>T: saveAndBroadcastMessage()
    
    T-->>S1: {ctrl} NoErrAccepted<br/>Подтверждение отправки
    
    Note over T: 3. Рассылка подписчикам
    
    T->>T: broadcastToSessions()<br/>Итерация по sessions
    
    loop Для каждой сессии подписчика
        T->>T: Проверка прав чтения<br/>userIsReader()
        T->>T: prepareBroadcastableMessage()<br/>Адаптация для пользователя
        T->>S2: queueOut(msgCopy)
        S2->>S2: Сериализация в JSON
        S2->>C2: {data} message<br/>(WebSocket/LongPoll/gRPC)
    end
    
    Note over T: 4. Push уведомления
    
    T->>T: pushForData()<br/>Определение офлайн подписчиков
    
    alt Получатель офлайн
        T->>P: sendPush()<br/>Push уведомление
        P-->>C2: Push notification<br/>(FCM/TnPG)
    end
    
    Note over T2: 5. Presence уведомления
    
    T->>T2: presSubsOffline("msg")<br/>Уведомление на 'me' топик
    
    loop Для каждого подписчика на 'me'
        T2->>S2: {pres} message<br/>Новое сообщение в топике
        S2->>C2: {pres} notification
    end
    
    Note over C1,C2: 6. Получение подтверждений
    
    C2->>S2: {note what="recv"}<br/>Получено
    S2->>T: Обработка подтверждения
    T->>DB: Обновление RecvSeqId
    
    C2->>S2: {note what="read"}<br/>Прочитано
    S2->>T: Обработка подтверждения
    T->>DB: Обновление ReadSeqId
    T->>S1: {info} notification<br/>Сообщение прочитано
    S1->>C1: {info} notification
```

## Детальная архитектура обработки сообщения в Topic

```mermaid
flowchart TD
    Start[Клиент отправляет pub]
    
    Session[Session.publish]
    
    CheckSub{Подписка существует?}
    
    ToSub[Отправка в sub.broadcast]
    ToHub[Отправка в hub.routeCli]
    
    HubRoute[Hub.routeCli]
    HubToTopic[Hub -> Topic.clientMsg]
    
    TopicReceive[Topic получает сообщение]
    
    CheckAccess{Проверка прав IsWriter?}
    
    AccessDenied[Отправить ErrPermissionDenied]
    
    SaveMsg[store.Messages.Save Сохранение в БД]
    
    DBResponse{Успешно сохранено?}
    
    DBError[Отправить ErrUnknown]
    
    UpdateSeq[Обновление t.lastID lastID++]
    
    SendAck[Отправка ctrl NoErrAccepted]
    
    CreateDataMsg[Создание ServerComMessage типа data]
    
    Broadcast[broadcastToSessions Рассылка подписчикам]
    
    ForEachSess[Для каждой сессии sessions map]
    
    CheckRead{Права чтения? userIsReader}
    
    PrepareMsg[prepareBroadcastableMessage Адаптация сообщения]
    
    QueueToSess[queueOut в Session]
    
    SessSerial[Сериализация в JSON в Session]
    
    SendToClient[Отправка data клиенту]
    
    CheckPush{Офлайн подписчики?}
    
    SendPush[sendPush Push уведомления]
    
    PresNotify[presSubsOffline Presence на me]
    
    PluginNotify[pluginMessage Уведомление плагинов]
    
    End[Сообщение доставлено]
    
    Start --> Session
    Session --> CheckSub
    CheckSub -->|Да| ToSub
    CheckSub -->|Нет| ToHub
    ToHub --> HubRoute
    HubRoute --> HubToTopic
    ToSub --> TopicReceive
    HubToTopic --> TopicReceive
    
    TopicReceive --> CheckAccess
    CheckAccess -->|Нет| AccessDenied
    CheckAccess -->|Да| SaveMsg
    
    SaveMsg --> DBResponse
    DBResponse -->|Ошибка| DBError
    DBResponse -->|Успех| UpdateSeq
    
    UpdateSeq --> SendAck
    SendAck --> CreateDataMsg
    
    CreateDataMsg --> PluginNotify
    PluginNotify --> Broadcast
    
    Broadcast --> ForEachSess
    ForEachSess --> CheckRead
    CheckRead -->|Да| PrepareMsg
    CheckRead -->|Нет| ForEachSess
    
    PrepareMsg --> QueueToSess
    QueueToSess --> SessSerial
    SessSerial --> SendToClient
    
    SendToClient --> CheckPush
    CheckPush -->|Да| SendPush
    CheckPush -->|Нет| PresNotify
    
    SendPush --> PresNotify
    PresNotify --> End
    AccessDenied --> End
    DBError --> End
```

## Структура данных сообщения

```mermaid
classDiagram
    class ClientComMessage {
        +string Id
        +string Topic
        +string RcptTo
        +string Original
        +string AsUser
        +PubMessage Pub
        +Session sess
    }
    
    class ServerComMessage {
        +string Id
        +string RcptTo
        +DataMessage Data
        +PresMessage Pres
        +InfoMessage Info
        +CtrlMessage Ctrl
        +MetaMessage Meta
        +Session sess
        +string SkipSid
    }
    
    class Topic {
        +string name
        +int lastID
        +map[Uid]perUserData perUser
        +map[*Session]perSessionData sessions
        +chan ClientComMessage clientMsg
        +chan ServerComMessage serverMsg
    }
    
    class Session {
        +string sid
        +Uid uid
        +SessionProto proto
        +chan any send
        +map[string]*Subscription subs
    }
    
    class Subscription {
        +chan~*ClientComMessage~ broadcast
        +chan~*ClientComMessage~ done
        +chan~*ClientComMessage~ meta
        +chan~*sessionUpdate~ supd
    }
    
    ClientComMessage --> Topic : отправляется в
    Topic --> ServerComMessage : создает
    ServerComMessage --> Session : отправляется в
    Session --> Subscription : использует
    Topic --> Session : содержит
    Session --> Subscription : имеет
```

## Жизненный цикл топика

```mermaid
stateDiagram-v2
    [*] --> NotLoaded: Запрос подписки
    
    NotLoaded --> Initializing: Создание Topic
    
    Initializing --> Loading: topicInit()<br/>Загрузка из БД
    
    Loading --> Ready: Загрузка успешна
    Loading --> Failed: Ошибка загрузки
    
    Ready --> Active: Session attached
    
    Active --> Paused: markPaused(true)<br/>Приостановка
    Active --> Active: Обработка сообщений
    Active --> Deleted: Удаление топика
    
    Paused --> Active: markPaused(false)<br/>Возобновление
    Paused --> Deleted: Удаление
    
    Ready --> Deleted: Удаление
    
    Deleted --> [*]: Завершение
    Failed --> [*]: Завершение
```

## Кластерная архитектура (распределение топиков)

```mermaid
graph TB
    subgraph "Cluster Node 1"
        Hub1[Hub]
        Topic1A[Topic A<br/>Master]
        Topic1B[Topic B<br/>Proxy]
        Sess1A[Session 1A]
        Sess1B[Session 1B]
    end
    
    subgraph "Cluster Node 2"
        Hub2[Hub]
        Topic2A[Topic A<br/>Proxy]
        Topic2B[Topic B<br/>Master]
        Sess2A[Session 2A]
        Sess2B[Session 2B]
    end
    
    subgraph "Consistent Hash Ring"
        Ring[Ring Hash<br/>Распределение топиков]
    end
    
    Ring -->|Topic A → Node 1| Hub1
    Ring -->|Topic B → Node 2| Hub2
    
    Hub1 --> Topic1A
    Hub1 --> Topic1B
    
    Hub2 --> Topic2A
    Hub2 --> Topic2B
    
    Sess1A --> Topic1A
    Sess1B --> Topic1B
    Sess2A --> Topic2A
    Sess2B --> Topic2B
    
    Topic1B -.->|Proxy request| Topic2B
    Topic2A -.->|Proxy request| Topic1A
    
    Topic1A -.->|Broadcast| Sess2A
    Topic2B -.->|Broadcast| Sess1B
```

## Особенности реализации

### 1. Каналы и горутины
- Все компоненты общаются через каналы (channels)
- Каждый Topic работает в отдельной горутине
- Каждая Session имеет горутины для чтения и записи
- Hub работает в главной горутине, координируя работу всех топиков

### 2. Буферизация
- Каналы буферизованы для предотвращения блокировок
- При переполнении буфера отправляется ошибка 503 (Service Unavailable)
- Логирование предупреждений при переполнении

### 3. Права доступа (Access Control)
- Каждый подписчик имеет `modeWant` (желаемые права) и `modeGiven` (предоставленные права)
- Эффективные права = `modeWant & modeGiven`
- Права включают: Read (R), Write (W), Join (J), Owner (O), Delete (D), Approve (A)

### 4. Сохранение сообщений
- Сообщения сохраняются синхронно перед рассылкой
- Каждому сообщению присваивается уникальный `SeqId`
- Поддерживаются вложения (attachments)
- Отправитель может пометить сообщение как прочитанное сразу

### 5. Push уведомления
- Отправляются только для офлайн подписчиков
- Определяются через проверку активных сессий
- Поддерживается несколько провайдеров (FCM, TnPG, Stdout)

### 6. Presence уведомления
- Отправляются на топик `me` для всех пользователей
- Уведомляют о новых сообщениях в подписанных топиках
- Фильтруются по правам доступа

## Производительность и масштабируемость

1. **Горизонтальное масштабирование**: Поддержка кластеризации через consistent hashing
2. **Асинхронная обработка**: Использование каналов и горутин для параллельной обработки
3. **Буферизация**: Буферизованные каналы для сглаживания пиковых нагрузок
4. **Кэширование**: Топики кэшируются в памяти Hub
5. **Пакетная обработка**: Поддержка батчинга сообщений в некоторых протоколах

## Заключение

Tinode Chat использует архитектуру publish-subscribe с центральным Hub для маршрутизации сообщений. Каждый топик обрабатывается независимо в своей горутине, что обеспечивает высокую производительность и масштабируемость. Сообщения сохраняются синхронно в базу данных, а затем асинхронно рассылаются всем подписчикам через их сессии.

