# About the project: Research and preparation of the network traffic dataset CSE-CIC-IDS2018

---

## 1. General information

The project is dedicated to the research, profiling and preparation of a reference dataset of network traffic **CSE-CIC-IDS2018** (Collaborative Project of the Communications Security Establishment & Canadian Institute for Cybersecurity). This dataset is an industry standard for the design, testing and validation of modern intrusion detection systems (IDS) and the application of classification algorithms using machine learning methods.

The project examines structured network flows containing both profiles of regular network activity (legitimate traffic) and vectorized traces of targeted cyber attacks of various types. Each record in the dataset is a vector of statistical features extracted from network packets at the transport and network layers of the OSI model.

## Data sources and origin of CSE-CIC-IDS2018

The original dataset **CSE-CIC-IDS2018** was developed jointly by the Canadian Cybersecurity Institute (CIC) and the Canadian Communications Security Center (CSE). The traffic was generated in a large-scale Amazon AWS cloud environment and includes network flow logs extracted using the CICFlowMeter-V3 utility.

**Official resources:**

* **Website of the authors (CIC):** [CSE-CIC-IDS2018 Dataset](http://www.unb.ca/cic/datasets/ids-2018.html )
* **AWS Open Data Registry:** [A Realistic Cyber Defense Dataset](https://registry.opendata.aws/cse-cic-ids2018/)
---

## 2. Relevance of the topic

The dynamic scaling of corporate networks and the increasing complexity of the cyber threat landscape determine strict requirements for automated network security tools. Traditional signature intrusion detection methods are ineffective against Zero-Day vulnerabilities and combined distributed attacks.

The use of mathematical machine learning tools makes it possible to identify hidden patterns and anomalies in network behavior. However, the quality and generalizing ability of preimplicative models directly depend on the purity of the initial sample. Network traffic collected in real time is characterized by high noise, the presence of technical defects, and extreme class imbalance. Studying the structure and correct preparation of such data is an essential foundation for building reliable information security systems.

---

## 3. The purpose and objectives of the project

**The purpose of the project:** Investigation of the internal structure of the network traffic dataset, identification of system anomalies, recording defects, and comprehensive preparation of the feature space for subsequent use in machine learning classification models.

**Project objectives:**

* To study the architecture of network traffic features and metrics in the CSE-CIC-IDS2018 dataset.
* To analyze the distribution structure of the target variable and assess the degree of class imbalance.
* Audit the data for omissions, infinite values, and data type inconsistencies.
* Create requirements for the purification and transformation of the original features.
* Develop analytical visualizations to interpret the relationships between features.
* Prepare a valid data set for the next stage of mathematical modeling.

---

## 4. Dataset description

### 4.1. Data origin and generation infrastructure

The dataset was developed on the basis of the Amazon Web Services (AWS) cloud infrastructure by specialists from the Canadian Cybersecurity Institute. The simulation of network activity included:

* An isolated network topology consisting of 50 target workstations (victim segment);
* Dedicated infrastructure of 30 attacking hosts;
* Collecting raw network traffic in the PCAP format over 10 systematic sessions (days).

A specialized utility **CICFlowMeter V3** was used to extract numerical features from `PCAP` dumps. It groups packets into bidirectional network flows, defined by a 5-signature key (source/receiver IP address, source/receiver port, protocol type), and calculates more than 80 mathematical and time metrics for each stream.

### 4.2. Taxonomy of the presented traffic categories

The dataset contains differentiated records, divided into enlarged classes, reflecting the profiles of real threats.:

1. **Benign:** Reference legitimate traffic generated during the execution of standard user protocols (HTTP, HTTPS, SSH, FTP, DNS).
2. **Brute Force (Brute Force):** Attempts to gain unauthorized access to network services by going through credentials (`FTP-BruteForce`, `SSH-BruteForce').
3. **DoS (Denial of Service):** Attacks aimed at exhausting server resources (memory, processor, connection pool). Include modifications: `DoS-GoldenEye', `DoS-Slowloris', `DoS-SlowHTTPTest', `DoS-Hulk'.
4. **DDoS (Distributed Denial of Service):** Distributed denial of service attacks implemented at the network and application layers: `DDoS-LOIC-HTTP', `DDoS-LOIC-UDP`, `DDoS-HOIC'.
5. **Web Attacks:** Web application vulnerabilities injected into HTTP sessions: `SQL Injection`, `Command Injection`, `Cross-Site Scripting (XSS)'.
6. **Infiltration:** Penetration into the protected perimeter of the network by exploiting client software vulnerabilities, followed by scanning the local network and deploying backdoors.
7. **Botnet:** Traffic generated by infected workstations when interacting with command servers (C&C servers), modeled on the Ares malware.

### 4.3. The structure of the feature space

80 dataset features are divided into the following key categories:

* **Stream IDs:** 'Dst Port' (Destination port), `Protocol' (Transport protocol identifier: TCP/UDP/ICMP).
* **Time characteristics (Duration metrics):**
* `Flow Duration' — the total duration of the network stream (in microseconds).
* `Flow IAT' (Inter-Arrival Time) — statistical indicators of the time intervals between packets (Mean, Std, Max, Min) separately for forward (Fwd) and reverse (Bwd) directions.


* **Volume traffic parameters:** `Total Fwd Packets', `Total Backward Packets` — the total number of transmitted and received packets within the session.
* **Speed characteristics:**
* `Flow Bytes/s' — data transfer rate (bytes per second).
* `Flow Packets/s' — the intensity of the packet flow (packets per second).


* **Packet length statistics:** `Fwd Packet Length Max/Min/Mean/Std`, `Bwd Packet Length Max/Min/Mean/Std' — describe the geometric traffic profile.
* **TCP Header Flags:** 'Fwd PSH Flags', `Bwd URG Flags`, `FIN Flag Count`, `SYN Flag Count`, `RST Flag Count`, `PSH Flag Count`, `ACK Flag Count' — detect the presence of specific control bits specific to port scanning or SYN-Flood attacks.
* **Buffering and Window settings:** `Init_Win_bytes_forward`, `Init_Win_bytes_backward` — the size of the initial TCP window, which is a strong discriminative feature in identifying the host operating system.
* **State Activity Metrics:** `Active Mean/Std/Max/Min` (the time of stream activity before switching to standby mode), `Idle Mean/Std/Max/Min' (the time when the stream is in a passive state).
* **Target attribute:** `Label' is a string label that determines whether the stream belongs to a legitimate class (`Benign`) or a specific type of attack.

---

## 5. Technologies used

A software stack based on the **Python 3 language has been deployed to implement research tasks.**:

* **Pandas:** The main tool for manipulating tabular data structures, aggregation and vectorized column processing.
* **NumPy:** Performing mathematical calculations and operating with system constants of numeric types.
* **Matplotlib / Seaborn:** Construction of a package of statistical graphs, histograms of distribution density and thermal maps of correlation matrices.
* **Git / GitHub:** Providing source code version control and sharing of materials.
* **MkDocs:** Generation of a static project website for maintaining project documentation.

---

## 6. Initial problems and defects in the data

The initial data set of CSE-CIC-IDS2018 contains a number of critical defects that impose limitations on the direct application of the mathematical apparatus of machine learning models.:

### 6.1. Missing Values (NaN)

Non-numeric voids are localized in the columns of speed characteristics. They occur due to failures of the CICFlowMeter utility parser when processing incomplete or corrupted TCP sessions, where it is impossible to correctly calculate the final metrics.

### 6.2. Infinite values (inf)

The presence of positive and negative infinity values in the `Flow Bytes/s` and `Flow Packets/s` features was revealed. This anomaly occurs when the network flow duration is zero (`Flow Duration = 0` microseconds), which leads to the mathematical uncertainty of dividing by zero when calculating the specific intensity.

### 6.3. Incorrect data types

A number of numeric attributes are initially interpreted as a text type (`object'). The reason is that the selection is littered with duplicate header lines with column names that appeared during the concatenation of the original CSV files.

###6.4. Extreme Class imbalance

The statistical distribution of the target variable `Label' is shifted:

* Legitimate traffic (`Benign`): **~87%**
* Total share of network attacks: **~13%**

This factor requires prior consideration when selecting performance metrics for future models, excluding the use of the standard `Accuracy' metric.
