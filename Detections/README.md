# 🛡️ Detection Engineering Portfolio

Esta carpeta contiene las reglas de detección desarrolladas, validadas y documentadas para el SOC Home Lab. El objetivo de este repositorio es demostrar el ciclo completo de *Detection Engineering*, desde la emulación de la amenaza hasta la creación de reglas portables (Sigma) y su implementación en Wazuh y Elastic Security.

## 📊 Matriz de Cobertura MITRE ATT&CK

| Técnica / Sub-técnica | Táctica | Estado | Enlaces |
| :--- | :--- | :--- | :--- |
| **T1059.001** - PowerShell | Execution | 🟢 **Validada** | [Ver Detección](./T1059.001-powershell-encoded/) |

*(Esta tabla se irá actualizando a medida que se agreguen nuevas técnicas).*

---

## 🧠 Metodología de Detección

Cada carpeta dentro de este directorio representa una técnica del MITRE ATT&CK y sigue un ciclo estricto de 5 pasos:

1. **Hipótesis:** ¿Cómo se vería esta técnica en la telemetría (Sysmon/Windows Event Logs)?
2. **Emulación:** Ejecución controlada de la técnica (usando comandos manuales o Atomic Red Team).
3. **Recolección de Evidencia:** Extracción del JSON crudo del evento en Elastic y Wazuh.
4. **Creación de Reglas:** 
   * Se escribe la regla en **Sigma** (como estándar universal).
   * Se traduce e implementa en **Wazuh (XML)** y **Elastic (KQL/JSON)**.
5. **Validación:** 
   * **True Positive (TP):** El ataque genera alerta en ambos SIEMs.
   * **True Negative (TN):** La acción legítima **no** genera alerta, controlando los falsos positivos.

---

## 📂 Estructura de cada Detección

Para garantizar la consistencia, cada carpeta contiene:

*   `README.md`: Análisis técnico, hipótesis, validación y Runbook de Triage.
*   `sigma.yml`: La regla en formato estándar de la industria.
*   `wazuh-rule.xml`: Implementación de la regla en el manager de Wazuh.
*   `elastic-rule.json`: Implementación de la regla en Elastic Security.
*   `evidence/`: JSON crudos de los eventos y capturas de pantalla de las alertas.
