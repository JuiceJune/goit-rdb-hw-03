# goit-rdb-hw-03 — Data-quality audit та EDA у PostgreSQL

Домашнє завдання 3 курсу «Реляційні бази даних»: завантаження реального датасету в PostgreSQL через `COPY FROM STDIN`, побудова staging- і clean-рівнів, аудит якості даних, аналіз розподілів і аномалій, десять DQL/EDA-запитів та reflection.

## Датасет

**NYC TLC Yellow Taxi Trip Records** (варіант A), **січень 2024**.

- Офіційне джерело: [TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
- Файл: [`yellow_tripdata_2024-01.parquet`](https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet) (~50 МБ, 2 964 624 поїздки)
- Опис колонок: [Data Dictionary – Yellow Taxi](https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf)
- Довідник зон (для запитів із назвами районів): [Taxi Zone Lookup](https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv)

## Формування вибірки

1. Parquet-файл завантажується з офіційного URL у `data/` (лише якщо його ще немає локально).
2. Читаються 12 колонок: `VendorID`, `tpep_pickup_datetime`, `tpep_dropoff_datetime`, `passenger_count`, `trip_distance`, `PULocationID`, `DOLocationID`, `payment_type`, `fare_amount`, `tip_amount`, `total_amount`, `store_and_fwd_flag`.
3. З усього місяця береться **випадкова вибірка 100 000 рядків** (`df.sample(n=100_000, random_state=42)`). Фіксований `random_state` робить вибірку відтворюваною; випадкова (а не `head`) — рівномірно покриває всі дні місяця.
4. Вибірка зберігається як контрольований CSV `data/hw3_taxi_sample.csv` (~7.3 МБ) і завантажується в PostgreSQL через `COPY FROM STDIN`.

## Кількість рядків

| Рівень | Таблиця | Рядків |
|---|---|---|
| staging (усі колонки `TEXT`) | `hw3_taxi_staging` | 100 000 |
| clean (типізована) | `hw3_taxi_trips` | 99 999 |
| довідник зон | `hw3_taxi_zones` | 265 |

Із clean виключено 1 рядок, у якому час висадки раніший за час посадки (`CHECK (dropoff_at >= pickup_at)`). Інші аномалії свідомо залишено й проаналізовано в notebook.

## Структура репозиторію

```
.
├── HW_2.ipynb   # notebook з усіма кроками та output клітинок
└── README.md
```

Дані до репозиторію не додаються. Notebook під час виконання створює папку `data/`, завантажує в неї Parquet-файл з офіційного URL і формує `data/hw3_taxi_sample.csv`. Завдяки фіксованому `random_state=42` CSV щоразу однаковий, тож результати відтворюються без ручного пошуку файлів.

## Запуск

### Google Colab

1. Відкрити `HW_2.ipynb` у Colab.
2. **Runtime → Restart and run all**.

Перша клітинка встановлює залежності через `%pip`, запускає локальний PostgreSQL через `pgserver` і виводить версію сервера. Дані завантажуються автоматично.

### Локально

Потрібен Python 3.12 та [uv](https://docs.astral.sh/uv/) (або `pip`).

```bash
uv venv --python 3.12
uv pip install ipykernel pgserver psycopg2-binary sqlalchemy pandas pyarrow ucimlrepo
```

Далі відкрити `HW_2.ipynb` у VS Code / Jupyter, вибрати ядро `.venv` і виконати **Restart & Run All**.

PostgreSQL окремо встановлювати не потрібно: `pgserver` містить вбудований сервер і зберігає дані в `/tmp/hw3_pg`.

## Зміст notebook

1. **Setup і завантаження** — `pgserver`, підключення через SQLAlchemy (`postgresql+psycopg2`), Parquet → CSV, staging-таблиця, `COPY FROM STDIN`.
2. **Staging і DDL** — типізована таблиця з `PRIMARY KEY`, `NOT NULL`, доменними `CHECK`; `INSERT INTO ... SELECT` з очищенням; обґрунтування типів (зокрема локалізація часу через `AT TIME ZONE 'America/New_York'` у `TIMESTAMPTZ`).
3. **Data-quality audit** — контроль кількості рядків, дублікати за business key (виявлено сторно-пари), null-rate та службові коди (`payment_type = 0`, зони 264/265, 0 пасажирів).
4. **Розподіли та аномалії** — розподіл `payment_type`, статистика `trip_distance` з медіаною й перцентилями, доменний outlier-check.
5. **Десять DQL/EDA-запитів** — формат business-question / SQL / interpretation.
6. **Reflection** — якість даних, features, leakage, рекомендації для pipeline, тип ML-задачі.
