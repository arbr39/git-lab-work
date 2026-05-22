# Практика. Аналитика BI — отчёт

**Логин:** `red02`
**XX (ваш номер):** `02`
**ВМ:** `red02@178.154.195.14`

**Доступы к развёрнутым сервисам:**
- Apache Airflow: <http://178.154.195.14:8082> (admin / `q2awCFeW3rbbhDwx`)
- Apache Superset: <http://178.154.195.14:8802> (admin / `admin_red02`)
- Дашборд: <http://178.154.195.14:8802/superset/dashboard/sales-profit-red02/>

---

## Что сделано

| Шаг | Действие | Артефакт |
|---|---|---|
| 1–3 | Подключение по SSH, установлен/проверен Docker, создана структура `~/analytical_practice_red02/{dags,logs,plugins,datasets}` с правами `chmod 777` на `logs`, загружен `sales.csv` в `datasets/` | — |
| 4 | Развёрнут стек `docker compose up -d`: Postgres 15, Airflow 2.7.1, Superset latest-dev, все в общей сети `analytics_network_red02` | `docker-compose.yaml` |
| 5 | Проверены интерфейсы Airflow (`8082`) и Superset (`8802`), получен сгенерированный пароль Airflow | — |
| 6–7 | Создан DAG `clean_sales_pipeline` (ETL: extract csv → transform → load в Postgres), запущен через UI/CLI | `dags/clean_sales_data.py` |
| 8 | Проверена таблица `clean_sales` через `psql` — **9834 строки** | — |
| 9 | В Superset подключён Postgres (`analytics_red02`, `airflow/airflow`) | — |
| 10 | Создан датасет `public.clean_sales` | — |
| 11 | Построены 4 чарта и собран дашборд «Продажи и прибыль» | dashboard `sales-profit-red02` |

**Стек портов (для red02):**
- Postgres: `5402` → внутри контейнера `5432`
- Airflow: `8082` → внутри `8080`
- Superset: `8802` → внутри `8088`

---

## 1. Скриншот `docker compose ps` на ВМ

Все три сервиса в статусе **Up**, Postgres и Superset — **healthy**.

![docker compose ps](screenshots/01_docker_ps.png)

```
NAME             IMAGE                             SERVICE          STATUS                   PORTS
airflow_red02    apache/airflow:2.7.1-python3.10   airflow_red02    Up 29 minutes            0.0.0.0:8082->8080/tcp
postgres_red02   postgres:15                       postgres_red02   Up 29 minutes (healthy)  0.0.0.0:5402->5432/tcp
superset_red02   apache/superset:latest-dev        superset_red02   Up 29 minutes (healthy)  0.0.0.0:8802->8088/tcp
```

---

## 2. Скриншот логов успешно завершённого DAG в Airflow

DAG `clean_sales_pipeline` отработал успешно. В логах задачи `clean_and_load_task` видна строка
`INFO - Загружено 9834 чистых строк.` и `Task exited with return code 0`.

![Airflow DAG logs](screenshots/02_airflow_dag_logs.png)

**Ключевые строки лога:**

```
[2026-05-22, 08:02:48 UTC] {logging_mixin.py:151} INFO - Загружено 9834 чистых строк.
[2026-05-22, 08:02:48 UTC] {python.py:194} INFO - Done. Returned value was: None
[2026-05-22, 08:02:48 UTC] {taskinstance.py:1398} INFO - Marking task as SUCCESS.
[2026-05-22, 08:02:48 UTC] {local_task_job_runner.py:228} INFO - Task exited with return code 0
```

**Проверка в Postgres (внутри ВМ):**

```bash
$ docker exec postgres_red02 psql -U airflow -d analytics_red02 -c "SELECT count(*) FROM clean_sales;"
 count
-------
  9834
(1 row)
```

Количество строк в БД совпадает с тем, что напечатал DAG → ETL отработал корректно.

---

## 3. Полноэкранный скриншот дашборда в Superset

Дашборд «Продажи и прибыль» собран из 4 визуализаций (требование PDF — не менее 3).

![Superset dashboard](screenshots/03_dashboard.png)

URL дашборда (виден IP:port ВМ в адресной строке):

![Superset dashboard URL](screenshots/03_dashboard_url.png)

**Состав дашборда:**

| Чарт | Тип | Поля |
|---|---|---|
| Продажи по месяцам | Line chart (echarts) | X: `Order_Date` (month), Y: `SUM(Sales)` |
| Прибыль по категориям | Pie chart | groupby: `Sub_Category`, metric: `SUM(Profit)` |
| Продажи по регионам | Bar chart (echarts) | X: `Region`, Y: `SUM(Sales)` |
| Прибыльность сегментов по регионам | Heatmap (v2) | X: `Region`, Y: `Segment`, color: `SUM(Profit)` |

---

## 4. Бизнес-выводы / инсайты / рекомендации

### Инсайт 1. Регион West — лидер по выручке, South — стабильный аутсайдер

На баре «Продажи по регионам» видно, что **West (~700k)** и **East (~640k)** генерируют почти вдвое больше выручки, чем **South (~380k)**. Central (~460k) — средний.

**Рекомендация.** Перераспределить маркетинговый бюджет в пользу South: запустить локализованные кампании и проанализировать причины слабого спроса (логистика? ассортимент? цена?). При этом не «обкрадывать» West/East — там работает текущая модель.

### Инсайт 2. Связка «сегмент × регион» — Consumer-West даёт максимум прибыли

Heatmap «Прибыльность сегментов по регионам» показывает аномалию: **Consumer × West = 56.7k** прибыли — это втрое выше среднего по матрице. Самые «холодные» ячейки — **Home Office × South (4.94k)** и **Corporate × Central (9.35k)**.

**Рекомендация.** Изучить паттерн Consumer-West (продуктовый микс, скидочная политика, сезонность) и попробовать его масштабировать на другие региональные комбинации. По «холодным» ячейкам — пересмотреть юнит-экономику или отказаться от убыточных подкатегорий.

### Инсайт 3. Выраженная сезонность: пик в Q4, провал в Q1

На лайн-чарте «Продажи по месяцам» виден устойчивый паттерн: **ноябрь-декабрь — годовые максимумы** (последний пик ~120k в декабре 2017), а **январь-февраль каждый год — заметный провал** (минимумы 10–30k).

**Рекомендация.** Готовить Q4 заранее — увеличить закупки и маркетинговую активность с октября. Для январь-февраля разработать промо-механики (распродажа остатков, B2B-контракты с фиксированной поставкой в Q1), чтобы сгладить «яму» и поддержать денежный поток.

### Инсайт 4. Бонусный — рост год-к-году и фокус на «дорогие» категории

Пир-обзор лайн-чарта показывает не только сезонность, но и **рост выручки по годам** (2017 > 2016 > 2015 > 2014). На pie-чарте «Прибыль по категориям» лидирующие сегменты — **Copiers, Phones, Accessories**, при этом Paper/Binders/Chairs прибыли приносят сильно меньше.

**Рекомендация.** Усилить ассортимент и склад по Copiers/Phones; рассмотреть сокращение позиций в низкоприбыльных подкатегориях, чтобы освободить оборотный капитал.

---

## Что я научился делать сегодня

- Развёртывать аналитический стек **Postgres + Airflow + Superset** через `docker compose` и проверять статус через `docker compose ps`.
- Автоматизировать обработку датасетов через **пайплайн в Airflow** (Python DAG: extract → transform → load) с очисткой дубликатов, починкой дат и `NaN`, заливкой чистых данных в Postgres.
- Подключать **BI-платформу к источнику данных**, создавать датасеты, строить различные типы визуализаций (line / bar / pie / heatmap) и собирать из них дашборд.

---

## Технические особенности реализации (нюансы)

1. **Connection string в DAG.** В тексте PDF в `clean_sales_data.py` указано `postgres_red64:5464` — порт `5464` это **хост-порт** (маппинг `5464:5432`). Внутри docker-сети сервис `postgres_red02` слушает на внутреннем порту `5432`, поэтому в DAG использован корректный URI:
   ```python
   create_engine('postgresql://airflow:airflow@postgres_red02:5432/analytics_red02')
   ```
   С хост-портом `5402` в этом месте DAG не запустился бы.

2. **`AIRFLOW__DATABASE__SQL_ALCHEMY_CONN`.** В `docker-compose.yaml` оставлен без указания порта (`@postgres_red02/analytics_red02`) — psycopg2 по умолчанию идёт на `5432`, что соответствует внутреннему порту контейнера.

3. **Heatmap в Superset latest-dev.** Старый viz-тип `heatmap` в современных версиях Superset (4.x) удалён, его место занял `heatmap_v2` (ECharts-реализация). Чарт «Прибыльность сегментов по регионам» использует `heatmap_v2`.
