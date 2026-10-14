# Homework #3: Завантаження даних та основи SQL. DQL-команди
---

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1TG18iD08py0eVWUt4r9F2hKtVNVCIhEx?usp=sharing)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12%2B-3776AB?logo=python&logoColor=white)
[![Dataset](https://img.shields.io/badge/Dataset-bank--additional--full.csv-F7931E?logo=databricks&logoColor=white)](https://archive.ics.uci.edu/dataset/222/bank+marketing)

**Варіант C — UCI Bank Marketing** (`bank-additional-full.csv`).

Відтворюваний шлях від публічного датасету до аналітичних висновків:
офіційний ZIP з UCI → оригінальний CSV без змін → `COPY FROM STDIN` у staging (усі колонки `TEXT`) → `INSERT INTO ... SELECT` з очищенням у типізовану таблицю → data-quality audit → розподіли й аномалії → 10 DQL/EDA-запитів → Reflection.

Усі перевірки та EDA виконано **SQL-запитами в PostgreSQL** (клітинки `%%sql`); Python використовується лише для завантаження файлу та `COPY`.

## Структура репозиторію

```
.
├── hw03_bank_marketing.ipynb     # notebook з output усіх клітинок
├── data/
│   └── bank-additional-full.csv  # оригінальний файл UCI без змін (≈5.6 МБ)
├── requirements.txt
└── README.md
```

`data/bank-additional-full.csv` закомічено в репозиторій (5.6 МБ, тобто менше за ліміт 100 МБ). Якщо файлу немає, notebook сам завантажить його з UCI.

## Джерело даних і вибірка

| | |
|---|---|
| Датасет | [UCI Bank Marketing, id = 222](https://archive.ics.uci.edu/dataset/222/bank+marketing) (Moro, Cortez, Rita, 2014), ліцензія CC BY 4.0 |
| Архів | `https://archive.ics.uci.edu/static/public/222/bank+marketing.zip` → вкладений `bank-additional.zip` → `bank-additional/bank-additional-full.csv` |
| Формат | CSV, роздільник `;`, текстові значення в лапках, UTF-8 |
| Обсяг | **41 188 рядків x 21 колонка**: файл узято повністю, і він вписується в рекомендовані 10–100 тис. рядків |
| Період | травень 2008 – листопад 2010 (рядки впорядковані за датою контакту) |
| MD5 | `f6cb2c1256ffe2836b36df321f46e92c` (notebook виводить його для контролю ідентичності файлу) |
| Target | `y`: чи оформив клієнт строковий депозит (`yes` / `no`), 11.27 % `yes` |

**Чому `bank-additional-full.csv`, а не `bank-full.csv`.** У цьому файлі є десяткові макроекономічні показники (для них
потрібен `NUMERIC(p,s)`), спеціальний код `pdays = 999` і більше значень `unknown`. Тому на ньому можна показати всі
перевірки якості.

**Чому не `ucimlrepo`.** `fetch_ucirepo(id=222)` повертає старший набір `bank-full` і вже замінює `unknown` на NaN.
Так зникає різниця між маркером `unknown` у джерелі та справжнім пропуском, а для завдання вона принципова.

Код завантаження (`Завдання 1.2`) рекурсивно шукає потрібний CSV усередині вкладених ZIP і ігнорує службові записи `__MACOSX/`.
Якщо файл уже лежить у `data/`, notebook бере його звідти. Щоб примусово перезавантажити файл з UCI, встановіть `FORCE_DOWNLOAD = True`.

## Як запустити

### Google Colab (рекомендовано)

1. Відкрийте notebook за кнопкою **Open in Colab** вище або завантажте `hw03_bank_marketing.ipynb` через File → Upload notebook.
2. Runtime → **Restart session and run all**.

Перша клітинка встановлює PostgreSQL 16 через `apt-get`, запускає сервер, задає пароль користувачу `postgres` і
перевіряє підключення через `SELECT version()`. Друга клітинка підключає `%%sql` до того самого сервера. Дані
завантажуються з UCI автоматично, тож ручних дій не потрібно.

### Локальний запуск

Потрібні Python **3.12+** і сервер PostgreSQL 16, доступний на `localhost:5432` з користувачем і паролем `postgres` / `postgres`.

```bash
git clone <repo-url> && cd <repo>
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# PostgreSQL, наприклад через Docker:
docker run -d --name hw03-pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 postgres:16

jupyter notebook hw03_bank_marketing.ipynb
```

Рядки `!apt-get`, `!service` і `!sudo` у першій клітинці призначені для Colab (Debian/Ubuntu з root). Якщо PostgreSQL
уже запущено локально або в Docker, закоментуйте їх, а решту notebook виконуйте без змін. Шлях до даних відносний (`data/...`).

## Об'єкти в базі даних

| Об'єкт | Призначення |
|---|---|
| `bank_staging` | сирий імпорт: 21 колонка `*_raw TEXT` і `src_row_num` (identity, тобто порядок рядка у файлі для lineage) |
| `bank_clean` | типізована таблиця: identity `PRIMARY KEY`, `NOT NULL`, 15 `CHECK`-обмежень (2 з них крос-колонкові), `BOOLEAN`, `SMALLINT`/`INTEGER`, `NUMERIC(p,s)`, `DATE`, `TIMESTAMPTZ`, `TEXT` |
| `bank_ml_features` | view для ML: без `duration` (leakage), без технічних колонок і точних дублікатів; невідомі значення закодовано явними індикаторами |

## Зміст notebook

| Завдання | Що зроблено |
|---|---|
| 1. Setup і завантаження | PostgreSQL + `SELECT version()`; завантаження з UCI з виводом MD5 і першого рядка; перевірка заголовка від зсуву колонок; `COPY FROM STDIN` з `DELIMITER ';'`; row-count у staging |
| 2. Staging і DDL | таблиця обґрунтування для кожної колонки (семантика, джерельний формат, тип, правило очищення); DDL `bank_clean`; `INSERT INTO ... SELECT` через CTE `typed → year_breaks → dated`; перевірка відновленої дати через Euribor 3M |
| 3. Data-quality audit | staging vs clean + rejected-рядки; точні дублікати й business key; маркери `NULL` / `''` / `?` / `unknown` / `nonexistent` / `999` по всіх 21 сирій колонці; null-rate 9 колонок у clean |
| 4. Розподіли й аномалії | розподіл `job`; статистика 5 числових колонок (з медіаною та p99); доменний outlier-check, порівняння з правилом 3σ і рішення щодо кожного типу аномалій |
| 5. 10 DQL/EDA-запитів | формат business-question → SQL → interpretation; таблиця покриття всіх обов'язкових конструкцій Теми 3 |
| 6. Reflection | проблеми якості, вибір features, leakage (`duration`, `campaign`), пропозиції для pipeline, тип ML-задачі |

## Ключові рішення щодо очищення

- `unknown` у `job` / `marital` / `education` залишено **окремою категорією** (це інформативна відповідь «не повідомлено»). У двійкових `default` / `housing` / `loan` значення `unknown` перетворено на `NULL` у полі `BOOLEAN`.
- `pdays = 999` перетворено на `NULL`, бо це код «дата попереднього контакту відсутня», а не 999 днів. `poutcome = nonexistent` вважається валідною категорією «попередньої кампанії не було», а не пропуском.
- Року контакту у файлі немає. Його відновлено з порядку рядків (`src_row_num`) і збережено як `DATE` першого дня місяця; день не вигадувався.
- `duration` збережено в `bank_clean`, але виключено з `bank_ml_features`, бо ця величина відома лише після завершення дзвінка (leakage).
- 12 точних дублікатів залишено в `bank_clean`, щоб не втрачати lineage, але відфільтровано у view для ML.

## Основні висновки

- Дисбаланс класів 1 : 7.9 (11.27 % підписок), тому для оцінки моделі потрібні precision / recall і PR-AUC, а не accuracy.
- Найсильніша легітимна ознака — `poutcome = success`: конверсія 65.11 %, тобто у 5.78 раза вища за базову.
- Невідомий статус дефолту сам є сигналом: конверсія 5.15 % проти 12.88 % у клієнтів без дефолту.
- Сильний часовий drift: 67 % записів припадає на 2008 рік. Конверсія зростає з 4.8 % у 2008 до 19.5 % у 2009 і 52.1 % у 2010, тому train/test треба ділити за часом.
- Природна ML-задача — бінарна класифікація (`subscribed`). На практиці це ранжування клієнтів для черги дзвінків.
