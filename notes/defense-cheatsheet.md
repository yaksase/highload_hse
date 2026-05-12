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

### Ключевые архитектурные решения

**Ключевые архитектурные решения:**

* **Справочники с бизнес-атрибутами.** `subscription_tiers` и `models` — single source of truth для тарифов (`tier_code` FK на `users`) и моделей (`model_code` FK на `messages` и `usage_records`). Цены per-1k токенов — индустриальный стандарт (OpenAI / Anthropic API): per-token (`$0.000005`) даёт нечитаемые числа в DECIMAL, per-million — норм, но 1k привычнее.
* **`model_code` живёт на `messages` (не на `conversations`).** Модель может меняться внутри одного диалога (юзер переключился с `gpt-4o-mini` на `gpt-4o` для сложного вопроса). Поле заполняется только для `role=assistant` (NULL для user). Это даёт честный audit «каким именно ответом был дан этот ответ» и убирает фиктивный «default model» на conversation.
* **`channel` (web/api) на `usage_records` оставлен.** Это **источник вызова**, не derived из `tier_code`: plus-юзер может вызывать через personal API. Нужен для (1) маржинальности по revenue streams, (2) quota policy (Web token-quota vs API balance), (3) abuse-детекции, (4) disputes по каналу.
* **`billing_accounts` хранит только деньги.** `tier_code` ушёл на `users` — биллинг знает про `balance` / `reserved`, а тариф — это свойство аккаунта.
* **`rate_limits` хранит только счётчики.** Лимиты (`rps_limit` / `tpm_limit`) — в справочнике `subscription_tiers`, читаются через Redis-кеш `tier:{tier_code}` — UPDATE миллионов строк при ребрендинге линейки не нужен.
* **Трассировка request → response:** `messages.parent_message_id` (self-FK) — явная связь «assistant отвечает на user», закрывает regenerate / branching / audit. `messages.request_id` / `usage_records.request_id` / `attachments.request_id` — общий correlation ID одного inference-цикла, совпадает с `X-Request-ID` HTTP-ответа и `trace_id` в Tempo.
* **Sessions упрощены.** Только `token_hash` (PK Redis-ключа `sess:{token_hash}`), `user_id`, `created_at`, `expires_at`. IP / User-Agent / device fingerprint **не хранятся** — для security-аудита есть `audit_log` с событиями `login_success`. Это убрало 280 B per row × 1 млрд = ~280 ГБ Redis RAM.
* **Attachments упрощены.** Метаданные содержат только то, что нужно в hot-path: ссылку в S3, mime_type для UI и `status` (uploading → ready → deleted). `size_bytes` и `sha256` (под дедупликацию) — отложены до when-needed; первое достанется через S3 HeadObject, второе требует content-addressable storage (отдельная фича).
* **`api_keys.last_used_at` убран.** Иначе UPDATE на каждом API-запросе (8 680 RPS). Если понадобится — derived: `SELECT MAX(created_at) FROM usage_records WHERE user_id=? AND channel='api'`.
* **ENUM оставлены** там, где у значения нет бизнес-атрибутов: `messages.role`, `attachments.status`, `usage_records.channel`, `audit_log.event_type`. Обоснование «ENUM vs справочник» — §6.1.

### «Почему 5 СУБД, а не одна?»

Разные паттерны нагрузки несовместимы в одном движке:

- **OLTP-реляционные** (users, billing, conversations, api_keys, attachments_metadata, audit_log) — PostgreSQL: ACID, JOIN, индексы, SERIALIZABLE для billing.
- **OLTP key-value in-memory** (sessions 95 700 RPS, rate_limits 173 200 RPS) — Redis: latency < 1 мс на каждый HTTP-запрос. PG бы добавил +5 мс ко всему.
- **OLTP wide-column, write-heavy** (messages 58k RPS write, 4.5 ПБ/год) — ScyllaDB: PB-объём + write-throughput. PG/Citus упрётся в дисковый I/O.
- **OLAP колоночный** (usage_records 250 ТБ/год, агрегации SUM tokens/cost) — ClickHouse: LZ4 ×10 сжатие, MergeTree, PARTITION BY toYYYYMM. PG бы делал агрегации минутами.
- **Object storage** (attachments blob, 126 ПБ/год) — Ceph RGW: self-hosted S3-совместимый, дешевле managed S3 в десятки раз на нашем объёме, согласован с on-prem архитектурой.

### «Почему `subscription_tiers` и `models` — версионируемые таблицы с UUID PK?»

Если PK = `tier_code` (`'plus'`) и мы UPDATE-им цену, то все исторические `usage_records`, ссылающиеся на этот код, начинают через JOIN показывать новую цену — **исторический revenue ломается на ровном месте**. Решение: PK = UUID, при изменении тарифа INSERT новой строки + UPDATE старой `effective_to = NOW()`. Старые `usage_records` ссылаются на старый `tier_id`, бизнес-аналитика по выручке стабильна.

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
| **ScyllaDB глобально (RF=3 в US, RF=1 в EU/APAC, total RF=5)** | Локальное чтение messages в EU/APAC спасает UX (открытие диалога не делает round-trip в US) | Storage cost ×1.7 (4.5 ПБ × 1.7 = 7.6 ПБ effective). RF=1 в EU/APAC — компромисс: при падении локальной ноды читаем из US (deg latency, но не downtime) |
| **billing_accounts только в US East** | SERIALIZABLE-транзакции без distributed coordination — нельзя списать одни деньги дважды | RTT в US East из EU/APAC при reserve/finalize — заложен в SLA. Альтернатива (per-region billing с conflict resolution) — отдельный продукт сложности enterprise |
| **ClickHouse только в US East** | Async insert от Chat Service из всех регионов; экономим на per-region инфре | Latency биллинг-инсертов терпит +150 мс (биллинг — не критичный путь) |
| **Redis per-region** (sessions / rate_limits) | Auth-check на каждый HTTP-запрос (95k RPS) локальный, без +150 мс из EU/APAC в US | Sessions, как и rate_limits, не реплицируется cross-region — пользователь привязан к региону Anycast (стабильно при нормальной маршрутизации, см. §3) |
| **S3 region-buckets в US East+West (origin) + EU/APAC (replicas)** | Origin-buckets рядом с GPU pool — multimodal pull без cross-region | EU/APAC bucket-ы держат replicated копии для disaster recovery; пишут в них только cross-region replication, прямые загрузки клиента — в ближайший US bucket |

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

Для PG users-кластера (64 шарда × 3 реплики = 192 ноды) формально N+1 хватает. Но:
- При **отказе целой AZ** теряем сразу 1/3 кластера (sync-replica + третья нода). N+1 не выдержит, нужно 2N.
- Sync replica в той же AZ нужна для RPO=0 (нельзя терять fresh writes billing-account-а).
- Async replica в US West — для cross-DC failover при outage AZ US East.

ScyllaDB RF=3 в US — это не «1+2», а полноценный кворум: запись в LOCAL_QUORUM (2 из 3) переживает падение 1 ноды без задержек.

### Запрещённые антипаттерны на защите

| «Спросят» | «Готовый ответ» |
|-----------|-----------------|
| Зачем `audit_log` в PG, а не в ClickHouse? | PG нужны индексы по `(user_id, created_at)` для security-расследований одной транзакцией; CH хорош для агрегаций, не для точечных lookup |
| Зачем `messages` в Scylla, а не в S3 как JSON? | S3 не даёт consistency для read-after-write по частым запросам; Scylla — durable + быстрый range-scan по `conv_id` |
| Зачем `sessions` в Redis, если можно в JWT (stateless)? | JWT нельзя ревокнуть на logout без external blacklist, который всё равно живёт в Redis. Сессии в Redis — простее и полностью контролируемо |
| Зачем `rate_limits` в Redis, а не в PG? | 173 200 RPS atomic check-and-incr через Lua — PG бы упёрся в lock contention на hot keys |
