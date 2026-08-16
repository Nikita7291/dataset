# Technical analysis of data quality: Step-by-step analysis of 8 critical defects of the CSE-CIC-IDS2018 dataset according to the methodology of the ISP RAS

When designing intrusion detection systems (IDS) based on machine learning, the quality of the input data determines the stability and generalizing ability of the models. The **CSE-CIC-IDS2018** dataset, despite its widespread use, contains a number of deep defects and physical anomalies that occurred at the stage of traffic interception and aggregation by the `CICFlowMeter' utility.

If you transfer this data to a classifier (for example, Random Forest) without specialized processing, the algorithm will learn from technical noise and data collection artifacts, completely losing the ability to detect real threats in the production network. Below is a detailed analysis of 8 key dataset issues fixed in the cleaning software pipeline.

## Raw dataset problems and prerequisites for cleaning

Despite its de facto standard status, the raw dataset **CSE-CIC-IDS2018** contains a number of critical technical issues. Our project implements a strict pre-cleaning pipeline (processing NaNs, infinities and removing duplicates). The need for such steps is confirmed by the research of other authors:

1. **Critical imbalance and lack of cleaning standards:**
In the scientific paper *"A survey and analysis of intrusion detection models based on CSE-CIC-IDS2018 Big Data" (Leevy et al.)* The authors analyzed the existing models and came to the conclusion that most researchers ignore the problem of class imbalance, and there is practically no information about dataset cleaning methods, which ruins the reproducibility of experiments. 
**Research:** [ResearchGate / Springer](https://www.researchgate.net/publication/346536711_A_survey_and_analysis_of_intrusion_detection_models_based_on_CSE-CIC-IDS2018_Big_Data)

2. **Labeling Errors:**
Other researchers who analyzed the structure of the logs in detail revealed that in CSE-CIC-IDS2018 there is a significant proportion of errors in the labeling errors itself, which is estimated at about 6.67%. This directly affects the quality of classifiers' training on raw data.
**Mention of the problem in reviews (Liu et al.):** [arXiv:2402.10974](https://arxiv.org/html/2402.10974v1)

3. **Multicollinearity and redundancy of features (Feature Redundancy):**
An analysis of the extracted CSE-CIC-IDS2018 network streams shows the presence of huge clusters of dependent variables. Many metrics (for example, packet counters and total byte volumes) have a correlation coefficient >0.9. As noted in the fundamental reviews of the dataset, the direct use of all the initial features without correlation analysis (Feature Selection) leads to overfitting algorithms and critical overspending of RAM.
**Investigation of the problem of feature selection (Leevy et al., 2020):** [Journal of Big Data: Analysis of models based on CSE-CIC-IDS2018](https://journalofbigdata.springeropen.com/articles/10.1186/s40537-020-00382-x )
---
### Problem 1. Data Leakage and collinear duplicate columns

####1. The essence and nature of the problem

The anomaly lies in the presence of static infrastructure parameters of the test bench: fixed IP addresses of attackers and victims, source ports, stream identifiers and the exact time of the tests (`Timestamp`, `Src IP`, `Dst IP`, `Source IP`, `Destination IP`, `Flow ID`, `Src Port`). Decision trees are instantly linked to these markers, giving a false accuracy of 99.9% on the laboratory test sample, but completely losing the ability to classify in a real network with other IP addresses. Additionally, due to hardware failures of the parser, hidden collinear duplicates of features with the suffixes `.1` and `.2` appeared.

#### 2. Software solution

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

#### 3. Technical analysis of the solution

Pre-reading the first 5 rows (`nrows=5`) allows you to dynamically identify and isolate hardware duplicate columns. The accumulated list of `cols_to_drop` cuts off the features of the network topology of the stand, preventing retraining of the model on specific network nodes.

---

### Problem 2. Embedded strings-headers and syntactic garbage of the target variable (`Label`)

####1. The essence and nature of the problem

The defect occurred at the stage of rough merging of the original CSV files into single arrays. As a result, the rows with column names repeatedly got inside the data matrices, interrupting the numerical sequences. The presence of text headers inside numeric fields forces Pandas to convert the entire column to the `object` type, blocking mathematical calculations. Additionally, the target feature `Label` contains invisible end spaces (for example, `"Benign "`), as well as spelling errors (`Infiltration`), which artificially multiplies the number of classes.

#### 2. Software solution

```python
    for chunk in pd.read_csv(file_path, chunksize=300000, low_memory=False, engine='c'):
        if 'Label' in chunk.columns:
            chunk = chunk[chunk['Label'].astype(str).str.lower() != 'label']
            chunk['Label'] = chunk['Label'].astype(str).str.strip()

        if 'Flow ID' in chunk.columns:
            chunk = chunk.dropna(subset=['Flow ID'])
            chunk = chunk[chunk['Flow ID'].astype(str).str.strip() != '']

```

#### 3. Technical analysis of the solution

The streaming pipeline (`chunksize=300000`) filters out embedded headers using a logical non-equality mask `label". The '.str.strip()` method completely removes the end spaces, restoring the correct class distribution and preparing the lines for correcting typos at the final assembly stage. The 'Flow ID` validation excludes sessions with a collapsed identifier structure.

---

### Problem 3. String garbage "Infinity" and "NaN" in the speed characteristics of traffic

####1. The essence and nature of the problem

The `Flow Bytes/s` and `Flow Packets/s` attributes contain the text strings `"Infinity"`, `"-Infinity"` or `"NaN"` instead of numeric values. This is a consequence of processing microsessions with a duration of `Flow Duration = 0` (packets arrived in one millisecond). At the Java code level, the `CICFlowMeter` utility calculates the speed using the formula:


![Speed = Volume / Duration](https://latex.codecogs.com/svg.latex?\mathrm{Speed}=\frac{\mathrm{Volume}}{\mathrm{Duration}})

When dividing a non-zero volume by `0.0`, Java returns the built-in object `Double.POSITIVE_INFINITY`, which is written as text when converted to CSV, distorting the data type of the entire column.

#### 2. Software solution

```python
        for col in numeric_cols:
            if col in chunk.columns:
                chunk[col] = pd.to_numeric(chunk[col], errors='coerce')

        chunk = chunk.replace([np.inf, -np.inf], np.nan)
        available_numeric = [c for c in numeric_cols if c in chunk.columns]
        chunk = chunk.dropna(subset=available_numeric)

```

#### 3. Technical analysis of the solution

The `pd.to_numeric` method with the `errors='coerce' parameter forcibly converts column data types to numeric ones, automatically replacing string garbage `"Infinity"` with system voids `NaN'. The subsequent call to `replace` neutralizes the float's real infinities, and `dropna` completely cuts out the defective lines from the current chunk.

---

### Problem 4. Premature closure and fragmentation of sessions by the FIN flag (FIN bug)

####1. The essence and nature of the problem

An architectural defect in the logic of tracking connection states in `CICFlowMeter'. As soon as the parser captures the first packet with the `FIN` flag set (a request to close a TCP connection), it immediately destroys the session structure in RAM and uploads its characteristics to the log. However, in the real network protocol, the `FIN` flag is always followed by confirmation packets from the receiving side (`FIN-ACK`, `ACK`). The capture utility sees these packets, but since the original session has already been deleted, it mistakenly generates new phantom sessions for them, which contain packets but have a payload length of zero (`Payload = 0`).

#### 2. Software solution

```python
        bad_mask = pd.Series(False, index=chunk.index)

        if 'Tot Fwd Pkts' in chunk.columns and 'TotLen Fwd Pkts' in chunk.columns:
            bad_mask |= ((chunk['Tot Fwd Pkts'] > 0) & (chunk['TotLen Fwd Pkts'] == 0))
        if 'Tot Bwd Pkts' in chunk.columns and 'TotLen Bwd Bwd' in chunk.columns:
            bad_mask |= ((chunk['Tot Bwd Pkts'] > 0) & (chunk['TotLen Bwd Pkts'] == 0))

```

#### 3. Technical analysis of the solution

The Boolean mask `bad_mask' detects physically impossible network events: situations where the packet counter in the forward or reverse direction has detected activity (`> 0`), but the total length of the transmitted bytes of data is zero. Such artificial fragments of sessions are subject to total deletion.

---

### Problem 5. Memory sessions are hanging due to ignoring the hard reset of RST (RST bug) and physical defects of the sign fields

####1. The essence and nature of the problem

The parser completely ignores the TCP 'RST' (Reset) flag. If the connection is forcibly hard reset, the session is not destroyed, but remains in the utility's RAM buffer until the global timeout occurs. During this time, the session mistakenly absorbs packets from completely different network connections that accidentally match the port numbers. This leads to an avalanche-like overflow of signed variable types in Java (`integer overflow`), which causes the parameters of ports, durations (`Flow Duration') and packet intervals (`IAT') to fall into the negative range. Also, the header sizes are inflated to physically unrealistic scales (>10 MB), and return packets are registered without direct connection initialization.

#### 2. Software solution

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
#### 3. Technical analysis of the solution

A complex filter of physical anomalies is constructed using bitwise operator addition `|=`. All lines with signed overflows (values `< 0`), abnormal network and transport layer headers exceeding 10 MB, as well as entries where return packets are flying at zero forward packets (`Tot Fwd Pkts == 0`) are deleted, indicating that the TCP session start handshake is skipped.

---

### Problem 6. Conflict of timeouts and hardcore session limits (120c vs 600c)

####1. The essence and nature of the problem

The official documentation for the dataset states that the network session duration limit (`flowTimeout') is 600 seconds (10 minutes). In fact, when analyzing the CICFlowMeter source code on GitHub, a hard-coded constant of 120 seconds was found. Because of this discrepancy, long standard legitimate sessions (downloading files, holding DBMS connections) were forcibly hacked by the parser every 2 minutes. This generated a huge number of duplicate rows with absolutely identical statistical features, artificially shifting the weights of the machine learning model.

#### 2. Software solution

```python
        if is_first_chunk:
            chunk_clean.to_csv(output_path, index=False, mode='w')
            is_first_chunk = False
        else:
            chunk_clean.to_csv(output_path, index=False, mode='a', header=False)

```

#### 3. Technical analysis of the solution

Streaming data chunks via `mode='a' prepares the cleaned tables for final deduplication. At the post-processing stage, calling the `drop_duplicates()' method It completely burns out excess duplicates of network sessions, eliminating artificial data volume bloating and restoring the natural balance of feature distribution.

---

### Problem 7. Incorrect accounting of Ethernet Padding (+6 bytes out of nowhere)

####1. The essence and nature of the problem

The distortion is caused by incorrect collection of traffic volume metrics at the link layer (L2) instead of the transport layer (L4). According to the Ethernet standard, the minimum frame size should be at least 64 bytes. If the packet is too small (for example, an empty TCP ACK with payload length Payload = 0), the network interface automatically complements it with zeros to a minimum — this process is called Ethernet Padding (frame padding, usually 6 bytes). CICFlowMeter mistakenly read the entire length of the frame from the network adapter, adding hardware garbage to the length of the packets. This turned the minimum packet lengths (`Fwd Pkt Len Min`, `Bwd Pkt Len Min`) into constant values (equal to 6), reflecting the specifics of the booth's network cards, rather than network traffic.

#### 2. Software solution

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

#### 3. Technical analysis of the solution

Signs of the minimum packet lengths of forward and reverse directions are added to the `ALWAYS_REMOVE` list for forced exclusion from the matrices. The model completely eliminates the false sensitivity to hardware noise of network equipment.

---

### Problem 8. Double counting of the first packet (bug Average Packet Size = 9) and common constant signs

####1. The essence and nature of the problem

A technical error has been made in the Java code of the CICFlowMeter network session initialization logic: the very first captured packet is added to the cumulative statistics object twice.

**An example of a mathematical failure of the `Pkt Size Avg` metric when transmitting 2 packets of 6 bytes each:**

* **The 1st packet arrives (6 bytes):** due to a bug, the software adds it twice. Accumulated sum of lengths = $6 + 6 = 12$. Packet counter in the statistics object = 2.
* **The 2nd packet arrives (6 bytes):** the software processes it correctly. Accumulated sum of lengths = $12 + 6 = $18. The number of packets to calculate the average size is taken as the actual number of packets per session (i.e. 2).

The result of the metric calculation is $\mathrm{Pkt\Size\Avg} = \frac{18}{2} = 9$. In this case, the neighboring feature of the average packet length (`Pkt Len Mean`) is calculated using another variable where this bug is missing, and returns the correct value of 6.

Mutually exclusive mathematical features break the logic of building branches in tree classification models (for example, Random Forest and Gradient Boosting), as they create false linear dependencies. Additionally, the dataset files contain hidden constant features (the number of unique values of `nunique <= 1`), which have no variance, do not carry useful information for class separation, and slow down the convergence of algorithms.

#### 2. Software solution

```python
# At the post-processing stage of all 10 dataset files:
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

# Forming a single exclusion list
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

#### 3. Technical analysis and specification of the deleted features

During the execution of the filtering algorithm, 23 columns are excluded from the data structure. Below is a detailed analysis of each column being removed, grouped by the nature of the defect.

##### 3.1. A sign with a software calculation defect

* **Pkt Size Avg (or Packet Size Avg):** Average package size. It is excluded due to a critical bug of double counting of the first packet in the session. Its preservation leads to duplication of the information of the "Pkt Len Mean` feature, but with a shift in the mathematical expectation, which distorts the assessment of the weights of the features.

##### 3.2. Common constant features (invariants with a value of 0 in all subsamples)

A group of features related to the statistics of block transfers (Bulk) and specific flags that, within the framework of the CSE-CIC-IDS2018 traffic generation topology, take a strictly constant value of 0:

* **Fwd Byts/b Avg (Forward Bytes/Bulk Avg):** The average number of bytes in a forward transmission block. It is equal to 0 for the entire dataset due to the lack of aggregation of block data by the parser.
* **Fwd Pkts/b Avg (Forward Packets/Bulk Avg):** The average number of packets in the forward transmission unit. A constant zero value.
* **Fwd Blk Rate Avg (Forward Bulk Rate Avg):** Average forward block transfer rate. It has no variance.
* **Bwd Byts/b Avg (Backward Bytes/Bulk Avg):** The average number of bytes in the transmission block in the opposite direction. The constant is 0.
* **Bwd Pkts/b Avg (Backward Packets/Bulk Avg):** The average number of packets in the transmission block in the opposite direction. The constant is 0.
* **Bwd Blk Rate Avg (Backward Bulk Rate Avg):** The average speed of transmission of blocks in the opposite direction. The constant is 0.
* **Bwd PSH Flags:** The number of packets with the PUSH flag set in the opposite direction. Due to the specifics of capturing traffic by the utility, it takes the value 0 in all sessions.
* **Fwd URG Flags:** The number of packets with the URGENT flag set in the forward direction. It does not contain an informative distribution (strict constant 0).
* **Bwd URG Flags:** The number of packets with the URGENT flag set in the opposite direction. It is equal to 0 in all records.
* **CWE Flag Count:** The number of packets with the Congestion Window Reduced flag. It takes a zero value in the entire population.

##### 3.3. Service metadata and network identifiers (ALWAYS_REMOVE list)

These features are removed in order to avoid false training (overfitting) of models for a specific architecture of the test bench and the time intervals of the experiment.:

* **Flow ID:** A unique text identifier of the network stream. It contains unique string values for each session and is not subject to mathematical generalization.
* **Src IP (Source IP):** The IP address of the packet sender. It is excluded that the model does not remember the specific IP addresses of attacking machines recorded on the stand, but is trained on the behavioral characteristics of traffic.
* **Dst IP (Destination IP):** The IP address of the packet recipient. It is deleted to prevent the classifier from being tightly linked to the addresses of victim servers inside the AWS infrastructure.
* **Src Port (Source Port):** The source port of the connection. It is most often generated dynamically by the operating system (dynamic ports from the range 49152-65535), therefore it is random noise for the model.
* **Timestamp:** The time stamp of the stream fixation. This leads to data drift and false attribution of network attacks to a specific time of day or date of the experiment.

The removal of these 23 columns reduces the feature space to 60 semantically independent informative variables, ensuring the identity of the data structure in all ten processed files.

### Cleaning conveyor results

The log of successful end-to-end editing of files on the hard disk:

```text
STEP 1: Search for constant columns in all files
  [final_clean_Friday-02-03- 2018_TrafficForML_CICFlowMeter.csv] constants: 12
  [final_clean_Friday-16-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 12
  [final_clean_Friday-23-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 12
  [final_clean_Thuesday-20-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 12
  [final_clean_Thursday-01-03- 2018_TrafficForML_CICFlowMeter.csv] constant: 12
  [final_clean_Thursday-15-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 12
  [final_clean_Thursday-22-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 12
  [final_clean_weedday-14-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 12
  [final_clean_weedday-21-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 13
  [final_clean_weedday-28-02- 2018_TrafficForML_CICFlowMeter.csv] constants: 12

============================================================
STEP 2: Search for common constant columns (intersection)
============================================================
  Constant in all files (11):
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

  TOTAL columns to be deleted: 23

STEP 4: Cleaning all files (modification of source files)

  File editing: final_clean_Friday-02-03- 2018_TrafficForML_CICFlowMeter.csv
Size: (483066, 79) → (482068, 60)
Duplicates removed: 998
    Classes: {'Benign': 339885, 'Bot': 142183}

  File editing: final_clean_Friday-16-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (467891, 79) → (467847, 60)
Duplicates removed: 44
    Classes: {'Benign': 442299, 'DoS attacks-Hulk': 25548}

  File editing: final_clean_Friday-23-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (402367, 79) → (400943, 60)
Duplicates removed: 1424
    Classes: {'Benign': 400719, 'Brute Force -Web': 124, 'Brute Force -XSS': 73, 'SQL Injection': 27}

  File edit: final_clean_Thuesday-20-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (3169271, 81) → (3135576, 60)
    Duplicates deleted: 33695
    Classes: {'Benign': 2846287, 'DDoS attacks-LOIC-HTTP': 289289}

  File edit: final_clean_Thursday-01-03- 2018_TrafficForML_CICFlowMeter.csv
Size: (104533, 79) → (104188, 60)
Duplicates removed: 345
    Classes: {'Benign': 80218, 'Infiltration': 23970}

  File edit: final_clean_Thursday-15-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (379583, 79) → (377087, 60)
Duplicates removed: 2496
    Classes: {'Benign': 341739, 'DoS attacks-GoldenEye': 27772, 'DoS attacks-Slowloris': 7576}

  File edit: final_clean_Thursday-22-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (400831, 79) → (399402, 60)
Duplicates removed: 1429
    Classes: {'Benign': 399200, 'Brute Force -Web': 142, 'Brute Force -XSS': 39, 'SQL Injection': 21}

  File edit: final_clean_Wednesday-14-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (359797, 79) → (358985, 60)
Duplicates removed: 812
    Classes: {'Benign': 265167, 'SSH-Bruteforce': 93818}

  File edit: final_clean_Wednesday-21-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (522380, 79) → (522379, 60)
    Duplicates removed: 1
    Classes: {'Benign': 358629, 'DDOS attack-HOIC': 163750}

  File edit: final_clean_Wednesday-28-02- 2018_TrafficForML_CICFlowMeter.csv
Size: (190881, 79) → (190166, 60)
Duplicates removed: 715
    Classes: {'Benign': 169334, 'Infiltration': 20832}
``
