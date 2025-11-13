# 🛡️ Blue Team Hunt: Detecting Covert C2 in Network Traffic

### 1. 🌍 Objective & Tools
* **Objective:** Analyze a malicious PCAP file to identify a compromised host and its Command & Control (C2) communication.
* **Tools:** Kali Linux, Zeek, Wireshark

---

### 2. 🔎 Phase 1: Evidence Acquisition
First, I acquired the malicious artifact (a password-protected `.zip` file) and processed it using `7z`. I then used Zeek to process the raw PCAP into a set of structured, hunt-ready logs (`conn.log`, `dns.log`, etc.).

![Successful extraction of the PCAP.](Screenshot%202025-11-13%20161642.png)
![Zeek processing the PCAP into `.log` files.](Screenshot%202025-11-13%20161711.png)

---

### 3.  hunt Phase 2: Triage & Pivoting with Zeek

My hunt began by searching for anomalies in the Zeek logs.

#### Hypothesis 1 (Failed): High-Volume Connections
I first looked for "top talkers" in the `conn.log` to find a simple beacon.

![Initial conn.log 'top talkers' list.](Screenshot%202025-11-13%20162215.png)

* **Finding:** This was inconclusive. The log was "noisy" with legitimate Microsoft traffic. No single IP stood out. The victim was identified as `10.1.17.215`, but its C2 was not obvious.

#### Hypothesis 2 (The Pivot): Anomalous DNS
Since the IP hunt failed, I pivoted to DNS. The `dns.log` immediately revealed two strong suspects:
1.  `ping3.dyngate.com` (344 queries)
2.  `wpad.bluemondaytuesday.com` (32 queries)

![Top DNS queries showing two suspects.](Screenshot%202025-11-13%20162341.png)

#### Hypothesis 3 (Failed Red Herring): DNS Tunneling
The `bluemondaytuesday.com` domain looked more complex. My `awk` command showed it was being appended to long, internal AD queries, suggesting DNS tunneling.

![Longest DNS queries pointing to bluemondaytuesday.](Screenshot%202025-11-13%20163728.png)

* **Finding:** I tested this hypothesis in Wireshark. Both high-level (`dns.qry.name`) and low-level (`frame contains`) filters returned **zero results**. This proved the `bluemondaytuesday.com` traffic was a "ghost" artifact and a red herring. This ability to invalidate a false lead was critical.

![Wireshark filter for 'dns.qry.name' (failed).](Screenshot%202025-11-13%20164502.png)
![Wireshark filter for 'frame contains' (failed).](Screenshot%202025-11-13%20164700.png)
*Above: Proving the `bluemondaytuesday.com` lead was a dead end.*

#### Hypothesis 4 (The Find): High-Frequency DNS Beacon
I returned to my strongest lead: `ping3.dyngate.com`. The sheer volume (344 queries) pointed directly to an automated C2 beacon.

---

### 4. 🔬 Phase 3: Packet-Level Proof with Wireshark

To confirm my final hypothesis, I filtered Wireshark for the true C2.

![The Smoking Gun: Wireshark filter for dyngate.com.](Screenshot%202025-11-13%20164926.png)

* **The Smoking Gun:** The filter `dns.qry.name contains "dyngate.com"` immediately lit up. The screenshot above clearly shows the victim `10.1.17.215` sending hundreds of `ping3.dyngate.com` queries. This is the definitive proof of the C2 beacon.

---

### 5. 🏁 Conclusion: Indicators of Compromise (IOCs)
* **Victim IP:** `10.1.17.215`
* **C2 Domain:** `ping3.dyngate.com`
* **C2 Method:** High-frequency DNS beaconing.

---

### 6. 💡 Proactive Detection & Automation

This hunt was manual, but in a real-world SecOps role, I would automate this detection.

**1. Detection Rule Logic:**
A detection rule could be written to alert on a high frequency of DNS queries to the same non-corporate domain.

* **Logic:** `Alert when a single host makes > 100 DNS queries for the same subdomain in 10 minutes, where the parent domain is NOT in (company.com, google.com, microsoft.com, etc.)`
* This logic could be implemented in a SIEM (like Chronicle's YARA-L or Splunk SPL) to catch this C2 beaconing behavior automatically.

**2. Hunt Automation:**
The `awk` command I used to find the "longest DNS query" can be turned into a simple `bash` script. This script could be run daily against all new Zeek logs, automatically flagging any new domains that use DNS exfiltration techniques.
