# Технический анализ качества данных: Пошаговый разбор 8 критических дефектов датасета CSE-CIC-IDS2018 по методологии ИСП РАН

При проектировании систем обнаружения вторжений (IDS) на базе машинного обучения качество входных данных определяет стабильность и обобщающую способность моделей. Набор данных **CSE-CIC-IDS2018**, несмотря на широкое применение, содержит ряд глубоких дефектов и физических аномалий, возникших на этапе перехвата трафика и его агрегации утилитой `CICFlowMeter`.

Если передать эти данные в классификатор (например, `Random Forest`) без специализированной обработки, алгоритм обучится на техническом шуме и артефактах сбора данных, полностью потеряв способность обнаруживать реальные угрозы в производственной сети. Ниже представлен подробный разбор 8 ключевых проблем датасета, устраненных в программном конвейере очистки.

---

### Модуль 1. Утечка данных (Data Leakage) и коллинеарные дубликаты столбцов

#### 1. Суть и природа проблемы

Аномалия заключается в наличии статических инфраструктурных параметров испытательного стенда: фиксированных IP-адресов атакующих и жертв, портов источников, идентификаторов потоков и точного времени проведения тестов (`Timestamp`, `Src IP`, `Dst IP`, `Source IP`, `Destination IP`, `Flow ID`, `Src Port`). Деревья решений мгновенно привязываются к этим маркерам, выдавая ложную точность 99.9% на тестовой выборке лаборатории, но полностью теряя способность к классификации в реальной сети с другими IP-адресами. Дополнительно из-за аппаратных сбоев парсера возникли скрытые коллинеарные дубликаты признаков с суффиксами `.1` и `.2`.

#### 2. Программное решение

```python
import pandas as pd
import numpy as np
import glob
import os

input_dir = r"C:\Users\user\CSE-CIC-IDS2018_ML"
source_files = glob.glob(os.path.join(input_dir, "*.csv"))

output_dir = os.path.join(input_dir, "final_cleaned_datasets")
os.makedirs(output_dir, exist_ok=True)

leakage_cols = ['Timestamp', 'Src IP', 'Dst IP', 'Source IP', 'Destination IP']

for file_path in source_files:
    file_name = os.path.basename(file_path)
    if file_name.startswith("final_clean_"):
        continue

    preview = pd.read_csv(file_path, nrows=5)
    cols_to_drop = [col for col in preview.columns if col.endswith('.1') or col.endswith('.2')]
    cols_to_drop.extend([col for col in leakage_cols if col in preview.columns])

```

#### 3. Технический разбор решения

Предварительное чтение первых 5 строк (`nrows=5`) позволяет динамически определить и изолировать аппаратные дубликаты столбцов. Накопленный список `cols_to_drop` отсекает признаки сетевой топологии стенда, предотвращая переобучение модели на конкретных узлах сети.

---

### Модуль 2. Вшитые строки-заголовки и синтаксический мусор целевой переменной (`Label`)

#### 1. Суть и природа проблемы

Дефект возник на этапе грубого слияния исходных CSV-файлов в единые массивы. В результате строки с названиями колонок многократно попали внутрь матриц данных, прерывая числовые последовательности. Присутствие текстовых заголовков внутри числовых полей заставляет Pandas приводить весь столбец к типу `object`, блокируя математические вычисления. Дополнительно целевой признак `Label` содержит невидимые концевые пробелы (например, `"Benign "`), а также орфографические ошибки (`Infilteration`), что искусственно размножает количество классов.

#### 2. Программное решение

```python
    for chunk in pd.read_csv(file_path, chunksize=300000, low_memory=False, engine='c'):
        if 'Label' in chunk.columns:
            chunk = chunk[chunk['Label'].astype(str).str.lower() != 'label']
            chunk['Label'] = chunk['Label'].astype(str).str.strip()

        if 'Flow ID' in chunk.columns:
            chunk = chunk.dropna(subset=['Flow ID'])
            chunk = chunk[chunk['Flow ID'].astype(str).str.strip() != '']

```

#### 3. Технический разбор решения

Потоковый конвейер (`chunksize=300000`) отфильтровывает вшитые заголовки с помощью логической маски не-равенства `'label'`. Метод `.str.strip()` полностью удаляет концевые пробелы, восстанавливая корректное распределение классов и подготавливая строки к исправлению опечаток на этапе финальной сборки. Валидация `Flow ID` исключает сессии с разрушенной структурой идентификаторов.

---

### Модуль 3. Строковый мусор "Infinity" и "NaN" в скоростных характеристиках трафика

#### 1. Суть и природа проблемы

В признаках `Flow Bytes/s` (скорость передачи байт) и `Flow Packets/s` (скорость передачи пакетов) вместо числовых значений содержатся текстовые строки `"Infinity"`, `"-Infinity"` или `"NaN"`. Это следствие обработки микросессий, у которых длительность `Flow Duration = 0` (пакеты прилетели в одну миллисекунду). На уровне Java-кода утилита `CICFlowMeter` вычисляет скорость по формуле:


$$\text{Speed} = \frac{\text{Volume}}{\text{Duration}}$$


При делении ненулевого объема на `0.0` Java возвращает встроенный объект `Double.POSITIVE_INFINITY`, который при конвертации в CSV записывается в виде текста, искажая тип данных всей колонки.

#### 2. Программное решение

```python
        for col in numeric_cols:
            if col in chunk.columns:
                chunk[col] = pd.to_numeric(chunk[col], errors='coerce')

        chunk = chunk.replace([np.inf, -np.inf], np.nan)
        available_numeric = [c for c in numeric_cols if c in chunk.columns]
        chunk = chunk.dropna(subset=available_numeric)

```

#### 3. Технический разбор решения

Метод `pd.to_numeric` с параметром `errors='coerce'` принудительно преобразует типы данных столбцов в числовые, автоматически заменяя строковый мусор `"Infinity"` на системные пустоты `NaN`. Последующий вызов `replace` нейтрализует вещественные бесконечности float, а `dropna` полностью вырезает дефектные строки из текущего чанка.

---

### Модуль 4. Преждевременное закрытие и фрагментация сессий по флагу FIN (Баг FIN)

#### 1. Суть и природа проблемы

Архитектурный дефект логики отслеживания состояний соединений в `CICFlowMeter`. Как только парсер фиксирует первый пакет с установленным флагом `FIN` (запрос на закрытие TCP-соединения), он немедленно уничтожает структуру сессии в оперативной памяти и выгружает её характеристики в лог. Однако в реальном сетевом протоколе за флагом `FIN` всегда следуют пакеты подтверждения закрытия от принимающей стороны (`FIN-ACK`, `ACK`). Утилита захвата видит эти пакеты, но, поскольку исходная сессия уже удалена, она ошибочно генерирует под них новые фантомные сессии, которые содержат пакеты, но имеют нулевую длину полезной нагрузки (`Payload = 0`).

#### 2. Программное решение

```python
        bad_mask = pd.Series(False, index=chunk.index)

        if 'Tot Fwd Pkts' in chunk.columns and 'TotLen Fwd Pkts' in chunk.columns:
            bad_mask |= ((chunk['Tot Fwd Pkts'] > 0) & (chunk['TotLen Fwd Pkts'] == 0))
        if 'Tot Bwd Pkts' in chunk.columns and 'TotLen Bwd Bwd' in chunk.columns:
            bad_mask |= ((chunk['Tot Bwd Pkts'] > 0) & (chunk['TotLen Bwd Pkts'] == 0))

```

#### 3. Технический разбор решения

Булева маска `bad_mask` выявляет физически невозможные сетевые события: ситуации, когда счетчик пакетов в прямом или обратном направлении зафиксировал активность (`> 0`), но суммарная длина переданных байт данных при этом равна нулю. Такие искусственные осколки сессий подлежат тотальному удалению.

---

### Модуль 5. Зависание сессий в памяти из-за игнорирования жесткого сброса RST (Баг RST) и физические дефекты знаковых полей

#### 1. Суть и природа проблемы

Парсер полностью игнорирует TCP-флаг `RST` (Reset). При принудительном жестком сбросе соединения сессия не уничтожается, а продолжает находиться в буфере оперативной памяти утилиты до наступления глобального таймаута. За это время сессия ошибочно вбирает в себя пакеты совершенно других сетевых соединений, которые случайно совпали по номерам портов. Это приводит к лавинообразному переполнению знаковых типов переменных в Java (`integer overflow`), из-за чего параметры портов, длительностей (`Flow Duration`) и межпакетных интервалов (`IAT`) уходят в отрицательный диапазон. Также размеры заголовков раздуваются до физически нереальных масштабов (>10 МБ), а обратные пакеты регистрируются без прямой инициализации соединения.

#### 2. Программное решение

```python
        if 'Dst Port' in chunk.columns:
            bad_mask |= (chunk['Dst Port'] < 0)
        if 'Flow Duration' in chunk.columns:
            bad_mask |= (chunk['Flow Duration'] < 0)

        iat_cols = [c for c in ['Flow IAT Min', 'Fwd IAT Min', 'Bwd IAT Min'] if c in chunk.columns]
        for col in iat_cols:
            bad_mask |= (chunk[col] < 0)

        if 'Init Fwd Win Byts' in chunk.columns:
            bad_mask |= (chunk['Init Fwd Win Byts'] < 0)
        if 'Init Bwd Win Byts' in chunk.columns:
            bad_mask |= (chunk['Init Bwd Win Byts'] < 0)

        if 'Fwd Header Len' in chunk.columns:
            bad_mask |= (chunk['Fwd Header Len'] < 0) | (chunk['Fwd Header Len'] > 10_000_000)
        if 'Bwd Header Len' in chunk.columns:
            bad_mask |= (chunk['Bwd Header Len'] < 0) | (chunk['Bwd Header Len'] > 10_000_000)

        if 'Tot Fwd Pkts' in chunk.columns and 'Tot Bwd Pkts' in chunk.columns:
            bad_mask |= ((chunk['Tot Fwd Pkts'] == 0) & (chunk['Tot Bwd Pkts'] > 0))

        chunk_clean = chunk[~bad_mask]

```

#### 3. Технический разбор решения

Через побитовое операторное сложение `|=` конструируется комплексный фильтр физических аномалий. Удаляются все строки со знаковыми переполнениями (значения `< 0`), аномальные размеры заголовков сетевого и транспортного уровней, превышающие 10 МБ, а также записи, где обратные пакеты летят при нулевом значении прямых пакетов (`Tot Fwd Pkts == 0`), что указывает на пропуск стартового хэндшейка TCP сессии.

---

### Модуль 6. Конфликт таймаутов и хардкод лимитов сессий (120с против 600с)

#### 1. Суть и природа проблемы

В официальной документации к датасету задекларировано, что лимит длительности сетевой сессии (`flowTimeout`) равен 600 секундам (10 минут). По факту при анализе исходного кода `CICFlowMeter` на GitHub была обнаружена жестко захардкоженная константа в 120 секунд. Из-за этого расхождения длинные стандартные легигенные сессии (скачивание файлов, удержание соединений СУБД) принудительно кромсались парсером каждые 2 минуты. Это породило генерацию огромного количества строк-дубликатов с абсолютно идентичными статистическими признаками, искусственно смещающих веса модели машинного обучения.

#### 2. Программное решение

```python
        if is_first_chunk:
            chunk_clean.to_csv(output_path, index=False, mode='w')
            is_first_chunk = False
        else:
            chunk_clean.to_csv(output_path, index=False, mode='a', header=False)

```

#### 3. Технический разбор решения

Потоковая запись кусков данных через `mode='a'` подготавливает очищенные таблицы к финальной дедупликации. На этапе пост-обработки вызов метода `drop_duplicates()` полностью выжигает избыточные дубликаты сетевых сессий, ликвидируя искусственное раздувание объема данных и восстанавливая естественный баланс распределения признаков.

---

### Модуль 7. Ошибочный учет Ethernet Padding (+6 байт из ниоткуда)

#### 1. Суть и природа проблемы

Искажение вызвано некорректным сбором метрик объема трафика на канальном уровне (L2) вместо транспортного (L4). По стандарту Ethernet минимальный размер кадра должен составлять не менее 64 байт. Если пакет слишком маленький (например, пустой TCP ACK с длиной полезной нагрузки Payload = 0), сетевой интерфейс автоматически дополняет его нулями до минимума — этот процесс называется Ethernet Padding (дополнение фрейма, обычно составляет 6 байт). `CICFlowMeter` ошибочно считывал полную длину кадра целиком с сетевого адаптера, прибавляя аппаратный мусор к длине пакетов. Это превратило минимальные длины пакетов (`Fwd Pkt Len Min`, `Bwd Pkt Len Min`) в константные величины (равные 6), отражающие специфику сетевых карт стенда, а не сетевой трафик.

#### 2. Программное решение

```python
ALWAYS_REMOVE = [
    'Flow ID', 'Src IP', 'Src Port', 'Dst IP', 'Dst Port',
    'Protocol', 'CWE Flag Count',
    'Fwd Pkt Len Min', 'Bwd Pkt Len Min',
    'Pkt Size Avg',
    'Fwd Byts/b Avg', 'Fwd Pkts/b Avg', 'Fwd Blk Rate Avg',
    'Bwd Byts/b Avg', 'Bwd Pkts/b Avg', 'Bwd Blk Rate Avg',
    'Subflow Fwd Pkts', 'Subflow Fwd Byts',
    'Subflow Bwd Pkts', 'Subflow Bwd Byts',
]

```

#### 3. Технический разбор решения

Признаки минимальных длин пакетов прямых и обратных направлений вносятся в список `ALWAYS_REMOVE` на принудительное исключение из матриц. Модель полностью избавляется от ложной чувствительности к аппаратному шуму сетевого оборудования.

---

### Модуль 8. Двойной учет первого пакета (баг `Average Packet Size = 9`) и общие константные признаки

#### 1. Суть и природа проблемы

В Java-коде логики инициализации сетевой сессии `CICFlowMeter` допущена ошибка: самый первый прилетевший пакет добавляется в накопительный объект статистики дважды.

Пример математического сбоя метрики `Pkt Size Avg` при передаче 2 пакетов по 6 байт:

1. Прилетает 1-й пакет (6 байт): из-за бага софт добавляет его два раза. Накопленная сумма длин = $6 + 6 = 12$. Счетчик пакетов в объекте статистики = 2.
2. Прилетает 2-й пакет (6 байт): софт обрабатывает его корректно. Накопленная сумма длин = $12 + 6 = 18$. Количество пакетов для расчета среднего размера берется как фактическое количество пакетов в сессии (то есть 2).
3. Итог расчета метрики `Pkt Size Avg` = $\frac{18}{2} = 9$. При этом соседний признак среднего значения длины пакета (`Pkt Len Mean`) рассчитывается через другую переменную, где баг отсутствует, и возвращает верное значение 6.

Взаимоисключающие математические признаки ломают логику ветвления деревьев решений. Также в файлах присутствуют общие скрытые константные признаки (с количеством уникальных значений `nunique <= 1`), замедляющие обучение и не несущие полезной информации.

#### 2. Программное решение

```python
# На этапе пост-обработки всех 10 файлов датасета:
constant_per_file = {}

for FILE in ALL_FILES:
    filepath = os.path.join(FOLDER, FILE)
    df = pd.read_csv(filepath)

    num_cols = df.select_dtypes(include=np.number).columns
    num_constant = set(c for c in num_cols if df[c].nunique() <= 1)

    try:
        txt_cols = df.select_dtypes(include='string').columns
    except:
        txt_cols = df.select_dtypes(include='object').columns
    txt_constant = set(c for c in txt_cols if df[c].nunique() <= 1 and c != 'Label')

    all_constant = num_constant | txt_constant
    constant_per_file[FILE] = all_constant

all_constant_sets = list(constant_per_file.values())
common_constant = all_constant_sets[0]
for s in all_constant_sets[1:]:
    common_constant = common_constant & s

ALL_TO_REMOVE = set(ALWAYS_REMOVE) | common_constant

for FILE in ALL_FILES:
    filepath = os.path.join(FOLDER, FILE)
    df = pd.read_csv(filepath)
    
    if 'Label' in df.columns:
        df['Label'] = df['Label'].str.strip()
        df['Label'] = df['Label'].replace('Infilteration', 'Infiltration')

    df.drop(columns=ALL_TO_REMOVE, inplace=True, errors='ignore')
    df.replace([np.inf, -np.inf], np.nan, inplace=True)
    df.dropna(inplace=True)
    df.drop_duplicates(inplace=True)
    df.to_csv(filepath, index=False)

```

#### 3. Технический разбор решения

Алгоритм сканирует все 10 файлов датасета и находит пересечение константных столбцов, присутствующих во всей выборке (выделено ровно 11 полей, таких как `Bwd Blk Rate Avg`, `Bwd Byts/b Avg`, `Bwd PSH Flags`, `Protocol` и др.). Ломающий математику признак `Pkt Size Avg` удаляется вместе со списком `ALWAYS_REMOVE` и найденными константами (всего удаляется 23 столбца). Метки классов очищаются от опечаток. Каждая таблица перезаписывается in-place, гарантируя строго унифицированную структуру из 60 информативных колонок.

---

### Результаты работы конвейера очистки

Лог успешного выполнения сквозного редактирования файлов на жестком диске:

```text
ШАГ 1: Поиск константных столбцов во всех файлах
  [final_clean_Friday-02-03-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Friday-16-02-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Friday-23-02-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Thuesday-20-02-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Thursday-01-03-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Thursday-15-02-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Thursday-22-02-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Wednesday-14-02-2018_TrafficForML_CICFlowMeter.csv] константных: 12
  [final_clean_Wednesday-21-02-2018_TrafficForML_CICFlowMeter.csv] константных: 13
  [final_clean_Wednesday-28-02-2018_TrafficForML_CICFlowMeter.csv] константных: 12

============================================================
ШАГ 2: Поиск общих константных столбцов (пересечение)
============================================================
  Константны во всех файлах (11):
    - Bwd Blk Rate Avg
    - Bwd Byts/b Avg
    - Bwd PSH Flags
    - Bwd Pkts/b Avg
    - Bwd URG Flags
    - CWE Flag Count
    - Fwd Blk Rate Avg
    - Fwd Byts/b Avg
    - Fwd Pkts/b Avg
    - Fwd URG Flags
    - Protocol

  ВСЕГО будет удалено столбцов: 23

ШАГ 4: Очистка всех файлов (модификация исходных файлов)

  Редактирование файла: final_clean_Friday-02-03-2018_TrafficForML_CICFlowMeter.csv
    Размер: (483066, 79) → (482068, 60)
    Удалено дубликатов: 998
    Классы: {'Benign': 339885, 'Bot': 142183}
    💾 Файл успешно перезаписан: final_clean_Friday-02-03-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Friday-16-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (467891, 79) → (467847, 60)
    Удалено дубликатов: 44
    Классы: {'Benign': 442299, 'DoS attacks-Hulk': 25548}
    💾 Файл успешно перезаписан: final_clean_Friday-16-02-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Friday-23-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (402367, 79) → (400943, 60)
    Удалено дубликатов: 1424
    Классы: {'Benign': 400719, 'Brute Force -Web': 124, 'Brute Force -XSS': 73, 'SQL Injection': 27}
    💾 Файл успешно перезаписан: final_clean_Friday-23-02-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Thuesday-20-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (3169271, 81) → (3135576, 60)
    Удалено дубликатов: 33695
    Классы: {'Benign': 2846287, 'DDoS attacks-LOIC-HTTP': 289289}
    💾 Файл успешно перезаписан: final_clean_Thuesday-20-02-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Thursday-01-03-2018_TrafficForML_CICFlowMeter.csv
    Размер: (104533, 79) → (104188, 60)
    Удалено дубликатов: 345
    Классы: {'Benign': 80218, 'Infiltration': 23970}
    💾 Файл успешно перезаписан: final_clean_Thursday-01-03-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Thursday-15-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (379583, 79) → (377087, 60)
    Удалено дубликатов: 2496
    Классы: {'Benign': 341739, 'DoS attacks-GoldenEye': 27772, 'DoS attacks-Slowloris': 7576}
    💾 Файл успешно перезаписан: final_clean_Thursday-15-02-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Thursday-22-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (400831, 79) → (399402, 60)
    Удалено дубликатов: 1429
    Классы: {'Benign': 399200, 'Brute Force -Web': 142, 'Brute Force -XSS': 39, 'SQL Injection': 21}
    💾 Файл успешно перезаписан: final_clean_Thursday-22-02-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Wednesday-14-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (359797, 79) → (358985, 60)
    Удалено дубликатов: 812
    Классы: {'Benign': 265167, 'SSH-Bruteforce': 93818}
    💾 Файл успешно перезаписан: final_clean_Wednesday-14-02-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Wednesday-21-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (522380, 79) → (522379, 60)
    Удалено дубликатов: 1
    Классы: {'Benign': 358629, 'DDOS attack-HOIC': 163750}
    💾 Файл успешно перезаписан: final_clean_Wednesday-21-02-2018_TrafficForML_CICFlowMeter.csv

  Редактирование файла: final_clean_Wednesday-28-02-2018_TrafficForML_CICFlowMeter.csv
    Размер: (190881, 79) → (190166, 60)
    Удалено дубликатов: 715    
    Классы: {'Benign': 169334, 'Infiltration': 20832}
    💾 Файл успешно перезаписан: final_clean_Wednesday-28-02-2018_TrafficForML_CICFlowMeter.csv
