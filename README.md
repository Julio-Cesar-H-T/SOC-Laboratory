# SOC & Threat Hunting Home Lab

![Project Status](https://img.shields.io/badge/Status-Fase%201%20Completada-brightgreen)
![Platform](https://img.shields.io/badge/Hypervisor-KVM%2FLibvirt-blue)
![Stack](https://img.shields.io/badge/SIEM-Wazuh%20%7C%20Elastic%20Stack-orange)

Un entorno de laboratorio defensivo (*Blue Team*) implementado desde cero para simular escenarios reales de un **Security Operations Center (SOC)**. Este proyecto demuestra la ingesta de telemetría, emulación de adversarios (**MITRE ATT&CK**), detección de amenazas mediante KQL y reglas XML, y análisis forense.

---

## 📁 Estructura del Repositorio

```text
soc-home-lab/
├── README.md                            ← Overview general del proyecto (este archivo)
├── fase1-infraestructura/
│   └── SOC-Lab-Fase1-Infraestructura.md ← Documentación completa del despliegue y telemetría
├── fase2-reglas-deteccion/              ← [Próximamente] Reglas XML de Wazuh y Alertas en Elastic
│   └── ...
└── detections/                          ← Artefactos y análisis por TTP (MITRE ATT&CK)
    ├── T1082-winpeas/
    │   └── ...
    └── T1021.004-plink-ssh/
        └── ...
```

---

## 🏗️ Arquitectura General

El laboratorio se ejecuta sobre un hipervisor **KVM/QEMU** en un host Linux (Fedora), configurado sobre una red NAT aislada (`192.168.200.0/24`).

```text
                           [ Fedora Host (KVM/QEMU) ]
                                        │
                         Red NAT: 192.168.200.0/24 (soc-lab)
                                        │
    ┌───────────────────────────┼───────────────────────────┐
    │                           │                           │
┌───▼─────────────────┐ ┌───────▼─────────────────┐ ┌───────▼─────────────────┐
│ VM1: Rocky Linux 8  │ │ VM2: Ubuntu Server 22.04│ │ VM3: Windows 10 Pro     │
│ IP: 192.168.200.96  │ │ IP: 192.168.200.87      │ | IP: 192.168.200.154     │
│                     │ │                         │ │                         │
│ • Wazuh Manager     │ │ • Elasticsearch 8.x     │ │ • Sysmon                │
│ • OpenSearch        │ │ • Kibana                │ │ • Wazuh Agent           │
│ • Wazuh Dashboard   │ │                         │ │ • Winlogbeat            │
└─────────────────────┘ └─────────────────────────┘ └───────────┬─────────────┘
                                                                │
                                                   (Emulación de Ataques)
                                                                │
                                                    ┌───────────▼─────────────┐
                                                    │ VM4: Kali Linux         │
                                                    │ IP: 192.168.200.82      │
                                                    │ (Bajo demanda)          │
                                                    └─────────────────────────┘
```

---

## 🔄 Flujo de Telemetría

El endpoint Windows genera eventos detallados del sistema operativo y del kernel a través de **Sysmon**. Estos eventos son consumidos de forma dual:

1. **Wazuh Agent** $\rightarrow$ **Wazuh Manager** (Respuesta activa, FIM y reglas basadas en XML).
2. **Winlogbeat** $\rightarrow$ **Elasticsearch + Kibana** (Formato ECS, búsquedas rápidas con KQL y visualizaciones).

---

## 🚀 Fases del Proyecto

### ✅ Fase 1: Infraestructura y Telemetría
- [x] Configuración de red virtualizada e hipervisor KVM/QEMU.
- [x] Despliegue de **Wazuh All-in-One** en Rocky Linux 8.
- [x] Despliegue de **Elastic Stack** (Elasticsearch + Kibana) en Ubuntu Server.
- [x] Aprovisionamiento del endpoint **Windows 10 Pro** con **Sysmon** (configuración SwiftOnSecurity).
- [x] Configuración y tuning de **Winlogbeat** y **Wazuh Agent**.
- [x] Validación de ingesta de datos y resolución de problemas técnicos en pipelines.
- [x] Pruebas de emulación iniciales con **Atomic Red Team** y **winPEAS**.

📂 *Ver detalles técnicos en [`Infraestructura del proyecto`](Fase1-Infraestructura/soc_lab_fase1.md).*

---

### 🧪 Fase 2: Reglas de Detección y Tuning *(En Progreso)*
- [ ] Creación de reglas personalizadas XML en Wazuh (`local_rules.xml`).
- [ ] Construcción de *Detection Rules* en Elastic Security basadas en KQL.
- [ ] Reducción de falsos positivos (Tuning de logs por defecto de Windows/PowerShell).

---

### 🛡️ Fase 3: Emulación de Ataques y Artefactos
- [ ] Documentación de detecciones paso a paso para técnicas específicas del MITRE ATT&CK:
  - **T1082** — System Information Discovery (`winPEAS`)
  - **T1021.004** — Remote Services: SSH (`plink.exe`)
  - **T1547.001** — Persistence: Registry Run Keys
  - **T1003.001** — OS Credential Dumping: LSASS
- [ ] Almacenamiento de JSONs de eventos crudos y capturas de pantalla de análisis forense en `detections/`.

---

## 🛠️ Tecnologías y Herramientas

* **Hipervisor:** KVM, QEMU, Libvirt, Virt-Manager
* **Sistemas Operativos:** Rocky Linux 8, Ubuntu Server 22.04 LTS, Windows 10 Pro, Kali Linux
* **SIEM / Visibilidad:** Wazuh 4.x, Elastic Stack 8.x (Elasticsearch, Kibana), Winlogbeat
* **Auditoría de Endpoint:** Microsoft Sysmon, Windows Event Forwarding
* **Emulación / Red Team:** Atomic Red Team, winPEAS, Sysinternals Suite

---

## 👤 Autor

* **Julio Cesar Hernandez Tibrey / Vejetto89**
* **Institución:** Instituto Tecnológico de Las Américas (ITLA)
* **Perfil:** Estudiante de Ciberseguridad / Aspirante a SOC Analyst & Threat Hunter
