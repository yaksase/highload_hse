# Drafts to be incorporated in §11 (Список серверов)

Этот файл — архив материалов, которые рождаются по ходу работы над §3–§6, но по сути относятся к §11 «Список серверов» / bill-of-materials. Не входят в финальный текст текущих секций — переносятся при заполнении §11.

Формат, ожидаемый в §11 (по `requirements/homework_architecture.md`):
> Таблица с конфигурациями, количеством серверов и сервисами расположенными на них. В случае использования kubernetes или иной виртуализации, список контейнеров с аллокацией ресурсов.

То есть две вложенные таблицы:
1. **Серверы (bare-metal)** — по типам нод, с конфигурацией и числом штук на ДЦ.
2. **Контейнеры (k8s pod-ы)** — какие pod-ы где живут, с CPU/RAM requests/limits и числом реплик.

---

## 1. Типы k8s-нод (из §4.2)

### Control plane

* **Роль:** kube-apiserver, etcd, kube-scheduler, kube-controller-manager. `taint: NoSchedule` для обычных pod-ов.
* **Конфигурация:** 8 vCPU / 16 GB RAM / SSD 200 GB (под etcd, нужны быстрые fsync). Сеть — внутренняя 10 GbE.
* **Число:** 3 ноды в каждом ДЦ (нужен кворум etcd, переживают 1 отказ). При желании 5 нод (переживают 2 отказа), но для нашего размера достаточно 3.
* **Итого:** 3 × 4 ДЦ = **12 нод**.

### Edge

* **Роль:** ingress-nginx (DaemonSet) + MetalLB speaker (DaemonSet). На пути внешнего HTTPS-трафика.
* **Конфигурация:** **32 vCPU / 64 GB RAM / 2×25 GbE NIC** (TLS handshake CPU-bound; 50 Gbps NIC даёт запас под пиковый throughput). Локальный SSD 200 GB под логи/буферы.
* **Число (из расчёта §4.3, модель 2N @ 50%):**
  * US East: 4
  * US West: 4
  * EU Central: 6
  * APAC: 6
* **Итого: 20 нод.**

### Worker

* **Роль:** Deployment pod-ы — `api-gateway`, `chat`, `files`, `regional-inference`, `inference-scheduler` (только в US).
* **Конфигурация:** 32 vCPU / 64 GB RAM / 10 GbE NIC. Локальный SSD 200 GB.
* **Число:** должно определиться суммой ресурсов pod-ов (см. §11.2 ниже), плюс запас на отказ ноды (N+1). Прикидка: **~12–15 нод на ДЦ**.

### GPU

* **Роль:** `gpu-worker` pod-ы (continuous batching на H100).
* **Конфигурация:** 8×H100 (80 GB HBM3) / 2× Intel Xeon Platinum / 2 TB DDR5 RAM / 8×3.84 TB NVMe / 4×400 Gbps NIC (NDR InfiniBand или RoCE для intra-cluster collective ops при tensor-parallel inference на больших моделях).
* **Число:** определяется через RPS / batch capacity GPU (зависит от модели). Прикидка: для пика 137 000 inference RPS при средней TTFT 1–3 сек и continuous batching ~64 запросов/GPU — нужно **~2000–2500 H100** = **250–315 нод** на US East + US West.
* **Размещение:** только US East и US West (см. §3.2 — обоснование через утилизацию, power и operational overhead).

---

## 2. Контейнеры (k8s pod-ы)

### Stateless backend (Deployment + HPA)

Baseline N+1 prim. расчёт исходя из RPS из §2/§3:

| Pod | Peak RPS (global) | Профиль | Pod capacity (без TLS) | Min N | **N+1 baseline (4 ДЦ)** | CPU/RAM на pod |
| --- | ---: | --- | ---: | ---: | ---: | --- |
| `api-gateway` | ~78 000 | короткие auth/billing/proxy | ~15 000 RPS | 6 | **7 × 4 = 28** | request 1 CPU / 2 GB, limit 2 CPU / 4 GB |
| `chat` | ~60 000 | долгие SSE-стримы, HTTP/2 | ~15 000 RPS | 4 | **5 × 4 = 20** | request 1 CPU / 2 GB, limit 2 CPU / 4 GB |
| `files` | ~20 000 | лёгкая HTTP-логика | ~20 000 RPS | 1 | **2 × 4 = 8** | request 0.5 CPU / 1 GB, limit 1 CPU / 2 GB |
| `regional-inference` (EU/APAC) | ~88 600 | gRPC streaming в US | ~20 000 RPS | 5 | **6 × 2 ДЦ = 12** | request 1 CPU / 2 GB, limit 2 CPU / 4 GB |
| `inference-scheduler` (US only) | ~137 000 | gRPC control plane | ~25 000 RPS | 6 | **7 × 2 ДЦ = 14** | request 2 CPU / 4 GB, limit 4 CPU / 8 GB |
| `gpu-worker` | — | continuous batching на H100 | ~зависит от модели | — | по числу H100 (см. §11.1) | 8 GPU / 2 CPU / 256 GB RAM |

HPA масштабирует выше baseline под текущую нагрузку. Метрика — CPU 60% для api-gateway/chat/files, queue depth для inference-scheduler.

### DaemonSet

| Pod | Где живёт | CPU/RAM на pod |
| --- | --- | --- |
| `ingress-nginx` | на каждой edge-ноде | request 8 CPU / 16 GB, limit 24 CPU / 32 GB |
| `metallb-speaker` | на каждой edge-ноде | request 0.1 CPU / 100 MB, limit 0.5 CPU / 500 MB |

---

## 3. Stateful серверы (вне k8s, bare-metal)

### PostgreSQL/Citus

* **users + colocated (conversations, api_keys, attachments_metadata):** 64 шарда; шарды по `hash(user_id)`. Кластер: 1 coordinator + worker-ноды.
  * Конфигурация worker: 16 vCPU / 64 GB / 4 TB NVMe SSD / 10 GbE.
  * Реплики: 1 primary + 1 sync + 1 async = ×3.
  * Размещение: US East + US West.
  * Число: TBD при заполнении §11 на основе размера шарда.
* **billing_accounts:** отдельный кластер, SERIALIZABLE consistency. 32 шарда. US East only (см. §6.4).
* **audit_log:** US East only. 16 партиций по дате.
* **subscription_tiers / models:** reference tables, logical replication на каждый шард.

### ScyllaDB

* **messages + messages_by_request:** multi-DC keyspace, RF=3 в US East+West, RF=1 в EU+APAC.
* Конфигурация ноды: 32 vCPU / 256 GB / 16 TB NVMe / 25 GbE.
* Число (из §6.5): **6 нод × 4 ДЦ = 24 ноды**.

### ClickHouse

* **usage_records:** ReplicatedMergeTree, 8 шардов × 2 реплики = 16 нод.
* Конфигурация: 16 vCPU / 128 GB / 8 TB NVMe (hot) + 32 TB HDD (cold) / 25 GbE.
* Размещение: US East only.

### Redis Cluster

* **sessions, rate_limits, tier_cache, model_cache:** per-region.
* Конфигурация ноды: 8 vCPU / 64 GB / 1 TB NVMe / 10 GbE.
* Число: ~16 master + 16 replica на регион (16384 hash slots / 1024 slots на master). Уточнить при §11.
* Размещение: ×4 региона.

### Ceph RGW

* **attachments blob:** ~1200 OSD-нод суммарно (из §6.7), EC 8+3, multisite.
* Конфигурация OSD-ноды: 24×18 TB HDD + 1 NVMe (BlueStore DB) / 16 vCPU / 64 GB / 25 GbE.
* Service-ноды (MON/MGR/RGW): 3 × 4 региона = 12.
* Распределение: US East 40% / US West 30% / EU 20% / APAC 10%.

---

## 4. Сетевые компоненты (cross-DC backbone)

Эти серверы и каналы — отдельная часть §11, потому что они не k8s-ноды, но без них архитектура не работает.

### Cross-DC backbone-каналы

Выделенные оптические каналы между нашими ДЦ (dark fiber или MPLS у telco), приватный iBGP внутри нашей AS.

| Канал | Конфигурация | Резервирование |
| --- | --- | --- |
| EU Central ↔ US East | 2× 100 Gbps DWDM, geo-diverse физические пути | ✓ |
| APAC ↔ US East | 2× 100 Gbps DWDM, geo-diverse физические пути | ✓ |
| US East ↔ US West | 2× 100 Gbps DWDM, geo-diverse | ✓ |
| EU Central ↔ APAC | опционально 1× 100 Gbps (для меньшей нагрузки) | — |

Пиковый трафик cross-DC (из обсуждения в чате):
* EU→US inference: ~9.2 Гбит/с peak
* APAC→US inference: ~7.5 Гбит/с peak
* US East→US West: репликация PG sync + Scylla multi-DC + Ceph multisite, ~10–20 Гбит/с peak
* Итого: 100 Gbps канала с большим запасом

### Edge-роутеры

В каждом ДЦ — пара edge-роутеров уровня Juniper MX / Arista 7800 / Cisco ASR (HA-pair, VRRP).

Роли:
* Внешний BGP — сессия с upstream-провайдерами ДЦ, анонс Anycast `/24` (см. §3.5)
* Приватный iBGP — связность с другими нашими ДЦ через backbone
* Маршрутизация внутренних подсетей `10.0.0.0/8` между ДЦ через backbone
* DDoS-фильтрация на первом hop-е (опционально, или у upstream-провайдера)

| ДЦ | Edge-роутеры |
| --- | --- |
| US East | 2 (HA-pair) |
| US West | 2 |
| EU Central | 2 |
| APAC | 2 |
| **Итого** | **8 edge-роутеров** |

---

## 5. Что ещё нужно уточнить при заполнении §11

* [ ] Точное число worker-нод на ДЦ (после суммирования CPU/RAM по pod-ам + запас N+1).
* [ ] Точное число H100 на US East / US West (зависит от выбранной модели — GPT-OSS 120B / Llama 3 70B / иное, тут нужно посчитать по batch capacity на 1 GPU и пиковому inference RPS).
* [ ] Точное число PG worker-нод (после расчёта `users / shard size`).
* [ ] Точное число Redis-нод per-region — сейчас в notes стоит «оверпровиженинг ×4», нужно пересчитать на основе peak QPS sessions/rate_limits.
* [ ] DDoS-фильтрация — у upstream-провайдера или своя (Arbor TMS, FastNetMon).
* [ ] CDN для статики (если будет в MVP web-UI) — Cloudflare / Fastly / своё.
