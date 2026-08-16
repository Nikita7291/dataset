# Comprehensive quality audit and intelligent analysis of the CSE-CIC-IDS2018 network security reference dataset

Welcome to the scientific and research web resource dedicated to the comprehensive analysis, verification and deep preprocessing (Data Cleaning) of a large-scale set of network activity data **CSE-CIC-IDS2018**. 

Within the framework of this project, a full cycle of Data engineering has been completed: from auditing hidden source file defects to building fault-tolerant machine learning models **Random Forest**. The main value of the work lies in the detection of 10 critical system and architectural errors in the data that make it impossible to directly apply AI without prior training, as well as in successfully solving the problem of acute class imbalance.

---

## Detailed history and nature of the dataset: What is it?

The **CSE-CIC-IDS2018** (Collaborative DDoS Detection Efficiency Dataset) dataset is currently one of the world's standards for evaluating the quality of intrusion detection systems (IDS). It was developed in 2018 as a joint fundamental project **of the Canadian Cybersecurity Institute (CIC)** at the University of New Brunswick and **the Center for Communications Security (CSE)**.

### Booth architecture and traffic generation:
In order to recreate the traffic structure of a real enterprise, the authors of the study deployed an isolated infrastructure based on the AWS cloud, simulating a medium-scale organization.:
* **Victim Network:** Included 5 isolated organizational departments. There were more than 420 physical and virtual workstations running Windows (10/8.1), as well as 30 servers simulating Active Directory, mail servers, web services, and databases. Legitimate daily employee traffic (web surfing, sending emails, SMB, RPC protocols, etc.) was generated on these machines.
* **The Attacker Network:** A separate infrastructure, from which planned cyber attacks were carried out within 10 days according to a clear schedule.

### Types of recorded network incidents:
During the stand's operation, 14 types of structured attacks were carried out, distributed by day of the week:
1. **Brute Force:** Brute-force password attacks on SSH and FTP protocols.
2. **DoS (Denial of Service):** Attempts to disable servers using the tools Slowloris, GoldenEye, Hulk and Slow HTTP Test.
3. **DDoS (Distributed DoS):** Massive distributed attacks using application and transport layer protocols (HOIC, LOIC tools via TCP/UDP).
4. **Web Attacks:** Vulnerabilities of Web applications (SQL Injection, Cross-Site Scripting (XSS), Brute Force of Web forms).
5. **Infiltration:** Penetration into the network perimeter through the exploitation of software vulnerabilities (for example, Adobe Acrobat), followed by network scanning from the inside and the organization of a botnet (Botnet).

### How raw traffic turned into CSV:
Network packets were intercepted directly from the network interfaces of the switches in the raw 'pcap` format. The total amount of logs was hundreds of gigabytes. To turn this array into a structured view, the authors used the proprietary utility **CICFlowMeter**. She grouped individual packets into bidirectional network sessions (Flows) based on the 5-tuple rule (source/destination IP addresses, ports, protocol type) and calculated **more than 80 statistical features** (session duration, byte transfer rate per second, average packet interval, amount of TCP flags, etc.). The output is a series of CSV tables by day of the week, containing millions of rows.
