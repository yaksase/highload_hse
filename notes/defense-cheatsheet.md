# Defense cheat-sheet

Рабочая шпаргалка для устной защиты РПЗ. Сюда уходят полезные обоснования и контр-аргументы, которые **не входят в финальный текст РПЗ** (там уже всё обосновано основными буллетами и таблицами), но могут пригодиться, если препод задаст уточняющий вопрос.

> Это **не** то же самое, что `notes/section-7-10-drafts.md` — там материал, который **переедет** в §7/§10 РПЗ. Здесь — материал, который **не пойдёт** в РПЗ ни в каком виде, останется только в голове на защите.

---

## §3 Глобальная балансировка

### «А зачем нам Anycast вообще, если можно GeoDNS?»

* **Один endpoint для всего мира** — клиент не знает про регионы, у нас один публичный IP. Не нужно поддерживать список региональных IP в DNS, не нужно фронтенду гадать, какой эндпоинт отдать клиенту.
* **Сетевой failover за 30–90 сек** при отказе ДЦ — анонс снимается, пакеты автоматически уходят к следующему ближайшему ДЦ. DNS-TTL ждать не нужно (а у GeoDNS реальный TTL — наш 60 сек + кеш мобильных операторов 5–15 минут).
* **DDoS-распределение** — атака размазывается по 4 точкам входа (по одной на ДЦ), нет единой точки перегрузки. У GeoDNS все клиенты одного региона указаны на один origin IP — точка сосредоточения.
* **Нет проблем GeoDNS** — ~10–15% мобильных resolver-ов имеют ошибочную IP-геолокацию (EU-клиент получает US-IP, потому что resolver мобильного оператора стоит в другой стране). Anycast от этого не страдает: выбор ДЦ делает сеть на уровне BGP, а не словарь резолвера.

### «А почему вы не делаете Anycast просто на уровне DNS-серверов и обычные IP origin'ов?»

Это GeoDNS + Anycast NS. Это **дороже и хуже**:
- Два TTL (TTL DNS + TTL клиентского кеша), failover не быстрее минут.
- При смене провайдера CDN/Anycast надо менять оба слоя.
- DNS-geoloc мобильных всё равно сломан.

Сейчас у нас **только Anycast NS** (для резервирования самого DNS) + **Anycast IP origin'ов** — оба сервиса на anycast, но независимо.

### «Почему именно 4 ДЦ, не 3 и не 6?»

Числовой ответ:
- **3 ДЦ** (US East + EU + APAC) — failover 1-в-2, при отказе одного оставшиеся на 150% нагрузки → деградация. Нет N+1.
- **4 ДЦ** (US East + US West + EU + APAC) — N+1 на уровне регионов: 3 из 4 держат пиковую нагрузку.
- **6 ДЦ** — каждый дополнительный сверх 4 экономит RTT 5–15 мс, что <3% от общего TTFT (250–400 мс). Не оправдывает удвоения сетевой инфраструктуры.

### «А что если упадут оба US-кластера?»

Глобальный inference-outage, новые промпты возвращают `503 Service Unavailable`. EU/APAC продолжают работать как HTTP-слой: история чатов, login, UI доступны. Это компромисс — вторая GPU-площадка вне США удвоила бы capex/opex и не оправдана при разнесении US East/West по разным AZ/регионам.

Реально вероятность одновременного отказа US East и US West:
- Разные облачные регионы (Virginia + Oregon, физически 4000 км).
- Независимые power grids, network providers, AZ.
- Скоординированный отказ — только при глобальной катастрофе (cyber-атака на cloud provider или сетевой backbone).

### «А почему GPU дешевле в США? У AWS p5.48xlarge цена на час почти одинакова везде.»

Главный аргумент **не цена за инстанс** (там разница 10–15%), а **утилизация**:
- GPU-inference экономически выгоден только при continuous batching с высокой утилизацией.
- 4 GPU-региона = очередь поделена на 4 = utilization падает с 80% до ~50%.
- Для той же пропускной способности нужно вдвое больше H100.
- На масштабе тысяч GPU это $10М+ capex/год сверху.

Цена инстанса — вторичный фактор; **availability** свежих GPU (capacity и quotas в US доступны быстрее и большими объёмами) — третий.

### «А почему не использовать Cloudflare/AWS Global Accelerator? Они делают то же самое.»

Используем те же концепции (Anycast IP, BGP, health-check регионов), но:
- На защите легче объяснять generic-механизм (BIRD2 + ExaBGP), чем «магию Cloudflare».
- В РПЗ vendor lock-in — минус балл, потому что нужно будет защищать выбор именно этого вендора (а у нас нет такого требования в MVP).
- Технически наш родовой стек = Cloudflare минус DDoS-сеть PoP-ов, что приемлемо для MVP.

---

## §4 Локальная балансировка

### «Почему только L7 на входе ДЦ, а не L4 + L7 как в классике?»

Главная причина — **профиль трафика**: 1.2 млн concurrent SSE peak. Лекция прямо: «L4: выбор бэкенда в момент установки соединения, дисбаланс при небольшом числе соединений; смещение соединений при мигании бэкендов». L4 (LVS/IPVS) не сможет ровно размазать долгоживущие SSE-стримы. L7 видит каждый запрос, может балансить по `least_conn` / `ewma`, умеет TLS termination, sticky cookies, `proxy_next_upstream`.

Платим «относительно низкой производительностью» — но узкое место у нас всё равно TLS handshake (~6 600 CPS на pod, §4.3), а не линейная скорость сети.

Дополнительный L4 на edge-роутере (LVS-DR) добавил бы hop без выигрыша: его задачи (приём Anycast IP, распределение по edge-нодам) уже покрыты **MetalLB + ingress-nginx**.

### «Sticky sessions для SSE — почему выключены?»

* **SSE = одно TCP-соединение** = автоматически попадает на один pod до закрытия. Балансировка нужна только при установлении соединения.
* **Reconnect после обрыва** — клиент шлёт новый запрос; в нашем MVP он начинает inference заново (состояние gRPC к GPU потеряно, resume по `Last-Event-ID` не реализуем).
* Sticky session тут **не помогла бы**: даже если попасть на тот же pod, gRPC уже закрыт.

Если когда-то понадобится resume — включается одной аннотацией `nginx.ingress.kubernetes.io/affinity: cookie`.

### «Почему ingress-nginx, а не Nginx Plus или Envoy?»

* **Nginx Plus** даёт `least_time`, JWT validation, live-activity API — это ~$2.5k/нода/год × 12 нод = $30k/год. Для MVP не оправдано: open-source `ewma` для SSE достаточно близок к `least_time`.
* **Envoy / Istio** — ещё один компонент в hot-path, отдельный control plane, на защите потребует объяснения, чего он добавляет сверх ingress-nginx. У нас нет use-case-ов сервис-меша (mTLS между сервисами, retries-policies, circuit breakers) — все они либо обеспечены k8s, либо решены client-side.
* **HAProxy** — не интегрирован с k8s ingress-controller-стандартом так же зрело, как ingress-nginx; есть `haproxy-ingress`, но community меньше.

### «Почему MetalLB, а не cloud LB (AWS NLB / GCP Network LB)?»

* В §3.4 мы выбрали **общий Anycast IP** через BGP, а не region-local IP. Cloud LB обычно работает на region-local IP — он не годится для Anycast-схемы из коробки.
* MetalLB BGP-mode даёт ровно то, что нужно: BIRD2/FRRouting на edge-нодах анонсирует наш `/24` префикс с k8s-нод. Тот же стек, что описан в §3.5.
* Vendor-lock-in минимизирован: один и тот же стек работает на bare-metal, AWS, GCP.

### «Что если все ingress-nginx pods в ДЦ упадут одновременно?»

* Liveness probe (1 сек) → k8s рестартует pod-ы.
* Если массовый отказ (например, баг в новой версии ingress-nginx после rolling update) — readiness probe обнаружит проблему, MetalLB снимет BGP-анонс с edge-нод, **Anycast уведёт трафик в соседний ДЦ за 30–90 сек** (§3.5).
* Контр-мера в deploy-time: rolling update с `maxUnavailable: 25%` + canary через namespace.

### «А зачем 2N @ 50%, а не N+1, как в эталонной работе?»

Считаем для EU — самого нагруженного региона (3 реплики min, 6 в 2N).

| Сценарий | 2N (6 реплик) | N+1 (4 реплики) |
|----------|---------------|------------------|
| Штатно (нет отказов) | 17 861 CPS / 6 = 2 977 / 6 600 = **45%** | 17 861 / 4 = 4 465 / 6 600 = **68%** |
| Отказ 1 реплики | 17 861 / 5 = 3 572 / 6 600 = **54%** | 17 861 / 3 = 5 954 / 6 600 = **90%** |
| Отказ 50% парка | 17 861 / 3 = 5 954 / 6 600 = **90%** | 17 861 / 2 = 8 931 / 6 600 = **135%** ⚠️ |

* **N+1** при двух одновременных отказах (k8s rolling update + одна нода с HW-проблемой) уходит за 100% → 503-ки клиентам.
* **2N @ 50%** держит 50% отказа парка с 90% headroom — ровно то, что декларируем в §9 (надёжность).
* Капекс 2N выше на 50% (6 vs 4 реплики на 32 vCPU), но edge-нода стоит копейки на фоне GPU-бюджета.

Эталонная работа берёт N+1 (`Y+1 балансировщиков`), потому что у них профиль другой: 64 ноды в одном api-домене → отказ 1 из 64 = 1.5%, не критично. У нас 4–6 на регион → отказ 1 из 4 = 25%, надо бронировать с запасом.

---

## §5–§6 Базы данных

ENUM vs справочник — решается по правилу «есть ли у значения бизнес-атрибуты»:

| Поле | Решение | Почему |
| --- | --- | --- |
| `users.tier_id`, `messages.model_id`, `usage_records.model_id` | Справочник (PG) | Есть атрибуты (цены, лимиты, max_context); версионируем при изменениях |
| `messages.role`, `attachments.status`, `usage_records.channel`, `messages.finish_reason`, `audit_log.event_type` | ENUM / VARCHAR / `LowCardinality` | Нет бизнес-атрибутов; в Scylla JOIN недоступен, в CH `LowCardinality` — это уже «словарь под капотом» |

### Ключевые архитектурные решения

**Ключевые архитектурные решения:**

* **Справочники с бизнес-атрибутами.** `subscription_tiers` и `models` — single source of truth для тарифов (`users.tier_id` FK на UUID версии тарифа) и моделей (`messages.model_id` / `usage_records.model_id` FK на UUID версии модели). Цены per-1k токенов — индустриальный стандарт (OpenAI / Anthropic API): per-token (`$0.000005`) даёт нечитаемые числа в DECIMAL, per-million — норм, но 1k привычнее.
* **`model_code` живёт на `messages` (не на `conversations`).** Модель может меняться внутри одного диалога (юзер переключился с `gpt-4o-mini` на `gpt-4o` для сложного вопроса). Поле заполняется только для `role=assistant` (NULL для user). Это даёт честный audit «каким именно ответом был дан этот ответ» и убирает фиктивный «default model» на conversation.
* **`channel` (web/api) на `usage_records` оставлен.** Это **источник вызова**, не derived из `tier_code`: plus-юзер может вызывать через personal API. Нужен для (1) маржинальности по revenue streams, (2) quota policy (Web token-quota vs API balance), (3) abuse-детекции, (4) disputes по каналу.
* **`billing_accounts` хранит только деньги.** `tier_id` живёт на `users` — биллинг знает про `balance` / `reserved`, а тариф — это свойство аккаунта.
* **`rate_limits` хранит только счётчики.** Web-лимиты по модели лежат в `web_tier_quotas`, технические API-лимиты — системные константы; горячий путь читает Redis-кеш `quota:{tier_id}:{model_id}` — UPDATE миллионов счётчиков при ребрендинге линейки не нужен.
* **Трассировка request → response:** `messages.parent_message_id` (self-FK) — явная связь «assistant отвечает на user», закрывает regenerate / branching / audit. `messages.request_id` / `usage_records.request_id` / `attachments.request_id` — общий correlation ID одного inference-цикла, совпадает с `X-Request-ID` HTTP-ответа и `trace_id` в Tempo.
* **Sessions упрощены.** Только `token_hash` (PK Redis-ключа `sess:{token_hash}`), `user_id`, `created_at`, `expires_at`. IP / User-Agent / device fingerprint **не хранятся** — для security-аудита есть `audit_log` с событиями `login_success`. Это убрало 280 B per row × 1 млрд = ~280 ГБ Redis RAM.
* **Attachments упрощены.** Метаданные содержат только то, что нужно в hot-path: ссылку в S3, mime_type для UI и `status` (uploading → ready → deleted). `size_bytes` и `sha256` (под дедупликацию) — отложены до when-needed; первое достанется через S3 HeadObject, второе требует content-addressable storage (отдельная фича).
* **`api_keys.last_used_at` убран.** Иначе UPDATE на каждом API-запросе (8 680 RPS). Если понадобится — derived: `SELECT MAX(created_at) FROM usage_records WHERE user_id=? AND channel='api'`.
* **ENUM оставлены** там, где у значения нет бизнес-атрибутов: `messages.role`, `attachments.status`, `usage_records.channel`, `audit_log.event_type`. Обоснование «ENUM vs справочник» — §6.1.

### «Почему 5 СУБД, а не одна?»

Разные паттерны нагрузки несовместимы в одном движке:

- **OLTP-реляционные** (users, billing, conversations, api_keys, attachments_metadata, audit_log) — PostgreSQL: ACID, JOIN, индексы, SERIALIZABLE для billing.
- **OLTP key-value in-memory** (auth ~169k peak RPS, rate_limits ~170k peak ops) — Redis: latency < 1 мс на каждый HTTP-запрос. PG бы добавил +5 мс ко всему.
- **OLTP wide-column, write-heavy** (messages 58k RPS write, 4.5 ПБ/год) — ScyllaDB: PB-объём + write-throughput. PG/Citus упрётся в дисковый I/O.
- **OLAP колоночный** (usage_records 250 ТБ/год, агрегации SUM tokens/cost) — ClickHouse: LZ4 ×10 сжатие, MergeTree, PARTITION BY toYYYYMM. PG бы делал агрегации минутами.
- **Object storage** (attachments blob, 126 ПБ/год) — Ceph RGW: self-hosted S3-совместимый, дешевле managed S3 в десятки раз на нашем объёме, согласован с on-prem архитектурой.

### «Почему `subscription_tiers` и `models` — версионируемые таблицы с UUID PK?»

Если PK = `tier_code` (`'plus'`) или `model_code` и мы UPDATE-им цену, то исторические расчёты начинают через JOIN показывать новую цену — **исторический revenue ломается на ровном месте**. Решение: PK = UUID, при изменении тарифа/модели INSERT новой строки + UPDATE старой `effective_to = NOW()`. `usage_records.cost` фиксирует уже рассчитанную сумму, а `usage_records.model_id` ссылается на историческую версию модели.

Альтернативы:
- **SCD type 2** в OLAP — то же самое, но на стороне CH; мы оставили SoT в PG и реплицируем версии справочника, потому что цены меняет продакт, а не аналитика.
- **Хранить snapshot цены в каждом `usage_records.cost`** (мы и так это делаем — `cost` денормализован) — но `model_id` всё равно нужен для разбивки по моделям; без версионности при rename модели сломается фильтр по моделям.

### «Почему MVP без multi-currency?»

Multi-currency требует:
1. Сервиса FX-rates (живые курсы, fallback при сбое провайдера).
2. Conversion-time pricing — фиксируем курс в момент списания, не в момент инвойса.
3. Audit на rate-shifts — комплаенс требует знать, какой курс применён к транзакции.
4. UI/инвойсы под локали и форматы валют.

Это самостоятельный модуль уровня billing-сервиса. В MVP — только USD; multi-currency — отдельный roadmap-эпик после launch.

### «Почему Ceph RGW, а не managed S3?»

На 126 ПБ/год накопления:
- **Стоимость хранения** managed S3 (Standard) ≈ $0.023/ГБ/мес = 126 ПБ × 12 мес × $23 000/ПБ ≈ $35M/год только на storage. Cold-tier (Glacier Instant) дешевле в ~5 раз, но на свежих данных не помогает.
- **Egress** managed S3 на multimodal pull в GPU pool — внутри одного региона бесплатно, но cross-region — $0.02/ГБ. У нас 64.81 Гбит/с пик → ~210 ТБ/час egress, что бьёт в десятки $M/год.
- **Self-hosted Ceph RGW** на наших OSD-нодах (HDD + NVMe metadata) с EC 8+3 даёт сопоставимую durability при ×3–×5 экономии CAPEX за горизонт 3 лет. Erasure Coding 8+3 переживает потерю 3 дисков из 11 на объект; при наших ~1 200 OSD-нодах вероятность одновременного отказа 4+ дисков на одном PG-объекте — ничтожна (CRUSH алгоритм размазывает chunks по разным failure domains).
- **MinIO** отклонён: основной репозиторий `minio/minio` в начале 2026 года переведён в read-only режим — это операционный риск для прод-системы.

«А почему не SeaweedFS / RustFS / Garage?» — Garage и RustFS подходят для small/medium на единицы ПБ; Ceph — индустриальный стандарт для десятков ПБ и выше, с зрелым multisite-replication и интеграцией в k8s через Rook.

### «Почему БД размещены так — что это даёт?»

| Решение | Что даёт | Цена / альтернатива |
|---------|----------|---------------------|
| **PostgreSQL только в US East+West** (users / conversations / api_keys / attachments_metadata) | Один primary, нет conflict resolution, простой MVP | EU/APAC `GET /v1/conversations` имеет +80–150 мс RTT, что на фоне inference TTFT 250–400 мс терпимо. Альтернатива — async replicas в EU/APAC; добавили бы сложности lag-handling без UX-выигрыша |
| **ScyllaDB глобально (RF=3 в каждом ДЦ, total RF=12)** | Локальное чтение messages в EU/APAC спасает UX (открытие диалога не делает round-trip в US) | Storage cost высокий (~37 ПБ effective/year), но LOCAL_QUORUM=2 в каждом ДЦ переживает отказ одной локальной ноды без cross-DC fallback |
| **billing_accounts primary в US East, DR в US West** | SERIALIZABLE-транзакции без distributed coordination — нельзя списать одни деньги дважды | RTT в US East из EU/APAC при reserve/finalize — заложен в SLA. Альтернатива (per-region billing с conflict resolution) — отдельный продукт сложности enterprise |
| **ClickHouse только в US East** | Async insert `usage_records` от Chat/API Service из всех регионов; экономим на per-region инфре | Latency usage-инсертов терпит +150 мс: деньги уже синхронно зарезервированы/списаны в PostgreSQL billing |
| **Redis per-region** (sessions / rate_limits) | Auth-check на каждый HTTP-запрос (~169k peak RPS) локальный, без +150 мс из EU/APAC в US | Sessions, как и rate_limits, не реплицируется cross-region — пользователь привязан к региону Anycast (стабильно при нормальной маршрутизации, см. §3) |
| **Ceph region-buckets во всех ДЦ** | Клиент загружает в ближайший RGW endpoint; GPU при multimodal-запросе может тянуть объект через backbone | Multisite-репликация асинхронная; US West — основной DR-поток, EU/APAC копии нужны для локального чтения/регуляторики |

### «Почему именно эти СУБД, а не альтернативы?»

| Таблица | Альтернатива | Почему отвергли |
|---------|--------------|-----------------|
| `users` etc. (PG) | MySQL | Сопоставим, но у PG лучше партиционирование (`PARTITION BY RANGE`), JSONB, partial-индексы |
| `messages` (Scylla) | Cassandra | Та же модель, но Cassandra на JVM медленнее (Java GC паузы); Scylla даёт ×3 throughput на той же железке |
| `messages` (Scylla) | PostgreSQL/Citus | Упрётся в дисковый I/O при 4.5 ПБ; нет write-amplification оптимизаций как у LSM-tree |
| `sessions` (Redis) | PostgreSQL | Выдержит RPS, но добавит ~5 мс к каждому запросу; PG не даёт нативного TTL |
| `usage_records` (CH) | Druid | Сложнее эксплуатировать при таком же выигрыше; ClickHouse compatible с pgwire-style клиентами |

### «Почему не Kafka?»

Один consumer (ClickHouse), транзакционный billing требует sync-связи. Kafka добавила бы:
- Дополнительную точку отказа между Chat Service и ClickHouse;
- Лаг доставки usage-событий, что усложнит anti-fraud (текущая задержка <1 сек async insert vs до минут с Kafka batching);
- Операционную нагрузку (broker cluster + ZooKeeper / KRaft).

Kafka появится, если возникнут **независимые consumer-ы** (anti-fraud / DWH / нотификации) — тогда fanout оправдает себя. Сейчас — over-engineering.

### «Почему 2N @ 50% для PG/Scylla, а не N+1?»

Для PG user-кластера в README заложены 64 шарда: 32 primary + 32 sync + 32 async + coordinators = 100 нод. Формально N+1 хватает для одиночной ноды, но:
- При **отказе целой зоны отказа** теряем сразу большой кусок локальных реплик. N+1 не выдержит, нужен запас 2N на уровне capacity.
- Sync replica в другой зоне отказа US East нужна для локального RPO=0: primary не подтверждает commit, пока WAL не дошёл до sync-replica.
- Async replica в US West — для cross-DC DR при полном отказе US East; её RPO равен фактическому replication lag.

ScyllaDB RF=3 в US — это не «1+2», а полноценный кворум: запись в LOCAL_QUORUM (2 из 3) переживает падение 1 ноды без задержек.

### Запрещённые антипаттерны на защите

| «Спросят» | «Готовый ответ» |
|-----------|-----------------|
| Зачем `audit_log` в PG, а не в ClickHouse? | PG нужны индексы по `(user_id, created_at)` для security-расследований одной транзакцией; CH хорош для агрегаций, не для точечных lookup |
| Зачем `messages` в Scylla, а не в S3 как JSON? | S3 не даёт consistency для read-after-write по частым запросам; Scylla — durable + быстрый range-scan по `conv_id` |
| Зачем `sessions` в Redis, если можно в JWT (stateless)? | JWT нельзя ревокнуть на logout без external blacklist, который всё равно живёт в Redis. Сессии в Redis — простее и полностью контролируемо |
| Зачем `rate_limits` в Redis, а не в PG? | ~250k peak QPS atomic check-and-incr через Lua — PG бы упёрся в lock contention на hot keys |

---

## §6 Технические механизмы: как это работает

Подробный разбор каждой технологии из §6 — на случай, если препод задаст «расскажи как X работает внутри». Идём по технологиям.

### PostgreSQL + Citus

**Что такое Citus.** Citus — расширение PostgreSQL, превращающее один PG-инстанс в распределённый кластер: один coordinator + N worker-нод. Распределяет таблицы по worker-ам по значению distribution column.

**Distribution column.** Столбец, по которому Citus считает `hash(col) % N_shards` и кладёт строку на соответствующий шард. У нас — `user_id`. Каждый запрос с `WHERE user_id = ?` направляется coordinator-ом сразу на нужный шард (single-shard query, ~миллисекунды). Запрос без `user_id` — multi-shard, scatter-gather на все шарды + сборка ответа в координаторе (медленно, антипаттерн в hot-path).

**Co-location.** Две distributed-таблицы с одним distribution column и одинаковым числом шардов размещают строки с одинаковым ключом на одном и том же worker-узле. JOIN между ними выполняется локально на каждом worker'е без межсетевого обмена. У нас `users` / `conversations` / `api_keys` / `attachments` все co-located по `user_id` — `JOIN users ON conversations.user_id = users.user_id` обслуживается локально.

Объявление: `SELECT create_distributed_table('conversations', 'user_id', colocate_with => 'users');`

**Reference tables.** Маленькая таблица копируется **целиком** на каждый worker-узел. `JOIN distributed_table → reference_table` выполняется локально, без cross-shard hop. У нас — `subscription_tiers`, `models`, `web_tier_quotas` (десятки-сотни строк, обновляются редко).

Объявление: `SELECT create_reference_table('subscription_tiers');`

**Logical replication.** Механизм PG, при котором изменения на primary публикуются в WAL-формате и применяются на subscriber-ах через SQL (а не на уровне бинарных страниц как physical replication). Гибче: можно реплицировать отдельные таблицы. Citus использует это для раздачи reference tables на worker-ноды. Лаг < 1 сек на здоровом кластере.

**Partial index** (`CREATE INDEX ... WHERE condition`). B-tree, который индексирует только строки, удовлетворяющие условию. Меньше места, быстрее обновляется. У нас:
- `(tier_code) WHERE effective_to IS NULL` — индексируем только активные версии тарифов, поиск «текущий тариф по коду» по тонкому индексу из ~5 строк вместо ~50.
- `(balance) WHERE balance < threshold` — для проактивных уведомлений о низком балансе.

**PgBouncer transaction mode.** Приложение коннектится в PgBouncer, тот выдаёт реальное PG-соединение **на одну транзакцию** и сразу возвращает в пул. Это самый плотный режим:
- Application connection (cheap, accept на TCP listen): десятки тысяч.
- PG backend connection (expensive, fork-процесс): сотни.

Ограничения transaction mode: нельзя использовать prepared statements между транзакциями, нельзя `SET LOCAL` за пределами транзакции, нельзя advisory locks с session scope. Для нас это ок — все запросы укладываются в одну транзакцию.

Альтернативы: `session mode` (соединение возвращается в пул при `DISCONNECT`, безопаснее для legacy-приложений, но плотность падает в 10×); `statement mode` (возврат после каждого statement, ломает транзакции).

**WAL (Write-Ahead Log) и PITR.**
- На каждое изменение PG сначала пишет в WAL (last-write-wins log), потом в shared_buffers, потом грязные страницы записываются на диск checkpoint-ом.
- `pg_basebackup` делает консистентный снимок кластера (физическая копия data directory + текущий WAL position).
- `WAL archiving` непрерывно копирует WAL-сегменты в Ceph RGW.
- **PITR (Point-In-Time Recovery)** = `pg_basebackup` + накат WAL-сегментов до нужного `recovery_target_time`. Можно восстановить состояние на любую секунду в пределах хранения WAL.

**Sync vs async replica.**
- `synchronous_commit = on` + `synchronous_standby_names = 'replica1'` — primary не подтверждает COMMIT клиенту, пока WAL не дошёл до sync-replica. RPO = 0 (нельзя потерять подтверждённую транзакцию), но latency = RTT до sync-replica. Для billing — обязательно.
- Async replica — primary подтверждает COMMIT сразу, WAL стримится в фоне. RPO ≤ lag (обычно секунды). Для user data и cross-DC DR.

**Replication slots.** Server-side bookmarks, гарантирующие что WAL не будет удалён, пока конкретный replica/subscriber не подтвердил приём. Без slot replica может отстать настолько, что primary вычистит нужные WAL — replica сломается. С slot — primary держит WAL до lag (риск: переполнение диска при долгой потере replica).

### ScyllaDB

**Wide-column model.** Каждая строка таблицы привязана к partition key. Partition key определяет, на какой ноде хранится строка. Clustering keys задают порядок строк внутри партиции (sorted run). Итого: partition = логический контейнер (могут быть млн строк), который физически лежит на одной ноде.

У `messages`:
- Partition key = `conv_id` — все сообщения одного диалога на одной ноде.
- Clustering keys = `(created_at DESC, message_id)` — внутри партиции отсортированы по времени, новые сверху.
- Запрос `SELECT * FROM messages WHERE conv_id=? LIMIT 50` — одна partition-операция, читается один SSTable-segment с начала.

**Murmur3 hash.** Non-cryptographic hash, стандарт Cassandra/Scylla для partition routing. Быстрая (десятки нс на 16-байт ключ), детерминированная, хорошая равномерность. По хешу `conv_id` Scylla определяет, на каких нодах лежит партиция (token ring + RF).

**LSM-tree write path.**
1. INSERT → memtable (in-memory write buffer, RB-tree или skiplist).
2. Параллельно → commit log (durability, append-only WAL на диске).
3. Memtable заполняется до порога → flush → новый SSTable (Sorted String Table — immutable отсортированный файл на диске).
4. Со временем SSTables копятся → compaction объединяет их.

Преимущества vs PG B-tree:
- Запись = sequential write в commit log + in-memory operation в memtable. Никаких random disk seeks на каждый INSERT.
- Нет vacuum/autovacuum — старые версии живут до compaction, удаляются партиями.
- Нет WAL-checkpoint pauses (PG checkpoint синхронизирует все dirty pages → impact на latency).

Цена: read amplification (запрос может потребовать прочесть несколько SSTables). Bloom-фильтр на каждом SSTable отсеивает «точно нет ключа», тогда читается только релевантный SSTable.

**TWCS (Time-Window Compaction Strategy).** Compaction-стратегия, оптимизированная под time-series: SSTables группируются по временным окнам (час/день). При истечении TTL Scylla удаляет целые SSTable атомарно, не делая read+rewrite. Альтернативы:
- `STCS` (Size-Tiered) — default, обычная стратегия, плохо работает с time-series (объединяет старые и свежие данные).
- `LCS` (Leveled) — для read-heavy, разбивает на уровни, write-amplification высокая.

**NetworkTopologyStrategy и RF per DC.** Multi-DC keyspace через `NetworkTopologyStrategy`: указываешь RF отдельно для каждого DC. У нас `us_east: 3, us_west: 3, eu_central: 3, apac: 3`. Это даёт:
- В каждом DC — 3 копии.
- `LOCAL_QUORUM=2` переживает отказ одной локальной ноды без cross-DC fallback.
- Total RF=12, чтения и записи с `LOCAL_QUORUM` остаются в локальном DC.

**LOCAL_QUORUM** = ⌈RF_local/2⌉+1 нод из локального DC должны подтвердить операцию. При RF=3 это 2 локальные ноды. Не ждёт cross-DC round-trip.

Альтернатива — `QUORUM` (cross-DC кворум, требует ⌈total_RF/2⌉+1 нод из всего кластера) — медленнее, но даёт strong consistency cross-DC. Нам не нужен.

**Hinted handoff.** Если нода-получатель upd временно недоступна, координатор сохраняет «hint» (буфер незаписанных мутаций) и доставляет их, когда нода поднимется. Гарантирует eventual consistency без полного repair.

**Read repair.** При чтении из нескольких реплик Scylla сравнивает версии и в фоне синхронизирует расхождения (если, например, hint потерялся). Контролируется параметром `read_repair_chance`.

**Materialized View vs Secondary Index.**
- **MV** = вторая копия данных с другим partition key, поддерживается Scylla автоматически на каждом write. Read по MV = partition-операция, O(1). Цена: ×1.05 storage + 1 дополнительный write per insert.
- **Secondary Index** = индекс по non-partition-key столбцу. Read — scatter-gather по всем нодам (потому что значение может быть в любой партиции). Latency нестабильна на больших кластерах.
- У нас `messages_by_request` = MV с `request_id` как PK для audit-lookup.

**Token-aware routing в драйвере.** Драйвер (scylla-driver, gocql) знает топологию кластера (token range → node) и считает Murmur3 хеш от partition key **на клиенте**. Запрос идёт сразу на нужную ноду, без посредника-coordinator-а. Экономит 1 hop vs Cassandra default routing.

### Redis Cluster

**16384 hash slots.** Redis Cluster делит ключевое пространство на 16 384 фиксированных слота. Каждый ключ маппится в слот через `CRC16(key) % 16384`. Каждый master-node владеет диапазоном слотов; replica хранит копию master-слотов.

**Hash-tag `{}`.** Если в ключе есть подстрока в `{...}`, CRC16 считается **только** от неё. Используется для **co-location** ключей на одном слоте/ноде, чтобы Lua-скрипт мог работать с ними атомарно. У нас:
- `rl:web:rps:{user_id}`, `rl:web:msg_3h:{user_id}:{model_id}`, `rl:web:msg_day:{user_id}:{model_id}` — все попадают в один слот за счёт `{user_id}`, Lua-скрипт `check_and_incr` работает с ними транзакционно.

Без hash-tag три ключа разойдутся по разным slot → разным мастер-нодам → Lua-скрипт нельзя выполнить атомарно (cluster mode требует ВСЕ ключи скрипта в одном slot).

**MOVED / ASK redirection.** Если клиент шлёт команду не на ту ноду (например, после resharding), мастер отвечает `MOVED <slot> <new_node>` или `ASK <slot> <node>` (для in-flight миграции). Клиент обновляет cluster map и повторяет запрос. Это автоматический механизм для смены ownership без downtime.

**Lua-скрипт атомарность.** `EVAL`/`EVALSHA` исполняет Lua-скрипт в single-threaded event-loop Redis. Внутри скрипта — set commands видят промежуточные изменения, никто другой не вмешивается. Это даёт **check-and-incr за один RTT**:
```lua
local current = redis.call("GET", KEYS[1])
if current and tonumber(current) >= tonumber(ARGV[1]) then
    return 0  -- DENY
end
redis.call("INCR", KEYS[1])
redis.call("EXPIRE", KEYS[1], ARGV[2])
return 1  -- ALLOW
```

Без Lua это были бы 2 RTT (`GET` + `INCR`), плюс race condition между ними.

**TTL и passive/active expiration.** Каждый ключ может иметь TTL. Redis удаляет expired ключи двумя путями:
- Passive: при чтении/записи проверяет TTL → если истёк, удаляет.
- Active: фоновый scan семплирует 20 случайных ключей с TTL, удаляет expired; если >25% удалены — повторяет немедленно.

Гарантирует, что expired ключи не накапливаются в RAM.

**RDB vs AOF.**
- **RDB** (Redis Database Backup) — point-in-time snapshot всей базы, fork-процесс копирует memory state в файл на диске. Компактно, быстро восстанавливается. Минус: между snapshot-ами данные не durable (RPO = период между RDB).
- **AOF** (Append-Only File) — все write-команды дописываются в файл. `fsync` режимы: `always` (sync на каждую операцию, RPO=0, медленно), `everysec` (sync каждую секунду, RPO≤1s, разумно), `no` (RPO = когда ОС сбросит). Файл периодически переписывается (rewriting) для компактности.

У нас `sessions` бэкапятся `RDB hourly + AOF`, но Redis не source of truth: production-восстановление идёт через failover, а после полного restore сессии можно пересоздать логином. Поэтому RPO Redis не является бизнес-гарантией.

**Pub/sub для cache invalidation.** Канал `cache:invalidate`, на который подписаны все экземпляры приложения. При изменении `subscription_tiers` в PG приложение публикует `PUBLISH cache:invalidate tier:<tier_id>` — все подписчики мгновенно дропают локальный кеш ключа. Lag < 1 сек по сети.

**Cluster vs Sentinel.** Sentinel — мониторинг master/replica + failover; всё всё ещё single-master, нет шардирования. Cluster — full sharding на 16384 slots + автоматическая ребалансировка. На наших QPS Sentinel упёрся бы в один CPU master-ноды.

### ClickHouse

**MergeTree семейство.** Базовый движок CH. Данные хранятся в part-ах (immutable sorted files на диске). При INSERT создаётся новый part. Background-процесс периодически объединяет малые parts в большие (merge). Запросы читают все active parts параллельно, сливая результаты по primary key.

**Granule.** CH разбивает part на granules — блоки фиксированного размера (по умолчанию 8192 строки, настраивается `index_granularity`). На границах granule пишется primary key (sparse index), а внутри granule поиск — full scan. Trade-off: меньше granule = быстрее точечные lookup, больше места под индекс. У нас 8192 — стандарт.

**ReplicatedMergeTree + ZooKeeper.** Реплицирует данные между нодами через координацию ZooKeeper (или ClickHouse Keeper — встроенная замена ZK на Raft). При INSERT нода пишет данные локально + регистрирует event в ZK; реплики тянут event и копируют part. Replication — eventually consistent (но обычно секунды).

**Distributed table.** Виртуальная таблица-обёртка, которая знает топологию shards. INSERT в distributed table → CH сам разбрасывает строки по shards по shard_key. SELECT → broadcast на все shards + сборка ответа.

**Partitioning by `toYYYYMM(created_at)`.** Партиция = логическая группа parts с одинаковым выражением. Запросы с фильтром по дате (`WHERE created_at >= '2026-01-01'`) читают только нужные партиции (partition pruning). Drop старых партиций — мгновенный, без перезаписи остальных данных. У нас drop after 1 year.

**LowCardinality(String).** Встроенное dictionary encoding. CH хранит словарь уникальных значений + per-row int8/int16-код вместо строки. Прозрачно для SQL: пишешь `VARCHAR`, под капотом — int. Экономит до 10× места на столбцах с малым числом значений (`channel`: 'web'/'api' = 2 значения, 1 байт вместо 3-4).

**LZ4 compression.** Default codec у MergeTree. Скорость декомпрессии ~3-4 GB/s/core, степень сжатия ~3-5× на текстовых/JSON-данных, ~10× на columnar data с LowCardinality. У нас 243 ТБ/год → ~25 ТБ on disk.

**Bloom filter skipping index.** Не настоящий индекс (нет O(log N) lookup), а statistic, помогающий пропускать granules. Для каждого granule CH строит bitmap, по которому быстро отвечает «точно нет значения». При фильтре `WHERE request_id = ?` CH:
1. Проходит по skipping index всех granules.
2. Granules, где bloom filter говорит «нет» — пропускает.
3. Granules, где «может быть» — читает с диска, проверяет.

Экономит I/O в десятки раз для точечных lookup по неупорядоченным колонкам.

**External dictionary.** CH-механизм для подгрузки lookup-таблицы из внешнего источника (PG, HTTP, файл) в RAM с периодическим обновлением. Используется как `dictGet('models', 'price', model_id)` в запросах. У нас `models` подгружается из PG с TTL 5 мин — позволяет считать revenue per model без cross-DB JOIN в hot-path.

**Async insert.** Альтернатива классическому INSERT (который ждёт fsync на каждый запрос). Async insert буферизует входящие INSERT-ы в RAM на стороне CH-ноды, flush-ит батчем (по времени или размеру). Используем только для `usage_records`/аналитики: денежный `reserve/finalize` уже синхронно прошёл через PostgreSQL `billing_accounts`, поэтому ClickHouse не является source of truth для списания денег.

### Ceph Object Gateway (RGW)

**Архитектура Ceph.**
- **OSD (Object Storage Daemon)** — процесс на одном диске. Отвечает за хранение объектов, репликацию, recovery. У нас 1200 OSD = 1200 процессов на ~50 серверах (24 диска × 50 = 1200).
- **MON (Monitor)** — Paxos-quorum (3-5 нод), хранит cluster map (членство OSD, CRUSH map, версии). Все клиенты сначала контактируют MON для получения map.
- **MGR (Manager)** — координация, метрики, dashboards. Stateless.
- **RGW (RADOS Gateway)** — обёртка над RADOS (низкоуровневый объектный store Ceph), реализует S3-совместимый HTTP API.

**CRUSH (Controlled Replication Under Scalable Hashing).** Алгоритм, по которому Ceph детерминированно вычисляет, на каких OSD лежит каждый объект. Не нужен центральный metadata-сервис (как в HDFS NameNode) — каждый клиент сам считает CRUSH по object key + cluster map.

CRUSH map — дерево узлов с весами: `root → datacenter → rack → host → osd`. Failure domain задаётся уровнем в дереве: для нас `failure_domain = host` означает «не клади две копии на один host». Это даёт переживание отказа целого сервера.

Преимущество vs hash-based (Cassandra): CRUSH учитывает иерархию (rack, DC) и веса (новые ноды получают больше данных).

**Placement Group (PG).** Логический контейнер для группы объектов. Объект мапится в PG (`hash(object_id) % num_pgs`), PG мапится в OSD-set через CRUSH. PG-уровень нужен, чтобы recovery работало на уровне групп (а не каждого объекта отдельно). Рекомендация: 100 PG на OSD.

**Erasure Coding 8+3 (Reed-Solomon).** Объект режется на 8 data chunks (data, 1/8 объекта каждый) + 3 parity chunks (вычислены по схеме Reed-Solomon как линейные комбинации data chunks в поле Галуа). Парности достаточно, чтобы восстановить любую из 11 = 8+3 потерянных chunks при условии целостности 8 любых.

Сравнение с triple replication:
- Storage overhead: EC 8+3 = ×1.375, replication = ×3.
- Read latency: EC требует чтения 8 chunks (можно параллельно с 8 OSD), replication — одного chunk.
- Write latency: EC требует записи на 11 OSD + вычисления parity, replication — 3 OSD без CPU.
- Recovery: EC требует чтения 8 chunks для восстановления одного, replication — копирование одного.

Для cold-tier (read-rarely, capacity-dominated) EC выигрывает кратно по стоимости. Для hot-path (low latency required) — replication лучше. У нас EC 8+3 — для всех blob, hot-tier даёт NVMe-speed на чтение data chunks параллельно.

**Multisite replication.** Ceph Object Gateway zone group = несколько zone (региональных кластеров), которые синхронизируются. Метаданные (bucket, IAM) реплицируются sync (one master, others slave). Данные — async (eventual). У нас US East ↔ US West (sync metadata, async data), US East → EU/APAC (async для DR).

**Presigned URL.** Метод S3 API: server (имеющий credentials) подписывает HTTP-URL с правами на PUT/GET конкретного объекта и временем жизни (например, 15 минут). Клиент может использовать URL напрямую без AWS credentials. У нас — для прямой загрузки/выгрузки клиентом, минуя backend (offload TLS termination + bandwidth).

**Bucket vs Object key.** Bucket — namespace + policy boundary (ACL, lifecycle, encryption). Object key — UTF-8 строка, до 1024 байт. Псевдо-иерархия делается через `/` в key (`{user_id}/{attachment_id}`), но физически в Ceph нет директорий — key хеширует в PG напрямую.

**Object Versioning.** Bucket-level флаг. При включении каждый PUT (даже DELETE) создаёт новую версию объекта; старые версии хранятся до явного удаления. Защита от случайного перезаписи / удаления.

### Применение: что где работает

Для устной защиты — короткие связки «фича → механизм»:

| Фича РПЗ | Что под капотом |
|----------|-----------------|
| Co-location `users` + `conversations` в PG | Citus distribution column `user_id` + `colocate_with` + reference table для tier/model |
| `subscription_tiers` доступен на каждом шарде без cross-shard hop | Citus reference table + PG logical replication |
| Версионируемые цены / лимиты без сломанной аналитики | UUID PK + `effective_to TIMESTAMP NULL` + INSERT новой строки на изменение |
| Найти активную версию тарифа по коду | Partial B-tree index `(tier_code) WHERE effective_to IS NULL` |
| `WHERE conv_id=? LIMIT 50` за ≤10 мс на любом объёме | Scylla partition key `conv_id` (Murmur3) + clustering DESC + SSTable sequential read |
| Lookup сообщений по `request_id` для audit | Scylla Materialized View `messages_by_request` (не Secondary Index — он scatter-gather) |
| Локальное чтение messages в EU/APAC без round-trip в US | NetworkTopologyStrategy с RF≥1 в каждом DC + LOCAL_QUORUM |
| Атомарная проверка-и-инкремент 3 счётчиков для rate-limit | Redis Lua скрипт + hash-tag `{user_id}` для co-location ключей в одном hash slot |
| Кеш `tier_id` в session с моментальной инвалидацией | Redis pub/sub канал `cache:invalidate` |
| Restore PG на любую секунду | pg_basebackup + WAL archiving в Ceph + PITR |
| Нельзя списать одни деньги дважды | PG SERIALIZABLE + sync replica в другой зоне отказа US East → локальный RPO=0 |
| `cost` не «разъезжается» при ребрендинге прайса | Денормализованный snapshot `cost` в `usage_records` + UUID FK на исторический `model_id` |
| `SELECT SUM(cost) GROUP BY model` за миллисекунды | ClickHouse MergeTree + ORDER BY (user_id, created_at) + bloom_filter skipping index |
| Drop старых партиций мгновенно | ClickHouse PARTITION BY toYYYYMM (DROP PARTITION — без перезаписи) |
| Один объект переживает отказ 3 дисков | Ceph Erasure Coding 8+3 (Reed-Solomon) + CRUSH с failure_domain=host |
| Клиент льёт файл в Ceph, минуя backend | Presigned PUT URL (S3 API), 15 мин TTL |
| Origin-bucket рядом с GPU, DR-копия в US West | Ceph multisite replication (sync metadata, async data) US East ↔ US West |
