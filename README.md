# Log Monitoring Stack

Централизованный сбор и визуализация логов Docker-контейнеров на базе **Loki + Promtail + Grafana**.

## Архитектура

```
Docker-контейнеры → Promtail → Loki → Grafana
```

- **Promtail** — собирает логи с Docker-сокета, отправляет в Loki
- **Loki** — хранит и индексирует логи (retention 30 дней)
- **Grafana** — визуализация, дашборды, фильтрация по сервисам

## Быстрый старт

```bash
# 1. Скопировать и заполнить переменные окружения
cp .env.example .env

# 2. Поднять стек
docker-compose up -d

# 3. Открыть Grafana
http://localhost:3000
```

Логин и пароль задаются в `.env` (`GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD`).

## Структура проекта

```
log_monitoring/
├── docker-compose.yml          # Оркестрация сервисов
├── .env                        # Переменные окружения (не в git)
├── .env.example                # Шаблон переменных
├── loki/
│   ├── config.yml              # Конфиг Loki (retention, лимиты)
│   └── data/                   # Данные Loki (не в git)
├── promtail/
│   ├── config.yml              # Конфиг Promtail (сбор логов)
│   └── data/                   # Позиции Promtail (не в git)
└── grafana/
    ├── data/                   # Данные Grafana (не в git)
    ├── dashboards/
    │   └── work_for_everyone.json  # Дашборд мониторинга
    └── provisioning/
        └── datasources/
            └── loki.yml        # Подключение Loki как источника данных
```

## Переменные окружения

| Переменная | Описание | Пример |
|---|---|---|
| `GRAFANA_VERSION` | Версия образа Grafana | `10.2.3` |
| `LOKI_VERSION` | Версия образа Loki | `2.9.3` |
| `PROMTAIL_VERSION` | Версия образа Promtail | `2.9.3` |
| `GRAFANA_PORT` | Порт Grafana на хосте | `3000` |
| `GF_SECURITY_ADMIN_USER` | Логин администратора | `admin` |
| `GF_SECURITY_ADMIN_PASSWORD` | Пароль администратора | `changeme` |

## Контроль размера логов

### Retention (Loki)
Логи автоматически удаляются через **30 дней**. Compactor запускается каждые 10 минут, фактическое удаление — через 2 часа после истечения.

### Rate limits (Loki)
| Параметр | Значение | Описание |
|---|---|---|
| `ingestion_rate_mb` | 16 МБ/с | Максимальная скорость приёма логов |
| `ingestion_burst_size_mb` | 32 МБ | Допустимый burst |
| `max_global_streams_per_user` | 10 000 | Лимит активных стримов |
| `max_entries_limit_per_query` | 10 000 | Лимит строк на запрос из Grafana |
| `max_query_length` | 720h | Максимальный диапазон запроса |

### Docker logging
Каждый контейнер ограничен по размеру stdout/stderr логов на хосте:
- Loki, Grafana: `max-size=10m`, `max-file=3` (до 30 МБ)
- Promtail: `max-size=5m`, `max-file=3` (до 15 МБ)

### Лимиты памяти контейнеров
| Контейнер | Лимит |
|---|---|
| Loki | 512 МБ |
| Grafana | 256 МБ |
| Promtail | 128 МБ |

## Подключение новых сервисов

Promtail автоматически обнаруживает все Docker-контейнеры. Дополнительная настройка не требуется — логи нового контейнера появятся в Grafana автоматически под именем контейнера (`job`).

## Управление стеком

```bash
# Запуск
docker-compose up -d

# Остановка (данные сохраняются)
docker-compose down

# Остановка с удалением данных
docker-compose down -v

# Просмотр логов стека
docker-compose logs -f

# Перезапуск одного сервиса
docker-compose restart loki
```

## Дашборд

Дашборд **"Работа для всех — Логи"** доступен сразу после запуска. Включает:
- График объёма логов по сервисам
- Потоки логов для каждого сервиса
- Фильтр ошибок и предупреждений (`error`, `warn`, `exception`, `critical`)
