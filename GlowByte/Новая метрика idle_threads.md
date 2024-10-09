#### **Добавил метрику в metrics.json:**
```json
{
    "description": "Service $0: Metrics.json TEST DESCROPTION FOR IDLE_THREADS.",
    "contexts": [
      "IMPALAD"
    ],
    "label": "Service $0 Metrcis.json TEST LABEL FOR IDLE_THREADS",
    "units": "NONE",
    "kind": "GAUGE",
    "key": "rpc.$0.idle_threads"
  },
```
Виды метрик:
- GAUGE
- COUNTER
- PROPERTY
- STATS
- SET
- HISTOGRAM
>[!note]
>В ImpalaServicePool::ToJson методе в файле impala-service-pool.cc видно, как значение idle_threads добавляется в json файл с метриками.
>По идее, это означает, что создавать такую метрику не нужно.
>Нужно, чтобы она отображалась в prometheus тоже.

Как я понимаю, prometheus метрики видно в вебсервере импалы в папке metrics, потому что изначально (в релизе 4.3.0 без моих правок) там есть метрика queue_overflow.
#### Добавил IDLE_THREADS_METRIC_KEY
В `impala-service-pool.h` объявил `IDLE_THREADS_METRIC_KEY` как `const char*` и переменную `idle_threads_` типа `IntGauge*`. Переменная `idle_threads_` должна хранить число свободных рабочих потоков.

>[!info]
> Вообще думаю что idle_threads_ можно не создавать потому что число свободных потоков можно узнать так:
> `service_queue_.estimated_idle_worker_count()`

В `impala-service-pool.cc` присвоил:
`const char * ImpalaServicePool::RPC_QUEUE_OVERFLOW_METRIC_KEY = "rpc.$0.rpcs_queue_overflow";`

В конструкторе
```cpp
ImpalaServicePool::ImpalaServicePool(const scoped_refptr<kudu::MetricEntity>& entity,
    int service_queue_length, kudu::rpc::GeneratedServiceIf* service,
    MemTracker* service_mem_tracker, const NetworkAddressPB& address,
    MetricGroup* rpc_metrics)
```
создал опсание метрики `idle_threads`:
```cpp
const TMetricDef& idle_threads_metric_def =
  MetricDefs::Get(IDLE_THREADS_METRIC_KEY, service_->service_name());
```
и проинициализировал `idle_threads_`:
```cpp
idle_threads_ =
rpc_metrics->RegisterMetric(new IntGauge(idle_threads_metric_def, 0L));
```

В методе `QueueInboundCall` добавил
```cpp
idle_threads_->SetValue(service_queue_.estimated_idle_worker_count());
```