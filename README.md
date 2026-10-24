# SOC & Threat Hunting Home Lab

![Project Status](https://img.shields.io/badge/Status-Fase%201%20Completada-brightgreen)
![Detection Engineering](https://img.shields.io/badge/Detection%20Engineering-Active-blue)
![Platform](https://img.shields.io/badge/Hypervisor-KVM%2FLibvirt-blue)
![Stack](https://img.shields.io/badge/SIEM-Wazuh%20%7C%20Elastic%20Stack-orange)

Un entorno de laboratorio defensivo (*Blue Team*) implementado desde cero para simular escenarios reales de un **Security Operations Center (SOC)**. Este proyecto demuestra la ingesta de telemetría, emulación de adversarios (**MITRE ATT&CK**), detección de amenazas mediante EQL/KQL y reglas XML, y análisis forense.

---

## 🏗️ Arquitectura General

El laboratorio se ejecuta sobre un hipervisor **KVM/QEMU** en un host Linux (Fedora), configurado sobre una red NAT aislada (`192.168.200.0/24`).

*   **VM1 (Wazuh):** Wazuh Manager + OpenSearch + Dashboard (Rocky Linux 8).
*   **VM2 (Elastic):** Elasticsearch + Kibana (Ubuntu Server 22.04).
*   **VM3 (Endpoint):** Windows 10 Pro + Sysmon + Wazuh Agent + Winlogbeat (8.19).
*   **VM4 (Atacante):** Kali Linux (Bajo demanda).

📂 *Ver detalles técnicos del despliegue en [`Fase 1 - Infraestructura`](fase1-infraestructura/SOC-Lab-Fase1-Infraestructura.md).*

---

## 🎯 Mapa de Cobertura MITRE ATT&CK

Este repositorio documenta el ciclo completo de *Detection Engineering* (Hipótesis → Emulación → Telemetría → Regla → Validación) para las siguientes técnicas:

| Técnica | Táctica | Estado | Documentación |
|---|---|---|---|
| **T1059.001** (PowerShell) | Execution | ✅ **Validada** | [Ver Detección](detections/T1059.001-powershell-encoded/README.md) |
| **T1059.003** (cmd.exe) | Execution | ⏳ Pendiente | - |
| **T1547.001** (Registry Run Keys) | Persistence | ⏳ Pendiente | - |

---

## 🛠️ Tecnologías y Herramientas

*   **Hipervisor:** KVM, QEMU, Libvirt.
*   **SIEM:** Wazuh 4.x, Elastic Stack 8.19.
*   **Auditoría:** Sysmon (SwiftOnSecurity), Windows Event Forwarding.
*   **Emulación:** Atomic Red Team, comandos manuales de ataque.
*   **Formatos de Detección:** Sigma (YAML), Wazuh (XML), Elastic (KQL/EQL).

---

## 👤 Autor
*   **Julio Cesar Hernandez Tibrey / Vejetto89**
*   **Institución:** Instituto Tecnológico de Las Américas (ITLA)
*   **Perfil:** Estudiante de Ciberseguridad / Aspirante a SOC Analyst & Threat Hunter
