# Drafts to be incorporated in §7 (Algorithms) and §10 (Project schema)

Этот файл — архив материалов, которые временно лежали в §3 (Глобальная балансировка), но по сути относятся к §7 «Алгоритмы» и §10 «Схема проекта». Не входят в финальный текст РПЗ — переносятся при заполнении §7/§10.

---

## (для §10) Балансировка inference-запросов и поток токенов

### Почему inference живёт отдельно от HTTP-слоя

* Запрос занимает GPU на 5–60 секунд — нельзя балансировать round-robin-ом
* GPU — ограниченный и дорогой ресурс, нужна приоритизация (Free / Plus / Pro / Max) и continuous batching
* Нужно точное знание, какой воркер какой запрос обслуживает (в том числе для отмены)

### Разделение control plane и data plane

Ключевое решение: **Inference Scheduler — это только control plane**. Он решает, какой GPU-воркер возьмёт запрос, но **через него не идут токены**. Токены стримятся напрямую от GPU-воркера в региональный Inference Service, минуя Scheduler.

```
   Client (Germany)
        │ 1. HTTP POST /v1/chat/completions  (SSE)
        ▼
   ┌────────────────┐
   │  Edge / Anycast│
   └───────┬────────┘
           │ 2. TLS terminate → BGP routes to nearest DC
           ▼
   ┌─────────────────────────┐
   │ EU Region               │
   │  Nginx (TLS, SSE)       │
   │  └─► API Gateway        │
   │      ├─► Auth Service (verify cookie/API token)
   │      ├─► Rate Limiter (Redis: check + incr, per-region)
   │      ├─► Billing Reservation (PG, US) — reserve funds (API only)
   │      └─► Chat Service
   │          └─► Regional Inference Service
   └─────────┬───────────────┘
             │ 3. control: AssignWorker(request)
             ▼
   ┌─────────────────────────┐
   │ US: Inference Scheduler │  ← control plane only
   │  • priority queue       │
   │  • batch / GPU alloc    │
   │  • возвращает endpoint  │
   └─────────┬───────────────┘
             │ 4. возвращает адрес воркера
             ▼
   ┌─────────────────────────┐
   │ EU: Regional Inference  │
   │ открывает               │
   │   rpc Generate(req)     │  ←── 5. server-streaming gRPC
   │   returns (stream Token)│       напрямую к назначенному
   └─────────┬───────────────┘       воркеру в US
             │ токены
             ▼
   ┌─────────────────────────┐
   │ US: GPU Worker (H100)   │
   │ стримит токены          │
   └─────────────────────────┘

   Путь токенов обратно:
     GPU Worker → gRPC stream → Regional Inference (EU)
       → SSE events → Nginx → Edge → Client
```

### Почему server-streaming, а не bidi gRPC

Пользовательский запрос — «один промпт → много токенов ответа». Это ровно semantics server-streaming RPC:

```proto
rpc Generate(GenerateRequest) returns (stream TokenEvent);
```

Bidi-стрим нам **не нужен**: в обычном чате клиент не дописывает ничего после отправки промпта. Bidi пригодился бы для speech-to-text (клиент льёт аудио параллельно с транскрипцией) или speculative decoding с draft-токенами от клиента — но в нашем MVP таких сценариев нет. Server-streaming проще (одна сторона закрывает стрим) и меньше расходует ресурсы на state-machine HTTP/2 потока.

### Backpressure и отмена

* **Backpressure**: если клиент медленно читает SSE, HTTP/2 flow control снижает окно на нём → Nginx перестаёт отдавать байты в Regional Inference → gRPC посылает `WINDOW_UPDATE` реже → GPU-воркер тормозит выдачу токенов. Не нужно никакого буфера в Redis — backpressure распространяется нативно поверх TCP/HTTP2.
* **Отмена**: пользователь нажимает «Stop» → браузер закрывает SSE-соединение → Nginx шлёт `RST_STREAM` → Regional Inference отменяет gRPC (`CANCEL`) → GPU-воркер останавливает генерацию и освобождает слот. Cancellation propagation занимает единицы миллисекунд.

### Почему отсутствуют Kafka и Redis Streams в горячем пути

Изначально напрашивается схема «API → Kafka → Scheduler → GPU → Redis Streams → SSE Gateway». Для нашей нагрузки она избыточна:

* **Kafka** добавила бы десятки миллисекунд latency и отдельный failure mode (insufficient ISR) без пользы: у нас нет необходимости хранить промпт после того, как GPU начал его обрабатывать.
* **Redis Streams как streaming buffer** превращается в точку с ~2.3M QPS чтения токенов — это дополнительный hop, который забирает 30–50% end-to-end latency (самый заметный для пользователя показатель — time-to-first-token).
* **Backpressure с буфером**: Redis не отдаёт обратное давление к GPU, и на медленном клиенте буфер растёт → нужно либо дропать токены, либо ограничивать concurrency. Прямой gRPC-стрим решает проблему «бесплатно».

Асинхронные события (usage, audit) пишутся **без Kafka** — напрямую в ClickHouse (async insert) и PostgreSQL (append-only таблица).

---

## (для §7/§10) Поток multimodal-запросов (картинка + промпт)

Мультимодальные модели получают картинку на vision-encoder в составе forward pass. Вопрос — как картинка, лежащая в S3, попадает на GPU worker. Выбираем **pull-model**: GPU worker сам тянет файл из S3 по короткоживущему presigned URL.

```
1. Client → presigned PUT → S3 (bucket в регионе GPU pool, US East)
                                       │
2. Chat Service: INSERT attachments (status=uploading);
   S3 event notification → status=ready
                                       │
3. Client → POST /v1/chat/completions
            { message: "что на фото?", attachments: [att_id] }
                                       │
4. Chat Service → attachments табл. → s3_bucket/s3_key
                                       │
5. Regional Inference:
   • генерирует presigned GET (TTL 120 сек, GET-only, pinned to sha256)
   • AssignWorker RPC к Scheduler
                                       │
6. Regional Inference → GPU worker: gRPC Generate(
        prompt_text,
        image_refs = [{ signed_url, mime_type, size_bytes }]
   )
                                       │
7. GPU worker ─(S3 GET по signed URL, ~20–50 мс)→ получает bytes
                  intra-region VPC, egress внутри AWS US East = $0
                                       │
8. GPU worker: vision encoder → image latents;
   склеивает prompt tokens + image latents → forward pass
                                       │
9. Стрим токенов обратно: GPU → gRPC → Regional Inference → SSE → клиент
```

### Почему pull-model, а не push через gRPC

| Вариант | Сеть внутри нашей инфры | Latency | Memory на Regional Inference |
|---------|-------------------------|---------|------------------------------|
| **GPU pull из S3** (выбрано) | 0 (intra-region AWS, бесплатно) | +20–50 мс S3 GET, выполняется параллельно с tokenize промпта | 0 |
| Push через gRPC от Regional Inference | пиковый трафик через gRPC-сеть | +S3 GET на Regional Inference **плюс** перепаковка в gRPC | размер картинки × concurrency на ноду |
| Предкалькуляция embeddings в Chat Service | меньше (embeds ~1 МБ), но нужен отдельный Vision Encoder сервис | +один hop | средняя |

Pull выигрывает по трём осям сразу: **intra-region трафик AWS бесплатен и быстр (25+ Gbit/s в пределах VPC)**, Regional Inference не становится прокси для мегабайтов, GPU worker уже умеет делать HTTP GET.

### Безопасность presigned GET для GPU

* TTL 120 секунд — дольше не нужно, обработка картинки занимает единицы секунд
* URL привязан к `sha256` объекта — невозможно подменить файл
* GET-only, bucket-policy: доступ только с IP-диапазона GPU VPC
* GPU worker **не имеет** IAM-роли к bucket — работает строго по signed URL, выданному Regional Inference

### Failure mode при S3 fetch

Если GPU worker не смог скачать картинку (S3 throttle / expired URL / network blip):
1. GPU worker возвращает `FETCH_FAILED` по gRPC
2. Regional Inference делает до 2 ретраев с новым presigned URL
3. При повторной неудаче — отмена: отправка `RST_STREAM` клиенту с error event в SSE; billing finalize с `actual_cost = 0`
4. Первые 2–3 секунды клиент видит keep-alive heartbeats (SSE `:ping\n\n`), потом либо токены либо error

---

## (для §10) Latency-бюджет для пользователей из разных регионов

| Регион пользователя | Клиент→edge | Edge→origin | Regional Inference→US GPU | Time to first token |
| ------------------- | ----------- | ----------- | ------------------------- | ------------------- |
| US East             | ~5 мс       | ~5 мс       | ~10 мс                    | ~150–300 мс         |
| EU                  | ~10 мс      | ~15 мс      | ~80 мс                    | ~250–400 мс         |
| APAC                | ~30 мс      | ~20 мс      | ~150 мс                   | ~400–600 мс         |

---

## (для §6 / §10) Размещение Rate Limiter и Billing

### Rate Limiter — per-region

Все лимиты (per-tier RPS, TPM для Web, usage-based для API) проверяются **локально в регионе**. Центральной синхронизации нет. Обоснование:

* Каждый HTTP-запрос проходит через Rate Limiter → пиковый RPS соответствует суммарному peak HTTP-трафика. Межрегиональный hop в US стоил бы +80–150 мс и стал бы единой точкой отказа.
* Допустимый дрейф: пользователь в теории может превысить лимит в 3 раза, если одновременно бьёт в три региона. На практике один клиент ходит в один регион по сетевой маршрутизации; multi-region abuse лечится отдельным anti-fraud (вне MVP).
* Сверка раз в минуту: агрегаты по user_id выгружаются в центральный ClickHouse для глобальных дашбордов и алертов на подозрительное поведение.

Данные хранит локальный Redis Cluster с TTL 60 сек (см. `rate_limits` в §5). Атомарность check-and-increment — Lua-скрипт.

### Billing Reservation — глобально централизован

В отличие от rate-лимитера, деньги **нельзя** списывать независимо в двух регионах — пользователь мог бы исчерпать баланс дважды. Поэтому `billing_accounts` живёт в **одном** PostgreSQL-кластере в US East.

| Аспект                   | Решение                                                     |
| ------------------------ | ----------------------------------------------------------- |
| Нагрузка                 | Только API: ~8 700 RPS peak (Web использует tier-лимиты Rate Limiter) |
| Latency от EU/APAC       | +80–150 мс — допустимо в рамках pre-inference шага          |
| Consistency              | Strong (SERIALIZABLE-транзакции для reserve / finalize)     |
| Failover                 | Sync-реплика в другой AZ US East (RPO=0, автопромоут), async-реплика в US West для DR (ручной promote при катастрофе US East, потеря ≤5 сек) |
| Shard key                | `hash(user_id) % 64` — все операции по одному аккаунту на одном шарде |

Flow биллинга для API-запроса:

```
1. API Gateway → Billing Reservation: reserve(user_id, max_cost)
     max_cost = prompt_tokens × prompt_price
              + max_completion_tokens × completion_price
              + Σ estimated_image_tokens × image_price   ← для multimodal
       где estimated_image_tokens ≈ ceil(attachment.size_bytes / 400)
     ▸ SELECT balance, reserved FROM billing_accounts WHERE user_id = ? FOR UPDATE
     ▸ IF balance - reserved >= max_cost THEN UPDATE reserved += max_cost
     ▸ возвращает reservation_id, либо 402 Payment Required
2. Inference выполняется
3. Chat Service (после закрытия SSE) → Billing Reservation: finalize(reservation_id, actual_cost)
     actual_cost учитывает фактические image_tokens от vision encoder
     ▸ UPDATE billing_accounts SET balance -= actual_cost, reserved -= max_cost
     ▸ разница (max_cost - actual_cost) возвращается на баланс
```

**Почему картинки нельзя тарифицировать только по `actual`:** без резервирования по `estimated_image_tokens` пользователь за $0.01 резерва прогонит 10 HD-картинок (~$0.05 фактической стоимости) → 5× overspend и отрицательный баланс. Оценка берётся **до** inference из `attachments.size_bytes`.

### Billing Ledger — аналитика (ClickHouse)

`usage_records` (см. §5) — append-only-лог каждого завершённого запроса для биллинговых дашбордов, выставления счетов и anti-abuse. Chat Service пишет туда напрямую через ClickHouse async insert после финализации резерва.
