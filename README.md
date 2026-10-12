# cy530-portscan-detection
## Port Scan Detection Using Machine Learning

### Overview
This repository contains the experimental implementation for a CY 530 Computer Network Security and Defence Research project investigating the ability of machine learning models trained on CICIDS2017 to detect port-scanning activity in independently captured network traffic. Two supervised learning algorithms, Decision Tree and Random Forest, were trained and evaluated using CICIDS2017. The trained models were subsequently tested on Nmap TCP SYN and FIN scanning traffic generated in a separate virtual-machine environment without retraining.

### Dataset
The training dataset is the Friday-WorkingHours-Afternoon-PortScan.pcap_ISCX.csv file from CICIDS2017's MachineLearningCSV collection.

Official dataset: https://cicresearch.ca/CICDataset/CIC-IDS-2017/
The dataset is not included in this repository and must be downloaded separately.

The original file contains 286,467 records. After removing duplicate records and invalid values, 213,777 records remained. A stratified 80/20 training-test split was used, with 68 numerical features retained for classification.

### Models and Tools
- Decision Tree and Random Forest (scikit-learn)
- Python, pandas, NumPy, and Matplotlib
- Google Colab for model training and evaluation
- VirtualBox and Ubuntu Server 24.04 for the independent network lab
- Nmap for generating scanning traffic
- tcpdump for packet capture
- CICFlowMeter for extracting network-flow features

### Experimental Procedure
1. Load and inspect the CICIDS2017 PortScan dataset.
2. Remove duplicate records and invalid numerical values.
3. Encode binary labels and prepare the training and testing partitions.
4. Train Decision Tree and Random Forest classifiers.
5. Evaluate both models using accuracy, precision, recall, F1-score, and confusion matrices.
6. Generate independent Nmap SYN, FIN, and UDP scanning traffic in a virtual lab.
7. Convert captured packets to network-flow records using CICFlowMeter.
8. Align the external features with the 68 features used during training.
9. Apply the trained models to valid SYN and FIN records without retraining.
10. Examine feature importance and classification errors.
    
The external CICFlowMeter implementation was modified to retain single-packet flows. Differences in exported feature names were mapped to the training dataset's feature names. Equivalent feature names do not necessarily guarantee identical feature calculations.
<img width="499" height="231" alt="image" src="https://github.com/user-attachments/assets/0efeec77-e56e-403f-ab51-2fb8a20eabe9" />

Both models achieved high performance on the CICIDS2017 test set.

However, during external evaluation, neither model identified any of the 1,000 valid SYN flows or 999 valid FIN flows as PortScan. UDP traffic was captured and processed, but model predictions for UDP flows are not included in the reported evaluation.

### Reproducing the Experiment
1. Download the CICIDS2017 dataset from the official source.
2. Open the Jupyter notebook in Google Colab.
3. Install the required Python libraries.
4. Update the dataset paths to match your environment.
5. Execute the training and internal evaluation sections in order.
6. To reproduce external testing, generate Nmap traffic in a controlled environment, process the captures using the documented CICFlowMeter setup, and provide the resulting CSV files at the notebook's expected paths.

## Experimental Setup and Reproducibility

Detailed instructions for the virtual-machine setup,
Nmap traffic generation, packet capture, CICFlowMeter
processing, and external evaluation are available in
[EXPERIMENT_SETUP.md](EXPERIMENT_SETUP.md).

The notebook contains exploratory and diagnostic steps in addition to the main experiment.

###Limitations
The external evaluation used a limited virtual lab and attack-only traffic. It does not establish false-positive rates on independent benign traffic. Differences between CICFlowMeter implementations may also have influenced external model predictions.
