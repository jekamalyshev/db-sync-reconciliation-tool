# DB Sync Reconciliation Tool

Инструмент двухэтапной сверки данных между **Oracle** и **PostgreSQL** для объёмов 3–5 млн строк.

## Назначение

Инструмент позволяет сравнивать данные одной и той же таблицы в двух базах данных и детально выявлять расхождения как на уровне агрегатов (суммы, количество строк по бизнес-ключу), так и на уровне каждого поля каждой записи.

## Архитектура: двухэтапная сверка

```
Этап 1: Агрегированная сверка
  Oracle: GROUP BY composite_keys → SUM(numeric), COUNT(*)
  Postgres: то же самое
  → outer merge → MISMATCH / ONLY_IN_ORACLE / ONLY_IN_POSTGRES / OK

Этап 2: Field-level comparison (для строк с расхождениями)
  → детальная загрузка строк по ключам расхождений
  → построчное, поколоночное сравнение
  → DataFrame: key | column_name | oracle_value | postgres_value | diff_type | delta
```

## P0-исправления

| # | Проблема | Исправление |
|---|---|---|
| P0-fix 1 | Двойное присваивание `self.results` перезаписывало `stage2_details` | Единственное итоговое присваивание, `stage2_field_diffs` всегда в результате |
| P0-fix 2 | `LIMIT`/`ROWNUM` без `ORDER BY` — Oracle и Postgres возвращали разные подмножества | `ORDER BY` по нормализованным ключам добавлен перед `LIMIT`/`ROWNUM` |
| P0-fix 3 | Stage 2 возвращал просто concat строк без сравнения полей | Реализован полноценный field-level diff через `_build_field_level_diffs()` |

## Выходные данные `results`

```python
results = {
    "stage1_merged":         pd.DataFrame,  # агрегированное сравнение
    "stage1_discrepancies":  pd.DataFrame,  # только строки с расхождениями
    "stage2_field_diffs":    pd.DataFrame,  # field-level расхождения:
    #   key           — строковый ключ записи
    #   column_name   — имя поля с расхождением
    #   oracle_value  — значение в Oracle
    #   postgres_value — значение в Postgres
    #   diff_type     — ONLY_IN_ORACLE | ONLY_IN_POSTGRES | NUMERIC_DELTA
    #                   | VALUE_MISMATCH | NULL_MISMATCH
    #   delta         — числовая разница (только для NUMERIC_DELTA)
    "summary": {
        "total_records": int,
        "discrepancies_count": int,
        "stage2_field_diffs_count": int,
        "status_counts": dict,
    }
}
```

## Требования

```
python >= 3.10
pandas
numpy
sqlalchemy
oracledb          # pip install oracledb
psycopg2-binary   # pip install psycopg2-binary
```

## Быстрый старт

1. Откройте `DB_Sinc-Reconciliation_tool.ipynb` в Jupyter.
2. Заполните `ReconciliationConfig` в Блоке 5:
   - строки подключения Oracle и Postgres
   - имена схем и таблиц
   - `composite_keys` — список колонок бизнес-ключа
3. Последовательно запустите все блоки.

```python
config = ReconciliationConfig(
    oracle_connection_string="oracle+oracledb://user:pass@host:1521/service",
    postgres_connection_string="postgresql+psycopg2://user:pass@host:5432/db",
    oracle_schema="MY_SCHEMA",
    oracle_table="MY_TABLE",
    postgres_schema="public",
    postgres_table="my_table",
    composite_keys=["regn", "datereport"],
    report_date_column="datereport",
    exclusions=["LOADID", "CHANGEFLAG"],
    sample_size=None,        # None = полный прогон; >0 = выборка с ORDER BY
    decimal_precision=2,
    year_from=2020,
    year_to=2024,
)
reconciliator = DataReconciliator(config)
reconciliator.connect()
results = reconciliator.run_full_reconciliation()
reconciliator.disconnect()
```

## Параметры ReconciliationConfig

| Параметр | Тип | Описание |
|---|---|---|
| `oracle_connection_string` | str | SQLAlchemy connection string для Oracle |
| `postgres_connection_string` | str | SQLAlchemy connection string для Postgres |
| `oracle_schema` / `postgres_schema` | str | Имена схем |
| `oracle_table` / `postgres_table` | str | Имена таблиц |
| `composite_keys` | List[str] | Поля бизнес-ключа для группировки и JOIN |
| `report_date_column` | str | Поле даты для фильтрации по годам |
| `exclusions` | List[str] | Технические поля, исключаемые из сверки |
| `sample_size` | int \| None | Лимит групп; `None` = без ограничений |
| `specific_keys` | Dict \| None | Точечный фильтр по значениям ключей |
| `decimal_precision` | int | Точность округления числовых агрегатов |
| `year_from` / `year_to` | int \| None | Диапазон лет для фильтрации |

## Лицензия

MIT
