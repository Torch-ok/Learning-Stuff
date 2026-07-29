---
tags:
  - Programming/Learning-Stuff/Python/API/FastAPI/Module_11
---
### 

#### Теоретический фундамент и архитектура

В сложных распределенных системах 2026 года логирования (Module 11.1) и метрик (Module 11.2) недостаточно для понимания того, где именно "застревает" запрос. **Распределенная трассировка (Distributed Tracing)** позволяет визуализировать жизненный цикл запроса в виде древовидной структуры из **Spans** (интервалов времени выполнения конкретных операций). Каждый путь запроса помечается уникальным **Trace ID**, а каждая подзадача — **Span ID**.

**Стандарт OpenTelemetry (OTel):** OpenTelemetry является индустриальным стандартом (CNCF) для сбора телеметрии. Архитектура включает:

- **OTel SDK**: Библиотеки внутри FastAPI для генерации данных.
- **Instrumentors**: Пакеты для автоматического перехвата вызовов к **SQLAlchemy** (AsyncSession), **Redis** и внешним HTTP-клиентам без ручного изменения кода.
- **Exporters**: Компоненты, отправляющие данные в **Collector** или напрямую в хранилище (Jaeger/Tempo) по протоколу OTLP (OpenTelemetry Line Protocol).

**Контекст и корреляция:** Для сохранения связности трассы между микросервисами используется **W3C Trace Context** (заголовок `traceparent`), который передается в HTTP-заголовках. Одной из важнейших практик является корреляция трасс и логов: `trace_id` инжектируется в структуру лога `structlog`, позволяя в интерфейсе **Grafana** мгновенно перейти от конкретной строки ошибки к полной визуальной схеме выполнения запроса в **Jaeger** или **Tempo**.

#### Практическая реализация и синтаксис

Инициализация трассировки в FastAPI 2026 года выполняется через асинхронный контекстный менеджер `lifespan`, обеспечивая готовность трейсера до начала обработки первого запроса [602, 8.3 draft].

**Код настройки OpenTelemetry и автоматической инструментации:**

```
from fastapi import FastAPI
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor

def setup_tracing(app: FastAPI, engine):
    # 1. Настройка провайдера и экспортера (Jaeger/Tempo)
    provider = TracerProvider()
    processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://tempo:4317"))
    provider.add_span_processor(processor)
    trace.set_tracer_provider(provider)

    # 2. Автоматическая инструментация стека [5.1 draft, 7.2 draft]
    FastAPIInstrumentor.instrument_app(app)
    SQLAlchemyInstrumentor().instrument(engine=engine.sync_engine)
    RedisInstrumentor().instrument()

# В main.py:
# app = FastAPI(lifespan=lifespan)
# setup_tracing(app, engine)
```

[Bigger Applications](https://fastapi.tiangolo.com/tutorial/bigger-applications/), [SQL (Relational) Databases](https://fastapi.tiangolo.com/tutorial/sql-databases/)

**Создание ручного спана и корреляция с логами:**

```
import structlog
from opentelemetry import trace

logger = structlog.get_logger()
tracer = trace.get_tracer(__name__)

@app.get("/heavy-task")
async def heavy_task():
    # Ручной спан для изоляции бизнес-логики
    with tracer.start_as_current_span("business_logic_calc") as span:
        span.set_attribute("user.plan", "premium")

        # Получение Trace ID для логирования [11.1 draft]
        trace_id = format(span.get_span_context().trace_id, '032x')
        logger.info("heavy_calc_started", trace_id=trace_id)

        # Симуляция тяжелой работы
        ...
        span.add_event("calculation_done")
```

[JSON Compatible Encoder](https://fastapi.tiangolo.com/tutorial/encoder/), [8.4 draft]

#### Глоссарий терминов

- **[Distributed Tracing Architecture](https://www.youtube.com/watch?v=JavaScriptMastery)** — технология мониторинга, отслеживающая путь запроса через все компоненты системы.
- **OpenTelemetry (OTel) SDK & Collector** — набор инструментов для сбора, обработки и экспорта данных телеметрии (трасс, метрик, логов).
- **Trace ID & Span ID** — уникальные идентификаторы: первый для всей цепочки запроса, второй для конкретного шага внутри цепочки.
- **W3C Trace Context (`traceparent`)** — стандартизированный формат HTTP-заголовка для проброса метаданных трассировки между сервисами.
- **Jaeger / Grafana Tempo** — системы хранения и визуализации трасс, позволяющие анализировать задержки в интерактивном режиме.
- **Trace-to-Log Correlation** — механизм связки событий в логах с соответствующими им спанами в трейсинге через общие ID.

#### Практический кейс: Поиск Bottleneck-задержек в сквозной цепочке запроса

**Сценарий:** Эндпоинт оформления заказа `/order/checkout` стал работать медленно (> 2 секунд). В цепочке участвуют: проверка остатков в **Redis**, запись в **PostgreSQL** и вызов внешней платежной системы по HTTP.

**Анализ через трассировку:**

1. **Визуализация**: В Jaeger/Tempo инженеры видят длинную полосу (Span) операции `checkout`.
2. **Выявление**: Автоматическая инструментация показывает, что запрос к Redis занял 10 мс, запись в PostgreSQL — 50 мс, а ожидание ответа от `requests.get` к банку — 1.9 секунды.
3. **Оптимизация**: Выявлено, что внешнее API вызывалось синхронно, блокируя воркер. Решение — переход на `httpx.AsyncClient` и установка жестких `timeouts`.
4. **Sampling**: Для экономии места в хранилище на High-load (3000 RPS) настраивается `Head-based Sampling`: записывается только 5% всех успешных трасс, но 100% трасс, завершившихся ошибкой (5xx) [66, 11.2 draft].

#### Вопросы и задания для самопроверки

1. **Вопрос:** Как механизм W3C Trace Context позволяет разным микросервисам (на Python, Go, Java) "понимать", что они обрабатывают один и тот один запрос?
    - _Ответ: Через заголовок `traceparent`, который содержит общую версию, `Trace ID`, родительский `Span ID` и флаги трассировки._
2. **Задача:** Трассы для SQLAlchemy отображаются в Jaeger как отдельные, не связанные с HTTP-запросом. Как заставить их стать "детьми" (Child Spans) основного запроса?
    - _Решение: Убедитесь, что `TracerProvider` инициализирован до инструментации SQLAlchemy и что асинхронные задачи выполняются в рамках того же контекста `contextvars`, который использует OTel SDK [11.1 draft]._
3. **Практическое задание:** Добавьте в функцию `process_payment` ручной спан. В блоке `except` используйте `span.set_status(StatusCode.ERROR)` и `span.record_exception(e)`, чтобы ошибка отображалась в интерфейсе Jaeger красным цветом с полным текстом исключения [514, 8.1 draft].