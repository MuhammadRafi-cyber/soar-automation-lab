# Automated Incident Response (SOAR) Lab Framework with Wazuh, n8n, VirusTotal & TheHive

## 📌 Project Overview
This project implements a **SOAR (Security Orchestration, Automation, and Response)** framework within a local laboratory environment to automate cyber threat detection, triage, and incident management in real-time. 

The system is specifically designed to detect adversarial execution behavior (**Mimikatz hacktool**) on a target Windows 11 endpoint, intelligently extract Indicators of Compromise (IoCs), automatically enrich threat intelligence contexts, log incident cases, and deliver comprehensive visual threat summaries directly to the Incident Response (SOC) team.

---

## 🏗️ System Architecture & Workflow

The security automation pipeline executes sequentially and in parallel through the following infrastructure components:

```text
       [ Windows 11 + Sysmon ]
                  │
             (Log Events)
                  ▼
         [ Wazuh Manager ] 
       (Trigger ossec.conf)
                  │
            (Payload JSON)
                  ▼
         [ n8n SOAR Platform ]
          (Regex SHA256 Parser)
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
[ VirusTotal ] [ TheHive ] [ SMTP Brevo ]
 (OSINT API)   (API Alert)      │
                              ▼
                           [ Gmail ]

1. Endpoint Detection (Windows 11 & Sysmon): Adversarial execution of the Mimikatz hacktool on the target Windows 11 host is captured at a deep process level by Microsoft Sysmon and forwarded to the Wazuh Manager.

2. Log Alert Trigger (Wazuh SIEM): The Wazuh Manager analyzes incoming logs against custom correlation rules. Once a critical rule match is met, the <integration> block in ossec.conf automatically transmits a raw JSON alert payload to the n8n Webhook URL.

3. Data Parsing & Extraction (n8n SOAR): The n8n Webhook node ingests the JSON payload. Utilizing a combination of an internal JavaScript node and a Regular Expression (Regex) pattern, n8n cleanly isolates the malware's SHA256 Hash string from the nested win.eventdata.hashes object.

4. Threat Enrichment / OSINT (VirusTotal API): n8n programmatically distributes the extracted SHA256 parameter to the VirusTotal API for real-time global reputation validation, retrieving critical malicious flags and threat classification data without human intervention.

5. Incident Ticketing (TheHive API): n8n authenticates securely via an Authorization Bearer Token / Basic Auth to the /api/v1/alert endpoint of TheHive to automatically generate a structured incident case alert with a HIGH severity level for documentation and triage.

6. Alert Notification (SMTP Relay Brevo to Gmail): As the final defensive response action, n8n compiles all dynamic variables (Wazuh alerts + VirusTotal intelligence) into a custom HTML notification template. This message is fired via SMTP Relay Brevo (Port 587) to safely bypass security restrictions and land instantly in the analyst's Gmail inbox.


**🛠️ Configuration & Deployment Guide**
1. Networking Prerequisites (VirtualBox)

Ensure that your Virtual Machines (VMs) are configured with dual Network Adapters to maintain connectivity:

- Adapter 1 (NAT): Provides public internet access to the VMs so n8n/Docker can reach out to external services like VirusTotal and Brevo SMTP servers.
- Adapter 2 (Host-Only): Establishes a secure local virtual network for cross-VM communication (Windows 11 ↔ Wazuh ↔ n8n ↔ TheHive).

2. Wazuh Manager Configuration

Append the following custom integration block inside the /var/ossec/etc/ossec.conf file on your Wazuh Manager Linux server:

<ossec_config>
  <integration>
    <name>custom-n8n-webhook</name>
    <hook_url>http://<YOUR_N8N_VM_IP>:5678/webhook/<YOUR_N8N_WEBHOOK_ID></hook_url>
    <rule_id>100002</rule_id> <alert_format>json</alert_format>
  </integration>
</ossec_config>

3. Importing the n8n Workflow

1. Download the n8n-workflow/soar_workflow.json file from this repository.

2. Open your n8n dashboard (http://<YOUR_N8N_VM_IP>:5678), click the options icon in the top-right corner of the canvas, and select Import from File.

3. Open each node credential configuration panel to map your environment-specific tokens:

- VirusTotal Node: Insert your personal API Key.
- TheHive Node: Set up Header Auth with Name: Authorization and Value: Bearer <YOUR_THEHIVE_API_KEY>, or utilize Basic Auth (Username & Password of an Authorized User).
- Send an Email Node (SMTP): Insert Host (smtp-relay.brevo.com), Port (587), User (Your Brevo Account Email), and Password (The master SMTP key generated from Brevo). Ensure the From Email parameter matches your verified Brevo sender email address.

**🚀 Testing & Final Results**

When an attack simulation is triggered by executing the Mimikatz binary on the target Windows 11 endpoint, the SOAR framework responds autonomously within milliseconds:

- Automated Orchestration: The entire node sequence inside the n8n canvas triggers sequentially and lights up with a perfect Green Checkmark Success Status.
- Incident Management: A brand new alert titled Wazuh SIEM - Mimikatz Attack Detected automatically populates on TheHive's web console under a New state, mapping back the exact log references from the Wazuh Manager.
- Proactive Notification: A clean, visually professional HTML threat summary lands successfully in the analyst's Gmail inbox, highlighting transparent metrics regarding the malicious flags confirmed by dozens of global antivirus engines.
