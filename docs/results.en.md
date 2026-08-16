# Conclusion and conclusions

---

## 1. The main scientific and practical results of the work

During the implementation of the project, a comprehensive analytical audit and structured preprocessing of a high-dimensional network security dataset **CSE-CIC-IDS2018** were performed. The key results of the conducted research include the following provisions:

* **Expert audit of the data structure:** A detailed analysis of the architecture of more than 80 signs of network activity has been carried out. Key groups of metrics have been identified that describe the time, volume, and speed characteristics of bidirectional network flows, as well as the distribution structure of the target variable.
* **Identification and localization of defects:** Critical anomalies of the initial sample have been identified and classified, which can distort the mathematical apparatus of predictive models. The nature of the occurrence of missing values (`NaN`), infinities (`inf` / `-inf`), as well as system typing errors caused by header lines littering the sample when merging files has been localized.
* **Mathematical cleaning of the feature space:** All identified destructive factors and incorrect data types have been successfully eliminated. The final matrix of features is reduced to a strictly numerical format without loss of meaningful integrity and representativeness of key classes of network traffic.
* **Statistical profiling and visualization:** Based on the processed data, a set of analytical graphs has been formed (correlation matrices, distribution density histograms, and span diagrams). This made it possible to isolate multicollinear features, assess the degree of class imbalance, and determine the physical nature of extreme statistical outliers reflecting the anomalous nature of network attacks.

---

## 2. Practical significance and applicability of the results

The resulting valid data set has a high degree of internal purity and is fully ready for integration into machine Learning pipelines.

The complete absence of mathematical noise (gaps and infinities) eliminates the risk of gradient divergence and the appearance of uncertainties when calculating loss functions. The prepared dataset can be directly used for training, validation and testing of a wide range of classifiers (such as algorithms based on decision trees, Gradient Boosting/Random Forest ensemble methods and neural networks) aimed at automatic detection, classification and detection of distributed cyber attacks in modern information security systems (IDS).

---

## 3. Acquired competencies and professional skills

Participation in the project allowed us to consolidate in practice a number of key engineering, analytical and team competencies.:

* **Working with high-dimensional data (Big Data):** Gained practical experience in handling heavy data arrays, as well as optimizing data types to reduce the load on computing resources (RAM) when transforming tabular structures.
* **Professional Version Control (Git/ GitHub):** Mechanisms for recording the history of changes, managing project branching, synchronizing local and remote repositories, and deploying static documentation using CI/CD have been mastered at a high level.
* **Team Development (Collaborative Development):** Experience has been gained in allocating areas of responsibility within the engineering group (Collaborators), coordinating changes, resolving merger conflicts, and coordinating actions while simultaneously managing the analytical and software parts of the project.
* **Exploratory Analysis and Visualization (EDA):** The skills of applying statistical data analysis methodologies, identifying hidden relationships, multicollinearity of features, and visual graphical representation of complex multidimensional distributions have been deepened.
