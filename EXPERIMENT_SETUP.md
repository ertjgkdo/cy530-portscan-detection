# Independent Nmap Traffic Experiment — Reproduction Notes

This document accompanies the CY 530 port-scan detection notebook. It records the independent two-VM traffic collection and external evaluation. Commands are for an **isolated, authorized lab only**. Paths and filenames should be adjusted to match the reproducing system.

## 1. Virtual lab

- Hypervisor: Oracle VirtualBox.
- Scanner: Ubuntu Server 24.04, `10.12.1.10/24`.
- Target: Ubuntu Server 24.04, `10.12.1.20/24`.
- Both machines: VirtualBox Internal Network `cy530-lab`; the internal NIC used during setup was `enp0s8`.
- NAT connectivity was used for package installation; the scan experiment targets the internal network.

Verify the internal interface and reachability:

```bash
ip -br addr
ping -c 3 10.12.1.20
```

## 2. Capture the traffic

On the **scanner VM**, start a separate capture before each scan. The following is a reproducible capture command consistent with the lab setup; the exact historical `tcpdump` command and capture filenames were not preserved in the report. Replace `enp0s8` if your interface differs.

```bash
sudo tcpdump -i enp0s8 -nn -s 0 -w syn_scan.pcap 'host 10.12.1.20'
```

In another terminal on the scanner VM, run the corresponding scan:

```bash
# TCP SYN: ports 1–1000
sudo nmap -sS -Pn -n -p 1-1000 10.12.1.20

# TCP FIN: ports 1–1000
sudo nmap -sF -Pn -n -p 1-1000 10.12.1.20

# UDP: ports 1–100, maximum one retry
sudo nmap -sU -Pn -n -p 1-100 --max-retries 1 10.12.1.20
```

Stop each `tcpdump` capture with Ctrl+C after its scan finishes. For FIN and UDP, run separate captures and use `fin_scan.pcap` and `udp_scan.pcap` respectively. Do not run the scans concurrently when collecting separate labeled scenarios.

## 3. Extract flows with CICFlowMeter

The experiment used the source repository:

- https://github.com/ahlashkari/CICFlowMeter
- Commit: `98a5ebad0df579cc8b43eedd3421b3ae87699901`
- Java 8 and the repository's Gradle 4.2 wrapper.

The CLI reads PCAP packets, constructs flows, and writes flow-level CSV files. The upstream code excluded single-packet flows remaining at final export. To retain those flows, edit `src/main/java/cic/cs/unb/ca/jnetpcap/FlowGenerator.java` in `dumpLabeledCurrentFlow()`:

```java
// Original condition
flow.packetCount() > 1

// Experimental modification
flow.packetCount() >= 1
```

Only the comparison is changed; preserve the surrounding code. The lab also used a local jNetPcap JAR dependency and native-library compatibility setup to build/run the older CICFlowMeter project. These platform-specific build adjustments may need to be repeated on a fresh machine.

The project was run from `/home/ubuntu/CICFlowMeter` using its `exeCMD` Gradle task. The PCAP input path and CSV output directory were configured in the `exeCMD` arguments in `build.gradle`. For the final extraction, the output directory was `/home/ubuntu/csv_output_modified`:

```bash
cd /home/ubuntu/CICFlowMeter
# First set the exeCMD input/output arguments in build.gradle for the selected PCAP.
LD_LIBRARY_PATH="$PWD/compat-libs:$PWD/jnetpcap/linux/jnetpcap-1.4.r1425" ./gradlew exeCMD
```

Repeat with each scan PCAP, setting the input path for that run. `compat-libs` refers to the local compatibility libraries used in this VM; it is not part of a standard fresh clone. The exact working native dependencies and modified `build.gradle` must be retained to make a fully automated rerun possible.

## 4. Prepare external flow CSVs

The external CICFlowMeter output did not have the same schema as the original CICIDS2017 machine-learning CSV (84 versus 79 columns). The notebook:

1. Filters the exported records to scanner-to-target traffic and the expected transport protocol.
2. Maps 13 differing external feature names to their CICIDS2017 counterparts.
3. Selects the same **68 numeric feature columns in the same order** used during model training.
4. Excludes rows with missing or infinite values in required features.
5. Applies the existing Decision Tree and Random Forest models **without retraining**.

Matching names and order permits inference but does **not** establish that both CICFlowMeter versions compute every feature identically. This is an important limitation of the external comparison.

## 5. Observed flow counts and predictions

| Scan | Candidate flows | Valid aligned flows | Decision Tree PortScan predictions | Random Forest PortScan predictions |
|---|---:|---:|---:|---:|
| TCP SYN | 1,000 | 1,000 | 0 | 0 |
| TCP FIN | 1,001 | 999 | 0 | 0 |
| UDP | 193 | 6 | Not evaluated | Not evaluated |

The reported external detection result covers **1,999 valid SYN and FIN flows**. Both models labeled all of them BENIGN. The six valid UDP flows were identified during preparation, but UDP model predictions were **not verified** and are not reported as detection results.

The external captures contain scan traffic only. They cannot support an external false-positive rate or precision estimate. Results may reflect both differences in traffic and differences in flow extraction.

## 6. Reproduce the model training

Download the `Friday-WorkingHours-Afternoon-PortScan.pcap_ISCX.csv` file from the official [CICIDS2017 dataset page](https://www.unb.ca/cic/datasets/ids-2017.html). Open `portscan_detection_cicids2017.ipynb` in Google Colab and update the file paths.

The notebook removes duplicate and invalid records, encodes `BENIGN=0` and `PortScan=1`, uses a stratified 80/20 split (`random_state=42`), removes training-set constant columns, and trains:

```python
DecisionTreeClassifier(random_state=42)
RandomForestClassifier(n_estimators=100, random_state=42, n_jobs=-1)
```

The cleaned dataset contains 213,777 rows and 68 retained numerical features. Execute the notebook cells in order, supplying the separately extracted external CSV files for the independent evaluation.

