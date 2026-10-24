# Security Policy

## Purpose & Scope
This repository contains documentation, detection rules (Sigma, Wazuh XML, Elastic KQL), and configuration files for a personal **SOC & Threat Hunting Home Lab**. 

**This project is for educational and defensive purposes only.** The author does not condone or support the use of these techniques, scripts, or configurations for malicious purposes or against systems without explicit authorization.

## Lab Isolation & Security Controls
To ensure safe operations and prevent accidental exposure, the following security controls are implemented in this lab environment:

1. **Network Isolation:** The entire lab operates within an isolated NAT network (`192.168.200.0/24`) managed by KVM/Libvirt. No services are exposed to the public internet.
2. **TLS Encryption:** Data in transit between endpoints (Winlogbeat) and the SIEM (Elasticsearch) is encrypted and validated using custom Certificate Authorities (CA).
3. **Log Filtering & DoS Prevention:** To prevent log flooding and resource exhaustion on the SIEM infrastructure (3 GB RAM VMs), Windows Event Channels (Security, PowerShell) are strictly filtered using Event IDs and XPath queries.
4. **Time Synchronization:** NTP (Network Time Protocol) is enforced across all hosts to ensure forensic integrity and accurate log correlation.
5. **Endpoint Emulation:** Adversary emulation is performed using safe, non-destructive payloads (e.g., `calc.exe` or local loopback addresses) to avoid actual damage to the infrastructure.

## Reporting a Vulnerability
While this is a personal educational lab, if you discover a security issue, misconfiguration, or a flaw in the detection logic provided in this repository, please report it responsibly.

*   **Contact:** www.linkedin.com/in/julio-cesar-hernandez-tibrey-620b81425
*   **Response Time:** I will acknowledge receipt of your report within 48 hours and provide an estimated timeline for addressing the issue.

Please **do not** open a public GitHub issue for security vulnerabilities. 

## Disclaimer
The detection rules and configurations provided in this repository are designed for a specific lab environment (Windows 10, Sysmon, Wazuh 4.x, Elastic 8.x). 
Applying these configurations to a production environment without proper testing, tuning, and understanding of the baseline may result in false positives, log flooding, or performance degradation. **Use at your own risk.**

The author assumes no liability for any damage or misuse caused by the use of the material in this repository.
