Данный README содержит отчет для задания "Методы инженерии и проектирования витрин"

## Содержание

- [Введение](#введение)
- [Бизнес-требования](#бизнес-требования)
- [Отчет](*отчет)
  - [Концептуальная модель](#концептуальная-модель)
  - [Логическая модель](#логическая-модель)
  - [Физическая модель и архитектурная схема](#физическая-модель-и-архитектура-хранилища-данных)

### Концептуальная модель

Концептуальная модель описывает предметную область рынка акций РФ. Источник данных - API ISS Московской биржи MOEX. Модель ориентирована на исторический анализ минутных баров по конкретизированному списку акций. В концептуальную модель включены следующие предмето-ориентированные сущности: сектор, компания-эмитет, валюта, инструмент, торговый календарь, торговая сессия (утро, день, вечер) и минутный бар.
Сектор классифицирует компанию - компания выпускает один или несколько инструментов - инструмент котируется в определенной валюте - торговый календарь определяет торговые дни и сессии - торговая сессия задает временные рамки, внутри которых формируются минутные бары.

```mermaid
erDiagram
	sector ||--o{ company : классифицирует
	company ||--o{ instrument : выпускает
	currency ||--o{ instrument : котируется
	instrument ||--o{ bar : связывается
	trading_calendar ||--o{ session : определяет
	session ||--o{ bar : задает
```

Так же надо отметить, что корпоративные действия, такие как дивиденды, сплиты и консолидации исключены из набора данных, в связи с этим ценовые метрики рассчитываются по сырым, нескорректированным данным. Доходность, волатильность и сравнение цен на длинных периодах времени могут быть некорректны при наличии корпоративных событий.

**Основной бизнес-процесс -** получение и хранение минутных OHLCV-наблюдений по акциям из строгого списка. Гранулярность основного факта: один минутный бар для одного инструмента в одной торговой сессии (бар + инструмент + торговая сессия)

### Логическая модель

Выбор гранулярности трактуется из концептуальной модели выше - один минутный бар для одного инструмента в одной торговой сессии (бар + инструмент + торговая сессия). Построение витрины для быстрого согласованного результата будем производить по Кимбаллу. Начнем с таблицы фактов и вокруг неё таблицы измерений - звезда.

Определим таблицу фактов и её атрибуты:

**FACT_MARKET_BAR_1MIN:**

| Атрибут         | Тип       | Ключ        | Описание                                |
| --------------- | --------- | ----------- | --------------------------------------- |
| bar_sk          | bigint    | PK          | Суррогатный ключ строки факта           |
| instrument_sk   | bigint    | FK          | Версия инструмента                      |
| session_sk      | bigint    | FK          | Торговая сессия                         |
| date_sk         | bigint    | FK          | Дата бара                               |
| bar_start_utc   | timestamp | Бизнес ключ | Начало бара в UTC                       |
| bar_end_utc     | timestamp |             | Конец бара в UTC                        |
| bar_start_local | timestamp |             | Начало бара по времени биржи            |
| bar_end_local   | timestamp |             | Конец бара по времени биржи             |
| trade_date      | date      |             | Торговая дата                           |
| open_price      | decimal   |             | Цена открытия                           |
| high_price      | decimal   |             | Наивысшая цена                          |
| low_price       | decimal   |             | Наименьшая цена                         |
| close_price     | decimal   |             | Цена закрытия                           |
| volume_lots     | bigint    |             | Объем лотов                             |
| volume_shares   | bigint    |             | Объем в акциях (`volume_lots*lot_size`) |
| turnover_amount | decimal   |             | Оборот в валюте                         |
| trade_count     | bigint    |             | Количество сделок                       |
| source_batch_id | string    |             | Индентификатор пакета загрузки          |
| loaded_at       | timestamp |             | Время загрузки                          |

Далее определим таблицы измерений и справочники, по концептуальной модели выше:

**DIM_SECTOR (справочник секторов/отраслей компании):**

| Атрибут     | Тип     | Ключ | Описание                     |
| ----------- | ------- | ---- | ---------------------------- |
| sector_id   | string  | PK   | Индентификатор сектора       |
| sector_name | string  |      | Название сектора             |
| active_from | date    |      | Дата начала действия         |
| active_to   | date    |      | Дата конца действия          |
| is_current  | boolean |      | Признак актуальности сектора |

**DIM_COMPANY (справочник компаний-эмитетов):**

| Атрибут      | Тип     | Ключ     | Описание                      |
| ------------ | ------- | -------- | ----------------------------- |
| company_id   | string  | PK       | Индентификатор компании       |
| company_name | string  |          | Название компании             |
| sector_id    | string  | FK       | Ссылка на сектор              |
| inn          | string  | optional | ИНН компании                  |
| issuer_code  | string  | optional | Код эмитета                   |
| active_from  | date    |          | Начало действии записи        |
| active_to    | date    |          | Конец действия записи         |
| is_current   | boolean |          | Признак актуальности компании |

Инн компании представлен для отчетности, биржа не отдает его по API, поэтому возможна реализация через API КонтурФокус.

**DIM_CURRENCY (справочник валют):**

| Атрибут       | Тип    | Ключ | Описание                   |
| ------------- | ------ | ---- | -------------------------- |
| currency_code | string | PK   | Бизнес-ключ кода (RUB)     |
| currency_name | string |      | Название валюты            |
| symbol        | string |      | Символ валюты              |
| minor_unit    | int    |      | Количество минорный единиц |

На Московской бирже в основном RUB, так что справочник исключительно для корректности модели. В дальнейшем возможно интеграция с СПБ-биржей, где раньше были представлены акции иностранных компаний.

**DIM_INSTRUMENT (главное историческое измерение):**

| Атрибут                 | Тип       | Ключ        | Описание                                           |
| ----------------------- | --------- | ----------- | -------------------------------------------------- |
| instrument_sk           | bigint    | PK          | Суррогатный ключ версии инструмента                |
| instrument_business_key | string    | natural key | Бизнес-ключ                                        |
| secid                   | string    | source key  | Тикер бумаги MOEX                                  |
| isin                    | string    | optional    | Международный идентификатор                        |
| instrument_name         | string    |             | Название инструмента                               |
| instrument_type         | string    |             | Тип, например common_share / preferred_share       |
| company_id              | string    | FK          | Компания-эмитент                                   |
| currency_code           | string    | FK          | Валюта котирования                                 |
| sector_id               | string    | FK          | Сектор на момент версии                            |
| section_code            | string    |             | Торговая секция, например TQBR                     |
| lot_size                | int       |             | Размер лота                                        |
| trading_status          | string    |             | Статус торгов                                      |
| is_watchlist            | boolean   |             | Признак включения в список отслеживаемых           |
| valid_from              | timestamp |             | Начало действия версии                             |
| valid_to                | timestamp |             | Конец действия версии                              |
| is_current              | boolean   |             | Признак текущей версии                             |
| hash_diff               | string    |             | Хэш изменяемых атрибутов для обнаружения изменений |
| source_updated_at       | timestamp |             | Время обновления в источнике                       |
| loaded_at               | timestamp |             | Время загрузки в DWH                               |

Самая большая таблица измерений, хранит:

- бизнес-индентичность акций;
- торговые параметры;
- исторические версии параметров;
- связь с компанией, валютой и сектором.

Бизнес-ключ intrument*business_key должен быть достаточно устойчив, поэтому будет формироваться из `secid+'*'+section_code`.
Обозначим события, при которых будут происходить исторические изменения:

- intrument_name: название бумаги может поменяться
- company_id: изменении привязки к эмитету, причина реорганизация или исправления справочника
- sector_id: сектор может быть изменен
- section_code: инструмент может перейти в другую секцию
- lot_size: изменение размера лота
- trading_status: изменение статуса торгов
- is_watchlist: исключение из состава отслеживаемых

**DIM_DATE (торговый календарь):**

| Атрибут           | Тип     | Ключ        | Описание                                       |
| ----------------- | ------- | ----------- | ---------------------------------------------- |
| date_sk           | int     | PK          | Суррогатный ключ даты                          |
| calendar_date     | date    | natural key | Календарная дата                               |
| trade_date        | date    |             | Торговая дата, может совпадать с calendar_date |
| is_trading_day    | boolean |             | Признак торгового дня                          |
| is_weekend        | boolean |             | Признак выходного                              |
| is_holiday        | boolean |             | Признак праздника                              |
| day_of_week       | int     |             | День недели                                    |
| week_number       | int     |             | Номер недели                                   |
| month_number      | int     |             | Номер месяца                                   |
| quarter_number    | int     |             | Номер квартала                                 |
| year_number       | int     |             | Год                                            |
| exchange_timezone | string  |             | Europe/Moscow                                  |
| session_count     | int     | optional    | Количество сессий за день                      |

**DIM_SESSION (торговая сессия):**

| Атрибут             | Тип       | Ключ             | Описание                             |
| ------------------- | --------- | ---------------- | ------------------------------------ |
| session_sk          | bigint    | PK               | Суррогатный ключ сессии              |
| date_sk             | int       | FK               | Дата сессии                          |
| session_code        | string    | natural key part | Код сессии                           |
| session_type        | string    |                  | Тип сессии                           |
| session_name        | string    |                  | Название сессии                      |
| session_start_local | timestamp |                  | Начало по времени биржи              |
| session_end_local   | timestamp |                  | Конец по времени биржи               |
| session_start_utc   | timestamp |                  | Начало в UTC                         |
| session_end_utc     | timestamp |                  | Конец в UTC                          |
| is_regular_session  | boolean   |                  | Признак основной сессии              |
| duration_minutes    | int       |                  | Длительность в минутах               |
| session_status      | string    |                  | scheduled / active / closed / halted |
| loaded_at           | timestamp |                  | Время загрузки                       |

```mermaid
erDiagram
	dim_sector ||--o{ dim_company : классифицирует
	dim_company ||--o{ dim_instrument : выпускает
	dim_currency ||--o{ dim_instrument : котируется
	dim_instrument ||--o{ fact_market_bar_1min : связывается
	dim_date ||--o{ dim_session : определяет
	dim_session ||--o{ fact_market_bar_1min : задает
	dim_date ||--o{ fact_market_bar_1min : задает

	fact_market_bar_1min {
		bigint bar_sk PK
		bigint instrument_sk FK
		bigint session_sk FK
		int date_sk FK
		timestamp bar_start_utc
		timestamp bar_end_utc
		timestamp bar_start_local
		timestamp bar_end_local
		date trade_date
		decimal open_price
		decimal high_price
		decimal low_price
		decimal close_price
		bigint volume_lots
		bigint volume_shares
		decimal turnover_amount
		bigint trade_count
		string source_batch_id
		timestamp loaded_at
	}

	dim_sector {
		string sector_id PK
		string sector_name
		date active_from
		date active_to
		boolean is_current
	}

	dim_company {
		string company_id PK
		string company_name
		string sector_id FK
		string inn
		string issuer_code
		date active_from
		date active_to
		boolean is_current
	}

	dim_currency {
		string currency_code PK
		string currency_name
		string symbol
		int minor_unit
	}

	dim_instrument {
		bigint instrument_sk PK
		string instrument_business_key
		string secid
		string isin
		string instrument_name
		string instrument_type
		string company_id
		string currency_code
		string sector_id
		string section_code
		int lot_size
		string trading_statuis
		boolean is_watchlist
		timestamp valid_from
		timestamp valit_to
		boolean is_current
		string hash_diff
		timestamp source_updated_at
		timestamp loaded_at
	}

	dim_date {
		int date_sk PK
		date calendar_date
		date trade_date
		boolean is_trading_day
		boolean is_weekend
		boolean is_holiday
		int day_of_week
		int week_number
		int month_number
		int quarter_number
		int year_number
		string exchange_timezone
		int session_count
	}

	dim_session {
		bigint session_sk PK
		int date_sk FK
		string session_code
		string session_type
		string session_name
		timestamp session_start_local
		timestamp session_end_local
		timestamp session_start_utc
		timestamp session_end_utc
		boolean is_regular_session
		int duration_minutes
        string session_status
        timestamp loaded_at
	}
```

### Физическая модель и архитектура хранилища данных

**Хранилища**:
HDFS - RAW
Hive - Staging
Hive - ODS
Greenplum - DDS
Clickhouse - Data Mart
Postgres Serving - Mart

Зафиксируем вывод из шагов выше, эти ограничения послужат для проектирования физической модели:

- Источник - ISS API MOEX
- Scope - список около 100 ликвидных акций РФ
- Основной интервал - 1 мин.
- Торговая секция считается фиксированной и принимается как TQBR
- Историчность торговых параметров инструмента обеспечивается через dim_instrument посредством SCD 2
- Все ценовые метрики являются raw / неаддитивными
- Факт содержит только фактически полученные бары
- Время хранится в UTC и в локальном времени биржи MSK
- Торговая дата может отличаться от календарной даты для вечерних сессий и клиринговых переходов

Общее физическое представление:

| Слой           | Система    | Физические объекты                                 | Назначение                                                             |
| -------------- | ---------- | -------------------------------------------------- | ---------------------------------------------------------------------- |
| RAW            | HDFS       | JSON-файлы ответов MOEX ISS                        | Сохранить сырой источник без изменений                                 |
| STAGING        | Hive       | `stg_moex_*`                                       | Развернуть JSON, привести типы, выполнить первичную валидацию          |
| ODS            | Hive       | `ods_moex_*`                                       | Нормализованные данные, близкие к источнику, с бизнес-ключами          |
| DDS            | Greenplum  | `dds.dim_*`, `dds.fact_market_bar_1min`            | Размерная модель, SCD2, конформированные измерения, аналитический факт |
| Data Mart OLAP | ClickHouse | `dm.market_liquidity_hourly`, `dm.market_bar_1min` | Быстрые time-series агрегации, аномалии, часовая витрина               |
| Serving Mart   | Postgres   | `mart.market_liquidity_hourly`                     | Стабильный слой для BI, отчетов и дашбордов                            |

1. **RAW-слой** - здесь будут храниться сырые данные, которые пришли с API биржи. Примерная структура путей:

```txt
/data/raw/moex/instrument/dt=YYYY-MM-DD/batch_id=.../part-*.json
/data/raw/moex/candles/interval=1/dt=YYYY-MM-DD/batch_id=.../part-*.json
```

Метаданные для слоя в манифест-файлах:

- batch_id - индентификатор пакета загрузки
- source_system - MOEX_ISS
- endpoint - ручка к которой обращаемся для получения результатов
- request_url - URL запроса
- http_status - код ответа
- loaded_at - время загрузки в HDFS
- file_path - путь к файлу
- row_count - количество записей
- checksum - контрольная сумма файла

2.  **Staging-слой** - здесь JSON разворачивается в колонки, приводятся типы, добавляются технические поля.
    Основные таблицы партиционируются по дате, для свечей дополнительно по интервалу

Схема staging-слоя:

```mermaid
erDiagram
	stg_moex_instruments {
		string secid
		string isin
		string instrument_name
		string instrument_type
		string section_code
		int lot_size
		string trading_status
		string currency_code
		string company_id
		string company_name
		string sector_id
		string sector_name
		timestamp source_updated_at
		date snapshot_date
		string _batch_id
		string _source_file
		timestamp _loaded_at
	}

	stg_moex_candles_1min {
		string secid
		string section_code
		smallint interval
		timestamp bar_start_local
		timestamp bar_end_local
		timestamp bar_start_utc
		timestamp bar_end_utc
		date trade_date
		decimal open_price
		decimal high_price
		decimal low_price
		decimal close_price
		bigint volume_lots
		bigint volume_shares_raw
		decimal turnover_amount
		int trade_count
		timestamp source_updated_at
		string _batch_id
		string _source_file
		timestamp _loaded_at
	}

	stg_moex_calendar {
		date calendar_date
		date trade_date
		boolean is_trading_day
		boolean is_weekend
		boolean is_holiday
		string exchange_timezone
		string _batch_id
		timestamp _loaded_at
	}

	stg_moex_sessions {
		date trade_date
		string session_code
		string session_type
		string session_name
		timestamp session_start_local
		timestamp session_end_local
		timestamp session_start_utc
		timestamp session_end_utc
		boolean is_regular_session
		int duration_minutes
		string session_status
		string _batch_id
		timestamp _loaded_at
	}
```

3. ODS-слой, близкий к источнику, но уже очищенный, типизированный и пригодный для загрузки в DWH. В ODS будут использоваться бизнес-ключи, суррогатные добавим на слое DDS. ODS хранит снимок, а не историю.

Схема ODS-слоя:

```mermaid
erDiagram
	ods_moex_instrument_snapshot {
		date snapshot_date
		string secid
		string section_code
		string instrument_business_key
		string isin
		string instrument_name
		string instrument_type
		string company_id
		string company_name
		string sector_id
		string sector_name
		int lot_size
		string trading_status
		boolean is_watchlist
		timestamp source_updated_at
		string hash_diff
		timestamp _loaded_at
	}

	ods_moex_market_bar_1min {
		string secid
		string section_code
		string instrument_business_key
		smallint interval
		timestamp bar_start_utc
		timestamp bar_end_utc
		timestamp bar_start_local
		timestamp bar_end_local
		date trade_date
		decimal open_price
		decimal high_price
		decimal low_price
		decimal close_price
		bigint volume_lots
		bigint volume_shares
		decimal turnover_amount
		int trade_count
		string source_batch_id
		timestamp _loaded_at
	}

	ods_moex_date {
		date calendar_date
		date trade_date
		boolean is_trading_day
		boolean is_weekend
		boolean is_holiday
		int day_of_week
		int week_number
		int month_number
		int quarter_month
		int year_number
		string exchange_timezone
	}

	ods_moex_session {
		date trade_date
		string session_code
		string session_type
		string session_name
		timestamp session_start_local
		timestamp session_end_local
		timestamp session_start_utc
		timestamp session_start_utc
		boolean is_regular_session
		int duration_minutes
		string session_status
	}
```

4. Greenplum DDS-слой, ядро хранилища, добавляем суррогатные ключи и храним историю посредством SCD2:

```mermaid
erDiagram
	dim_sector ||--o{ dim_company : классифицирует
	dim_company ||--o{ dim_instrument : выпускает
	dim_currency ||--o{ dim_instrument : котируется
	dim_instrument ||--o{ fact_market_bar_1min : связывается
	dim_date ||--o{ dim_session : определяет
	dim_session ||--o{ fact_market_bar_1min : задает
	dim_date ||--o{ fact_market_bar_1min : задает

	fact_market_bar_1min {
		bigint bar_sk PK
		bigint instrument_sk FK
		bigint session_sk FK
		int date_sk FK
		timestamp bar_start_utc
		timestamp bar_end_utc
		timestamp bar_start_local
		timestamp bar_end_local
		date trade_date
		decimal open_price
		decimal high_price
		decimal low_price
		decimal close_price
		bigint volume_lots
		bigint volume_shares
		decimal turnover_amount
		bigint trade_count
		varchar source_batch_id
		timestamp loaded_at
	}

	dim_sector {
		varchar sector_id PK
		varchar sector_name
		date active_from
		date active_to
		boolean is_current
		timestamp loaded_at
	}

	dim_company {
		varchar company_id PK
		varchar company_name
		varchar sector_id FK
		varchar inn
		varchar issuer_code
		date active_from
		date active_to
		boolean is_current
		timestamp loaded_at
	}

	dim_currency {
		varchar currency_code PK
		varchar currency_name
		varchar symbol
		int minor_unit
		timestamp loaded_at
	}

	dim_instrument {
		bigint instrument_sk PK
		varchar instrument_business_key
		varchar secid
		varchar isin
		varchar instrument_name
		varchar instrument_type
		varchar company_id
		varchar currency_code
		varchar sector_id
		varchar section_code
		int lot_size
		varchar trading_statuis
		boolean is_watchlist
		timestamp valid_from
		timestamp valit_to
		boolean is_current
		varchar hash_diff
		timestamp source_updated_at
		timestamp loaded_at
	}

	dim_date {
		int date_sk PK
		date calendar_date
		date trade_date
		boolean is_trading_day
		boolean is_weekend
		boolean is_holiday
		int day_of_week
		int week_number
		int month_number
		int quarter_number
		int year_number
		varchar exchange_timezone
		int session_count
		timestamp loaded_at
	}

	dim_session {
		bigint session_sk PK
		int date_sk FK
		varchar session_code
		varchar session_type
		varchar session_name
		timestamp session_start_local
		timestamp session_end_local
		timestamp session_start_utc
		timestamp session_end_utc
		boolean is_regular_session
		int duration_minutes
		varchar session_status
		timestamp loaded_at
	}
```

5. ClickHouse Data Mart - используется как аналитический OLAP-слой для часовой витрины ликвидности и аномалий.

```mermaid
flowchart TD
dm.market_bar_1min-- Две витрины, первая для минутной детализации, вторая для агрегирования за час ---dm.market_liquidity_hourly
```

```mermaid
erDiagram
	dm.market_bar_1min {
		datetime bar_start_utc "Начало бара UTC"
		datetime bar_start_local "Начало бара local"
		date trade_date "Торговая дата"
		uint instrument_sk "Ключ версии инструмента"
		string secid "Тикер"
		string session_code "Сессия"
		decimal open_price
		decimal high_price
		decimal low_price
		decimal close_price
		uint volume_shares "Объем в акциях"
		decimal turnover_amount "Оборот"
		uint trade_count "Сделки"
		datetime updated_at "Время обновления"
	}

	dm.market_liquidity_hourly {
		date report_date "Торговая дата"
		datetime report_hour_local "Час по времени биржи"
		datetime report_hour_utc "Час в UTC"
		uint instrument_sk "Версия инструмента"
		string secid "Тикер"
		string instrument_name "Название"
		string company_name "Компания"
		string sector_name "Сектор"
		string currency_code "Валюта"
		string section_code "Секция"
		string session_code "Сессия"
		decimal open_price
		decimal high_price
		decimal low_price
		decimal close_price
		uint volume_lots "Суммарный объем в лотах"
		uint volume_shares "Суммарный объем в акциях"
		decimal turnover_amount "Суммарный оборот"
		decimal trade_count "Суммарное количество сделок"
		decimal price_range "high - low"
		decimal price_range_pct "(high - low) / open"
		decimal raw_hourly_return_pct "close / open - 1"
		uint missing_bars_count "Сколько ожидаемых минут отсутствует"
		decimal volume_zscore "Аномальность объема"
		uint price_jump_flag "Признак резкого движения цены"
		uint abnormal_volume_flag "Признак аномального объема"
		datetime updated_at "Время обновления"
	}
```

6. Postgres Serving Mart - используется как стабильный слой для BI, отчетов и пользовательских дашбордов, на нем будет только часовой отчет:

```mermaid
erDiagram
	mart.market_liquidity_hourly {
		bigserial id "Технический PK"
		date report_date "Торговая дата"
		timestamp report_hour_local
		timestamp report_hour_utc
		bigint instrument_sk "Ключ инструмента"
		varchar secid "Тикер"
		varchar instrument_name "Название"
		varchar company_name "Компания"
		varchar sector_name "Сектор"
		varchar currency_code "Валюта"
		varchar section_code "Секция"
		varchar session_code "Сессия"
		numeric open_price
		numeric high_price
		numeric low_price
		numeric close_price
		bigint volume_lots "Объем в лотах"
		bigint volume_shares "Объем в акциях"
		numeric turnover_amount "Оборот"
		int trade_count "Сделки"
		numeric price_range "Диапазон"
		numeric price_range_pct "Процентный диапазон"
		int missing_bars_count "Пропуски"
		numeric volume_zscore "Аномальность объема"
		boolean price_jump_flag "Скачок цены"
		boolean abnormal_volume_flag "Аномальный объем"
		timestamp refreshed_at "Время обновления"
	}
```

**Архитектура движения данных:**

```mermaid
flowchart LR
	API["MOEX ISS API"]
	subgraph HDFS["HDFS"]
		RAW["RAW<br/>JSON snapshots"]
	end

	subgraph HIVE["Hive"]
		STG["STAGING<br/>parsed typed tables"]
		ODS["ODS<br/>business-key tables"]
	end

	subgraph GP["Greenplum"]
		DDS["DDS<br/>dim_instrument SCD2<br/>dim_date<br/>dim_session<br/>fact_market_bar_1min"]
	end

	subgraph CK["ClickHouse"]
		HMART["Hourly Liquidity & Bar 1MIN"]
	end

	subgraph PG["Postgres"]
		SMART["Serving Mart<br/>BI / Reports"]
	end

	BI["BI / Dashboards"]

	API --> RAW
	RAW --> STG
	STG --> ODS
	ODS --> DDS
	DDS --> HMART
	DDS --> SMART
	HMART --> BI
	SMART --> BI
```

Архитектура хранилища данных построена на 5 слоях источник -> RAW -> STAGING -> ODS -> DDS -> MARTS -> дашборды.

- Источник данных включает справочники инструментов, минутные бары (candles), торговый календарь и торговые сессии;
- Сырые ответы сохраняются в HDFS Raw в формате JSON без изменений, слоя является неизменяемым и обеспечивает воспроизводить ETL, аудит источника и возможность повторной обработки при изменении правил парсинга и валидации, хранение метаданных;
- Данные поступают в Hive Staging, где выполняется парсинг JSON, приведение типов, название полей, добавление технических атрибутов батча и загрузки, также первичная проверка качества: выявляются дубликаты, некорректные значения, пропуски обязательных полей и записи, не попадающие в торговые сессии, если такие имеются;
- Следом данные попадают в Hive ODS, прослойка уже имеет очищенные данные для загрузки в хранилище, но пока что сущности используют бизнес-ключи;
- Ядро хранилища реализуется в Greenplum DDS. Здесь строится звезда. Генерируются суррогатные ключи, имеет историчность, гранулярность - "один инструмент за одну минуту торговой сессии";
- ClickHouse - OLAP Data Mart для минутных баров и часовой витрины ликвидности. ClickHouse - колоночное хранение, эффективные агрегации по временным диапазонам и способность быстро обслуживать аналитические запросы по большим объемам минутных данных;
- Postgres Serving Mart - для BI и отчетности. В нем хранятся агрегированная витрина mart.market_liquidity_hourly. Главная цель предоставить стабильный и совместимый с BI-инструментами интерфейс доступа к готовым аналитическим данным.
